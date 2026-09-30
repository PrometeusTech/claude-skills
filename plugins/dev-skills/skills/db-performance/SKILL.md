---
name: db-performance
description: Rules and review checklist for query and list performance — pagination, N+1 queries, eager loading, indexes, EXPLAIN, search/filter queries, aggregates, JSON columns, large-table migrations, and the frontend side of big lists (duplicate requests, debounce, stale responses, huge DOMs). Use it whenever you add or change a list/index endpoint, a filter or search, a serializer that walks associations, a dashboard/count, a migration or index, or a page that renders many rows — and whenever you review such changes or investigate "this page is slow", even if nobody mentioned performance.
---

# Database and list performance

Performance bugs in CRUD apps are rarely clever: a list without pagination that is fine with 20
rows and takes 4 seconds with 450, a serializer that fires one query per row, a filter on a column
with no usable index, a page that renders 30,000 DOM nodes. They are cheap to prevent while
building and expensive to find later, so treat these rules as part of "done".

Stack specifics:
- `references/rails-activerecord.md` — ActiveRecord patterns, query-count tests, Bullet
- `references/mysql.md` — MySQL 8 indexes, EXPLAIN, collations, JSON, online DDL
- `references/react-lists.md` — React data fetching and big lists

## 1. Every list that can grow is paginated

- Page/per-page (or cursor) with a default and a hard maximum; stable ordering with a unique
  tiebreaker (`created_at DESC, id DESC`, or `title, id`), otherwise pages skip or repeat rows.
- Return pagination metadata the client needs (current page, per page, total or "has more").
  `COUNT(*)` on a big filtered set has a cost — only include totals the UI uses.
- When converting an existing unpaginated endpoint, keep the old behavior behind an explicit
  opt-in (e.g. only paginate when `page` is sent) until every client migrated, then remove it.
- Invalid page/per-page → 400/422, not 500; per-page above the max is clamped or refused.

## 2. Query count is constant per request

- Eager load exactly what the serializer touches — no more (unused includes cost memory), no less
  (N+1). Check the serializer, not the controller, to know what is needed.
- Anything computed per row from associations (sums, "active borrow", "has disputes") becomes one
  aggregate query or a preloaded association with the filter in SQL — not Ruby over every loaded
  row.
- Guard it with a test that counts queries for 2 records and for N records and asserts they are
  equal.

## 3. Indexes match how you query

- For each new query: which columns filter, which order? An index on `(tenant_id, filter_col,
  sort_col)` usually serves "this tenant's items with status X, newest first".
- Left-prefix rule: `(a, b)` also serves `WHERE a = ?`; a separate index on `a` is then redundant
  (but keep one that a foreign key needs).
- Uniqueness the model validates must also be a unique index (otherwise races create duplicates);
  before adding one to an existing table, audit for duplicates under the column's collation.
- `LIKE '%term%'` cannot use a B-tree index. Fine for small per-tenant sets; measure and note the
  threshold where full-text search becomes necessary.
- JSON arrays (tags) cannot be indexed directly; for frequent filtering use a join table, a
  generated column or a multi-valued index — or accept a per-tenant scan with a measured limit.
- Prove it with `EXPLAIN` on a realistic volume, not an empty dev database.

## 4. Measure with realistic data

Seed a realistic volume (script in a scratch directory, not in the repo): the largest tenant you
expect × a margin. Record before/after: response time (median of a few warm requests), query count,
response size. Put the numbers in the PR. "Feels fast locally" with 5 rows proves nothing.

## 5. Migrations on large tables

- Know whether the DDL is online (MySQL 8 `ALGORITHM=INSTANT/INPLACE`, `LOCK=NONE`) or copies
  the table. Note it in the PR with the table's row count.
- Data migrations run in batches; unique indexes run after a duplicate audit; everything is
  reversible or explicitly irreversible with a reason.
- Deploy order matters when code and schema change together (add column → deploy code → backfill
  → constraint).

## 6. The frontend is half of "slow"

- One request per need: no duplicate fetches on mount (StrictMode double effects, parents and
  children both fetching), filters/options fetched once, not on every keystroke.
- Debounce free-text search; cancel or ignore stale responses (AbortController / request id) so a
  slow old response cannot overwrite a newer one.
- Paginate or virtualize long lists; keep DOM nodes in the low thousands. Show a loading indicator
  when changing pages without blanking the list.
- Lazy-load heavy pages/routes; do not ship admin-only code in the main bundle.

## Implementation checklist

- [ ] New/changed lists paginated with stable order, max per-page, validated params.
- [ ] Serializer associations eager loaded; per-row computations moved to SQL/preload.
- [ ] Query-count test (2 vs N records) for each new list endpoint.
- [ ] Indexes for new filters/orders; no redundant index added; unique constraints backed by indexes.
- [ ] Measured on realistic volume; numbers in the PR.
- [ ] Migration: online or not, row count, batching, reversibility noted.
- [ ] Frontend: no duplicate requests, debounce, stale-response guard, pagination/virtualization.

## Review / judge checklist

1. Seed volume; hit each new endpoint; record time, query count (log / Bullet / counter), size.
2. Scale the data ×10: does time or query count grow with rows? (It should not grow per row.)
3. `EXPLAIN` each new WHERE/ORDER combination: index used? rows examined vs returned?
4. Look for Ruby loops over relations, `.map(&:assoc)`, `.count` inside serializers, `.all` on
   growing tables, per-row HTTP/storage calls.
5. Aggregations: "list of tags used", dashboards, counters — computed in SQL or by loading every row?
6. New unique index: duplicate audit exists? Collation considered (case/diacritics)?
7. Frontend network tab: requests per page load and per filter change; stale response race;
   DOM node count on the largest realistic list; bundle size change.
