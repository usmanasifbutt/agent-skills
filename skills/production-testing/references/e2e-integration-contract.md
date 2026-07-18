# E2E, Integration, Contract & Performance Testing

## E2E (browser) — page objects & discipline

> The suite here runs on a WebdriverIO-style browser framework with a Mocha runner — **not
> Playwright**. The patterns are tool-agnostic and map cleanly onto Playwright (page objects,
> role/label selectors, retries, sharding).

**Layered structure (reuse over duplication):**
```
e2e/
├── common/      # base Page class + CommonSelectors (shared by ALL apps)
├── helper/      # waits, validation, reporting, i18n label lookup, quarantine
├── api/         # data setup/teardown, URL constants
├── shared/specs/# specs reused across brands
├── core/        # shared page objects: common|desktop|mobile
└── <app>/       # per-brand layer: pages, specs, config
```
Runtime import aliases resolve layers by the selected app, so a shared spec imports page objects
"for the current app" and gets brand overrides when they exist, else the shared object.

**Page-object pattern & method ordering.** Page objects are classes extending a base `Page`;
selectors are separate classes extending `CommonSelectors`. The base groups its API in the canonical
order **wait → action → verify**, and a well-formed page method follows the same shape internally:
**wait for readiness → act → verify**.
- wait: `waitForElementDisplayed`, `waitForElementToHide`, `waitForAnimationToFinish`…
- action: `clickElement` (scrolls into view first — the preferred click), `enterInputElement`,
  `selectFromDropdown`… (`hardClickElement` JS-click only as a last resort).
- verify: `verifyURLEquals`, `verifyElementExistence`…

**Selector discipline.** Primary strategy is **`aria-label` (via XPath)** — which doubles as
accessibility pressure. Localized text goes through a `getLabel(key)` helper (never hard-coded per
language). **No raw selectors in generic/base files** — reusable selectors live in the selector
classes so a DOM change is fixed in one place; check the shared `CommonSelectors` before adding one.

**Shared vs per-brand specs.** Shared specs import page objects dynamically "for the current app",
so **one spec runs against every brand** (the per-app pages supply overrides); a brand's config lists
which suites it runs. App-specific specs import platform paths directly. `describe('Verify <feature>
- <platform>')`, `it('should …')`; multi-page flows factored into reusable `steps/` modules.

**"A test must be able to fail."** After writing/fixing a spec, run it and confirm success — and
guard the inverse failure mode: a spec that **passes without asserting anything** (e.g. a
`forEach(el => it(...))` when the selector is broken produces zero assertions). Add a minimal guard
(`expect(elements.length).toBeGreaterThanOrEqual(min)`), and prove a new spec can go red before
trusting green.

**Never skip to green (strict).** Forbidden: env-based skips, undocumented conditional assertions,
commenting out cases, `.only()`/`.skip()` to dodge execution, `browser.pause()` for timing. Required:
root-cause the failure, fix selector/timing/logic, add a proper `waitFor…`, inspect the live DOM.
Temporary behavior differences get a documented conditional (TODO + tracking link + removal plan);
permanent ones get separate spec files.

**Flakiness handled as data, not by disabling code.** A **DB-backed quarantine system** is the source
of truth: an append-only flake log (one insert per CI job — no read-modify-write races), an
auto-quarantine threshold over a rolling window, and quarantined specs run as **skipped-but-reported**
(session stubbed so no browser compute is paid). To test a fix without un-quarantining globally
(which affects every open PR), add a **branch-scoped override**; record a resolution **only after the
fix reaches production** (resolving early resets the counter while it's still flaky on the main
branch).

**E2E DB etiquette.** Runs against a **restored database dump** (immutable per-brand baseline baked
into the CI DB image). Do explicit **per-spec data setup/teardown** via the `api/` helpers rather than
mutating shared fixtures; coordinate any dump change through the DB-build workflow.

**Wiring a new folder into CI.** A new spec folder does **not** run until its glob is registered as a
**suite** in that brand's config and listed in the brand's CI matrix.

- **Common mistakes:** raw XPath in a shared base page (breaks all brands, un-fixable in one place);
  `browser.pause()` instead of a wait; un-quarantining globally to test a fix (use a branch override);
  resolving a flake before it's in production; a new folder not registered as a suite (silently never
  runs).

## Integration tests

Cross-boundary flows tested end-to-end against a real DB, under their **own config and a separate CI
job** (so the fast unit suite still gates most PRs): upstream-source sync, search-index
reconciliation, multi-step pipelines. Seed reference data from serialized fixtures where a full
object graph is needed. Keep single-function logic out of this tier.

## Contract testing

- **No Pact / consumer-driven-contract tooling.** Two things stand in for it:
- **Schema-shape contract tests** — a lightweight standalone suite validates a live environment's API
  **response shapes** against declarative schemas (regex field patterns, nested objects,
  `eachItemMatches(arraySchema)`, ignorable keys) via a `validateSchema({ reference, objectToValidate,
  strict })` helper, run with no browser against a target env (`APP_URL=… run api-contracts`), one
  schema + one test per endpoint group.
- **Informal consumer-contract discipline (the real contract)** — a **separate mobile/3rd-party team
  consumes the serializer payloads**. The rule: **keep serializer output backward compatible; additive
  changes only; confirm any breaking change to an existing field/shape with that consumer before
  shipping.** Localized fields are emitted through shared serializer helpers so the wire shape stays
  consistent.
- **Phrasing to reuse:** "We don't run Pact. We (a) assert response shapes with schema-based contract
  tests against a live environment, and (b) treat a downstream consumer of our serializers as an
  implicit contract — additive changes only, breaking changes confirmed first."
- **Common mistake:** renaming/removing/retyping an existing serialized field for frontend
  convenience without checking the downstream consumer.

## Performance testing

- **No k6/autocannon load testing** in these repos. Performance is watched two ways:
- **Lighthouse CI — scheduled, not per-PR.** A nightly cron runs Lighthouse (performance category,
  multiple runs, throttling/CPU-slowdown) against dev/staging/production via a matrix, resolving real
  URLs dynamically and uploading to a self-hosted LHCI server for trend tracking.
- **Bundle-size** tracked as its own workflow; targeted map/perf helpers exist in the E2E helper dir.
- **Backend micro-benchmarks** live in the backend suite via `pytest-benchmark`, gated behind a
  marker (see `backend-testing.md`).
- **Common mistake:** expecting Lighthouse to gate a PR — it's a scheduled trend monitor across
  environments, not a merge gate. (If you need per-PR budgets, add explicit assertions/thresholds.)

## CI integration (E2E & cross-cutting)

- **Matrix** fans out per **brand × app × shard × service-group**; the busy brand is sharded, others
  unsharded. A reusable template pulls the frontend image by content hash, restores the brand's DB
  backup image, runs the suite, uploads results to a test-reporting service.
- **Rerun-only-failed** for both unit and E2E (a labeled re-run workflow re-runs just the extracted
  failures; a passing rerun overrides the red check). It **fails loudly** if results can't be parsed,
  so a crash never masquerades as green.
- **Smoke tests** run on cron (several times/weekday) plus manual dispatch, notifying on failure.
  **Review-app E2E** can target an ephemeral environment URL.
- **Concurrency** groups with `cancel-in-progress` kill superseded runs; brand jobs whose builds are
  unchanged are skipped.
- **Gating:** the unit-test check and per-brand E2E/visual checks must be green to merge; lint/types
  is a separate required check; coverage is reported but not a numeric gate. Flaky specs are
  auto-excluded via quarantine so they don't block unrelated PRs.
- **Common mistake:** adding a brand/shard's specs but not adding the brand to the CI matrix (job
  never runs); relying on a full re-run when the rerun-failed path re-runs only extracted failures.
