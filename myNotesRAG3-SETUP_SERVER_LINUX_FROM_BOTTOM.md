# Setup server Linux da zero — RAG Enterprise (CompeteSrl)

Runbook tecnico per configurare un server Linux pulito fino ad avere l'intero
stack Docker in esecuzione e verificato. Ogni sezione riporta il *perché* oltre
al comando, così i passi non sono riproducibili a memoria ma comprensibili.

Lo stack è composto da 9 servizi: `fastapi`, `chainlit`, `celery-worker-high`,
`celery-worker-default`, `celery-beat`, `flower`, `qdrant`, `redis`, `sqlserver`.
I quattro servizi Python "pesanti" (fastapi + i 3 celery) condividono la stessa
base `python:3.11-slim-bookworm` e lo stesso `requirements.txt`.

---

## 0. Prerequisiti e dimensionamento

### Sistema operativo
Ubuntu Server 24.04 LTS (o equivalente Debian). Le note sui path assumono questo.

### Risorse minime
- **RAM**: 8 GB sono il minimo praticabile. A riposo lo stack usa ~700 MB, ma
  sotto carico i celery worker caricano in memoria i modelli fastembed
  (BGE-M3 + SPLADE + reranker) e salgono a 1.5–2.5 GB complessivi. SQL Server
  cresce fino al suo `MSSQL_MEMORY_LIMIT_MB`.
- **Disco**: **48 GB non bastano.** È il singolo punto più critico di tutto il
  setup. Le 4 immagini pesanti, i loro layer intermedi, la build cache e i dati
  di SQL Server + Qdrant saturano rapidamente un disco da 48 GB, specialmente
  durante i rebuild. Prevedere **almeno 100 GB**, di più se sullo stesso host
  convivranno dev + staging + prod.

### Estendere il disco (se il VG ha spazio non allocato)
L'installer Ubuntu di default alloca solo metà del volume group alla LV di root.
Verificare:

```bash
lsblk
# se sda3 è ~100G ma ubuntu-lv monta solo ~49G, ci sono ~49G liberi nel VG
sudo vgs   # colonna VFree
```

Estendere a caldo, senza perdita dati:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
df -h /
```

Richiede `sudo`. Se l'utente operativo non ha privilegi di root, questa è una
richiesta da girare all'amministratore del server **prima** di iniziare: senza
spazio adeguato ogni build fallisce con `no space left on device` a metà
dell'export dei layer, lasciando snapshot orfani che bloccano i build successivi.

---

## 1. Utente operativo e permessi

Lo stack gira sotto un utente di servizio (es. `deploy`) che appartiene a un
gruppo condiviso (es. `srv-deploy`) e al gruppo `docker`.

```bash
# (da eseguire come amministratore con sudo)
sudo useradd -m -s /bin/bash deploy
sudo groupadd -f srv-deploy
sudo usermod -aG srv-deploy deploy
sudo usermod -aG docker deploy      # consente 'docker' senza sudo
```

**Nota sul modello di permessi**: l'utente operativo tipicamente **non ha sudo**.
Questo è accettabile per l'uso quotidiano (i comandi `docker` funzionano grazie
al gruppo `docker`), ma alcune operazioni una tantum richiedono root e vanno
delegate all'amministratore:
- `lvextend` / `resize2fs` (disco)
- `systemctl restart docker`
- `chown` verso UID che non appartengono all'utente
- `sysctl` (es. il warning `vm.overcommit_memory` di Redis)

Far uscire e rientrare l'utente perché l'appartenenza ai gruppi abbia effetto:

```bash
exit   # e riconnettersi via SSH
id     # verificare che compaiano 'docker' e 'srv-deploy'
```

---

## 2. Docker Engine e Compose

```bash
# (come amministratore)
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verifica (come utente operativo, senza sudo):

```bash
docker version
docker compose version
docker run --rm hello-world
```

---

## 3. Struttura delle directory

Due alberi separati: **codice** e **dati**. Il codice sta dove si fa il deploy;
i dati (bind mount persistenti) stanno sotto un `DATA_ROOT` dedicato.

```bash
# codice
sudo mkdir -p /srv/environments/dev
sudo chown -R deploy:srv-deploy /srv/environments
sudo chmod -R g+s /srv/environments      # setgid: i nuovi file ereditano il gruppo

# dati persistenti (DATA_ROOT)
sudo mkdir -p /srv/data/dev/rag-system
sudo chown -R deploy:srv-deploy /srv/data
sudo chmod -R g+s /srv/data
```

Il flag **setgid** (`g+s`) è importante: garantisce che tutto ciò che l'utente
`deploy` crea sotto questi path erediti il gruppo `srv-deploy`, così l'utente
resta owner e può gestire i permessi senza `chown` (che richiederebbe root).

Layout risultante:
- Codice: `/srv/environments/dev/rag-system`
- Dati:   `/srv/data/dev/rag-system` (valore di `DATA_ROOT` nel `.env`)

---

## 4. Rete Docker condivisa

Lo stack usa una rete esterna, così eventuali servizi ausiliari (es. la futura
UI admin ASP.NET) possono agganciarsi allo stesso network e risolvere i servizi
per nome DNS (`http://fastapi:8000`) senza esporre porte.

```bash
docker network create rag-dev-network
```

Il nome deve coincidere con la variabile `DOCKER_NETWORK` del `.env` e con il
blocco `networks:` del compose.

---

## 5. Clone del codice da Forgejo

```bash
cd /srv/environments/dev
git clone ssh://git@<forgejo-host>:222/CompeteSrl/rag-system.git
cd rag-system
git checkout develop
```

**Convenzione di deploy**: sul server si resta su `develop` e si aggiorna con
`git fetch origin && git reset --hard origin/develop`. Evitare `checkout` tra
branch sul server, perché i file generati a runtime (config Chainlit,
translations) nel working tree causano conflitti a ogni switch.

---

## 6. File di configurazione `.env`

Il `.env` **non è versionato** (è in `.gitignore`) e va creato a mano sul server.
Contiene tutte le variabili che il compose interpola. Chiavi essenziali:

```dotenv
# path dati persistenti
DATA_ROOT=/srv/data/dev/rag-system
DOCKER_NETWORK=rag-dev-network

# SQL Server
SQLSERVER_PASSWORD=<password_forte>        # 8+ char, 3 categorie su 4
                                           # (maiusc/minusc/numeri/simboli)
# Redis
REDIS_PASSWORD=<password>

# App
APP_DEBUG=true                             # in dev; abilita /docs
APP_ENVIRONMENT=development

# Chainlit
CHAINLIT_DEFAULT_TENANT=demo-corp
CHAINLIT_AUTH_SECRET=<generato>            # vedi sezione Chainlit

# integrazioni esterne
BASE_URL_OLLAMA=http://<host-ollama>:11434/
```

**Attenzione alla password SA**: SQL Server 2022 rifiuta di avviarsi se la
password non rispetta la policy (minimo 8 caratteri, almeno 3 categorie tra
maiuscole, minuscole, numeri, simboli). Una password debole causa un crash
all'avvio con "Password validation failed" nei log.

**Nota operativa ricorrente**: la shell interattiva **non** legge il `.env`.
Prima di usare variabili come `$DATA_ROOT` o `$SQLSERVER_PASSWORD` in comandi
manuali, caricarle:

```bash
set -a; source .env; set +a
echo "DATA_ROOT=$DATA_ROOT"   # deve stampare il path, non vuoto
```

Dimenticare questo passo fa "scivolare" i comandi su path sbagliati
(es. `chmod 777 /sqlserver` invece di `chmod 777 /srv/data/.../sqlserver`).

---

## 7. Lock delle dipendenze — SEMPRE su Linux

Questo è il secondo errore concettuale più costoso dopo il disco.

**Regola**: `requirements.txt` va compilato con `pip-compile` sulla **stessa
piattaforma di produzione** (Linux, `python:3.11-slim-bookworm`, Python 3.11.x).
`pip-compile` non produce un lock universale: risolve le dipendenze per l'OS su
cui gira. Un lock compilato su Windows:
- include pacchetti Windows-only (`pywin32`) → con `--no-deps` in Docker
  falliscono con `No matching distribution found`;
- **omette** pacchetti Linux-only che servirebbero a runtime;
- può risolvere versioni diverse per dipendenze condizionate alla piattaforma.

I file `.in` (scritti a mano) sono platform-agnostic e si editano ovunque, anche
su Windows. È il `.txt` generato che è vincolato al target.

### Rigenerare il lock in un container bookworm

```bash
cd /srv/environments/dev/rag-system

docker run --rm -v "$(pwd):/w" -w /w python:3.11-slim-bookworm bash -c "
  pip install pip-tools &&
  pip-compile requirements.in     --output-file requirements.txt     --resolver=backtracking --no-header --annotate &&
  pip-compile requirements-dev.in --output-file requirements-dev.txt --resolver=backtracking --no-header --annotate
"
```

### PyTorch CPU — eliminare le librerie CUDA
`torch` entra come dipendenza transitiva (via sentence-transformers, docling,
easyocr, ecc.) e nella build di default trascina **~15 GB di librerie CUDA**
(`nvidia-*`, `triton`, `cuda-*`) per immagine. Su un server **senza GPU**
(inferenza LLM delegata a Ollama esterno) sono peso morto: i modelli girano già
in CPU. Forzare la variante CPU in `requirements.in`:

```
--extra-index-url https://download.pytorch.org/whl/cpu
torch==2.13.0+cpu
torchvision==0.28.0+cpu
```

Usare le stesse versioni già risolte nel lock, col suffisso `+cpu`.

### Verifiche dopo ogni rigenerazione del lock

```bash
grep -iE 'nvidia|cuda-|triton' requirements.txt   # deve essere VUOTO
grep -i 'email-validator' requirements.txt        # presente (pydantic[email])
grep -i '^bcrypt' requirements.txt                # bcrypt==4.0.1 (vedi sotto)
head -3 requirements.txt                          # --extra-index-url cpu in cima
```

### Pin espliciti necessari
- `pydantic[email]==...` — `EmailStr` richiede `email-validator`, altrimenti
  l'app crasha all'import di `auth.py`.
- `bcrypt==4.0.1` — `passlib==1.7.4` è incompatibile con `bcrypt>=4.1`/`5.x`
  (errore `module 'bcrypt' has no attribute '__about__'` e fallimento di
  `verify()`, quindi login sempre rotto).

I file `.in`/`.txt` rigenerati vanno **committati** su Forgejo, altrimenti al
prossimo `git reset --hard` sul server tornano quelli vecchi.

---

## 8. Dockerfile — punti critici

### Repository Microsoft ODBC (fastapi + celery, stage builder)
Non modificare il file `prod.list` di Microsoft con `sed`: contiene già un blocco
di opzioni `[arch=... signed-by=...]` e aggiungerne un secondo produce
`Malformed entry (URI parse)`. Usare la repo esplicita con una sola chiave:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl gnupg2 ca-certificates unixodbc-dev build-essential \
    && mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://packages.microsoft.com/keys/microsoft.asc \
       | gpg --dearmor -o /etc/apt/keyrings/microsoft.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/microsoft.gpg] https://packages.microsoft.com/debian/12/prod bookworm main" \
       > /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update \
    && ACCEPT_EULA=Y apt-get install -y --no-install-recommends msodbcsql18 \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
```

### Base image — pinnare bookworm ovunque
Usare **sempre** `python:3.11-slim-bookworm`, mai `python:3.11-slim` (che ora
punta a Debian 13 trixie, dove `apt-key` non esiste più e la repo Microsoft
`debian/12` non è coerente). Pinnare `-bookworm` su **entrambi** gli stage
(builder e runtime) per evitare mismatch glibc quando si copiano le `.so` da uno
stage all'altro.

### Registrazione del driver ODBC nel runtime
Nello stage runtime, copiare **anche `/etc/odbcinst.ini`** dal builder, non solo
le librerie. Senza quel file unixODBC non sa mappare il nome
`ODBC Driver 18 for SQL Server` alla `.so`, e SQLAlchemy fallisce con
`Can't open lib '...' : file not found`.

```dockerfile
COPY --from=builder /usr/lib/x86_64-linux-gnu/libodbc* /usr/lib/x86_64-linux-gnu/
COPY --from=builder /opt/microsoft /opt/microsoft
COPY --from=builder /etc/odbcinst.ini /etc/odbcinst.ini        # <-- necessario
COPY --from=builder /usr/local/lib/python3.11 /usr/local/lib/python3.11
COPY --from=builder /usr/local/bin /usr/local/bin
```

### Chainlit
Il `chainlit_app/requirements.txt` è un file **separato**, gestito a mano (non
passa da pip-compile). Pinnare esplicitamente per riproducibilità:

```
chainlit==2.11.1
httpx>=0.27.0
```

Chainlit 2.x è compatibile con Pydantic 2.13; le versioni 1.x no
(`CodeSettings is not fully defined` / `rebuild_dataclass`).

---

## 9. Permessi dei bind mount (SQL Server)

SQL Server 2022 gira come utente non-root **`mssql` (UID 10001)** dentro il
container e deve scrivere in `/var/opt/mssql`, montato dal bind mount
`${DATA_ROOT}/sqlserver`. Un bind mount **preserva i permessi dell'host**: se la
directory non è scrivibile dall'UID 10001, SQL Server muore all'avvio con
`The system directory [/.system] could not be created ... Permission denied`.

Qdrant e Redis non hanno questo problema perché girano come root nei loro
container; SQL Server è l'unico servizio che pretende una data dir scrivibile da
un UID specifico.

```bash
set -a; source .env; set +a
mkdir -p "$DATA_ROOT/sqlserver"
chmod 777 "$DATA_ROOT/sqlserver"
ls -land "$DATA_ROOT/sqlserver"   # deve mostrare drwxrwxrwx
```

`chmod 777` è la via praticabile **senza sudo** (l'alternativa pulita è
`chown -R 10001:10001`, ma richiede root). Per un ambiente dev è adeguato.

Se la directory contiene residui di tentativi precedenti owned da UID 10001 che
`deploy` non riesce a cancellare, usare un container che gira come root:

```bash
docker run --rm -v "$DATA_ROOT/sqlserver:/data" busybox \
  sh -c "rm -rf /data/* /data/.[!.]*"
chmod 777 "$DATA_ROOT/sqlserver"
```

---

## 10. Script `entrypoint.sh` di SQL Server

Lo script di init viene montato come bind mount ed eseguito. Tre requisiti,
tutti già inciampati in passato:

### 10.1 Bit di esecuzione committato in Git
Il bind mount preserva i permessi dell'host; se lo script non è `+x`, il
container fallisce con `Permission denied` (exit 126) e va in restart loop. Il
bit va registrato **in Git** (mode 100755), altrimenti ogni `git reset --hard`
lo perde — critico perché lo sviluppo avviene su Windows, dove il bit eseguibile
Unix non esiste.

```bash
chmod +x docker/sqlserver/entrypoint.sh
git update-index --chmod=+x docker/sqlserver/entrypoint.sh
git ls-files -s docker/sqlserver/entrypoint.sh   # deve iniziare con 100755
git commit -m "fix: entrypoint.sh eseguibile" && git push
```

Cintura di sicurezza aggiuntiva: invocare lo script con l'interprete nel
`command` del compose (`bash /init/entrypoint.sh`), così parte anche senza bit
`+x`.

### 10.2 Niente CRLF
Script editati su Windows possono avere line ending CRLF, che rompono lo shebang
in modo criptico. Forzare LF e blindare con `.gitattributes`:

```bash
sed -i 's/\r$//' docker/sqlserver/entrypoint.sh
echo '*.sh text eol=lf' >> .gitattributes
```

### 10.3 Readiness robusta + retry
Lo script deve attendere che **tutti i database siano ONLINE**, non solo che il
socket risponda: al primo avvio (o dopo un upgrade di versione dell'immagine)
SQL Server annuncia "ready for client connections" mentre RAGChat è ancora in
recovery, e un `USE RAGChat` fallisce con `Msg 904 / errore 4060`.

Struttura consigliata dell'entrypoint (concettuale):

```bash
#!/bin/bash
set -e
# attende che nessun database sia in stato != ONLINE
until sqlcmd -S localhost -U SA -P "$SA_PASSWORD" -C \
    -Q "SET NOCOUNT ON; IF (SELECT COUNT(*) FROM sys.databases WHERE state_desc<>'ONLINE')>0 RAISERROR('not ready',16,1); SELECT 1" \
    -b > /dev/null 2>&1; do sleep 3; done
# applica init.sql con retry (gli errori non uccidono il container)
for i in 1 2 3; do
  if sqlcmd -S localhost -U SA -P "$SA_PASSWORD" -C -i /docker-entrypoint-initdb.d/init.sql -b; then
      echo "init.sql completato."; exit 0
  fi
  echo "init.sql fallito (tentativo $i), riprovo tra 10s..."; sleep 10
done
```

**Nota `-C`**: `mssql-tools18` cifra e valida il certificato per default; con il
cert self-signed del container serve `-C` (trust) su **ogni** invocazione di
sqlcmd, sia nell'entrypoint sia nell'healthcheck. L'equivalente lato SQLAlchemy
è `TrustServerCertificate=yes` nella connection string.

### 10.4 Propagazione dei segnali (shutdown pulito)
Se il `command` lancia `sqlservr &` in background e bash è PID 1, un
`docker stop` manda SIGTERM a bash che **non lo inoltra** a sqlservr → dopo 10s
SIGKILL → recovery del transaction log a ogni riavvio (`N transactions rolled
forward`, numero crescente). Usare la forma a lista con blocco letterale `|`
(preserva i newline) e un trap:

```yaml
command:
  - /bin/bash
  - -c
  - |
    /opt/mssql/bin/sqlservr &
    SQLSERVR_PID=$$!
    trap 'kill -TERM $$SQLSERVR_PID' TERM INT
    bash /init/entrypoint.sh
    wait $$SQLSERVR_PID
```

I `$$` sono necessari: Compose interpola le variabili su tutto il file, quindi
`$` singolo verrebbe letto come variabile. **Non** usare lo scalare folded `>`
(unisce le righe con spazi e distrugge i separatori di comando).

---

## 11. `init.sql` — idempotenza e seed

`init.sql` viene rieseguito a ogni avvio (e dalla retry), quindi ogni statement
deve essere idempotente:
- `CREATE TABLE ... IF NOT EXISTS` / controllo su `sys.tables`
- ogni `CREATE INDEX` protetto da `IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name=...)`
- ogni `INSERT` di seed protetto da `IF NOT EXISTS (SELECT 1 FROM ... WHERE ...)`

Attenzione alle **parole riservate** come nomi di colonna: `plan` va delimitato
`[plan]` ovunque compaia (CREATE TABLE, CHECK constraint, INSERT). I parametri
`@plan` invece non hanno il problema.

**Hash bcrypt dentro `init.sql`**: i `$` **non** vanno scappati — il file è letto
da sqlcmd con `-i`, è SQL puro senza espansione di shell. L'escaping `\$` serve
solo quando l'INSERT è passato via `-Q` dalla riga di comando bash.

---

## 12. Build sequenziale

Build **un servizio alla volta**, non in parallelo: 4 immagini che esportano
layer multi-GB simultaneamente saturano il disco al picco.

```bash
df -h /
docker compose build chainlit
docker compose build celery-worker-high  ; df -h /
docker compose build celery-worker-default ; df -h /
docker compose build celery-beat         ; df -h /
docker compose build fastapi             ; df -h /
```

`docker compose up -d --build` builda in parallelo → evitarlo su disco stretto.
Se durante la sequenza si scende sotto ~6 GB liberi:

```bash
docker builder prune -af
```

I pacchetti Python finiscono **nell'immagine** (`/usr/local/lib/python3.11/
site-packages`), non nel container: modificare `requirements.txt` richiede
sempre un rebuild, non basta il restart. `requirements-dev.txt` **non** viene
installato dalle immagini (solo `requirements.txt`), quindi `pytest` non è
disponibile nei container a meno di aggiungere uno stage `dev`.

---

## 13. Avvio dello stack

```bash
docker compose up -d
docker compose ps
```

L'ordine è gestito dai `depends_on: condition: service_healthy`: fastapi e i
celery attendono che `sqlserver` sia healthy; chainlit attende fastapi. Al primo
avvio, con la data dir vuota, SQL Server impiega ~40s per il setup completo dei
database di sistema.

Seguire il provisioning:

```bash
docker logs -f rag-system-dev-sqlserver-1
# attendere: "Tenant provisioned: tenant_demo_corp" e "init.sql completato."
```

### Healthcheck — usare strumenti presenti nell'immagine
Un healthcheck fallisce se usa un binario assente:
- `qdrant/qdrant` **non** ha `curl` → usare un test TCP con `bash`
- `chainlit` (base python-slim) **non** ha `curl` → usare `python -c urllib`
- worker celery: `celery inspect ping` **senza** `-d worker@%h` (il `%h` non si
  espande nell'healthcheck)
- sqlserver: healthcheck con `USE RAGChat` (non solo `SELECT 1`) così risulta
  healthy solo quando il DB applicativo è realmente apribile

### Esposizione porte
Pubblicare su `0.0.0.0` solo ciò che serve dall'esterno (fastapi 8000, chainlit
8080, flower 5555). Servizi interni come redis e sqlserver: `127.0.0.1:PORT:PORT`
(raggiungibili solo da chi è già sul server) oppure nessuna porta (la
comunicazione tra container passa dalla rete Docker per nome DNS).

---

## 14. Seed dell'utente admin

Se `init.sql` non crea già l'utente, generarne l'hash con la **funzione dell'app**
(così coincide con `verify_password`) e inserirlo:

```bash
docker compose exec fastapi python -c \
  "from app.core.security import hash_password; print(hash_password('CompeteRag2026'))"
```

Inserimento senza problemi di escaping (heredoc quotato + `-i`):

```bash
set -a; source .env; set +a
cat > /tmp/seed.sql <<'EOF'
USE RAGChat;
IF NOT EXISTS (SELECT 1 FROM tenant_demo_corp.users WHERE email='admin@competesrl.it')
  INSERT INTO tenant_demo_corp.users (email, full_name, role, password_hash)
  VALUES ('admin@competesrl.it', 'Admin', 'admin', 'INCOLLA_HASH_INTERO');
EOF
docker compose cp /tmp/seed.sql sqlserver:/tmp/seed.sql
docker compose exec sqlserver /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U SA -P "$SQLSERVER_PASSWORD" -C -i /tmp/seed.sql
```

Verifica: `LEN(password_hash)` deve essere **60** (un valore minore indica hash
troncato dall'escaping → login sempre in 401).

---

## 15. Verifica finale end-to-end

```bash
# 1. health
curl -s http://localhost:8000/health | python3 -m json.tool

# 2. login → token
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@competesrl.it","password":"CompeteRag2026","tenant_slug":"demo-corp"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo "${TOKEN:0:40}"

# 3. rotta protetta (200 = auth completa: JWT + middleware + impersonazione schema)
curl -i http://localhost:8000/api/v1/auth/me -H "Authorization: Bearer $TOKEN"
```

Usare `curl -i` (non `-s | json.tool`) finché non si è certi del 200: con
`json.tool` un errore appare come "Expecting value" senza rivelare se è 401, 404
o 500.

Con `APP_DEBUG=true`, Swagger UI è su `http://<server>:8000/docs` — il modo più
comodo per testare tutti gli endpoint (conosce già payload e schemi). In
staging/prod con `APP_DEBUG=false`, `/docs` non esiste (comportamento voluto).

Ordine di test consigliato, seguendo le dipendenze:
`health → auth → tenants → users → collections → documents (Celery) → jobs → chat`.

---

## 16. Manutenzione e monitoraggio

### Spazio
```bash
docker system df                              # immagini/cache/volumi (deduplicato)
du -sh /srv/data/dev/rag-system/* | sort -rh  # dati persistenti (bind mount)
df -h /
```

La somma ingenua delle immagini sovrastima: i 4 servizi pesanti condividono la
maggior parte dei layer (base + ODBC + deps comuni), contati una sola volta sul
disco. Il numero reale è la riga `Images` di `docker system df`.

**Igiene dopo ogni ciclo di build**: `docker builder prune -af` (la cache di
build può accumulare decine di GB). Per rimuovere immagini vecchie non
referenziate: `docker image prune -af`. **Mai** `--volumes`, e mai toccare
`/srv/data` (dati reali).

### RAM/CPU
```bash
docker stats --no-stream $(docker compose ps -q)
```
Netdata (se presente sull'host) offre lo stesso con storico.

### Hot-reload in dev
Il codice è bind-mount + uvicorn `--reload`: modifiche a `./app`, `./config`,
`./main.py` si ricaricano da sole. **I celery worker non hanno hot-reload** →
dopo modifiche a task/servizi: `docker compose restart celery-worker-high
celery-worker-default celery-beat`. Modifiche a `docker-compose*.yml` o `.env`
richiedono `docker compose up -d` (ricrea il container); `restart` non basta.

---

## 17. Reset completo del database (da zero)

Per ricostruire schemi e DB da zero, azzerando anche il recovery persistente:

```bash
docker compose down
set -a; source .env; set +a
docker run --rm -v "$DATA_ROOT/sqlserver:/data" busybox sh -c "rm -rf /data/* /data/.[!.]*"
chmod 777 "$DATA_ROOT/sqlserver"
ls -lan "$DATA_ROOT/sqlserver"        # verificare che sia VUOTA (solo . e ..)
docker compose up -d sqlserver
docker logs -f rag-system-dev-sqlserver-1
# attendere "init.sql completato.", poi:
docker compose up -d
```

Verificare sempre con `ls` che il wipe sia riuscito **prima** di riavviare: se
`$DATA_ROOT` non era caricata, il `rm` scivola su un path inesistente e il DB
resta intatto (i file `data/`, `log/`, `.system/` owned da UID 10001 sono ancora
lì). Ripartire con la data dir vuota elimina anche il blocco `transactions
rolled forward` all'avvio.

---

## Appendice — Tabella di troubleshooting

| Sintomo | Causa | Fix |
|---|---|---|
| `Malformed entry (URI parse)` su mssql-release.list | doppio blocco `[...]` dal sed sul file Microsoft | repo esplicita con singola `signed-by` |
| `apt-key: not found` | base `python:3.11-slim` = trixie | pinnare `-bookworm` su entrambi gli stage |
| `No matching distribution found: pywin32` | lock compilato su Windows | rigenerare `pip-compile` su Linux |
| `no space left on device` durante export layer | CUDA nelle immagini / disco 48G / build parallelo | torch `+cpu`, `builder prune -af`, build sequenziale, `lvextend` |
| `snapshot ... does not exist during commit` | cache BuildKit orfana (spesso da build falliti per disco) | `docker compose build --no-cache <svc>` |
| `Can't open lib 'ODBC Driver 18...'` | manca `/etc/odbcinst.ini` nel runtime | copiarlo dal builder |
| `The system directory could not be created` | bind mount SQL Server non scrivibile da UID 10001 | `chmod 777 $DATA_ROOT/sqlserver` |
| `entrypoint.sh: Permission denied` (exit 126) | bit `+x` perso al checkout | `git update-index --chmod=+x` + commit |
| `Msg 904 / errore 4060 su RAGChat` | connessione durante recovery del DB | readiness su `state_desc='ONLINE'`, healthcheck con `USE RAGChat` |
| `transactions rolled forward` a ogni avvio | SIGTERM non propagato a sqlservr | trap nel `command`, forma a lista `|` |
| `module 'bcrypt' has no attribute '__about__'` | passlib 1.7.4 + bcrypt 5.x | pin `bcrypt==4.0.1` |
| `email-validator is not installed` | `EmailStr` senza dipendenza | `pydantic[email]` nel `.in` |
| `NoneType object is not callable` in auth | uso di `_async_factory` (privato, None) | usare la property `async_factory` |
| `Msg 109 more columns than values` | INSERT con colonna aggiunta senza valore | allineare colonne e VALUES |
| container Chainlit `Permission denied .chainlit/` | bind mount owned da UID diverso da appuser | `chmod 777` via busybox, non tracciare translations |
| `$DATA_ROOT` / `$SQLSERVER_PASSWORD` vuote | `.env` non caricato nella shell | `set -a; source .env; set +a` |
