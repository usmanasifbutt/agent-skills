# Columns, Keys & Constraints

Each pattern below: **Why / How / Risks / Rollback / Deployment / Locks / Example.** Only
the newer services use the migration-safety toolkit; some older/legacy services hand-roll
raw SQL (`SET statement_timeout='600s'; CREATE INDEX CONCURRENTLY ...`) to solve the same
problems — legacy, not the standard.

## Adding a nullable column (the default, safe pattern)

- **Why:** a nullable `AddField` on PG 11+ is metadata-only — instant, no rewrite.
- **How:** `AddField(field=models.TextField(null=True, blank=True))` (pair `null` +
  `blank`; the linter's `NoNullOnly`/`NoBlankOnly` require both). `TimeoutAwareMigration`
  with `lock_timeout=15s`, `statement_timeout=20s`.
- **Risks:** minimal. The brief `ACCESS EXCLUSIVE` can still queue behind a long query —
  the `lock_timeout` bounds that.
- **Rollback:** reverse drops the column (brief lock).
- **Deployment:** additive; old code ignores the new column. Single release.
- **Locks:** brief `ACCESS EXCLUSIVE` (metadata only).
- **Example** — `NNNN_add_record_is_featured.py`:
  ```python
  class Migration(TimeoutAwareMigration):
      lock_timeout = timedelta(seconds=15)
      statement_timeout = timedelta(seconds=20)
      operations = [migrations.AddField(model_name="record", name="is_featured",
          field=models.BooleanField(default=False, null=True))]
  # SQL: ALTER TABLE ... ADD COLUMN ... boolean DEFAULT false NULL; then ALTER COLUMN ... DROP DEFAULT;
  ```
  Django's own `DROP DEFAULT` afterward is fine here because a **constant** default doesn't
  rewrite (PG 11+) and `null=True` keeps it out of the `NoAddFieldWithDefault` rule. This
  is the overwhelmingly common shape, usually `TextField(null=True, blank=True)`.

## Adding a column that must carry a value (NOT NULL / needed default)

- **Why NOT the obvious way:** `AddField(default=..., null=False)` is banned by
  `NoAddFieldWithDefault`. Django adds the value to existing rows then **drops the db-level
  default**, so between the migration and the code rollout, old code's INSERTs (which don't
  specify the new column) fail. A non-constant default also rewrites the table.
- **How:** raw `RunSQL` that adds the column WITH a db-level default and **deliberately
  keeps it** (omit the `DROP DEFAULT`), plus `state_operations=[AddField(...)]` to sync
  Django's state.
- **Risks:** you own the raw SQL; keep forward/reverse correct.
- **Rollback:** reverse `DROP COLUMN ... CASCADE`.
- **Deployment:** the db-level default means in-flight INSERTs from old code succeed. Safe
  in one release.
- **Locks:** brief `ACCESS EXCLUSIVE` (constant default, no rewrite on PG 11+).
- **Example** — `NNNN_add_record_tenant_default.py`:
  ```python
  migrations.RunSQL(
      f'ALTER TABLE "app_record" ADD COLUMN "tenant_id" text DEFAULT \'{APP_DEFAULT_TENANT_ID}\' NOT NULL;',
      'ALTER TABLE "app_record" DROP COLUMN "tenant_id" CASCADE;',
      state_operations=[migrations.AddField(model_name="record", name="tenant",
          field=models.TextField(default=APP_DEFAULT_TENANT_ID, db_column="tenant_id"))],
  )
  # comment: "Raw SQL because Django would drop the default to force a table rewrite. We do
  #           NOT want to rewrite the table; a db-level default is cheap since Postgres 11."
  ```
- **Never** use a callable default on AddField/AlterField (`NoNonStaticDefault`) — Django
  evaluates it once and writes the same value to every row.

## Making an existing column NOT NULL

- **Why NOT `AlterField(null=False)`:** `SET NOT NULL` takes `ACCESS EXCLUSIVE` and full-
  scans the table, blocking everything.
- **How (two migrations):**
  1. `ADD CONSTRAINT <c> CHECK (col IS NOT NULL) NOT VALID` — brief lock, no scan.
  2. In a **separate** migration: `VALIDATE CONSTRAINT <c>` — `SHARE UPDATE EXCLUSIVE`,
     scans but doesn't block reads/writes. (On PG 12+ a validated CHECK lets a later
     `SET NOT NULL` skip its scan, if you want the real NOT NULL.)
- **Risks:** rows must already satisfy the predicate before VALIDATE (backfill first).
- **Rollback:** forward reverse = `DROP CONSTRAINT IF EXISTS`; the VALIDATE step's reverse
  is `RunSQL.noop`.
- **Locks:** brief `ACCESS EXCLUSIVE`, then `SHARE UPDATE EXCLUSIVE`.
- **Example** — `NNNN_add_published_at_not_null_check_1.py`
  (`ADD CONSTRAINT ... CHECK ("published_at" IS NOT NULL) NOT VALID`) + `NNNN_..._2.py`
  (`VALIDATE CONSTRAINT`, reverse `noop`).
- **Variant with backfill** — `NNNN_add_status_not_empty_check.py`:
  one `atomic=False` migration using `RunWithTimeouts(lock_timeout=15s, operations=[...])`:
  `ADD CONSTRAINT CHECK (col <> '') NOT VALID` → `UPDATE ... SET col=NULL WHERE col=''` →
  `VALIDATE CONSTRAINT`. (`RunWithTimeouts` is the per-op timeout wrapper for non-atomic
  migrations; it errors if `lock_timeout >= statement_timeout`. It's now deprecated in
  favor of `TimeoutAwareMigration` but appears in history.)

## Avoiding table rewrites (the recurring theme)

The in-code rationale comments to internalize:
- *"Django would drop the default right after creating it to force the table to be
  rewritten. We really do not want to rewrite the entire table; a db-level default is cheap
  since Postgres 11 — it just updates metadata."*
- *"Add the FK as NOT VALID so the entire table isn't checked immediately while holding an
  ExclusiveLock."* / *"VALIDATE in a separate transaction… only obtains a ShareUpdateExclusive lock."*
- *"AddConstraint builds the backing index under ACCESS EXCLUSIVE… the only lock-free path
  is raw SQL: build the UNIQUE INDEX CONCURRENTLY, then ADD CONSTRAINT … USING INDEX."*

Rule: any operation that would **scan or rewrite the whole table** (SET NOT NULL, ALTER
TYPE, inline FK/UNIQUE validation) is converted to a NOT VALID→VALIDATE or
build-index-concurrently-then-attach two-step, or done as add-new-column + backfill + swap
across releases.

## Adding a foreign key to an existing table

Two accepted approaches:

**(a) Single nullable `AddField(ForeignKey)` under a bounded lock.** Relies
on `lock_timeout=15s`/`statement_timeout=20s` to bound the whole thing (column + FK +
index) rather than splitting.
- **Example** — `NNNN_add_record_default_category_fk.py`:
  `ForeignKey(null=True, blank=True, on_delete=SET_NULL, to="app.category")`.
- **Locks:** `ACCESS EXCLUSIVE` + validation scan — acceptable only because the timeout
  makes it fail fast; use for small/medium tables.

**(b) Explicit 3-migration split for large tables:**
1. Add the column (raw `RunSQL`, `state_operations=[AddField]`).
2. `ADD CONSTRAINT ... FOREIGN KEY (...) REFERENCES ... NOT VALID DEFERRABLE INITIALLY
   DEFERRED` (brief lock, no scan).
3. `VALIDATE CONSTRAINT` (`SHARE UPDATE EXCLUSIVE`, reverse `noop`).
- **Example** — `NNNN_add_record_organization_fk_{1,2,3}.py` (paired series: the column
  add, the NOT VALID FK, then the VALIDATE, each its own file).
- **Deployment:** the column is nullable and additive; the FK is validated last, so no
  step blocks writes.

## Primary-key migrations

- **What exists:** the `AutoField`→`BigAutoField` bump (from `DEFAULT_AUTO_FIELD`) — a
  **state-only** change for existing serial PKs, so `lock_timeout=None`,
  `statement_timeout=None`, empty SQL. Example: `NNNN_alter_ids_to_bigautofield.py`
  (many `AlterField(id=BigAutoField(...))` ops). New tables define the PK at `CreateModel`
  time (`id bigint ... GENERATED BY DEFAULT AS IDENTITY`).
- **What to avoid:** an actual PK *swap* (changing which column is the PK) on a populated
  table rewrites the table and its FKs — avoid it. If ever needed: build a
  new unique index concurrently, `ADD CONSTRAINT ... PRIMARY KEY USING INDEX` (brief lock,
  column must already be NOT NULL), migrate FKs across releases.

## UUID migrations

- **The safe case:** UUID as a PK **at initial table creation** (empty table = no risk)
  — e.g. a service's `0001_initial.py` with `UUIDField(default=uuid.uuid4,
  primary_key=True)`. Adding a UUID column to an existing table, or converting a PK to
  UUID, is the risky case handled below.
- **If you must add a UUID to an existing table:** add it nullable (`UUIDField(null=True)`),
  backfill in batches (a callable default would write the same UUID to every row —
  forbidden by `NoNonStaticDefault`; generate per-row in a `RunPython` backfill instead),
  then enforce NOT NULL / uniqueness via the concurrent + NOT VALID→VALIDATE steps across
  releases.

## AlterField that changes only options / `on_delete` (state-only)

- **How:** `TimeoutAwareMigration` with `lock_timeout=None`, `statement_timeout=None`
  (no DB SQL → real timeouts would be redundant, flagged by `RedundantMigrationTimeouts`).
- **Example** — `NNNN_alter_order_user_ondelete.py`:
  ```python
  class Migration(TimeoutAwareMigration):
      lock_timeout = None
      statement_timeout = None
      # migrationlinter: ignore NoRename This only changes the on_delete option in state.
      operations = [migrations.AlterField(model_name="order", name="user",
          field=models.ForeignKey(..., db_constraint=False, db_index=False, on_delete=PROTECT))]
  ```

## State-only migrations & `SeparateDatabaseAndState`

A frequently-used tool. Key uses:
- **Drop a field from Django state without dropping the column** — the safe first half of
  the `NoRemoveField` two-release dance: `SeparateDatabaseAndState(database_operations=[],
  state_operations=[RemoveField(...)])` (`NNNN_remove_record_category_from_state.py`).
  A later release drops the actual column.
- **Keep Django state in sync with hand-written DDL** — every raw-`RunSQL` column/FK/unique
  migration passes `state_operations=[...]` (RunSQL is itself a mini `SeparateDatabaseAndState`).
- **Unique-constraint swap without a rewrite** — `NNNN_product_frequency_unique.py`
  (concurrent index in `database_operations`, `RemoveConstraint`/`AddConstraint` in
  `state_operations`, `atomic=False`, timeouts `None`).

## Removing a column or model — split across two releases

`NoRemoveField` / `NoDeleteModel` forbid dropping in the same release the code stops using
it (old code hits the missing column during the deploy window). Sequence: **release 1** —
stop referencing it in code (optionally remove from state via `SeparateDatabaseAndState`);
**release 2** — drop the actual column/table. Renames are worse — never `RenameField`;
do the three-release add-new → dual-write → drop-old dance (`NoRename`).
