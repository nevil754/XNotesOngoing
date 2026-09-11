# Split GPU / no-GPU su due server Docker separati

> Obiettivo: capire cosa nel codebase usa davvero la GPU, come far comunicare due `docker compose` che girano su due Docker engine diversi (quindi senza network Docker condivisa), e quanto (poco) serve toccare il codice.

---

## 1. Cosa gira dove OGGI (1 solo server, no GPU)

Dal `docker-compose.yml` attuale, un solo Docker engine ospita tutto:

- `fastapi` — API, embedding/rerank **a query-time**
- `celery-worker-high` / `celery-worker-default` — ingestion documenti, embedding **a ingestion-time**
- `celery-beat`, `flower` — scheduler + monitoring, ininfluenti per la GPU
- `qdrant`, `redis`, `sqlserver` — storage/infra

Il **LLM (Ollama) è GIÀ esterno** e chiamato via HTTP: vedi `.env` → `BASE_URL_OLLAMA=http://srvai2k22:11434/`, usato da `app/core/llm_factory.py` (`ChatOllama(base_url=...)`). Questo probabilmente È il secondo server che ti hanno dato in SSH: quindi il pattern "container che chiama un endpoint HTTP su un altro host invece che un nome-servizio Docker" **esiste già** nel progetto, non lo stai inventando ora.

## 2. Cosa userebbe davvero la GPU

Cercando nel codice, la parte "AI" pesante non è solo l'LLM:

| Componente | File | Dove gira oggi | Tipo |
|---|---|---|---|
| LLM (Ollama) | `app/core/llm_factory.py` | **già esterno**, via HTTP | già a posto |
| Embedding denso (fastembed/ONNX) | `app/core/embeddings.py::get_embedding_model/embed_texts/embed_query` | **in-process** dentro `fastapi` (query-time) e dentro i `celery-worker-*` (ingestion-time) | libreria locale, non HTTP |
| Embedding sparso (SPLADE) | `app/core/embeddings.py::get_sparse_model` | idem | idem |
| Reranker (CrossEncoder, `sentence_transformers`) | `app/core/embeddings.py::get_reranker_model`, usato da `app/rag/retrieval/retriever.py` | **in-process** dentro `fastapi` (query-time, in `retriever.py`) e potenzialmente nei worker se richiamato in ingestion | idem |

Punto chiave: embedding e reranker **non sono un servizio HTTP**, sono librerie Python (`fastembed`, `sentence_transformers`) che girano dentro il processo di `fastapi` e dei `celery-worker-*`. Per beneficiare della GPU, deve essere il *processo stesso* (container `fastapi` e container `celery-worker-*`) a girare fisicamente sulla macchina con GPU — non basta "chiamare un endpoint".

### 2.1 Attenzione: "girare sulla macchina GPU" da solo non basta per fastembed

C'è un'asimmetria importante tra le due librerie, e vale la pena saperla PRIMA di mettersi al lavoro:

- **`sentence_transformers.CrossEncoder`** (reranker) **si auto-rileva la GPU**: internamente controlla `torch.cuda.is_available()` e ci gira sopra da solo, a patto che il `torch` installato sia una build CUDA. → **zero modifiche a `embeddings.py`** per il reranker.
- **`fastembed`** (embedding denso + sparso) gira su **ONNX Runtime**, che **non si auto-rileva niente**: se non gli dici esplicitamente quale "execution provider" usare, resta su `CPUExecutionProvider` anche se hai installato `onnxruntime-gpu` e la GPU è lì fisicamente e libera. Serve passare `cuda=True` (parametro già supportato da `fastembed.TextEmbedding`/`SparseTextEmbedding`) in `get_embedding_model()` e `get_sparse_model()`.

Oggi inoltre `requirements.txt` pinna esplicitamente le build **CPU**: `--extra-index-url https://download.pytorch.org/whl/cpu`, `torch==2.13.0+cpu`, `torchvision==0.28.0+cpu`, `onnxruntime==1.28.0` (non `-gpu`). Quindi sul server GPU non basta "gli gira sopra" — servono pacchetti diversi (§6) **più** un piccolo parametro esplicito in `embeddings.py` per fastembed. Non è "zero codice" in senso stretto, ma è un cambiamento di poche righe, mirato, opzionale via env var (così il server CPU non cambia comportamento):

```python
# app/core/embeddings.py
@lru_cache(maxsize=1)
def get_embedding_model() -> Any:
    from app.core.settings import get_settings
    settings = get_settings()
    from fastembed import TextEmbedding
    model = TextEmbedding(
        model_name=settings.embeddings_model,
        cache_dir=settings.embeddings_cache_dir,
        max_length=512,
        threads=4,
        cuda=settings.embeddings_use_cuda,                              # nuovo
        device_ids=[0] if settings.embeddings_use_cuda else None,       # nuovo
    )
    return model

@lru_cache(maxsize=1)
def get_sparse_model() -> Any:
    from app.core.settings import get_settings
    settings = get_settings()
    from fastembed import SparseTextEmbedding
    return SparseTextEmbedding(
        model_name="prithivida/Splade_PP_en_v1",
        cuda=settings.embeddings_use_cuda,                              # nuovo
    )
```

E in `app/core/settings.py`, accanto alle altre `embeddings_*`:
```python
embeddings_use_cuda: bool = False   # True solo nel .env del server GPU
```

Default `False` → sul server CPU (o se dimentichi di settarla) il comportamento resta identico a oggi.

Nota anche: in `app/core/settings.py` esistono già `embeddings_provider` ed `embeddings_base_url`, ma **non sono collegati a nessuna logica reale** in `embeddings.py` (sono placeholder morti, coerente con quanto già segnalato in `README2.md` sulle chiavi di config inefficaci). Quindi oggi non esiste una modalità "embedding via HTTP remoto" già pronta all'uso.

## 3. Due strategie possibili

### Strategia A — Spostare i container "AI" sul server GPU così come sono (consigliata)
Sposti `fastapi` + `celery-worker-high` + `celery-worker-default` sul server GPU, **stesso codice Python, modifiche minime e mirate in `app/`**. Cambiano: quale Dockerfile li builda (per installare le versioni GPU delle librerie), a quali host puntano per DB/cache/vector-store (via env var, già parametriche oggi), e le poche righe in `embeddings.py`/`settings.py` viste in §2.1 per dire esplicitamente a `fastembed` di usare CUDA.

- Codice Python: **poche righe** (il flag `cuda=` in `embeddings.py`, dietro una nuova env var — il reranker non richiede nulla, si auto-rileva)
- Nuovi file: 2 Dockerfile "gpu" (additivi, non toccano quelli esistenti) + eventualmente `requirements-gpu.txt`
- `docker-compose.yml`: poche righe (profiles + blocco `deploy.resources` per la GPU)

### Strategia B — Trasformare embedding/reranker in un microservizio HTTP (come già fatto per Ollama)
Crei sul server GPU un servizietto (es. FastAPI minimale, oppure Hugging Face **Text Embeddings Inference**) che espone `/embed` e `/rerank`, e modifichi `app/core/embeddings.py` per chiamarlo via HTTP invece di caricare i modelli in-process — esattamente come fa già `llm_factory.py` con Ollama.

- Codice Python: **modifiche reali** a `embeddings.py` (e uso di `embeddings_base_url` che oggi è morto)
- Più "pulito" architetturalmente (solo 2 container GPU-only sul secondo server, `fastapi`/`celery` restano dov'erano)
- Più lavoro adesso, migliore se in futuro vuoi scalare i worker CPU indipendentemente da quelli GPU

**Dato che vuoi toccare il meno possibile il codice: vai con la Strategia A.** La B resta un'evoluzione naturale se in futuro ti serve scalare orizzontalmente (§8).

---

## 4. Architettura risultante (Strategia A)

**Server 1 — attuale, NO GPU → "infra/CPU server"**
```
sqlserver   (dati persistenti, resta qui — è dove hai già i volumi in DATA_ROOT)
redis       (broker Celery + cache)
qdrant      (vector store — CPU va benissimo per l'indice, non serve GPU)
celery-beat (scheduler, carico trascurabile)
flower      (monitoring, trascurabile)
```

**Server 2 — nuovo, HA la GPU → "AI server"**
```
fastapi              (API + embedding/rerank a query-time)
celery-worker-high   (ingestion + embedding pesante)
celery-worker-default
(ollama, se non è già lì come servizio nativo/esterno che già usi via srvai2k22)
```

Nota: se vuoi restare più conservativo al primo giro, puoi lasciare `celery-worker-default` (coda "default,low", concurrency bassa) sul server 1 in CPU, e spostare sul server GPU solo `fastapi` + `celery-worker-high` (che fa la maggior parte dell'embedding pesante). Scelta puramente di composizione, i meccanismi sotto sono identici.

## 5. Networking: niente rete Docker condivisa → si parla via IP/porte

Due `docker compose` su due Docker engine diversi **non possono** condividere una bridge network Docker (`rag-network` su un host è isolata da quella sull'altro, anche se le chiami uguale). Quindi:

- I nomi-servizio Docker (`sqlserver`, `redis`, `qdrant`) funzionano **solo dentro lo stesso engine**.
- Dall'engine del server GPU, per raggiungere `sqlserver`/`redis`/`qdrant` sul server 1 devi usare **IP o hostname raggiungibile via rete + porta esposta sull'host**, non più il nome del container.

Il bello è che il codice **è già pronto per questo**: `SQLSERVER_HOST`, `REDIS_URL`/`CELERY_BROKER_URL`, `QDRANT_URL` sono già letti da env var (vedi `docker-compose.yml` righe `environment:` e `.env.example`), esattamente come già succede per `BASE_URL_OLLAMA`. Basta cambiarne il **valore** nel `.env` del server GPU.

### Cosa cambia lato server 1 (esposizione porte)
Oggi `redis` e `sqlserver` sono bindati solo su `127.0.0.1` (raggiungibili solo da chi è già sull'host):
```yaml
redis:
  ports:
    - "127.0.0.1:6379:6379"
sqlserver:
  ports:
    - "127.0.0.1:1433:1433"
```
Per farli raggiungere dal server 2 devi bindarli su un'interfaccia raggiungibile dall'altro server (es. IP privato/VPN) e **non** su `0.0.0.0` aperto a tutta internet, perché sono servizi con solo password (nessun TLS) — un'esposizione pubblica è un rischio serio. `qdrant` è già su tutte le interfacce (`6333:6333`, `6334:6334`), stesso discorso: va ristretto via firewall.

### Come collegare i due server in sicurezza (scegli una)
1. **VPN privata tra i due host** (consigliato): WireGuard o Tailscale. Ottieni un IP privato stabile per ciascun server (es. `10.x.x.1` e `10.x.x.2`) e nel `.env` del server GPU punti a quell'IP. Il firewall dei due host resta chiuso verso internet, aperto solo sull'interfaccia VPN.
2. **Firewall a whitelist per IP**: se i due server sono già su una rete che li fa comunicare (stesso datacenter/VPC), basta `ufw`/security group che apre le porte 6379/1433/6333/6334 **solo** dall'IP del server 2, bindando comunque i container sull'interfaccia giusta (non `127.0.0.1`).
3. **Tunnel SSH** (dato che hai già accesso SSH al server GPU): `autossh` con port-forward locale dal server 2 verso il server 1, così sul server GPU i servizi restano raggiungibili su `localhost:<porta-forwardata>` e nel `.env` usi `localhost` invece dell'IP pubblico. Più semplice da mettere su subito, meno robusto in caso di caduta del tunnel (mitigabile con `autossh` + restart policy).

Qualunque opzione tu scelga, è pura configurazione di rete/`.env` — **zero codice**.

## 6. Cosa tocchi concretamente

### `docker-compose.yml` (uno solo, condiviso via git tra i due server) — uso dei `profiles`
Aggiungi un `profiles:` alle sezioni che vuoi far girare solo su un host, così lo stesso file resta identico ovunque e decidi cosa avviare col flag `--profile` al momento dello start:

```yaml
services:
  fastapi:
    profiles: ["gpu"]
    build:
      context: .
      dockerfile: docker/fastapi-gpu.Dockerfile   # nuovo file, additivo
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    depends_on: []   # IMPORTANTE: vedi nota subito sotto la tabella
    environment:
      SQLSERVER_HOST: "10.x.x.1"      # IP/hostname del server 1 (o VPN/tunnel, vedi §5)
      QDRANT_URL: "http://10.x.x.1:6333"
    # REDIS_URL/CELERY_BROKER_URL: già presi da .env del server 2, aggiornali lì

  celery-worker-high:
    profiles: ["gpu"]
    build:
      dockerfile: docker/celery-gpu.Dockerfile    # nuovo file, additivo
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    depends_on: []   # idem
    environment:
      SQLSERVER_HOST: "10.x.x.1"
      QDRANT_URL: "http://10.x.x.1:6333"

  celery-worker-default:
    profiles: ["gpu"]     # o lascialo senza profilo se lo tieni sul server 1, vedi §4



  sqlserver:
    profiles: ["infra"]  #sta x 'infrastructure' cioe i servizi base (database, cache, vector store, scheduler, ...)
    ports:
      - "10.x.x.1:1433:1433"   # invece di 127.0.0.1, bind sull'interfaccia verso il server 2

  redis:
    profiles: ["infra"]
    ports:
      - "10.x.x.1:6379:6379"

  qdrant:
    profiles: ["infra"]

  celery-beat:
    profiles: ["infra"]

  flower:
    profiles: ["infra"]
```

> **`depends_on: []` non è cosmetico, è necessario.** Oggi `fastapi`/`celery-worker-*` hanno `depends_on: sqlserver (condition: service_healthy)` + `redis`/`qdrant`. Quando `sqlserver`/`redis`/`qdrant` finiscono nel profilo `infra` e sul server GPU avvii solo `--profile gpu`, quei servizi **non esistono in quel `docker compose up`**: Compose non riesce a valutare una `depends_on: condition: service_healthy` su un servizio che non fa parte del set di profili attivi, e si rifiuta di partire. Vanno quindi rimosse (o svuotate) le `depends_on` verso servizi che ora vivono sull'altro host — `depends_on` funziona solo *dentro lo stesso* `docker compose up`, mai tra due engine diversi. L'ordinamento "aspetta che il DB sia pronto" lo fa comunque già l'app stessa: `main.py::_check_services()` all'avvio pinga Redis/SQL Server/Qdrant e logga solo un warning se non risponde ancora (non blocca lo startup), e i pool SQLAlchemy/redis hanno `pool_pre_ping`/`retry_on_timeout`. Con `restart: unless-stopped` già impostato, se qualcosa si connette troppo presto il container si riprende da solo al retry successivo.

Avvio:
```bash
# server 1 (no GPU)
docker compose --profile infra up -d

# server 2 (GPU)
docker compose --profile gpu up -d
```

Ogni `.env` resta **specifico per server** (non committato, come già fai oggi — `.env.example` è il template): sul server 2 punti `SQLSERVER_HOST`/`QDRANT_URL`/`REDIS_URL`/`CELERY_BROKER_URL`/`CELERY_RESULT_BACKEND` all'host raggiungibile del server 1, e aggiungi `EMBEDDINGS_USE_CUDA=true` (§2.1); sul server 1 non cambia nulla rispetto ad oggi.

### 2 nuovi Dockerfile GPU (additivi — non toccano `docker/fastapi.Dockerfile` e `docker/celery.Dockerfile` esistenti)
`docker/fastapi-gpu.Dockerfile` e `docker/celery-gpu.Dockerfile`: stessa struttura di quelli attuali, ma:
- immagine base con supporto CUDA (es. `nvidia/cuda:12.x-runtime-ubuntu22.04` + Python, oppure resti su `python:3.11-slim` e installi solo le librerie CUDA-enabled se ti bastano)
- `requirements-gpu.txt` (nuovo file, non tocca `requirements.txt`) con `onnxruntime-gpu` al posto di `onnxruntime` e `torch` con build CUDA al posto della build CPU (da cui dipende `sentence-transformers`/sua `CrossEncoder`)
- stesso `COPY app/ ./app/`, stesso entrypoint: **il codice Python copiato dentro l'immagine è letteralmente lo stesso**, cambia solo cosa c'è nel virtualenv sotto.

Sul server GPU serve inoltre installare **NVIDIA Container Toolkit** (`nvidia-ctk`) perché Docker possa esporre la GPU ai container — è un setup one-time a livello di host, non di questo repo.

## 7. "Devo duplicare il codice nei 2 server?"

No, non nel senso di biforcarlo. Il **repo Git è uno solo**: lo cloni (o `git pull`) anche sul server GPU, esattamente come fai oggi sul server 1 — è la normale prassi di un deploy multi-host, non una duplicazione logica. Le cose realmente nuove che vivono nel repo sono i 2 Dockerfile GPU, l'eventuale `requirements-gpu.txt`, e le poche righe di §2.1 in `embeddings.py`/`settings.py` (dietro flag `EMBEDDINGS_USE_CUDA`, quindi innocue sul server CPU): il resto del codice in `app/` resta unico e identico su entrambi gli host — nessun fork, nessuna versione parallela.

Se in futuro vuoi automatizzare, un piccolo script di deploy (o CI) che fa `git pull && docker compose --profile gpu up -d --build` sul server 2 ti evita di doverlo fare a mano via SSH ogni volta — ma è un'ottimizzazione successiva, non un prerequisito.

## 8. Se in futuro vuoi la Strategia B

Quando/se vorrai scalare i worker CPU (parsing, chunking, I/O) indipendentemente dalla capacità GPU (embedding/rerank), la strada pulita è trasformare embedding+reranker in un servizio HTTP dedicato sul server GPU (stesso pattern già usato per Ollama), e modificare `embed_texts`/`embed_query`/`get_reranker_model` in `app/core/embeddings.py` per chiamarlo via HTTP invece che in-process, riusando i campi `embeddings_provider`/`embeddings_base_url` già presenti (ma oggi morti) in `app/core/settings.py`. A quel punto `fastapi` e i `celery-worker-*` potrebbero restare anche loro sul server 1, e solo il piccolo servizio di inferenza girerebbe sul server GPU. Non necessario ora, utile da tenere a mente come step successivo.

## 9. Verifica del flow di chat AI end-to-end

Ho ripercorso tutta la catena reale di una richiesta di chat (`app/api/routes/chat.py` → `ChatService.query`/`stream_query` → `retrieve()` → `arun_rag_chain`/`astream_rag_chain` → `check_faithfulness` → salvataggio messaggi), per essere sicuro che con lo split su 2 server non si rompa nulla di nascosto oltre a quanto già coperto.

**Tutte le dipendenze esterne toccate dalla chat, e dove sono gestite:**

| Passo | File | Dipendenza esterna | Coperta da |
|---|---|---|---|
| Cache/sessione conversazione | `chat_service.py` → `TenantRedis` | Redis (db 0 + db 1) | `REDIS_URL`/`REDIS_CACHE_URL` in `.env`, §5 |
| Embedding query (dense + sparse) | `retriever.py` → `aembed_query`/`aembed_sparse_query` | fastembed in-process | §2.1 (flag `cuda=`) |
| Ricerca vettoriale | `retriever.py` → `get_async_qdrant_client()` | Qdrant | `QDRANT_URL` in `.env`, §5 |
| Reranking | `retriever.py::_cross_encoder_rerank` → `get_reranker_model()` | sentence-transformers in-process | auto-rilevamento GPU, nessuna modifica (§2.1) |
| Generazione risposta | `chain.py::arun_rag_chain/astream_rag_chain` → `get_llm()` | Ollama via HTTP | **già esterno oggi** (`BASE_URL_OLLAMA`), nessuna modifica |
| Check allucinazioni | `hallucination.py::check_faithfulness` → `get_llm()` | Ollama via HTTP (stessa chiamata di sopra, una seconda invocazione LLM) | idem, nessuna modifica |
| Salvataggio messaggi/conversazioni | `chat_service.py::_save_messages` → `tenant_db.aget_session()` | SQL Server (schema-per-tenant, `EXECUTE AS USER`, **una sola connection string**, non una per tenant) | `SQLSERVER_HOST` in `.env`, §5 |
| Statistiche uso token | `chat_service.py::_increment_usage_stats` | Redis | idem |

Non ho trovato altre sorprese: né host hardcoded fuori da quelli già mappati in §5, né un secondo Qdrant/Redis/SQL per-tenant nascosto altrove (`app/core/vectorstore.py`, `app/core/redis_client.py`, `app/db/sqlserver.py` usano tutti e soli i client singleton con `lru_cache`, letti da un'unica `Settings`). Il multi-tenant è realizzato con **uno schema SQL Server per tenant sulla stessa istanza/connessione**, non con un DB per tenant — quindi non hai N host da riconfigurare, solo i 3 di sempre (SQL Server, Redis, Qdrant).

**Un dettaglio operativo utile da sapere (non un problema, solo per non stupirsi nei log):** l'health-check Docker di `fastapi` chiama `/health`, che risponde sempre `200 ok` senza controllare Redis/SQL/Qdrant — quindi Docker segnerà il container "healthy" anche se il collegamento verso il server 1 non è ancora su. Il controllo reale è su `/ready` (usato solo se lo chiami tu o un load balancer esterno) e nei log di avvio (`main.py::_check_services()`), che loggano un warning — non un errore fatale — se Redis/SQL/Qdrant non rispondono ancora al boot. Utile in fase di test iniziale: se la chat non risponde, guarda prima i log di `fastapi` per `"non raggiungibile"` prima di sospettare un bug applicativo.

## 10. Checklist pratica per il primo giro

1. Sul server GPU: installare Docker Engine (se non c'è) + NVIDIA Container Toolkit, verificare `nvidia-smi` visibile dentro un container di test.
2. Decidere il canale di rete tra i due host (VPN/firewall/tunnel SSH, §5) e verificare che dal server 2 riesci a fare `curl`/`telnet` verso le porte 1433/6379/6333 del server 1.
3. Aprire/ribindare le porte di `sqlserver`/`redis`/`qdrant` sul server 1 solo verso l'IP del server 2 (mai `0.0.0.0` senza firewall).
4. Clonare il repo sul server 2, creare il suo `.env` (basato su `.env.example`) puntando `SQLSERVER_HOST`/`QDRANT_URL`/`REDIS_URL`/`CELERY_BROKER_URL`/`CELERY_RESULT_BACKEND` all'host/porta del server 1.
5. Aggiungere i `profiles` al `docker-compose.yml` come in §6 (**incluso svuotare le `depends_on` cross-host**, vedi nota in §6), creare `docker/fastapi-gpu.Dockerfile`, `docker/celery-gpu.Dockerfile`, `requirements-gpu.txt`; aggiungere `embeddings_use_cuda` in `settings.py` e il flag `cuda=`/`device_ids=` in `embeddings.py` come in §2.1.
6. `docker compose --profile infra up -d` sul server 1 (ridotto rispetto a oggi: niente più `fastapi`/`celery-worker-*` lì, se li sposti tutti).
7. `docker compose --profile gpu up -d --build` sul server 2, controllare i log di `fastapi` per confermare che l'embedding/reranker carica sul device CUDA (di solito loggano il provider ONNX/torch usato) e che `_check_services()` (§9) riporti Redis/SQL/Qdrant connessi, non solo warning.
8. Test end-to-end del flow di chat (§9): upload documento → ingestion sul server 2 → query in chat → risposta con fonti corrette → verifica in `nvidia-smi` sul server 2 che la GPU venga effettivamente utilizzata durante embedding/rerank (sia in ingestion che a query-time), e controllo su `GET /ready` che tutti e 3 i servizi esterni risultino `"ok"`.
