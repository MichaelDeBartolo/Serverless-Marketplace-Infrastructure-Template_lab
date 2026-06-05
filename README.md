# 🚀 Serverless Marketplace Infrastructure Template

Questo repository contiene l'architettura infrastrutturale backend, serverless ed **Event-Driven** per un marketplace generico ad alte prestazioni, scalabile ed *"Edge-first"*. Il progetto è nativamente focalizzato sulla sicurezza granulare del dato, sull'isolamento multi-tenant e sulla resilienza dei flussi logistici e di pagamento automatizzati tramite terze parti.

---

## 🛠️ Stack Tecnologico & Architettura

L'applicazione è ingegnerizzata per azzerare i costi di gestione infrastrutturale fissa (**Scale-to-Zero**) e massimizzare la velocità di risposta globale:

* **Frontend:** Next.js / React (Ospitato su *Vercel Edge Network* per massimizzare il Time to First Byte).
* **Database & Cloud BaaS:** Supabase (PostgreSQL relazionale con estensioni attive come `pgcrypto` e `uuid-ossp`).
* **Serverless Compute:** Supabase Edge Functions (TypeScript strutturato su runtime *Deno* ad altissima velocità, isolamento V8 e bassa latenza).
* **Integrazioni API:** Stripe API (Core dei pagamenti e infrastruttura di fatturazione) & REST Shipping API Wrapper (Automazione logistica multi-corriere).

---

## 🔄 Flusso dei Dati ed Ingegneria delle API

L'architettura sfrutta i Webhook e le funzioni serverless per garantire transazioni atomiche, asincrone e resistenti ai guasti di rete.

```text
[Client / Vercel UI] ──(Inizializza sessione)──> [Edge Function: create-checkout]
                                                            │
                                                     (Genera Sessione)
                                                            ▼
                                                   [Stripe Hosted UI]
                                                            │
                                                     (Invia Webhook)
                                                            ▼
[Shipping API] <──(Genera Etichetta)── [Edge Function: stripe-webhook]
                                                            │
                                                 (Se errore API Corriere)
                                                            ▼
[Dashboard Admin] <──(Forzatura Manuale)── [Edge Function: retry-shipping]

🛰️ Dettaglio dei Processi di Integrazione API
1. Orchestrazione del Pagamento (create-checkout)
L'endpoint riceve le specifiche degli articoli (ID inserzione, prezzo e metadati del venditore). La funzione:

Valida i prezzi direttamente lato database, impedendo la manomissione dei prezzi lato client.

Interroga le policy di spedizione e istanzia una sessione crittografata tramite l'SDK ufficiale di Stripe.

Restituisce un URL sicuro di reindirizzamento, riducendo l'esposizione del frontend ai dati sensibili di pagamento (Conformità PCI-DSS).

2. Gestione degli Eventi e Idempotenza (stripe-webhook)
L'endpoint agisce come orchestratore asincrono centrale. All'evento checkout.session.completed:

Fase Atomica 1 (Database): Aggiorna lo stato dell'inserzione da active / reserved a sold all'interno di una transazione PostgreSQL per bloccare l'asset ed evitare condizioni di race condition (double spending dell'oggetto).

Fase Atomica 2 (Integrazione Logistica REST): Effettua una chiamata POST asincrona verso l'API del provider logistico strutturando il payload con i dati di spedizione estratti dai metadati della sessione Stripe.

3. Tolleranza ai Guasti e Disaccoppiamento (retry-shipping)
Le API esterne dei corrieri logistici presentano tassi di latenza elevati o down temporanei. Per evitare il blocco del webhook di Stripe (che richiede una risposta HTTP 200 entro un timeout rigido), l'architettura implementa un pattern di Circuit Breaker / Fallback:

Se l'API del corriere risponde con un codice di errore (4xx o 5xx) o va in timeout, la Edge Function intercetta l'eccezione, committa l'ordine sul database contrassegnandolo come payment_confirmed ma valorizza il flag shipping_status su failed_api_pending.

L'amministratore del sistema può invocare l'endpoint isolato retry-shipping. Questa funzione recupera i payload originari archiviati, riesegue l'autenticazione verso i server logistici e forza la generazione asincrona della lettera di vettura senza alterare lo stato del pagamento Stripe associato.

🔒 Cybersecurity, Hardening & Verifica delle Firme
Validazione Crittografica dei Webhook: La funzione stripe-webhook implementa una strategia di hardening contro attacchi di spoofing. Estrae l'header stripe-signature e calcola l'HMAC con algoritmo SHA256 utilizzando la chiave segreta STRIPE_WEBHOOK_SECRET. Qualsiasi richiesta non firmata dall'autorità di Stripe viene rigettata istantaneamente con un codice HTTP 401 Unauthorized.

Prevenzione dagli Attacchi Replay: Il meccanismo di convalida verifica i timestamp all'interno dell'header di firma Stripe, scartando payload validi ma duplicati oltre una tolleranza di 300 secondi.

Isolamento delle API di Emergenza via Entropy Secret: Poiché l'endpoint retry-shipping viene invocato al di fuori del ciclo di sessione standard dell'utente, l'accesso è protetto bypassando il gateway JWT tradizionale per evitare token scaduti durante operazioni batch. La protezione è demandata all'ispezione obbligatoria dell'header custom x-retry-secret, validato direttamente sull'Edge con un segreto ad alta entropia (256-bit) estratto dal Vault cifrato di Supabase.

PostgreSQL Row Level Security (RLS): Il database applica policy RLS native su ogni tabella. L'accesso alle API lato client non permette in nessun caso l'iniezione di query esternne o l'esfiltrazione di dati di ordini altrui (Mitigazione delle vulnerabilità OWASP Broken Object Level Authorization - BOLA).

📂 Struttura del Repository
Plaintext
├── .supabase/                 # Configurazioni locali dell'ambiente Supabase
├── supabase/
│   ├── functions/             # Core Backend (Supabase Edge Functions su runtime Deno)
│   │   ├── create-checkout/   # Inizializzazione transazioni sicure Stripe
│   │   ├── stripe-webhook/    # Gestione eventi di pagamento, idempotenza e trigger logistica
│   │   └── retry-shipping/    # Gateway di ripristino, gestione errori e fallback API
│   ├── migrations/            # Script SQL di migrazione, schemi DDL e policy RLS
│   └── config.toml            # Configurazione sistemistica e allocazione risorse funzioni
├── src/                       # Struttura Frontend generica (UI / Componenti / Gestione API Client)
└── .env.example               # Template per l'estrazione sicura delle variabili d'ambiente
⚙️ Configurazione delle Variabili d'Ambiente
Il progetto aderisce alla metodologia Twelve-Factor App: nessuna chiave o credenziale è hardcoded nel codice sorgente.

Bash
# --- STRIPE GATEWAY CONFIGURATION ---
STRIPE_SECRET_KEY=sk_live_...
VITE_STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# --- LOGISTICS API CREDENTIALS ---
SHIPPING_API_KEY=your_api_key_here
SHIPPING_API_TOKEN=your_token_here
SHIPPING_API_BASE_URL=[https://api.provider.com/v1](https://api.provider.com/v1)

# --- SYSTEM SECURITY & EMERGENCY RECOVERY ---
RETRY_SHIPPING_SECRET=un_segreto_ad_alta_entropia_memorizzato_nel_vault
🎯 Considerazioni Finali & AI-Driven Engineering
Questo repository rappresenta una solida dimostrazione di ingegneria del software moderna ed "Edge-First". L'intero ciclo di progettazione, architetturazione delle API, refactoring delle Edge Functions e hardening della sicurezza è stato condotto applicando metodologie avanzate di AI-Driven Development (attraverso l'uso strategico di prompt ingegneristici complessi e l'orchestrazione dell'ambiente di sviluppo integrato tramite modelli di IA di ultima generazione).

💡 Nota sulla Proprietà Intellettuale & Logica di Business
Astrazione del Codice: Per motivi di riservatezza commerciale e protezione del posizionamento strategico sul mercato, questo repository non contiene le logiche di business proprietarie sensibili, i moduli specifici di messaggistica asimmetrica in tempo reale per le trattative private, gli algoritmi di determinazione dei prezzi degli asset o l'interfaccia utente finale del brand.

Scopo del Progetto: Il codice e l'architettura qui presenti hanno scopi puramente dimostrativi ed espositivi per illustrare competenze verticali in ambito Enterprise API Integration, Serverless Cloud Infrastructure (DevOps), Asynchronous System Design e Cybersecurity (Hardening di endpoint e conformità dei dati).
