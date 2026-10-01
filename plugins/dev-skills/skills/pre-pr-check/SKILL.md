---
name: pre-pr-check
description: Light, diff-only quality gate to run at the end of a change, before committing the last time and opening the pull request — runs the project's local CI gate, Claude Code's /code-review and /security-review on the branch, applies the project's *-checks and the matching topic-skill checklists to the changed files only, adds cheap test probes for new fields, roles and error codes, fixes what blocks, and writes a short "Pre-PR check" section for the PR that tells the later judge what was already verified. Use it at the end of implement-pr, or whenever the user asks to "check before the PR", "verify my changes", "is this ready to push", "run the checks" — not as a substitute for the full feature-judge on the open PR.
---

# Pre-PR check

A cheap pass that catches the common problems while the change is still in your hands, so the
pull request — and the full judge that runs on it later — starts from a clean diff. It reads code
and runs tests; it does **not** start the app, seed volume, run concurrency experiments or drive
a browser. Budget: tens of thousands of tokens, minutes — not a judge run.

**Scope requested:** $ARGUMENTS (empty = the current branch in each repo you changed in this
session)

## 1. Scope

For each repo touched: the diff of the current branch against the default branch, plus any
uncommitted changes (`git diff <default>...HEAD` and `git diff`). List the changed files and group
them by area (endpoints/policies, models/migrations, uploads, text fields, jobs/notifications,
pages/routes, translations, contract).

## 2. Gate

Run the project's full local gate on the final state (the local CI script named in the project's
agent docs or `*-checks` skill). If it already ran green on exactly this state in this session,
say so instead of re-running. Fix failures before going on; never skip or loosen a test.

## 3. Built-in reviews

- `/code-review` on the branch (medium effort): correctness bugs.
- `/security-review` on the branch when the diff touches authentication, authorization, input
  handling, files, redirects/URLs, external calls, cookies/CORS, logging/analytics or personal
  data. It compares the current branch with the default branch, so run it from the branch.

Treat their output as candidates: confirm each against the code (and a test when cheap) before
acting on it.

## 4. Checklists, on the changed files only

Read the project's `*-checks` skills in `.claude/skills/` first, then the review checklists of the
topic skills that match the changed areas (the mapping table in the feature-judge skill). For each
item that applies to **this diff**, check it statically (read the code) and mark it OK / problem /
left for the judge. Skip items that need the running app or volume — list them as left for the
judge rather than guessing.

Always, whatever the diff:
- Every new endpoint authorizes, with the exact roles from the plan; lists mirror details.
- New params go through the project's shape guards; new text fields are cleaned and length-limited
  before any side effect.
- Contract updated (OpenAPI source + bundle), error codes per operation match the code, and every
  code has translations; generated clients regenerated, not edited.
- No secrets, tokens or personal data in logs, job arguments, analytics or error-tracker context.
- No swallowed errors (`rescue`/`catch` that reports success).
- Plan / status docs updated.

## 5. Cheap probes as tests

Where a check above is cheaper to prove than to argue, add a small permanent test rather than a
throwaway script: a role that must be refused, a foreign-tenant id, a malformed param, a hostile
string (NUL, bidi override, over-long) in each new text field, each new error code. Run them.

## 6. Fix, re-run, report

- Fix every **blocking** problem (security, authorization, data loss, 500 on input, contract
  drift, failing gate) now, then re-run the gate.
- Non-blocking problems: fix if small and in scope; otherwise list them as follow-ups.
- Add a **Pre-PR check** section to the PR description (or return it, if no PR is being opened):

```
### Pre-PR check
- Gate: <script> → <result>
- /code-review: <n findings → fixed / refuted (why)>
- /security-review: <run on branch X → n findings → fixed / refuted> | not needed (<why>)
- Checklists applied: <skills> — <n items OK, n fixed>
- Probes added as tests: <list>
- Left for the judge: <items that need the running app, browser, volume or concurrency>
- Follow-ups: <non-blocking items not done>
```

The judge reads this section and spends its budget on what is left, so be exact: only write
"OK" for what you actually verified.
