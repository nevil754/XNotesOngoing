# Analisi performance: flusso di Ingestion e flusso di Chat AI

> Analisi statica del codice, con riferimenti `file:riga`.
>
> **Aggiornamento**: le ottimizzazioni applicabili sono state implementate nel codice (vedi sezione 4). Restano fuori scope, su indicazione esplicita, i punti su: download/pre-caricamento manuale dei 3 modelli (già gestito manualmente in dev/staging prima della prima query), tuning della concorrenza Celery vs GPU (GPU aziendale molto potente, non è un collo di bottiglia), e il grafo LangGraph (disconnesso volutamente, nessuna ricerca web in uso).

## Verdetto in sintesi

| Flusso | Architettura di base | Stato attuale |
|---|---|---|
| **Ingestion** (`fastapi` + `celery-worker-high`) | Corretta: upload non bloccante, coda Celery dedicata su GPU, batching su Qdrant | **Non ottimizzata al 100%** — c'è un ritardo artificiale di 3s per documento e nessun pre-caricamento dei modelli nei worker (cold-start ripetuto) |
| **Chat AI** | Corretta: hybrid search async, cache Redis sulle query ripetute, streaming SSE | **Non ottimizzata al 100%** — l'endpoint non-streaming raddoppia la latenza con un secondo giro LLM sincrono, e il pre-caricamento modelli è disattivo nell'ambiente realmente usato |

Nessuno dei due problemi principali è "strutturale": sono entrambi correzioni mirate, a basso rischio, che tagliano tempo reale senza toccare l'architettura.

---

## 1. Flusso di Ingestion (`fastapi` → `celery-worker-high`)

### Come funziona oggi
`documents.py` → `DocumentService.upload_and_queue` salva il file e mette in coda `ingest_document` su Celery (queue `high`, che gira solo sul worker GPU) → `ingestion_tasks.py` esegue `run_ingestion_pipeline`: parse (Docling) → clean → chunk → embedding denso (+ sparso) → upsert su Qdrant a batch di 100. Il disaccoppiamento upload/elaborazione via Celery è la scelta giusta: l'utente non aspetta l'ingestion in HTTP.

### Problemi trovati (in ordine di impatto)

**1.1 — Ritardo artificiale di 3 secondi ad ogni documento**
`app/services/document_service.py:90` — `ingest_document.apply_async(..., countdown=3, ...)`.
Ogni singolo documento aspetta 3 secondi *prima ancora* che il worker lo prenda in carico. È un ritardo fisso, pagato sempre, indipendentemente dal carico.
- **Causa probabile**: race condition — `apply_async` (riga 81) viene chiamato *prima* di `await self.db.commit()` (riga 103). Se il worker (che gira su un altro server/processo) legge le tabelle `documents`/`ingestion_jobs` prima che la transazione sia committata, le UPDATE in `ingestion_tasks.py:37-50` non trovano la riga.
- **Correzione corretta**: non è un timing da allungare/accorciare, ma un ordine di operazioni da invertire — chiamare `apply_async` **dopo** il commit (o inserire il job come "pending" e lasciare che sia il worker a fare polling/retry breve). Elimina i 3s fissi per ogni documento.

**1.2 — Nessun pre-caricamento dei modelli nei worker Celery**
I modelli (embedding denso `intfloat/multilingual-e5-large`, sparso `Qdrant/bm25`) sono caricati in modo *lazy* via `@lru_cache` (`app/core/embeddings.py:8-27` e `:54-63`), quindi al primo task che un processo worker riceve. In `main.py:154-166` esiste `_preload_models()`, ma:
- viene chiamato solo dal container **fastapi** (lifespan), mai dai container Celery;
- `celery_app.py` non registra nessun `worker_process_init` signal per pre-caricare i modelli.
- `celery-worker-high` gira con `--concurrency=4` (pool prefork, `docker-compose.yml:77`): sono **4 processi separati**, ognuno con la propria cache `lru_cache` → il cold-load (pesi da disco + inizializzazione CUDA) si ripete fino a 4 volte, una per ogni processo figlio che riceve il suo primo documento.
- **Impatto**: i primi documenti ingeriti dopo ogni riavvio/deploy del container sono molto più lenti (caricamento modello su GPU, tipicamente diversi secondi) di quelli successivi, e questo si ripete per ciascuno dei 4 worker.
- **Correzione corretta**: aggiungere un handler sul segnale Celery `worker_process_init` in `celery_app.py` che chiama `get_embedding_model()` (e `get_sparse_model()` se `qdrant_use_sparse`) appena il processo worker parte, non al primo documento reale.

**1.3 — Imbarazzo tra `concurrency=4` e una singola GPU**
`docker-compose.yml:77` (`celery-worker-high`) usa pool prefork con `concurrency=4` su un container con **1 sola GPU** riservata (`deploy.resources.reservations.devices`, righe 12-18). Con il punto 1.2, questo significa fino a 4 contesti CUDA indipendenti sulla stessa scheda, con conseguente duplicazione di VRAM per gli stessi pesi modello e possibile contesa GPU tra task paralleli.
- **Da verificare con i dati reali**: se la GPU ha VRAM abbondante e i documenti sono piccoli, 4 processi paralleli abbattono il tempo totale di ingestion in caso di upload multipli simultanei (più throughput). Se invece la GPU è satura o si osservano errori OOM, conviene abbassare la concorrenza (es. `--concurrency=2`) o usare un pool `solo`/`threads` che condivide un solo contesto CUDA.
- Non è un bug, ma un parametro da tarare sulla base dell'hardware reale — al momento sembra scelto senza un test di carico documentato nel repo.

**1.4 — `ensure_collection()` fa una chiamata di rete a Qdrant ad ogni documento**
`app/rag/ingestion/pipeline.py:57` chiama `ensure_collection(tenant_slug)` per **ogni** documento ingerito, e `ensure_collection` (`app/core/vectorstore.py:41-58`) fa sempre un `client.get_collection(...)` verso Qdrant per controllare se la collection esiste già, anche quando esiste da tempo.
- È un costo piccolo (una chiamata di rete extra), ma inutile nel 99% dei casi dopo la prima ingestion per tenant.
- **Correzione corretta**: cache in-process (es. `set` di collection già verificate) per saltare la verifica di rete quando la collection è già nota al processo worker.

**1.5 — Embedding denso e sparso calcolati in sequenza**
`app/rag/ingestion/pipeline.py:48-55`: `embed_texts()` (denso, GPU) e poi `embed_sparse_texts()` (sparso/BM25, CPU) vengono eseguiti uno dopo l'altro nello stesso task sincrono. Sono due carichi di lavoro su risorse diverse (GPU vs CPU) che oggi non si sovrappongono.
- **Correzione corretta**: eseguirli in parallelo (es. `ThreadPoolExecutor`/`concurrent.futures`) dentro `run_ingestion_pipeline`, dato che non dipendono l'uno dall'altro. Riduce il tempo di pipeline per documento in proporzione al tempo speso sull'embedding sparso.

**1.6 — Docling con `do_table_structure=True` sempre attivo**
`app/rag/ingestion/parser.py:53` (`ingestion_extract_tables: bool = True` di default, `settings.py:100`). Il riconoscimento strutturale delle tabelle in Docling è l'operazione più lenta della pipeline di parsing PDF, ed è sempre attiva per tutti i documenti, anche quelli senza tabelle rilevanti.
- Non è un difetto se serve qualità sulle tabelle, ma è un trade-off velocità/qualità che oggi non è configurabile per singolo documento/tenant — è un'impostazione globale.

### Cosa è già corretto e non va toccato
- Upload → coda Celery → risposta HTTP immediata (`document_service.py:80-92`): pattern giusto.
- Retry con backoff esponenziale e stato persistito su `ingestion_jobs`/`documents` (`ingestion_tasks.py:117-146`): robusto.
- Upsert Qdrant a batch da 100 (`pipeline.py:91-95`), indici HNSW su soglia (`vectorstore.py:81-84`), payload/vettori `on_disk=True`: scelte corrette per throughput e uso di RAM.
- Invalidazione cache query alla fine dell'ingestion (`ingestion_tasks.py:93-94`): garantisce che i nuovi documenti siano visibili subito in chat.

---

## 2. Flusso di Chat AI

### Come funziona oggi
`chat.py` → `ChatService.query` / `stream_query` → cache Redis su hash query → `retrieve()` (hybrid dense+sparse, RRF, MMR opzionale, reranker cross-encoder) → `arun_rag_chain`/`astream_rag_chain` (LLM) → `validate_answer` → `check_faithfulness` (secondo giro LLM, anti-allucinazione) → salvataggio su SQL Server + Redis.

### Problemi trovati

**2.1 — L'endpoint non-streaming (`/chat/query`) fa due chiamate LLM in sequenza, bloccando la risposta**
`app/services/chat_service.py:85-97`: prima `arun_rag_chain` (genera la risposta), poi — **prima di restituire qualunque cosa al client** — `check_faithfulness` (`hallucination.py`) fa un **secondo** `llm.ainvoke(...)` completo per valutare l'allucinazione. L'utente aspetta la somma delle due latenze LLM, non solo quella della risposta.
- Nel path **streaming** (`stream_query`, righe 188-208) questo è già mitigato bene: i token vengono inviati man mano che arrivano, e il controllo allucinazioni gira *dopo* che l'utente ha già visto la risposta completa (arriva solo nel messaggio `meta` finale). Quindi lo streaming è già ottimizzato correttamente su questo punto.
- **Correzione corretta per `/chat/query`**: restituire la risposta subito dopo `arun_rag_chain`+`validate_answer`, ed eseguire `check_faithfulness` in modo asincrono (fire-and-forget / task in background) aggiornando `hallucination_score` a posteriori sul messaggio salvato, invece di farlo bloccare la response HTTP.

**2.2 — Pre-caricamento modelli attivo solo in ambiente "production", ma il deploy reale è GPU/staging**
`main.py:21-22`: `if settings.app_environment == "production": await _preload_models()`. Il default è `development` (`.env.example:10`), e dai commenti nel `docker-compose.yml` (porte Redis 6379/6380/6381 per dev/staging/prod) il deploy GPU attuale sembra girare come **staging**, non production.
- Risultato pratico: il modello di embedding e il reranker cross-encoder (`BAAI/bge-reranker-v2-m3`, non piccolo) vengono caricati **al volo alla prima vera domanda utente** dopo ogni riavvio del container `fastapi`, invece che durante lo startup. La prima interazione chat di ogni deploy è quindi molto più lenta del normale.
- **Correzione corretta**: il pre-caricamento (embedding + reranker) ha senso farlo sempre quando gira su GPU, non condizionarlo al nome dell'ambiente — es. condizionarlo a `EMBEDDINGS_USE_CUDA=true` oppure semplicemente eseguirlo sempre in `lifespan`, indipendentemente da `app_environment`.

**2.3 — Il grafo LangGraph (routing / web search / SQL agent) non è collegato al flusso reale**
`app/rag/graph/graph.py`, `nodes.py`, `app/rag/agents/router_agent.py` implementano un flusso completo (route → retrieve/web_search → generate → check_hallucination → save_to_memory), ma **nessun endpoint lo chiama** — `chat.py` usa direttamente `ChatService`, che ha una sua logica (senza routing web/SQL, senza `run_rag_graph`). Verificato con ricerca nel codice: `run_rag_graph`/`get_rag_graph` non sono referenziati fuori da `graph.py` stesso.
- Non è un problema di performance in sé (il codice morto non gira, quindi non consuma tempo), ma è codice completo, testato concettualmente, e **non usato**: se l'obiettivo è avere anche instradamento verso web-search o query SQL, oggi non è raggiungibile dall'utente reale. Vale la pena chiarire se è un flusso da collegare o da rimuovere.

**2.4 — Finestra di contesto LLM piccola rispetto al contesto RAG costruito**
`settings.py:38`: `llm_num_ctx: int = 2048` (token). `context_builder.py:11`: `max_context_chars: int = 12000` (caratteri, non token) di contesto documentale, più storico conversazione, più prompt di sistema. 12.000 caratteri sono grossolanamente ~3.000-4.000 token in italiano — già da soli rischiano di superare una finestra di 2048 token prima ancora di aggiungere storico e domanda.
- Non è direttamente un problema di *velocità*, ma di **qualità/affidabilità** delle risposte (contesto troncato silenziosamente da Ollama) che si ripercuote indirettamente sulle performance percepite: risposte incomplete generano più domande di follow-up e più giri di chat.
- Andrebbe allineato `llm_num_ctx` alla reale dimensione di `max_context_chars` + storico, oppure ridotto `max_context_chars` in proporzione a `llm_num_ctx`.

### Cosa è già corretto e non va toccato
- Retrieval interamente async, con embedding query, ricerca sparsa e reranking cross-encoder eseguiti via `run_in_executor` per non bloccare l'event loop (`retriever.py:41-52`, `88-95`, `247-249`).
- Cache Redis sulle query identiche (stessa domanda + stessa conversazione + stessa collection) con hash MD5, sia per `/chat/query` che per `/chat/stream` (`chat_service.py:49-76`, `148-175`): evita di rifare retrieval+LLM su richieste duplicate.
- Streaming SSE (`/chat/stream`) con `X-Accel-Buffering: no` per non bufferizzare lato proxy (`chat.py:112-119`): corretto per bassa latenza percepita.
- `reranker_min_score`/`score_threshold` per scartare chunk poco pertinenti prima di mandarli all'LLM (`retriever.py:81`, `112`): riduce token inutili nel prompt → risposte più veloci e più pulite.

---

## 3. Azioni prioritarie (ordinate per impatto/sforzo)

| # | Azione | Flusso | Impatto | Sforzo | Stato |
|---|---|---|---|---|---|
| 1 | Spostare `apply_async` dopo `db.commit()` ed eliminare `countdown=3` | Ingestion | Alto (−3s fissi per documento) | Basso | ✅ Applicato |
| 2 | Pre-caricare embedding/sparse model nei worker Celery via `worker_process_init` | Ingestion | — | — | ⏭️ Non necessario (modelli già scaricati/caricati manualmente prima della prima query) |
| 3 | Rendere `check_faithfulness` non bloccante su `/chat/query` | Chat | Alto (dimezza la latenza percepita sull'endpoint non-streaming) | Medio | ✅ Applicato |
| 4 | Pre-caricare i modelli sempre su GPU, non solo se `app_environment=="production"` | Chat | — | — | ⏭️ Non necessario (stesso motivo del punto 2) |
| 5 | Parallelizzare embedding denso (GPU) e sparso (CPU) in `run_ingestion_pipeline` | Ingestion | Medio | Basso | ✅ Applicato |
| 6 | Cache in-process per `ensure_collection` (evitare `get_collection` ad ogni documento) | Ingestion | Basso | Basso | ✅ Applicato |
| 7 | Chiarire se il grafo LangGraph (routing/web/SQL) va collegato o rimosso | Chat | — | — | ⏭️ Confermato: disconnesso volutamente, nessuna azione |
| 8 | Rivedere `concurrency=4` su `celery-worker-high` in base alla VRAM reale della GPU | Ingestion | — | — | ⏭️ Non necessario (GPU aziendale molto potente, condivisa con 10-20 LLM già in produzione) |
| 9 | Allineare `llm_num_ctx` a `max_context_chars` (12000) del context builder | Chat | Qualità risposte, indirettamente velocità (meno round-trip) | Basso | ✅ Applicato (`num_ctx` 2048 → 8192, in `config/config.yaml` e `app/core/settings.py`) |

## 4. Dettaglio delle modifiche applicate

**4.1 — Fix race condition + rimozione ritardo fisso (`app/services/document_service.py`)**
`INSERT` su `documents` e `ingestion_jobs` ora vengono committati **prima** di chiamare `ingest_document.apply_async(...)`, così il worker Celery (che gira su un processo/server separato) trova sempre le righe già visibili quando parte. Rimosso `countdown=3`. Il `celery_task_id` viene scritto con un secondo `UPDATE` subito dopo l'accodamento (non bloccante per l'ingestion: se fallisce, lo stesso worker lo scrive comunque all'avvio del task in `ingestion_tasks.py`).

**4.2 — Cache in-process delle collection Qdrant (`app/core/vectorstore.py`)**
Aggiunto un set `_known_collections` in memoria di processo: `ensure_collection()` salta la chiamata di rete `client.get_collection(...)` per i tenant già verificati in questo processo, invece di farla ad ogni singolo documento ingerito. Invalidato correttamente in `adelete_tenant_collections`.

**4.3 — Embedding denso e sparso in parallelo (`app/rag/ingestion/pipeline.py`)**
`embed_texts` (GPU) ed `embed_sparse_texts` (CPU/BM25) vengono ora lanciati insieme via `ThreadPoolExecutor` invece che in sequenza, dato che non dipendono l'uno dall'altro e usano risorse diverse.

**4.4 — Controllo anti-allucinazione non bloccante (`app/services/chat_service.py`)**
Sull'endpoint non-streaming (`ChatService.query`), la risposta viene ora salvata e restituita subito dopo la generazione; il secondo giro LLM (`check_faithfulness`) parte in background (`asyncio.create_task`, tracciato in `_background_tasks` per evitare garbage collection prematura) e aggiorna `hallucination_score` sul messaggio già salvato non appena disponibile. Lo streaming (`/chat/stream`) non è stato toccato: era già ottimizzato correttamente, dato che il controllo allucinazioni lì avviene dopo che l'utente ha già ricevuto tutta la risposta in streaming.

**4.5 — Finestra di contesto LLM allineata al contesto RAG reale (`config/config.yaml`, `app/core/settings.py`)**
`llm.num_ctx` alzato da 2048 a 8192 token: con `max_context_chars=12000` nel context builder più storico conversazione e prompt di sistema, 2048 token erano insufficienti e Ollama troncava silenziosamente il contesto. Nessun problema di risorse dato l'hardware GPU disponibile.
