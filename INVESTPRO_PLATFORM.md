# InvestPro — Documentazione Tecnica della Piattaforma

**Versione:** 2.1.0  
**Firebase Project:** `analisimmobiliare-3b4a7`  
**Ultimo aggiornamento delle regole Firestore:** v2.1.0-secure

---

## 1. Panoramica della Piattaforma

InvestPro è una **SaaS di analisi finanziaria immobiliare** interamente client-side (Single-Page Application in HTML/CSS/JS vanilla) con backend su Firebase. Non esiste un server applicativo proprio: tutta la logica di business gira nel browser, mentre Firebase gestisce autenticazione, persistenza e integrazione pagamenti. Stripe è integrato tramite la **Firebase Stripe Extension** (nessuna custom Cloud Function necessaria).

**Scopo:** Consentire a investitori immobiliari di simulare due strategie — affitto a breve termine (*Messa a Reddito*) e compravendita speculativa (*Trading / Flip*) — con output finanziari precisi (cashflow, ROI, ammortamento, break-even) ed export in PDF e XLSX.

---

## 2. Stack Tecnologico

### 2.1 Frontend

| Libreria | Versione | Ruolo |
|---|---|---|
| Vanilla HTML/CSS/JS | — | Core applicativo, nessun framework |
| Firebase JS SDK (compat) | 9.22.0 | Auth + Firestore |
| Chart.js | latest CDN | Grafici revenue breakdown e proiezione |
| html2canvas | 1.4.1 | Cattura DOM per export PDF |
| jsPDF | 2.5.1 | Generazione file PDF |
| SheetJS (xlsx) | 0.18.5 | Generazione file `.xlsx` nativo (non CSV mascherato) |
| Decimal.js | 10.4.3 | Aritmetica decimale esatta per calcoli finanziari |
| Font Awesome | 6.4.0 | Iconografia |
| Google Fonts — Inter | — | Tipografia |

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
├── success.html      # Pagina post-pagamento con verifica PRO real-time
├── cancel.html       # Pagina annullamento checkout (nessun addebito)
└── firestore.rules   # Security Rules v2.1.0-secure
```

---

## 4. Modello di Business e Piani

### 4.1 Piano FREE

- Limite: **5 operazioni/giorno** (counter su Firestore, non localStorage — resistente alla manomissione da console)
- Funzionalità disponibili: analisi sensitivity, ammortamento
- Funzionalità bloccate: Flip Mode, grafici Chart.js, export PDF, export Excel

### 4.2 Piano PRO

**Abbonamento Mensile**
- Price ID Stripe: `price_1T0KhVCP1ytB9mXWR6roA7uB`
- Importo: **€29.90/mese**
- Tipo: subscription ricorrente

**Lifetime (pagamento unico)**
- Price ID Stripe: `price_1T0KhVCP1ytB9mXW9kBSKwkI`
- Importo: **€199.00**
- Tipo: `payment` (one-time)

### 4.3 Determinazione Stato PRO (triplo check lato client)

Il client verifica lo stato PRO in **cascata** al login:

1. **Custom claim JWT** (`stripeRole === 'pro'`) aggiornato dalla Stripe Extension
2. **Fallback pagamento lifetime**: query su `customers/{uid}/payments` dove `status == 'succeeded'`
3. **Admin override**: se l'utente è presente in `admins/{uid}`, `isPro` viene forzato a `true`

> Il campo `plan` nel documento `users/{uid}` è **scritto esclusivamente dalla Stripe Extension via Admin SDK**. Nessun client può scrivere su quel campo (enforced da Security Rule `notTouchingSensitiveFields()`).

---

## 5. Struttura Firestore

### 5.1 Collection `admins/{uid}`

| Campo | Tipo | Note |
|---|---|---|
| `email` | string | Leggibile solo dal proprietario del documento |
| `createdAt` | timestamp | — |

**Regole:** lettura consentita solo all'`uid` corrispondente; **nessuna scrittura client** (solo Firebase Console o Admin SDK).

**Aggiunta admin:** creare manualmente il documento `admins/{TARGET_UID}` dalla Firebase Console.

### 5.2 Collection `users/{uid}`

| Campo | Tipo | Scrivibile dal client |
|---|---|---|
| `email` | string | Solo in creazione |
| `plan` | string (`'free'` \| `'pro'`) | **NO** — solo Stripe Extension |
| `isPro` | boolean | **NO** |
| `joined` | timestamp | **NO** — `serverTimestamp()` in creazione |
| `version` | string | **NO** |
| `dailyOps` | object `{ date: string, count: int }` | SÌ, con vincolo incremento +1 |

**Vincolo `dailyOps` (Security Rule A1):**
- Se `dailyOps.date` cambia (nuovo giorno): `count` deve essere esattamente `1`
- Se `dailyOps.date` rimane uguale: `count` deve essere esattamente `resource.data.dailyOps.count + 1`
- Impedisce qualsiasi reset o decremento arbitrario dal client

### 5.3 Collection `projects/{projectId}`

| Campo | Tipo | Note |
|---|---|---|
| `uid` | string | UID del proprietario, immutabile in update |
| `name` | string | Nome progetto |
| `mode` | string (`'rent'` \| `'flip'`) | Modalità analisi |
| `inputs` | object | Snapshot di tutti i valori dei campi input |
| `createdAt` | timestamp | — |

**Regole:** CRUD solo per il proprietario (`uid == request.auth.uid`). Admin può eliminare.

### 5.4 Collection `customers/{uid}` (Stripe Extension)

**Root document:** scritto solo dalla Stripe Extension. Il client crea il documento se non esiste (field `email`) per garantire la compatibilità con utenti pre-esistenti al deployment della Extension.

| Sub-collection | Lettura client | Scrittura client |
|---|---|---|
| `checkout_sessions/{sessionId}` | SÌ (owner) | SÌ — avvia il checkout |
| `subscriptions/{subscriptionId}` | SÌ (owner) | **NO** |
| `payments/{paymentId}` | SÌ (owner) | **NO** |

**Campi obbligatori per creare una `checkout_session`:** `price`, `success_url`, `cancel_url`.

---

## 6. Flusso di Pagamento Stripe

```
Client                         Firestore                      Stripe Extension
  │                                │                                │
  │── crea checkout_session ──────>│                                │
  │   { price, success_url,        │── trigger Extension ──────────>│
  │     cancel_url }               │                                │── crea Stripe Session
  │                                │<── scrive { url } ─────────────│
  │<── onSnapshot: legge url ──────│                                │
  │── redirect a Stripe ───────────────────────────────────────────>│
  │                                │                                │── pagamento completato
  │                                │<── webhook ────────────────────│
  │                                │   aggiorna payments/           │
  │                                │   aggiorna plan:'pro'          │
  │<── success.html ───────────────│                                │
  │── onSnapshot users/{uid} ─────>│                                │
  │<── plan == 'pro' → unlock ─────│                                │
```

**Timeout di sicurezza in `success.html`:** 20 secondi. Se entro 20 secondi Firestore non conferma `plan:'pro'`, la pagina mostra un avviso di sincronizzazione lenta (possibile ritardo webhook) senza bloccare l'utente.

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

**AutoSave:** ogni 6 secondi salva uno snapshot dei valori degli input via `saveDraft()` (solo se utente loggato).

**Keyboard shortcuts:**
- `Ctrl/Cmd + S` → salva progetto su Firestore
- `Escape` → chiude tutti i modali attivi

---

## 9. Security Fixes Implementati

| Fix ID | Area | Descrizione |
|---|---|---|
| **A1** | Rate limiting | Counter `dailyOps` spostato da `localStorage` a Firestore con vincolo incremento +1 per Security Rule. Non bypassabile dalla console del browser. |
| **B1** | Piano PRO | Campi `plan`, `isPro`, `joined`, `version` protetti da scrittura client tramite `notTouchingSensitiveFields()`. |
| **B2** | Stripe data | `subscriptions` e `payments` sono in sola lettura per il client. La creazione di pagamenti finti per ottenere stato PRO è bloccata. |
| **B3** | Success page | `success.html` ascolta Firestore in real-time e sblocca l'UI **solo** quando `plan:'pro'` è confermato. Include timeout a 20s per ritardi webhook. |
| **B4** | Success page | `success.html` verifica lo stato PRO reale prima di mostrare conferma. La navigazione diretta all'URL non bypassa il controllo. |
| **C1** | Admin | `isAdmin()` legge il documento reale in `admins/{uid}`, non un'email hardcoded in CONFIG. I client non possono fare `list` sulla collection `admins`. |
| **D1** | Export | L'export Excel genera un vero file `.xlsx` con SheetJS, non un CSV con estensione rinominata. |
| **D2** | Aritmetica | Decimal.js applicato sia nel modale ammortamento che nel grafico proiezione per coerenza numerica. |
| **D4** | Ammortamento | Rilevamento e blocco di piani insostenibili (rata < interessi del primo mese). |

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
        plans: {
            monthly:  { priceId: "price_1T0KhVCP1ytB9mXWR6roA7uB", amount: 2990 },
            lifetime: { priceId: "price_1T0KhVCP1ytB9mXW9kBSKwkI", amount: 19900, mode: 'payment' }
        }
    },
    maxFreeOps: 5,
    version:    "2.1.0"
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

Nessun client può scrivere in `admins` — solo Firebase Console o Admin SDK.

### Applicare le Security Rules
```bash
# Via Firebase CLI
firebase deploy --only firestore:rules

# Oppure: Firebase Console → Firestore → Regole → incolla e pubblica
```

### Aggiungere un Price Stripe
1. Creare il Price in Stripe Dashboard
2. Aggiornare `CONFIG.stripe.plans` in `index.html` con il nuovo `priceId`
3. Verificare che la Stripe Firebase Extension sia configurata per ascoltare quel Price ID

### Espandere CITY_DB
Aggiungere voci all'array `CITY_DB` in `index.html` seguendo la struttura:
```javascript
{ name: "NomeCittà", region: "Regione", touristTax: 0.00 }
```

---

## 13. Aree di Sviluppo Futuro (Gap Identificati)

Queste aree sono assenti nella versione attuale e rappresentano i punti di intervento prioritari:

1. **Nessuna Cloud Function custom** — tutta la logica di promozione PRO è delegata alla Stripe Extension. Se la Extension si rompe o viene deprecata, il sistema di upgrade si interrompe senza fallback.

2. **`customers/{uid}` scritto dal client** — al login, se il documento non esiste, il client crea `customers/{uid}` con solo il campo `email`. La Security Rule attuale non lo consente (`allow write: if false`). Questo è un conflitto documentato nel codice; richiede o una Cloud Function `auth.onCreate` o una regola di eccezione per la creazione iniziale.

3. **Nessuna validazione server-side degli input di calcolo** — tutti i calcoli avvengono lato client. Un utente mal intenzionato può manipolare i valori degli input nel DOM prima del salvataggio. I progetti salvati in Firestore non vengono ri-validati.

4. **CITY_DB hardcoded** — 15 città statiche. Per scalare, richiederebbe una collection Firestore `cities` con query server-side.

5. **Export PDF tramite screenshot DOM** — html2canvas produce output di qualità variabile su schermi ad alta densità e non è accessibile (non testo selezionabile). Una generazione PDF strutturata lato server o tramite PDF template sarebbe superiore.

6. **Nessun sistema di webhook audit** — non esiste un log degli eventi Stripe ricevuti. In caso di pagamenti falliti o rimborsi, non c'è traccia applicativa indipendente da Stripe Dashboard.

7. **Autenticazione single-provider** — solo Google Sign-In. Email/password non è implementato.

## 14. Ambiente di Sviluppo e Restrizioni Tecniche

* **Nessun Build Tool:** L'applicazione non utilizza Node.js, Webpack, Vite o altri bundler. Tutto il codice è eseguito direttamente nel browser.
* **Gestione Librerie:** Tutte le dipendenze sono importate via CDN tramite tag `<script>`.
* **Moduli ES6 vietati:** Non utilizzare sintassi `import/export`. L'architettura si basa su script globali e sull'oggetto namespace `app`.
* **Vanilla JS Assoluto:** Non suggerire l'introduzione di framework reattivi (React, Vue, Alpine.js) né librerie come jQuery. Il DOM deve essere manipolato esclusivamente con API native (es. `document.getElementById`, `addEventListener`).

## 15. Istruzioni Operative per l'IA

Quando ti viene richiesto di analizzare, debuggare o scrivere nuovo codice per InvestPro, devi attenerti rigorosamente a queste direttive:

1.  **Aderenza allo Stack:** Scrivi solo codice compatibile con le versioni dichiarate nella Sezione 2. Utilizza rigorosamente la sintassi Firebase Compat (`firebase.auth()`, `firebase.firestore()`).
2.  **Integrazione nell'Oggetto "app":** Qualsiasi nuova logica di business o di interfaccia deve essere incapsulata all'interno dell'oggetto globale `app` già esistente nel file `index.html`. Non creare variabili o funzioni globali esterne.
3.  **Preservazione di Decimal.js:** Qualsiasi operazione finanziaria o modifica al motore di calcolo deve utilizzare i metodi della libreria `Decimal.js`.
4.  **Output del Codice:** Quando fornisci codice aggiornato, restituisci solo il blocco o la funzione modificata con un breve contesto di dove inserirla. Non riscrivere l'intero file `index.html` a meno che non ti venga esplicitamente richiesto, per evitare troncamenti nell'output.
5.  **Rispetto delle Security Rules:** Prima di proporre una query o una scrittura su Firestore, verifica mentalmente che non violi le regole definite nella Sezione 5 e nel file `firestore.rules`.
