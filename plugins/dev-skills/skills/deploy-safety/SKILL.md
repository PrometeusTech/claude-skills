---
name: deploy-safety
description: Rules and review checklist for changes that must survive the deploy itself — migration order and reversibility, schema and code deployed at different moments, new environment variables and secrets, backward compatibility between a new API and the old frontend still open in browsers (and the reverse), background jobs lost on restart or run twice, scheduled jobs that overlap or span many tenants, new or upgraded dependencies (security advisories, licenses, lockfiles), feature flags and rollback. Use it whenever a change adds a migration, an env var, a job or scheduler, a dependency, a breaking API change, or anything that needs a specific deploy order, and whenever you review such changes — even if the task only says "add a column" or "bump the gem".
---

# Deploy safety

A change can be correct in the repository and still break production: the migration locks a big
table, the old frontend in users' tabs calls a field that no longer exists, a job enqueued before
the deploy runs against new code, a new env var is missing on the server, a dependency brings a
known vulnerability. The rules below make the deploy boring; the checklist is what a reviewer
asks before merge.

## 1. Migrations

- Each migration is reversible, or says why not (`irreversible` with a reason).
- Know whether the DDL is online or copies the table, and the table's size in production (see
  db-performance §5). Data backfills run in batches, outside the schema migration when large.
- Constraints that existing data might violate (NOT NULL, unique, foreign key) come with an audit
  query and a plan for the rows that fail it, run **before** the migration in production.
- The schema file is regenerated and committed with the migration.

## 2. Expand, then contract

Code and schema do not change at the same instant, and old clients linger:

- **Adding:** add the column/endpoint/field first (nullable or with a default), deploy code that
  writes and reads it, backfill, then add constraints.
- **Removing or renaming:** stop using it in code and deploy; only then drop it in a later
  release. A rename is add + copy + switch + drop.
- **API:** a field the current frontend reads is not removed or retyped in the same release; new
  required params are optional for one release, or the frontend ships first. Breaking changes are
  versioned or coordinated, and the contract check (e.g. oasdiff) is green or explicitly accepted.
- **Frontend after a deploy:** users keep old tabs open for hours — old bundles must keep working
  against the new API until they reload; chunk-load failures after a deploy reload cleanly.
- State the deploy order in the PR (backend first, then frontend, or the reverse) when it matters.

## 3. Configuration and secrets

- Every new env var is documented in the env template with a safe example, read in one place,
  and validated at boot when required (fail fast, not at first use).
- Defaults are safe for production (feature off, strict security), not convenient for development.
- New secrets are added to the server before the deploy; no fallback from one secret to another.

## 4. Background and scheduled jobs

- Jobs enqueued before the deploy may run after it: job arguments stay compatible (or the job
  handles both shapes) for one release.
- With an in-process / non-durable queue, jobs in flight are lost on restart: anything that must
  happen (emails, payments, cleanup) is re-derivable from DB state, or uses a durable mechanism.
- Jobs are idempotent (they may run twice); schedulers run in one process only and are guarded
  against running in consoles, rake tasks and tests.
- A scheduled run must not overlap the previous one (a lock or a "running since" marker with
  expiry); if a run takes longer than the interval, the next one is skipped, not stacked.
- Jobs over many tenants or rows work in batches, commit per batch, have a time budget, and can
  resume where they stopped; one tenant's failure does not stop the others (reported, then
  continue).

## 5. Dependencies

- New or upgraded gems / packages: why needed, maintained, license compatible, no known advisory
  (`bundler-audit`, `yarn audit` / `npm audit`); lockfile committed and consistent.
- Major upgrades read the changelog for breaking changes and are not mixed into a feature PR.
- Bundle impact for frontend dependencies (see frontend-quality §7).

## 6. Rollback

- Know how to roll back: can the previous release run against the migrated schema? If not, say so
  in the PR and keep the migration additive.
- Risky behavior goes behind a flag that can be turned off without a deploy, when the project has
  flags.

## Implementation checklist

- [ ] Migration reversible (or reasoned), online/size noted, audit for new constraints, schema committed.
- [ ] Expand/contract respected; old frontend works against new API; deploy order stated.
- [ ] Env vars documented, validated at boot, safe defaults; secrets provisioned first.
- [ ] Job args backward compatible; jobs idempotent; lost-job impact understood.
- [ ] Scheduled runs cannot overlap; multi-tenant jobs batched, resumable, isolated per tenant.
- [ ] Dependencies justified, audited, lockfiles committed; no major upgrades hidden in features.
- [ ] Rollback path known.

## Review / judge checklist

1. For each migration: run `migrate` and `rollback` locally; check the schema diff; estimate the
   lock on the production table size; is there an audit for new constraints?
2. Run the **base** frontend against the **head** backend (and the reverse if the frontend ships
   first): does every existing screen still work?
3. Diff the contract: removed/retyped fields or new required params? Breaking-change check result?
4. `grep` the diff for new `ENV[...]` / `process.env` reads: documented? validated? safe default?
5. Enqueue a job with the base code's arguments and run it with the head code. Start a
   scheduled job twice at once; make it fail on one tenant out of several.
6. New dependencies: audit output, license, maintenance, size; lockfile matches the manifest.
7. Write down the rollback: can the base release boot against the head schema?
