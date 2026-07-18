# Frontend Testing (Jest + React Testing Library)

## Jest setup

- **Multi-project runner:** one root config, each app a `project` produced by a shared
  `createConfig(appName)` factory (sets `displayName`, `rootDir`, per-app globals, shared
  transform/resolver/setup). The root aggregates `collectCoverageFrom` across projects and registers
  one `globalSetup`.
- **Custom transformers:** TS/TSX via an SWC transformer mirroring the bundler (a small i18n
  pre-pass only for files importing message macros so IDs match extraction); tiny transformers map
  raw assets (`.txt/.html`→string, images→path, CSS→identity-proxy, SVG→dummy component).
- **Global mocks (in the shared setup file):** `global.fetch` stubbed to resolve empty (no real
  network by default); `global.CONFIG = { runtime:{}, build:{} }` populated per-app from the settings
  module (the bundler define-plugin doesn't run under Jest, so build/runtime/`process.env` values are
  loaded there); localStorage mocked before modules load; web/node globals polyfilled;
  `IntersectionObserver` stubbed; a request-context module reset in `beforeEach`.
- **Forced non-local timezone** in a custom jsdom environment so developers must reason about TZ;
  overridable per file via a docblock.
- **Custom resolver** maps app-prefixed import paths to the right app or the shared core.
- **Common mistake:** reading `CONFIG.build.X`/`process.env.X` in a test without seeding it — under
  Jest those only exist because setup loaded them; new keys must be added there (or via the mock
  helpers below).

## React Testing Library — render helpers

Wrap RTL so every test gets the same providers (**Redux store + router + i18n**):
- **`renderWithStateAsync(state, element)`** — the primary helper. Builds a store, creates a test
  router, **actually invokes the current route and awaits its data-loading promise** before
  rendering, then returns the RTL result. Exercises the real route/data flow (catches loader
  regressions a "pre-filled state" render would hide).
- **`renderWithState(state, element)`** — synchronous render for pure output/markup checks.
- **`renderHookWithState(state, hook)`** — `renderHook` under the same providers.
- **`ReduxStore.Builder`** — a fluent builder (`.withRouterURL().withPurpose().withPrice()…build()`)
  so a test declares only the slice it cares about; reducers/selectors are tested against it.
- **Interaction:** `user-event` for real typing/clicks; query by **role/text** (`getByRole`,
  `findByText`), not test-ids.

```tsx
const state = new ReduxStore.Builder().withRouterURL('/search').build();
const { getByRole, findByText } = await renderWithStateAsync(state, <SearchResults />);
await userEvent.click(getByRole('button', { name: 'Filters' }));
expect(await findByText('Price')).toBeInTheDocument();
```

- **Common mistake:** using the **sync** helper for a component that loads data via the route, then
  asserting on content that only appears after the route promise resolves — use the async helper.

## Mocking (frontend)

- **Restorable config/env:** `addToConfig(buildVars, runtimeVars)` and `addToEnv(vars)` merge into
  the global `CONFIG`/`process.env` and attach `.restore()`. Mutate in `beforeAll`/`beforeEach`,
  **always `.restore()` in teardown** so brands/tests don't bleed config.
- **Network:** `fetch` is globally stubbed; tests that need responses mock the **API client module**
  (via `__mocks__/`), not `fetch` directly.
- **Module mocks in `__mocks__/`:** hand-written mocks for the true boundaries — the backend API
  client, the search client (an in-memory "DB" with fluent seeders `.records([...])` + `reset()`
  that emulates simple filtering), analytics. Each exposes a stateful `setup()`/`reset()` API
  returning canned `{ status, data }`.
- **"Don't mock what you can render."** Mock only real external boundaries (network, search, auth
  provider, analytics); render real components/reducers/selectors through the provider wrapper.
```ts
beforeAll(() => addToConfig({ FEATURE_X_ENABLED: true }, {}));
afterAll(() => CONFIG.restore());
searchServiceMock.setup().records([{ externalID: '1', price: 100 }]);
afterEach(() => searchServiceMock.reset());
```
- **Common mistake:** mocking a component/selector you could render (over-mocking); seeding a
  `__mocks__` DB without `reset()` in teardown so later tests inherit stale hits.

## Snapshot testing

- Used for structural/markup stability (via the sync render + test renderer), with explicit
  `snapshotFormat` for stable diffs. But the broader philosophy leans on **behavioral assertions**
  (role/text queries, explicit expectations) — a large snapshot everyone re-blesses asserts nothing.
- Keep snapshots **small and paired with real assertions**; never reflexively `--snapshot-update`.

## Organization & naming

- **Location:** `__tests__/` next to the code; snapshots in `__snapshots__/`; hand-written mocks in
  `__mocks__/`. Files are `*.test.ts` / `*.test.tsx` (match `**/__tests__/**/?(*.)(spec|test).ts?(x)`).
- **`//@ts-nocheck` is tolerated at the top of test files and test helpers** so fixtures can be loose
  — a **test-only** allowance; **product code must remain fully typed** (fix TS errors properly, never
  suppress).
- **`describe`/`it` naming:** `describe('Verify <feature> - <platform>')`, `it('should <do X>')` —
  behavior-oriented.
- **Common mistake:** carrying `//@ts-nocheck` (or the looseness it enables) into product code.

## Coverage

- `collectCoverageFrom` composed per-project (`${app}/**/*.{ts,tsx}` minus top-level and `test/`
  dirs); reporter `lcov`. **No enforced numeric threshold** — coverage is collected/reported in CI
  but does not gate merges; the **test result** is the gate.
- Per-project collection keeps each brand's coverage attributable.
- **Common mistake:** assuming a coverage % blocks the PR — it doesn't; a passing-but-shallow test
  still merges, so make the tests meaningful.

## What to test

- **Components** — through the provider wrapper, asserting rendered output/behavior via role/text
  and `user-event`, not implementation detail.
- **Hooks** — via `renderHookWithState`.
- **Reducers & selectors** — as pure functions against a store built by the `ReduxStore.Builder`
  (derived/memoized selectors especially).
- **Route/data flow** — via the async render helper (don't hand-fill "loaded" state).
