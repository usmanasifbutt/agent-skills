---
name: production-react-engineering
description: How to build a production, server-rendered React frontend the disciplined way — a multi-tenant/multi-brand layered structure, hand-rolled component primitives with CSS Modules, a custom SSR router, plain Redux + a custom Fetcher framework (no React Query), Keycloak-via-BFF-cookie auth with role predicates, an HTTPApi/fetch client, a Redux-integrated search abstraction over a hosted search SaaS or Elasticsearch, Formik+Yup forms, lightweight SVG charts, and code-split loadable components. Load this BEFORE writing or reviewing ANY frontend code — components, hooks, routes, Redux state, API calls, auth/protected routes, forms, search, charts, config/feature flags, or tests — and when scaffolding a new React frontend that should follow these conventions.
---

# Production React Engineering

Engineering standards for a production React frontend built for SSR and multi-tenant delivery.
React 17 + TypeScript, server-side rendered, serving many branded portals from a single
codebase (plus a React Native mobile app and a separate Nx-based real-time platform for
contrast).

**The single most important thing:** this codebase deliberately does **not** use many popular
libraries. Before reaching for one, check this list — there is an established house pattern
for each:

| You might reach for… | The house standard is… |
|----------------------|------------------------|
| React Query / SWR | a custom **Fetcher framework** (`FetcherFactoryWithContext`) + Redux |
| Redux Toolkit / Zustand | **plain Redux** + `redux-thunk` + `reselect` + a reducer-manager |
| React Router | a custom **isomorphic (SSR) router** |
| `React.lazy` / `<Suspense>` | a **code-splitting/loadable library** (`Loadable`) |
| Material UI / Tailwind / styled-components | **hand-rolled primitives** + **CSS Modules (`.module.css`)** + `classnames` |
| react-hook-form | **Formik** |
| Zod (for UI) | **Yup** |
| recharts / chart.js / d3 | a **lightweight SVG charting library** |
| axios | **`fetch`** via the `HTTPApi` base class |
| `keycloak-js` browser adapter | a **Redux auth SDK + Express BFF cookie bridge** |
| Vendor InstantSearch UI / direct search-engine SDK calls | a **Redux-integrated search abstraction** (hosted-SaaS / Elasticsearch backends) |
| runtime `useMediaQuery` | **build-time desktop/mobile bundle split** |

Match the file you're editing, then match the repo, then follow this skill.

## Non-negotiable foundations (the short list)

1. **`import * as React from 'react'`**; prefix hooks `React.useState` etc. Props in a
   `readonly` `Props` **type** (never `interface`). Default-export the component.
2. **Styling is CSS Modules (`.module.css`) + `classnames` + CSS-variable theme tokens** — no CSS-in-JS,
   no Tailwind, no UI kit.
3. **Never call `fetch()` directly** — go through an `HTTPApi` subclass (`BackendAPI`/`InternalAPI`)
   in `core/api`. Never call search-engine SDKs from components — go through the search abstraction.
4. **Auth tokens live in HttpOnly cookies set by the BFF**, never in Redux/localStorage. Read
   identity via selectors/`useUser`-style hooks (client) or `ServerAuthContext` (server).
5. **Data fetching is the Fetcher framework**; loading/error come off the `FetchState` slice.
   Server-load data in a route's `onEnter` via `context.promise.wait(...)` for SSR.
6. **Every user-facing string is localized** with an ICU-message i18n library (`<Trans>` / `t`);
   run the i18n message-extraction command before committing string changes.
7. **`lint:types` is clean; no `as`, no `@ts-ignore`/`@ts-expect-error`** in product code.
8. **Feature flags are `APP_ENABLE_*` config settings with an expiry**; gate in JSX or a
   selector; delete the flag + its checks when fully rolled out.
9. **Put code at the right layer** (shared core → shared UI library → brand layer); import features
   through their `index.ts` barrel, never deep paths.
10. **Code-split with `Loadable`**, memoize deliberately (`useMemo`/`useCallback`/`React.memo`/
    reselect), and rely on the global render-batching already wired into the store.

## Reference map — read the file for the area you touch

| Working on… | Read |
|-------------|------|
| Project layout, components, hooks, CSS Modules, TypeScript, naming, folder org | `references/structure-components-styling.md` |
| The custom router, config/settings, feature flags, i18n | `references/routing-config-i18n.md` |
| Auth/Keycloak, RBAC, protected routes, HTTPApi/axios, error handling, loading | `references/auth-api-errors.md` |
| Redux, the Fetcher framework, caching, SSR/hydration, performance, code splitting, virtualization | `references/state-data-performance.md` |
| Component library, styling system, responsive, accessibility, forms (Formik), validation (Yup), charts | `references/ui-forms-charts.md` |
| Search pages & search-service integration (the search abstraction), testing, the other apps | `references/search-testing-other-apps.md` |

## Architecture guidelines

- **Layer by sharing scope.** The shared core = platform plumbing (router, state, API, i18n, BFF);
  the shared UI library = generic UI shared across all products; brand layers = brand-specific code;
  country app folders = country-level locale/assets/overrides. Push code as high as it correctly
  lives; a component used by two products belongs in the shared UI library.
- **Feature folders with a public `index.ts` barrel.** A feature owns its components, hooks,
  small single-purpose logic files, `state/`, `types.ts`, `styles/`, `__tests__/`. Consumers
  import from the barrel; internals stay private and refactorable.
- **Thin components, logic in Redux/selectors/hooks.** Views render; derived data is a
  `reselect` selector; side-effects and fetching are thunks / the Fetcher framework; reusable
  behavior is a `useX` hook.
- **SSR is a first-class constraint.** Data loads in `onEnter` and is awaited via
  `context.promise.wait`, serialized into `window.state`, and hydrated without refetching.
  Anything touching `window`/`document` must be guarded and hydration-safe; secrets stay
  behind `process.env.IS_SERVER`.
- **One codebase, many portals.** Ask "does this apply to all portals or some?" Gate portal
  differences behind `APP_ENABLE_*` flags or config, not hard-coded brand checks.
- **Composition over configuration in UI.** Build new components from the existing primitives
  (`Button` uses `Text`, `Dialog` wraps the modal) rather than adding a UI dependency.

## React checklist (before you push a component/feature)

- [ ] `import * as React`; hooks `React.`-prefixed; `readonly` `Props` type; default export.
- [ ] Styling via `.module.css` + `classnames` + CSS-variable tokens; no inline hard-coded colors,
      no CSS-in-JS/Tailwind/UI-kit.
- [ ] All user-facing strings localized; `aria-label`s localized too; the i18n message-extraction
      command run.
- [ ] Data via the Fetcher framework or an `HTTPApi` subclass — no direct `fetch`; loading/error
      read off the slice; a `LoadingSpinner`/skeleton for loading and an error UI for failure.
- [ ] Selectors memoized with `createSelector` when derived/non-primitive; stable refs via
      `EMPTY_OBJECT`/`EMPTY_ARRAY`; memoize callbacks passed to memoized children.
- [ ] Accessible: semantic HTML, `aria-*` on interactive elements, keyboard support + focus
      management for anything interactive.
- [ ] Responsive handled via the desktop/mobile page split + breakpoint tokens (no runtime
      viewport branching in JS).
- [ ] Heavy/rarely-used component code-split with `Loadable` (with `webpackChunkNames` +
      `modules`), wrapped in an error boundary.
- [ ] Forms use Formik; validation uses Yup with localized messages; errors gated on
      `touched && errors`.
- [ ] Feature gated behind an `APP_ENABLE_*` flag if it's a rollout; flag has an expiry.
- [ ] `npm run lint:types` clean; no `as`, no `@ts-ignore`/`@ts-expect-error`/`@ts-nocheck` in
      product code.
- [ ] Imports go through feature `index.ts` barrels, not deep paths; code lives at the correct
      layer.
- [ ] Tests: component/hook via `renderWithState`/`renderHookWithState`, `CONFIG`/env mocked
      with `addToConfig`/`addToEnv` + `.restore()`; selectors/reducers/URL logic covered.

## PR checklist

- [ ] `lint:types` and the linters (ESLint incl. `jsx-a11y`, Prettier) pass; the i18n
      message-extraction command committed if strings changed.
- [ ] Multi-portal impact considered; shared core / shared UI library changes verified against more
      than one brand; portal-specific behavior gated by flag/config.
- [ ] SSR-safe: no unguarded `window`/`document`; hydration matches server output; no secret in
      `CONFIG.build`/client bundle.
- [ ] No new dependency unless justified (check `npm explain` for an existing/indirect one);
      no reach for a library the house already replaces (React Query, MUI, axios, …).
- [ ] Backwards-compatible API/response usage; new settings added to the settings config +
      the config types file + the config manifest.
- [ ] Errors reported via `logError` (→ logger + the error tracker), not `console.error`; user-facing
      errors surfaced (form error / toast), not swallowed.
- [ ] Tests added/updated and run; no `//@ts-nocheck` outside test files.
- [ ] CSS classes/modules confirmed unused elsewhere before rename/delete.

## Common anti-patterns (reject in review)

- ❌ `import React from 'react'` / un-prefixed hooks / `interface Props` — use the namespace
  import, `React.`-prefixed hooks, `type Props`.
- ❌ `fetch()`/axios in a component; calling search-engine SDKs directly — use `HTTPApi`
  subclasses / the search abstraction.
- ❌ Adding React Query, Zustand, Redux Toolkit, MUI, Tailwind, styled-components,
  react-hook-form, `React.lazy`/`Suspense` — all have a house replacement above.
- ❌ Tokens in Redux/localStorage; a client-side `keycloak-js` adapter — auth is BFF cookies +
  the Redux auth SDK.
- ❌ Hard-coded user-facing strings or hard-coded language codes; forgetting message extraction.
- ❌ `as` casts / `@ts-ignore` to silence the type checker; `//@ts-nocheck` in product code.
- ❌ Inline styles / hard-coded colors instead of `.module.css` + CSS-variable tokens; building
  `className` strings by hand instead of `classnames`.
- ❌ Deep cross-feature imports (`core/search/internal/foo`) instead of the `index.ts` barrel.
- ❌ Per-component loading booleans and ad-hoc caches instead of the Fetcher's
  `FetchState`/dedup/caching; unmemoized derived selectors causing re-renders.
- ❌ Fetching in `useEffect` for data that should be SSR-loaded in the route's `onEnter`.
- ❌ Hard-coded brand checks (`if (brand === 'someBrand')`) instead of a flag/config.
- ❌ New settings read via bare `process.env` in client code (leaks/undefined) instead of
  `CONFIG.runtime`/`CONFIG.build`.

## Reusable snippets

**Component + CSS module**
```tsx
import * as React from 'react';
import classNames from 'classnames';
import styles from './styles/priceTag.module.css';

type Props = {
    readonly amount: number;
    readonly highlighted?: boolean;
};

const PriceTag = ({ amount, highlighted }: Props) => (
    <span className={classNames(styles.tag, { [styles.highlighted]: highlighted })}>
        {amount}
    </span>
);

export default PriceTag;
```

**Fetcher-backed data (replaces React Query)**
```ts
export const recordFetcher = new FetcherFactoryWithContext<Params, RecordData>({
    name: 'RECORD',
    fetch: ({ params }) => new BackendAPI().record(params.recordID),
    options: { cacheResults: true, dropStaleResponses: true },
}).build();

// component: read status off the slice
const { loading, data, error } = useSelector(selectRecordFetchState);
if (loading) return <LoadingSpinner />;
if (error) return <ErrorMessage reason={error.reason} />;
```

**Memoized selector**
```ts
const selectVisibleRecords = createSelector([selectRecords, selectFilters], (records, filters) =>
    records.filter((record) => matches(record, filters)),
);
```

**API subclass (never call fetch directly)**
```ts
class BackendAPI extends HTTPApi {
    record(recordID: number) {
        return this.request(`/api/record/${recordID}`);
    }
    protected authHeader() {
        return process.env.IS_SERVER ? ServerAuthContext.asBackendHeaders() : {};
    }
}
```

**Route with SSR data load + auth gate**
```ts
onEnter(context: RoutingContextWithMiddlewares): void {
    if (!selectIsUserLoggedIn(context.redux.store.getState())) {
        redirectToURISafely(context, loginPath);
        return;
    }
    const { recordID } = context.match.params;
    context.promise.wait(
        context.redux.store.dispatch(fetchRecord({ recordID }))
            .then(() => context.rendering.renderPage(Page.RECORD_DETAILS))
            .catch((e) => { if (isAPIError(e, APIErrorReason.NOT_FOUND)) renderNotFoundPage(context); else throw e; }),
    );
}
```

**Code-split component (replaces React.lazy)**
```tsx
const HeavyDialogAsync = Loadable({
    loader: () => import(/* webpackChunkName: "heavyDialog" */ './heavyDialog'),
    webpackChunkNames: ['heavyDialog'],
    modules: ['./heavyDialog'],
    loading: LoadingSpinner,
});
```

**Feature flag**
```tsx
{CONFIG.runtime.APP_ENABLE_MY_FEATURE && <MyFeature />}
// or, for cross-cutting logic, inside a selector:
const selectMyFeatureEnabled = createSelector([selectX], (x) => x && CONFIG.runtime.APP_ENABLE_MY_FEATURE);
```

**Formik + Yup (localized messages)**
```ts
const schema = (i18n) => Yup.object({
    name: Yup.string().trim().required(t(i18n)`Please enter your name`),
    phone: Yup.string().phone().required(t(i18n)`Please enter your phone`),
});
```

**Localized accessible label**
```tsx
<button aria-label={t(i18n)`Close dialog`} onClick={onClose}>×</button>
```

**Error boundary + logging**
```tsx
export default withErrorBoundary(MyComponent);
// on catch, logError({ e, msg: 'MyComponent crashed', context: {...} }) → logger + the error tracker
```

**Test with mocked store + config**
```tsx
beforeAll(() => addToConfig({ APP_ENABLE_MY_FEATURE: true }));
afterAll(() => CONFIG.restore());
it('renders the feature', async () => {
    const { getByRole } = await renderWithStateAsync(<MyFeature />, { state });
    expect(getByRole('button', { name: 'Save' })).toBeInTheDocument();
});
```
