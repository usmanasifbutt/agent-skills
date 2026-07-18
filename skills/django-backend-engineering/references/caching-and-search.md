# Caching, Redis, Search (Elasticsearch/OpenSearch/hosted search), Pagination, Filtering

Two dialects: one family of services (the multi-brand platform, the content-moderation service) uses `django-redis` + `elasticsearch_dsl`/`opensearchpy` `Document` classes; another family (the recommendation, analytics/ingestion and traffic-analysis services) hand-rolls `redis-py` connection-pool wrappers + pydantic/ninja schemas. The analytics/ingestion service shares the platform's ES index base-class design almost verbatim.

## Caching

### Named, purpose-specific caches, each on its own Redis DB index
Declare cache aliases as constants (`DEFAULT_CACHE`, `GUIDE_CACHE`, `CATEGORY_PAGE_CACHE`, `OTP_CACHE`, …) in `common.py`; build `CACHES` in `production.py` with each alias on a distinct Redis logical DB (`redis_cache_url(db_index=N)`). **Why:** one cache can be flushed or evicted under memory pressure without touching the others; key namespaces don't collide. **When:** add an alias whenever a feature has a distinct invalidation lifecycle or TTL. **Bad alternative:** a single shared `default` where a sitemap flush also nukes OTPs.

### Caches fail open; load-bearing stores fail loud
django-redis caches use `IGNORE_EXCEPTIONS: True` + sub-second `SOCKET_TIMEOUT`/`SOCKET_CONNECT_TIMEOUT` (`redis_options()`), so a Redis outage degrades to cache-miss, not 500s, and a slow Redis can't blow up latency. **Contrast:** the raw redis-py stores/locks/rate-limiters (below) raise typed errors because their data is load-bearing. TLS: only inject `ssl_cert_reqs=None` for `rediss://` URLs when insecure is explicitly allowed (managed Redis uses unverifiable self-signed certs but should still be encrypted).

### `cache_page` is wired per-URL in `urls.py`, DEBUG-aware
HTTP response caching is applied at the routing layer via a project-local `cache_page` that no-ops under `DEBUG`; TTL in seconds, non-default cache via `cache=settings.<ALIAS>`.

```python
from .cache import cache_page
re_path(..., cache_page(24 * 60 * 60, cache=settings.GUIDE_CACHE)(views.GuideView.as_view()))
```

**Why:** the whole cache policy is visible in one file, not scattered in view bodies; the DEBUG wrapper avoids stale local responses. **When:** GET responses identical across users, safe to serve stale for a bounded window (reference data 24h, contact info 1h, sitemaps 20m). **Balance** TTL against cache-key cardinality — every distinct query-param combination is a new entry. For cookie-personalized responses, cache only requests missing the cookie (bots) and bypass for real users.

### Cache keys & TTLs
Keys are deterministic, namespaced, colon-delimited (`f"{domain}:{parts}"`), with collections `sorted()` and large/opaque bodies hashed (`sha256(json.dumps(body, sort_keys=True))`). TTLs are named constants / env settings with an intent comment; TTL 0 means "disable this cache".

### In-process caches for near-static reference data
Use `functools.cache` for data that never changes (category tree) and `cachetools.TTLCache(ttl=3600)` for hourly-ish data (regions), with a custom hashable-key function. **Why:** avoid a Redis/DB round-trip for a tree that barely changes. **Not** for user-specific data or anything needing precise invalidation.

## Redis (direct, non-cache)

- **One module-global `ConnectionPool` per process, created lazily behind a lock**, then `redis.Redis(connection_pool=pool)` per op. Cap `max_connections` just below the server limit (favor throttling threads over exceeding the cap). Add `Retry(ExponentialBackoff())`, health checks, keep-alive for async pools.
- **Distributed lock** = `SET NX EX` with a **UUID token**; only delete the key if it still holds your token. Cache-fill of expensive queries uses **double-checked locking**: check cache → acquire lock → re-check cache → compute → `setex`. Prevents a cache stampede against the search cluster. **Bad alternative:** plain `SETNX` without a token (deletes someone else's lock).
- **Rate limiting**: fixed-window buckets via atomic `INCR` + expiry, with exponential suspension for repeat offenders.

## Elasticsearch / OpenSearch

### Alias-fronted, revisioned `Document` index classes
Every index is an `elasticsearch_dsl.Document` / `opensearchpy.Document` subclass. Reads/writes go through an **alias**; the physical index is `{name}-{revision}`. Zero-downtime remap: create `{name}-{rev+1}`, dual-write via the alias, reindex, flip the alias. `recreate()` is flagged "do not run during business hours". **Bad alternative:** writing to a named index → mapping changes need downtime.

### The `query` meta-field
Searchable fields are declared `copy_to="query"` (via a `SearchableKeyword` subclass); free-text search targets **only** the synthetic `query` `Text` field. **Why:** querying one analyzed field is much faster than a multi-field query and centralizes analysis.

### Static vs dynamic settings; dynamic mapping off
Split `STATIC_INDEX_SETTINGS` (shards, analysis, sort — only settable at create) from `DYNAMIC_INDEX_SETTINGS` (replicas, slowlog — settable live); deep-merge them; `setup()` sends only dynamic settings if the index exists (idempotent, safe to re-run). Core mapping is `dynamic: false`; `dynamic_templates` type open-ended `extraFields.*` (numbers→float, strings→keyword with `copy_to: query`). **Bad alternative:** sending the full settings blob to a live index (errors on immutable settings).

### Per-language indices; batch bulk indexing
One physical index per language (`{name}-{lang}`); serialize once per language under `translation.override(lang_code)`; bulk-index in batches of 500. Two connection styles: DSL `connections` for owned indices; a thin `Elasticsearch(...)` client + a generic `ESIndexDataSource` (`helpers.scan`) for reading foreign/analytics indices.

### Index documents come from the same serializers as the API
The index layer doesn't build documents — it calls the entity's `for_indexing()` obtained from the config factory (`factory().serializers.record(obj).for_indexing()`). Adding a searchable field means editing `for_indexing()` **and** the index mapping — never ad-hoc dict-building in the indexer. See `serializers-and-scrubbers.md`.

### Hosted search service (some portals)
Some portals also use a hosted search service via `factory().search_indexes.*` with `save`/`save_batch`/`delete`/`search`/`setup` (the self-hosted alternative is Elasticsearch). A fan-out index writes to the hosted service + Elasticsearch together. Publishing (async task) serializes → indexes → caches rendered JSON.

## Pagination

Every list endpoint is paginated with an explicit, bounded page size; unbounded list responses are avoided. Mechanism follows the framework:
- **DRF platform:** `APIPaginator(PageNumberPagination)` — `page_size=10`, `max_page_size=1000`, returns `{count, next, previous, results, **metadata}` with **relative** next/previous URLs (scheme/host stripped so it works behind proxies/CDN).
- **Recommendation service:** DRF global `LimitOffsetPagination`, `PAGE_SIZE=100`.
- **Analytics/ingestion service:** pydantic `QueryPagingData` with validated `offset`/`limit` and hard ceilings (protect ES/DB from deep-pagination blowups).
- **Traffic-analysis service:** django-ninja `@paginate`.

## Filtering

- **DRF platform:** `django_filters` FilterSets with a custom `InFilter`/`InField` that builds `field__in` from **repeated** params (`field=1&field=2`) via `getlist`, doing a true `IN` instead of DRF's CSV/OR default, and skipping validation so unknown choices don't 400.
- **Hierarchy filtering:** filter regions/categories by the tree library's **`lft/rght/tree_id` range** (or pre-cached descendant-id sets), never a giant `region_id__in=[thousands]` — the range predicate hits the index and is far faster.
- **Analytics/ingestion service:** DRF global `DjangoFilterBackend`. **Ninja services:** ninja query schemas.

## Query-parameter validation

**Never read `request.GET` raw.** Validate GET params declaratively:
- **DRF:** the view declares `serializer_class`, then `serializer = self.serializer_class(data=request.GET); serializer.is_valid(raise_exception=True)` and reads `serializer.validated_data`. Types, choices, defaults, `min/max_value`, custom validators, and `CommaSeparatedListField` (parses CSV *and* repeated params) are enforced declaratively — uniform 400s, coerced values, bounded inputs that protect downstream queries.
- **Ninja/pydantic services:** pydantic / ninja schemas — same principle in each framework.
- **Config/URL parsing** fails with actionable errors (`ImproperlyConfigured` stating the exact expected format), so misconfiguration surfaces at boot, not deep in a request.

## Summary rules

1. Fail-open caches (`IGNORE_EXCEPTIONS`, sub-second timeouts); fail-loud load-bearing stores.
2. Isolate concerns by Redis DB index and named cache aliases.
3. TTLs/pool limits explicit, named, env-tunable, commented.
4. Search indices are alias-fronted, revisioned `Document` classes with a `copy_to="query"` field, `dynamic:false` + `dynamic_templates`, split static/dynamic settings, per-language physical indices.
5. Index docs come from the API serializers' `for_indexing()`.
6. Everything list-shaped is paginated with bounded page sizes; deep pagination capped.
7. Query params validated declaratively (DRF serializer / pydantic / ninja), never raw; hierarchy filters use index-friendly tree ranges.
8. Cache keys deterministic/namespaced/sorted/hashed; expensive cache-fills use double-checked locking.
