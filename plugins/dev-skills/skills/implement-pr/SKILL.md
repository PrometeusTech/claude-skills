---
name: implement-pr
description: Workflow for implementing a feature, fix or refactor as one reviewable pull request in a real codebase — read the project rules and the plan, load the matching topic skills, write tests first, keep scope tight, verify locally with the project's own CI steps, self-review, and report honestly. Use it whenever the user asks to implement, build, add, fix or refactor something that will end up as a PR ("implement step 3 of the plan", "add the documents page", "fix the upload bug", "do the next item in the plan"), especially in multi-repo (backend + frontend) projects.
---

# Implement a change as one clean PR

The goal is a PR a reviewer can merge with confidence: it does what the plan says, nothing more,
it is proven by tests and by the project's own checks, and it says honestly what it did not do.

## 1. Load context before touching code

1. Read the project's agent instructions (`AGENTS.md`, `CLAUDE.md`, contributing docs) in every
   repo you will change. They override generic habits here (naming, layering, language of
   user-facing messages, how to run things).
2. Read the plan / issue / section you are implementing, plus any "discovered", "follow-up" or
   "decisions" notes that target it. Decisions already taken are not reopened in the PR.
3. Decide which topic skills apply and follow them while building, not only at review time:
   - files, attachments, downloads, storage/CDN, upload validators → **file-uploads-cdn**
   - any endpoint, action, policy, role, serializer field, route guard → **authz-multitenancy**
   - lists, filters, search, serializers over associations, migrations, indexes, big pages →
     **db-performance**
   - any free-text field (titles, tags, names, file names), uniqueness or de-duplication →
     **text-input-hardening**
   - create endpoints, idempotency keys, retries, concurrent writes, external HTTP clients →
     **idempotency-retries**
   - params, redirects, fetching URLs, cookies/CORS, logging, analytics events, error handling,
     public endpoints → **web-security**
   - pages, forms, modals, routes, translations, data fetching → **frontend-quality**
   - migrations, env vars, jobs, dependencies, breaking API changes, deploy order →
     **deploy-safety**
   - statuses, amounts, dates and ranges, bookings/stock, counters, soft delete, audit trail →
     **domain-integrity**
   - e-mails, notifications, message templates, webhooks, provider clients →
     **notifications-integrations**
   Most features touch at least two. Also read the project's own `*-checks` skills in
   `.claude/skills/` of each repo, if any: they hold the project's rules and win over the generic
   ones. If something important is unclear (a product rule, who may
   do what), ask before building rather than guessing in code.

## 2. Tests first

Write the tests that express the requirement and watch them fail for the right reason, then
implement. For behavior changes to existing flows, first pin current behavior (a characterization
test), then change it on purpose and show the diff of expectations. Cover the matrices the topic
skills name (roles × tenants, accepted × rejected file types, 2 vs N query counts, hostile strings,
same-key concurrent requests). Assert effects, not fields: rows and storage objects left after a
failure, the file name actually served, the second request's outcome.

## 3. Keep the scope tight

- Implement what the plan says. Bugs or improvements you notice outside it go into the plan's
  "discovered" list (or the PR description) with a proposed follow-up — not silently into this PR.
- Changing shared infrastructure (upload pipeline, auth, base controllers, global interceptors) is
  in scope only if the feature needs it, and then every other consumer gets a regression check.
- Follow the project's layering (thin controllers, services, policies, serializers; managers /
  view-models on the frontend). Match surrounding code style.
- API changes update the contract (OpenAPI source files, generated bundle, error codes and their
  translations) in the same PR; the frontend syncs the contract instead of hand-writing types.

## 4. Verify like CI would

- Find the project's CI definition and any local runner script (`bin/ci`, `make check`, an npm `ci` script,
  `make check`…). Run all of it locally before pushing — tests, lint, type check, security scan,
  schema drift, contract checks, build — and fix what fails. Do not rely on remote CI being up.
- If a step cannot run in your environment, say which and why; do not report it as passing.
- For performance-relevant changes, measure on realistic volume and put the numbers in the PR.
- For UI, run the app and try the flow for each affected role at a laptop width and on mobile.

## 5. Review yourself before anyone else does

- Run the available review tooling on your diff (e.g. `/code-review`; `/security-review` when the
  change touches auth, input handling, files, external calls or personal data — it reviews the
  current branch against the default branch), then re-read the diff as an adversary: what input, role, race or failure would break this? Apply the topic skills' review
  checklists to your own work.
- Fix what you find before pushing. One validated push beats three speculative ones.

## 6. Hand over honestly

- Update the plan / status docs the project keeps (status row, details, discovered items).
- PR description: what changed and why, how it was verified (commands and results), what was not
  done and why, follow-ups discovered. Link the sibling PR in multi-repo changes and state merge
  order (backend first when the frontend depends on a new contract).
- Never skip, disable or loosen a test or check to get green; never claim a check passed that you
  did not run.
