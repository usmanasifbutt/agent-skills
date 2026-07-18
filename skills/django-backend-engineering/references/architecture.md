# Architecture, App Organization & Reusable Abstractions

## Two architectural lineages — pick by the kind of service

| Lineage | Use for | Signature |
|---------|---------|-----------|
| **Multi-brand platform** | a new **multi-tenant / multi-brand** product | one shared core + thin per-portal apps, DRF, a config-factory registry, per-portal setting overrides, `ObjectSerializer`/`DataScrubber` |
| **Layered / clean** | analytics/ingestion, embeddable domain libraries, focused services | explicit layers (`repository/ query/ data/`), decorator/registry DI, Django-Ninja or DRF, module-per-concern |

Both share the foundations (a Postgres-aware base model, structlog, an in-house Celery task wrapper, a timeout-aware migration base class, per-env settings, a base-exception + custom handler).

## Project layout

**Multi-brand platform:** one giant reusable app (the shared core) holds all business logic, subdivided into dozens of **concern packages** (`records/ organization/ search/ serializers/ scrubbers/ upserting/ sync/ publishing/ tasks/ tools/ views/ models/ managers/ query/ middleware/ authentication/`). Each deployable portal (`backend/<app>/`) is a thin Django project importing the shared core and overriding pieces. The Django "project" lives under `config/` (settings, urls, wsgi, factory, portal). Organization is **by concern (folder), not by Django app** — essentially one app. **Tradeoff:** a hard-to-navigate god-app, accepted to avoid per-portal duplication/drift across many portals.

**Layered:** single app split into named layers — `views/ serialization/ repository/ query/ data/ (DTOs) db/ core/ middleware/ ingestion/ indexing/ delivery/ net/`. **Monorepo of packages:** `packages/{taxonomies,client,app,server,mockportal}` — `taxonomies` = shared enums/DTOs (no logic, pinned to an older Python/Django because it ships to consumers), `app` = embeddable core, `server` = one deployment of it. **Classic multi-app:** conventional apps (`aggregator/ recommender/ tracker/`; `analysis/ worker_configuration/`).

**Universal sub-layout:** within an app, split into plural concern-dirs (`views/ serializers/|serialization/ models/ tasks/ tools/|utils/ middleware/ management/`). View files are named after the resource with **no `_view` suffix** (`views/records.py`).

## Service layer

There is **no classic `services/` layer.** Business logic lives in dedicated concern modules invoked from views:
- **Multi-brand platform:** thin-ish `APIView`s (`@classmethod` handlers that parse a request into a **params dataclass** then delegate) + free functions + registered processor/scrubber/serializer classes in `scrubbers/ upserting/ sync/ scoring/ publishing/`.
- **Analytics/ingestion service:** the **repository layer is the service layer**.
- **Content-moderation service:** the **subroutine registry** is the pluggable-logic layer.
- **Recommendation service:** strategy classes (`recommenders/`, `recommender_builder.py`) + queued tasks.

Keep views thin by extracting a request-params dataclass and pushing DB/formatting work into concern modules; **don't** build a formal Service class hierarchy. Registries provide the discoverability a service layer would.

## The config factory + per-portal overrides (the multi-brand dependency injection)

`config/factory.py`, registry in `config/settings/common.py` (`COMPONENT_REGISTRY`)

- **What:** a settings dict `COMPONENT_REGISTRY` maps string keys → dotted class paths. `@functools.cache def factory()` turns it into a nested namedtuple tree of **imported classes**. Code resolves implementations via `factory().serializers.record`, `factory().data_scrubbers.record`, `factory().search_indexes.record`, etc. Portals swap the string path in their `common_overrides.py`.
- **Why:** one core, many portals — each must substitute serializers/indexes/scrubbers/celery-app without editing core imports. **A direct `import XSerializer` silently bypasses the override and uses the base class.**
- **Registry keys:** `serializers`, `view_serializers`, `search_indexes`, `prerendered_stores`, `sync`, `data_scrubbers`, `celery_app`, `staticdata`, `product_scorer`, `translator`, …
- **Rule:** resolve overridable classes only via `factory().<group>.<key>`.
- **Tradeoff:** loss of static go-to-definition / IDE types vs. clean per-portal substitution from one lazy cached source.

```python
# portal override
settings["COMPONENT_REGISTRY"]["serializers"]["record"] = "core.serializers.portal.PortalARecordSerializer"
settings["COMPONENT_REGISTRY"]["celery_app"] = "portal_a.celery.app"
```

## Multi-portal safety (the first question on every change)

Before editing the shared core, ask **"does this apply to all portals or some?"** — shared code affects many portals spanning multiple countries and brands. Portals differ in categories, fields, languages, payment providers, moderation. Three rollout mechanisms, in order of preference:
1. **Feature flag** (recommended) — `APP_ENABLE_X = env.bool(..., False)`, enabled per-portal in `common_overrides.py`.
2. **Factory override** — register a base class, override the path per portal.
3. **`Portal.current()` check** (sparingly) — `if Portal.current() in [Portal.BRAND_A, Portal.BRAND_A_REGION2]:`. Never `if portal_name == "brand_a":` (string check).

Backward-compatible changes only: new fields optional with defaults; don't assume a field exists on all portals; never hard-code language codes. Test on at least two different portals.

## DTO / dataclass patterns

Three idioms, chosen by purpose:
- **`@dataclass` for request params & internal value objects** — `RecordViewParams` and many similar dataclass files; enums via `StrEnum`/`IntEnum`. Cheap, memory-efficient, type-safe containers — prefer over loose dicts.
- **Pydantic `BaseModel` for validated/serialized DTOs** — `SubroutineArgs/Context/Result`, typed object access `RecordDataDTO`. Use when you need validation/serialization or handle external input. Rule: **access object data through `wrap_object_data(obj.type, obj.data)`, never `obj.data` directly** — a typed, validated façade over a JSON blob.
- **`@dataclass`/Pydantic + metaclass meta-config** — `IngestionData` whose metaclass reads a nested `class IngestionMeta` to attach `model_type`/`rate_limit`, so a repository can resolve the target model from the DTO alone.

## Registries (dependency injection without a container)

No DI framework. Injection is via: (a) the **config factory**, (b) **decorator registries** — a `SubroutineRegistry.register(...)` stores callables by name with a Pydantic `args_class`, a Django form, and an output template so admins pick logic from a discoverable catalog with a uniform `(context, args) -> SubroutineResult` signature, (c) **constructor injection** into repositories/strategies, (d) a repository `__init__.py` re-exporting dozens of repos as one import surface.

## Settings module layout

Base module (`common.py`/`base.py`) holds all defaults via `env.str/bool/dict` reads; per-env modules (`production.py`/`local.py`/`test.py`) do `from ...common import *` then layer env-specifics. **No per-env `if/else` inside one file.** The multi-brand platform additionally layers a per-portal `override_settings(globals(), env)` function that mutates the imported globals.

```python
# backend/<app>/.../settings/production.py
from config.settings.production import *
from .common_overrides import override_settings
override_settings(globals(), env)
```

Settings rules: **explicit** — define every setting in `common.py`; **never `getattr(settings, "X", default)`** (hides typos/config drift). Prefix `APP_` for platform features, the vendor name for integrations. Don't add settings for constants that never change per env — declare a module-level constant. Prefer long, verbose, prefixed names (don't assume "there's only one" of something).

## Middleware & request lifecycle

- **Request-context setup middleware goes first; error-handling / DB-routing middleware goes last.** `SetDefaultPrincipalMiddleware` is first (stamps anonymous `request.principal` before auth), `DatabaseRouterMiddleware` is last (so it only wraps view-layer 5xx, not other middleware). Give middleware a common base where you have several (a base `Middleware`).
- **Lifecycle:** `SetDefaultPrincipalMiddleware` → DRF auth classes → default permission `BaseViewPermission` → view → custom exception handler → `DatabaseRouterMiddleware` (replica vs primary). See `auth-and-keycloak.md` and `queries-database.md`.

## Exception handling & framework error handlers

Standard: **one app-base exception class per service; specific typed subclasses carrying HTTP status; a custom framework error handler returning machine-readable `code`+`message`, injecting a request id, never leaking tracebacks to non-superusers.**
- **Multi-brand platform:** `AppAPIException(message, code, status_code, data)` + typed subclasses (`ResourceConflict(409)`, `PreconditionFailed(412)`, `ResourceLocked(423)`, `BadGateway(502)`, …). A DRF handler `drf_error_view` always emits `{code, message, requestID}`, flattens nested validation errors, hides tracebacks unless superuser/DEBUG.
- **Analytics/ingestion service:** base `ValidationError(RuntimeError)`; `SerializationValidationError` sanitizes PII keys out of Pydantic error output; `ErrorHandlingMiddleware` maps `(TypeError, ValueError, ValidationError, QueryHandleError)` → 400.
- **Content-moderation service:** a base error hierarchy; let exceptions propagate unless meaningfully handleable; tracebacks gated to superusers/DEBUG.
- Python-level rule: **don't catch `Exception`** (pokemon handler that also swallows `SyntaxError`); catch specific errors or `RuntimeError` for a deliberate catch-all. Name new exceptions `...Error` (not `...Exception`) and inherit `RuntimeError`.

## Feature flags

- **Settings env flags with linter-enforced expiry:** `APP_ENABLE_* = env.bool(...)`, default `False`, enabled per-portal. Every env var must be declared in a central config manifest (`config/main.yml`); an env linter writes an expiry comment for rollout flags so stale flags get flagged for removal. Enable per portal, then delete the flag and its checks once fully rolled out.
- **Enum registry:** `Feature(str, Enum)` with `.enabled` reading `settings.APP_FEATURES` (dict), `Feature.defaults()` fallback; `if Feature.MULTI_PROJECT.enabled:`.
- **`DebugFlag` enum** for debug toggles. **Domain env knobs** for tunable weights (`FEATURE_SCALING`, weights).
- **Standard:** explicit, discoverable, removable, time-boxed flags; declare in the central config manifest, never scatter `os.environ` reads.
