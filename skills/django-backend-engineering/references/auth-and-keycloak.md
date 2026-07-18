# Authentication, Authorization & Keycloak

There are **two auth stacks**. Identify which the service uses first:

| Stack | Where | Identity | Credentials |
|-------|-------|----------|-------------|
| **DRF + `Principal`** | the multi-brand platform (shared core) | `request.principal` (Keycloak / Django / anonymous) | Keycloak JWT, Django API token, TAP, CRM dual-identity |
| **django-ninja + API key** | the content-moderation & traffic-analysis services | Django user OR named API key | Bearer API key, Django session |
| **Simpler DRF baseline** | the analytics/ingestion & recommendation services | Django user | `TokenAuthentication` / `BasicAuthentication`, `IsAuthenticated` default |

All services share an **in-house SSO wrapper library** (Google OAuth) for admin/human login — never API auth.

---

## The `Principal` abstraction (the central standard)

`core/authentication/principal.py`

- **What:** a `@dataclass Principal{type, id, roles}` with `PrincipalType ∈ {ANONYMOUS_USER, DJANGO_USER, KEYCLOAK_USER}`. Every request carries `request.principal`. Read identity from **`request.principal`, never `request.user`** — Keycloak auth classes deliberately return `AnonymousUser()` as `request.user`; the real identity lives on `request.principal`.
- **Why:** DRF's `request.user` conflates an internal Django user (staff/service with a token) and an external website end-user (Keycloak). Splitting identity lets permissions ask semantic questions (`is_portal_user()`, `is_django_user()`, `has_role()`) instead of `isinstance` checks.
- **Where set:** every authentication class sets it; `SetDefaultPrincipalMiddleware` (`authentication/middlewares.py`) stamps an anonymous `Principal` on every request *before* auth, so `request.principal` always exists (no `AttributeError` on unauthenticated requests).
- **When:** any view/permission/serializer that needs "who is calling".
- **Bad alternative:** overloading `request.user` with a fake user or sentinel and brittle attribute checks everywhere.

```python
def is_portal_user(self, external_id: str | None = None) -> bool:
    if self.type != PrincipalType.KEYCLOAK_USER:
        return False
    return True if not external_id else self.id == external_id
```

## Keycloak token validation

`authentication/keycloak.py`, `authentication/token.py`, wired in `config/settings/common.py`

- **What:** Keycloak access tokens are JWTs verified **locally with Keycloak's public key** (no introspection round-trip). Two classes: `KeycloakUserPrincipalTokenAuth` (reads `Authorization: Bearer`) and `KeycloakUserPrincipalCookieAuth` (reads the `access_token` cookie). User id comes from a custom `external_id` claim; roles from `realm_roles`. Token type is pinned to `Bearer` (`KeycloakAccessToken`), rejecting refresh/ID tokens. `MultiKeyTokenBackend` supports RS256 *and* ES256 via `KEYCLOAK_PUBLIC_KEYS_BY_ALGORITHM`, verifying with the key matching the token's `alg` header.
- **Why local verification:** avoids a Keycloak call per request (latency + hard dependency on the hot path).
- **Key convention — auth fails *open*:** both classes catch token errors and **return `None`, never raise**. A stale cookie shouldn't hard-fail *public* endpoints; returning `None` means "unauthenticated", so public views still work and protected views 403 via their permission class. **Authentication fails open; permissions fail closed.**
- **When:** the three DEFAULT auth classes (Keycloak-header → Keycloak-cookie → Django-token) cover everything — **don't set `authentication_classes` on views**; only choose `permission_classes`.
- **Impersonation:** tokens may carry `impersonator_id`/`impersonation_id`; the auth class populates `request.impersonator_principal` and `request.impersonation_session_id`.

## Django API-token auth (internal / service callers)

`authentication/django.py` — `DjangoUserPrincipalTokenAuth(TokenAuthentication)` handles `Authorization: Token <key>`, always tied to a real Django user → `Principal(DJANGO_USER)`. `authenticate_credentials` is wrapped in a **1-hour `TTLCache`** so a hot master token isn't a DB hit per request (tradeoff: up to 1h staleness after revocation). Django-user principals are privileged — several permissions let them act on behalf of any user.

## Permission classes and composition

`authentication/permissions.py`

- **What:** the authorization vocabulary — `IsAuthenticatedPortalUser`, `IsSelfPortalUser` (URL kwarg must match principal id), `RecordOwner`, `OrganizationOperator`, `IsPremiumMember`, `IsReadRequest` (GET only), `IsNotDeletedUser`, `IsInSamePrivateNetwork`, `IsLocalhost`, `has_view_permission()` / `IsTrustedThirdParty`.
- **Composition standard:** combine with DRF operators — `|` (OR), `&` (AND), `~` (NOT); a tuple/list is implicit AND. This is the primary authorization language; **never write auth `if` ladders inside views** — keeping it at the class attribute makes it introspectable (a release-time scanner reads it).
- **Recurring idioms to reuse:**
  - Public read, protected write: `[has_view_permission() | IsReadRequest]`
  - Self-service + service override: `[IsSelfPortalUser | IsTrustedThirdParty]`
  - Self OR service OR in-cluster cron: `[... | IsLocalhost]`
  - Subtract a capability: `[RecordOwner, ~IsPremiumMember]`

```python
# example declarations
permission_classes = [IsTrustedThirdParty | (IsAuthenticatedPortalUser & IsSelfPortalUser)]
permission_classes = (IsSelfPortalUser | IsTrustedThirdParty, IsReadRequest | ~IsPremiumMember)
```

## Third-party vs website users — the API design rule

APIs should be usable by **both** third-party apps (API key) and website users (Keycloak). **Require the caller to pass the user ID** (query param / body) even though it's in the token — this lets a third-party app (not bound to a user) call the same endpoint — and then **always verify the passed user ID matches the authenticated user** (see `IsSelfPortalUser`). Default to private/authenticated endpoints; only make an endpoint public when it truly must be.

## Network-origin auth (service-to-service)

`IsLocalhost` (loopback/link-local) and `IsInSamePrivateNetwork` (RFC-1918) gate on caller IP read from `X-Real-IP` then `REMOTE_ADDR`. **Why:** in-cluster pods / cron have no user token; the trust boundary is the network. **Tradeoff:** trusts `X-Real-IP`, so only safe behind an ingress that sets it authoritatively. Use for internal-only endpoints, or OR with `has_view_permission()` for dual public/internal use.

## Fine-grained per-endpoint permissions

`has_view_permission(allow=[...])` returns a `BaseViewPermission` subclass requiring a `DJANGO_USER` with a Django permission `"<app>.view-permission-<url_name>-<method>"`. `apply_view_permissions()` walks the URLconf on every release and bulk-upserts the matching `Permission` rows. **Why:** Django's built-in permissions are model-level, but endpoints run tasks touching arbitrary models — the team makes **endpoint+method the unit of authorization**, grantable per Django user. It's the DEFAULT permission, so an endpoint with no explicit `permission_classes` requires an explicit grant. **Constraints enforced by the scanner:** every protected URLPattern must have a unique `name`, and at most one view-permission class per view.

## Temporary Access Pass (TAP)

`authentication/tap.py` — a self-issued signed JWT granting access to a **specific URL list** for a **limited TTL**, scoped by `audience`. `create_access_pass(urls, audience, ttl)` mints it; `create_access_pass_permission(...)` validates `?tap=`. URLs are normalized (sorted query params) so ordering doesn't matter. **When:** one-off scoped external access (a signed report link) where a real account or master token would be over-provisioned.

## CRM dual-identity (partner integrations)

`integration/authentication.py` — `IntegrationTokenAuthentication` resolves two identities per request: an *integration account* (organization via `Auth-Token`) and an *acting user* (via `X-External-User-Id`, must be an active member of that organization). On success it sets `request.principal` to the acting user as a `KEYCLOAK_USER`, so downstream permissions (`RecordOwner`, …) work unchanged. **When:** partner integrations where one credential fronts many end-users.

## django-ninja API-key auth

`httpapi/auth.py` — two auth schemes tried in order:
- `HTTPBearerAPIKeyAuth(HttpBearer)` — machine callers. Keys are a **name→key dict from settings** (`APP_API_KEYS`, `REST_API_KEYS`), inverted to key→name; the matched **name** is stamped as `request.user.username = "api_key:<name>"` for log attribution. Sets `request._dont_enforce_csrf_checks = True`.
- `DjangoSessionCookieAuth(SessionAuth)` — the website/staff user.

`NinjaAPI(auth=[HTTPBearerAPIKeyAuth(), DjangoSessionCookieAuth()])` = "valid API key OR logged-in session". **API-key strength is enforced at deploy** via a Django system check (`@register(deploy=True)`, min length 15 = a UUIDv4). **Tradeoff:** static named keys are fast but can't be revoked without a redeploy — mitigated by per-key naming (identify a leaked key) and the length check.

## SSO / Google OAuth admin login (in-house SSO wrapper library)

A thin wrapper over `django-allauth` for **staff/admin login only**. `configure_auth_settings(globals(), env)` injects Google provider config + backends. `ConnectSocialAccountAdapter` enforces: an **email-domain allowlist** (only corporate domains), auto-`is_staff` only for the corporate domain, auto-links Google login to an existing user and removes the password (`set_unusable_password`). **Why:** one shared, audited definition of who gets into the admin, tied to corporate Google accounts, so offboarding is centralized. Integrate in 3 steps: add dep, call `configure_auth_settings`, include the library's `urls`. **Never hand-roll admin auth.**

---

## Rules to codify

1. Read identity from `request.principal`, never `request.user` (DRF stack).
2. Authentication is global (3 DEFAULT classes); authorization is per-view. Don't set `authentication_classes`.
3. Authentication fails open (returns `None`); permissions fail closed (produce 403).
4. Compose permissions with `|`/`&`/`~`; reuse the named vocabulary; no auth `if` ladders in views.
5. Match the credential to the caller: website user → Keycloak JWT; trusted service → Django token + `has_view_permission()`; in-cluster → `IsLocalhost`/`IsInSamePrivateNetwork`; one-off external → TAP; partner → CRM dual-identity; ninja service → named bearer key.
6. Private by default; require + verify a passed user ID so third parties and users share endpoints.
7. Verify Keycloak JWTs locally with public key(s); support multiple algorithms; pin token type `Bearer`.
8. Enforce API-key strength with a deploy-time system check; name each key for log attribution.
9. Delegate admin login to the in-house SSO wrapper with the shared domain allowlist.
