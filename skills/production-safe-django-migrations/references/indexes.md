# Index Migrations

**Cross-cutting rule:** every index add/remove on an existing table is **concurrent,
non-atomic, no timeouts, one per migration**:

```python
class Migration(TimeoutAwareMigration):
    atomic = False
    lock_timeout = None          # CONCURRENTLY cannot run under a lock/statement timeout
    statement_timeout = None
    operations = [ AddIndexConcurrentlyWithRetry(...) ]   # exactly one
```

`AddIndexConcurrentlyWithRetry` / `RemoveIndexConcurrentlyWithRetry` (from the toolkit's
`operations.postgres` module) run `DROP INDEX CONCURRENTLY IF EXISTS <name>` before
(re)creating, which recovers from a prior failed `CREATE INDEX CONCURRENTLY` that left an
**invalid index** behind. Stock `AddIndex`/`RemoveIndex` on an existing model is rejected
by the linter (`PreferIndexConcurrently`). *Only the newer services use the toolkit; some
older/legacy services use plain `migrations.Migration` with non-concurrent `AddIndex` —
treat those as legacy, not the standard.*

## Adding an index concurrently

- **Why:** a non-concurrent `CREATE INDEX` takes a `SHARE` lock that blocks all writes for
  the entire build; on a large table that's minutes of write downtime.
- **How:** `AddIndexConcurrentlyWithRetry(model_name=..., index=...)`, `atomic=False`,
  timeouts `None`.
- **Risks:** builds are slow and I/O-heavy; a failed build leaves an INVALID index (the
  retry op's pre-drop handles this). Can't run in a transaction.
- **Rollback:** the reverse drops it concurrently (`RemoveIndexConcurrentlyWithRetry` on
  reverse).
- **Deployment:** schedule large builds off-peak; the index is additive so old code is
  unaffected.
- **Locks:** `SHARE UPDATE EXCLUSIVE` (does not block reads or writes).
- **Example** — `NNNN_add_record_path_gist_index.py`:
  ```python
  AddIndexConcurrentlyWithRetry(
      model_name="record",
      index=GistIndex(fields=["path"], name="app_record_ltree_path"),
  )
  # SQL: CREATE INDEX CONCURRENTLY "app_record_ltree_path" ON "app_record" USING gist ("path");
  ```

## Removing an index

- **Why/How:** same lock reasoning; `RemoveIndexConcurrentlyWithRetry(model_name=, name=)`,
  `atomic=False`.
- **Rollback:** reverse re-creates it concurrently (with a pre-drop for safety).
- **Locks:** `SHARE UPDATE EXCLUSIVE`. (Non-concurrent `DROP INDEX` is a brief
  `ACCESS EXCLUSIVE` — still avoid on hot tables because it queues behind running queries.)
- **Example** — `NNNN_drop_record_category_location_index.py`; siblings that drop other
  now-unused indexes one per file.

## Composite (multi-column) indexes

- **How:** `models.Index(fields=["a", "b", "c"], name=...)` — order matters (leftmost-prefix
  rule); put the most selective / most-filtered column first, add `-col` for a DESC sort
  column when the query orders by it.
- **Example (concurrent + partial + DESC)** — `NNNN_add_order_user_sort_index.py`:
  ```python
  AddIndexConcurrentlyWithRetry(
      model_name="order",
      index=models.Index(
          fields=["user_id", "-last_activity_at"],
          condition=Q(last_accessed_at__isnull=False),
          name="app_order_user_sort_idx"),
  )
  # SQL: CREATE INDEX CONCURRENTLY "..." ON "..." ("user_id", "last_activity_at" DESC) WHERE "last_accessed_at" IS NOT NULL;
  ```

## Partial / conditional indexes

- **Why:** index only the rows you query (e.g. active/enabled/non-null), shrinking the
  index and speeding writes for the excluded rows. Also enables **conditional uniqueness**
  ("unique among active rows").
- **How:** `models.Index(..., condition=Q(...))`, or a PostgreSQL-extensions library's
  `ConditionalUniqueIndex(condition="...", fields=[...], name=...)` for a partial UNIQUE
  index. Always built concurrently.
- **Risk:** a query only uses a partial index if its `WHERE` is provably implied by the
  index predicate.
- **Examples:**
  - Partial conditional-unique index —
    `ConditionalUniqueIndex(condition='"category_id" IS NULL AND "author_id" IS NULL', fields=["language","kind","status"], name=...)`.
  - Multi-column partial index —
    `condition=Q(("enabled", True), ("linked", True))` → `WHERE ("enabled" AND "linked")`.
  - "Unique active record" — `WHERE NOT ("state" = 'archived')`.

## Expression / functional indexes

- **Prefer materializing over `Index(expressions=...)` functional indexes.** When you need
  to index a derived value, materialize it instead:
  - **Compute a column, then index the plain column** — a `RunSQL` sets
    `payload_md5 = md5(payload::text)`, then a normal `Index` on that column.
  - **`GeneratedField` (STORED) + index the column** — a `Coalesce(...)` generated column;
    note this is a non-concurrent, atomic migration with real timeouts
    `lock_timeout=15s, statement_timeout=20s`.
- **Takeaway:** prefer a stored generated column + ordinary index over a raw expression
  index — it's introspectable, works with the ORM, and indexes concurrently like any column.

## JSONB indexes

- **Common pattern:** a **GIN index with a JSONB opclass** on a `jsonb` column:
  ```python
  GinIndex(fields=["metadata_path"], name="app_record_meta_idx", opclasses=["jsonb_path_ops"])
  # SQL: CREATE INDEX "..." ON "..." USING gin ("metadata_path" jsonb_path_ops);
  ```
- `jsonb_path_ops` is smaller/faster for containment (`@>`) queries than the default
  `jsonb_ops` (which also supports key-existence `?`). Choose by query shape.
- **Avoid indexing an `HStoreField`/a localized-field type** by default — hstore columns are
  often guarded by CHECK constraints, not indexed. Add a GIN index only if you actually query
  hstore keys/values.

## GIN indexes

- **Why:** the right structure for multi-value columns — `ArrayField`, full-text/trigram,
  jsonb, `intarray`, `inet`. Choose the **opclass** by data + query:
  | Column / query | opclass |
  |----------------|---------|
  | `ArrayField` containment | (default) |
  | trigram / `ILIKE` / fuzzy text | `gin_trgm_ops` (needs `TrigramExtension()`) |
  | jsonb `@>` containment | `jsonb_path_ops` |
  | int arrays | `intarray.gin__int_ops` |
  | `inet`/CIDR | `inet_ops` |
- **How:** build concurrently on existing tables. Since stock `AddIndexConcurrently` can't
  pre-drop, some services pair it with an explicit `RunSQL('DROP INDEX CONCURRENTLY IF
  EXISTS ...;', reverse_sql="")` and a `# migrationlinter: ignore
  core.NoNonAtomicWithMultipleOperations` note (the ops are individually re-entrant).
- **Examples:** a plain `GinIndex(["category_slugs"])`; a `gin_trgm_ops` index paired with
  `TrigramExtension()`; a concurrent GIN swap via `SeparateDatabaseAndState`; an `inet_ops`
  index (plain non-concurrent, legacy).

## BRIN indexes

- **Rarely needed.** When you would reach for one: BRIN suits very large,
  naturally-ordered append-only tables (time-series by `created_at`) where a tiny
  block-range index is enough. Build it concurrently and non-atomically like any other
  index; note it helps only when physical row order correlates with the indexed column.

## Unique constraints, concurrently (the canonical recipe)

Django's `AddConstraint(UniqueConstraint)` builds its backing index under
`ACCESS EXCLUSIVE`, and `AddIndexConcurrently` refuses unique indexes — so the only
lock-free path is raw SQL in **two migrations**:

1. **Non-atomic migration:** `CREATE UNIQUE INDEX CONCURRENTLY ...` (`atomic=False`,
   timeouts `None`).
2. **Atomic migration:** `ALTER TABLE ... ADD CONSTRAINT ... UNIQUE USING INDEX
   "<prebuilt>"` (brief `ACCESS EXCLUSIVE`, with real `lock_timeout=15s/statement_timeout=20s`).

Keep Django's state in sync with `SeparateDatabaseAndState` (state side = `RemoveConstraint`
+ `AddConstraint(UniqueConstraint(...))`). A good example (`NNNN_product_frequency_unique.py`)
carries a comment explaining exactly this reasoning. Contrast the anti-pattern: a plain
`AddConstraint(UniqueConstraint)` — locks the table for the whole index build.

- **Rollback:** dropping a unique constraint is a brief `ACCESS EXCLUSIVE`; make the reverse
  a `noop` if re-adding could fail (`NoReversibleRemoveConstraint`).
- **Deployment:** the constraint is additive but *rejects violating writes the moment it
  exists* — ensure the app already can't create duplicates before adding it.

## The SQL comment header

The linter writes a `"""SQL: ..."""` + `"""Reverse SQL: ..."""` docstring at the top of
each managed migration showing the estimated raw SQL (including the `SET LOCAL
statement_timeout=...; SET LOCAL lock_timeout=...` for timeout migrations). Reviewers
review the SQL. Regenerate with the migration-lint command; a stale header fails CI.
