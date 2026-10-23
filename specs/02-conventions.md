# 02 — Conventions

Status: Draft v0.1 · Applies to all code

## 1. Stack

| Area | Choice |
|---|---|
| Language | Python 3.12 (backend), TypeScript 5 (widget, dashboard) |
| API | FastAPI, Pydantic v2, async everywhere |
| ORM / migrations | SQLAlchemy 2.x (async) + Alembic |
| DB | PostgreSQL 16, extensions: `pgcrypto`, `vector`, `citext` |
| Cache / queue | Redis 7; Celery or ARQ (decide in f01, then fixed) |
| Agent | LangGraph, LangChain (only where useful), LiteLLM for providers |
| HTTP client | httpx (async) |
| Auth libs | PyJWT, Authlib (OAuth), argon2-cffi |
| Widget | TypeScript, esbuild bundle, Shadow DOM, no heavy framework |
| Dashboard | Next.js (React), Tailwind |
| Tests | pytest, pytest-asyncio, httpx test client, testcontainers (Postgres/Redis) |
| Lint / types | ruff, mypy (strict on `app/`), eslint, tsc |
| Packaging | uv or poetry (decide in f01), Docker |

## 2. Repository layout

```
convertly/
  CLAUDE.md
  specs/
  backend/
    app/
      main.py
      core/          # config, logging, security, errors, deps
      db/            # base, session, models/, migrations via alembic/
      modules/
        tenancy/     # tenants, admin users, auth
        keys/        # tenant_keys, user-token verification
        chat/        # conversations, messages, SSE
        knowledge/   # sources, ingestion, retrieval
        agent/       # graph, tools, prompts, guardrails
        connectors/  # api_connections, api_tools, HTTP caller
        leads/       # leads, webhooks, notifications
        channels/    # web, whatsapp adapters
        usage/       # metering, limits
      workers/       # background job definitions
    tests/
      unit/ integration/ evals/
    alembic/
    pyproject.toml
  widget/            # TypeScript embeddable widget
  dashboard/         # Next.js admin app
  deploy/            # docker-compose, Dockerfiles, CI
```

Each module has: `router.py`, `schemas.py` (Pydantic), `service.py` (business logic), `repo.py` (DB access), `models.py` (SQLAlchemy). Routers stay thin; logic lives in services.

## 3. Naming

- DB: `snake_case`, plural table names, `id uuid`, `tenant_id uuid`, `created_at`/`updated_at timestamptz`.
- Python: `snake_case` functions, `PascalCase` classes; Pydantic schemas end with `In` / `Out` (e.g. `LeadIn`, `LeadOut`).
- Endpoints: `/v1/<resource>` plural nouns, kebab-case for multi-word paths.
- Env vars: `UPPER_SNAKE_CASE`, prefixed by area (`DB_`, `REDIS_`, `LLM_`, `WHATSAPP_`).
- Agent tool names: `snake_case`, verb first (`get_order_status`).

## 4. API conventions

- JSON in/out, UTF-8. Version prefix `/v1`.
- Auth: `Authorization: Bearer <token>`. Three principal types: **admin** (dashboard), **widget session**, **system** (webhooks via signatures).
- Request ID: accept `X-Request-ID` or generate; return it; include in every log line.
- Pagination: cursor-based (`?limit=50&cursor=...`), response `{ "items": [], "next_cursor": null }`.
- Idempotency: write endpoints that may be retried accept `Idempotency-Key`.
- Streaming: SSE (`text/event-stream`) with events `token`, `tool_status`, `done`, `error`.
- Status codes: 200/201/204 success; 400 validation; 401 unauthenticated; 403 forbidden; 404 not found (also used for cross-tenant ids); 409 conflict; 422 schema errors; 429 rate limited; 5xx server.

### Error format (all endpoints)

```json
{
  "error": {
    "code": "validation_error",
    "message": "Human-readable summary",
    "details": [{"field": "email", "issue": "invalid"}],
    "request_id": "01J..."
  }
}
```

`code` values are stable, lowercase snake_case. Never leak stack traces, SQL, or secrets.

## 4a. Tenant context rule

Every request resolves a `TenantContext(tenant_id, principal)` in a dependency. The DB session is created with `SET LOCAL app.tenant_id = '<uuid>'` so row-level security applies (see 03, 04). Code must not accept `tenant_id` from request bodies for scoping.

## 5. Configuration

- `pydantic-settings`, env vars only, validated at startup (fail fast).
- `.env.example` lists every variable with comments; real `.env` is gitignored.
- No secrets in code, images, or logs. Production secrets from the secrets manager.

## 6. Logging and errors

- Structured JSON logs: `ts`, `level`, `request_id`, `tenant_id`, `conversation_id` (when present), `event`, plus fields.
- Never log: passwords, tokens, API keys, full message bodies at INFO (message content only at DEBUG in non-prod), PII beyond ids.
- Use typed domain exceptions mapped to error codes in one handler.
- Metrics: request latency, error rate, LLM tokens and cost per tenant, tool-call success rate, queue depth.

## 7. Database access rules

- All DB access goes through `repo.py` functions using the tenant-scoped session.
- Cross-tenant/system queries (e.g. resolving `widget_key`, WhatsApp `phone_number_id`) live in one module `db/system_queries.py` using the restricted system role, and are individually reviewed.
- Migrations: one Alembic revision per logical change; must be reversible; never edit an applied revision.
- No raw SQL string concatenation; parameters only.

## 8. LLM usage rules

- All calls go through one `llm` client wrapper (LiteLLM) with timeouts, retries, token accounting, and tracing.
- Prompts live in versioned files under `modules/agent/prompts/`, not inline strings.
- Structured outputs validated with Pydantic; reject and retry once on invalid output.
- Model names come from config, never hard-coded in logic.

## 9. Testing rules

- Unit tests for services; integration tests with real Postgres/Redis containers; API tests via the test client.
- Every acceptance criterion in a feature spec maps to a named test (`test_<ac_id>_...`).
- **Mandatory cross-tenant test** for every module that reads or writes tenant data.
- Agent evals live in `tests/evals/` (see 07); they run separately from unit tests.
- External services (LLM, WhatsApp, client APIs) are mocked in unit/integration tests.

## 10. Git and review

- Trunk-based with short-lived branches: `feat/f02-tenancy`, `fix/...`.
- Conventional commits (`feat:`, `fix:`, `docs:`, `test:`, `chore:`).
- One feature spec = one PR (or a small number of PRs). PR description lists the spec, acceptance criteria covered, and deviations.
- CI: lint, type check, tests, migration check, secret scan.
