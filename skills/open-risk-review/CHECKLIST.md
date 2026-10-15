# Open Risk Review checklist

Version 1.0 · 59 controls in 12 areas.

Every control gets one status in the report:

| Status | Meaning |
|---|---|
| **PASS** | The code does this, and you can cite where. Pattern-matching is not enough: you read the code path. A framework or library default counts when the app relies on it and doesn't turn it off: cite the package, its locked version, the behaviour, and exactly what it covers (for example "Next.js 14.2 server actions check `Origin` against `Host`": server actions only, not route handlers). |
| **PARTIAL** | Done in some places and missing in others, or done in a way that can be bypassed. |
| **FAIL** | Not done, or done wrong, with evidence. |
| **NOT VERIFIABLE** | The control lives outside this repository (the hosting dashboard, the auth or billing provider, a database console, a runbook kept elsewhere). Say where it lives. Never guess. |
| **N/A** | The app has no such surface (no billing, no AI calls, no uploads), or the surface exists but nothing depends on it yet (billing with no paid features to gate). Say why in one line, and what would make it apply. |

PASS means "the code we read does this". It never means "this app is safe".

**When something is missing from the repo**, ask what would have to exist
for the control to pass:

- **FAIL** if it would be code, config, tests, CI, migrations, or a file
  teams keep next to the code (threat model, ADRs, changelog, rollback
  notes). Example: no tests (T1–T5), no down migrations (D4).
- **NOT VERIFIABLE** if it could just as well be a provider setting or a
  record kept elsewhere: a rotation schedule, backups, alert rules, MFA
  policy, key scopes, per-environment values, a host's rollout settings,
  a flag service. Say where it would live, so the owner can check.
- **Both parts?** Rate the part you can see. Step-up re-authentication
  is code even when sign-in MFA is a provider setting, so a missing
  re-auth check is PARTIAL or FAIL, not NOT VERIFIABLE. If the part you
  can see passes and the rest lives elsewhere, mark NOT VERIFIABLE and
  cite what you saw.

This applies to every control. The "usually NOT VERIFIABLE" notes below
are reminders of where things tend to live, not exceptions.

Each control lists what to look for. Stack hints at the end say where these
things usually live in common frameworks.

---

## 1. Auth & sessions

| # | Control | What to look for |
|---|---|---|
| A1 | Auth middleware on all protected routes | List every route, page loader, API handler, server action and RPC method. For each, find the check that rejects an anonymous caller. Auth on `/dashboard` but not `/api/user` is the classic miss. Supabase: server code that decides access from `getSession()` trusts the cookie without checking it; use `getUser()` or `getClaims()`. Queries that go through RLS are still checked by the database. |
| A2 | Session revocation | Logout and password change invalidate the session server-side. Deleting a cookie or clearing local storage alone is not logout. |
| A3 | JWT algorithm allowlisted | Verification pins the algorithm (for example `HS256` or `RS256`). Accepting the algorithm from the token header allows `alg: none` forgery. If the app never verifies tokens itself and asks the auth provider's server instead (Supabase `getUser()`), mark N/A and name the provider. A library that verifies locally (Supabase `getClaims()` with asymmetric signing keys, though on the legacy shared secret it calls the server like `getUser()`; `jose`; most Clerk and Auth0 server SDKs) is verifying: check it pins algorithms or uses the provider's published keys. |
| A4 | MFA for high-privilege actions | Billing changes, user deletion and role changes require a second factor or recent re-authentication. Sign-in MFA is often a provider setting (NOT VERIFIABLE); step-up re-authentication before sensitive actions is code. |
| A5 | OAuth redirect URI allowlisted | Exact-match redirect URIs. Wildcards or prefix matching allow open redirects and token theft. Check any `redirect`, `returnTo`, `next` parameter the app follows. |
| A6 | Cookie flags | Session cookies set `HttpOnly`, `Secure`, `SameSite` and a sensible max-age. Some auth libraries leave cookies readable on purpose because their browser client needs them (`@supabase/ssr`); then `HttpOnly` isn't required, but `Secure` and `SameSite` still are, and any XSS finding becomes more serious because it can read the session. Check what the library actually sets: `@supabase/ssr` (0.5.1 and later) defaults to `SameSite=Lax` and a 400-day max-age with no `Secure`, so look for `secure` in `cookieOptions` passed to `createServerClient` / `createBrowserClient` or in the app's `setAll` cookie handler. |
| A7 | Admin gating | Admin and platform-wide actions check a platform role, not "owner of their own workspace". If every signup becomes an owner, an owner check is no gate. |
| A8 | CSRF | Cookie-authenticated state-changing requests are protected (SameSite, CSRF token, or origin check). Look for GET requests that change state. |

## 2. Secrets & environment

| # | Control | What to look for |
|---|---|---|
| S1 | Secrets via env, not source | No keys in source, config or client bundles. Search for `sk_live_`, `sk_test_`, `AKIA`, `service_role`, `-----BEGIN`, `password =`, `token =`, `api_key`. Anything prefixed `NEXT_PUBLIC_`, `VITE_`, `EXPO_PUBLIC_` ships to browsers: check nothing secret carries those prefixes. Include tool config and docs: `.mcp.json`, `.cursor/`, `.claude/`, `.vscode/`, README and troubleshooting notes. |
| S2 | Rotation plan documented | Who rotates which key, how, and when. Usually NOT VERIFIABLE unless the repo has a runbook. |
| S3 | `.env` gitignored, history clean | `.env*` in `.gitignore`, and `git log --all --full-history -- '**/.env*'` shows no committed env file. |
| S4 | Webhook secrets verified | Every incoming webhook of any kind checks its signature against a secret before acting. Payment webhooks are also rated under B1. |
| S5 | API keys least-privilege | Keys used by the app have only the scopes it needs. Usually NOT VERIFIABLE; note which keys exist. |
| S6 | Separate secrets per environment | Dev, preview and production use different keys and databases. Look for one database URL or one Stripe key shared by every environment. |

## 3. Database & data access

| # | Control | What to look for |
|---|---|---|
| D1 | Tenant isolation on every query | Every query on user or tenant data is scoped to the caller (a `WHERE tenant_id = ?` or a row-level-security policy). Supabase/Postgres: RLS is enabled on every table holding user data, and policies check `auth.uid()`, not a client-supplied id. A tenant id taken from a header or body, rather than from the verified session, is a finding. |
| D2 | Parameterized queries | No SQL built by string concatenation or template literals with user input. |
| D3 | DB errors sanitized | Database error messages never reach API responses. |
| D4 | Migrations reversible | Each schema change has a down migration or a written rollback. |
| D5 | No client-side DB access with privileged credentials | The browser never holds a service-role key or a database URL. A public anon key is fine only when RLS is on (see D1). |
| D6 | Backup strategy documented | Backups exist and a restore has been tested. Usually NOT VERIFIABLE. |

## 4. API & input

| # | Control | What to look for |
|---|---|---|
| I1 | Input validation on bodies | Request bodies, query strings and form data are validated by schema (zod, valibot, pydantic, serializers) for shape, type, length and allowed values. |
| I2 | Rate limiting on auth and expensive operations | Login, signup, password reset, OTP verify, AI calls, email sends and uploads are rate-limited. |
| I3 | CORS explicit | No `Access-Control-Allow-Origin: *` on authenticated endpoints, no origin reflection, no regex that matches attacker domains (`.*example.com` matches `evil-example.com`). |
| I4 | Request size limits | Body and upload size limits on JSON and file endpoints. |
| I5 | Content-type validation on uploads | Upload type checked server-side (magic bytes or a safe parser), not trusted from the client. |

## 5. Billing & payments

| # | Control | What to look for |
|---|---|---|
| B1 | Webhook signature verified | Stripe: `constructEvent` (or equivalent) on the raw body before any processing. Other providers: their signature check. If payment webhooks are handled by another service, B1–B3 are NOT VERIFIABLE; say which service. |
| B2 | Webhook handler idempotent | Replaying an event does nothing twice (event id stored, upsert, or idempotency key). |
| B3 | Failed payment leaves DB consistent | Multi-step updates use a transaction or are safe to retry. Failed and cancelled payments downgrade access. |
| B4 | Tier gating server-side | Paid features are checked on the server for every request. A check only in the UI is bypassed by calling the API directly. |
| B5 | Checkout pre-flight for active subscription | Creating a checkout first checks for an existing subscription, so a double click does not create two. |
| B6 | Credits cannot be double-spent | Credit or quota decrements are atomic, and reserved credits are released on failure. |

## 6. AI-specific

| # | Control | What to look for |
|---|---|---|
| X1 | Prompt injection defenses | User content placed in prompts is delimited and treated as data. Model output never decides authorization. Tools the model can call are limited to what that user may do. |
| X2 | Model output validated | Output is parsed against a schema before it is stored, executed, or rendered as HTML. |
| X3 | Cost cap, authenticated | Per-user or per-tenant limits on AI spend. |
| X4 | Cost cap, anonymous | Anonymous or free-tier AI endpoints have stricter limits than paid ones, not weaker. |
| X5 | Telemetry excludes prompts and images | Analytics, logs and error trackers do not receive raw prompts, uploads or completions containing user data. |

## 7. Logging & observability

| # | Control | What to look for |
|---|---|---|
| L1 | Error logging to monitoring | Errors go to a persistent service (Sentry, Datadog, a log sink), not only `console.log`. |
| L2 | No PII in logs | Logs carry user ids, not emails, phone numbers, tokens or payment data. |
| L3 | Request tracing | A request or trace id flows through downstream calls. |
| L4 | Alerts on error rates | Someone is told when errors spike, not only when the site is down. Often NOT VERIFIABLE. |
| L5 | Health endpoint returns build hash | A health route returns status plus build or version, so you know what is deployed. |

## 8. Error handling

| # | Control | What to look for |
|---|---|---|
| E1 | Generic errors in production | No stack traces or internal messages in API responses. |
| E2 | Async operations have catch blocks | Promises and background jobs handle failure. Look for swallowed `catch {}` blocks that hide failures too. |
| E3 | Graceful degradation on dependency failure | When the database, cache or an external API fails, the app fails safe (no partial writes, no open access). |

## 9. Testing

| # | Control | What to look for |
|---|---|---|
| T1 | Auth flow tests | Tests for login, logout, expiry and rejection of anonymous calls on protected routes. |
| T2 | Multi-tenant isolation test | A test seeds two tenants and proves tenant A cannot read or change tenant B's data. In an app with no tenants, the same test with two users. |
| T3 | Billing test | Tests for checkout, webhook receipt, upgrade and failed payment. |
| T4 | Rate-limit test | A test proves limits fire at the stated threshold. |
| T5 | Tests run in CI on every change | Tests run in CI on every push or pull request, not only locally, and a failure blocks merge or deploy. A coverage target measured in CI is a bonus. |

## 10. Governance

| # | Control | What to look for |
|---|---|---|
| G1 | Threat model | A written threat model, even one page. |
| G2 | ADRs for key decisions | Auth strategy, data isolation and key dependencies are written down with reasons. |
| G3 | Constraints documented for AI agents | `CLAUDE.md`, `AGENTS.md`, `.cursor/rules` or similar tell coding agents what they must not change (auth, tenant scoping, billing). |
| G4 | CHANGELOG maintained | Significant changes are recorded with dates. |

## 11. Deployment & rollback

| # | Control | What to look for |
|---|---|---|
| R1 | Rollback plan per deploy | A documented way back (redeploy previous version, revert, down migration). |
| R2 | Feature flags for risky changes | Billing and auth changes can be switched off without a redeploy. |
| R3 | Migrations tested on staging first | A staging or preview environment with its own database, used before production. |
| R4 | Cache invalidation documented | Deploys do not leave old clients calling new APIs with stale assets. |
| R5 | Zero-downtime verified | Schema changes are backward compatible while old code is still serving. |

## 12. Supply chain

| # | Control | What to look for |
|---|---|---|
| C1 | Lockfile, frozen installs, dependency review | A committed lockfile, CI installs with it frozen (`npm ci`, `pnpm install --frozen-lockfile`, `uv sync --locked`, pinned `requirements.txt`), and every dependency name exists in its registry. AI-written code sometimes imports packages that don't exist, which attackers then register. A dependency pulled from git or a URL is not a missing package; check it separately. |

---

## Stack hints

Where these controls usually live. Use them as a starting point, not a list of the only places to look.

- **Next.js:** `middleware.ts` (often covers pages but not every `app/api/**/route.ts`, and sometimes only refreshes the session, so each page and action must check for itself), server actions (`"use server"` functions are public endpoints, and every argument is attacker-controlled, including whole objects passed from a component), `next.config.*` headers, `NEXT_PUBLIC_*` env. Server actions default to a 1 MB body limit; App Router route handlers have no limit that rejects a large body, so each one that reads a body needs its own (I4). The server-action `Origin` check doesn't cover route handlers (A8).
- **Supabase:** `supabase/migrations/*.sql` for RLS (`enable row level security`, `create policy`), `service_role` key usage (server only, and never fed a user-chosen id or email), edge functions in `supabase/functions/`. A policy that uses `true`, or compares against a client-supplied column, is a finding on user data and on any insert, update or delete; `using (true)` on select for a genuinely public table (a product catalogue) is fine. Check update policies for columns users shouldn't set themselves (billing, role, plan). Live RLS settings in the dashboard are NOT VERIFIABLE from the repo unless migrations define them.
- **Vercel:** environment variables are scoped per environment in the dashboard, not in `vercel.json` (S6 is NOT VERIFIABLE unless the repo's docs say how they're set), and preview deployments may share production keys. Skew Protection can cover R4, but it is an opt-in project setting with a time limit, so it is NOT VERIFIABLE unless the repo shows it's on. Instant rollback covers part of R1 only if someone has written down when to use it.
- **Stripe:** the webhook's event selection and the account's default API version live in the dashboard. Prices and trial lengths must come from the server or Stripe, never from the client. A handler that returns non-2xx for event types it doesn't handle gets those events retried for days and fills the dashboard with failures that hide real ones. Customer lookups by email (`customers.list({ email })`) can link a new signup to someone else's existing customer.
- **Firebase:** `firestore.rules` and `storage.rules` (`allow read, write: if true` is a Critical), Cloud Functions auth checks.
- **Express / Hono / Fastify:** middleware order (auth registered after the routes it should protect), `cors()` options, body parser limits.
- **Cloudflare Workers:** `wrangler.toml` / `wrangler.jsonc` (vars vs secrets, shared bindings across environments), D1 queries (`prepare(...).bind`), service-binding RPC methods (each one is an entry point).
- **FastAPI:** `Depends(...)` auth on each router or route, pydantic models on bodies, `CORSMiddleware` settings, SQLAlchemy `text()` with f-strings.
- **Django:** `@login_required` / permission classes on every view and DRF viewset, `DEBUG`, `ALLOWED_HOSTS`, `SECRET_KEY` in settings, `raw()` and `extra()` queries, querysets not filtered by owner.
- **Flask:** `@login_required` coverage, `app.secret_key`, `debug=True`, `flask_cors` settings.
- **Lovable / Bolt / v0 exports:** usually Supabase or Firebase behind a React client: start with RLS or security rules and with what the client calls directly.
