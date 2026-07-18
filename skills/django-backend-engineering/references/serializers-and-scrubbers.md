# Serializers, Scrubbers & the Scrub→Upsert→Publish Pipeline

This is the heart of the multi-brand platform's data flow:

```
External input → DataScrubber (validate + transform) → Upsert to DB → Publish (serialize → index + cache)
```

| Stage | Module | Factory key | Job |
|-------|--------|-------------|-----|
| Scrub | `scrubbers/` | `factory().data_scrubbers.*` | validate & normalize input |
| Upsert | `upserting/` | direct | write to DB (native upsert, in a transaction) |
| Serialize | `serializers/` | `factory().serializers.*` | shape output for API / index / cache |
| Publish | `publishing/` | `factory().prerendered_stores.*` | index to search + cache rendered JSON |

## Two kinds of serializer — know which

**1. `ObjectSerializer`** (`serializers/basic/serializer.py`) — the custom **output** serializer for API responses AND search indexing AND caching. **Not DRF.** It describes output as a field map `{"outputKey": "source_field"}` in `for_api()` / `for_indexing()` / `for_caching()`. For each field it looks for a `serialize_<source_field>(self, output, field_name)` method; otherwise it copies the raw attribute. `self.obj` is the object, `self._context` the shared context.

```python
class RecordSerializer(ObjectSerializer):
    def for_api(self) -> dict:
        return self.serialize({"id": "external_id", "title": "title", "price": "price"})

    def serialize_price(self, output, field_name):
        output[field_name] = float(self.obj.price) if self.obj.price else None
```

**2. DRF `Serializer`** (`serializers/serializers.py`) — standard DRF, used **only for validating request bodies and query params** (registered under `view_serializers`); `create`/`update` are intentionally stubbed. Includes a localized-value serializer field and `CommaSeparatedListField`.

## The config-factory registry (never import serializers directly)

Serializers are registered by key in `COMPONENT_REGISTRY["serializers"]` and resolved via `factory().serializers.<key>` — which returns whatever the **current portal** configured. Portal subclasses live in `serializers/portal/` (`PortalARecordSerializer` → `PortalARegion2RecordSerializer`), wired in `common_overrides.py`, and call `super().for_api()` / `super().serialize_<field>()` then add/replace fields. **A direct `from ...serializers import XSerializer` silently bypasses the portal override.** A new base serializer must be registered in `common.py`.

## Batch DB work on the context — the #1 serializer mistake

Serializers run over whole collections, so any DB access **per object** = one query per item (N+1). Do it once on a **context class** and expose a per-item accessor. An `ObjectSerializer` subclass declares `context_class` + a `create_context(all_objs)` classmethod; the caller builds one context for the whole list and shares it across every per-object serializer.

```python
# BAD — cached_property on the per-object serializer → one query per item
class CategoryPageSerializer(ObjectSerializer):
    @cached_property
    def _child_region_pages(self):
        return {... CategoryPage.objects.filter(region__in=self.obj.region.get_descendants()) ...}

# GOOD — one bulk query on the context, indexed by id
class CategoryPageSerializerContext:
    def __init__(self, pages): self.pages = pages
    def child_region_pages(self, page): return self._prefetched.get(page.id, {})
    @cached_property
    def _prefetched(self):  # single query covering self.pages, grouped by id
        ...

class CategoryPageSerializer(ObjectSerializer):
    context_class = CategoryPageSerializerContext
    @classmethod
    def create_context(cls, pages): return cls.context_class(pages)
    def serialize_child_region_pages(self, output, field_name):
        output[field_name] = self._context.child_region_pages(self.obj)
```

Look for `_prefetched_*` accessors on the context class and a `RecordSerializerContext` alongside the record serializer. Cheap to write wrong, expensive in production.

## Localized fields

Go through `ObjectSerializer._serialize_localized_field(...)` (with default-language fallback) or `_serialize_localized_field_without_fallback(...)` — they emit the base key plus `_l1..._lN` suffixed keys. Don't hand-roll localized output.

## Search indexing reuses the serializer

The index layer calls the entity's `for_indexing()` via the factory — it does **not** build documents itself. Adding a searchable field means editing `for_indexing()` **and** the index mapping (see `caching-and-search.md`), never ad-hoc dict-building in the indexer.

## DataScrubbers — all request/input validation

`scrubbers/data_scrubber.py` (base). **Views never validate; scrubbers do.** A scrubber validates+transforms input, collecting errors/warnings rather than raising per-field.

```python
class MyRecordScrubber(DataScrubber):
    def scrub(self) -> dict | None:
        output = {}
        self._validate_localized_field(output, "title", self.obj.get("title"), clean_function=clean_html_text)
        self._validate_decimal_field(output, "price", self.obj.get("price"))
        if self.get_errors():
            return None
        return output
```

Helpers: `_validate_type_and_value`, `_validate_localized_field`, `_validate_decimal_field`, `_validate_geography_and_get_point`, `_validate_phones_format`, `_validate_and_get_list_field`. Errors via `add_message(level=ProcessingStatus.ERROR, field=..., message=...)`, read via `get_errors()`/`get_warnings()`. Nested scrubbers bubble messages with `_bubble_messages(child, "field")`. Resolve via `factory().data_scrubbers.record(data)`.

## The complete view pattern

```python
class CreateRecordView(APIView):
    def post(self, request):
        scrubber = factory().data_scrubbers.record(request.data)      # 1. validate + transform
        data = scrubber.scrub()
        if scrubber.get_errors():
            return Response(scrubber.get_errors(), status=400)
        record = upsert_record_to_db(data)                            # 2. business logic / DB
        return Response(factory().serializers.record(record).for_api(), status=201)  # 3. format output
```

**Views orchestrate. Scrubbers validate. Serializers format. Everything overridable goes through `factory()`.** Never construct response dicts by hand.

## Upsert flow

`scrub_record(raw) → ScrubResult(data, errors, warnings)` → `upsert_record_to_db(data)` (`@transaction.atomic`, native upsert + related objects) → `publish_record.delay(record.id, trigger=...)`. `upsert_record.delay(raw, trigger=...)` does all three async. Publishing serializes → indexes to the search backend → caches rendered JSON in `RenderedRecord` → submits to moderation / triggers webhooks. Related data (attachments, `region_id` by external id) is handled inside the upsert.

## Sync — diff-based external synchronization

`sync/` diff classes (`RecordDiff`, `MemberDiff`, …) fetch existing records, compare incoming data by **hash** (`hashing.py`), process only changed records, and optionally deactivate missing ones — so a daily full sync only upserts what actually changed. Portal-overridable via `factory().sync.*`.

## Backwards compatibility (hard constraint)

A separate mobile-app / third-party team consumes these payloads. Keep serializer output and endpoints backwards compatible — additive changes only on existing fields/shapes; confirm any breaking change with the owning dev before shipping.
