---
name: production-safe-django-migrations
description: How to write Django schema & data migrations that deploy to a live, high-traffic PostgreSQL database with zero downtime — using a timeout-aware migration base class, concurrent indexes, lock/statement timeouts, expand-contract multi-release changes, batched RunPython, and a migration linter. Load this BEFORE writing, reviewing, or editing ANY migration (makemigrations output, RunPython/RunSQL, indexes, adding/removing/altering columns, constraints, foreign keys, or data backfills) in any Django service — and when setting up migration safety in a new Django service.
---

# Production-Safe Django Migrations

Migrations run against a **live, high-traffic PostgreSQL database**, and — critically —
**migrations are applied BEFORE the new application code rolls out.** So for a window
during every deploy, the *old* code runs against the *new* schema (and during a rolling
deploy, old and new code run simultaneously). Every rule in this skill exists to make
that window safe. A migration that takes a heavy lock, rewrites a table, or removes
something the old code still uses causes an outage.

These rules are encoded in an **in-house migration-safety toolkit** (a
`TimeoutAwareMigration` base class, concurrent-index operations, a migration **linter**
modeled on established strong-migration linters, and a **reorderer**). The linter runs
automatically inside `makemigrations` and in CI (the migration-lint command, in `--fix`
mode). Treat a linter finding as a real production risk, not a nit.

## The two golden rules

1. **Never take a long lock on a hot table.** Either the operation is instant
   (metadata-only) or it runs `CONCURRENTLY`, or it has a bounded `lock_timeout`/
   `statement_timeout` so it fails fast instead of blocking traffic.
2. **Every schema change is backward compatible with the currently-running code.**
   If it isn't, split it across **multiple releases** (expand → migrate → contract).
   Old code + new schema must coexist; new schema + old code must coexist.

## The migration-safety toolkit foundation

### `TimeoutAwareMigration` — mandatory base class
Every migration inherits the toolkit's `TimeoutAwareMigration`, **never**
`migrations.Migration`. The custom `makemigrations` generates it automatically. It
**requires** both `lock_timeout` and `statement_timeout` attributes to exist (else
`apply()` raises `TimeoutNotProvidedError`) and wraps the operation's SQL in those
timeouts — but only when the migration actually emits SQL.

```python
from datetime import timedelta
from migration_toolkit.migration import TimeoutAwareMigration

class Migration(TimeoutAwareMigration):
    lock_timeout = timedelta(seconds=15)       # fail fast if a lock can't be grabbed
    statement_timeout = timedelta(seconds=20)  # fail fast if the statement runs too long
    dependencies = [...]
    operations = [...]
```

- **When to set real timeouts:** anything that emits DDL against an **existing** table
  (AddField, AlterField that rewrites/scans, AddConstraint, FK). `lock_timeout` **must be
  < `statement_timeout`** (otherwise the statement timeout always fires first — enforced).
- **When to set `None`, `None`:** state-only migrations (`AlterModelOptions`, an
  `AlterField` that only changes `choices`/`on_delete`) and **all `CONCURRENTLY`
  migrations** (see below). The linter's `RedundantMigrationTimeouts` flags timeouts on a
  migration whose only SQL is the timeout statements themselves.
- **Why timeouts, mechanically:** atomic migrations use `SET LOCAL` (auto-reset on
  commit); non-atomic use `SET SESSION` and must reset, because otherwise **pgbouncer
  connections get corrupted with leftover timeouts**.

### `atomic` — one operation per non-atomic migration
Concurrent operations **cannot run in a transaction**, so those migrations set
`atomic = False`. The linter (`NoNonAtomicWithMultipleOperations`) forbids more than one
operation in a non-atomic migration: if one fails you get a half-applied migration
needing manual repair. **One concurrent operation = one migration file.**

### Concurrent indexes with retry
Index create/drop on a big table takes a lock that blocks writes (or everything). Always
use the concurrent, retry-safe operations from the toolkit's `operations.postgres` module:
`AddIndexConcurrentlyWithRetry` / `RemoveIndexConcurrentlyWithRetry`. They run
`DROP INDEX CONCURRENTLY IF EXISTS` first, which recovers from a prior failed
`CREATE INDEX CONCURRENTLY` that left an **invalid index** behind (otherwise the retry
dies with "relation already exists"). The linter auto-converts `AddIndex`/`RemoveIndex`
→ the concurrent retry variants, sets timeouts to `None`, and splits multi-index
migrations into one-per-file.

### The SQL comment header
The linter's `SQLComment` rule inserts (and keeps up to date) a comment at the top of
each migration showing the **exact forward and reverse SQL** it will run. Reviewers read
the SQL, not the Python. Run the linter in `--fix` mode to generate it. If it's stale,
CI fails.

### The reorderer — migration ordering
The toolkit reorders migrations by number + modification time, renames them, and
rewrites `dependencies` to resolve conflicts when two branches both add migration
`0123_*`. Use it to fix ordering after a rebase/merge rather than hand-editing
`dependencies`.

## What the linter forbids (each is a production hazard)

| Rule | Forbids | Because |
|------|---------|---------|
| `PreferTimeoutAwareMigration` | plain `migrations.Migration` / missing timeouts | unbounded lock/statement duration |
| `MissingTimeoutWithoutConcurrently` | non-`CONCURRENTLY` DDL with no timeouts | long DDL holds locks indefinitely |
| `PreferIndexConcurrently` | `AddIndex`/`RemoveIndex` on existing model | full table lock during index build |
| `PreferUniqueConstraintConcurrently` | `AddConstraint(UniqueConstraint)`/`RemoveConstraint` | unique constraint = unique index = table lock |
| `ConcurrentIndexInTimeout` | `CONCURRENTLY` + timeouts in same migration | non-atomic + timeouts breaks the migration |
| `NoAddFieldWithDefault` | non-null `AddField` with a default | old code's INSERTs fail (no db-level default) |
| `NoNonStaticDefault` | callable default on AddField/AlterField | callable runs once, same value for all rows |
| `NoNullOnly` / `NoBlankOnly` | text field with only `null=True` or only `blank=True` | must pair them; NULL is the single empty representation |
| `NoRemoveField` / `NoDeleteModel` | dropping a field/model | old code still queries it during the deploy window → split over 2 releases |
| `NoRename` | renaming a field/model/column | old code expects the old name → split over 3 releases |
| `NoIrreversiblePython` / `NoIrreversibleSQL` | `RunPython`/`RunSQL` with no reverse | must be rollback-able; use `.noop` if truly none |
| `NoReversibleRemoveConstraint` | reversible `RemoveConstraint` | re-adding on rollback can fail if new rows violate it → make reverse a `noop` |
| `NoNonAtomicWithMultipleOperations` | >1 op in a non-atomic migration | half-applied migration on failure |
| `PreferExplicitIndex` | `db_index=True` on a field | keep indexes in `Meta.indexes` for visibility |
| `PreferUniqueConstraint` | `unique=True` / `unique_together` | use `Meta.constraints` (UniqueConstraint) |
| `PreferTextField` | `CharField` | no perf gain; avoid length constraints |
| `RespectMaxIdentifierLength` | index/constraint name > 63 chars | Postgres silently truncates |

## Deep-dive references

For the per-pattern treatment (**Why / How / Risks / Rollback / Deployment strategy /
Expected PostgreSQL locks / example**) read the relevant file:

| Pattern family | Reference |
|----------------|-----------|
| Nullable columns, NOT NULL, defaults, table rewrites, FK, PK, UUID | `references/columns-keys-constraints.md` |
| Concurrent add/remove, composite, partial, expression, JSONB, GIN, BRIN, unique | `references/indexes.md` |
| RunPython/RunSQL, batching/chunked updates, backfills, backward-compat, feature flags, ordering, rolling/concurrent deploys | `references/data-and-deployment.md` |
| PostgreSQL lock levels & which operation takes which | `references/postgres-locks.md` |

## Decision tree — how to write this migration

```
1. Is this a DATA change (backfill / transform rows), not a schema change?
   → RunPython (with reverse or .noop), batched (see data-and-deployment.md).
     Where available, prefer a runtime data-patch system over a data migration.

2. Am I ADDING something?
   • Nullable column (TextField null=True, blank=True)? → single AddField, timeouts set. Safe.
   • Column that needs a value on old rows? → add it NULLABLE now; backfill separately;
     enforce NOT NULL in a later release. NEVER AddField(non-null, default=...).
   • Index? → AddIndexConcurrentlyWithRetry, atomic=False, timeouts=None, one per migration.
   • Unique constraint? → CREATE UNIQUE INDEX CONCURRENTLY (own migration), then
     ADD CONSTRAINT ... USING INDEX (own migration, brief lock).
   • Foreign key? → add nullable FK column; ADD CONSTRAINT ... NOT VALID (brief lock);
     VALIDATE CONSTRAINT in a separate migration (no write-blocking lock).
   • Check / NOT NULL enforcement? → ADD CONSTRAINT ... CHECK (...) NOT VALID, then VALIDATE.

3. Am I CHANGING something in place?
   • Only choices/on_delete/verbose_name (no SQL)? → AlterField, timeouts=None (state-only).
   • Column type / size / a real rewrite? → DON'T. Add a new column + backfill + swap
     (expand/contract over releases). A rewrite takes ACCESS EXCLUSIVE for the whole table.
   • Rename? → 3 releases: add-new+dual-write → stop using old → drop old. Never RenameField.

4. Am I REMOVING something?
   • The code still references it? → STOP. Remove the code first (this or an earlier
     release), then drop the field/model in a LATER release.
   • Index? → RemoveIndexConcurrentlyWithRetry, atomic=False.
   • Constraint? → RunSQL DROP CONSTRAINT with reverse_sql=noop (don't auto re-add on rollback).

5. Did I set lock_timeout < statement_timeout, atomic correctly, a reverse, and run the
   linter (--fix) so the SQL comment is current? → see the checklists below.
```

## Migration checklist (before you commit the file)

- [ ] Inherits `TimeoutAwareMigration`; `lock_timeout` **and** `statement_timeout` set —
      real values for DDL on existing tables, `lock_timeout < statement_timeout`.
- [ ] `None, None` for state-only migrations and all `CONCURRENTLY` migrations.
- [ ] Index add/remove uses `AddIndexConcurrentlyWithRetry`/`RemoveIndexConcurrentlyWithRetry`,
      `atomic = False`, **one index per migration**.
- [ ] No `CONCURRENTLY` operation shares a migration with timeouts, or with another operation.
- [ ] No `AddField` with a non-null default; new columns are `null=True, blank=True`.
- [ ] No `RemoveField`/`DeleteModel`/`RenameField` unless the code no longer references
      it AND it's the correct release in the expand/contract sequence.
- [ ] Every `RunPython`/`RunSQL` has a reverse (or explicit `.noop`); data backfills are batched.
- [ ] `RemoveConstraint` reverse is a `noop` (won't fail re-adding on rollback).
- [ ] FK / NOT NULL / unique done via the concurrent / NOT VALID→VALIDATE two-step.
- [ ] Index & constraint names ≤ 63 chars; indexes in `Meta.indexes`, uniqueness in `Meta.constraints`.
- [ ] The migration-lint command (`--fix`) run — SQL comment header current, no findings.
- [ ] Migration is backward compatible with the currently-deployed code (old code + new schema OK).
- [ ] Dependencies/ordering correct after any rebase (reorderer if needed); no duplicate numbers.

## Production deployment checklist

- [ ] **Sequence understood:** migrations apply *before* code; plan for old-code-vs-new-schema
      (and, in a rolling deploy, both code versions at once).
- [ ] **Expand/contract:** additive/backfill this release; enforcing/removing in a later
      release once all instances run the new code.
- [ ] **Lock risk assessed on the actual table size.** Estimate row count; anything that
      scans/rewrites a large hot table is rejected in favor of concurrent / NOT VALID→VALIDATE.
- [ ] **Concurrent index builds are out-of-transaction and can take minutes/hours** — they
      don't block traffic but do consume I/O; schedule big ones off business hours;
      full rebuilds never during business hours.
- [ ] **Timeouts sized to the table** so a bad migration fails fast and the deploy rolls back
      cleanly rather than piling up lock waits.
- [ ] **Rollback path exists:** every operation reversible or a deliberate `.noop`; know what
      reversing does to data (re-adding constraints can fail).
- [ ] **Feature-flag the code path**, not the schema, for risky behavior changes; flag defaults off.
- [ ] **Backfills run as batched jobs / data-patches**, not one giant `UPDATE` in a migration.
- [ ] **New settings/flags** the migration or new code needs are configured in every target env.
- [ ] **Monitor** locks (`pg_locks`, `pg_stat_activity`) and error rates during and after apply.

## Common mistakes

1. **`AddField(null=False, default=...)`** on a big table → old INSERTs fail (Django doesn't
   keep a db-level default) and/or a table rewrite. Add nullable, backfill, enforce later.
2. **Creating an index without `CONCURRENTLY`** → `SHARE` lock blocks all writes for the whole build.
3. **Putting timeouts on a `CONCURRENTLY` migration** → it's non-atomic and long-running; the
   timeout kills it mid-build, leaving an invalid index.
4. **`SET NOT NULL` directly** → `ACCESS EXCLUSIVE` + full table scan blocks everything. Use a
   `CHECK (col IS NOT NULL) NOT VALID` then `VALIDATE`.
5. **`ADD CONSTRAINT ... FOREIGN KEY`/`UNIQUE` in one shot** → validates under a heavy lock. Split
   into build-index/NOT VALID + VALIDATE.
6. **Dropping a column/model in the same release the code stops using it** → old code hits the
   missing column during the deploy window.
7. **`RenameField`/`RenameModel`** → old code queries the old name. Three-release dance instead.
8. **Reversible `RemoveConstraint`** → rollback tries to re-add a constraint new rows now violate.
9. **Un-batched `RunPython` `UPDATE`/`.all()` loop** → loads/locks the whole table, long transaction,
   replication lag, OOM.
10. **Multiple operations in a non-atomic migration** → half-applied on failure, manual repair.
11. **Editing an already-applied/released migration** instead of adding a new one.
12. **Importing app code/constants into a migration** — migrations are immutable history; inline
    constants and small helpers instead.

## New-service setup

- Add the in-house migration-safety toolkit; override `makemigrations` with its linter mixin so
  new migrations are generated as `TimeoutAwareMigration` and auto-linted with `--fix`.
- Configure the linter in `pyproject.toml`: `rule_groups = ["core", "postgres", ...]`,
  the app list, and `ignore_before`/`recent_n` if adopting on an existing codebase.
- Wire the migration-lint command into CI and the pre-deploy release script.
- Base models on a Postgres-aware base model (from a PostgreSQL-extensions library); TextField-only,
  `Meta.indexes`/`Meta.constraints` (see a companion backend-engineering reference).
