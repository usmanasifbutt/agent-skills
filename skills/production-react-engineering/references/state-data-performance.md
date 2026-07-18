# State Management, Data Fetching, Caching & Performance

**Library verdicts (verified against package.json + imports):**

| Concern | Standard | Verdict |
|---------|----------|---------|
| State | plain **Redux** + `redux-thunk` + `reselect` + `react-redux` 8 | ✅ |
| Redux Toolkit | — | **NOT used** (plain `redux`) |
| Zustand | — | **NOT used** |
| React Query / TanStack Query | — | **NOT used** |
| SWR | — | **NOT used** |
| Data fetching | custom `FetcherFactory` / `FetcherFactoryWithContext` (Redux-thunk) | ✅ |
| Code splitting | a code-splitting/loadable library | ✅ |
| `React.lazy` / `Suspense` | — | **NOT used** |
| Virtualization | a windowing/virtualization library (one app only, narrow) | ✅ narrow / ✗ elsewhere |
| Image lazy-load | a lazy-load-on-scroll library | ✅ |

When a task calls for "add React Query / a Zustand store / Suspense", **don't** — use the
house patterns below (there's an established, SSR-aware equivalent for each).

## State — Redux + thunk + reselect

- **Plain Redux** (not RTK) + `redux-thunk` (async actions), `reselect` (memoized selectors),
  `react-redux` 8. redux-thunk abstracts async actions together with state transitions.
- **Store bootstrap** (`core/state/createDefaultStore.ts`): base middlewares
  `[thunk, promiseMiddleware, cookieMiddleware, taskSchedulerMiddleware]`, base reducers, and
  **a batched-subscribe enhancer** wrapping `ReactDOM.unstable_batchedUpdates` in a debounce
  so a burst of dispatches produces **one** render pass — a deliberate global perf convention.
- **Code-split state:** `createReducerManager` implements the reducer-manager pattern —
  `store.reducerManager.add('key', reducer)` / `.remove(key)` attach/detach per-page slices at
  runtime. App middlewares are assembled in `core/state/store.ts` as a big
  `[...].filter(Boolean)` array gated on `settings.*`/`CONFIG.*` flags.
- **Feature state co-located** under `core/<feature>/state/`, registered in
  `core/state/defaultReducers.ts`. Typed helpers `GlobalState`, `AppDispatch`, `Store` from
  `core/state`.
- **Selectors:** `select*` naming; plain selectors for cheap field access, **`createSelector`
  for derived/computed non-primitive output** (used in well over a hundred files). Keep
  referential stability with `EMPTY_OBJECT`/`EMPTY_ARRAY` sentinels from `core/empty`.

**When to use Redux:** complex state shared across components, or anything that must be
populated server-side and survive hydration. Local, ephemeral UI state stays in
`React.useState`.

## Data fetching & caching — the Fetcher framework (React Query's replacement)

Instead of a query library, the team hand-rolls a Redux-thunk **`FetcherFactory`** /
**`FetcherFactoryWithContext`** (`core/fetcher/`; an older single-file `fetcher.ts` exists in
one app). Each factory generates start/success/error/preload/clear actions + a reducer,
storing results in a `FetchState` slice `{ loading, loaded, preloaded, enabled, params, data,
error }`. It gives you, for free, exactly what React Query would:

1. **Param-equality caching** — identical params ⇒ return stored data, no network call.
2. **In-flight de-duplication** — a module-level `pendingPromises` map keyed by
   `JSON.stringify([name, params])`; concurrent identical calls share one promise.
3. **Opt-in in-memory result cache** (`cacheResults: true`) keyed by `JSON.stringify(params)`.
4. **Stale-response dropping** (`dropStaleResponses`) — a late response whose params no longer
   match current state is ignored, so an old request can't clobber a newer one.

`FetchOptions`: `successCodes` (200/304 always), `getParams`, `skipParamsCheck` (force
refetch), `cacheResults`, `checkParamsEquality`, `dropStaleResponses`. **Why not React
Query:** the fetcher is SSR-native (results land in the Redux store the server serializes into
`window.state`, so the client hydrates without refetching) and integrates with the existing
thunk/selector machinery.

**Loading states** are read straight off the `FetchState` slice (`loading`/`loaded`/`error`)
via selectors — components render a `LoadingSpinner`/skeleton while `loading`, the data when
`loaded`, an error UI on `error`. Don't invent per-component loading booleans; use the
fetcher's status.

**Persistence:** `saveInLocalStorage.ts` writes an `activity` blob (search history, recently
viewed, leads) to **localStorage**; `drainStateFromLocalStorage.ts` rehydrates it into Redux
on boot.

**Direct calls:** one-off requests use `BackendAPI`/`InternalAPI` directly in a route/thunk;
search goes through the Redux-integrated search abstraction. See the API reference.

## SSR & hydration

The custom router is built for SSR. A route's `onEnter` dispatches fetches and registers the
aggregate promise via **`context.promise.wait(promise)`**; the server awaits all registered
promises, so the Redux store is fully populated before render. State is serialized into
`window.state` (`connectAppRenderer.ts`, via `serialize-javascript`, with
`cleanSensitiveServerState` stripping keys); the client (`app/browser.ts`) builds a store from
`window.state` and calls `ReactDOM.hydrate` — no client refetch because the fetcher's
param-equality check sees the data already present. `onClientEnter` runs client-only work
after hydration. **Why it matters:** SEO + fast first paint for a public site; every
data-loading pattern must be hydration-safe.

## Performance

- **Global render batching** (the batched-subscribe enhancer + `unstable_batchedUpdates`) is
  the primary optimization; a `TaskScheduler` flushes work `AFTER_RENDER`.
- **Memoization is idiomatic and heavy:** `useCallback` (hundreds of files), `useMemo`,
  `React.memo`, `reselect` selectors, and `EMPTY_OBJECT`/`EMPTY_ARRAY` for referential
  stability. Memoize values/callbacks passed to memoized children and expensive computations —
  but don't reflexively wrap everything; measure.
- **Prefetching:** `loadable/prefetchPage.tsx` preloads a page's chunk during
  `requestIdleCallback`; `page.preload()` is awaited before hydration to avoid an empty flash.
- **Image lazy-load** via a lazy-load-on-scroll library (`LazyLoad`) on cards and galleries.

## Code splitting & lazy loading (no `React.lazy`/`Suspense`)

- **Async components** use a code-splitting/loadable library via `core/loadable/loadable.tsx`
  (`Loadable(...)`). Every loadable **must** declare `webpackChunkNames` + `modules` (enforced
  by `invariant`), wraps content in an `ErrorBoundary`, and preserves SSR markup via
  `PreserveServerSideRendering`. Dozens of `*Async.tsx` modules use it.
- **`React.lazy` and bare `<Suspense>` are NOT used** (zero `<Suspense>` in the codebase) —
  they lacked the SSR support and chunk-name control the app needs on React 17; the loadable
  library provides both. When adding a lazily-loaded component, use `Loadable`, not
  `React.lazy`. (On newer React versions, `React.lazy` + `<Suspense>` with streaming SSR is
  the modern standard alternative.)
- **Per-page bundle splitting + traditional page load:** `historyRouter.ts` `pushRoute` soft-
  navigates (SPA) only when `isRouteInBundle(name)`; otherwise `window.location.href = url`
  triggers a full reload into the other bundle. `HardLink` renders a plain `<a href>` for
  cross-bundle links.
- **Build:** one frontend app builds with **rspack**, the other with **webpack 5**.
  `splitChunks` uses an `enforce`d `framework` group (`react|react-dom|scheduler`) and an
  intentionally **unnamed** `vendors` group so each entrypoint pulls only the vendor code it
  uses. Separate bundle per portal per layout (desktop/mobile/tablet); manifest from
  `build/stats.json` (`app/bundles.ts`), SSR inlines exactly one CSS/JS pair per entrypoint.

## Virtualization

- **A windowing/virtualization library** (`useVirtualizer`) is used **narrowly in one app
  only** — `components/virtualizedInfiniteList.tsx` (paired with a `useAutoLoadMore`
  infinite-scroll hook and dynamic `measureElement` sizing), consumed by **long
  selector/dialog lists** (`virtualizedSearchInput`, member-selector/record-linking dialogs),
  **not** the main search results (that uses server-side pagination + lazy-loaded images).
- **No `react-window` / `react-virtualized`.** **The other frontend app has no virtualization
  at all** — a divergence; virtualize only when a list is genuinely long and unbounded.
