# Crea Store Task giornalieri su Jira OSSIT — v3

## Contesto operativo (fisso, non chiedere)

- **Istanza:** https://jira.media-saturn.com
- **Progetto:** OSSIT — "IT Onsite Store Support IT"
- **Auth:** cookie di sessione forniti nella conversazione; se `GET /myself` ≠ 200 → fermati e richiedi cookie aggiornati
- **Naming Summary:** `<SAP Code> - Mediaworld <Nome Negozio> - DD/MM/YYYY`
- **Reporter:** SEMPRE `denz` (Denz, Claudius) — mai il creator
- **Assignee:** SEMPRE `bravozuniga` (me stesso)
- **Countries:** SEMPRE IT → `customfield_17000 = [{"id":"20511"}]`
- **Epic Link:** `customfield_10006 = "OSSIT-1233"` (Store Task)
- **Component:** ereditato dal Task padre del negozio (tipicamente `"Store Task "` con trailing space)
- **Priorità:** NON settarla nel payload (default `Banale` applicato da Jira)
- **Link ITOPS:** `type=Relates`, `outwardIssue=<ITOPS-key>`
- **Transizioni:**
  - `id=211` → "In corso" (per task ancora da fare)
  - `id=171` → "Chiuso" (per task già fatti) — richiede `resolution` obbligatoria

---

## STEP 0 — Auth check
`GET /rest/api/2/myself` → se ≠ 200 fermati e chiedi cookie aggiornati.

---

## STEP 1 — Raccolta input
Chiedi quanti Store Task creare. Per OGNUNO raccogli:

| Campo | Obbligatorio | Esempio |
|---|---|---|
| SAP Code | ✅ | `I396` |
| Nome Negozio | ✅ | `Cuneo` |
| Data task (DD/MM/YYYY) | ✅ | `28/04/2026` |
| Attività del giorno | ✅ | testo libero multilinea → description |
| Ticket ITOPS da linkare | ✅ — **senza questo BLOCCATI subito** | `ITOPS-107` |
| **Il task è già stato fatto?** | ✅ (sì/no) | determina la transizione finale |

Se anche UN solo ITOPS manca → segnala l'errore PRIMA di qualunque scrittura.

---

## STEP 2 — Trova il Task padre del negozio

**JQL primaria:**
```sql
project = OSSIT AND issuetype = Task AND summary ~ "<SAP> Store Task" AND status != Chiuso
```

**Fallback se nessun match:**
```sql
project = OSSIT AND summary ~ "<SAP>" AND summary ~ "Store Task" ORDER BY created DESC
```

Se ancora zero → fermati e chiedi quale Task padre usare come template (key esplicita).

Dal padre estrai `components` per ereditarli.

---

## STEP 3 — Costruisci il payload create
```json
POST /rest/api/2/issue
{
  "fields": {
    "project":    {"key": "OSSIT"},
    "issuetype":  {"name": "Task"},
    "summary":    "<SAP> - Mediaworld <Nome> - DD/MM/YYYY",
    "description":"<attività del giorno>",
    "assignee":   {"name": "bravozuniga"},
    "components": [<copia dal parent>],
    "customfield_10006": "OSSIT-1233"
  }
}
```

⚠️ **NON** includere nel payload `priority`, `reporter`, `customfield_17000`:
- `priority` non è sul create screen (Jira applica default Banale)
- `reporter` e `Countries` vanno settati via PUT subito dopo (STEP 5.2)
- Altri campi (Product ID, Value Stream, Product Label, Delivery Team) si popolano automaticamente

---

## STEP 4 — Anteprima (dry-run obbligatorio)
Mostra tabella con:

| # | Summary | Parent | ITOPS | Già fatto? | Transizione finale |
|---|---|---|---|---|---|

Chiedi conferma esplicita "procedo?". Crea i ticket solo dopo "sì".

---

## STEP 5 — Esecuzione (sequenza per ogni ticket)

### 5.1 — Crea il ticket
```
POST /rest/api/2/issue
```
→ ottieni `OSSIT-XXXX` dalla response.

### 5.2 — Imposta reporter + Countries (PUT)
```json
PUT /rest/api/2/issue/OSSIT-XXXX
{
  "fields": {
    "reporter":          {"name": "denz"},
    "customfield_17000": [{"id": "20511"}]
  }
}
```

### 5.3 — Transizione finale (dipende da "già fatto?")

**Se "già fatto?" = NO** → porta in corso:
```json
POST /rest/api/2/issue/OSSIT-XXXX/transitions
{"transition": {"id": "211"}}
```

**Se "già fatto?" = SÌ** → chiudi direttamente:
```json
POST /rest/api/2/issue/OSSIT-XXXX/transitions
{
  "transition": {"id": "171"},
  "fields": {"resolution": {"name": "Completato"}}
}
```

⚠️ La transizione 171 richiede `resolution` ★obbligatoria★.
Valori validi (locale it_IT):
- **`Completato`** (default per task chiusi correttamente)
- `Won't Do`
- `Duplicato`
- `Corretto`
- `Impossibile riprodurre`

### 5.4 — Link al ticket ITOPS
```json
POST /rest/api/2/issueLink
{
  "type": {"name": "Relates"},
  "inwardIssue":  {"key": "OSSIT-XXXX"},
  "outwardIssue": {"key": "ITOPS-XXX"}
}
```

### Gestione errori
Se uno step fallisce → **interrompi il batch**, riporta cosa è stato fatto finora.
**NON** tentare retry automatici.

---

## STEP 6 — Report finale

Tabella Markdown:

| Key | Summary | Status | Reporter | Assignee | Link ITOPS | URL |
|---|---|---|---|---|---|---|

+ riga di sintesi: `N creati / M attesi / X falliti`

---

## Headers HTTP standard per tutte le chiamate

```
Cookie: <cookie di sessione forniti>
Content-Type: application/json
Accept: application/json
X-Atlassian-Token: no-check     (obbligatorio per POST/PUT, bypassa XSRF)
```

---

## Note operative apprese

1. L'istanza è **Jira Server 10.3.19** (self-hosted, NON Cloud).
2. Il proxy SSO blocca PAT/Bearer/Basic-auth: **solo cookie di sessione funzionano**, in particolare il cookie `k` (JWT firmato dall'SSO `auth.mediamarktsaturn.com`).
3. Il cookie `k` ha durata circa **15 minuti** (verificare `exp` nel JWT) — se le chiamate iniziano a tornare 302, richiedere nuovi cookie all'utente.
4. La UI Jira è in italiano: gli status hanno nomi tradotti (`Aperto`, `In corso`, `Chiuso`) ma le transition hanno `name` in inglese (`In Progress`, `Close`).
5. Custom field non settabili da API (gestiti da Jira config in base a project+components):
   - `customfield_17773` (Product ID)
   - `customfield_21404` (Value Stream / Platform Domain Label)
   - `customfield_21405` (Product / Platform Label)
   - `customfield_21406` (Delivery Team / Platform Foundation)
