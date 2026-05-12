# Session Handoff — Jira OSSIT / OSS-App project

**Generato:** 2026-05-12
**User:** Bravo Zuniga, Fabrizio (`bravozuniga`)
**Scopo:** ripristinare in una nuova sessione tutto il contesto Jira/OSSIT senza dover rifare l'esplorazione iniziale.

---

## 🔑 1. Autenticazione Jira (CRITICA — leggi prima)

L'istanza **`https://jira.media-saturn.com`** è Jira **Server 10.3.19** dietro un reverse proxy SSO (`auth.mediamarktsaturn.com`).

### ❌ Cosa NON funziona
- Personal Access Token (PAT) → tutte le richieste → HTTP 302 redirect a OAuth
- Basic Auth (email:token) → 302
- Bearer token con qualunque formato → 302

Il proxy SSO intercetta **prima** che la richiesta raggiunga Jira.

### ✅ L'unico metodo che funziona
**Cookie di sessione** estratti dal browser dopo login SSO, in particolare il cookie **`k`** (JWT firmato da `auth.mediamarktsaturn.com`).

**Cookie completi richiesti** (passare tutti come header `Cookie:`):
```
atlassian.xsrf.token
JiraSDSamlssoLoginV2
JSESSIONID
k                          ← critico, JWT con exp ~15 min
seraph.rememberme.cookie
t
```

### Come estrarli (ad inizio sessione)
1. Apri `https://jira.media-saturn.com/secure/Dashboard.jspa` in browser, login via SSO
2. F12 → Console → `copy(document.cookie)` → incollare in chat
3. Verifica: `GET /rest/api/2/myself` deve restituire HTTP 200

### Headers obbligatori per POST/PUT
```
Cookie: <stringa cookie completa>
Content-Type: application/json
Accept: application/json
X-Atlassian-Token: no-check     ← bypassa XSRF, obbligatorio per write
```

### ⚠️ Scadenza cookie
Il JWT `k` ha `exp` ~15 minuti dall'emissione. Se le chiamate iniziano a tornare 302 in mezzo a una sessione, richiedi cookie aggiornati all'utente.

---

## 🏢 2. Contesto progetto OSSIT

| Campo | Valore |
|---|---|
| **Progetto** | OSSIT — IT Onsite Store Support IT |
| **ID** | 25211 |
| **Tipo** | software |
| **Lead** | Denz, Claudius (`denz`) |
| **Categoria** | Platform Foundation Team |
| **Ticket totali (12/05/2026)** | 4.483 |
| **Issue types abilitati** | 23 (Epic, Task, Sotto-Task, Support, Maintenance, Enabler, Bug, Business Request, Risk, Patch, Defect, Impediment, Test*, ecc.) |

### Gerarchia operativa
```
Epic "Store Task"  (OSSIT-1233)
 └─ Task per-negozio "<SAP> - Media World <NEGOZIO> - Store Task"
     ├─ Sotto-Task / Task giornaliero  ← QUELLI CHE CREIAMO
     └─ ...
```

### Componenti del progetto
`Closing` · `Mainrack change` · `NGN` · `Opening` · `Optimization` · `PA Hardware Lifecycle IT` · `Relocation` · `Server room check` · `Store Task ` (⚠️ trailing space) · `SW Hardware Lifecycle IT` · `unknown devices`

### Workflow Task (17 stati, locale it_IT)
**Da completare:** Aperto · In Refinement · Refined · Selected for Development · Reopen
**In corso:** In corso · In Review · In UX Review · In Test on INT · In Test on QA · In Acceptance · Impeded
**Completato:** UAT Approved · In UAT · In Rollout · Risolti · Chiuso

---

## 🔢 3. Mapping ID — fissi per OSSIT

### Transizioni Task
| ID | Nome (en) | → Stato (it) | Note |
|---|---|---|---|
| 301 | In Refinement | In Refinement | |
| 291 | Refined | Refined | |
| 201 | Selected for Development | Selected for Development | |
| **211** | **In Progress** | **In corso** | per task ancora da fare |
| **171** | **Close** | **Chiuso** | per task già fatti — **richiede `resolution`** |

### Resolution (per transition 171)
| Nome (it_IT) | Uso |
|---|---|
| **Completato** | default per task chiusi correttamente |
| Won't Do | rinunciato |
| Duplicato | duplicato di altro |
| Corretto | bug fix |
| Impossibile riprodurre | bug non riproducibile |

### Custom fields rilevanti
| Field ID | Nome | Tipo | Valore standard OSSIT |
|---|---|---|---|
| `customfield_10006` | Epic Link | text | `"OSSIT-1233"` (Store Task) |
| `customfield_17000` | Countries | multi-select | `[{"id":"20511"}]` → IT |
| `customfield_10800` | Rank | (Jira-managed) | non settare |
| `customfield_17773` | Product ID | (auto) | NON settabile da API |
| `customfield_21404` | Value Stream / Platform Domain Label | (auto) | NON settabile da API |
| `customfield_21405` | Product / Platform Label | (auto) | NON settabile da API |
| `customfield_21406` | Delivery Team / Platform Foundation | (auto) | NON settabile da API |

### Issue Link Types disponibili
`Belongings` · `Blocks` · `Cloners` · `Defect` · `Demand` · `Deployment` · `Duplicate` · `Issue split` · `Part of` · `Problem/Incident` · **`Relates`** ← usato per linkare OSSIT ↔ ITOPS · `Test`

### Priorità (display name = API name)
| ID | Name |
|---|---|
| 1 | Bloccante |
| 2 | Critico |
| 3 | Grave |
| 4 | Medium |
| 5 | Banale |

⚠️ **NON includere `priority` nel payload `POST /issue`** — non è sul create screen per il progetto OSSIT. Jira applica Banale come default.

---

## 👤 4. Utenti chiave

| Username | Display Name | Ruolo |
|---|---|---|
| `bravozuniga` | Bravo Zuniga, Fabrizio | Me (assignee dei task creati) |
| `denz` | Denz, Claudius | Reporter fisso, Project Lead OSSIT |

---

## 📋 5. Prompt operativo — Crea Store Task v3

Salvato in `OSSIT-create-store-task-prompt.md` (committato in `hunterprog-afk/test`).
**Versione canonica** del workflow in 6 step:
1. **STEP 0** Auth check (`GET /myself`)
2. **STEP 1** Raccolta input + domanda **"task già fatto?"** (sì → chiudi / no → in corso)
3. **STEP 2** Trova Task padre del negozio via JQL
4. **STEP 3** Build payload `POST /issue`
5. **STEP 4** Dry-run obbligatorio con conferma
6. **STEP 5** Esecuzione: create → PUT reporter+Countries → transition → link ITOPS
7. **STEP 6** Report finale

---

## 🛠️ 6. Endpoints REST API usati

| Operazione | Endpoint | Method |
|---|---|---|
| Auth check | `/rest/api/2/myself` | GET |
| Lista progetti | `/rest/api/2/project` | GET |
| Dettaglio progetto | `/rest/api/2/project/{key}` | GET |
| Statuses progetto | `/rest/api/2/project/{key}/statuses` | GET |
| Search JQL | `/rest/api/2/search?jql=...` | GET |
| Dettaglio issue | `/rest/api/2/issue/{key}` | GET |
| Create issue | `/rest/api/2/issue` | POST |
| Update issue | `/rest/api/2/issue/{key}` | PUT |
| Transitions list | `/rest/api/2/issue/{key}/transitions?expand=transitions.fields` | GET |
| Esegui transition | `/rest/api/2/issue/{key}/transitions` | POST |
| Crea link | `/rest/api/2/issueLink` | POST |
| Priority list | `/rest/api/2/priority` | GET |
| Link types | `/rest/api/2/issueLinkType` | GET |
| Server info | `/rest/api/2/serverInfo` | GET |

---

## 🚨 7. Insidie note (lessons learned 12/05/2026)

1. **Component name** = `"Store Task "` con spazio finale → riportare letterale, non sanitizzare.
2. **`priority` nel POST /issue** → errore "Il nome della priorità 'Banale' non è valido" anche se è il nome esatto. Causa: non è sul create screen. → Rimuovere dal payload.
3. **`reporter` nel POST /issue** → ignorato. Impostarlo con `PUT /issue/{key}` subito dopo create.
4. **`customfield_17000` (Countries)** → settabile via PUT post-create, NON nel payload create.
5. **Transition 171 (Close)** → richiede `resolution` obbligatoria. Includere `"fields":{"resolution":{"name":"Completato"}}` nel body.
6. **`X-Atlassian-Token: no-check`** → obbligatorio per qualunque POST/PUT, altrimenti XSRF reject.
7. **Status names sono in italiano** (Aperto/In corso/Chiuso), ma **transition names sono in inglese** (In Progress/Close).
8. **Cookie `k` scade ~15 min** → checare 302 e rinnovare.

---

## 🗂️ 8. Stato lavoro 12/05/2026

### Ticket creati in sessione (OSSIT)
| Key | Summary | Status finale | Note |
|---|---|---|---|
| OSSIT-4947 | I704 - Mediaworld Catania - 20/04/2026 | Chiuso (Completato) | Rinominato + chiuso |
| OSSIT-4948 | I704 - Mediaworld Catania - 21/04/2026 | Chiuso (Completato) | Rinominato + chiuso |
| OSSIT-4949 | I058 - Mediaworld Rescaldina - 22/04/2026 | Chiuso (Completato) | Rinominato + chiuso |
| OSSIT-4950 | I396 - Mediaworld Cuneo - 28/04/2026 | Chiuso (Completato) | Rinominato + chiuso |

Tutti con: reporter=denz, assignee=bravozuniga, Countries=IT, Epic Link=OSSIT-1233, Component=Store Task, Link "relates to" ITOPS-107.

### File pushati su `hunterprog-afk/test` branch `claude/jira-api-integration-dGXKc`
- `OSSIT-create-store-task-prompt.md` (prompt v3)
- `SESSION-HANDOFF.md` (questo file)

PR draft: https://github.com/hunterprog-afk/test/pull/1

---

## 🚀 9. Prossimi step pianificati — Progetto OSS-App

Repo creata: **`https://github.com/hunterprog-afk/OSS-App`** (vuota).

### TODO sessione successiva
- [ ] Verificare/abilitare whitelist GitHub MCP per `hunterprog-afk/OSS-App`
- [ ] Push scaffolding iniziale:
  - `README.md` con overview
  - `docs/jira-api-notes.md` (porting di questo handoff)
  - `docs/OSSIT-create-store-task-prompt.md` (prompt v3)
  - `data/stores.csv.template` (anagrafica negozi: sap_code, as400_code, store_name, city, region, store_type, manager, phone, email, notes)
  - `.gitignore` baseline
- [ ] Scegliere stack app (Python FastAPI vs Node/React)
- [ ] Definire flusso UI per OSS in store (3-tap goal)

### Anagrafica negozi — schema atteso
Colonne suggerite (l'utente la manterrà a mano):
```
sap_code,as400_code,store_name,city,region,store_type,opening_date,store_manager,phone,email,notes
I396,CN2W,Cuneo Xpress,Cuneo,Piemonte,Xpress,15/04/2024,...,...,...,...
```

---

## 📥 10. Bootstrap rapido per nuova sessione

Incolla questo blocco all'inizio della nuova sessione:

```
Riprendiamo il lavoro su Jira OSSIT / progetto OSS-App.
Leggi il file /home/user/TEST/SESSION-HANDOFF.md per ricostruire il contesto.
File correlati nella stessa directory:
  - OSSIT-create-store-task-prompt.md (prompt v3 operativo)

Repo:
  - hunterprog-afk/test (sandbox, branch claude/jira-api-integration-dGXKc)
  - hunterprog-afk/OSS-App (target finale, da abilitare in whitelist)

Per autenticarti su Jira chiedimi i cookie aggiornati (il cookie `k`
del JWT SSO scade ogni ~15 min).
```

Fine handoff.
