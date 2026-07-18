# Models & Migrations

## Base classes & libraries

- Every model inherits a **Postgres-aware base model** (from a PostgreSQL-extensions library that adds native `ON CONFLICT` upserts, explicit locks, and conditional/partial unique indexes), and every DB alias uses that library's PostgreSQL backend. This unlocks native `ON CONFLICT` upserts, explicit locks, conditional/partial unique indexes.
- Multi-language content uses a **localized-field type** (translations stored in one PostgreSQL `hstore`/JSONB column as `{"en": ..., "ar": ...}`, not one column per language).
- Migrations use `TimeoutAwareMigration` (from an in-house migration-safety toolkit) — never `migrations.Migration`.

## Model rules

1. **`Meta` class first** — place it at the top of the model class.
2. **`TextField`, never `CharField`** — and **no `max_length`** or length constraints. (Length limits belong in validation, not the schema.)
3. **Optional text fields:** `null=True, blank=True`.
4. **Constraints & indexes in `Meta`** — `Meta.constraints` (`UniqueConstraint`, `ConditionalUniqueIndex`) and `Meta.indexes` (`models.Index`).
5. **No field-level flags** — no `unique=True`, no `db_index=True` on individual fields; no `unique_together`. Use `Meta` instead. (Makes all constraints/indexes visible in one place and controllable by the migration linter.)
6. **Singular model names** (`CategoryPriceStat`, not `...Stats`) — linter-enforced.
7. **`DecimalField` over `FloatField`** for numeric precision.
8. Common fields: `created_at = DateTimeField(auto_now_add=True)`, `updated_at = DateTimeField(auto_now=True)`.
9. **`external_id`** is the business key for synced models (the ID from the external system); `pk`/`id` is the internal DB key. Look up by `external_id` for business logic, by `pk` for internal joins.
10. **Model-level validation** lives in `clean()` and is called explicitly from `save()` (Django does not call `full_clean()` automatically). `clean()` both normalizes (`self.comment = self.comment.strip() or None`) and validates per-row invariants.

```python
from django.db import models
from app_db.models import PostgresAwareModel  # Postgres-aware base model


class Article(PostgresAwareModel):
    class Meta:
        indexes = [models.Index(fields=["slug"])]
        constraints = [
            models.UniqueConstraint(fields=["slug"], name="article_slug_unique"),
            models.UniqueConstraint(fields=["title", "author"], name="article_title_author_unique"),
        ]

    title = models.TextField()
    subtitle = models.TextField(null=True, blank=True)
    slug = models.TextField()
    author = models.ForeignKey("Author", on_delete=models.CASCADE)
```

**Specialized field types:** a localized-field type for multi-language; PostGIS `gis_models.PointField(null=True, blank=True)` for coordinates (stored `Point(lng, lat)`); a nested-set/tree library's tree base model / `TreeForeignKey` for hierarchies (regions, categories); state fields use `StrEnum.choices()` from a shared enum module. Pre-rendered cache models (`RenderedRecord`) store pre-serialized JSON per language to skip serialization at request time.

## Migrations — safety is the whole point

### `TimeoutAwareMigration` is mandatory
All migrations use `TimeoutAwareMigration` instead of `migrations.Migration`. The custom `makemigrations` generates it; the developer **must** set `lock_timeout` and `statement_timeout` (both attributes must exist, even if `None`, or `apply()` raises `TimeoutNotProvidedError`). Timeouts are wrapped around the operation only when it actually emits SQL.

- **Set real timeouts** (e.g. `lock_timeout=15s`, `statement_timeout=20s`) for anything touching an **existing** table: `AddField`, adding a FK to an existing table, changing/removing a field.
- **Set both to `None`** for state-only operations: `AlterModelOptions`, an `AlterField` that only changes `on_delete`.

**Why:** a schema change that grabs a long lock on a hot table stalls production. Bounded timeouts make the migration fail fast rather than block traffic. (On non-atomic migrations the tool also resets timeouts because pgbouncer connections otherwise stay corrupted with leftover timeouts.)

```python
from datetime import timedelta
from django.db import migrations, models
from app_migrations.migration import TimeoutAwareMigration


class Migration(TimeoutAwareMigration):
    lock_timeout = timedelta(seconds=15)
    statement_timeout = timedelta(seconds=20)
    dependencies = [...]
    operations = [
        migrations.AddField(
            model_name="article", name="subtitle",
            field=models.TextField(null=True, blank=True),
        ),
    ]
```

### Indexes must be concurrent and isolated
New indexes go in their **own non-atomic migration** using concurrent operations. The linter auto-converts `AddIndex`/`RemoveIndex` → `AddIndexConcurrentlyWithRetry`/`RemoveIndexConcurrentlyWithRetry`, sets timeouts to `None`, and **forces one operation per migration** (splitting if needed). The "WithRetry" variants run `DROP INDEX CONCURRENTLY IF EXISTS` first to recover from a prior failed `CREATE INDEX CONCURRENTLY` that left an invalid index.

```python
class Migration(TimeoutAwareMigration):
    atomic = False
    lock_timeout = None
    statement_timeout = None
    operations = [
        AddIndexConcurrentlyWithRetry(model_name="renderedcategorypage", index=ConditionalUniqueIndex(...)),
    ]
```

### Workflow
```bash
cd backend/<app> && uv run python manage.py makemigrations
cd backend && poe lint_migrations_fix        # lints + inserts forward/reverse SQL as a docstring for reviewers
cd backend/<app> && uv run python manage.py migrate
```
`poe lint_migrations_fix` inserts the SQL that will run as a comment at the top of the file so reviewers see the exact effect. The linter ruleset enforces: use timeouts, don't put a concurrent index inside a timeout, prefer `UniqueConstraint ... CONCURRENTLY`, no redundant timeouts.

### Don't import app code into migrations
Migrations are long-lived and immutable. If a function/constant imported in a migration changes later, the migration breaks. Hard-code constants and small utilities in the migration instead. Name new migration files generically (`NNNN_add_<thing>.py`).

## Bad alternatives this avoids
- `CharField(max_length=...)` → arbitrary schema-level length limits that are painful to change and belong in validation.
- Field-level `unique=True`/`db_index=True`/`unique_together` → scatters constraint definitions and makes concurrent-index migration rewriting impossible.
- A plain `migrations.Migration` altering a large table → uncontrolled lock duration, production stalls.
- `AddIndex` inside an atomic migration → `CREATE INDEX` takes an exclusive lock for the whole build → downtime.
