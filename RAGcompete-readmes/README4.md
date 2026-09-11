# `/AdminTenants` vs `/Spaces` — a cosa servono e perché sono diverse

> Domanda: loggato come admin di uno Space, perché `http://.../AdminTenants` si comporta/appare diversamente da `http://.../Spaces`? Non dovrebbero essere la stessa cosa?

## Risposta breve

Creano lo **stesso tipo di dato** (un tenant, riga in `shared.tenants` sul backend), ma sono due **strumenti diversi per due pubblici diversi**:

| | `/Spaces` ("I miei Space") | `/AdminTenants` ("Gestione Tenant") |
|---|---|---|
| Chi può accedere | qualsiasi utente platform loggato (self-service) | **solo superadmin di piattaforma** |
| Backend chiamato | `GET/POST/PATCH /spaces...` | `GET/POST/PATCH /tenants...` |
| Cosa vedi nella lista | solo gli Space **di tua proprietà** | **tutti** i tenant del sistema |
| Slug | generato automaticamente dal nome | scelto a mano da chi crea |
| Admin del tenant creato | **tu stesso** (riusa il tuo account platform) | opzionale, email/password separate da passare a mano |
| Piano | sempre `starter` | scelto a mano (`starter`/`pro`/`enterprise`) |

Il motivo per cui tu, "admin di uno Space", vedi qualcosa di diverso (probabilmente **"Accesso negato"**) su `/AdminTenants` è che **essere admin del tuo Space non è la stessa cosa che essere superadmin della piattaforma** — sono due ruoli su due livelli completamente diversi. Vedi §3.

---

## 1. `/Spaces` — self-service, per chiunque

Frontend: `SpacesController` (`RagSystemFrontend.UI/Controllers/SpacesController.cs`) → chiama il backend `/spaces` (`app/api/routes/spaces.py`).

- Chiunque loggato come utente platform (`[Authorize(Policy = "PlatformAuth")]`, senza altro controllo) può:
  - vedere **solo i propri** Space (`list_spaces` filtra `WHERE owner_user_id = :owner_id`, `spaces.py:56-66`)
  - crearne uno nuovo (`create_space`, `spaces.py:70-113`): lo slug è auto-generato da `slugify(nome)`, il piano è sempre `"starter"`, e **tu diventi automaticamente l'admin** di quel tenant — il backend riusa id/email/password-hash del tuo account platform (`owner_user_id`/`owner_email`/`owner_password_hash`, vedi `app/services/tenant_service.py:62-79`), niente credenziali separate da inventare
  - rinominarlo, disabilitarlo, o "**selezionarlo**" (`select_space`, `spaces.py:180-209`) — quest'ultimo emette un JWT tenant-scoped e ti fa entrare in quello Space, un'azione che `/AdminTenants` non ha nemmeno

In pratica: è il flusso "crea il tuo ambiente e lavoraci", pensato per un cliente/utente che si autogestisce.

## 2. `/AdminTenants` — backoffice, solo superadmin

Frontend: `AdminTenantsController` (`RagSystemFrontend.UI/Controllers/AdminTenantsController.cs`) → chiama il backend `/tenants` (`app/api/routes/tenants.py`).

- **Ogni** endpoint richiede `SuperAdminOnly` lato backend (`tenants.py:34,56,71` → dependency `require_superadmin`, `app/api/deps.py:213-223`, che verifica `SELECT is_superadmin FROM shared.platform_users WHERE id = :id`)
- Mostra **tutti** i tenant esistenti nel sistema (`list_tenants`, `tenants.py:55-67`, nessun filtro per owner)
- La creazione (`create_tenant`, `tenants.py:33-52`) è pensata per provisioning "d'ufficio": slug scelto a mano, piano scelto a mano, admin **opzionale** con email/password passate esplicitamente (o nessuno, se li ometti — il tenant resta senza utente admin, "andrà aggiunto separatamente", come dice la nota nella view stessa: `Views/AdminTenants/Index.cshtml:44-48`)
- Non esiste un'azione "seleziona" — questo schermo serve a *provisionare* tenant per conto di altri, non a *lavorarci dentro*

La view stessa lo dice esplicitamente in cima alla pagina (`Views/AdminTenants/Index.cshtml:6-10`):
> "Provisioning 'senza owner', in gran parte ridondante con gli Space self-service: utile per creare tenant da gestire indipendentemente da un account platform specifico."

Quindi sì — **è in gran parte ridondante** con `/Spaces` per il caso d'uso comune. Ha senso quando serve creare un tenant che non deve appartenere a nessun account specifico (demo, tenant interni, tenant "orfani" gestiti solo da backoffice) o quando serve un piano diverso da `starter` fin dalla creazione.

## 3. Perché tu, "admin di uno Space", vedi qualcosa di diverso

Sono **due concetti di "admin" completamente separati e su livelli diversi**:

1. **Ruolo dentro il tuo tenant** — la colonna `role` nella tabella `<tenant_schema>.users` (valori `admin`/`user`/`viewer`, vedi `docker/sqlserver/init.sql:190`). Essere "admin" qui vuol dire poter gestire *quel* tenant/Space (utenti, collezioni, documenti al suo interno). È lo stesso ruolo che ti viene assegnato in automatico quando crei un tuo Space (`tenant_service.py:66`, `role: 'admin'`).
2. **Flag superadmin di piattaforma** — la colonna `is_superadmin` in `shared.platform_users` (`docker/sqlserver/init.sql:29`), completamente slegata dal punto 1. È un flag globale, non per-tenant, e nel DB è impostato **solo** per l'account seed `admin@platform.competesrl.it` inserito da `init.sql:136-143` — nessun altro platform user lo ha di default, nemmeno chi crea/possiede Space.

Essere admin (1) del tuo Space **non** ti dà (2). Per questo `/AdminTenants` ti restituisce "Accesso negato" (`Views/AdminTenants/AccessDenied.cshtml`) anche se dentro al tuo Space sei effettivamente l'amministratore.

### Un dettaglio UX che probabilmente ha contribuito alla tua confusione

Il link **"Gestione Tenant"** nel menu di navigazione (`Views/Shared/_Layout.cshtml:41`) è mostrato a **chiunque sia loggato come platform user**, non solo ai superadmin — la condizione nel layout è solo `User.IsPlatformAuthenticated()`, senza controllo `is_superadmin`. Quindi il link lo vedi comunque, ci clicchi, e solo *dopo* il backend ti risponde 403 e il frontend mostra la pagina di accesso negato.

Questo non è un errore ma una scelta implicita del disegno attuale: il JWT emesso da `/auth/platform-login` (`app/api/routes/auth.py:162-166`) porta solo `sub`, `email`, `is_platform: True` — **non** un claim `is_superadmin`. Senza quel claim nel token, il frontend non ha modo di sapere se nascondere il link senza fare una chiamata API dedicata in più, quindi mostra il link a tutti e lascia che sia il backend a bloccare chi non ha i permessi. Risultato pratico: la voce di menu è fuorviante per un utente non-superadmin, ma il controllo di sicurezza reale (lato backend) è corretto e non aggirabile.

## 4. In sintesi, quando usare quale

- **Sei un cliente/utente che vuole il proprio ambiente RAG**: usa `/Spaces` — self-service, immediato, diventi admin del tuo Space in automatico.
- **Sei il superadmin di piattaforma e devi provisionare un tenant per conto di qualcun altro, senza legarlo a un account specifico, o con un piano diverso da starter fin da subito**: usa `/AdminTenants`.
- Se non sei superadmin, `/AdminTenants` non ti servirà mai a nulla nell'attuale disegno del sistema — anche se il link resta visibile nel menu.
