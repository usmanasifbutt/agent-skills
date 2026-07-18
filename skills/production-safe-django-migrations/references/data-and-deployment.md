# Data Migrations, Ordering & Deployment Strategy

## Where data changes belong: migration vs data-patch

The dividing line to use:
- **Schema-coupled, one-time backfill that must run in lockstep with a column/constraint
  change → a data migration** (RunPython, usually split from the DDL).
- **Standalone reference/static data the production DB owns** (category fields, labels,
  translations, enum values) → a **separate runtime data-patch system** (in services that
  have one), *not* a migration.

The runtime data-patch system: a `DataPatch(ABC)` base with `portals` (a list of target
portals), `models`, and `run()`; versioned by numeric filename prefix; applied once and
tracked in a sync-state table; portal-scoped idempotent upserts (e.g. a versioned patch
that adds a new category field). This is why category/label/translation "data migrations"
are scarce in the migration tree of services that have it — they live in the patch system
instead. Services without a patch system do their rarer seeding directly in RunPython
migrations.

## RunPython data migrations

- **Always reversible** — enforced by `NoIrreversiblePython`. Provide a real `reverse_code`
  or explicit `migrations.RunPython.noop`. CI fails on a bare `RunPython`.
- **Get models via `apps.get_model("app", "Model")`** (the historical-state model), never a
  direct import — the model's real shape may differ from the current code at the migration's
  point in history.
- **Reverse defensively:** a reverse that undoes a forward change must tolerate concurrent
  state (e.g. only drop a shared crontab "if nothing else points at it — an admin could have
  linked another task between forward and reverse").
- **Locks:** whatever the queries take; a long unbatched `UPDATE`/`DELETE` holds row locks
  and a long transaction — batch it (below).
- **Examples:** a real reverse + cross-app dependency (`NNNN_periodic_task_log_db_load.py`);
  a one-way cleanup with `noop` reverse (`NNNN_record_url_remove_double_slash.py`).

```python
def forwards(apps, schema_editor):
    Record = apps.get_model("app", "Record")
    Record.objects.filter(source_url__startswith="//").delete()

operations = [migrations.RunPython(forwards, migrations.RunPython.noop)]
```

## Batching / chunked updates

- **Why:** a single `.update()`/`.all()` loop over a big table = one giant transaction, long
  row locks, replication lag, and possible OOM.
- **How:** `bulk_update(objs, [fields], batch_size=5000)` / `bulk_create(batch_size=...)`.
  Split the **population** (RunPython, re-runnable, no side effects) from the **DDL** that
  depends on it (RunSQL), mark the migration `atomic=False`, and add an explicit
  `# migrationlinter: ignore NoNonAtomicWithMultipleOperations <reason>` since the pop step
  is safely re-runnable.
- **Example** — `NNNN_backfill_external_id.py`:
  ```python
  class Migration(migrations.Migration):
      # migrationlinter: ignore NoNonAtomicWithMultipleOperations the 1st step is just a
      # population script without side-effects and can be re-run multiple times
      atomic = False
      operations = [
          migrations.RunPython(populate_external_id, migrations.RunPython.noop),   # bulk_update(..., batch_size=5000)
          migrations.RunSQL("ALTER TABLE ... ALTER COLUMN external_id SET NOT NULL",
                            reverse_sql=migrations.RunSQL.noop),
      ]
  ```
- **Note:** true `.iterator()`/keyset streaming is rarely used *inside migrations* — large
  iterative reshaping is better pushed to background (Celery) tasks or a data-patch system.
  For an in-migration backfill of a very large table, prefer batched `bulk_update`, or a
  keyset loop (see a companion backend-engineering reference), and keep each batch its own
  transaction.

## RunSQL

- Reversible by convention (`NoIrreversibleSQL`): provide `reverse_sql` or `RunSQL.noop`.
- Overwhelmingly used for **DDL** (NOT VALID constraints, concurrent index SQL,
  add-column-with-default), wrapped in `SeparateDatabaseAndState` (or the `state_operations=`
  arg) so Django's migration state stays in sync with the hand-written SQL.
- **Example** — `NNNN_record_url_no_backslash.py`:
  `SeparateDatabaseAndState(database_operations=[RunSQL("ADD CONSTRAINT ... CHECK (...) NOT
  VALID;", "... DROP CONSTRAINT ...")], state_operations=[AddConstraint(CheckConstraint(...))])`,
  with `lock_timeout=15s/statement_timeout=20s`.

## DB routing in migrations

Default: migrations target the primary; RunPython functions don't pass `using=`. When
explicit routing is needed, use the migration's own connection:
`Model.objects.using(schema_editor.connection.alias)...` or
`with schema_editor.connection.cursor() as cursor:`.

## `atomic = False`

Almost always paired with a `CONCURRENTLY` index op (can't run in a transaction). Set
`lock_timeout=None, statement_timeout=None` **explicitly** (the base class requires the
attributes to exist). `NoNonAtomicWithMultipleOperations` forbids >1 op — **one op per
non-atomic file** (a failure mid-migration otherwise leaves a half-applied migration needing
manual repair). Legitimate exceptions carry a `# migrationlinter: ignore ...` note.

## Migration ordering

- **Avoid `run_before`** — express ordering purely via `dependencies` + the **reorderer**.
- **The reorderer** re-sorts migrations by number + modification time, **renames files, and
  rewrites `dependencies`** to linearize conflicts. When two branches both add migration
  `0123_*`, run the reorderer instead of hand-editing dependencies or creating a merge node.
- **Merge migrations should be effectively absent** — the reorderer replaces them.
- **Cross-app dependencies** are declared where needed (e.g. an app migration depending on
  `django_celery_beat`).
- The linter's `ignore_before` per-app anchor lets linting apply only from a known-good
  baseline forward; `recent_n` keeps lint speed constant as history grows.

## Backward compatibility & rolling deployments (expand/contract)

Migrations apply **before** code, and during a rolling deploy old and new code run at once —
so each migration must be safe against the currently-deployed code. Safety is encoded
**structurally in migration ordering**, split into individually-safe steps across releases:

- **Two-phase constraint:** `ADD CONSTRAINT ... NOT VALID` (release/migration 1, cheap
  metadata lock) then `VALIDATE CONSTRAINT` (a later migration, `SHARE UPDATE EXCLUSIVE`,
  no write block). E.g. a NOT-VALID migration followed by a separate `..._validate.py`; the
  three-phase non-nullable series (constraint → validate → SET NOT NULL).
- **Multi-phase column lifecycle:** add nullable column → populate (RunPython) → enforce NOT
  NULL / drop db default → (later release) drop the old column — each step its own migration
  across releases.
- **Expand/contract index swap:** create the new index concurrently in one migration, drop
  the old index in a separate later one.
- **Removal is two releases** (stop using in code → drop later); **rename is three**
  (add-new+dual-write → stop using old → drop old). Never `RemoveField`/`RenameField` in the
  same release the code changes (`NoRemoveField`/`NoDeleteModel`/`NoRename`).

The general rule: **expand** (add the new, backward-compatible thing) → **migrate** (backfill
+ switch the code to use it) → **contract** (remove the old thing), with each phase a
separate release so old code + new schema and new code + old schema both work.

## Feature-flag / settings-gated migrations — don't

Referencing `settings.<X>` in a migration is **blocked by the linter** (`RestrictSettings`):
"the setting can be changed after the migration was created, changing how it executes
later." Only narrow build-time brand/identity constants are whitelisted
(`APP_PORTAL_NAME`, `AUTH_USER_MODEL`, `APP_DEFAULT_TENANT_ID`) — and `APP_PORTAL_NAME`
is a fixed per-deployment identity, not a toggle. There should be **no**
`getattr(settings, "FEATURE_...")`-conditional migrations. **Feature-flag the code path, not
the schema:** ship the schema unconditionally (backward compatible), and gate the *behavior*
in application code behind the flag. Runtime-conditional data changes go through the
data-patch system or application code, never a migration.

## Deployment sequencing summary

1. Migration runs (before code). It must be safe for the **old** code still serving traffic.
2. New code rolls out gradually; for a window, old + new code both hit the new schema.
3. Only in a **later** release do you contract (enforce NOT NULL, drop old columns/indexes),
   once every instance runs the new code.

Every migration carries an SQL/Reverse-SQL docstring header (so reviewers see the exact SQL);
schema migrations subclass `TimeoutAwareMigration` with explicit timeouts (or `None` for
CONCURRENTLY/state-only); reverses are always provided (`.noop` when trivial); one op per
non-atomic file; ordering via `dependencies` + reorderer (no `run_before`, no merges); no
runtime settings in migrations.
