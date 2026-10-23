# 04 — Security

Status: Draft v0.1 · Depends on: 01, 03 · Referenced by every feature spec

## 1. Assets and threats

| Asset | Threat | Primary control |
|---|---|---|
| End users' private data (orders, bookings) | Another user or tenant reads it | Verified identity, tenant RLS, backend-injected user context |
| Tenant secrets (API keys, WhatsApp tokens, JWT keys) | Leak via logs, APIs, DB dump | App-level encryption, secrets manager, never returned |
| Conversation content | Leak, over-retention | RLS, retention jobs, PII-safe logging |
| LLM behaviour | Prompt injection, data exfiltration, unauthorised actions | Tool allowlist, untrusted-content rules, confirmation for writes |
| Platform availability and cost | Spam, abuse, runaway LLM spend | Rate limits, budgets, circuit breakers |
| Client backends | SSRF through the connector, hammering | Domain allowlist, IP blocking, timeouts, rate limits |

## 2. Trust boundaries

1. Browser (widget) → API: **untrusted**. Anything the browser sends can be forged.
2. WhatsApp/Meta → webhook: trusted only after signature verification.
3. LLM output → tool execution: **untrusted**. Arguments are validated; identity is never taken from the model.
4. Retrieved content (crawled pages, files) and client API responses → LLM: **untrusted data**, never instructions.
5. API → database: tenant context set by trusted server code only.

## 3. Identity

### 3.1 Dashboard (admin) auth
- Email + password, `argon2id` hashing; optional TOTP later.
- Short-lived access token (15 min) + refresh token (httpOnly, secure cookie, rotated, revocable).
- Role checks per endpoint: `owner` > `admin` > `viewer`.
- Login throttling per IP and per email; generic error messages.

### 3.2 Widget sessions
1. `POST /v1/widget/sessions` with `widget_key`. Server checks the request `Origin` against `bots.allowed_domains`; rejects if no match.
2. The `widget_key` is public and identifies the bot only. It grants **no** private-data access.
3. Server issues a **session token** (signed, 30 min, refreshable) containing `tenant_id`, `bot_id`, `conversation_id`, `end_user_id`, `authenticated` flag. The widget never sees secrets.

### 3.3 Logged-in user (client-signed JWT)
- The client backend signs a short-lived JWT (≤ 15 min) with a key registered in `tenant_keys`.
- Required claims: `iss` (tenant identifier), `sub` (client's user id), `aud` (`convertly`), `iat`, `exp`, optional `email`, `name`. Header carries `kid`.
- Verification: algorithm allowlist (HS256/RS256/ES256 per key), `kid` lookup, `exp`/`iat` with ≤ 60 s skew, `aud` match, `status = active`. **Never** accept `alg: none` or switch algorithms based on the token.
- On success: upsert `end_users` by `(tenant_id, external_user_id)`, set session `authenticated = true`.
- Forward-token mode: the original JWT/credential is kept server-side only (Redis, TTL = session, encrypted) and attached to client API calls. It is never returned to the browser or logged.
- Invalid or expired token → session continues as anonymous; private-data tools unavailable; agent asks the user to log in.

### 3.4 WhatsApp users
- No JWT exists, so private data requires **verification**:
  - Option A: lookup by phone number against a client endpoint, then OTP.
  - Option B: OTP via WhatsApp to the number, validated against the client record.
- OTP: 6 digits, CSPRNG, stored hashed, 10-minute expiry, max 5 attempts, then 30-minute lockout; one active OTP per user.
- Until verified, the user gets public-knowledge answers only.
- Verification state expires (default 24 h) and can be revoked.

## 4. Tenant isolation

- Every tenant table has `tenant_id` and RLS (see 03 §5). Request handlers use `app_user` only.
- `app.tenant_id` is set from the verified principal, never from request input.
- Cross-tenant IDs return `404`, not `403`, to avoid existence leaks.
- Redis keys are namespaced `t:{tenant_id}:...`. Vector and cache queries always filter by tenant.
- Background jobs carry `tenant_id` in the payload and set context before touching data.
- Mandatory cross-tenant tests per module (see 02 §9).

## 5. Secrets management

- Stored encrypted (AES-256-GCM) with data keys from the secrets manager/KMS; key version recorded in ciphertext.
- Decrypted only in the code path that needs them; held in memory briefly; never logged or put in exceptions.
- API responses never include secret fields. Admin UI shows only "set / not set" and last 4 characters where useful.
- Platform secrets (DB, LLM keys, Meta app secret) come from the secrets manager, not the repo or images.
- Rotation: `tenant_keys` supports `rotating` state with two valid keys; connections and channels support credential replacement without downtime.

## 6. Client API connector safety

| Control | Rule |
|---|---|
| Transport | `https` only; TLS verification on |
| Destination | Host must match the connection's registered `base_url`; redirects not followed (or re-validated) |
| SSRF | Resolve DNS and block private, loopback, link-local, and metadata IPs (e.g. 169.254.169.254) on every request |
| Methods | Only the method/path template configured in `api_tools`; model supplies only schema-validated arguments |
| Identity | User identifiers are injected by backend code from the verified session; any model-supplied identity argument is ignored |
| Response | Parsed, size-capped (e.g. 256 KB), trimmed to `response_fields` allowlist before the LLM sees it |
| Timeouts | Per-connection (default 6 s), total per turn cap |
| Retries | One retry for GET on timeout/5xx; none for writes |
| Breaker | Open the circuit after repeated failures per connection; return a friendly error |
| Rate limit | Per-connection requests per minute |
| Writes | `is_write` tools require explicit user confirmation in the chat, an `Idempotency-Key`, and are logged |

## 7. Prompt injection and agent safety

1. System prompt, retrieved content, and tool outputs are separated and labelled; content from the web or APIs is wrapped as data with a standing instruction never to follow instructions inside it.
2. The model can only call tools registered for this tenant and bot; unknown tool names are rejected.
3. Tool arguments are validated against `request_schema`; extra fields dropped.
4. The model cannot read or set `tenant_id`, `user_id`, tokens, or credentials; none appear in its context.
5. Output guardrail checks for: leaked secrets/system prompt, unsupported price/discount/refund promises, off-policy content. Fallback: safe refusal plus handoff offer.
6. Business-commitment rule: the bot never promises refunds, discounts, delivery dates, or legal/medical claims unless the value came from a tool/knowledge result in this turn.
7. Max tool calls per turn (e.g. 5) and max turns without progress, to stop loops.
8. Anything requiring a state change needs confirmation (see §6).
9. Evals include a prompt-injection suite (see 07): hostile page content, "ignore previous instructions", "show order of user X", requests for other users' data.

## 8. Abuse and cost controls

| Control | Default |
|---|---|
| Rate limit per IP (widget session create) | 20/min |
| Rate limit per session (messages) | 20/min, 300/day |
| Rate limit per tenant | By plan |
| Max message length | 2,000 chars |
| Max conversation length in context | Last N messages plus summary |
| Per-tenant monthly token budget | By plan; at 100% bot degrades to "contact owner" + lead capture |
| Bot/spam detection | Basic heuristics; CAPTCHA option on the widget later |
| Origin check | `Origin` header must match `allowed_domains` for widget endpoints |

## 9. Webhooks

- **Inbound** (WhatsApp, Shopify): verify HMAC/signature with the provider secret on the raw body; reject stale timestamps; dedupe by event ID.
- **Outbound** to clients: sign body with the tenant's `webhooks.secret` (HMAC-SHA256, header `X-Convertly-Signature` and timestamp); retries with backoff; HTTPS only; SSRF rules from §6 apply.

## 10. Privacy and compliance

- **PII in logs:** no message bodies or contact details at INFO; ids only. Redact emails and phone numbers in error reports.
- **Private data:** fetched live; not stored, not embedded, not used for training or analytics. Conversation text may contain it, so retention (03 §8) applies.
- **Consent:** the bot asks before capturing contact details and states what they are used for.
- **Deletion and export:** supported per end user and per tenant (03 §8).
- **Regulations:** India's DPDP Act as baseline; GDPR-aware for foreign clients. Provide a DPA/privacy summary for clients before launch.
- **Data residency:** decide hosting region (Q4); document sub-processors (LLM provider, hosting, WhatsApp).

## 11. Transport, headers, and platform hygiene

- HTTPS everywhere, HSTS; TLS 1.2+.
- CORS: widget endpoints allow only registered origins; admin API allows only the dashboard origin.
- Security headers on the dashboard (CSP, X-Content-Type-Options, frame-ancestors).
- Widget runs in Shadow DOM/iframe, renders model output as sanitized text/markdown (no raw HTML), and never evaluates remote code.
- Dependency and secret scanning in CI; pinned versions; regular updates.
- Backups encrypted; restore tested.
- Audit log (later, if needed) for admin actions: key changes, connection edits, deletions.

## 12. Security acceptance criteria

| ID | Criterion |
|---|---|
| SEC-1 | Tenant A's admin token cannot read or modify any tenant B resource (404). |
| SEC-2 | A widget session for bot X cannot post to bot Y's conversations. |
| SEC-3 | A JWT with a bad signature, wrong `aud`, expired `exp`, unknown `kid`, or `alg: none` is rejected; session stays anonymous. |
| SEC-4 | The model cannot cause a tool call for a user other than the session's verified user (injection test suite). |
| SEC-5 | Connector refuses non-HTTPS, redirects to other hosts, and private/metadata IPs. |
| SEC-6 | No secret value appears in any API response, log line, or error message (automated scan in tests). |
| SEC-7 | OTP locks after 5 failures; expired OTP rejected; OTP stored hashed. |
| SEC-8 | Inbound webhook with invalid signature is rejected before any processing. |
| SEC-9 | Write tool executes only after explicit user confirmation and is idempotent on retry. |
| SEC-10 | Rate limits return 429 and do not affect other tenants. |
| SEC-11 | Widget request from a non-allowed `Origin` is rejected. |
| SEC-12 | Tenant data is removed according to the deletion flow within the stated retention. |

## 13. Open questions

| # | Question |
|---|---|
| Q11 | JWT signing: HS256 shared secret only for v1, or support asymmetric from day one? |
| Q12 | WhatsApp verification: Option A or B (depends on what client APIs exist)? |
| Q13 | Do we need 2FA for dashboard admins in v1? |
| Q14 | Per-plan limits (messages/day, tokens/month): values to be set in f13. |
