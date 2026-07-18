# Authentication, Keycloak, RBAC, Protected Routes, API Architecture, Errors, Loading

## Authentication & Keycloak

**No client-side `keycloak-js` adapter, no OIDC redirect dance in the SPA.** Auth is a Redux
auth SDK (thunks + selectors) on the client plus an **Express BFF that owns HttpOnly token
cookies**. Email/phone/OTP/social flows are driven by SDK thunks and rendered by React dialogs.

- **Config gate** (`core/auth/keycloak/config.ts`): `isKeycloakEnabled()` is true only when
  realm/url/clientID/publicKey runtime config are all present; otherwise the app falls back to
  legacy Express-session login (`core/auth/express`).
- **Cookie bridge:** the client never writes token cookies. It POSTs the session to the BFF
  (`InternalAPI.saveKeycloakSession()` → `POST /api/keycloak/session`), which sets
  `access_token` / `id_token` HttpOnly cookies (`GET`/`POST`/`DELETE /api/keycloak/session`
  in the base server app, wrapping the Redux auth SDK's Express helpers).
- **`ServerAuthContext`** (`core/server/serverAuthContext.ts`) — a request-bound, **server-only**
  static class (backed by `express-http-context`) holding the per-request access/ID token,
  cookies, request-id, and lazily-decoded identity (externalID/email/roles from `realm_roles`).
  It builds `asBackendHeaders()` (forwards `Authorization: Bearer <token>` to the backend API)
  and `asFrontendHeaders()` (forwards cookies to the Node BFF). It's wrapped in
  `if (process.env.IS_SERVER)` so it tree-shakes out of the browser bundle.
- **Token refresh** (`keycloakSessionRefreshTimer.ts`): a client timer refreshes ~60s before
  the earliest token `exp`, with exponential backoff (max 5 attempts) and a `visibilitychange`
  listener; uses a **drift-corrected clock** (`core/clock`), not `Date.now()`.

**Why this shape:** tokens live in server-set HttpOnly cookies (readable at SSR, invisible to
JS), so credentials never leak into the serialized Redux store (`window.state`) or JS storage,
and refresh is centralized. **Never** put tokens in Redux/localStorage or add a browser
Keycloak adapter.

## Current user, login/logout

- **`selectIsUserLoggedIn`** (`core/user/selectors.ts`) bridges two slices: the legacy global
  `user` slice and the Keycloak session slice (`core/user/session/selectors.ts`:
  `selectActiveUser`, `selectUserRoles`, `selectUserExternalID`).
- **`useLogout()`** (`core/auth/useLogout.ts`) branches on `isKeycloakEnabled()` — dispatches
  the SDK `logout(siteConfig)` or `expressLogout()`; redirects home if on a restricted route.

## Role-based access control

- A flat **`UserRole` string enum** + **pure predicate helpers** over `readonly UserRole[]`
  (`core/user/roles.ts`): `isMember`, `isOrganizationOwner`, `isPremiumMember`,
  `isPaidMember`, … Roles map to Keycloak `realm_roles`.
- **Gating is per-component/per-route**, not a decorator: read roles via selectors
  (`selectUserRoles`, `selectMemberProfile` — returns a profile only if paid on the current
  platform) and role **hooks** (`useIsPremiumMember`, `useIsPaidMember`, …), or
  `ServerAuthContext.userRoles()` server-side.

```ts
export const isMember = (roles) =>
    roles.includes(UserRole.MEMBER) || isPremiumMember(roles) || isPaidMember(roles);
```

## Protected routes

There is **no declarative `<ProtectedRoute>`** — auth/role gating is **imperative inside
`onEnter`** (the custom router; see `routing-config-i18n.md`):

- **Redirect authed users away from login** (`loginRoute.ts`): if `selectIsUserLoggedIn(...)`,
  `redirectToURISafely(context, ...)`; else render `Page.LOGIN` / `Page.EXPRESS_LOGIN`.
- **Auth-aware content:** e.g. favorites renders empty results when logged out rather than hard
  -blocking.
- **Denials:** `renderNotFoundPage(context)` for 404 (and access-denied surfaces as an
  `APIError` with `reason: ACCESS_DENIED`); `context.http.status(...)` / `redirect(...)` for
  SSR status. `restrictedRouteNames` (`routeNames.ts`) drives redirect-home-on-logout.
- **`CancelableRoute`** tracks in-flight promises via `this.cancelable(promise)` and **cancels
  them all in `onLeave`** — the mechanism that prevents stale fetches/state updates after
  navigation.

## API architecture

- **Transport is `fetch`** (polyfilled `whatwg-fetch`) — **no axios, no direct XMLHttpRequest**.
  Timeouts via `AbortSignal` (`withTimeoutSignal`); server attaches keep-alive http/https
  agents.
- **One `HTTPApi` base class** (`core/util/httpApi.ts`): `request`/`requestBinary`/
  `requestStream`/`post`/`postJSON`/`patch`/`put`; sets `credentials`, CSRF (`X-CSRFToken` from
  cookie), `Accept`. **`authHeader()` is the override point** — subclass per target, never
  touch the transport:

  | Class | Talks to | `authHeader()` |
  |-------|----------|----------------|
  | `BackendAPI` (`api/backendApi.ts`) | the primary backend | `ServerAuthContext.asBackendHeaders()` (Bearer) on server |
  | `InternalAPI` (`api/internal.ts`) | Node BFF (`/api/*`) | `asFrontendHeaders({ withCookies: true })` |
  | legacy form-data API | an older backend | form-data; mostly deprecated |

  (**Never call `fetch()` directly** — always go through an `HTTPApi` subclass in `core/api`;
  create a new one deriving from `HTTPApi` if none fits.)
- **No generic HTTP retry** in `HTTPApi` (retry exists only for auth refresh + per-API rate
  limiters). Cancellation is at the routing layer (`CancelableRoute`) and request layer
  (`AbortSignal`).
- `BackendAPI` prefixes the absolute backend URL only on the server (client uses relative), and
  repeats query keys for array params (a common backend convention).

## Error handling

- **Typed `APIError extends ContextfulError`** (`core/api/error.ts`) with a machine-readable
  **`APIErrorReason` enum** (`ACCESS_DENIED`, `UNAUTHORIZED`, `NOT_FOUND`, `ALREADY_EXISTS`,
  `TOO_MANY_REQUESTS`, `UNPROCESSABLE_ENTITY`, `PRECONDITION_FAILED`, …) and bidirectional
  status↔reason mapping (`reasonFromStatusCode`, `statusCode()`). `InternalAPI.handleResponse`
  is the canonical status→error translator. Error fields are non-enumerable to avoid error-tracker
  double-extraction.
- **React error boundary:** `withErrorBoundary(Component, Fallback)` HOC
  (`core/util/errorBoundary.tsx`, class + `componentDidCatch`) — retries one render, then
  shows a fallback and logs. Code-split components are auto-wrapped by `Loadable`.
- **Central logger:** `logError({ e, msg, context })` (`core/error/log.ts`) logs to the app
  logger **and** the error-tracking service (`captureException`), normalizes non-Error throwables, walks the
  `cause` chain, and extracts `toJSON()` context. **Always report client errors through
  `logError`** — never bare `console.error`.
- **How API errors reach the UI:** the fetcher stores the `APIError` in the slice's `error`
  field; components read it and render form-level error components / toasts (there's no single
  global error toast).

## Loading states

- Read loading straight off the fetcher's `FetchState` slice
  (`{ loading, loaded, preloaded, data, error, params, enabled }`) via selectors — don't invent
  per-component loading booleans.
- **Visual:** `LoadingSpinner`/`LoadingSpinnerOverlay`/`LoadingDots` primitives and
  **hand-built skeletons** (e.g. `search/mapBasedSearch/skeletonCard.tsx`) — no third-party
  skeleton lib. Code-split boundaries show a loading spinner via `core/loadable`.
