---
name: domain-integrity
description: Rules and review checklist for business logic that must stay correct over time — status workflows and state machines, money and decimal amounts, dates, times and time zones, bookings and availability (overlaps, double booking), counters and cached totals, invariants that span several tables, transaction boundaries, soft-deleted records, and an audit trail of who changed what. Use it whenever you add or change a status, a transition, a price/amount/total, a date range or schedule, a reservation or stock rule, a counter, a multi-step write or an admin action that changes someone else's data, and whenever you review such code — even if the task only says "add a cancelled status" or "show the total".
---

# Domain integrity

The bugs that cost the most in business apps are quiet: a booking cancelled after it was
completed, two reservations for the same tool on the same day, a total that is off by 0.01, a
deadline that is "today" in UTC but "tomorrow" in Bucharest, a counter that drifts from the rows it
counts, an admin change nobody can trace. They pass every happy-path test. The rules below keep
the model honest; the checklist is what a reviewer tries.

## 1. Status changes go through one state machine

- One place defines the states and allowed transitions per entity; services call it, controllers
  never assign `status` directly, and params never contain `status`.
- Terminal states (completed, cancelled, closed) have no outgoing transitions unless the product
  says so (e.g. reopen), and that transition is explicit.
- Each transition states who may trigger it (role) and its side effects (notifications, refunds,
  stock); side effects run **after commit** of the transition.
- Concurrent transitions on the same record are serialized (`with_lock` / optimistic locking /
  conditional `UPDATE … WHERE status = 'expected'`), so "approve" and "cancel" racing cannot both
  win.
- Illegal transitions raise a domain error the API maps to a documented 4xx/409, never a 500 and
  never a silent no-op reported as success.

## 2. Money

- Amounts are `decimal` with a fixed scale in the DB and `BigDecimal` in code; never floats
  (`to_f`, JS `Number` arithmetic on amounts).
- Rounding is decided once (half-up / banker's, at which step) and applied in one helper; totals
  are computed from rounded line items, and the sum of the parts equals the shown total.
- Currency is explicit when more than one is possible.
- Amounts that must stay separate (deposits vs charges, guarantees vs obligations) are not summed
  by convenience; each has its own field and label.
- Negative, zero and very large amounts are validated against product rules and column capacity.

## 3. Dates, times and time zones

- Store timestamps in UTC; decide business dates ("today", "due on", "this month") in the
  product's time zone, configured once (e.g. `config.time_zone`, `Time.zone.today`), never
  `Date.today` / `Time.now` / `new Date()` local to the server or browser.
- A "date" column is a calendar date, not a timestamp at midnight UTC.
- Ranges state whether ends are inclusive; an end date of "the 31st" includes that day.
- Month ends, leap days, DST switches (the 23- and 25-hour days) and year boundaries are tested
  with frozen time (`travel_to`, fake timers).
- Durations computed in days use calendar days in the business zone, not 24-hour blocks.

## 4. Availability and overlaps

- "No two bookings overlap" is enforced under concurrency: a lock on the resource row, an
  exclusion/unique constraint, or a conditional insert — not a `where(...).exists?` check followed
  by `create` in another statement.
- Overlap test is `start_a < end_b AND start_b < end_a` (adjust for inclusive ends) — check the
  edge where one ends the day the other starts.
- Stock / quantity decrements are atomic (`UPDATE … SET qty = qty - 1 WHERE qty > 0`) or locked.

## 5. Invariants across tables

- Write down the invariants the feature relies on (a counter equals its rows, a parent's total
  equals its children, a membership exists for every active role, one active row per key) and
  where they are enforced (DB constraint, transaction, recomputation).
- Cached counters / totals are updated in the same transaction as the rows, or recomputed; an
  audit task (e.g. a `data:audit` invariant) checks them in production.
- A multi-step write is one transaction; external effects (emails, storage, HTTP) are outside it
  and after commit.
- Callbacks that change other records are visible in the service, not hidden in model callbacks
  that fire on every save.

## 6. Soft delete

- Soft-deleted records are excluded from lists, counts, lookups, uniqueness checks (or the unique
  index includes the deleted flag deliberately), associations and background jobs.
- Restoring a record re-validates uniqueness and dependent state.

## 7. Audit trail

- Actions that change another person's data or money (admin edits, role changes, approvals,
  refunds, deletions) record who, what, when, from which tenant, and the before/after of the
  relevant fields.
- Audit rows are append-only, tenant-scoped, and exclude secrets.

## Implementation checklist

- [ ] Transitions only via the state machine; terminal states closed; races serialized; illegal
      transitions → documented 4xx/409.
- [ ] Amounts in decimal/BigDecimal; one rounding helper; parts sum to the total; separate
      amounts kept separate.
- [ ] Business dates in the configured zone; inclusive/exclusive ends stated; edge dates tested
      with frozen time.
- [ ] Overlaps and stock enforced atomically, not check-then-insert.
- [ ] Invariants listed with their enforcement; counters in the same transaction; audit task.
- [ ] Soft-deleted rows excluded everywhere they should be.
- [ ] Audit rows for admin actions on others' data or money.

## Review / judge checklist (try to break it)

1. Draw the transition table from the code; try every transition from every state via the API
   (including from terminal states) as each role.
2. Fire two conflicting transitions on the same record concurrently (approve vs cancel, two
   approvals): exactly one wins, the other gets the documented error, side effects happen once.
3. Amounts: 0, 0.005, 0.015, a negative, the column maximum; a total of many items with
   fractions — does the sum of shown parts equal the shown total? Any `to_f` / float math?
4. Freeze time at 23:30 and 00:30 Bucharest (21:30 / 22:30 UTC), on the last day of a month, on
   Feb 29 and on DST days; check "today", deadlines, ranges and reports.
5. Two concurrent bookings for the same resource and overlapping or touching ranges; a stock of 1
   claimed twice at once.
6. Change rows behind a counter or total (create, delete, soft-delete, restore): do the counter
   and the audit query still agree?
7. Soft-delete a record and look for it in lists, counts, search, suggestions, jobs and unique
   checks.
8. Perform an admin action on another user's data: is there an audit row with who, what, before
   and after?
