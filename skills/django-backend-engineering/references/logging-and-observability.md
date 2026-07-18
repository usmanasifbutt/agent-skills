# Logging & Observability

## structlog is the exclusive logger

**No `print`, no stdlib `logging`.** Every service configures `structlog` (JSON output in prod, console in dev).

```python
import structlog
LOGGER = structlog.get_logger()          # or structlog.getLogger(__name__)
LOGGER.info("Record published", record_id=123, trigger="sync")   # message + structured key/values
```

- **Bound loggers** carry context across multiple lines: `logger = LOGGER.bind(record_id=record_id); logger.info("Step 1"); logger.warning("Unusual", extra="x")`.
- **Config:** stdlib integration via `ProcessorFormatter.wrap_for_formatter`, two formatters — `verbose` (`ConsoleRenderer`, dev) and `json_only` (`JSONRenderer(sort_keys=True)`, prod); noisy libs (`pika`, `celery`) pinned to `CRITICAL`; `cache_logger_on_first_use=True`. Team-wide: services add a `trace_id_processor` / `merge_threadlocal` but use the same shape.

## Reuse existing keys — the key budget is limited

**Never invent new structured-log keys casually.** The downstream store (the log-aggregation platform) can index only a limited number of distinct keys. Reuse established ones (`id`, `count`, `status`, `trigger`, `total`, `url`, `request_id`). Grep the codebase / the log platform before adding a key.

## Log-platform-aware helpers (`core/tools/logging.py`)

- `create_request_logger(request, module_name, view=...)` — binds `request_id` (from `X-Request-ID`) and `url` onto every line.
- `loggable_json_dumps(value)` — prefixes JSON with `#` so the log platform doesn't try to parse it into keys.
- `parse_data_for_logging(...)` — flattens dicts to lists (logging a dict blows the key budget and mis-parses).
- `sample_ordered_list(list, sample_size=10)` — logs head+tail of long lists to bound log size.
- `loggable_chunks_progress(count, offset, limit)` — emits `"2/10"` progress.

## Views don't log; tasks log via context

- **Views should not routinely generate log messages** — only exceptionally, to track a specific error.
- **Celery tasks** attach context with `self.set_logger_context(id=..., trigger=...)`; the in-house Celery task wrapper auto-logs a lifecycle event per task (`task_state`, `task_name`, `task_id`, retries, `task_queue`, `task_routing_key`, `task_queue_time`). See `celery-and-jobs.md`.

## Error tracking

`setup_error_tracking(dsn, release, environment, ...)` with `DjangoIntegration` + `CeleryIntegration` + a logging integration (`level=DEBUG` breadcrumbs, `event_level=CRITICAL` events). Standard `before_send` hooks trim structlog frames from the traceback and re-parse the structured message so the error tracker gets `message` + `params` instead of a stringified dict; `ignore_errors` drops Celery lifecycle noise (`Ignore`, `SystemExit`, `WorkerShutdown`) and transient infra errors (kombu/RabbitMQ `OperationalError`, transient psycopg2 connection errors, per-task 502/503/504/429). Same shape team-wide. Because the custom exception handler bypasses Django's default, `server_error_view` explicitly logs + `capture_exception()`.

## APM (the multi-brand platform only)

An APM agent (`elasticapm.contrib.django`-style) in `INSTALLED_APPS`; APM config with `SERVICE_NAME=f"{APP_PORTAL_NAME}-python"`, env-driven `TRANSACTION_SAMPLE_RATE`, `TRANSACTION_MAX_SPANS=50`, `SPAN_STACK_TRACE_MIN_DURATION="10ms"`, gated by an `APM_ENABLED` flag. Celery tasks are instrumented via an instrumentation hook so transactions/spans name correctly despite the custom `@shared_task`. (Other services rely on structlog + error tracking + the log platform only.)

## Layered-style logging (analytics/ingestion service)

The layered services use a `LoggingEvent` **enum** as the message key and a shared `get_logger(__name__)`, plus a `trace_id_processor` and `AggregateLogProcessor` — a stricter form of "reuse existing keys": the set of event names is a closed enum. `SerializationValidationError` sanitizes PII keys (`email_address`, `session_id`, `phone_number`, `user_id`) out of error output.

## Rules to codify

1. `structlog` only; `LOGGER = structlog.get_logger()`; message + structured kwargs.
2. Reuse existing keys; adding a key is a deliberate decision (limited log-platform budget).
3. Views don't log routinely; bind request context via `create_request_logger`; tasks use `set_logger_context`.
4. JSON renderer in prod, console in dev, via a per-env settings switch.
5. Error tracking with a logging integration + `before_send` filtering; never leak PII or secrets into logs/URLs.
6. Never put personal/sensitive data in URLs or query strings.
