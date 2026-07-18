# API Design, DRF Views & Request Lifecycle

## Framework by service
- **DRF `APIView`:** the multi-brand platform, the recommendation service, the analytics/ingestion service.
- **Django Ninja:** the content-moderation service, the traffic-analysis service (bearer API-key auth, OpenAPI schemas).

## RESTful, `/api/`, camelCase everywhere

- Build **RESTful** APIs (`POST /api/record`, not `/api/updateRecord`); deviate only for multi-resource or complex operations.
- **Request paths start with `/api/`.** Don't add new top-level URL folders (`/myendpoint`) — routing rules across portals are already complex and each new folder makes releases costlier. Nest under `/api/`, `/assets`, `/static`.
- **camelCase everywhere:** request paths (`/api/myRecords`, `/api/attachments` — not kebab-case), query params, request body keys, response body keys. Abbreviations keep capitals in camelCase JSON (`externalID`, `imageURL`, `recordID`).
- **JSON responses**, using DRF's `Request`/`Response` (not `django.http`).

## View structure

Place views in `core/views/<name>.py` (no `_view` suffix); wire the path in `core/urls.py`. Inherit `APIView`, set `permission_classes`, keep the body thin.

```python
import structlog
from rest_framework.views import APIView
from rest_framework.response import Response
from django.db import transaction
from core.authentication import RecordOwner, OrganizationOperator
from config import factory

LOGGER = structlog.getLogger(__name__)

class MyView(APIView):
    permission_classes = [RecordOwner | OrganizationOperator]     # | (OR), & (AND), ~ (NOT)

    @transaction.atomic
    def post(self, request):
        scrubber = factory().data_scrubbers.record(request.data)
        data = scrubber.scrub()
        if scrubber.get_errors():
            return Response(scrubber.get_errors(), status=400)
        ...
        return Response({"id": result.id}, status=201)
```

**Views orchestrate only** — validation in a `DataScrubber` (or DRF serializer for query params), output via `ObjectSerializer`, never manual response dicts. See `serializers-and-scrubbers.md`.

## Query-parameter validation

Never read `request.GET` raw. Declare `serializer_class` and validate:
```python
serializer = self.serializer_class(data=request.GET)
serializer.is_valid(raise_exception=True)
data = serializer.validated_data
```
Enforces types, choices, defaults, `min/max_value`, custom validators, bounded page sizes. Multi-value params use `CommaSeparatedListField`. (Ninja/pydantic services use typed schemas.) See `caching-and-search.md` for pagination/filtering.

## Permissions & identity

Default **private/authenticated**; make an endpoint public only when it truly must be. Compose permission classes with `|`/`&`/`~`. Identify the caller via `request.principal` (Django user = API key/third-party; Keycloak user = website user). For dual third-party/user access, require the caller to pass the user ID and verify it matches the token. Full detail in `auth-and-keycloak.md`.

## Responses & errors

- **Standard response:** `Response({...}, status=...)`. For pre-rendered/cached JSON use `RawJsonResponse(str(rendered.rendered_value))` to skip re-serialization.
- **Errors:** raise DRF exceptions or `AppAPIException(message=..., code=..., status_code=..., data={})`. A custom exception handler (`drf_error_view`) returns `{code, message, requestID}` uniformly, flattens nested validation errors, injects `X-Request-ID`, and hides tracebacks from non-superusers. See `architecture.md` for the exception hierarchy.

## Caching

Cache read endpoints in `urls.py` via the project `cache_page` wrapper (DEBUG-aware), not in the view — so the whole cache policy is visible in one place. Most endpoints tolerate a few minutes' cache; balance TTL against query-param cache-key cardinality. See `caching-and-search.md`.

## Logging

**Views should not routinely log** — only in exceptional cases to track a specific error. Use `create_request_logger(request, __name__, view="...")` when you do. See `logging-and-observability.md`.

## Backwards compatibility

A separate mobile/third-party team consumes these endpoints. Changes to existing endpoints/serializers must be backwards compatible; confirm breaking changes with the dev first.

## Ninja services

`NinjaAPI(auth=[HTTPBearerAPIKeyAuth(), DjangoSessionCookieAuth()])`; typed endpoint signatures / `Schema` for validation; `@paginate` for lists; custom OpenAPI schema class. Type view params: `def get(self, request: HttpRequest, ...) -> HttpResponse`. Use `TemplateResponse` over `render()` for testability. The traffic-analysis service is fully async (ASGI, async middleware).
