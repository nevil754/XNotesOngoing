# README3.2 — Librerie GPU, file `requirements*.in/.txt`, Dockerfile

> Approfondimento mirato di `README3.md` §2.1 e §6 (Strategia A). Qui il focus è solo su: quali librerie cambiano, come organizzare `.in`/`.txt` restando riproducibili, e come vanno fatti i Dockerfile GPU. Nessun codice toccato, solo spiegazione.

---

## 1. Cosa cambia davvero a livello di librerie Python

| Pacchetto | Oggi (`requirements.in`, CPU) | Da usare sul server GPU | Perché |
|---|---|---|---|
| `torch` | `torch==2.13.0+cpu` (via `--extra-index-url .../whl/cpu`) | `torch==2.13.0+cuXXX` (stessa versione, build CUDA, da index url CUDA) | serve a `sentence-transformers.CrossEncoder`, che si auto-rileva la GPU (`torch.cuda.is_available()`) — README3.md §2.1 |
| `torchvision` | `torchvision==0.28.0+cpu` | `torchvision==0.28.0+cuXXX` (stesso build-tag di torch) | dipendenza di `torch`/`sentence-transformers`, deve combaciare come build |
| `fastembed` | `fastembed==0.8.0` | **`fastembed-gpu==0.8.0`** (pacchetto PyPI separato, non un extra) | vedi punto 1.1 sotto — è la parte meno ovvia |
| `onnxruntime` | `onnxruntime==1.28.0`, ma **non è in `requirements.in`**: è risolto in automatico come dipendenza transitiva di `fastembed` | diventerà `onnxruntime-gpu` automaticamente come dipendenza transitiva di `fastembed-gpu`, una volta cambiato il `.in` | non va aggiunto a mano |
| `sentence-transformers` | `3.3.1` | **invariato** | si auto-rileva CUDA da solo, zero modifiche |
| tutto il resto (`langchain*`, `qdrant-client`, `fastapi`, `celery`, `docling`, ecc.) | — | **invariato** | non fanno inferenza tensoriale, girano su CPU comunque |

### 1.1 Il punto delicato: `fastembed` vs `fastembed-gpu`

`fastembed` (CPU) dichiara come dipendenza il pacchetto PyPI `onnxruntime`. Se aggiungi semplicemente `onnxruntime-gpu` accanto a `fastembed==0.8.0` nel tuo `.in`, ottieni **due pacchetti PyPI diversi che installano entrambi nella stessa cartella `site-packages/onnxruntime/`** → convivenza order-dipendente, rischio concreto che al build risulti installata la build sbagliata (o che l'ultima vinca in modo silenzioso e imprevedibile).

Il team Qdrant ha risolto esattamente questo problema pubblicando un pacchetto gemello: **`fastembed-gpu`**, stesso namespace di import (`from fastembed import TextEmbedding` funziona identico), ma dichiara `onnxruntime-gpu` come dipendenza invece di `onnxruntime`. Quindi:
- `requirements.in` (CPU) → resta `fastembed==0.8.0`
- `requirements-gpu.in` → `fastembed-gpu==0.8.0` (stessa versione, se disponibile — **da verificare su PyPI al momento del `pip-compile`**, dato che le due distribuzioni non sono garantite avere sempre lo stesso numero di release in parallelo)

Nessuna modifica di codice: l'import resta `from fastembed import TextEmbedding` in `app/core/embeddings.py` in entrambi i casi, cambia solo cosa c'è installato nel venv dietro le quinte (esattamente lo spirito di README3.md §2.1/§7).

---

## 2. Prerequisito da chiarire PRIMA di fissare le versioni: quale CUDA ha il server GPU

Sul server 2, prima di scrivere qualunque `.in`:
```bash
nvidia-smi
```
La riga `CUDA Version: XX.Y` in alto a destra è la versione **massima** supportata dal driver installato (non è detto sia quella che vuoi installare, ma è il tetto). Da lì scegli l'indice PyTorch giusto (es. `cu121`, `cu124`, `cu126` — vedi la matrice ufficiale su pytorch.org/get-started/locally) e una versione di `onnxruntime-gpu` compatibile con la stessa combinazione CUDA/cuDNN (la tabella di compatibilità è nella doc di ONNX Runtime).

**Asimmetria importante tra le due librerie**, e condiziona la scelta dell'immagine Docker al punto 5:
- le wheel PyPI di `torch` con suffisso `+cuXXX` **si portano dietro** le librerie CUDA runtime necessarie come dipendenze pip (`nvidia-cublas-cu12`, `nvidia-cudnn-cu12`, ecc.) — bastano driver NVIDIA + NVIDIA Container Toolkit sull'host, l'immagine Docker può restare "nuda" (es. `python:3.11-slim-bookworm`).
- `onnxruntime-gpu` **non** si porta dietro CUDA Toolkit + cuDNN via pip: si aspetta che siano già presenti come librerie di sistema nell'immagine. Su un `python:3.11-slim-bookworm` senza CUDA Toolkit/cuDNN installati, `onnxruntime-gpu` o fallisce all'import o silenziosamente non trova il provider CUDA e resta su CPU (nessun errore fragoroso, solo prestazioni non migliorate — il tipo di bug che scopri solo controllando i log/`nvidia-smi` come suggerito in README3.md §10.7-8).

---

## 3. Come organizzare `requirements-gpu.in`

Data la tua priorità di riproducibilità e lo stile attuale (un unico file `.in` che pinna tutto esplicitamente, non frammentato in file "common"), l'opzione più semplice e coerente è: **`requirements-gpu.in` come copia di `requirements.in` con sole 4 righe diverse**, senza toccare `requirements.in` esistente.

```
--extra-index-url https://download.pytorch.org/whl/cuXXX     # invece di /cpu — cuXXX deciso al punto 2
torch==2.13.0+cuXXX                                            # invece di +cpu
torchvision==0.28.0+cuXXX                                      # invece di +cpu

...

fastembed-gpu==0.8.0     # invece di fastembed==0.8.0 (verificare versione disponibile su PyPI, §1.1)
```
Tutto il resto (langchain, qdrant-client, docling, fastapi, celery, ecc.) resta identico riga per riga a `requirements.in`.

**Costo esplicito di questa scelta:** ogni aggiornamento futuro di una libreria "condivisa" (es. bump di `langchain`) va applicato a mano in **entrambi** i file `.in`. È accettabile finché i due file non devono cambiare spesso; se in futuro la doppia manutenzione diventa fastidiosa, `pip-compile` accetta più file `.in` in input nello stesso comando (`pip-compile requirements-common.in requirements-gpu.in -o requirements-gpu.txt`), quindi si potrebbe fattorizzare in un terzo file `requirements-common.in` — ma è un refactor volontario a parte, non necessario per partire.

`requirements-dev.in` → **nessuna modifica, nessun file GPU dedicato**. Contiene solo tooling (pytest, ruff, mypy, chainlit, ecc.), identico su entrambi i server.

---

## 4. Comando `pip-compile` per generare `requirements-gpu.txt`

Stesso pattern che già usi oggi, container usa-e-getta:
```bash
docker run --rm -v "$(pwd):/w" -w /w python:3.11-slim-bookworm bash -c "
  pip install pip-tools &&
  pip-compile requirements-gpu.in --output-file requirements-gpu.txt --resolver=backtracking --no-header --annotate
"
```
Punti da tenere a mente:
- questo comando **risolve solo metadati/versioni** (scarica gli indici dei pacchetti, non esegue nulla), quindi va benissimo lanciarlo dentro l'immagine CPU `python:3.11-slim-bookworm` così com'è — non serve un'immagine CUDA per generare `requirements-gpu.txt`, serve solo per l'installazione runtime (punto 5).
- `requirements.in` e `requirements-gpu.in` sono due file separati, compilati con due comandi separati (esattamente come già fai oggi per `requirements.in` vs `requirements-dev.in`): nessun conflitto tra i due `--extra-index-url` diversi perché non vengono mai passati allo stesso comando `pip-compile`.
- Il file `requirements-gpu.txt` risultante va committato in git esattamente come `requirements.txt` oggi, stesso principio di riproducibilità futura.

---

## 5. Dockerfile: perché non basta "stesso Dockerfile con requirements-gpu.txt"

`docker/fastapi.Dockerfile` e `docker/celery.Dockerfile` sono multi-stage con **builder e runtime entrambi `python:3.11-slim-bookworm`** (Debian, nessun CUDA Toolkit/cuDNN). Per il motivo spiegato al punto 2 (asimmetria torch vs onnxruntime-gpu), questa base runtime **non basta** per `onnxruntime-gpu` — mancano le librerie CUDA Toolkit/cuDNN di sistema che si aspetta di trovare.

### Struttura consigliata per i 2 nuovi file (additivi, non toccano gli esistenti)
```
docker/fastapi-gpu.Dockerfile
docker/celery-gpu.Dockerfile
```
Runtime stage basato su un'immagine NVIDIA con cuDNN incluso, versione coerente con quanto scelto al punto 2:
```dockerfile
FROM nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04 AS runtime
# la coppia CUDA/cuDNN qui deve combaciare con: il +cuXXX di torch (§3) e la versione di onnxruntime-gpu risolta in requirements-gpu.txt (§4)
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3.11 python3.11-venv python3-pip \
    unixodbc curl libgl1 libglib2.0-0 libmagic1 poppler-utils \
    && ...
```
poi `pip install --no-deps -r requirements-gpu.txt`, `COPY app/ config/ main.py`, stesso `CMD` di oggi (uvicorn / celery worker) — nessuna riga applicativa cambia.

**Un dettaglio da non sottovalutare sul multi-stage:** l'immagine `nvidia/cuda` ufficiale è basata su **Ubuntu**, mentre l'attuale builder è **Debian bookworm**. Copiare `site-packages` compilati (torch, onnxruntime, ecc.) da un builder Debian dentro un runtime Ubuntu è a rischio ABI/glibc (versioni glibc leggermente diverse tra le due distro). Il modo più sicuro per i 2 Dockerfile GPU è **buildare ed eseguire nella stessa famiglia di immagine NVIDIA/Ubuntu** (builder e runtime entrambi `nvidia/cuda:...`, anche rinunciando al multi-stage se serve semplicità) invece di riusare il builder Debian esistente — immagine finale più pesante, ma senza incompatibilità sottili da scoprire in produzione.

### `docker-compose.yml`
I nuovi Dockerfile si agganciano con `build.dockerfile` + `deploy.resources.reservations.devices` (driver `nvidia`) dentro il profilo `gpu`, già descritto in dettaglio in README3.md §6 — non ripetuto qui.

---

## 6. Setup one-time sull'host GPU (fuori dal repo, non file di progetto)

- driver NVIDIA installato, `nvidia-smi` funzionante sull'host
- **NVIDIA Container Toolkit** (`nvidia-ctk`) installato e configurato, così Docker può esporre la GPU ai container
- verifica propedeutica prima di buildare le immagini applicative:
  ```bash
  docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
  ```
  se questo non mostra la GPU, i container applicativi non la vedranno comunque — niente da cercare nel codice/Dockerfile finché questo comando non funziona.

---

## 7. Checklist riassuntiva (solo librerie/requirements/docker — sottoinsieme di README3.md §10)

1. `nvidia-smi` sul server 2 → annotare CUDA Version massima supportata dal driver.
2. Scegliere build `+cuXXX` di torch/torchvision compatibile (pytorch.org/get-started/locally) e versione `onnxruntime-gpu` compatibile con la stessa coppia CUDA/cuDNN.
3. Creare `requirements-gpu.in` = copia di `requirements.in` con le 4 righe del §3 cambiate (incluso verificare che `fastembed-gpu==0.8.0` esista su PyPI, §1.1).
4. Compilare `requirements-gpu.txt` con lo stesso comando `pip-compile` in un container usa-e-getta (§4) — non serve immagine CUDA per questo passo.
5. Creare `docker/fastapi-gpu.Dockerfile` e `docker/celery-gpu.Dockerfile`, base `nvidia/cuda:<versione>-cudnn-runtime-ubuntuXX.YY`, che installano `requirements-gpu.txt` (§5) — build+runtime nella stessa famiglia immagine, non riusare il builder Debian esistente.
6. Aggiornare `docker-compose.yml` con `profiles`/`deploy.resources` (rimando a README3.md §6).
7. Installare NVIDIA Container Toolkit sul server 2, verificare `nvidia-smi` dentro un container di test (§6).
8. Build + avvio, controllare nei log che fastembed/torch carichino sul device `cuda` e non su `cpu` (README3.md §10.7-8).
