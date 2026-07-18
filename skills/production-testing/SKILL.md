---
name: production-testing
description: How to write, organize, and verify tests the disciplined way — backend (pytest + DRF, real-DB unit/integration/API tests, factories/seed-collections/fixtures, boundary-only mocking, snapshot testing, coverage) and frontend (Jest + React Testing Library provider-wrapped renders, restorable config mocks, an E2E/browser suite with layered page objects and DB-backed flakiness quarantine), plus contract and performance testing. Critically, it defines a mandatory VERIFICATION LOOP: when validating a test or an API, actually run it → on failure raise a structured bug report → after the fix, rerun the same command → repeat until green. Load this BEFORE writing or reviewing ANY test (unit, integration, API, component, hook, E2E), before choosing what to mock, and whenever verifying that code actually works.
---

# Production Testing

How this team tests, reverse-engineered from real backend (Python/Django/pytest/DRF) and frontend
(Jest/React Testing Library + an E2E browser suite) codebases. Expressed generically — apply to any
Django + React estate. Use a neutral demo domain (Record, Organization, Member, Category, User).

## Testing philosophy (the through-line)

- **"Unit" means one behavior, not zero I/O.** Backend unit tests freely hit a **real database with
  real models** — the DB is cheap and truthful. Mocking the ORM produces tests that pass while the
  code is broken. Integration tests exercise cross-boundary flows; API tests drive the HTTP layer.
- **Mock only at the boundary** — network, async task dispatch, external SaaS, wall-clock time,
  search index. Never the DB/ORM; never a component/selector you could just render.
- **A test must be able to fail.** Before trusting a green run, prove the test *can* go red
  (temporarily break the assertion/selector). Never let a test that is structurally incapable of
  failing into the suite.
- **Assert observable behavior** — HTTP status + parsed body, resulting DB rows, mock call
  assertions, query counts — not internal implementation detail.
- **Hermetic by default** — no real network/config/clock leaking in; a non-local default timezone
  forces TZ-awareness.
- **Backward compatibility is a contract** — additive changes to serialized output are safe;
  changing/removing an existing field is a breaking change to confirm with downstream consumers.
- **Layered reuse over duplication** — shared render/data helpers; one shared E2E spec runs across
  every brand.
- **Never skip to green.** Fix the root cause; suppression (`//@ts-nocheck`) is tolerated only in
  test files, never in product code.

---

## THE VERIFICATION LOOP (mandatory)

**Writing or changing a test — or being asked whether something works — is not done until you have
run it and seen the result with your own tools.** Do not report success from reading code. Follow
this loop; the full command playbook + bug-report template is in `references/verification-loop.md`.

```
1. IDENTIFY the target and its test type (API / unit / integration / component / hook / E2E).
2. RUN it with the real command (Bash): call the API, or run the exact test path. Capture output.
3. EVALUATE:
     • GREEN → (for a NEW test) prove it can fail: break the assertion, rerun, confirm RED, restore,
       rerun, confirm GREEN. Then done — report the passing command + result.
     • RED → do NOT paper over it, do NOT loosen the assertion. Raise a BUG (structured report:
       what was tested, exact command, expected vs actual, error/traceback excerpt, suspected
       cause, minimal repro). Decide test-bug vs product-bug.
4. FIX the correct side (test or product), or hand the bug off if it's out of scope.
5. RERUN the exact same command (re-enter the loop). Repeat 2–5 until GREEN.
6. Only a GREEN run (that you have proven can go RED) closes the task. If still RED after a
   reasonable fix attempt, STOP and escalate with the bug report — never disable/skip/`.only()`
   your way to a passing suite.
```

**For an API specifically** (the case the request calls out): hit the endpoint (its test via the
test client, or a live call with the right auth), assert **status + body shape + permissions**; if
it fails, report the bug (request, expected, actual, response body/traceback); after the fix, **call
it again**; loop until it returns what the contract says. When driving this from a subagent, re-run
the *same* verification command after each fix rather than trusting the fixer's word.

---

## Reference map

| Working on… | Read |
|-------------|------|
| pytest/DRF: layout, test style, factories/collections/fixtures, mocking, API tests, snapshots, coverage, perf, CI | `references/backend-testing.md` |
| Jest + React Testing Library: config, render helpers, mocking, snapshots, organization, coverage | `references/frontend-testing.md` |
| E2E (browser) page objects & flakiness quarantine, integration tests, contract testing, performance testing | `references/e2e-integration-contract.md` |
| The run → report-bug → rerun loop in detail: per-test-type commands, bug-report template, test-vs-product triage | `references/verification-loop.md` |

---

## Testing checklist (before you commit a test)

- [ ] **You ran it and saw it pass** (the verification loop) — not "looks right".
- [ ] For a new test, **you proved it can fail** (temporarily broke it → RED → restored → GREEN).
- [ ] Backend: module-level **function**, not a `TestClass`; named `test_<file>_<case>`; **no
      inline comments** (refactor for clarity instead).
- [ ] Test data via **real models** (factories / seed collections / fixtures) — the **DB is never
      mocked**; enums/constants used, not magic numbers/strings.
- [ ] `@pytest.mark.parametrize` first arg is a **tuple**; `pytest.param(..., id=...)` for >3 cases.
- [ ] Mocks only at boundaries (HTTP, task `.delay`, external SaaS, clock, search); shared mocks
      live in `conftest.py`/fixtures, restored after use.
- [ ] API test asserts **status + body shape + the unauthorized/forbidden/invalid-param cases**,
      using `reverse(...)` not hard-coded URLs.
- [ ] Frontend: rendered through the shared provider wrapper (store + router + i18n); **async render
      helper** used when the component loads data via the route; queries by **role/text**, not
      test-ids; `user-event` for interaction.
- [ ] `CONFIG`/`process.env` mutations use the restorable helpers with `.restore()` in teardown.
- [ ] Snapshots are **small and paired with real assertions**; not blindly `--snapshot-update`d.
- [ ] E2E: selectors come from the shared selector classes (aria-label based), localized text via
      the label helper, **no raw selectors in shared/base files**; a proper `waitFor…` (never
      `pause()`); new spec folder registered as a CI suite.
- [ ] Test is scoped to your change (don't run/author the whole world); slow flows go in the
      integration/E2E tier, not the fast unit suite.

## PR testing checklist

- [ ] The fast unit suite passes for the areas you touched; **run only what's relevant** (full
      suites are too slow to run locally).
- [ ] New/changed API or serializer output is **backward compatible** (additive) — breaking changes
      confirmed with the downstream consumer.
- [ ] Coverage of the change is meaningful (aim high, **don't overtest** trivial/generated code);
      per-package thresholds still pass where enforced.
- [ ] Integration tests updated/added if a cross-boundary flow changed (they run as a separate job).
- [ ] E2E: if you added a spec folder, it's wired into the brand's CI suite/matrix; flaky specs
      handled via the quarantine system (a **branch override** to test a fix), not `.skip()`.
- [ ] No `.only()`/`.skip()`/commented-out cases/`browser.pause()` left in; no `//@ts-nocheck` in
      product code.
- [ ] CI is green *because the tests pass*, not because a flake was re-blessed or a snapshot was
      reflexively updated. Used **rerun-only-failed** for genuine flakes rather than loosening tests.
- [ ] Migrations/tasks that CI checks for (`makemigrations --check`, task-route checks) are satisfied.

## When to MOCK

- **External HTTP / third-party APIs** — canned responses (a request-mock lib or a hand-written mock
  client); fake the cloud SDK (e.g. `moto`).
- **Async task dispatch** — patch `.delay`/`.apply_async` and assert the call; don't run the worker
  in a unit test.
- **Search index / other network data stores** — an in-memory fake that records writes and serves
  seeded hits.
- **Wall-clock time** — freeze it (in UTC), so time-dependent logic is deterministic.
- **Redis / cache** — a fake in-memory implementation; clear caches/registries between tests.
- **Frontend network** — global `fetch` stubbed; mock the **API/search client module** (via
  `__mocks__`), not `fetch` calls one by one.
- **Config/env** — via restorable helpers so brands/tests don't bleed config.

## When NOT to MOCK

- **The database, models, or ORM** — ever. Create real rows with factories/collections and assert on
  real query results. Mocking the DB is the cardinal sin — it hides real breakage.
- **The code under test** — including components, hooks, reducers, selectors, serializers. Render/run
  the real thing through the provider wrapper.
- **The route/data-loading flow** in a frontend integration test — use the async render helper that
  actually runs the route and awaits its data, rather than hand-filling "loaded" state.
- **Anything you're asserting the behavior of** — if you mock it, your test asserts your mock.
- **Internal pure functions** you could just call directly.

## Common mistakes

- Reporting "tests pass"/"the API works" **without running it** — the verification loop is mandatory.
- **Mocking the ORM/queryset** or executing real Celery/HTTP in a unit test.
- A test that **can't fail** (assertion-free `forEach`, truthiness checks, a snapshot nobody reads).
- `@pytest.mark.parametrize("a, b", ...)` (string first arg) instead of a tuple; hard-coded enum
  ints/strings; grouping tests into a class "to share setup" (use a fixture).
- Asserting **only status** and not body (or vice-versa); hard-coding URLs instead of `reverse`;
  testing only the happy path and skipping unauthorized/forbidden/invalid cases.
- Frontend: using the **sync render helper** for a component that loads data via the route; mocking a
  component/selector you could render; seeding a `__mocks__` DB without `reset()` in teardown;
  reading `CONFIG.build.X` without seeding it.
- Reflexive `--snapshot-update` that commits a regression as the new baseline; giant snapshots full
  of unmasked ids/timestamps that churn every run.
- E2E: raw XPath in a shared base page; `browser.pause()` instead of a proper wait; un-quarantining
  globally to test a fix (use a branch override); resolving a flake before it's fixed in production;
  a new spec folder not registered as a CI suite (silently never runs).
- **Skipping to green** — `.only()`/`.skip()`, env-gated skips, commented-out cases, loosened
  assertions — instead of fixing the root cause.
- Expecting coverage % or Lighthouse to gate the PR — the **test result** is the gate; coverage is
  reported, Lighthouse is a scheduled trend monitor.

## Production examples (generic, drawn from the real suites)

- **API test** — parametrize over client-identity fixtures to cover authorized vs forbidden, assert
  status **and** body:
  ```python
  @pytest.mark.parametrize("client", [lf("member_client"), lf("permissionless_client")])
  def test_records_endpoint_permissions(client, data_seed):
      data_seed.set_defaults(records={1: dict(segment=3)}); data_seed.setup()
      response = client.get(reverse("api:records"))
      assert response.status_code in (200, 403)
      if response.status_code == 200:
          assert response.json()[0]["externalID"] == data_seed.records.get_one(1).external_id
  ```
- **Component integration test** — run the real route + await data, then assert on rendered output:
  ```tsx
  const state = new ReduxStore.Builder().withRouterURL('/search').build();
  const { getByRole, findByText } = await renderWithStateAsync(state, <SearchResults />);
  await userEvent.click(getByRole('button', { name: 'Filters' }));
  expect(await findByText('Price')).toBeInTheDocument();
  ```
- **N+1 guard** — assert the query count, not just the result:
  ```python
  with CaptureQueriesContext(connection) as ctx:
      serialize_records(records)
  assert len(ctx) == 1
  ```

See the references for the full patterns behind each.
