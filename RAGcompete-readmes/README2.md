# Bug e Inconsistenze — Code Base RAG System

> Nota: questo file era stato azzerato (0 byte) da un evento esterno alla sessione tra il primo e il secondo giro di audit. Ricostruito includendo entrambi i giri.

---

# Giro 1 — Audit generale del codebase

Analisi condotta su tutto il codebase Python (`app/`, `chainlit_app/`, `scripts/`, `tests/`, `config/`, `main.py`). File Docker esclusi dall'ambito.

## BUG CRITICI (crash a runtime, verificati) — ✅ CORRETTI

1. ✅ `app/api/middleware/rate_limit.py:27,45` — `logger` usato senza import → aggiunto `from loguru import logger`.
2. ✅ `app/api/routes/jobs.py` — `logger` usato senza import in `cancel_job` → aggiunto `from loguru import logger`.
3. ✅ `app/api/routes/tenants.py` — `logger` usato senza import → aggiunto `from loguru import logger`.
4. ✅ `app/workers/scheduled_tasks.py:16` — `time.perf_counter()` senza `import time` (task `rollup_usage`, schedulato ogni notte) → aggiunto `import time`.
5. ✅ `scripts/benchmark_retrieval.py:38` — `tenant_db._async_factory()` (attributo privato `None`) → ora usa la property pubblica `tenant_db.async_factory()`.
6. ✅ `scripts/benchmark_retrieval.py:60` — `retrieve(...)` chiamata senza `await` (è `async def`) → aggiunto `await`.

## BUG MINORI / fragilità — ✅ CORRETTI

7. ✅ `app/db/sqlserver.py:105,120` — `EXECUTE AS USER = N'{user_name}'` costruito con f-string → parametrizzato con `text("EXECUTE AS USER = :user_name")`.
8. ✅ `app/services/document_service.py` + `app/api/deps.py:148-152` — race condition tra il commit differito a fine request e il countdown Celery (3s) → `await self.db.commit()` esplicito subito dopo l'insert di `ingestion_jobs`.

## INCONSISTENZE ARCHITETTURALI

9. ⬜ **Il modulo LangGraph è completamente morto.** `app/rag/graph/{graph,nodes,edges,state}.py` implementano un grafo con routing RAG/web-search, ma `run_rag_graph`/`get_rag_graph` non hanno nessun chiamante. Il flusso chat reale (`app/services/chat_service.py`) reimplementa manualmente retrieve+generate+validate+hallucination-check **senza** routing verso web search. `router_agent.py` e `web_agent.py` sono di fatto irraggiungibili dall'API. *(fuori scope: fa parte del flow chat, esplicitamente escluso dalle richieste di fix finora)*

10. ⬜ **`config/config.yaml` in gran parte inefficace.** `_apply_yaml_overrides()` in `app/core/settings.py` mappa solo una parte delle chiavi. Sezioni senza effetto reale via YAML: `ingestion.*`, `web_search.*`, `security.*`, `rate_limit.*`, `retriever.retrieval_strategy` (nome-chiave diverso dal field pydantic `retriever_strategy`), `retriever.auto_filter`, `reranker.model`/`initial_k`, `vectorstore.api_key`/`force_recreate`/`distance`/`on_disk_payload`, `observability.langsmith_project`/`opentelemetry_enabled`, `logging.console_output`/`colored`.

11. ✅ **`config/logging.yaml` non è mai caricato** — logging su file ora **funziona**: aggiunti i campi `log_file_enabled`/`log_file_path`/`log_file_rotation`/`log_file_retention`/`log_file_compression`/`log_file_level` in `app/core/settings.py`, mapping `logging.file.*` in `_apply_yaml_overrides()`, e un secondo sink loguru in `app/core/observability.py::setup_logging()` (stesso formatter testuale della console, `colorize=False`, `enqueue=True`). Attivato di default (`enabled: true`) in `config/config.yaml` (il file realmente letto — `config/logging.yaml` resta non caricato da nessun modulo, ma non serve più: la sezione `file:` effettiva vive in `config.yaml`).

12. ⬜ **`config/prompts.yaml` in gran parte non referenziato.** Solo `system.base`, `rag.main`, `rag.no_context` e `router.classify` vengono letti. `system.with_date`, `rag.insufficient_context`, `query.rewrite`, `query.expand`, `hallucination.check`, `conversation.generate_title`, `sql_agent.system`, `web_agent.system` sono definiti ma mai usati. *(non è stato chiesto di collegare le chiavi inutilizzate — vedi però punto 16 sotto, risolto)*

13. ⬜ **Layer "repository" abbandonato.** `app/db/repositories/EXTRA{base,conversation_repo,document_repo,user_repo}.py` + `app/db/models/EXTRAshared.py` mai importati da alcuna route/service reale.

14. ⬜ **`main.py:52`** — CORS in produzione punta a un placeholder letterale mai sostituito: `allow_origins=[... "https://FRONTEND-OR-BACKENDASPNETCORE"]`, hardcoded.

15. ⬜ **File `OLD*.py` tutti confermati morti** (nessun altro modulo li importa): elenco completo in `app/schemas/`, `app/db/models/`, `app/db/repositories/`, `app/api/middleware/`, `app/rag/agents/`, `app/rag/agents/tools/`, `app/rag/generation/`, `app/rag/ingestion/`, `app/rag/retrieval/`.

## CODICE MORTO / DUPLICATO

- ⬜ `app/mcp/{clients,servers,tools}/` contengono solo `.gitkeep`.
- ⬜ `requirements*_linuxOLD*.txt`, `*_windowsOLD*.txt`, `requirements-dev*_OLD*.txt`.

## ALTRO

- ⬜ **`app/api/routes/auth.py:148,155,161,167`** — log di debug residui con nomi placeholder (`"my1LOG-..."` ecc.) a livello `warning` in `platform_login`.
- ✅ **`app/rag/generation/prompts.py:36`** — path hardcoded `/app/config/prompts.yaml` → **fix**: aggiunto `prompts_config_file: str = "/app/config/prompts.yaml"` in `app/core/settings.py`, stesso pattern di `metadata_config_file`; `_load_prompts()` ora usa `Path(get_settings().prompts_config_file)`. Ora entrambi i file di config seguono lo stesso pattern (settings configurabile via env var, non più un literal hardcoded nel codice) — override possibile con `PROMPTS_CONFIG_FILE`, utile fuori Docker/nei test.
- ⬜ **`app/core/settings.py:39`** — `llm_api_key` definito ma mai letto da nessun codice.

---

# Giro 2 — Audit endpoint (esclusi upload documenti e chat AI) — ✅ TUTTI CORRETTI

Review a 360° di tutti gli endpoint FastAPI in `app/api/routes/`, ESCLUSI `chat.py` e `POST /documents/upload`. Inclusi i middleware, `app/api/deps.py`, e i service layer collegati.

## BUG E INCOERENZE

1. ✅ **Le richieste autenticate solo via `X-API-Key` bypassavano il rate limiting** — `app/api/middleware/tenant.py` valorizzava `request.state.tenant_id`/`user_id` solo dal Bearer JWT, mai da `X-API-Key`, quindi `RateLimitMiddleware` vedeva sempre `None` e saltava il controllo.
   **Fix**: `TenantMiddleware.dispatch` ora gestisce anche `X-API-Key`, riusando `_validate_api_key()` (già presente in `app/api/deps.py`) per popolare `request.state` allo stesso modo del ramo Bearer. Come effetto collaterale si risolve anche la perdita di `tenant_id`/`user_id` nel contesto di `LoggingMiddleware` per queste richieste, e si evita una doppia validazione DB in `get_current_tenant` (che ora trova lo state già popolato).

2. ✅ **Disabilitare un utente/tenant non invalidava i JWT già emessi, e `/auth/refresh` li rinnovava all'infinito senza mai toccare il DB.**
   **Fix**: `POST /auth/refresh` (`app/api/routes/auth.py`) ora rilegge `shared.tenants.is_active` e `users.is_active` dal DB prima di riemettere il token, rispondendo 403 se uno dei due è disattivato — stesso controllo già presente in `/auth/login`. Un utente/tenant disabilitato non può più rinnovare il proprio token.

3. ✅ **`POST /collections` — `IntegrityError` su `qdrant_name` duplicato non gestito → 500 grezzo.**
4. ✅ **`POST /users` — `IntegrityError` su `email` duplicata non gestita → 500 grezzo.**
5. ✅ **`POST /tenants` — `IntegrityError` su `slug` duplicato non gestita → 500 grezzo.**
   **Fix (3-5)**: aggiunto un exception handler globale in `main.py` per `sqlalchemy.exc.IntegrityError` → risposta 409 pulita (`"Conflitto: risorsa già esistente o vincolo di unicità violato"}`) per qualunque violazione di vincolo UNIQUE, invece di un 500 non gestito. Copre questi tre endpoint e qualunque futuro INSERT/UPDATE con lo stesso problema, senza pre-check manuali fragili sotto race condition in ogni route.

6. ✅ **Pattern di "successo silenzioso" su update senza verifica rowcount** — `disable_tenant`, `deactivate_user`, `delete_collection` rispondevano sempre 200/204 anche con zero righe toccate.
   **Fix**: `deactivate_user` (`users.py`) e `disable_tenant` (`tenants.py`) ora controllano `result.rowcount == 0` → 404 se l'ID/slug non esisteva. `delete_collection` (`collections.py`) ora fa un fetch-then-check esplicito (`SELECT ... WHERE id = :id AND is_active = 1`) prima di agire, stesso pattern già usato in `cancel_job`/`delete_document`.

7. ✅ **`DELETE /collections/{collection_id}` non aveva cascata su documenti/vettori Qdrant.**
   **Fix**: `delete_collection` ora, dopo la verifica di esistenza: cancella i vettori Qdrant filtrati per `collection_id` (stesso `FilterSelector` già usato in `delete_document`, con lo stesso comportamento "non fare soft-delete SQL se la cancellazione Qdrant fallisce" → 502), poi marca `status = 'deleted'` su tutti i documenti della collection, infine soft-elimina la collection stessa. Nessun documento/vettore orfano resta ricercabile dopo l'eliminazione di una collection.

## MIGLIORAMENTI SUGGERITI (non richiesti come fix, restano aperti)

1. ⬜ `app/api/routes/health.py:28-53` — timeout espliciti + esecuzione in parallelo (`asyncio.gather`/`wait_for`) per i controlli di readiness.
2. ⬜ Endpoint mancante per riattivare/aggiornare ruolo utente in `users.py`.
3. ⬜ Allineare `create_tenant` al pattern anti-collisione slug di `create_space` (retry-loop con suffisso random) — non implementato: a differenza di `create_space`, lo slug in `create_tenant` è scelto esplicitamente dal superadmin, non derivato da un nome, quindi un retry silenzioso cambierebbe un valore intenzionale; il 409 pulito del nuovo exception handler è già sufficiente a coprire il bug segnalato.
4. ⬜ `/metrics` referenziato nei middleware ma mai implementato come route.


---

## ⚠️⚠️Giro 3 — Audit file Docker ATTUALMENTE NON TOCCO NULLA DI DOCKER!!!⚠️⚠️

Analisi di tutti i file relativi a Docker: `docker-compose.yml`, `docker-compose.override.yml`, `docker/fastapi.Dockerfile`, `docker/celery.Dockerfile`, `docker/chainlit.Dockerfile`, `docker/sqlserver/entrypoint.sh`, `docker/sqlserver/init.sql`, `docker/OLD{qdrant,redis,sqlserver}.yaml`, `.dockerignore`. Solo analisi, nessuna modifica — molte scelte qui sono chiaramente deliberate e già ben ragionate (commenti espliciti nel file stesso), quindi quanto segue sono le cose che sembrano genuinamente in contraddizione con le scelte già fatte altrove nello stesso file, non nitpick stilistici.

### BUG E INCOERENZE

1. **`docker-compose.yml:21` — `BASE_URL_OLLAMA` hardcoded nel blocco `environment:` di `fastapi` (e righe 64/103 per `celery-worker-high`/`celery-worker-default`) rende impossibile cambiare l'endpoint Ollama via `.env` per ambiente.** A differenza di `SQLSERVER_HOST`/`QDRANT_URL` (che puntano a nomi DNS interni al compose stesso, quindi plausibilmente devono restare fissi finché si usa questo file), Ollama **non è un servizio di questo docker-compose** (nessun servizio `ollama:` definito) — `http://srvai2k22:11434/` è un host esterno reale. L'autore stesso ha già riconosciuto esattamente questo problema per `REDIS_URL` (riga 16, commentata con nota "andrebbe commentata... altrimenti il corretto REDIS_URL in .env viene ignorato") ma non lo ha applicato a `BASE_URL_OLLAMA`: cambiare `.env` per puntare a un altro server Ollama (es. GPU diversa in staging/produzione) non avrebbe alcun effetto, perché il valore hardcoded nell'`environment:` del compose vince sempre su quello di `env_file: .env`.

2. **Qdrant esposto su tutte le interfacce senza autenticazione, incoerente con la scelta deliberata di restringere Redis/SQL Server.** `docker-compose.yml:187-189` pubblica le porte Qdrant come `"6333:6333"`/`"6334:6334"` (bind su `0.0.0.0`), mentre `redis` (riga 211) e `sqlserver` (riga 244) usano esplicitamente `127.0.0.1:PORT:PORT` con un commento che ne spiega il motivo ("non ho necessario che sia accessibile dall'esterno... avaible solo x ki sul server, non dalla LAN"). Qdrant in dev non ha nessuna password (`.env.example`: `QDRANT_API_KEY=  #lascia vuoto in dev`), quindi risulta raggiungibile da chiunque sia sulla LAN (o oltre, se l'host ha un IP pubblico) senza credenziali. A differenza di `flower` (che va esposto perché è una dashboard per un umano), Qdrant non ha bisogno di avere le porte pubblicate sull'host: fastapi/celery lo raggiungono già via rete interna (`http://qdrant:6333`), quindi l'esposizione è sia superflua sia il vettore d'accesso più debole dell'intero stack. `docker-compose.override.yml:86-88` ripete la stessa esposizione anche in dev, senza restringerla.

3. **`docker/sqlserver/entrypoint.sh:14-25` — il retry fisso (3 tentativi, ~10s di pausa ciascuno) non aspetta davvero che SQL Server sia pronto, a differenza del loop originale (commentato alle righe 4-12) che faceva un vero polling "attendi finché `SELECT 1` risponde".** Nel `command:` custom di `sqlserver` in `docker-compose.yml:255-263`, `sqlservr` viene avviato in background e `entrypoint.sh` parte subito dopo, senza attendere che accetti connessioni. Su un cold start lento (disco lento, primo avvio, `MSSQL_MEMORY_LIMIT_MB: 2048` piuttosto contenuto) i 3 tentativi possono esaurirsi prima che SQL Server sia pronto ad accettare connessioni: lo script stampa "ATTENZIONE: init.sql non completato dopo 3 tentativi" ed esce con `exit 1`, ma questo exit code **non viene controllato** dal blocco `command:` (niente `set -e`, si passa comunque a `wait $SQLSERVR_PID`) — il container resta "up" ma il database `RAGChat` non esiste mai. La healthcheck (righe 264-275, `USE RAGChat; SELECT 1`) fallirà per sempre, `fastapi` (che dipende da `service_healthy`) non partirà mai, e l'unico segnale visibile è una riga di log facile da perdere in mezzo all'output di boot di SQL Server — nessun crash esplicito che segnali il problema.

4. **`docker/celery.Dockerfile` non crea un utente non privilegiato, a differenza di `fastapi.Dockerfile` (righe 51-56) e `chainlit.Dockerfile` (righe 12-16) — i worker Celery girano come root.** Il file imposta `ENV C_FORCE_ROOT=1` (riga 58), che è esattamente il flag che serve per bypassare il rifiuto di Celery di partire come root — invece di creare un utente dedicato come fanno gli altri due Dockerfile. Questo è rilevante perché i worker `celery-worker-high`/`celery-worker-default` sono quelli che eseguono `run_ingestion_pipeline` su file caricati dagli utenti (PDF/DOCX/XLSX/PPTX non fidati, tramite librerie di parsing/OCR che storicamente hanno avuto CVE); un problema di parsing sfruttabile avrebbe superficie d'attacco più ampia girando come root rispetto a un utente applicativo dedicato.

5. **Porta 5678 pubblicata in `docker-compose.override.yml:30` per debugpy, ma nessun `debugpy` è configurato/avviato da nessuna parte nel codice** (verificato via grep su tutto il repo: zero occorrenze). Mapping di porta morto — o manca la riga `import debugpy; debugpy.listen(...)` in `main.py`/entrypoint dev, oppure è un residuo di un setup di debug remoto mai completato.

6. **`docker-compose.yml:132` — `celery-beat` imposta `HOME: "/cache/fastembed"` ma non monta mai un volume su quel path per questo servizio** (a differenza di `fastapi`/`celery-worker-high`/`celery-worker-default`, che montano tutti `${DATA_ROOT}/fastembed-cache:/cache/fastembed`). Probabile copia-incolla dagli altri blocchi `environment:`: `celery-beat` è puro scheduler (RedBeat), non carica mai i modelli di embedding, quindi la variabile non ha effetto pratico — ma la sua presenza senza il volume corrispondente è fuorviante per chi legge il file.

### INCOERENZE MINORI

7. **`docker-compose.yml:131` — `celery-beat` imposta esplicitamente `REDIS_URL: ${REDIS_URL}` in `environment:`, mentre `fastapi`/`celery-worker-high`/`celery-worker-default` commentano deliberatamente la stessa riga** (con nota "andrebbe commentata"). RedBeat legge l'URL Redis da `settings.redis_url` (`app/workers/celery_app.py:56`), cioè dalle Pydantic Settings popolate da `env_file: .env` — la stessa via usata da tutti gli altri servizi. La riga per `celery-beat` è quindi ridondante (usa `${...}`, non un literal, quindi non è "sbagliata" come lo sarebbe stata con un valore hardcoded, ma è comunque un'inconsistenza rispetto al pattern applicato altrove nello stesso file) — probabile residuo di una sessione di debug in cui si sospettava che RedBeat non leggesse correttamente l'URL.

8. **`docker/OLD{qdrant,redis,sqlserver}.yaml` sono frammenti superati** (compose v3.8 pre-consolidamento, nessuna healthcheck, nessuna auth su Redis, porte pubblicate su tutte le interfacce) **e `docker/OLDsqlserver.yaml:12` contiene ancora una password SA in chiaro** (`SA_PASSWORD: "YourStrong!Passw0rd"`) committata nel repo. File morti (nessun compose li referenzia), ma vale la pena rimuoverli: sia per igiene, sia perché una password in chiaro in un file YAML — anche se "solo" quella di sviluppo e anche se non più in uso — è comunque una cattiva pratica da non lasciare in git.

9. **`.dockerignore` non esclude i requirements obsoleti** (`requirements_linuxOLD*.txt`, `requirements_windowsOLD.txt`, `requirements-dev*OLD*.txt`, 8 file totali) — dato che `context: .` in `docker-compose.yml` usa la root del repo come build context, questi file vengono comunque inviati al Docker daemon ad ogni build (gonfiano il context e possono invalidare la cache più spesso del necessario) anche se nessun Dockerfile li `COPY`a mai (solo `requirements.txt` viene copiato).

### FRAGILITÀ DA MONITORARE (non bug conclamati, da tenere d'occhio)

10. **`docker/fastapi.Dockerfile:23` — `pip install --no-cache-dir --no-deps -r requirements.txt`**: `--no-deps` richiede che `requirements.txt` sia sempre un lockfile completo (generato via `pip-compile` da `requirements.in`) per l'ambiente Linux/py3.11-slim-bookworm del container. Una modifica manuale a `requirements.txt`, o una rigenerazione dimenticata dopo aver toccato `requirements.in`, non farebbe fallire la build ma produrrebbe `ImportError` solo a runtime dentro un container già avviato. La presenza di 4 varianti "OLD" (`requirements_linuxOLD.txt`, `linuxOLD2`, `linuxOLD3`, `windowsOLD`) suggerisce che ottenere un lockfile Linux funzionante sia già stato un punto di attrito in passato.

11. **`docker/fastapi.Dockerfile:62-63` — `CMD` di "produzione" usa `--workers 1`.** Se lo scaling previsto è verticale (più worker Uvicorn nello stesso container) questo è un collo di bottiglia; se invece è orizzontale (più repliche del container fastapi dietro un load balancer, non presente in questo compose) allora è corretto così. Da confermare in base alla strategia di deploy reale, non deducibile dai soli file Docker.

12. **`depends_on` incoerente tra servizi**: `fastapi` richiede `sqlserver: condition: service_healthy` (rigoroso) ma solo `redis: condition: service_started` e `qdrant: condition: service_started` (righe 25-31), pur avendo entrambi una healthcheck definita (righe 196-201, 229-234). `fastapi` può quindi partire prima che Redis/Qdrant siano realmente pronti a rispondere — mitigato solo parzialmente dai fallback fail-open già presenti nel codice applicativo (es. rate limiting) ma non da tutti i path (es. `_check_services()` in `main.py` logga solo un warning e continua comunque).

---

## ⭐️⭐️⭐️ Giro 4 — Audit flow upload documenti e chat AI — ✅ TUTTI CORRETTI

Review a 360° dei due flow rimasti esclusi dai giri precedenti: upload documenti e chat con l'AI. File Docker non toccati/letti in dettaglio, su richiesta esplicita (restano fuori scope dopo il Giro 3).

## FLOW UPLOAD DOCUMENTI — Bug e incoerenze

1. ✅ **CRITICO — `app/rag/ingestion/pipeline.py:56`** — `ensure_collection(tenant_slug, settings.qdrant_force_recreate)` veniva chiamata su OGNI singola ingestion di documento, non solo al provisioning del tenant. Se `QDRANT_FORCE_RECREATE=true`, la prima chiamata a `ensure_collection` per QUALSIASI upload cancellava e ricreava l'INTERA collection Qdrant del tenant (`vectorstore.py:56`, `client.delete_collection`), perdendo tutti i vettori di tutti i documenti già ingeriti — mentre le righe SQL `documents.status='ready'` restavano invariate, quindi l'app continuava a mostrarli come "pronti" pur essendo invisibili al retrieval.
   **Fix**: `ensure_collection(tenant_slug)` chiamata senza propagare `qdrant_force_recreate` — resta idempotente (crea solo se non esiste già), mai distruttiva durante un upload.

2. ✅ **`app/rag/ingestion/parser.py:59-71` + `app/rag/ingestion/chunker.py:64-68`** — il numero di pagina associato ai chunk era spesso sbagliato/assente per documenti Docling: `full_text` (markdown export) e `pages` (item.text grezzo) erano due rappresentazioni testuali diverse dello stesso documento, e `_find_page_number` cercava `chunk_text[:100] in page_text` tra le due senza normalizzazione — il match falliva spesso.
   **Fix**: `_parse_with_docling` ora applica `clean_text()` (stessa funzione usata su `full_text` in pipeline.py) anche a ciascuna pagina prima di restituirla. `_find_page_number` normalizza sia il chunk che le pagine rimuovendo la sintassi markdown (`#`, `*`, `` ` ``, `|`, ecc.) e gli spazi multipli prima del confronto, invece di un substring match sul testo "sporco".

3. ✅ **`app/rag/ingestion/metadata.py:43-55`** — `_classify_document` veniva richiamata una volta per OGNI chunk invece che una volta per documento, con lo stesso identico input — spreco CPU ripetuto ad ogni ingestion.
   **Fix**: `classify_document` (rinominata, ora pubblica) è richiamata una sola volta in `pipeline.py` prima del loop sui chunk; `build_chunk_metadata` ora riceve `doc_type` già calcolato invece di ricalcolarlo internamente ad ogni chiamata.

4. ✅ **`app/workers/ingestion_tasks.py:156-183` (`reprocess_document`)** — non inseriva mai una riga in `ingestion_jobs` prima di schedulare `ingest_document`, che quindi avrebbe aggiornato/confuso la riga storica della ingestion originale invece di tracciare il nuovo tentativo separatamente. *(nota: task tuttora non raggiungibile da nessuna route/servizio — dead code, corretto comunque per completezza/coerenza)*
   **Fix**: inserita una nuova riga `ingestion_jobs` (status `'queued'`, `celery_task_id`) subito dopo lo scheduling del task, stesso pattern già usato in `document_service.py`.

## FLOW CHAT AI — Bug e incoerenze

5. ✅ **`app/rag/generation/hallucination.py:34-36`** — il fallback su errore di `check_faithfulness` restituiva `1.0` (massima fedeltà), l'opposto di un default prudente per un controllo anti-allucinazione.
   **Fix**: il ramo `except Exception` ora ritorna `0.0` invece di `1.0`.

6. ✅ **`app/rag/generation/hallucination.py:13` vs `app/rag/memory/context_builder.py`** — `check_faithfulness` valutava la risposta solo contro `chunks[:5]`, mentre `build_rag_context` (che genera il contesto dato al LLM) ne include un numero variabile in base a `max_context_chars=12000` — le due funzioni non condividevano la stessa nozione di "cosa ha visto davvero il LLM".
   **Fix**: `check_faithfulness` ora accetta direttamente `context: str` (non più `chunks`) e viene invocata con lo stesso `ctx["context"]` prodotto da `build_rag_context` e realmente passato al LLM — sia nel path non-streaming (`arun_rag_chain` ora restituisce anche `"context"` nel suo dict) sia in streaming (`astream_rag_chain` lo espone tramite l'ultimo yield `("final", {...})`, vedi punto 9). Aggiornati tutti i chiamanti: `chat_service.py`, il nodo LangGraph morto `nodes.py` (per coerenza, anche se irraggiungibile) e `scripts/benchmark_retrieval.py`.

7. ✅ **`hallucination_score` esposto in modo incoerente tra i due endpoint** — `POST /chat/query` lo calcolava ma `ChatResponse` non aveva il campo (scartato prima del client), mentre `POST /chat/stream` lo includeva.
   **Fix**: aggiunto `hallucination_score: float | None = None` a `ChatResponse` (`app/schemas/chat.py`) e al dict di risposta di `ChatService.query()` — ora esposto in modo identico su entrambi gli endpoint.

8. ✅ **`app/services/chat_service.py:222`** — la chat in streaming non tracciava mai token realmente consumati (`tokens_in`/`tokens_out` sempre 0), mentre `/chat/query` (canale non usato dall'unica UI del progetto) tracciava i valori reali.
   **Fix**: `astream_rag_chain` ora accumula i chunk della risposta LLM (`AIMessageChunk` concatenati) ed espone `usage_metadata` (quando il provider la fornisce in streaming, es. OpenAI/Google) tramite l'ultimo yield; `stream_query` passa questi valori reali a `_increment_usage_stats`/`_save_messages` invece di hardcoded `0, 0`. Con provider che non espongono usage in streaming (es. Ollama) restano 0 — non c'è modo di inventarli — ma non sono più *sempre* 0 a prescindere dal provider.

9. ✅ **`chat_service.py:93` + `chat.py:93`** — il protocollo streaming usava il carattere di controllo `\x1e` come sentinella non protetta nel testo per distinguere token di risposta da metadata JSON finale.
   **Fix**: `astream_rag_chain` e `ChatService.stream_query` ora sono generatori di tuple strutturate `(kind, payload)` — `("token", str)` per ogni pezzo di risposta, un singolo `("final"/"meta", dict)` finale — invece di un marker testuale nel contenuto. `chat.py` consuma `async for kind, payload in ...` e discrimina sul tag, non più su un prefisso di stringa: un token non può più essere scambiato per l'inizio dei metadata, qualunque carattere contenga. Il formato SSE effettivamente inviato al client (`data: {...}\n\n`) resta invariato — la modifica è solo nel canale interno Python tra `chat_service.py` e `chat.py`, nessun impatto su `chainlit_app/app.py`.

10. ✅ **`app/core/settings.py:81`** — `retriever_search_type` (default `"hybrid"`) non era mai letto da `retriever.py`; solo `qdrant_use_sparse` controllava dense-vs-hybrid.
    **Fix**: `retrieve()` ora usa `settings.retriever_search_type` per decidere quali rami eseguire a runtime — `"dense"` salta del tutto la ricerca sparse, `"sparse"` salta la ricerca dense, `"hybrid"` (default, comportamento invariato) esegue entrambe. `qdrant_use_sparse` resta il gate su "la collection HA vettori sparse indicizzati" (necessario perché il ramo sparse possa funzionare), `retriever_search_type` è ora la scelta di strategia a runtime, entrambi coerenti tra loro.

---

# Giro 5 — Nuovo audit completo del codebase — ✅ TUTTI CORRETTI

Ricontrollo a 360° dell'intero codebase applicativo (file Docker esclusi — già coperti a fondo nel Giro 3, invariati da allora). Include un self-review dei fix applicati nei Giri 1/2/4 per verificare che non abbiano introdotto regressioni. Su richiesta esplicita: file mai usati e file `OLD*` non sono considerati errori e non vengono qui riportati.

## Bug preesistenti non ancora notati

1. ✅ **`app/core/redis_client.py:116-131` (`TenantRedis.check_rate_limit`) — il contatore di rate limit non si resettava mai sotto traffico continuativo, causando un blocco 429 permanente per client legittimi.** La pipeline faceva `pipe.incr(key)` seguito INCONDIZIONATAMENTE da `pipe.expire(key, window_seconds)` ad ogni singola richiesta, non solo alla creazione della chiave — la TTL non scadeva mai e `count` cresceva senza limiti, bloccando per sempre l'utente una volta superata la soglia.
   **Fix**: `EXPIRE` ora viene impostato solo quando `INCR` ritorna `1` (prima richiesta della finestra) — reset a finestra fissa corretto, invece della pipeline incondizionata.

2. ✅ **`app/workers/cleanup_tasks.py:33-40` (`purge_tenant`) — `DROP SCHEMA IF EXISTS` falliva sempre, perché lo schema tenant non è mai vuoto quando la purge viene invocata** (SQL Server non ha un `DROP SCHEMA ... CASCADE`, e lo schema contiene sempre le tabelle create da `shared.sp_provision_tenant`). Il task cancellava prima Qdrant/Redis (irreversibile), poi crashava sul DROP SCHEMA prima del commit finale — il tenant restava "attivo" in SQL con tutti i dati, avendo già perso vettori e sessioni.
   **Fix**: prima del `DROP SCHEMA`, il task ora droppa dinamicamente tutte le tabelle dello schema (via `sys.tables`, stesso pattern di SQL dinamico già usato in `sp_provision_tenant`; nessuna FK tra le tabelle di uno schema tenant, verificato in `init.sql`) e l'utente dedicato `usr_<schema>`, così il `DROP SCHEMA IF EXISTS` finale trova lo schema vuoto e riesce. Corretta anche la nota minore collegata: `schema_name` ora applica `.lower()`, coerente con `_slug_to_schema()` in `app/db/sqlserver.py`.

## Regressioni dai fix precedenti

Nessuna trovata. Rilette con attenzione le modifiche dei Giri 1/2/4 (`tenant.py`, `auth.py::refresh_token`, `main.py` exception handler, `collections.py` cascata, `parser.py`/`chunker.py`/`metadata.py`/`pipeline.py`, `ingestion_tasks.py::reprocess_document`, `hallucination.py`/`chain.py`/`chat_service.py`/`chat.py`/`schemas/chat.py`, `retriever.py`, `settings.py`/`observability.py`, `graph/state.py`+`nodes.py`) — nessun nuovo bug introdotto: `chainlit_app/app.py` resta compatibile con il nuovo protocollo streaming interno (il formato SSE sul wire non è cambiato), tutti i chiamanti di `check_faithfulness`/`build_chunk_metadata`/`classify_document` sono coerenti con le nuove firme, e la cascata in `delete_collection` segue lo stesso pattern di commit già usato altrove.

---

# Giro 6 — Nuovo audit completo del codebase (incl. ottimizzazioni ingestion/chat della sessione precedente)

Richiesto: "analizza tutto il code, cerca bugs/incosistenze". Ricontrollato l'intero codebase applicativo, con focus aggiuntivo su `chat_service.py`/`document_service.py`/`vectorstore.py`/`pipeline.py` dopo le modifiche di ottimizzazione performance della sessione precedente (fix countdown Celery, cache `ensure_collection`, parallelizzazione embedding, hallucination check non bloccante — vedi `README5.md`). File Docker esclusi (già coperti a fondo nei Giri 3/5).

> **Aggiornamento**: su richiesta esplicita, i punti 1-5 (bug e inconsistenze veri) sono stati corretti nel codice — dettaglio del fix sotto ogni punto. I punti 6-12 (codice morto: `reprocess_document` irraggiungibile, import inutilizzati, parametro `stream` morto) sono stati **lasciati invariati**, di nuovo su richiesta esplicita ("lascia così com'è il codice morto, intanto non lo utilizzo").

## BUG — Scrittura non protetta per proprietà (IDOR)

1. ✅ **CRITICO — `_save_messages` (`app/services/chat_service.py:313-363`) scrive su un `conversation_id` fornito dal client senza verificare che appartenga all'utente che chiama.** `ChatRequest.conversation_id` (`app/schemas/chat.py:8`) è un campo libero accettato da `POST /chat/query` e `POST /chat/stream`. A differenza di `get_history` (righe 259-310), che filtra correttamente con `JOIN conversations c ON ... WHERE c.user_id = :user_id`, sia `query()` che `stream_query()` passavano il `conversation_id` ricevuto direttamente a `_save_messages` e a `self.redis.append_message`/`get_session` **senza controllare che quella conversazione appartenga a `self.user_id`**. La query `IF NOT EXISTS (SELECT 1 FROM conversations WHERE id = :id) INSERT ... ELSE UPDATE` (righe 328-336) eseguiva l'`UPDATE`/insert dei messaggi su QUALSIASI conversazione esistente nel tenant, indipendentemente dal proprietario.
   - **Impatto pratico**: un utente autenticato dello stesso tenant (o chi usa `X-API-Key`, che ha `user_id` sintetico) poteva indovinare/riusare un `conversation_id` (UUID) di un altro utente e: (a) far apparire proprie domande/risposte nella cronologia altrui (visibile alla vittima al successivo `GET /chat/history`, che *quello* sì era protetto in lettura); (b) inquinare la memoria a breve termine condivisa su Redis (`TenantRedis` è scoped solo per `tenant_id`, non per `user_id` — righe 63-64 in `redis_client.py`), influenzando le risposte del LLM nei turni successivi della vittima con contenuto iniettato da un terzo.
   - **Fix**: aggiunto `ChatService._check_conversation_ownership(conv_id)` (`chat_service.py`), invocato in `query()` e `stream_query()` solo quando `conversation_id` è esplicitamente fornito dal client (una nuova conversazione generata server-side non ha bisogno del check). Verifica `SELECT user_id FROM conversations WHERE id = :id`: se la conversazione esiste e appartiene a un altro utente, solleva `PermissionError`. In `chat.py::chat_query` (endpoint non-streaming) `PermissionError` viene mappata a `HTTPException(403)`; in `chat_stream` non serve una modifica dedicata, il generatore esistente cattura già qualunque `Exception` e la espone come evento SSE `{"error": ...}` prima che venga scritto alcun token.

## BUG — Race condition (TOCTOU)

2. ✅ **`document_service.py:39-46` — il controllo anti-duplicato su `file_hash` era check-then-insert senza vincolo UNIQUE a livello DB.** `docker/sqlserver/init.sql:238-243` crea solo un indice normale (`IX_..._doc_hash`), non un `UNIQUE INDEX`/constraint. Due upload concorrenti dello stesso file (stesso tenant) potevano entrambi superare la `SELECT ... WHERE file_hash = :hash` prima che il primo avesse fatto commit, risultando in due righe `documents` identiche e due ingestion complete duplicate (doppio consumo GPU/embedding, doppi chunk indicizzati su Qdrant per lo stesso contenuto, doppie fonti citate in chat).
   - **Fix**: aggiunto l'hint `WITH (UPDLOCK, HOLDLOCK)` alla `SELECT` di controllo in `document_service.py` — non richiede toccare lo schema/Docker: il lock resta attivo fino al commit/rollback della stessa transazione (che avviene poche righe dopo, subito dopo l'`INSERT`), serializzando upload concorrenti dello stesso `file_hash` per lo stesso tenant invece di lasciarli correre entrambi in parallelo sullo stesso controllo.

3. ✅ **`documents.py::delete_document` (righe 130-171) non considerava lo stato del documento né annullava un'ingestion in corso.** Se un documento veniva cancellato mentre `status = 'processing'` (worker Celery ancora al lavoro su di esso), la sequenza era: (a) l'endpoint cancella i vettori Qdrant e marca `status = 'deleted'`; (b) il worker, ignaro, termina la pipeline e fa `client.upsert(...)` reinserendo i vettori appena cancellati, poi esegue `UPDATE documents SET status = 'ready', ...` (`ingestion_tasks.py`) **sovrascrivendo `'deleted'` con `'ready'`** — il documento "resuscitava" pienamente ricercabile dopo essere stato esplicitamente cancellato.
   - **Fix (due parti)**: (a) `documents.py::delete_document` ora, se `status IN ('pending','processing')`, cerca il `celery_task_id` più recente in `ingestion_jobs` e chiama `celery_app.control.revoke(..., terminate=False)` prima di cancellare i vettori — best-effort, blocca i task non ancora partiti; (b) `ingestion_tasks.py::ingest_document`, l'`UPDATE documents SET status='ready', ...` finale ora ha `AND status != 'deleted'` — se il documento è stato cancellato mentre il task era già in esecuzione (revoke troppo tardivo per fermarlo), l'update non tocca la riga (`rowcount == 0`) e il task ripulisce da solo i vettori appena inseriti su Qdrant invece di lasciare il documento "resuscitato" e ricercabile. Copre entrambe le finestre di race, non solo la più comune.

## INCONSISTENZE

4. ✅ **`documents.py::delete_document` non cancellava mai il file fisico su disco** (`documents.storage_path`). **Fix**: dopo l'`UPDATE ... status = 'deleted'`, se `storage_path` è valorizzato, il file viene rimosso (`Path(...).unlink(missing_ok=True)`), con `try/except OSError` che logga un warning senza far fallire la richiesta se la cancellazione fisica non riesce (lo stato applicativo — DB + Qdrant — resta comunque coerente).

5. ✅ **`ensure_collection()` (`app/core/vectorstore.py`) non verificava che la dimensione della collection Qdrant esistente corrispondesse a `get_embedding_dimension()` del modello embedding attualmente in `settings.embeddings_model`.** **Fix**: nuova `_validate_collection_dimension(collection_name, existing)`, chiamata quando la collection risulta già esistente (prima di aggiungerla a `_known_collections`): confronta `existing.config.params.vectors["dense"].size` con `get_embedding_dimension()` e solleva un `ValueError` esplicito in caso di mismatch, invece di lasciar fallire con un errore Qdrant poco chiaro al primo `upsert`. La lettura della struttura è avvolta in `try/except (AttributeError, KeyError, TypeError): return` — se la forma esatta dell'oggetto `CollectionInfo` differisse per qualche versione di `qdrant-client`, il controllo si disattiva silenziosamente invece di rompere `ensure_collection` per tutti (fail-open, nessuna regressione possibile su questo punto).

6. ⬜ **`app/workers/ingestion_tasks.py::reprocess_document` resta non raggiungibile da alcun endpoint API** (confermato di nuovo in questo giro: nessuna route in `app/api/routes/` lo invoca). Coerente con quanto già notato al Giro 4 punto 4 (lì corretto solo l'inserimento della riga `ingestion_jobs` mancante) — resta comunque codice funzionalmente morto dal punto di vista dell'utente finale, stesso pattern del grafo LangGraph (Giro 1 punto 9).

## CODICE MORTO / IMPORT INUTILIZZATI (segnalati dall'analisi statica, pre-esistenti)

Non introdotti dalle ottimizzazioni della sessione precedente — erano già presenti nei file toccati, semplicemente non ancora notati:

7. ⬜ `app/rag/ingestion/pipeline.py:2` — `import hashlib` mai usato nel modulo.
8. ⬜ `app/rag/ingestion/pipeline.py:12` — `get_collection_name` importato ma mai chiamato (il nome collection si ottiene sempre tramite `ensure_collection(...)`, che lo restituisce già).
9. ⬜ `app/services/document_service.py:3-4` — `import os` e `import shutil` mai usati.
10. ⬜ `app/services/chat_service.py:10` — `AsyncSession` importato ma mai usato (il parametro `db: AsyncSession` del costruttore è commentato, righe 30-31/37).
11. ⬜ `app/services/chat_service.py:50` — il parametro `stream: bool = False` di `ChatService.query()` non viene mai letto nel corpo del metodo, né alcun chiamante lo valorizza (`chat.py:39` chiama `service.query(...)` senza `stream=`) — parametro morto/fuorviante: lascia intendere che `query()` possa comportarsi diversamente in streaming, ma lo streaming reale passa sempre da `stream_query()`, un metodo separato.
12. ⬜ `app/core/vectorstore.py:57` — `existing = client.get_collection(collection_name)` assegnata ma mai letta (collegato al punto 5 sopra: sarebbe il punto naturale per confrontare `existing`'s vector size con `get_embedding_dimension()`).

## Regressioni dai fix di performance della sessione precedente

Nessuna trovata. Riletti con attenzione tutti i file toccati (`document_service.py`, `vectorstore.py`, `pipeline.py`, `chat_service.py`, `settings.py`, `config/config.yaml`): la nuova sequenza commit-prima-di-`apply_async`, la cache `_known_collections`, la parallelizzazione denso/sparso via `ThreadPoolExecutor` e l'`asyncio.create_task` per l'hallucination check in background sono tutte corrette e coerenti con il resto del codice — i problemi elencati sopra (punti 1-6) erano già presenti prima di quelle modifiche, non introdotti da esse.


