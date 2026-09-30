# ActiveRecord specifics

## Eager loading
- `preload` (separate queries) is the safe default; `includes` switches to a JOIN when you also
  filter on the association (`references`); `eager_load` forces the JOIN. Pick deliberately.
- Filtered associations: define a scoped association (`has_many :active_borrows, -> { active }`)
  and preload that, instead of preloading everything and filtering in Ruby.
- Preload after authorization and only for the fields the serializer uses; different actions
  (index vs show) usually need different includes — keep them per action, not one big constant.
- `ActiveRecord::Associations::Preloader.new(records:, associations:).call` for manual preloads.

## Aggregates
- `group(:x).count`, `sum`, `pluck` instead of loading records. For "distinct tags used in the
  tenant" on a JSON column, use SQL (`JSON_TABLE` in MySQL 8) or a join table — not
  `Model.where(...).flat_map(&:tags).uniq`.
- Counter caches for counts shown in lists.

## Pagination
- A shared concern (`Paginatable`) that validates `page`/`per_page`, applies `limit/offset`, adds
  the stable order tiebreaker and builds the metadata. Reuse it; do not hand-roll per controller.
- Deep offsets get slow on huge tables; switch that endpoint to keyset/cursor pagination when it
  matters.

## Detecting N+1
- Bullet in development and in a few targeted tests (`Bullet.raise = true` around the request).
  Bullet has false positives/negatives — confirm with a query-count test.
- Query-count helper: subscribe to `sql.active_record` (ignore `SCHEMA`/`TRANSACTION`/cached) and
  assert equal counts for 2 and N records.

## Serialization
- Time spent in Ruby serializers often dominates after the DB is fixed: return a lighter list
  representation (list-lite) and keep expensive computed fields for the detail endpoint.

## Migrations
- The MySQL adapter accepts `add_index ..., algorithm: :inplace` (also `:instant`, `:copy`);
  asking explicitly makes the migration fail instead of silently copying the table. Check an
  unfamiliar change on a copy with `ALTER TABLE ... ALGORITHM=INPLACE, LOCK=NONE` if unsure.
- Use `change_column_null` / defaults carefully on big tables; backfill in batches
  (`in_batches(of: 1_000)`), outside the schema migration if long.
- Keep `db/schema.rb` in sync and committed with the migration.
