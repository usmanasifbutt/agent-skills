# Backend Testing (pytest + Django + DRF)

## Framework & layout

- **pytest is the sole runner** (`pytest` + `pytest-django`), never `manage.py test`. Each
  independently-runnable component ships its **own `pytest.ini`** (or `[tool.pytest.ini_options]`)
  declaring `DJANGO_SETTINGS_MODULE` (a dedicated **test** settings module, env-independent),
  `testpaths`, and `filterwarnings`. **Run tests from the directory that owns that `pytest.ini`** —
  `cd` into the component first, or the wrong settings load and collection silently misses tests.
- **Tests live in a `tests/` package** per component, mirrored by domain and tier
  (`tests/unit/<domain>/`, `tests/integration/<domain>/`). Shared data-building code sits in sibling
  packages (`tests/data_factory/`, `tests/data_seed/`, `tests/fixtures.py`/`conftest.py`). Snapshots
  in `__snapshots__/`.
- **DB reuse:** use `--reuse-db` locally for speed; after a schema change or a DB error, run once
  with `--create-db`, then go back to `--reuse-db`.
- **`django_db` default-on:** a `conftest.py` hook applies `@pytest.mark.django_db(databases="__all__")`
  to every collected test, so you **don't** add the marker by hand. (A pure-library may keep it
  off and grant DB access per test to keep pure-Python tests fast.)

## Test style

- **Module-level test functions, never a `TestClass`.** Flat functions read top-to-bottom, compose
  with fixtures, and avoid shared-state/inheritance pitfalls.
- **Naming `test_<file_under_test>_<case>`** — the prefix derives from the test file
  (`test_record_stats.py` → `test_record_stats_...`), so `-k` selection and failure output are
  self-locating.
- **No inline comments in test bodies.** If a test isn't self-explanatory, refactor (extract a
  helper, rename). A one-line docstring stating expected behavior is fine.
- **`parametrize` conventions:** the first argument is **always a tuple** of names (never a
  space/comma string); for >3 cases wrap each in `pytest.param(..., id="short-id")`; stack multiple
  `@parametrize` for a cartesian product.
- **Enums/constants, not magic values** — `RecordType.LISTING`, not `1`; `RecordState.ACTIVE`, not
  `"active"`.

```python
@pytest.mark.parametrize(
    ("segment", "expected_status"),
    [pytest.param(1, 200, id="valid"), pytest.param(999, 404, id="unknown")],
)
def test_record_stats_returns_expected_status(api_client, segment, expected_status):
    response = api_client.get(_make_url(segment=segment))
    assert response.status_code == expected_status
```

## Unit vs integration vs API

- **Unit** (the bulk) = *one behavior / one code path*, **not** "no I/O" — they create real rows and
  assert on real query results.
- **Integration** lives under its own path with its **own `pytest-integration.ini`** and runs as a
  separate CI job — cross-boundary flows (sync from an upstream source, index reconciliation,
  multi-step pipelines) end-to-end against a real DB.
- **API tests** drive the HTTP layer through a test client (below).
- **Common mistake:** a slow multi-system flow parked in `tests/unit/` dragging the fast suite; or
  heavy integration setup to test one pure function.

## Test data — factories, seed collections, fixtures

- **Seed collections (preferred for ORM data):** a `DataSeedFactory` exposes one collection per
  model; tests declare data by `ref` and read instances back:
  ```python
  def test_record_stats_structure(api_client, data_seed):
      data_seed.set_defaults(organizations={1: dict(segment=3, name={"en": "Example Org"})})
      data_seed.setup()
      org = data_seed.organizations.get_one(1)
  ```
  Collections fill sensible defaults and cross-reference each other; extend them (one per model).
- **Data factories (the alternative):** `DataFactory` subclasses build data **through public
  interfaces only** (`as_dict()`, `create()`, `create_multiple(count, custom_fields=...)`) with an
  auto-incrementing sequence for unique external ids — so tests don't depend on implementation
  details.
- **`conftest.py` fixtures:** authenticated clients, role users, frozen clocks, shared mocks.
- **The cardinal rule:** create data with **real models; NEVER mock the DB/ORM**. Factories/
  collections ultimately call real create/upsert paths.

## Mocking — boundary only

Mock at the system boundary; the DB/models/ORM are always real.
- **External HTTP / upstream APIs** — a request-mock lib or a hand-written mock client swapped via
  `override_settings`/`patch`; fake the cloud SDK (e.g. `moto`).
- **Async task dispatch** — patch the task's `.delay`/`.apply_async` and assert the call, don't run
  the worker:
  ```python
  mocked_deliver.delay.assert_called_once_with(record_id=record.id)
  ```
- **Search index** — an in-memory fake index class so indexing is exercised without a live backend.
- **Time/clock** — freeze it (a freezegun-style fixture), always **UTC**.
- **Redis/cache** — a fake in-memory implementation; autouse fixtures clear `lru_cache`s/registries
  (and the class/config factory `cache_clear()`) between tests.
- **Shared mocks live in fixtures** (`conftest.py`), not re-`patch`ed inline in ten tests.
- **Isolation guardrail:** an autouse fixture that **fails the test if a global setting or the active
  language was changed and not restored** — prevents cross-test bleed.

## DRF / API testing

- Go through a real test client and assert **status + body shape**. DRF services use
  `rest_framework.test.APIClient` (often via a small builder attaching auth —
  `.with_bearer_token(...)`, `.as_user(user)`); plain-Django/Ninja services use `django.test.Client`
  with role-specific client fixtures (`anonymous_client`, `permissionless_client`, `moderator_client`).
  Target `reverse("namespace:route")`, **never hard-coded URLs**.
- **Permissions/auth are first-class cases** — parametrize over client fixtures (a lazy-fixture helper
  lets fixtures be parametrize values) and assert `403`/`404` with the exact error body.
  **Query-param validation** is tested (missing/invalid → `400` + body); localization via an
  `Accept-Language` header.
```python
response = api_client.get(_make_url(segment=3))
assert response.status_code == 200
assert response.json()[0]["organizationExternalID"] == org.external_id
```
- **Common mistake:** asserting only status (or only body); happy-path only; hard-coded URLs.

## Snapshot testing

- Assert large structured payloads (serializer output, full API responses) against a `snapshot`
  fixture (`assert value == snapshot` / `snapshot.assert_match(...)`) instead of dozens of brittle
  field asserts. Use **path/type matchers** to ignore volatile fields (auto-increment ids,
  timestamps) so snapshots don't churn.
- Update deliberately with `--snapshot-update`; **never** reflexively (that commits a regression as
  the baseline).

## Assertions

Plain `assert` with pytest introspection (no `unittest` `self.assertEqual`). Assert on **observable
outcomes** — status + parsed body, resulting rows via real queries, mock call assertions
(`assert_called_once_with`, `call_count`, `assert_not_called`), and **query counts** via a
`CaptureQueriesContext` to guard against N+1. Prefer one snapshot over dozens of field asserts.
Avoid asserting truthiness (`assert response.data`) instead of the concrete expected value.

## Coverage

- Target **~100% and all edge cases**, tempered by **"don't overtest"** (less than 100% is fine with
  a good reason). Exclude `*/tests/*`, `*/migrations/*`, generated code; `exclude_lines` for
  `pragma: no cover`, dunders, abstract methods.
- **Per-package `--cov-fail-under`** thresholds (a shipped client library held higher than a thin
  server) rather than one global number. In CI, per-shard `.coverage` files are `coverage combine`d
  and posted to the PR.
- Command: `pytest --cov=<pkg> --cov-report=term-missing:skip-covered`.
- **Common mistake:** chasing 100% on trivial/generated code instead of covering real edge cases.

## Performance testing

- **`pytest-benchmark`**, segregated by marker: the normal config runs `-m "not benchmark"`; a
  separate config runs `-m "benchmark"` (and app suites default to `--benchmark-disable`). Benchmarks
  compare a naive vs optimized implementation of the same operation.
  ```python
  @pytest.mark.benchmark()
  def test_upsert_native(benchmark):
      benchmark(_native_upsert, model, value)
  ```
- **Common mistake:** letting benchmarks run in the regular suite (nondeterministic timings, slow CI).

## CI integration

- **Shard within a suite** with `pytest-xdist` (`-n N`, tuned by suite size); integration tests run
  as a **separate job** with their own config. Matrices across Python/Django/driver versions (tox
  `envlist` or a GH-Actions matrix with a Postgres service container).
- Tests run on `pull_request` with `concurrency` + `cancel-in-progress`; a separate lint/type-check
  job (`verify`) runs alongside; `--cov-fail-under` + a coverage bot gate/report; migration
  completeness (`makemigrations --check`) and task-route checks are CI-gated.
- **Scope to your change** — the full suite is too slow to run locally; a `[skip tests]` PR-title
  escape hatch exists for backports.
- **Common mistake:** relying on running everything locally; forgetting integration tests need their
  own config/job and silently not running them.
