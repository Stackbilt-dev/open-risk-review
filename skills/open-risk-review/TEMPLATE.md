# Risk review — {{APP_NAME}}

> **Self-review, generated with Open Risk Review.** An AI model read this repository at the commit below and wrote what it found. It can miss things and it can be wrong: confirm each finding before acting on it. This is not a penetration test, a compliance assessment, or a statement that the app is safe or ready to launch. **This file describes how to attack your app. Don't commit it to a public repository.**

**Repository:** `{{repo name or path}}` at commit `{{FULL_COMMIT_SHA}}` (committed {{COMMIT_DATE}}), reviewed {{REVIEW_DATE}}. Uncommitted changes: {{none / listed}}. `.risk-review/` ignored by git: {{yes / no, user told / no, offered, no answer yet}}.
**Context:** {{What the owner told you about the app (live? payments? other people's data?). Mark anything you assumed: "(assumed, no answer) live, takes payments, holds other people's data."}}
**Checklist:** Open Risk Review checklist v{{CHECKLIST_VERSION}} ({{N}} controls, all listed in section 3).
**Method:** a survey of the codebase, four focused review passes (auth & API; billing, data & secrets; AI & operations; delivery & supply chain), then direct re-reading of every Critical and High finding. Read-only: no code was changed, nothing was deployed, no live system or credential was used. {{Test suite: "ran `<command>`: N passed, M failed." or "not run, because ...".}}
**Not in scope:** {{systems outside the repository, e.g. the Supabase dashboard, the Stripe account, the hosting provider}}. Controls that live there are marked NOT VERIFIABLE.

---

## 1. Summary

**{{n}} Critical, {{n}} High, {{n}} Medium, {{n}} Low.** {{Two sentences: what kind of app this is, and which findings matter most. Describe findings; don't declare the app safe, unsafe, ready or not ready.}}

**Fix first:**

- {{One bullet per Critical: what someone can do today, and roughly how big the fix is. If it is Critical only because of an unconfirmed Condition, say which fact to check first.}}

**Then:**

- {{One bullet per High.}}

**What's already good:** {{Three to five real strengths, specific enough to repeat to a teammate.}}

| Severity | Count |
|---|---|
| Critical | {{n}} |
| High | {{n}} |
| Medium | {{n}} |
| Low | {{n}} |

---

## 2. Findings

Severity: **Critical** is exploitable now by someone with no special access (a stranger, or any signed-up user) and affects other users' data, money or the whole platform. **High** is exploitable now with more limited impact, or a gap that will ship a regression to users without anyone noticing. **Medium** needs a second failure before it becomes an incident. **Low** is hygiene.

### F1 · [{{SEV}}] {{What can happen, as a sentence}}

- **Evidence:** `{{file:line}}`: {{what the code does, quoting the decisive line}}. {{Any other file needed to make the finding real.}}
- **Scenario:** {{The concrete attack or failure, step by step, from the attacker's or the on-call person's seat.}}
- **Fix:** {{The specific change: the function, middleware, policy or config key.}}
- **Effort:** {{S / M / L}}.
- **Verify:** {{The test or check that proves it is fixed.}}
- **Condition:** {{Only when an unconfirmed fact about the owner's setup decides severity. The finding is rated on the higher reading; name the fact that would lower it: "Medium if email confirmation is on in the Supabase project." Otherwise omit this line.}}
- **Fix prompt:**

  ```text
  {{A self-contained instruction the owner can paste into their AI coding tool. Name the files, the exact change, what must not change, and the test to add from "Verify" (or, for a dashboard setting or a document, the check that proves it). End with: "Show me the diff before applying it."}}
  ```

### F2 · [{{SEV}}] {{...}}

{{Repeat for every finding, Critical first. Lows can be one line each with a short fix prompt; a test is optional for Lows.}}

---

## 3. Control coverage

| # | Area | Control | Status | Evidence |
|---|---|---|---|---|
| A1 | Auth & sessions | Auth middleware on all protected routes | | |
| A2 | Auth & sessions | Session revocation | | |
| A3 | Auth & sessions | JWT algorithm allowlisted | | |
| A4 | Auth & sessions | MFA for high-privilege actions | | |
| A5 | Auth & sessions | OAuth redirect URI allowlisted | | |
| A6 | Auth & sessions | Cookie flags | | |
| A7 | Auth & sessions | Admin gating | | |
| A8 | Auth & sessions | CSRF | | |
| S1 | Secrets | Secrets via env, not source | | |
| S2 | Secrets | Rotation plan documented | | |
| S3 | Secrets | `.env` gitignored, history clean | | |
| S4 | Secrets | Webhook secrets verified | | |
| S5 | Secrets | API keys least-privilege | | |
| S6 | Secrets | Separate secrets per environment | | |
| D1 | Database | Tenant isolation on every query | | |
| D2 | Database | Parameterized queries | | |
| D3 | Database | DB errors sanitized | | |
| D4 | Database | Migrations reversible | | |
| D5 | Database | No client-side DB access with privileged credentials | | |
| D6 | Database | Backup strategy documented | | |
| I1 | API & input | Input validation on bodies | | |
| I2 | API & input | Rate limiting on auth and expensive operations | | |
| I3 | API & input | CORS explicit | | |
| I4 | API & input | Request size limits | | |
| I5 | API & input | Content-type validation on uploads | | |
| B1 | Billing | Webhook signature verified | | |
| B2 | Billing | Webhook handler idempotent | | |
| B3 | Billing | Failed payment leaves DB consistent | | |
| B4 | Billing | Tier gating server-side | | |
| B5 | Billing | Checkout pre-flight for active subscription | | |
| B6 | Billing | Credits cannot be double-spent | | |
| X1 | AI-specific | Prompt injection defenses | | |
| X2 | AI-specific | Model output validated | | |
| X3 | AI-specific | Cost cap, authenticated | | |
| X4 | AI-specific | Cost cap, anonymous | | |
| X5 | AI-specific | Telemetry excludes prompts and images | | |
| L1 | Logging | Error logging to monitoring | | |
| L2 | Logging | No PII in logs | | |
| L3 | Logging | Request tracing | | |
| L4 | Logging | Alerts on error rates | | |
| L5 | Logging | Health endpoint returns build hash | | |
| E1 | Error handling | Generic errors in production | | |
| E2 | Error handling | Async operations have catch blocks | | |
| E3 | Error handling | Graceful degradation on dependency failure | | |
| T1 | Testing | Auth flow tests | | |
| T2 | Testing | Multi-tenant isolation test | | |
| T3 | Testing | Billing test | | |
| T4 | Testing | Rate-limit test | | |
| T5 | Testing | Tests run in CI on every change | | |
| G1 | Governance | Threat model | | |
| G2 | Governance | ADRs for key decisions | | |
| G3 | Governance | Constraints documented for AI agents | | |
| G4 | Governance | CHANGELOG maintained | | |
| R1 | Deployment | Rollback plan per deploy | | |
| R2 | Deployment | Feature flags for risky changes | | |
| R3 | Deployment | Migrations tested on staging first | | |
| R4 | Deployment | Cache invalidation documented | | |
| R5 | Deployment | Zero-downtime verified | | |
| C1 | Supply chain | Lockfile, frozen installs, dependency review | | |

**Totals (59 controls):** {{n}} PASS · {{n}} PARTIAL · {{n}} FAIL · {{n}} NOT VERIFIABLE · {{n}} N/A.

---

## 4. Fix plan

Ordered by risk removed per hour of work. Each row's check is how you know it's done.

| # | Fix | Closes | Effort | Depends on | Check |
|---|---|---|---|---|---|
| 1 | {{fix}} | F1 | S | — | {{check}} |
| 2 | | | | | |

Rows 1–{{n}} are a reasonable first day. When to launch is your call.

---

## Appendix: how this was checked

- What the survey mapped: {{stack, entry points, data stores, external services}}.
- Critical and High findings re-read directly: {{IDs}}.
- Suspicions that turned out fine, recorded so nobody re-raises them: {{list, with the reason each one is fine}}.
- Couldn't check: {{anything the passes could not confirm either way, and why}}.
