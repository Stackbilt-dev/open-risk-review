# Risk review — Northstar Projects

> **Worked example.** Northstar Projects is a made-up app, and every file, line and commit below is invented to show what an Open Risk Review report looks like. It does not describe any real codebase. It is the same fictional app as the [sample report on stackbilder.com](https://stackbilder.com/production-risk-review/sample-report), with the full coverage table, fix prompts and three Low findings that the shorter web sample leaves out.

> **Self-review, generated with Open Risk Review.** An AI model read this repository at the commit below and wrote what it found. It can miss things and it can be wrong: confirm each finding before acting on it. This is not a penetration test, a compliance assessment, or a statement that the app is safe or ready to launch. **This file describes how to attack your app. Don't commit it to a public repository.**

**Repository:** `northstar-projects` at commit `4f1c2a9e7b3d5c8a1e6f0b2d9c4a7e3f5b8d1c6a` (committed 2026-10-14), reviewed 2026-10-15. Uncommitted changes: none. `.risk-review/` ignored by git: yes.
**Context:** the owner said the app is live, takes card payments for team plans, and holds client project data.
**Checklist:** Open Risk Review checklist v1.0 (59 controls, all listed in section 3).
**Method:** a survey of the codebase, four focused review passes (auth & API; billing, data & secrets; AI & operations; delivery & supply chain), then direct re-reading of every Critical and High finding. Read-only: no code was changed, nothing was deployed, no live system or credential was used. Test suite: ran `pnpm test`: 84 passed, 0 failed.
**Not in scope:** the hosting provider's environment settings, the Stripe dashboard, the managed Postgres console, and the Google and GitHub OAuth app settings. Controls that live there are marked NOT VERIFIABLE.

---

## 1. Summary

**2 Critical, 1 High, 1 Medium, 3 Low.** Northstar Projects is a multi-tenant project tracker on Next.js (App Router), Auth.js with database sessions, Postgres through Drizzle, and Stripe subscriptions, with one AI feature that summarises project updates. The two Criticals let a stranger give any account a paid plan for free and let any customer read another customer's projects.

**Fix first:**

- **F1:** the Stripe webhook accepts unsigned events, so anyone can post a fake `invoice.paid` and upgrade any account. About half a day, including a replay test.
- **F2:** `GET /api/projects/:id` checks that you're signed in but not that the project is yours. One to two days for a tenant-scoped query helper and isolation tests.

**Then:**

- **F3:** the data export is hidden from non-admins in the UI but not refused by the API. About half a day.

**What's already good:**

- No secrets in the code or in git history; `.env*` is ignored and env vars are validated at startup (`lib/env.ts`).
- Session cookies are `HttpOnly`, `Secure` and `SameSite=Lax`, and logout deletes the session row server-side.
- Sign-in and the AI summary route are rate-limited, and both limits have tests.
- The AI feature treats user text as data, validates the model's JSON with zod, and has a per-workspace monthly cap.
- Migrations run against a preview database on every pull request before production.

| Severity | Count |
|---|---|
| Critical | 2 |
| High | 1 |
| Medium | 1 |
| Low | 3 |

---

## 2. Findings

Severity: **Critical** is exploitable now by someone with no special access (a stranger, or any signed-up user) and affects other users' data, money or the whole platform. **High** is exploitable now with more limited impact, or a gap that will ship a regression to users without anyone noticing. **Medium** needs a second failure before it becomes an incident. **Low** is hygiene.

### F1 · [Critical] Anyone can upgrade any account to Pro with a forged Stripe webhook

- **Evidence:** `app/api/webhooks/stripe/route.ts:14`: `const event = JSON.parse(await req.text())`. The handler never reads the `Stripe-Signature` header, then applies `invoice.paid` at line 31 by setting `plan = 'pro'` on the workspace named in `event.data.object.metadata.workspaceId`. `middleware.ts:8` excludes `/api/webhooks` from auth, as it should, so nothing else checks the caller.
- **Scenario:** A stranger posts `{"type":"invoice.paid","data":{"object":{"metadata":{"workspaceId":"<any id>"}}}}` to `/api/webhooks/stripe`. The workspace becomes Pro with no payment. Workspace ids appear in shared project links, so they are easy to collect. Replaying a real event also re-applies it, because nothing records event ids.
- **Fix:** Read the raw body with `await req.text()`, verify it with `stripe.webhooks.constructEvent(body, signature, env.STRIPE_WEBHOOK_SECRET)` before parsing, and return 400 on failure. Store each processed `event.id` in a `stripe_events` table with a unique constraint and skip duplicates.
- **Effort:** S (half a day). One small migration for `stripe_events`.
- **Verify:** Tests prove that a missing or invalid signature returns 400 and changes nothing, and that a valid event replayed twice changes the plan once.
- **Fix prompt:**

  ```text
  In app/api/webhooks/stripe/route.ts, the Stripe webhook parses the body with JSON.parse and never verifies the Stripe-Signature header. Change it so that:
  1. It reads the raw body with `await req.text()` and verifies it with `stripe.webhooks.constructEvent(body, req.headers.get("stripe-signature"), env.STRIPE_WEBHOOK_SECRET)` before doing anything else. On failure, return 400 and change nothing.
  2. It records each processed event.id in a new `stripe_events` table (unique on id) and skips events it has already processed. Add the Drizzle migration.
  Don't change how each event type updates the workspace, and keep /api/webhooks excluded from auth in middleware.ts.
  Add tests in tests/webhooks/stripe.test.ts: a missing signature returns 400, an invalid signature returns 400, and a valid invoice.paid event delivered twice upgrades the plan exactly once.
  Show me the diff before applying it.
  ```

### F2 · [Critical] Any customer can read another customer's projects by id

- **Evidence:** `app/api/projects/[id]/route.ts:9`: `db.select().from(projects).where(eq(projects.id, params.id))`. The route calls `requireSession()` at line 6, but neither the query nor the code after it compares `project.workspaceId` with the session's workspace. `PATCH` and `DELETE` in the same file (lines 22 and 41) have the same gap.
- **Scenario:** Any signed-up user requests `/api/projects/<id>` for an id from another workspace (ids are sequential integers) and gets its name, members and description. With `PATCH` and `DELETE` they can also change or remove it.
- **Fix:** Add a `projectForWorkspace(id, workspaceId)` helper in `lib/db/projects.ts` that filters on both columns, use it in all three handlers with the workspace from the session, and return 404 when it finds nothing.
- **Effort:** M (one to two days, mostly tests). No schema change.
- **Verify:** An integration test seeds two workspaces and proves workspace A gets 404 on GET, PATCH and DELETE for workspace B's project.
- **Fix prompt:**

  ```text
  In app/api/projects/[id]/route.ts, the GET, PATCH and DELETE handlers look up a project by id only, so any signed-in user can read, change or delete another workspace's project.
  1. Add a helper `projectForWorkspace(id, workspaceId)` in lib/db/projects.ts that queries with both `eq(projects.id, id)` and `eq(projects.workspaceId, workspaceId)`.
  2. Use it in all three handlers, taking workspaceId from the session returned by requireSession(), never from the request.
  3. Return 404 (not 403) when no project matches, so ids from other workspaces look the same as missing ones.
  Don't change the response shape for a project the user owns.
  Add tests/api/projects-isolation.test.ts: seed two workspaces, and prove a user in workspace A gets 404 on GET, PATCH and DELETE for workspace B's project, and 200 for their own.
  Show me the diff before applying it.
  ```

### F3 · [High] Any member can export workspace data meant for admins

- **Evidence:** `app/(app)/settings/page.tsx:41` shows the export button only when `isAdmin` is true, but `app/api/export/route.ts:8` only calls `requireSession()` and then starts the export job at line 12. No role check runs on the server.
- **Scenario:** A member (for example a contractor invited to one project) calls `POST /api/export` directly and receives a full export of the workspace, including every project and member email.
- **Fix:** Call `requireRole(session, "admin")` before creating the job, and log who requested the export and whether it was allowed.
- **Effort:** S (half a day).
- **Verify:** An API test returns 403 for a member and 202 for an admin with otherwise identical requests.
- **Fix prompt:**

  ```text
  In app/api/export/route.ts, the export endpoint checks only that the user is signed in. The admin restriction exists only in the UI (app/(app)/settings/page.tsx).
  Add a server-side check: after requireSession(), call requireRole(session, "admin") from lib/auth.ts and return 403 if it fails, before any export job is created. Log the acting user id, workspace id and the allow/deny decision.
  Don't change the export format or the UI.
  Add a test in tests/api/export.test.ts: a member gets 403 and no job is created; an admin gets 202.
  Show me the diff before applying it.
  ```

### F4 · [Medium] Upstream error messages reach API responses and logs

- **Evidence:** `lib/http/errors.ts:22`: `return Response.json({ error: err.message }, { status: 502 })` for any upstream failure, including Stripe and the AI provider. `lib/log.ts:15` logs the full request body on errors.
- **Scenario:** When Stripe or the AI provider fails, its message (which can include request fragments and internal ids) is returned to the client. The error log also stores request bodies, which can hold project text and email addresses. Exploiting this needs an upstream failure first, so it is Medium.
- **Fix:** Return a stable public error code (`{"error":"upstream_unavailable"}`) and keep details in a structured server log with tokens, cookies, `authorization` headers and request bodies removed.
- **Effort:** S (half a day to a day).
- **Verify:** A test forces a provider failure and checks the public response contains only the code, and the log record contains no body or headers.
- **Fix prompt:**

  ```text
  lib/http/errors.ts returns err.message from upstream failures to the client, and lib/log.ts logs full request bodies on errors.
  1. Change the upstream error path to return a fixed public body, {"error":"upstream_unavailable"}, with the same status code.
  2. In lib/log.ts, log the error message and a request id, but remove request bodies, cookies, and authorization headers before writing.
  Don't change successful responses.
  Add tests/http/errors.test.ts: force a provider error and assert the response body is exactly {"error":"upstream_unavailable"} and the captured log record has no body, cookie or authorization field.
  Show me the diff before applying it.
  ```

### F5 · [Low] Billing and role changes don't ask the user to sign in again

- **Evidence:** `app/api/billing/portal/route.ts:7` and `app/api/members/[id]/role/route.ts:9` act on any valid session, however old. Sessions last 30 days (`lib/auth.ts:18`).
- **Fix:** Require a sign-in within the last 15 minutes for these two routes. **Effort:** S.
- **Fix prompt:**

  ```text
  Add a requireRecentSignIn(session, 15 * 60) check to app/api/billing/portal/route.ts and app/api/members/[id]/role/route.ts that returns 401 with code "reauth_required" when the session's sign-in is older than 15 minutes. Don't change other routes. Show me the diff before applying it.
  ```

### F6 · [Low] No documented way back from a bad deploy

- **Evidence:** `drizzle/` has 23 forward migrations and no down migrations or rollback notes. There is no rollback runbook, and `app/api/health/route.ts:3` returns `{"status":"ok"}` with no build id, so you can't confirm which version is live.
- **Fix:** Add a short `docs/rollback.md` (redeploy the previous build; how to undo the last migration) and return the commit SHA from the health route. **Effort:** S.
- **Fix prompt:**

  ```text
  Write docs/rollback.md describing how to redeploy the previous production build on our host and how to reverse the most recent Drizzle migration by hand. Then change app/api/health/route.ts to return {"status":"ok","build":process.env.COMMIT_SHA ?? "unknown"}. Show me the diff before applying it.
  ```

### F7 · [Low] No threat model or record of key decisions

- **Evidence:** No threat model, and no ADRs or decision notes, anywhere in the repository. `CLAUDE.md` lists coding constraints but not why auth and tenancy work the way they do.
- **Fix:** One page covering who can attack what (strangers, members, admins, Stripe, the AI provider), and short notes on the auth, tenancy and billing decisions. **Effort:** S.
- **Fix prompt:**

  ```text
  Draft docs/threat-model.md (one page: assets, the people and systems that can reach them, and the top risks, citing F1–F4 from .risk-review/) and docs/decisions/ with three short notes: database sessions, workspace tenancy, Stripe as the billing source of truth. Ask me about anything you can't tell from the code. Show me the files before saving them.
  ```

---

## 3. Control coverage

| # | Area | Control | Status | Evidence |
|---|---|---|---|---|
| A1 | Auth & sessions | Auth middleware on all protected routes | PASS | `middleware.ts:8` matches `/app` and `/api` except `/api/webhooks` and `/api/health`; every route handler also calls `requireSession()` |
| A2 | Auth & sessions | Session revocation | PASS | Logout deletes the session row (`lib/auth.ts:44`) |
| A3 | Auth & sessions | JWT algorithm allowlisted | N/A | Database sessions; no JWTs issued or accepted |
| A4 | Auth & sessions | MFA for high-privilege actions | PARTIAL | Provider MFA available at sign-in; no step-up for billing or role changes (F5) |
| A5 | Auth & sessions | OAuth redirect URI allowlisted | PASS | Fixed callback URLs; `returnTo` must be a relative path (`lib/auth.ts:61`) |
| A6 | Auth & sessions | Cookie flags | PASS | `HttpOnly`, `Secure`, `SameSite=Lax`, 30-day max-age (`lib/auth.ts:18`) |
| A7 | Auth & sessions | Admin gating | FAIL | Export admin check is UI-only (F3) |
| A8 | Auth & sessions | CSRF | PASS | The `SameSite=Lax` session cookie (`lib/auth.ts:18`) isn't sent on cross-site POSTs to route handlers; the Auth.js CSRF token covers only its own sign-in and sign-out; no state-changing GET routes found |
| S1 | Secrets | Secrets via env, not source | PASS | No keys in source or history; `lib/env.ts` validates env at startup |
| S2 | Secrets | Rotation plan documented | NOT VERIFIABLE | No runbook in the repo; would live in the team's ops docs or password manager |
| S3 | Secrets | `.env` gitignored, history clean | PASS | `.gitignore:4`; no `.env*` in git history |
| S4 | Secrets | Webhook secrets verified | FAIL | Stripe webhook unsigned (F1) |
| S5 | Secrets | API keys least-privilege | NOT VERIFIABLE | Stripe restricted-key settings live in the dashboard |
| S6 | Secrets | Separate secrets per environment | NOT VERIFIABLE | Env values live in the host's settings; `lib/env.ts` reads one set |
| D1 | Database | Tenant isolation on every query | FAIL | Project GET/PATCH/DELETE unscoped (F2); other queries go through `forWorkspace()` |
| D2 | Database | Parameterized queries | PASS | Drizzle query builder throughout; the one `sql` template binds values |
| D3 | Database | DB errors sanitized | PARTIAL | DB errors map to 500 with a code; upstream errors leak (F4) |
| D4 | Database | Migrations reversible | FAIL | No down migrations or rollback notes (F6) |
| D5 | Database | No client-side DB access with privileged credentials | PASS | All queries run server-side; no DB URL in client bundles |
| D6 | Database | Backup strategy documented | NOT VERIFIABLE | Nothing in the repo; backups and restore tests would be in the managed Postgres provider's settings |
| I1 | API & input | Input validation on bodies | PASS | zod schemas on every JSON route (`lib/validate.ts`) |
| I2 | API & input | Rate limiting on auth and expensive operations | PASS | Sign-in and `/api/ai/summarize` limited (`lib/ratelimit.ts`) |
| I3 | API & input | CORS explicit | PASS | Same-origin only; no CORS headers set |
| I4 | API & input | Request size limits | PARTIAL | Server actions (settings forms) keep Next.js 14.2's 1 MB default; the route handlers read `req.json()` with no size check, and App Router route handlers have no default limit |
| I5 | API & input | Content-type validation on uploads | N/A | No file uploads |
| B1 | Billing | Webhook signature verified | FAIL | F1 |
| B2 | Billing | Webhook handler idempotent | FAIL | No event-id record (F1) |
| B3 | Billing | Failed payment leaves DB consistent | PASS | `invoice.payment_failed` downgrades in one transaction (`lib/billing.ts:52`) |
| B4 | Billing | Tier gating server-side | PASS | `requirePlan()` on every Pro route |
| B5 | Billing | Checkout pre-flight for active subscription | PASS | `app/api/billing/checkout/route.ts:11` refuses when a subscription is active |
| B6 | Billing | Credits cannot be double-spent | N/A | No credits |
| X1 | AI-specific | Prompt injection defenses | PASS | User text sent as a delimited data block; the model has no tools and decides nothing (`lib/ai/summarize.ts:20`) |
| X2 | AI-specific | Model output validated | PASS | JSON output parsed with zod and rendered as text |
| X3 | AI-specific | Cost cap, authenticated | PASS | Per-workspace monthly cap (`lib/ai/budget.ts`) |
| X4 | AI-specific | Cost cap, anonymous | N/A | No anonymous AI access |
| X5 | AI-specific | Telemetry excludes prompts and images | PASS | Analytics events carry ids and counts only |
| L1 | Logging | Error logging to monitoring | PASS | Sentry configured (`sentry.server.config.ts`) |
| L2 | Logging | No PII in logs | PARTIAL | Request bodies logged on error (F4) |
| L3 | Logging | Request tracing | PASS | Request id set in middleware and passed to Sentry |
| L4 | Logging | Alerts on error rates | NOT VERIFIABLE | Alert rules live in Sentry |
| L5 | Logging | Health endpoint returns build hash | PARTIAL | Health route exists without a build id (F6) |
| E1 | Error handling | Generic errors in production | FAIL | F4 |
| E2 | Error handling | Async operations have catch blocks | PASS | Route handlers wrapped by `withErrors()`; no empty `catch` blocks |
| E3 | Error handling | Graceful degradation on dependency failure | PASS | AI summary fails closed with a message; billing writes are transactional |
| T1 | Testing | Auth flow tests | PASS | `tests/auth/*.test.ts` cover sign-in, logout, expiry and anonymous rejection |
| T2 | Testing | Multi-tenant isolation test | FAIL | None for projects (F2) |
| T3 | Testing | Billing test | PARTIAL | Checkout and downgrade tested; no signature or replay test (F1) |
| T4 | Testing | Rate-limit test | PASS | `tests/ratelimit.test.ts` checks the threshold and reset |
| T5 | Testing | Tests run in CI on every change | PASS | `.github/workflows/ci.yml` runs on every pull request; required check |
| G1 | Governance | Threat model | FAIL | F7 |
| G2 | Governance | ADRs for key decisions | FAIL | F7 |
| G3 | Governance | Constraints documented for AI agents | PASS | `CLAUDE.md` forbids unscoped queries and client-side secrets |
| G4 | Governance | CHANGELOG maintained | PASS | `CHANGELOG.md`, dated entries |
| R1 | Deployment | Rollback plan per deploy | FAIL | F6 |
| R2 | Deployment | Feature flags for risky changes | PASS | Billing changes behind flags (`lib/flags.ts`) |
| R3 | Deployment | Migrations tested on staging first | PASS | Preview database migrated on every pull request (`ci.yml:41`) |
| R4 | Deployment | Cache invalidation documented | NOT VERIFIABLE | Static assets are hashed and API responses `no-store`; whether old clients keep reaching old servers during a rollout is a host setting |
| R5 | Deployment | Zero-downtime verified | PARTIAL | The last five migrations only add tables and nullable columns (`drizzle/0019`–`0023`), but nothing in the repo says schema changes must stay backward compatible |
| C1 | Supply chain | Lockfile, frozen installs, dependency review | PASS | `pnpm-lock.yaml`; CI uses `--frozen-lockfile`; all 61 dependencies exist on npm |

**Totals (59 controls):** 31 PASS · 7 PARTIAL · 11 FAIL · 6 NOT VERIFIABLE · 4 N/A.

---

## 4. Fix plan

Ordered by risk removed per hour of work. Each row's check is how you know it's done.

| # | Fix | Closes | Effort | Depends on | Check |
|---|---|---|---|---|---|
| 1 | Verify Stripe signatures; record event ids | F1 | S | — | Unsigned and replayed events refused in tests |
| 2 | Workspace-scoped project helper in all three handlers | F2 | M | — | Isolation test: 404 across workspaces |
| 3 | Server-side admin check on export | F3 | S | — | Member 403, admin 202 |
| 4 | Fixed public error body; redact logs | F4 | S | — | Error and log snapshot tests |
| 5 | Recent sign-in for billing and role changes | F5 | S | — | Stale session gets `reauth_required` |
| 6 | Rollback notes; build id in health | F6 | S | — | `/api/health` shows the commit |
| 7 | Threat model and decision notes | F7 | S | 1–4 | Docs merged |

Rows 1–3 are a reasonable first day and a half. When to launch is your call.

---

## Appendix: how this was checked

- What the survey mapped: Next.js App Router with 31 route handlers, Auth.js (database sessions, Google and GitHub sign-in), Postgres through Drizzle (14 tables, 23 migrations), Stripe subscriptions and one webhook, one AI route (`/api/ai/summarize`), Sentry, GitHub Actions CI with a preview database.
- Critical and High findings re-read directly: F1, F2, F3. Each was traced from the route through middleware to the query or state change.
- Suspicions that turned out fine:
  - `sql` template in `lib/db/search.ts:12` looked like string-built SQL; it binds every value.
  - `/api/health` is unauthenticated; it returns no data beyond status, which is intended.
  - The AI route passes project text into the prompt; the model has no tools and its output is only displayed, so injection can't act on anything.
- Couldn't check: alert rules, backups, key rotation, environment separation, Stripe key scopes and the host's rollout settings, which live outside the repository.
