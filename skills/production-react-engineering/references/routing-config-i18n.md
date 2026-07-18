# Routing, Configuration, Feature Flags & Localization

## Routing — a custom isomorphic router

The app uses a **custom isomorphic (SSR) router** (`frontend/core/router`), not React Router.
It runs the same route lifecycle on the server (for SSR) and the client.

### Defining a route
Each route extends `CancelableRoute`, is constructed with a name from the `RouteNames` enum
plus URL patterns (use `{ name, pattern }` for named capture groups), and **must implement
`createURL`** so links can be generated from params:

```typescript
class RecordRoute extends CancelableRoute {
    constructor() {
        super(RouteNames.RECORD_DETAILS, [
            ['^/record/', { name: 'recordID', pattern: '([0-9]+)' }, '\\.html', '(?:\\?)?'],
        ]);
    }
    createURL(params: { recordID: number }): EnhancedLocation {
        return { pathname: `/record/${params.recordID}.html` };
    }
}
export default new RecordRoute();
```

- **All route names live in the `RouteNames` enum** (`core/routes/routeNames`); routes are
  registered in a brand layer's `routes/index.ts`.
- **Why a custom router:** the app is split into **multiple bundles per portal/section**;
  the router navigates as an SPA when the target route is in the same bundle and as a
  traditional page load when it's in a different bundle, and it runs `onEnter` on the server
  for SSR.

### Links & router access
- **`<Link route={RouteNames.X} params={{...}}>`** renders an anchor via the route's
  `createURL` — never hand-build hrefs.
- **`useRouter()`** hook exposes the `Router`; the active route + params are at
  `router.context.match`.

### Route lifecycle (`onEnter`) — SSR data loading
`onEnter` runs server- and client-side. Read capture groups/query params from
`context.match.params`. For async work, hand the promise to `context.promise.wait()` so the
server waits before rendering, then call `context.rendering.renderPage(Page.X)`:

```typescript
onEnter(context: RoutingContextWithMiddlewares): void {
    const dispatch: AppDispatch = context.redux.store.dispatch;
    const { recordID } = context.match.params;
    context.promise.wait(
        dispatch(fetchRecord({ recordID }))
            .then(() => context.rendering.renderPage(Page.RECORD_DETAILS)),
    );
}
```

- Redux store via `context.redux.store`; i18n via `context.i18n` (`t(context.i18n)\`...\``).
- **Pages** are rendered with `context.rendering.renderPage(Page.<key>)` and must be mapped
  in both `core/pages/desktop.ts` and `core/pages/mobile.ts` (desktop/mobile split — see
  the UI reference for responsive design).
- **Errors / not-found:** `renderNotFoundPage(context[, status[, message]])` for 404/403;
  `context.http.status(404)` sets the code; `context.http.redirect(url, { status: 302 })`
  redirects. (Auth/role gating in `onEnter` is covered in the auth reference.)

## Configuration & feature flags

Three kinds of setting:
- **Runtime** — changeable without rebuilding; `CONFIG.runtime.<name>`.
- **Build-time** — baked in at build; `CONFIG.build.<name>`.
- **Secrets** — never in the settings config; `process.env.<SECRET_NAME>` only (still
  documented in the config manifest).

**Adding a setting:** (1) add to the frontend settings config under `runtime`/`build`,
(2) add its type to the config types file, (3) run the config-sync command to auto-add it
(sorted, tagged) to the config manifest.

**Feature flags** are just config settings **with an expiry date**. Prefix them
**`APP_ENABLE_*`** to get automatic expiry handling; otherwise add an expiry comment in the
config manifest (`# env-linter: expire | <date> | <email>`). Gate the feature in code on
the flag; **when fully rolled out, delete the flag from every config file and remove the
code that checks it.**

- **Why:** flags decouple deploy from release and enable per-portal rollout; the expiry
  mechanism forces cleanup so flags don't accumulate forever (the same discipline used on
  the backend's `APP_ENABLE_*` env-linter).
- **Access in code:** `if (CONFIG.runtime.enableMyFeature) { ... }`. Prefer runtime flags for
  anything you want to toggle without a rebuild.

## Localization (an ICU-message i18n library)

**Every user-facing string must be localized.** Portals ship in different primary languages,
so nothing is hard-coded.

```tsx
import { Trans, t } from 'i18n/macro';

<Trans>Hello, {name}!</Trans>                    // in JSX
const placeholder = i18n._(t`Enter your name`);   // in JS/attributes (useI18n() for i18n)
const msg = t(context.i18n)`Access denied.`;      // in route/server context
```

- **`<Trans>` in JSX, `t` for strings/attributes** (placeholders, `aria-label`).
- **CRITICAL: run the i18n message-extraction command before committing** translation
  changes — it scans code and updates the catalogs; production builds fail on strings missing
  from the catalog. Run it whenever you add, move, or edit translatable strings.
- For content needing all languages at once, use `translate(catalog, ...)` from the i18n
  translate helper.
- **Why:** debug builds fall back to the English source string, but production builds only
  render strings present in the catalog — so an un-extracted string renders broken in prod.
