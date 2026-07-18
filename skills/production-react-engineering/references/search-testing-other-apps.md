# Search & Search-Service Integration, Testing, and the Other Frontends

## Search pages & search-service integration

**Search does NOT hit the database or the search engine directly from the browser, and does
NOT use a vendor InstantSearch-style UI.** Everything goes through an in-house
**Redux-integrated search abstraction**, with pluggable backends: **a hosted search SaaS**
(default), **Elasticsearch**, and a **Generic** backend. The same client runs on the Express
BFF (SSR) and in the browser.

- **Backend selection** (`core/search/searchBackend.ts`): a build flag `APP_SEARCH_BACKEND`
  (`saas` | `elastic` | `generic`) picks the backend via `createKnownSearchBackend(...)`. On
  hydration, `app/browser.ts` dispatches `setBackendOptions({ saas: { appId, apiKey,
  hosts } })` so the search middleware can run searches from Redux.
- **API-key privacy split** (`search/backend/determineSearchAPIKey.ts`): the browser gets
  `CONFIG.runtime.SEARCH_BROWSER_API_KEY` (public), the server uses
  `process.env.SEARCH_API_KEY` (private) — `process.env.IS_SERVER ? private : public`.
  **The private key never leaves the Node server.**
- **Query building:** the unit is a `SearchJob(indexName, filters, options)` run by a
  `SearchService({ backend })`; filters are a typed `FilterCollection`
  (`ExactFilter`, `RefinementFilter`, …). The **main results page builds a map of named jobs**
  run together (`routes/search/createSearchJobs.ts`: `SearchJobName` enum → RECORDS,
  CATEGORIES, ORGANIZATIONS, USERS). Pagination is `page`/`hitsPerPage`; facets per job.

```ts
const job = new SearchJob(indexName, filters, { facets: [], hitsPerPage });
const service = new SearchService({ backend: configuredBackend });
return service.fetchJob(job);
```

- **Results page** (`search/search.tsx` → `SearchResults`/`SearchHits`/`SideBar`, map view
  lazy via `Loadable`). **URL ↔ filter state** is first-class: `search/url.ts`
  (`selectSearchLink`, reselect over the `FilterCollection`), `createSearchParams.ts`
  (URL→filters), `state/urlUpdater.ts` (`updateURL`), regex URL patterns per locale in
  `routes/search/flatSearchRoute.ts` (`category`, `page-N`, plus per-locale pattern sets).
  Filter components sit flat in `search/` (`priceFilter.tsx`, `categoryFilter.tsx`, …) with
  hooks (`usePriceFilter.ts`); pagination in `search/pagination/`. Component↔Redux glue uses
  HOC connectors (`connectSearchLink`).
- **Elastic** is reached via the search abstraction's elastic backend or **raw HTTP to
  `ELASTIC_SEARCH_API_URL` on the BFF** — there's **no `@elastic/elasticsearch` npm client**.
  Direct Elastic BFF reads exist for a few adjacent features — not the main records search.
- **Why this design:** one search abstraction lets the same filter/URL/Redux code target the
  hosted SaaS or Elasticsearch per portal, keeps the private search key server-side, and runs
  on both SSR and client. When adding search behavior, extend `FilterCollection`/`SearchJob`
  and the URL processors — don't call search-engine SDKs directly from components.

## Feature flags & environment variables in practice

Three-tier `CONFIG` global (from the settings config):
- **`CONFIG.build.*`** — build-time, inlined by DefinePlugin, **publicly visible** (the file
  warns "DO NOT INCLUDE SECRETS HERE").
- **`CONFIG.runtime.*`** — injected at run time (not baked into the bundle).
  `runtimeSettings.ts`: server merges core + project runtime settings; browser reads
  `window.CONFIG.runtime`.
- **`process.env.*`** — secrets, only read under `process.env.IS_SERVER` (guarded).

**Feature flags are `APP_ENABLE_*`**, each `(process.env.APP_ENABLE_X || 'false') === 'true'`.
Gate them **inline in JSX** for simple show/hide
(`{CONFIG.build.APP_ENABLE_MY_FEATURE && <...>}`) or, preferred for cross-cutting logic,
**inside a selector** (`selectIsMyFeatureEnabled = createSelector(..., v => v &&
CONFIG.runtime.APP_ENABLE_MY_FEATURE)`). See `routing-config-i18n.md` for adding/expiring
flags. Env conventions: comma-lists via `(process.env.X || '').split(',').filter(Boolean)`,
JSON via `JSON.parse(...)`, A/B via `DISABLE_EXPERIMENTS`/`EXPERIMENTS_OVERRIDE_VARIANT`.

## Testing

- **Jest 29 + React Testing Library 12** (`@testing-library/react`, `/react-hooks`,
  `/user-event`) is the standard; **legacy enzyme** (`@wojtekmaj/enzyme-adapter-react-17`)
  survives in old tests. One Jest **project per app** (`jest.config.js` `projects:
  [createConfig('core'), ...]`), SWC transformer, plus `jest-fetch-mock`,
  `jest-localstorage-mock`, `@googlemaps/jest-mocks`, `jest-junit`.
- **Shared helpers** (`core/test/`, re-exported from `test/index.ts`):
  - `renderWithState` / `renderWithStateAsync` — render a component with a real Redux store +
    router + i18n provider; the async variant invokes the router and awaits
    `context.promise.wait()` (for route-driven data).
  - `renderHookWithState` — `renderHook` with the same store/router wrapper.
  - `addToConfig(...)` / `addToEnv(...)` — reassign the global `CONFIG` / `process.env` with
    merged values and a `.restore()`; convention: set in `beforeAll`, `CONFIG.restore()` in
    `afterAll`.
- **What's tested & where:** per-feature `__tests__/` folders; files `*.test.ts(x)`; snapshots
  in `__tests__/__snapshots__/*.snap`; module mocks in sibling `__mocks__/` (e.g. a
  `MockSearchServiceDB` with jest-mocked `fetchJob`/`fetchJobs`). Coverage spans components,
  pure helpers/reducers, selectors, and URL logic. **Note:** many test files carry a top-level
  `//@ts-nocheck` — tolerated in tests, never in product code.
- E2E is separate (WebdriverIO under `frontend/e2e/`, per brand).

```ts
beforeAll(() => addToConfig({ LOCATION_WITH_CITY_LEVEL_SCORE: ['central-district'] }));
afterAll(() => CONFIG.restore());
it('returns CITY_LEVEL_SCORE', () => expect(getDefaultSortByValue([])).toBe(SortByValues.CITY_LEVEL_SCORE));
```

## The other frontends (context, not the primary standard)

- **A separate Nx-based real-time platform:** a real-time/collaboration + data-migration
  platform, not a product frontend. Groups projects under `packages/` (`packages/apps/*`
  deployables, `packages/client/*` publishable SDKs/UI). The React surface is an `admin-portal`
  app (React 18 + `react-redux` 7 + `redux`/`redux-thunk`) and a `client/widgets` component
  library bundled with **Rollup** for external consumption. Classic Nx app/lib separation;
  **older Redux 3 / react-redux 7 stack** — do not treat as the primary standard.
- **A React Native mobile app (RN 0.69):** it **mirrors the web architecture** — shared core +
  feature-folder + `index.ts` barrel, per-brand app wrappers, i18n, and the **same search
  stack** (the Redux-integrated search abstraction + the hosted-search client). RN-specific divergences:
  **React Navigation v6** (not the custom router), **UI Kitten** + `react-native-reanimated`/
  Lottie (not CSS Modules), **Redux + redux-saga** (not thunk/reselect middleware), and a
  **dual-vendor native layer** — Google Firebase **and** Huawei HMS for maps/push/auth — with
  `remote-config` as the RN analogue of `CONFIG.runtime`.
