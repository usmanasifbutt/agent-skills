---
name: django-backend-engineering
description: Engineering standards for building Django/DRF backend services — multi-brand factory architecture, an ObjectSerializer + DataScrubber pipeline, Keycloak/Principal auth, an in-house Celery task wrapper, PostgreSQL-native upserts, structlog, and safe migrations. Load this before writing or reviewing any backend feature (views, models, serializers, tasks, queries, auth, caching, search) in a Django service. Also load when scaffolding a NEW Django service so it inherits these conventions.
---

# Django Backend Engineering

This skill captures a set of **engineering standards** applied across a family of
Django backends. It is not a description of any one repo — it is the set of
rules, patterns, and judgment calls that recur across all of them, with the
reasoning behind each so you can apply them to new code and new services.

## The two house styles

There are two distinct but compatible architectures. Identify which one the
service uses before writing code:

| Style | Where | Signature |
|-------|-------|-----------|
| **Multi-brand factory** | the multi-brand platform (and the shared core it builds on) | One codebase serves many portals. Classes resolved through `factory()`, overridden per portal. Pipeline: `Scrubber → Upsert → Publish (serialize → index + cache)`. Custom `ObjectSerializer`, **not** DRF serializers. |
| **Layered / repository** | the analytics/ingestion, recommendation, traffic-analysis and content-moderation services | Classic layered app: `views → services → repository (raw SQL templates) → models`. Explicit `data/` DTOs, `repository/*.py`, read-replica routing, DRF or plain Django views. |

Both styles share the same **foundations**: a Postgres-aware base model, TextField-only
models, `Meta`-level constraints/indexes, PostgreSQL-native upserts, `structlog`,
an in-house Celery task wrapper, `TimeoutAwareMigration`, `uv` env management, and the
naming/exception conventions below.

When in doubt, **match the file you are editing**, then match the repo, then fall
back to this skill.

## Topic map — read the reference for the area you touch

Each reference file explains every practice with **why it exists / where it's used
/ when to use it / tradeoffs / the bad alternative / a production example**.

| You are working on… | Read |
|---------------------|------|
| Project layout, apps, the factory/DI pattern, multi-portal scope, feature flags, settings layering, DTOs, service vs repository layers | `references/architecture.md` |
| API endpoints, DRF views, request lifecycle, responses, pagination, filtering, validation flow | `references/api-and-views.md` |
| ObjectSerializer, DataScrubber, the scrub→upsert→publish pipeline, N+1 avoidance on context | `references/serializers-and-scrubbers.md` |
| Models, fields, the Postgres-aware base model, localized/geo fields, migrations, timeouts, concurrent indexes | `references/models-and-migrations.md` |
| ORM queries, upserts/`ON CONFLICT`, bulk ops, batching, transactions, concurrency, read replicas | `references/queries-database.md` |
| Authentication, authorization, permission classes, `Principal`, Keycloak, API keys, service-to-service | `references/auth-and-keycloak.md` |
| Celery, the in-house task wrapper, retries, idempotency, routing, RabbitMQ, periodic tasks, background jobs | `references/celery-and-jobs.md` |
| Caching, Redis, Elasticsearch/hosted search, search indexing, the publishing fan-out | `references/caching-and-search.md` |
| Logging, structlog, observability, APM/error tracking, exception classes and handlers | `references/logging-and-observability.md` |
| Naming, code style, type hints, PEP-8 deltas, commit messages, testing | `references/code-style-and-testing.md` |

## Non-negotiable foundations (the short list)

These hold in **every** service. Everything else is in the references.

1. **Query the ORM, query little.** Select only the fields you need
   (`.values_list`/`.only()`), aggressively `select_related`/`prefetch_related`,
   never run a query per item in a loop or serializer. Fall back to raw SQL rarely.
2. **Upserts are native and atomic.** Use the PostgreSQL-extensions library's `on_conflict(...)` /
   `bulk_upsert`, never `get_or_create` / `update_or_create` (racy).
3. **Models: Postgres-aware base model, `TextField` only, constraints/indexes in `Meta`.**
   No `max_length`, no field-level `unique=True`/`db_index=True`, no `unique_together`.
4. **Migrations touching existing tables set `lock_timeout` + `statement_timeout`**
   via `TimeoutAwareMigration`; indexes go in their own non-atomic concurrent migration.
5. **Views orchestrate; they don't validate or format.** Validation lives in
   scrubbers/serializers, output formatting in serializers. Views are thin.
6. **Auth is private by default.** Protect endpoints with permission classes;
   identify the caller via `request.principal`; verify a passed user ID matches the token.
7. **Tasks fail loudly and retry.** Use the in-house `shared_task` with
   explicit dot-name, `bind=True`, `set_logger_context`, `retry_on_exceptions`; no
   pokemon `try/except` swallowing errors.
8. **Log with `structlog` only** — no `print`, no stdlib `logging`. Reuse existing
   keys; the number of distinct keys is limited downstream (the log-aggregation platform).
9. **Settings are explicit.** Define every setting in `common.py`; never
   `getattr(settings, ...)`. Prefix platform settings `APP_`, integrations by vendor.
10. **Don't catch `Exception`.** Catch specific errors (or `RuntimeError` for a
    deliberate catch-all); name new exception classes `...Error`, inherit `RuntimeError`.

---

## Best Practices (cross-cutting)

- **Decide portal scope first.** Before editing the shared core, ask "does this apply to
  all portals or some?" Shared code affects many sites. Roll out with a feature flag
  or a factory override, not a `if portal_name == "brand_a":` branch.
- **Resolve overridable classes through `factory()`**, never by direct import — a
  direct import silently bypasses the portal's override.
- **Thin views, rich pipeline.** `scrub → business logic → serialize`. Push
  validation into `DataScrubber`, output into `ObjectSerializer`/DRF serializer.
- **Batch database work on the collection**, not per object. Build one context /
  one bulk query and index results by id. This is the single most common perf bug.
- **Prefer passing objects over IDs** between functions to avoid re-querying; use
  `.exists()` not `.count()`; use `update(...)`/`bulk_update`/`bulk_create` over loops.
- **`update_fields=` on every partial `.save()`.** Cheaper writes, fewer race windows.
- **Cache read endpoints** in `urls.py` (not in the view) so the whole cache policy
  is visible in one place; balance TTL against cache-key cardinality from query params.
- **Backwards compatibility is a hard constraint.** A separate mobile team consumes
  these payloads — additive changes only on existing serializers/endpoints; confirm
  breaking changes with the owning dev.
- **Type-hint everything**; use `@dataclass`/`StrEnum` for typed, memory-efficient
  data containers instead of loose dicts.
- **Localization-safe code.** Never hard-code language codes; iterate `settings.LANGUAGES`,
  read defaults from `settings.LANGUAGE_CODE`, use the localized-field query expressions.

## Anti-Patterns (reject in review)

- ❌ `from core.serializers... import XSerializer` for a portal-overridable
  class → bypasses the factory. Use `factory().serializers.x`.
- ❌ `if portal_name == "brand_a":` string checks → use `Portal.current()` / feature flags.
- ❌ Validation logic inside a view; manual response-dict construction → scrubbers + serializers.
- ❌ `get_or_create` / `update_or_create` → racy; use native `on_conflict` upserts.
- ❌ `Model.objects.get(...)` in a loop, `cached_property` doing a query on a
  per-object serializer → N+1; batch on the context.
- ❌ `except Exception:` / bare `except:` / try-except swallowing in Celery tasks →
  hides failures from the error tracker and defeats retries.
- ❌ `getattr(settings, "X", default)` → silent config drift; define it in `common.py`.
- ❌ `CharField` + `max_length`, field-level `unique=True`/`db_index=True`,
  `unique_together` → use `TextField` + `Meta.constraints`/`Meta.indexes`.
- ❌ A migration that alters an existing table with no `lock_timeout`/`statement_timeout`,
  or an `AddIndex` in an atomic migration → long locks / downtime.
- ❌ `print()` or stdlib `logging`; inventing new structlog keys casually.
- ❌ New settings for constants that never change per env → declare a module-level constant.
- ❌ New top-level URL folders (`/myendpoint`) → nest under `/api/`; use camelCase paths.

## Checklist — before merging a PR

- [ ] **Portal scope evaluated.** If in the shared core, verified impact across
      multiple portals/brands (or gated behind a flag/factory override).
- [ ] Overridable classes resolved via `factory()`, not imported directly.
- [ ] View is thin: validation in scrubber/serializer, output via serializer, no manual dicts.
- [ ] Permissions set explicitly; default private; passed user IDs verified against the token.
- [ ] No N+1: relations `select_related`/`prefetch_related`; per-collection work batched on context.
- [ ] Queries select only needed fields; `.exists()` over `.count()`; bulk ops over loops.
- [ ] Upserts use `on_conflict`; no `get_or_create`/`update_or_create`.
- [ ] Writes use `update_fields=`; multi-write paths wrapped in `transaction.atomic`.
- [ ] Models: Postgres-aware base model, `TextField`, `Meta` first, constraints/indexes in `Meta`.
- [ ] Migrations: `TimeoutAwareMigration` with timeouts set (or `None` for state-only);
      indexes in their own non-atomic concurrent migration; `poe lint_migrations_fix` run.
- [ ] Celery tasks: explicit dot-name, `bind=True`, `set_logger_context`, retries configured,
      no error-swallowing; idempotent / `skip_if_running` where concurrency matters.
- [ ] Logging via `structlog`, reusing existing keys; no `print`/stdlib logging; views don't log routinely.
- [ ] Exceptions specific (`...Error`, inherit `RuntimeError`); no `except Exception`.
- [ ] No `getattr(settings, ...)`; new settings defined in `common.py` and prefixed correctly.
- [ ] Backwards-compatible payloads; breaking API/serializer changes confirmed with the consumer.
- [ ] Tests added (module-level functions, `parametrize` with tuples), run from the correct `pytest.ini` dir.
- [ ] `poe fix` / `poe verify` (Black, Ruff, mypy) clean.

## Checklist — before releasing to production

- [ ] Migrations reviewed for lock risk on large/hot tables; timeouts sane; index
      builds concurrent; migration ordering independent of unmerged code.
- [ ] New settings listed in the commit body and configured for **every** target
      portal/environment (not just dev).
- [ ] Feature gated behind a flag defaulting **off**; rollout plan is per-portal.
- [ ] Caches: TTLs chosen; invalidation/republish path exists for changed serializer output.
- [ ] Search index changes have a reindex/republish plan; backfill task identified.
- [ ] Celery: queue routing correct; worker memory budget respected (~300mb @ 1x concurrency);
      retry/backoff bounded; periodic task wired in all 4 required places (see reference).
- [ ] Observability: key operations emit structlog events; errors reach the error tracker/APM;
      no secrets in logs or URLs.
- [ ] Backwards compatibility verified against mobile/third-party consumers.
- [ ] Rollback path understood (data patches reversible or forward-only-and-safe).

## Checklist — when creating a new Django service

- [ ] `uv` for env/deps; `pyproject.toml` with Black (line length per repo, 110 default),
      Ruff (`select = ["ALL"]` + explicit ignores), mypy `--strict`, poe tasks (`fix`, `verify`).
- [ ] Base model = the Postgres-aware base model; adopt TextField/Meta-constraints rules.
- [ ] Adopt the in-house migration-safety toolkit's `TimeoutAwareMigration` + custom `makemigrations`; add the migration linter.
- [ ] Settings layered `common.py → *_overrides.py → env`; explicit settings, no `getattr`.
- [ ] `structlog` configured (JSON output); define a `LoggingEvent` key registry if following the layered style.
- [ ] Auth: decide the `Principal`/permission model up front; private-by-default; reuse
      the in-house SSO wrapper / Keycloak integration rather than rolling your own.
- [ ] Celery via the in-house task wrapper; define routing keys and a periodic-task registry.
- [ ] Choose the architecture style (factory-multiportal vs layered-repository) and lay out
      folders accordingly (`views/ serializers/ scrubbers/ tasks/ tools/` or `views/ services/ repository/ data/`).
- [ ] DRF configured with a default auth/permission class, a standard exception handler, and a pagination class.
- [ ] `pytest` + `--reuse-db`, data factories/collections, `CODEOWNERS`, CI quality gate.
- [ ] `CLAUDE.md` / guidelines file documenting the above so the conventions are discoverable.
