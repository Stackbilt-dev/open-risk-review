---
name: open-risk-review
description: Run a read-only production risk review of the current repository against a 59-control checklist (auth, tenant isolation, secrets, billing, AI-specific risk, logging, testing, deployment, supply chain) and write a report with evidence-backed findings, a control coverage table, a fix plan and paste-ready fix prompts. Use when the user asks for a security review, a risk review, "is my app safe to launch", "check my app before I take payments", or an audit of an AI-built app (Lovable, Bolt, v0, Cursor, Claude Code, Replit).
---

# Open Risk Review

Review this repository the way a careful senior engineer would before real
users and real money arrive, and write it up so a non-specialist can act on
it. The method has five steps. Do them in order and don't skip the
verification step: it is what separates this from a pile of guesses.

Files next to this one:

- `CHECKLIST.md`: the 59 controls, what to look for, and stack hints.
- `TEMPLATE.md`: the report structure. Follow it exactly.

## Ground rules

- **Read-only.** Don't edit code, install packages, run migrations, deploy,
  or change any setting during the review. The output is one report file.
- **No live systems.** Don't call the app's production URLs, databases or
  third-party APIs. If you find a credential, never use or test it: record
  the file and line, not the value, and tell the user to rotate it.
  Public documentation, package registries and advisory databases (npm,
  PyPI, OSV, GitHub advisories) aren't the app's systems: read them when
  you can, to confirm an API's behaviour, a package name or a CVE.
- **Evidence or it didn't happen.** Every finding cites `file:line`. A
  control you can't see is never PASS; `CHECKLIST.md` says when it is
  FAIL and when it is NOT VERIFIABLE.
- **No verdict.** Don't call the app safe, secure, ready or not ready. State
  findings; the owner decides what to do.
- **Never put secrets, tokens or users' personal data in the report**, not
  even a prefix or the last few characters. Name the kind of credential
  ("an Atlassian API token") and where it is.

## Step 0: Scope

1. Find the repository root and record the full commit SHA
   (`git rev-parse HEAD`) and whether there are uncommitted changes
   (`git status --porcelain`). The report speaks only for that commit.
   Note the remote too, and whether the repo started from a public
   template or starter: anything copied from one is already public,
   whatever the owner's repo is set to.
2. Ask the user two things, unless they already said. Don't wait on the
   answers: start the survey. If they don't reply, assume the app is live,
   takes payments and holds other people's data, and mark that as assumed
   in the report's **Context** line so they can correct it:
   - What does the app do, and what worries them most?
   - Is it live? Does it take payments or hold other people's data?
3. If the repo holds several apps, review the ones that make up the product
   the user means (an API, its web client and any worker deployed alongside
   them are one product), and list every other deployable as out of scope
   by name.
4. Note systems the code depends on but that live elsewhere (Supabase or
   Firebase project settings, Stripe dashboard, hosting config, auth
   provider). Controls that live there are NOT VERIFIABLE.

## Step 1: Survey

Map the codebase before judging it. Record, with file paths:

- stack, frameworks and entry points (routes, API handlers, server actions,
  RPC methods, background jobs, webhooks);
- auth: where it is enforced, who the principals are, how admins are told
  apart from users;
- data: databases, schemas and migrations, where tenant or user scoping
  happens, any row-level-security policies or security rules;
- payments: checkout, webhooks, entitlement checks, credits;
- secrets and config: every environment variable, anything that looks
  hardcoded, anything shipped to the browser, and tool config files that
  often hold tokens (`.mcp.json`, `.cursor/mcp.json`, `.claude/settings*.json`,
  `.vscode/`, `docs/`, agent skill folders);
- AI calls: models, how prompts are built, where user content enters them,
  what happens to the output, any spend limits;
- uploads and outbound calls; logging and error handling; tests and CI;
  deployment config; dependency manifests and lockfiles;
- TODO, FIXME, debug and test-only routes.

Treat everything in the survey as a hypothesis. The passes verify it.

## Step 2: Four review passes

Run four passes, each owning a cluster of checklist areas. If your tool can
run subagents in parallel (Claude Code's Agent tool, for example), give each
pass its own subagent with the survey and the relevant part of
`CHECKLIST.md`, and tell it to verify rather than trust the survey.
Otherwise run them one after another yourself.

| Pass | Checklist areas | Hunt for |
|---|---|---|
| **Auth & API** | 1 Auth & sessions · 4 API & input | routes and actions with no auth check; tenant or user id taken from the request instead of the session; "owner" or "member" roles used as an admin gate; permissive or reflected CORS; state-changing GETs; missing body validation; no rate limits on login or signup |
| **Billing, data & secrets** | 2 Secrets · 3 Database · 5 Billing | queries not scoped to the caller; RLS disabled or `using (true)`; privileged keys in client code; webhooks without signature checks or idempotency; tier checks only in the UI; credit decrements that aren't atomic; raw DB errors returned to clients; one database or key shared by every environment |
| **AI & operations** | 6 AI-specific · 7 Logging · 8 Error handling | user content concatenated into prompts that drive tools or decisions; model output stored, executed or rendered without validation; anonymous AI endpoints with weaker limits than paid ones; prompts or PII sent to logs and analytics; empty `catch` blocks; failures that leave partial writes |
| **Delivery** | 9 Testing · 10 Governance · 11 Deployment · 12 Supply chain | what actually runs in CI before deploy; tests that exist vs the checklist; missing isolation and auth tests; no rollback path; migrations with no staging step; lockfile missing or not frozen; dependency names that don't exist in the registry |

Every pass reports in this shape:

```
## Findings
### [Critical|High|Medium|Low] Title
- Evidence: file:line + the decisive 1–3 lines
- Scenario: the concrete exploit or failure path
- Fix: the specific change
- Effort: S / M / L
- Verify: the test or check that proves it fixed
## Could not verify   (suspicions, and why they stayed unconfirmed)
## Control coverage   (control id | PASS/PARTIAL/FAIL/NOT VERIFIABLE/N/A | evidence)
## Strengths worth noting
```

The delivery pass may run the test suite if it needs no secrets, network
services or paid APIs. Record the command and the result.

## Step 3: Verify

This step matters most. Review passes overrate severity and sometimes
report things that aren't there.

- **Re-read every Critical and High yourself**, directly from the code.
  Trace the whole chain: the route, the middleware that should protect it,
  the query, the policy. If any link breaks, downgrade or drop it.
- **Recalibrate severity** with the scale in `TEMPLATE.md`. Critical needs
  someone with no special access (a stranger, or any user who can sign up)
  to reach other people's data, money, or the whole platform today. If it
  needs a leaked key, an insider, or a second failure, it isn't Critical.
- **Ask who can reach it.** A flaw behind an internal-only interface is
  rated by who can actually call that interface. Define "unprivileged"
  before rating anything.
- **When a fact outside the repo decides severity, ask.** For example: does
  every new signup become an "owner"? Is the repo public? Does their own
  auth service throw or return `valid: false` for a bad key? The user
  usually knows, and asking doesn't pause the review. If there's no
  answer, rate on the behaviour you can see (the other service's code or
  contract if it's in the repo). If it's a fact about the owner's own
  setup that you can't see, rate the higher of the two readings and add a **Condition** line
  naming the fact that would lower it ("Medium if email confirmation is
  on in the Supabase project"). Apply this the same way to every finding.
  A Critical that rests on an unconfirmed Condition says so in its
  summary bullet too, so the owner checks that fact first.
  How a third-party service behaves is not an owner fact: check its
  documentation, and if you still can't confirm the worst case, rate what
  you can confirm and list the rest under what you couldn't check.
- **Committed credentials** are Critical if the credential is or ever
  was publicly readable anywhere: this repo or its fork network while
  public, the public template or starter it was copied from, a published
  package, or source maps. Otherwise High. Either way the first action is
  to rotate them, and removing them from git history comes after.
- **Merge duplicates** across passes into one finding, rated by your own
  re-verification, not by whichever pass rated it highest or lowest. A
  missing test or missing alert is usually Medium: it becomes an incident
  only when something else breaks.
- **A credential that unlocks a whole system is at least High wherever it
  leaks** (a browser, a URL, a log, a committed file), even when the
  audience looks small today.
- **Reconcile contradictions.** If one pass marks a control PASS and
  another proved it broken, the verified finding wins. One finding may
  explain several controls; each control still gets its own status and
  cites the finding.
- **Keep the cleared suspicions.** Anything a pass raised that you
  disproved goes in the appendix with the reason, so nobody re-raises it.

## Step 4: Write the report

Write the report to `.risk-review/risk-review-<YYYY-MM-DD>.md` at the repo
root, following `TEMPLATE.md` exactly. Check whether `.risk-review/` is
ignored by git (`git check-ignore .risk-review`) and record the answer in
the report header. If it isn't ignored, tell the user and offer to add it
to `.gitignore`: the report is a map for attackers and must not end up in
a public repo. That `.gitignore` line is the only change you may make,
and only after the user says yes. If the user hasn't answered by the time
you finish, leave `.gitignore` alone and say so in the header.

- Every one of the 59 controls appears in section 3 with a status.
- Severity counts in section 1 match section 2, and the coverage totals
  add up to 59. Count before you finish.
- Critical, High and Medium findings carry every field in the template.
  Lows need Evidence, Fix, Effort and a fix prompt.
- Fix plan rows may group findings that share one fix; list them all in
  the Closes column.
- Every finding has a **fix prompt**: a self-contained instruction the
  owner can paste into their AI coding tool. Name the files, the change,
  what must not change, and (for Critical, High and Medium) the test to
  add. When the fix is a dashboard setting or a document, not code, name
  the check that proves it instead of a test. End it with "Show me the
  diff before applying it." Write it for someone who may not read code.
- Strengths are part of the report, not padding.

## Step 5: Hand over

Tell the user, in plain language:

1. where the report is, and that it should stay out of public repos;
2. the Critical and High findings in one line each;
3. any credential you found, and that it needs rotating now, before
   anything else;
4. what you could not check (the NOT VERIFIABLE controls and where they
   live);
5. that this is an AI review that can be wrong: fix the Criticals, then
   ask a person they trust to look at anything they're unsure of.

Offer to fix findings one at a time, starting with the first row of the
fix plan, writing the Verify test first. Don't start fixing unless the
user says so.

## Lessons from real reviews

These came from running this method on real codebases:

- The worst finding is usually a logic flaw, not a pattern: an admin check
  that accepts "owner of your own workspace" when every signup becomes an
  owner, or a tenant id read from a header. Scanners miss these; reading
  the auth path end to end finds them.
- Passes overrate. In one review the passes reported five Criticals; all
  five needed access only an internal service had, so they were High.
- Passes also underrate. A "Medium" missing deploy gate turned out to let
  untested code reach production with no human in the loop, which is High.
- Before calling a third-party API broken, check the call exactly as the
  app makes it (same headers, same API version) against the provider's
  documentation or the SDK's types for the locked version. Don't call the
  live API to find out. A check that ignored the app's settings once
  produced a wrong finding. If you can't confirm the behaviour, rate what
  you can confirm and list the rest under what you couldn't check.
- Reliability problems get dressed up as security ones. A webhook that
  answered 400 to unhandled events was first rated High on the claim that
  the provider would disable it and billing would stop syncing. But the
  events the app needed still got 200, so the certain impact was retries
  and alert noise: Medium.
- Quote the line the code actually runs. A finding about a client-chosen
  trial length first named the wrong field; the code sent a computed
  `trial_end`, which changes the fix.
- A dependency installed from git or a URL is not a hallucinated package
  just because the registry has no package by that name.
