# 01 — Architecture (HLD)

Status: Draft v0.1 · Depends on: 00-overview

## 1. System view

```
 Client website            WhatsApp user           Client staff
 (widget, JS)                    │                  (dashboard, React)
      │                          │                        │
      └──────────────┬───────────┴────────────────────────┘
                     ▼
            API gateway layer (FastAPI)
            auth · tenant resolution · rate limits
                     │
      ┌──────────────┼──────────────────────────┐
      ▼              ▼                          ▼
 Chat service   Admin API                 Webhook receivers
 (SSE/HTTP)     (config, leads, usage)    (WhatsApp, Shopify)
      │
      ▼
 Agent orchestrator (LangGraph + LLM via LiteLLM)
      │
      ├── Knowledge/RAG module ──► Postgres + pgvector
      ├── Connector layer ───────► Client backend APIs (live, per-user)
      ├── Lead capture / handoff ► DB + notifications + webhooks
      └── Guardrails (input/output checks)

 Background workers (Celery/ARQ): crawl, chunk, embed, webhooks, notifications
 Data: Postgres (state + vectors) · Redis (sessions, rate limits, queue) · S3 (uploads) · Secrets/KMS
```

## 2. Components

| Component | Responsibility | Tech |
|---|---|---|
| Widget | Chat UI injected into client sites; creates sessions, streams replies | TypeScript, Shadow DOM, SSE |
| Dashboard | Tenant self-service: bot config, knowledge, APIs, leads, usage | React/Next.js |
| API gateway layer | AuthN/Z, tenant context, rate limiting, request IDs | FastAPI middleware |
| Chat service | Session lifecycle, message persistence, streaming | FastAPI |
| Agent orchestrator | Decides what to do per message; runs tools | LangGraph |
| Knowledge/RAG | Ingest, chunk, embed, retrieve (tenant-filtered) | pgvector, embeddings API |
| Connector layer | Safe HTTP calls to client APIs with the right auth | httpx, tool registry |
| Lead & handoff | Capture leads, mark handoff, notify | DB, email, webhooks |
| WhatsApp adapter | Receive/send messages, OTP verification flow | Meta Cloud API |
| Workers | Async jobs | Celery or ARQ |
| Observability | Traces, logs, metrics, cost | Langfuse/LangSmith, Sentry, Prometheus |

## 3. Key flows

### 3.1 Web chat, anonymous visitor (public knowledge only)
1. Widget calls `POST /v1/widget/sessions` with `widget_key` (origin checked against `bots.allowed_domains`).
2. Server creates `end_users` (anonymous) and `conversations`, returns a short-lived **session token**.
3. Widget sends messages to `POST /v1/chat/messages` with the session token; reply streams over SSE.
4. Agent answers from RAG; private-data tools are unavailable (`is_authenticated = false`).

### 3.2 Web chat, logged-in user (private data)
1. Client backend signs a short-lived **user JWT** (`tenant_id`, `sub` = their user id, `exp`) with a key registered in `tenant_keys`.
2. Widget passes it at init; `POST /v1/widget/sessions` verifies signature, tenant, expiry, `kid`.
3. Server links/creates `end_users` by `(tenant_id, external_user_id)`, marks session authenticated, and keeps the original token **server-side only** (Redis, TTL) for forward-token mode.
4. When the agent calls a tool, backend code injects the verified user context. The LLM never supplies identity.
5. Connector calls the client API, validates and trims the response, returns it to the agent, which writes the reply.

### 3.3 WhatsApp
1. Meta posts to `POST /v1/webhooks/whatsapp`; signature verified; `channels.external_id` (phone number ID) resolves tenant, bot, channel.
2. Sender's number resolves to `end_users.channel_identity`. Unverified users get public-knowledge answers only.
3. For private data the agent triggers verification: OTP sent (via the client's API or WhatsApp), stored hashed in `otp_verifications`, user replies with code, `is_verified = true`.
4. Reply sent via Cloud API. Outside the 24-hour window only approved templates may be sent.

### 3.4 Knowledge ingestion
1. Admin adds a URL/file/text in the dashboard → `knowledge_sources` row (`pending`).
2. Worker fetches, cleans, chunks, embeds → `knowledge_chunks`; source becomes `ready` or `failed` with an error.
3. Re-sync recomputes `content_hash` and replaces changed chunks only.

### 3.5 Lead capture and handoff
1. Agent calls `capture_lead` (name, email/phone, interest) when intent is clear, with user consent in the chat.
2. Lead stored; `lead.created` webhook and email dispatched by worker.
3. On handoff (user request, low confidence, or sensitive topic): conversation `status = handoff`, owner alerted with a transcript link.

## 4. Agent design summary (detail in 06)

- One LangGraph graph: input guardrail → route (answer / tool / lead / handoff) → generate → output guardrail.
- Tools are generated from `api_tools` plus built-ins (`search_knowledge`, `capture_lead`, `request_handoff`).
- Per-turn context: bot config, retrieved chunks, last N messages, verified user context (never editable by the model).
- Hard limits: max tool calls per turn, per-tool timeout, per-tenant token budget.

## 5. Key decisions (ADR-lite)

| # | Decision | Rationale | Trade-off |
|---|---|---|---|
| D1 | Modular monolith (FastAPI) plus workers | Small team, simpler ops | Split services later if needed |
| D2 | Postgres + pgvector for relational and vector data | One datastore, tenant filters in one query | Dedicated vector DB at scale |
| D3 | LLM-agent-driven (no flow builder) | Fast setup, low scope | Needs guardrails and evals |
| D4 | Private data fetched live, never stored | Privacy, no sync issues | Latency depends on client API |
| D5 | Row-level security with `app.tenant_id` | Defence in depth for isolation | Extra care with system lookups |
| D6 | SSE for streaming (not WebSocket) | Simpler, proxy-friendly | One-way only; fine for chat |
| D7 | LiteLLM provider abstraction | Fallback and cost control | Extra dependency |
| D8 | Session token issued by us; original user JWT kept server-side | Token not re-exposed, supports forward-token mode | Needs Redis state |

## 6. Non-functional requirements

| Area | Requirement |
|---|---|
| Availability | Target 99.5% monthly; health checks; LLM fallback provider |
| Latency | p95 first token < 3 s; tool call timeout default 6 s |
| Scalability | Stateless API pods; scale horizontally; workers scale independently |
| Isolation | Zero cross-tenant reads (RLS plus tests) |
| Cost | Per-tenant token budgets and rate limits |
| Observability | Request ID on every log; trace per conversation turn; cost per tenant |
| Compliance | DPDP-aware retention and deletion (see 04) |

## 7. Failure handling

| Failure | Behaviour |
|---|---|
| Client API timeout/5xx | Friendly message, no retry for writes, one retry for GET, circuit breaker per connection |
| LLM provider error | Fallback model; if both fail, apology plus offer handoff/lead capture |
| Vector search empty | Say "I don't know", offer handoff |
| Invalid/expired user token | Treat as anonymous, prompt login for private data |
| WhatsApp send failure | Retry with backoff; mark message failed; alert |
| Worker crash | Idempotent jobs, retries, dead-letter queue |

## 8. Environments

`local` (docker-compose: api, worker, postgres, redis) → `staging` → `production`. Config via environment variables only (see 02).
