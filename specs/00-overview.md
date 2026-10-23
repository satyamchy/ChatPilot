# 00 — Overview

Status: Draft v0.1 · Owner: Satya · Working name: Convertly

## 1. Problem

Small websites lose sales and waste time because:

- Visitors leave without getting answers (product, price, shipping, availability).
- Customers can't quickly check their own data (order status, booking, account), so owners answer the same questions repeatedly.
- Leads and abandoned carts are never followed up.
- 24/7 human support is unaffordable, and enterprise tools (e.g. Yellow.ai) are priced and built for large companies.

## 2. Intent

Provide an **embeddable AI assistant layer** that sits on top of a client's existing website and backend. It answers questions from the client's knowledge, fetches **the logged-in user's own data** from the client's APIs, captures leads, and hands off to humans when needed. Clients onboard with a script tag and a dashboard, with no flow building.

Inspired by Yellow.ai's "conversation layer over existing systems" model, with these deliberate differences:

| Yellow.ai | Convertly |
|---|---|
| Enterprise, flow-driven | Small business, LLM-agent-driven |
| Visual flow builder | Dashboard configuration and prompts, no flow builder |
| Support/employee-service heavy | Sales-first (leads, recommendations, cart recovery) |
| Large connector library | Generic REST connector plus Shopify/WooCommerce |

## 3. Personas

| Persona | Description | Main needs |
|---|---|---|
| Site owner / admin | Runs the client business, uses the dashboard | Easy setup, leads, low cost, trust |
| Client developer | Integrates the widget and APIs (optional, for custom sites) | Clear docs, simple token flow |
| End user (visitor/customer) | Chats on the client's site or WhatsApp | Fast, correct answers; own data safely |
| Platform operator (us) | Runs the platform | Isolation, cost control, observability |

## 4. Scope

### In scope (v1)
- Web chat widget (embeddable JS) and WhatsApp channel
- Admin dashboard: bot config, knowledge sources, API connections, leads, conversations, usage
- RAG over client knowledge (URLs, files, text)
- Per-user private data via client API connector (read-first; writes need confirmation)
- Lead capture and webhook/email notification
- Human handoff (owner notified, conversation marked `handoff`)
- Multi-tenancy with strict isolation
- Usage metering and plan limits
- English plus Hindi/Hinglish (quality tested, not guaranteed for other languages)

### Out of scope (v1)
- Voice/telephony, native mobile SDK
- Visual flow builder
- Live-agent inbox inside the dashboard (handoff = notification only)
- Enterprise connectors (SAP, Workday, Salesforce)
- Other channels (Instagram, Telegram, SMS, Teams)
- SOC 2 and similar certifications

### Later (phase 2+)
Shopify/WooCommerce plugins (`f14`), cart-recovery nudges, live-agent inbox, analytics depth, voice.

## 5. Principles

1. **Lean first.** Smallest thing that proves the value; no speculative generality.
2. **Security is structural.** Identity and tenant context come from verified code paths, never from the LLM.
3. **Answers must be grounded.** Facts come from RAG or client APIs; otherwise say "I don't know" and offer handoff.
4. **Private data is fetched live, never stored or embedded.**
5. **Spec before code.** Each feature is specified, reviewed, then implemented.

## 6. Success metrics

| Metric | Target (first pilot) |
|---|---|
| Resolution rate without handoff | ≥ 60% of conversations |
| Lead capture rate | ≥ 5% of conversations produce a lead |
| Grounded-answer accuracy (eval set) | ≥ 90% |
| Cross-tenant data leaks | 0 |
| p95 first-token latency | < 3 s |
| LLM cost per conversation | Tracked per tenant; budgeted per plan |

## 7. Assumptions

- Clients can add a script tag; custom-backend clients have a developer who can sign a JWT.
- Clients for private-data features expose HTTP APIs (REST/JSON).
- WhatsApp is accessed via the Meta Cloud API (or a BSP such as Twilio; decision pending).

## 8. Open questions (resolve before the related feature spec)

| # | Question | Needed by |
|---|---|---|
| Q1 | WhatsApp route: Meta Cloud API direct vs Twilio? | f12 |
| Q2 | Primary LLM provider and fallback model? | 06 |
| Q3 | Pricing model (per message / per tenant / tiers)? | f13 |
| Q4 | Hosting region and data residency requirements? | 08 |
| Q5 | Message retention default (proposed: 90 days)? | 03 |
| Q6 | Embedding model (determines vector dimension)? | 03, f05 |

## 9. Glossary

- **Tenant**: a client business using the platform.
- **Bot**: a configured assistant belonging to a tenant.
- **Channel**: a way users reach a bot (`web`, `whatsapp`).
- **End user**: a visitor or customer of the tenant.
- **Connector / tool**: a configured client API endpoint the agent may call.
- **Handoff**: transferring a conversation to a human.
- **RAG**: retrieval-augmented generation over tenant knowledge.
