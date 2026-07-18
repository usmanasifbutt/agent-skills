# ORM Queries, PostgreSQL, Transactions, Concurrency, Idempotency

## Query rules

1. **Use the ORM; query as little as possible.** ORM queries are type-checked Python; raw SQL is an unchecked string — fall back to it rarely.
2. **Select only the fields you need:** `.values_list("id", "title")` (tuples) / `.values(...)` (dicts) / `.values_list("id", flat=True)` (flat) / `.only("id", "title")` (instances with limited fields). Prefer `values_list` unless you need many fields (`.values` stores column names per row).
3. **Kill N+1 aggressively:** `.select_related(...)` for FK/one-to-one (single JOIN), `.prefetch_related(...)` for reverse-FK/M2M (separate query). Never run a query per item in a loop or per object in a serializer.
4. **Query only the languages you need** for localized fields — use the localized-field query expressions, either for the current language or for a specific language (`localized_ref("field", "ar")`-style), rather than pulling the whole translations column.
5. **`update_fields=` on partial `.save()`** — updates just that column; cheaper writes, smaller race window.
6. **Set FKs by id, not object:** `Attachment.objects.create(record_id=123, ...)` — avoids a query to fetch the related object.
7. **Prefer `.filter().update(...)`** over fetch-then-`.save()` when you don't already hold the instance.
8. **`.exists()` not `.count()`** for existence checks; **bulk ops not loops** (`bulk_create`, `bulk_update(objs, [fields])`, `bulk_upsert`).
9. **Pass objects, not IDs, between functions** to avoid re-querying; reuse data already loaded (e.g. via `select_related`).

## Upserts — native, atomic (never `get_or_create`)

`get_or_create`/`update_or_create` are **racy** and don't use PostgreSQL's native mechanisms. Use the PostgreSQL-extensions library's `on_conflict`:

```python
from app_db.query import ConflictAction

# single upsert
obj = MyModel.objects.on_conflict(["external_id"], ConflictAction.UPDATE).insert_and_get(
    external_id="123", title="...")

# insert, do nothing on conflict
MyModel.objects.on_conflict(["external_id"], ConflictAction.NOTHING).insert(external_id="123")

# bulk upsert
MyModel.objects.on_conflict(["name"], ConflictAction.UPDATE).bulk_insert([
    {"name": "alice", "email": "a@example.com"}, {"name": "bob", "email": "b@example.com"}])
```

- **Why:** single-statement, atomic, concurrency-safe; the conflict target must match a (partial) unique constraint. This is the DB-level idempotency mechanism (`RecordStateDetails.objects.on_conflict(["record_id"], ConflictAction.UPDATE).insert(...)`).
- **Where:** the `upserting/` package, and a `BaseRepository.insert` in the layered services (`on_conflict(["datetime","trace_id"], ConflictAction.NOTHING)`).

## Batching large querysets

Prefer **keyset pagination**: order by a unique non-null column, repeatedly fetch `column > last_seen` slices (each is an index range scan, no held connection). The pagination key must be a selected column.

```python
records_qs = Record.objects.filter(state=RecordState.ACTIVE).order_by("id").values_list("id", flat=True)
last_seen_id = 0
while batch := list(records_qs.filter(id__gt=last_seen_id)[:2000]):
    last_seen_id = batch[-1]
    ...
```

**Avoid OFFSET/LIMIT batching on large tables** — it re-scans the consumed prefix each batch (quadratic). When there's no unique non-null column, use `iterator()` + a batch generator — and **always use `.using(settings.DATABASE_DIRECT)` with `iterator()`** (server-side cursors need a real, non-transaction-pooled connection; pgbouncer's pooled `default` breaks them). Also batch `.delete()` — Django loads all rows to delete.

## Transactions

- **`transaction.atomic`**: wrap multi-statement writes that must commit all-or-nothing (parent + FK children + side effects in the *same* block). An `IntegrityError` mid-block rolls everything back.
- **`ATOMIC_REQUESTS = True`** on all aliases **except `direct`** (which is `False`). Reason: atomic-requests prepares a connection even when unused, bloating connection count; the `direct` alias serves long/bulk background work that must not be request-wrapped.
- **`durable=True`** (`with transaction.atomic(durable=durable):`) asserts a block is the outermost transaction so row locks release on commit — important for Celery tasks holding locks.
- **Serializable isolation** is opt-in via a dedicated `serializable` DB alias, not ad-hoc `SET`.

## Concurrency

- **`select_for_update()`** inside `transaction.atomic()` for read-modify-write on a row a concurrent worker could touch. Competing-consumers queue pattern: `.select_for_update(of=("self",), skip_locked=True)[:1]` — `of` locks only this table's rows; `skip_locked` lets workers grab *different* rows instead of blocking. Bad alternative: `get()` then `save()` → lost updates.
- **`on_conflict` upserts** (above) for concurrency-safe create-or-update.
- **Table locks** via `atomic_exclusive(model, lock_mode=LockMode.Exclusive)` — `LOCK TABLE ... IN <mode> MODE` inside a transaction, for tree/category rebuilds.
- **PostgreSQL advisory locks** (a `tools/lock.py` helper over a PG-locks library) for cross-process mutual exclusion. Rules: acquire on **`settings.DATABASE_DIRECT`** (a pgbouncer transaction-pooled connection can't reliably hold a session lock); use `wait=False` (non-blocking try-lock — on failure raise e.g. `RecordAlreadyLockedError` and let the task retry, don't block a worker); named per-domain helpers (`lock_record`, `lock_alert`). `skip_if_running=True` on Celery tasks is built on this (advisory lock keyed on the function name). Never "check then acquire" — just try to acquire (TOCTOU).

## Read/write DB routing

Alias taxonomy: `default` (pooled read/write via pgbouncer, `ATOMIC_REQUESTS=True`), `direct` (non-pooled, `ATOMIC_REQUESTS=False`, server-side cursors allowed), `readonly` (read replica), `bi_read_replica` (analytics), `serializable`. Every alias tags `OPTIONS.application_name` so connections are identifiable in `pg_stat_activity`; `DISABLE_SERVER_SIDE_CURSORS=True` on pooled aliases (pgbouncer).

**`DatabaseRouterMiddleware`** (must be the **last** middleware): writes (POST/PUT/PATCH/DELETE), the admin, and read-your-write-sensitive paths always hit the main DB; GETs hit the `readonly` replica; on a 5xx it retries once on main and, if results differ, flips a timed fallback (`RETRY_INTERVAL=3*60s`) while probing the replica with `select 1` before recovering. Health checks probe main via `DATABASE_DIRECT` and replica via `DATABASE_READONLY`.

## Validation is layered

- **Serializer / query-param layer** = shape and format of the request (DRF serializer `validate()`, pydantic/ninja schema). See `caching-and-search.md` and `api-and-views.md`.
- **Model `clean()`** = per-row invariants + normalization at persistence, called from `save()`.
- **Service / scrubber layer** = cross-entity business rules, external-data cleaning, concurrency-safe writes; business-rule violations raise domain exceptions (`AppAPIException`, `AlertExists`). Request-payload validation is a `DataScrubber`, never inline in the view (see `serializers-and-scrubbers.md`).
