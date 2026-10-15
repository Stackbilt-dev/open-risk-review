# Changelog

## 1.0.0 — 2026-10-15

- First release: the Open Risk Review method (`SKILL.md`), the 59-control
  checklist with stack hints (`CHECKLIST.md`), the report template with
  per-finding fix prompts (`TEMPLATE.md`), a Claude Code plugin and
  marketplace manifest, a Cursor rule and `/risk-review` command, and a
  worked example report for a fictional app (`examples/northstar-projects.md`).
- Tested cold on a Next.js + Supabase + Stripe app. From that run: a
  stated default when the owner doesn't answer (marked as assumed in the
  report's **Context** line), one rule for when something missing is FAIL
  or NOT VERIFIABLE, framework defaults accepted as PASS evidence with a
  version, Condition lines only for facts about the owner's setup
  (third-party behaviour is checked in its docs), public registries and
  docs allowed for confirmation, credentials publicly readable anywhere
  (including a template they were copied from) treated as public, and new
  Supabase, Vercel and Stripe hints (public-catalogue `using (true)`,
  readable `@supabase/ssr` cookies, `getSession()` on the server, client
  trial lengths, email customer lookups).
