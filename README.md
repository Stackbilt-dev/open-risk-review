# Open Risk Review

A production risk review for your app, run by your own AI.

You built an app with Lovable, Bolt, v0, Cursor, Claude Code or Replit, and
it's about to take payments or hold other people's data. Open Risk Review
gives your AI coding tool a tested method for checking it first: 59
controls across auth, tenant isolation, secrets, billing, AI-specific risk,
logging, testing, deployment and supply chain.

The result is a report with:

- **Findings with evidence.** Each one cites the file and line, says what
  could happen, and says how to fix it.
- **A fix prompt for every finding.** Paste it into your AI tool to make
  the change, even if you don't read code.
- **A coverage table.** All 59 controls, marked PASS, PARTIAL, FAIL, NOT
  VERIFIABLE or N/A, so you can see what was checked and what wasn't.
- **A fix plan** ordered by how much risk each fix removes per hour of work.

It runs entirely in your own tool. Nothing is sent to us, and there's
nothing to install beyond a few markdown files.

## Install

### Claude Code

```text
/plugin marketplace add Stackbilt-dev/open-risk-review
/plugin install open-risk-review@open-risk-review
```

Then, in your project, ask: **"Run an open risk review on this repo."**

Or install it as a personal skill without the plugin system:

```bash
git clone --depth 1 https://github.com/Stackbilt-dev/open-risk-review
cp -r open-risk-review/skills/open-risk-review ~/.claude/skills/
```

### Cursor

From your project root:

```bash
git clone --depth 1 https://github.com/Stackbilt-dev/open-risk-review /tmp/open-risk-review
cp -r /tmp/open-risk-review/skills/open-risk-review .open-risk-review
mkdir -p .cursor/rules .cursor/commands
cp /tmp/open-risk-review/cursor/open-risk-review.mdc .cursor/rules/
cp /tmp/open-risk-review/cursor/commands/risk-review.md .cursor/commands/
```

On Windows, run these in Git Bash or WSL, or use any folder in place of
`/tmp`.

Then type **`/risk-review`** in the agent chat, or ask: **"Run an open risk review on this repo."** Cursor runs the four review passes one after another.

### Any other AI coding tool

Copy `skills/open-risk-review/` to `.open-risk-review/` in your project and tell
your agent:
**"Read `.open-risk-review/SKILL.md` and follow it."** The method is plain
markdown and works with any agent that can read files and search code.

## What it does

1. **Scope.** Records the exact commit, asks what the app does and what you
   worry about, and lists the outside systems it can't see. If you don't
   answer, it assumes the app is live, takes payments and holds other
   people's data, and says so in the report.
2. **Survey.** Maps routes, auth, data, payments, secrets, AI calls, tests
   and deployment.
3. **Four review passes.** Auth & API; billing, data & secrets; AI &
   operations; delivery & supply chain. Run in parallel where your tool
   supports subagents.
4. **Verification.** Re-reads every Critical and High finding directly from
   the code, and drops or downgrades anything that doesn't hold up. AI
   reviewers overrate severity; this step is why the method exists.
5. **Report.** Writes `.risk-review/risk-review-<date>.md` and walks you
   through the top findings.

It is read-only. It doesn't change code, call your live systems, or use any
credential it finds.

**See a full report:** [examples/northstar-projects.md](examples/northstar-projects.md),
written for a made-up app so it exposes nobody's real code.

## What it isn't

An AI model reads your code and reports what it finds. It can miss things
and it can be wrong, so confirm a finding before acting on it. This is not
a penetration test, a compliance assessment (SOC 2, ISO 27001, HIPAA), or a
statement that your app is safe. It can't see settings that live outside
the repository, such as your Supabase dashboard or Stripe account; those
controls are marked NOT VERIFIABLE instead of guessed.

**Keep the report private.** It describes how to attack your app. Don't
commit it to a public repository; the skill reminds you to add
`.risk-review/` to `.gitignore`.

## Who made this

[Stackbilt](https://stackbilder.com) built this from the method behind our
paid [AI App Production Risk Review](https://stackbilder.com/production-risk-review).
We're open about the connection: the method here is the same 59-control
checklist and the same steps. In the paid review we run it for you, and a
named person personally verifies every Critical and High finding before you
see it, then walks you through the results on a call. You don't need the
paid review to use this, and nothing here phones home.

## Files

| File | What it is |
|---|---|
| `skills/open-risk-review/SKILL.md` | The method your AI follows |
| `skills/open-risk-review/CHECKLIST.md` | The 59 controls, what to look for, and stack hints (Next.js, Supabase, Vercel, Stripe, Firebase, Express, Hono, Fastify, Cloudflare Workers, FastAPI, Django, Flask, Lovable / Bolt / v0 exports) |
| `skills/open-risk-review/TEMPLATE.md` | The report structure |
| `cursor/open-risk-review.mdc` | Cursor rule that points the agent at the method |
| `cursor/commands/risk-review.md` | Cursor `/risk-review` command |
| `examples/northstar-projects.md` | A complete example report for a fictional app |
| `.claude-plugin/` | Claude Code plugin and marketplace manifests |

## Contributing

Issues and pull requests are welcome, especially:

- a control the checklist misses, with a real-world example;
- stack hints for frameworks not listed;
- a finding the method got wrong (a false alarm or a miss), with what
  would have caught it.

Please don't post real vulnerabilities in other people's apps in public
issues.

## License

MIT. See [LICENSE](LICENSE).
