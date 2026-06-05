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
