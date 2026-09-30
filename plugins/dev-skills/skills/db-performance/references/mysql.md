# MySQL 8 specifics

## EXPLAIN
- `EXPLAIN FORMAT=TREE` / `EXPLAIN ANALYZE` (8.0.18+) shows the plan and actual rows. Look at:
  access type (`ref`/`range` good, `ALL` = full scan), `rows` examined vs returned, `Using
  filesort`, `Using temporary`.
- Run it on realistic data; the optimizer's choices on an empty table are meaningless.

## Indexes
- Composite index order: equality columns first, then the range/order column.
  `(tenant_id, status, created_at)` serves `WHERE tenant_id=? AND status=? ORDER BY created_at DESC`.
- MySQL 8 can scan an index backwards, so `DESC` order does not need a descending index.
- Redundant: an index that is a left prefix of another (unless required for a foreign key, which
  needs *some* index starting with the FK column).
- Unique indexes follow the column collation: with `utf8mb4_0900_ai_ci`, "Ana" = "ana" = "Ană".
  Audit duplicates with `GROUP BY col HAVING COUNT(*) > 1` under the same collation first.

## Text search
- `LIKE 'term%'` can use an index; `LIKE '%term%'` cannot. Per-tenant filtering first (indexed)
  then `LIKE` on the remaining rows is fine up to tens of thousands of rows per tenant.
- `FULLTEXT` indexes (InnoDB) for real search; mind the minimum token size and stopwords.

## JSON columns
- Not directly indexable. Options: generated column + index for a scalar path; multi-valued
  index (`CAST(tags->'$' AS CHAR(64) ARRAY)`) for `MEMBER OF` / `JSON_CONTAINS`; or a join table,
  which is usually simplest for tags.
- `JSON_TABLE` turns an array into rows for "distinct values used" queries.

## Online DDL
- Most `ADD INDEX` / `ADD COLUMN` are online (`INPLACE`/`INSTANT`) in 8.0; changing a column
  type or charset usually copies the table. Verify with an explicit `ALGORITHM=...` clause on a
  copy; the statement fails instead of silently locking if the algorithm is not possible.

## Locks and concurrency
- `SELECT ... FOR UPDATE` inside a transaction to serialize state changes on a row; keep the
  transaction short; deadlocks are possible — retry a small number of times, then 409.
