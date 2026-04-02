# InvestPro — Documentazione Tecnica della Piattaforma

**Versione:** 2.2.0
**Firebase Project:** `analisimmobiliare-3b4a7`
**Ultimo aggiornamento delle regole Firestore:** v2.2.0-hardened

---

## Changelog v2.2.0

Questa versione consolida tre sessioni di hardening successive. Le modifiche non introducono nuove funzionalità di prodotto ma elevato la superficie di sicurezza, la conformità agli standard di accessibilità e le prestazioni di caricamento.

| Area | Tipo | Sintesi |
|---|---|---|
| `firestore.rules` | **Security** | 5 vulnerabilità chiuse (VUL-01 critica: list escalation su `projects`; VUL-02÷05: dailyOps bypass, whitelist create/update, checkout_sessions validation) |
| `index.html` | **Security / UX** | Validazione URL Stripe prima del redirect; `mode` esplicito su entrambi i piani; `confirm()` nativo sostituito con toast; ID residui PayPal rinominati |
| `index.html` | **Performance** | Lazy loading on-demand di html2canvas, jsPDF, SheetJS (~2.3 MB risparmiate sul caricamento iniziale) |
| `index.html` | **Accessibilità** | Skip link; `aria-label` su tutti i bottoni icon-only; `aria-labelledby` su tutti i dialog; `role="tab/tabpanel"` sul mode toggle; animazione modale corretta |
| `success.html` | **Security** | `defer` su SDK Firebase; flag `confirmed` separato da `redirected`; `{once:true}` su listener pulsante; verifica PRO sincrona prima del listener real-time |
| `success.html` | **UX** | Titolo iniziale neutro ("Verifica Pagamento") fino alla conferma Firebase; `aria-live` su status bar |
| `cancel.html` | **Accessibilità** | `<main>` landmark; `aria-hidden` su icona decorativa; `crossorigin` su Font Awesome |
| `dailyOps` | ⚠️ **Breaking** | Campo `ts: FieldValue.serverTimestamp()` ora **obbligatorio** nel payload di aggiornamento. Vedere Sezione 5.2. |

---

## 1. Panoramica della Piattaforma

InvestPro è una **SaaS di analisi finanziaria immobiliare** interamente client-side (Single-Page Application in HTML/CSS/JS vanilla) con backend su Firebase. Non esiste un server applicativo proprio: tutta la logica di business gira nel browser, mentre Firebase gestisce autenticazione, persistenza e integrazione pagamenti. Stripe è integrato tramite la **Firebase Stripe Extension** (nessuna custom Cloud Function necessaria).

**Scopo:** Consentire a investitori immobiliari di simulare due strategie — affitto a breve termine (*Messa a Reddito*) e compravendita speculativa (*Trading / Flip*) — con output finanziari precisi (cashflow, ROI, ammortamento, break-even) ed export in PDF e XLSX.

---

## 2. Stack Tecnologico

### 2.1 Frontend

| Libreria | Versione | Ruolo | Caricamento |
|---|---|---|---|
| Vanilla HTML/CSS/JS | — | Core applicativo, nessun framework | — |
| Firebase JS SDK (compat) | 9.22.0 | Auth + Firestore | `defer` (eagerly needed) |
| Chart.js | latest CDN | Grafici revenue breakdown e proiezione | `defer` |
| Decimal.js | 10.4.3 | Aritmetica decimale esatta per calcoli finanziari | `defer` |
| Font Awesome | 6.4.0 | Iconografia | CSS link |
| Google Fonts — Inter | — | Tipografia | CSS link |
| html2canvas | 1.4.1 | Cattura DOM per export PDF | **Lazy** (on-demand) |
| jsPDF | 2.5.1 | Generazione file PDF | **Lazy** (on-demand) |
| SheetJS (xlsx) | 0.18.5 | Generazione file `.xlsx` nativo (non CSV mascherato) | **Lazy** (on-demand) |

> **Lazy loading (v2.2.0):** html2canvas, jsPDF e SheetJS non vengono più scaricati al caricamento della pagina. Il modulo `CDN` in `index.html` li inietta dinamicamente con `document.createElement('script')` solo quando l'utente invoca `export.toPDF()` o `export.toExcel()`. Il risparmio sul caricamento iniziale è ~2.3 MB per la maggioranza degli utenti che non utilizza mai l'export.

> **Perché Decimal.js:** Le 360 iterazioni del piano di ammortamento con aritmetica IEEE 754 accumulano errori fino a ±€12 sul saldo finale. Decimal.js con `precision: 20` e `ROUND_HALF_EVEN` garantisce coerenza tra il modale di ammortamento e il grafico di proiezione.

### 2.2 Backend (Firebase)

| Servizio | Utilizzo |
|---|---|
| Firebase Authentication | Google Sign-In (unico provider) |
| Firestore | Utenti, progetti, dati Stripe |
| Firebase Stripe Extension | Checkout, subscriptions, payments (via Admin SDK) |

---

## 3. Architettura dei File

```
/
├── index.html        # SPA principale — dashboard, calcoli, modali
├── success.html      # Pagina post-pagamento con verifica PRO real-time via Firebase
├── cancel.html       # Pagina annullamento checkout (nessun addebito, nessuna Firebase call)
└── firestore.rules   # Security Rules v2.2.0-hardened
```

---

## 4. Modello di Business e Piani

### 4.1 Piano FREE

- Limite: **5 operazioni/giorno** (counter `dailyOps` su Firestore — resistente alla manomissione da console browser; `ts: serverTimestamp()` obbligatorio da v2.2.0)
- Funzionalità disponibili: analisi sensitivity, ammortamento
- Funzionalità bloccate: Flip Mode, grafici Chart.js, export PDF, export Excel

### 4.2 Piano PRO

**Abbonamento Mensile**
- Price ID Stripe: `price_1T0KhVCP1ytB9mXWR6roA7uB`
- Importo: **€29,90/mese**
- Tipo: `subscription` (ricorrente, `mode` ora esplicito nel payload checkout — v2.2.0)

**Lifetime (pagamento unico)**
- Price ID Stripe: `price_1T0KhVCP1ytB9mXW9kBSKwkI`
- Importo: **€199,00**
- Tipo: `payment` (one-time)

### 4.3 Determinazione Stato PRO (triplo check lato client)

Il client verifica lo stato PRO in **cascata** al login:

1. **Custom claim JWT** (`stripeRole === 'pro'`) aggiornato dalla Stripe Extension — `getIdTokenResult(true)` forza il refresh del token
2. **Fallback pagamento lifetime**: query su `customers/{uid}/payments` dove `status == 'succeeded'`
3. **Admin override**: se l'utente è presente in `admins/{uid}`, `isPro` viene forzato a `true`

> **v2.2.0:** Il campo `plan` nel documento `users/{uid}` è **scritto esclusivamente dalla Stripe Extension via Admin SDK**. La Security Rule v2.2.0 usa una whitelist `affectedKeys().hasOnly(['dailyOps'])` sull'update: qualsiasi campo diverso da `dailyOps` — inclusi `plan`, `isPro`, `stripeRole` e campi futuri non ancora previsti — è rigettato per default, eliminando la dipendenza dalla funzione `notTouchingSensitiveFields()` che usava una blocklist.

---

## 5. Struttura Firestore

### 5.1 Collection `admins/{uid}`

| Campo | Tipo | Note |
|---|---|---|
| `email` | string | Leggibile solo dal proprietario del documento |
| `createdAt` | timestamp | — |

**Regole (v2.2.0):**
- `allow get`: solo `isOwner(uid)`
- `allow list`: **`if false`** — impedisce l'enumerazione dell'intera collection (fix v2.2.0)
- `allow write`: **`if false`** — SOLO Firebase Console o Admin SDK

**Aggiunta admin:** creare manualmente il documento `admins/{TARGET_UID}` dalla Firebase Console.

### 5.2 Collection `users/{uid}`

| Campo | Tipo | Scrivibile dal client | Note |
|---|---|---|---|
| `email` | string | Solo in creazione | Deve corrispondere a `request.auth.token.email` |
| `joined` | timestamp | Solo in creazione | Deve essere `FieldValue.serverTimestamp()` |
| `plan` | string (`'free'` \| `'pro'`) | **NO** | Solo Stripe Extension via Admin SDK |
| `isPro` | boolean | **NO** | Solo Stripe Extension via Admin SDK |
| `stripeRole` | string | **NO** | Solo Stripe Extension via Admin SDK |
| `stripeId` | string | **NO** | Solo Stripe Extension via Admin SDK |
| `version` | string | **NO** | — |
| `dailyOps` | object | SÌ, con vincoli | Schema: `{ date: string, count: int, ts: timestamp }` |

> ⚠️ **Breaking change v2.2.0 — campo `dailyOps.ts` obbligatorio:**
> La Security Rule `validDailyOpsUpdate()` ora richiede `ts == request.time`, verificabile solo se il client invia `FieldValue.serverTimestamp()`. Il payload di aggiornamento in `index.html` **deve** includere questo campo:
> ```javascript
> userRef.update({
>   dailyOps: {
>     date:  new Date().toISOString().split('T')[0],  // YYYY-MM-DD UTC
>     count: newCount,
>     ts:    firebase.firestore.FieldValue.serverTimestamp()  // OBBLIGATORIO
>   }
> })
> ```

**Vincolo `dailyOps` (Security Rule `validDailyOpsUpdate()`):**
- `ts` deve essere `FieldValue.serverTimestamp()` — non falsificabile lato client
- `count` deve essere ≥ 1 e ≤ 50 (upper bound contro overflow)
- Se `dailyOps.date` cambia (nuovo giorno): `count` deve essere esattamente `1`
- Se `dailyOps.date` rimane uguale (stesso giorno): `count` deve essere `resource.data.dailyOps.count + 1`
- Impedisce qualsiasi reset o decremento arbitrario

**Creazione (whitelist v2.2.0):** solo i campi `email` e `joined` sono accettati. Qualsiasi campo aggiuntivo causa il rifiuto del write.

**Aggiornamento (whitelist v2.2.0):** solo `dailyOps` è modificabile dal client. Whitelist `hasOnly(['dailyOps'])` — approccio più sicuro della precedente blocklist `notTouchingSensitiveFields()`.

### 5.3 Collection `projects/{projectId}`

| Campo | Tipo | Note |
|---|---|---|
| `uid` | string | UID del proprietario; immutabile in update (v2.2.0) |
| `name` | string | Max 200 caratteri; validato in create e update (v2.2.0) |
| `mode` | string (`'rent'` \| `'flip'`) | Validato con `in ['rent','flip']` in create e update (v2.2.0) |
| `inputs` | map | Snapshot di tutti i valori dei campi input |
| `createdAt` | timestamp | `serverTimestamp()` alla creazione; immutabile in update (v2.2.0) |
| `updatedAt` | timestamp | `serverTimestamp()` sia in create che in update (v2.2.0) |

**Regole (v2.2.0):**
- `allow read`: unificato su `get` e `list` — la regola `resource.data.uid == request.auth.uid` si applica a entrambe le operazioni. Query non filtrate per `uid` vengono rifiutate deterministicamente (fix VUL-01 critica).
- `allow create`: campi obbligatori: `uid`, `name`, `mode`, `inputs`, `createdAt`, `updatedAt`
- `allow update`: `uid` e `createdAt` immutabili; `updatedAt` deve essere `request.time`
- `allow delete`: proprietario o admin

### 5.4 Collection `customers/{uid}` (Stripe Extension)

**Root document:** Il client può **creare** il documento solo con il campo `{ email }` (whitelist stretta) per triggerare la Stripe Extension nei nuovi account. Senza questa creazione iniziale, la Extension non trova il documento e il checkout fallisce. Tutti gli altri campi (`stripeId`, ecc.) sono scritti esclusivamente dalla Extension via Admin SDK. Nessun `update` client è permesso.

> **v2.2.0:** In precedenza la regola era `allow write: if false` anche per la creazione, causando un errore silenzioso ad ogni login. La Security Rule aggiornata ammette `allow create` con whitelist `{ email }` e validazione `email == request.auth.token.email`.

| Sub-collection | Lettura client | Scrittura client |
|---|---|---|
| `checkout_sessions/{sessionId}` | SÌ (owner) | SÌ — `create` con validazione completa |
| `subscriptions/{subscriptionId}` | SÌ (owner) | **NO** |
| `payments/{paymentId}` | SÌ (owner) | **NO** |

**Campi obbligatori e validati per creare una `checkout_session` (v2.2.0):**

| Campo | Validazione |
|---|---|
| `price` | `string`, formato `price_[A-Za-z0-9]+` |
| `success_url` | `string`, schema `https://` obbligatorio |
| `cancel_url` | `string`, schema `https://` obbligatorio |
| `mode` | `string in ['subscription', 'payment']` — **ora obbligatorio** (v2.2.0) |

---

## 6. Flusso di Pagamento Stripe

```
Client (index.html)               Firestore                    Stripe Extension
  │                                   │                               │
  │── valida URL Stripe prima del ─── │                               │
  │   redirect (FIX STR-01, v2.2.0)  │                               │
  │                                   │                               │
  │── crea checkout_session ─────────>│                               │
  │   { price, success_url,           │── trigger Extension ─────────>│
  │     cancel_url, mode }            │                               │── crea Stripe Session
  │                                   │<── scrive { url } ────────────│
  │<── onSnapshot: legge url ─────────│                               │
  │── if url.startsWith(             │                               │
  │     'https://checkout.stripe.com')│                               │
  │── redirect a Stripe ──────────────────────────────────────────── >│
  │                                   │                               │── pagamento completato
  │                                   │<── webhook ───────────────────│
  │                                   │   aggiorna payments/          │
  │                                   │   aggiorna plan:'pro'         │
  │                                   │   aggiorna custom claims      │
  │   (browser → success.html)        │                               │
  │                                   │                               │
Client (success.html)             Firestore                    Stripe Extension
  │                                   │                               │
  │── get users/{uid} (check immediato│                               │
  │   sincrono — v2.2.0) ────────────>│                               │
  │── onSnapshot users/{uid} ────────>│                               │
  │── getIdTokenResult(true) ─────────────────────────── >JWT Server  │
  │<── plan == 'pro' │                │                               │
  │    o stripeRole == 'pro'          │                               │
  │── unlock UI ──────────────────────│                               │
```

**Validazione URL Stripe (FIX STR-01, v2.2.0):** prima di `window.location.assign(data.url)`, il client verifica che l'URL inizi con `https://checkout.stripe.com` o `https://billing.stripe.com`. Un documento `checkout_sessions` con URL arbitrario (scenario di misconfiguration delle Security Rules) non causa redirect a pagine terze.

**URL di redirect (FIX STR-02, v2.2.0):** costruiti tramite getter su `CONFIG.stripe` usando `window.location.origin + '/success.html'` — deterministici indipendentemente da query string e hash nell'URL corrente.

**Timeout di sicurezza in `success.html`:** 20 secondi. Se entro 20 secondi Firestore non conferma `plan:'pro'` (via documento, custom claim o collection `payments`), la pagina mostra un avviso di sincronizzazione lenta senza bloccare l'utente.

---

## 7. Motore di Calcolo

### 7.1 Modalità Messa a Reddito (Rent)

**Input principali:** prezzo acquisto, ristrutturazione, costi notarili, LTV, tasso annuo, durata mutuo, ADR (Average Daily Rate), occupazione %, aliquota fiscale %, commissioni OTA %, costi fissi/mese, costi variabili/notte, manutenzione/mese, vacancy %.

**Formula rata mutuo (ammortamento francese):**
```
mr = (rate/100) / 12
np = years * 12
mp = loanAmt * (mr * (1+mr)^np) / ((1+mr)^np - 1)
```

**KPI calcolati:**
- `cashflow mensile` = (grossRevenue − opCosts − taxes − mortgageAnnual) / 12
- `CoC ROI` = netIncome / equity × 100
- `MOL mensile` = (grossRevenue − opCosts) / 12
- `break-even` = equity / netIncome (anni) — "Mai" se netIncome ≤ 0
- `utile netto/notte` = netIncome / nights

**Sensitivity Analysis** (3 scenari fissi: 50%, 65%, 85% occupazione) — disponibile nel piano FREE.

### 7.2 Modalità Trading / Flip

**Input aggiuntivi:** prezzo rivendita, durata operazione (mesi), commissioni agenzia %, holding mensile, tassazione plusvalenza %.

**KPI calcolati:**
- `profitto lordo` = flipPrice − price − reno − closing − agencyCosts − holdingCosts
- `profitto netto` = grossProfit − taxes
- `ROI` = netProfit / equity × 100
- `profitto mensile` = netProfit / flipMonths

### 7.3 Ammortamento (PRO)

Piano annuale calcolato con Decimal.js (`precision: 20`, `ROUND_HALF_EVEN`). Include rilevamento del piano insostenibile: se `monthlyPayment < loan × monthlyRate` (primo mese in negativo), l'operazione viene bloccata con toast di errore.

---

## 8. State Manager

Classe `StateManager` con store in-memory sincronizzato su `localStorage` (solo dati non sensibili: tema, modalità, nome progetto, snapshot input). I campi `user`, `isPro`, `isAdmin` **non** vengono mai persistiti su localStorage.

**AutoSave (v2.2.0):** ogni 6 secondi salva uno snapshot dei valori degli input via `saveDraft()` (solo se utente loggato). Il riferimento all'intervallo è ora salvato in `app._autoSaveTimer` e liberato tramite `window.addEventListener('beforeunload', ...)` per prevenire memory leak.

**Keyboard shortcuts:**
- `Ctrl/Cmd + S` → salva progetto su Firestore
- `Escape` → chiude tutti i modali attivi

---

## 9. Security Fixes Implementati

### Fix storici (v2.1.0)

| Fix ID | Area | Descrizione |
|---|---|---|
| **A1** | Rate limiting | Counter `dailyOps` spostato da `localStorage` a Firestore con vincolo incremento +1 per Security Rule. Non bypassabile dalla console del browser. |
| **B1** | Piano PRO | Campi `plan`, `isPro`, `joined`, `version` protetti da scrittura client. |
| **B2** | Stripe data | `subscriptions` e `payments` sono in sola lettura per il client. La creazione di pagamenti finti per ottenere stato PRO è bloccata. |
| **B3** | Success page | `success.html` ascolta Firestore in real-time e sblocca l'UI **solo** quando `plan:'pro'` è confermato. Include timeout a 20s per ritardi webhook. |
| **B4** | Success page | `success.html` verifica lo stato PRO reale prima di mostrare conferma. La navigazione diretta all'URL non bypassa il controllo. |
| **C1** | Admin | `isAdmin()` legge il documento reale in `admins/{uid}`, non un'email hardcoded in CONFIG. I client non possono fare `list` sulla collection `admins`. |
| **D1** | Export | L'export Excel genera un vero file `.xlsx` con SheetJS, non un CSV con estensione rinominata. |
| **D2** | Aritmetica | Decimal.js applicato sia nel modale ammortamento che nel grafico proiezione per coerenza numerica. |
| **D4** | Ammortamento | Rilevamento e blocco di piani insostenibili (rata < interessi del primo mese). |

### Fix v2.2.0 — Security Rules Hardening

| Fix ID | Area | Descrizione |
|---|---|---|
| **VUL-01** ⚠️ CRITICO | Firestore Rules | Rimossa `allow list: if isAuth()` dalla collection `projects`. Qualsiasi utente autenticato poteva leggere tutti i progetti di tutti gli utenti tramite query non filtrata. Causa radice: coesistenza di `allow read` e `allow list` — Firestore autorizza se almeno una regola passa. |
| **VUL-02** | Firestore Rules | `dailyOps.ts == request.time` nella funzione `validDailyOpsUpdate()` forza `FieldValue.serverTimestamp()`, rendendo impossibile bypassare il limite giornaliero con date arbitrarie. Cap superiore `count <= 50` aggiunto. |
| **VUL-03** | Firestore Rules | `users create`: sostituita blocklist `!hasAny(['plan','isPro'])` con whitelist `hasOnly(['email','joined'])`. Validazione `email == request.auth.token.email` e `joined == request.time`. |
| **VUL-04** | Firestore Rules | `users update`: sostituita `notTouchingSensitiveFields()` (blocklist) con `affectedKeys().hasOnly(['dailyOps'])` (whitelist). Qualsiasi campo non esplicitamente permesso — inclusi quelli futuri — è bloccato per default. |
| **VUL-05** | Firestore Rules | `checkout_sessions create`: aggiunta validazione `price.matches('price_[A-Za-z0-9]+')`, URL schema `https://` obbligatorio, `mode in ['subscription','payment']` obbligatorio, `hasOnly` per bloccare campi aggiuntivi. |
| **STR-01** | index.html | Validazione `data.url` prima di `window.location.assign()`. Whitelist `['https://checkout.stripe.com', 'https://billing.stripe.com']`. URL arbitrari da Firestore non causano redirect. |
| **STR-02** | index.html | `success_url` e `cancel_url` costruiti con `window.location.origin + '/success.html'` — deterministici indipendentemente da query string presenti nell'URL corrente. |
| **STR-03** | index.html | `mode: 'subscription'` ora esplicito per il piano mensile. Evita ambiguità con la Stripe Extension in caso di cambio del valore di default. |
| **STR-04** | Firestore Rules | `customers/{uid}` ora ammette `create` con whitelist `{ email }` per triggerare la Extension nei nuovi account. Risolve il conflitto documentato in v2.1.0 (Gap #2). |

### Fix v2.2.0 — Frontend e Accessibilità

| Fix ID | Area | Descrizione |
|---|---|---|
| **ACC-01** | success.html | `aria-live="polite"` + `role="status"` sulla status bar: i cambiamenti dinamici di stato sono annunciati dagli screen reader. |
| **ACC-02** | Tutti i file | `aria-hidden="true"` su tutte le icone Font Awesome decorative. |
| **ACC-03** | index.html | Skip link `.skip-link`, `aria-label` su tutti i bottoni icon-only, `aria-labelledby` su tutti i `role="dialog"`, `role="tab/tabpanel"` sul mode toggle. |
| **ACC-04** | index.html | Fix animazione modale: `visibility + opacity` invece di `display:none → flex`. La transizione CSS ora funziona correttamente su tutti i browser. |
| **PERF-01** | success.html | Aggiunto `defer` ai tre script Firebase SDK nel `<head>`. Eliminato il blocco del rendering (~250 KB) prima del primo paint. Codice applicativo spostato in `DOMContentLoaded`. |
| **PERF-02** | index.html | Lazy loading on-demand di html2canvas, jsPDF, SheetJS tramite modulo `CDN`. Risparmio ~2.3 MB sul caricamento iniziale per utenti che non usano l'export. |
| **PERF-03** | index.html | `clearInterval(app._autoSaveTimer)` su `beforeunload`. Prevenzione memory leak sull'intervallo di auto-save. |
| **UX-01** | success.html | Titolo iniziale "Verifica Pagamento" (neutro) invece di "Pagamento Ricevuto" (ottimistico). Evita di mostrare conferma falsa a chi naviga direttamente all'URL. |
| **UX-02** | index.html | `confirm()` nativo sostituito con toast di conferma in-app (`confirmDelete()`). Non blocca il thread UI, stilizzabile, coerente con il design system. |

---

## 10. Configurazione Firebase (CONFIG)

```javascript
const CONFIG = {
    firebase: {
        apiKey:            "AIzaSyDiGDvR2yXm0DE7vlJNGrIPQB0G-ntLHc0",
        authDomain:        "analisimmobiliare-3b4a7.firebaseapp.com",
        projectId:         "analisimmobiliare-3b4a7",
        storageBucket:     "analisimmobiliare-3b4a7.firebasestorage.app",
        messagingSenderId: "29979040948",
        appId:             "1:29979040948:web:597ea331e06567d8d50be1"
    },
    stripe: {
        // v2.2.0: mode esplicito su entrambi i piani (FIX STR-03)
        plans: {
            monthly:  { priceId: "price_1T0KhVCP1ytB9mXWR6roA7uB", amount: 2990,  mode: 'subscription' },
            lifetime: { priceId: "price_1T0KhVCP1ytB9mXW9kBSKwkI", amount: 19900, mode: 'payment' }
        },
        // v2.2.0: getter runtime invece di costruzione stringa fragile (FIX STR-02)
        // window.location.origin è deterministico indipendentemente da hash e query string
        get successUrl() { return window.location.origin + '/success.html'; },
        get cancelUrl()  { return window.location.origin + '/cancel.html'; }
    },
    maxFreeOps: 5,
    version:    "2.2.0"
};
```

> **Nota di sicurezza:** L'`apiKey` Firebase è un identificativo pubblico, non un segreto. L'accesso ai dati è regolato esclusivamente dalle Security Rules di Firestore, non dall'oscuramento della chiave.

---

## 11. Database Città (CITY_DB)

15 città italiane precaricate con aliquota tassa di soggiorno:

| Città | Regione | €/notte |
|---|---|---|
| Roma | Lazio | 3.50 |
| Milano | Lombardia | 3.00 |
| Venezia | Veneto | 4.50 |
| Firenze | Toscana | 4.00 |
| Napoli | Campania | 2.50 |
| Torino | Piemonte | 2.00 |
| Bologna | Emilia-Romagna | 2.50 |
| Verona | Veneto | 2.50 |
| Genova | Liguria | 2.00 |
| Palermo | Sicilia | 1.50 |
| Bari | Puglia | 1.50 |
| Catania | Sicilia | 1.50 |
| Padova | Veneto | 2.00 |
| Trieste | Friuli-Venezia Giulia | 2.00 |
| Brescia | Lombardia | 1.50 |

---

## 12. Operazioni di Sviluppo / Deployment

### Aggiungere un admin

1. Firebase Console → Firestore → crea documento manuale
2. Percorso: `admins/{UID_TARGET}`
3. Campi: `{ email: "admin@example.com", createdAt: <timestamp> }`

Nessun client può scrivere in `admins` — solo Firebase Console o Admin SDK. La collection non è enumerabile da client (`allow list: if false`).

### Applicare le Security Rules

```bash
# Via Firebase CLI (consigliato per CI/CD)
firebase deploy --only firestore:rules

# Oppure: Firebase Console → Firestore → Regole → incolla e pubblica
```

### Aggiornare il payload `dailyOps` (breaking change v2.2.0)

Ogni write su `users/{uid}.dailyOps` deve includere il campo `ts`:

```javascript
// In app.access.checkDailyLimit() — index.html
userRef.update({
    dailyOps: {
        date:  today,      // YYYY-MM-DD, UTC
        count: newCount,
        ts:    firebase.firestore.FieldValue.serverTimestamp()  // OBBLIGATORIO
    }
})
```

### Aggiungere un Price Stripe

1. Creare il Price in Stripe Dashboard
2. Aggiornare `CONFIG.stripe.plans` in `index.html` con `priceId` e `mode`
3. Verificare che la Stripe Firebase Extension sia configurata per ascoltare quel Price ID
4. La Security Rule `checkout_sessions` valida automaticamente il formato `price_[A-Za-z0-9]+`

### Espandere CITY_DB

Aggiungere voci all'array `CITY_DB` in `index.html`:
```javascript
{ name: "NomeCittà", region: "Regione", touristTax: 0.00 }
```

---

## 13. Aree di Sviluppo Futuro (Gap Identificati)

| # | Stato | Descrizione |
|---|---|---|
| 1 | **Aperto** | **Nessuna Cloud Function custom** — tutta la logica di promozione PRO è delegata alla Stripe Extension. Se la Extension viene deprecata, il sistema di upgrade si interrompe senza fallback. *Parzialmente mitigato in v2.2.0*: validazione URL Stripe lato client aggiunta (FIX STR-01). |
| 2 | ✅ **Risolto in v2.2.0** | ~~`customers/{uid}` scritto dal client — la Security Rule `allow write: if false` bloccava la creazione iniziale~~ La Security Rule ora ammette `allow create` con whitelist `{ email }` (FIX STR-04). Il conflitto documentato in v2.1.0 è eliminato. |
| 3 | **Aperto** | **Nessuna validazione server-side degli input di calcolo** — tutti i calcoli avvengono lato client. Un utente può manipolare i valori degli input nel DOM prima del salvataggio. I progetti salvati in Firestore non vengono ri-validati. |
| 4 | **Aperto** | **CITY_DB hardcoded** — 15 città statiche. Per scalare, richiederebbe una collection Firestore `cities` con query server-side. |
| 5 | **Aperto** | **Export PDF tramite screenshot DOM** — html2canvas produce output di qualità variabile su schermi ad alta densità e non è accessibile (testo non selezionabile). Una generazione PDF strutturata lato server sarebbe superiore. |
| 6 | **Aperto** | **Nessun sistema di webhook audit** — non esiste un log degli eventi Stripe ricevuti. In caso di pagamenti falliti o rimborsi, non c'è traccia applicativa indipendente da Stripe Dashboard. |
| 7 | **Aperto** | **Autenticazione single-provider** — solo Google Sign-In. Email/password non è implementato. |
| 8 | **Aperto** | **Bypass `dailyOps.date` residuo** — le Security Rules non possono verificare che la stringa YYYY-MM-DD corrisponda al giorno UTC corrente del server (nessuna funzione di parsing date su `Timestamp` disponibile nelle Rules). Il rischio è mitigato da `ts == request.time` ma non eliminato. Soluzione definitiva: Cloud Function HTTP. |

---

## 14. Ambiente di Sviluppo e Restrizioni Tecniche

- **Nessun Build Tool:** L'applicazione non utilizza Node.js, Webpack, Vite o altri bundler. Tutto il codice è eseguito direttamente nel browser.
- **Gestione Librerie:** Dipendenze critiche (Firebase, Chart.js, Decimal.js) tramite tag `<script defer>` nel `<head>`. Librerie di export (html2canvas, jsPDF, SheetJS) iniettate on-demand dal modulo `CDN`.
- **Moduli ES6 vietati:** Non utilizzare sintassi `import/export`. L'architettura si basa su script globali e sull'oggetto namespace `app`.
- **Vanilla JS Assoluto:** Non suggerire l'introduzione di framework reattivi (React, Vue, Alpine.js) né librerie come jQuery. Il DOM deve essere manipolato esclusivamente con API native (`document.getElementById`, `addEventListener`, ecc.).
- **Firebase Compat API:** Utilizzare rigorosamente la sintassi Compat (`firebase.auth()`, `firebase.firestore()`), non la Modular API v9+.

---

## 15. Istruzioni Operative per l'IA

Quando ti viene richiesto di analizzare, debuggare o scrivere nuovo codice per InvestPro, devi attenerti rigorosamente a queste direttive:

1. **Aderenza allo Stack:** Scrivi solo codice compatibile con le versioni dichiarate nella Sezione 2. Utilizza rigorosamente la sintassi Firebase Compat (`firebase.auth()`, `firebase.firestore()`).

2. **Integrazione nell'Oggetto `app`:** Qualsiasi nuova logica di business o di interfaccia deve essere incapsulata all'interno dell'oggetto globale `app` già esistente in `index.html`. Non creare variabili o funzioni globali esterne all'IIFE.

3. **Preservazione di Decimal.js:** Qualsiasi operazione finanziaria iterativa (ammortamento, proiezione) deve utilizzare i metodi di `Decimal.js`. I calcoli KPI lineari (single-pass) possono usare aritmetica JS nativa.

4. **Output del Codice:** Quando fornisci codice aggiornato, restituisci solo il blocco o la funzione modificata con un breve contesto di dove inserirla. Non riscrivere l'intero `index.html` a meno che non venga esplicitamente richiesto.

5. **Rispetto delle Security Rules v2.2.0:** Prima di proporre una query o una scrittura su Firestore, verifica che non violi le regole definite nella Sezione 5. In particolare:
   - Nessuna `list` su `projects` senza filtro `where('uid','==',user.uid)` (VUL-01)
   - Qualsiasi update su `users/{uid}` deve includere solo il campo `dailyOps` con il campo `ts: FieldValue.serverTimestamp()` (VUL-02/04)
   - Qualsiasi `checkout_session` deve includere `mode` tra i campi obbligatori (VUL-05)

6. **Lazy Loading delle librerie di export:** Non referenziare `html2canvas`, `jsPDF` o `XLSX` come disponibili globalmente al caricamento della pagina. Verificare che il modulo `CDN` abbia caricato la libreria necessaria prima dell'uso, o inserire la chiamata `await CDN.loadPdfDeps()` / `await CDN.loadExcelDeps()` all'inizio della funzione di export.

7. **`success.html` — DOMContentLoaded obbligatorio:** Qualsiasi codice che utilizzi Firebase SDK in `success.html` deve essere all'interno del listener `DOMContentLoaded`, poiché gli script SDK hanno `defer`. Scrivere codice che acceda a `firebase.*` fuori da questo contesto causerebbe `ReferenceError` al caricamento.
