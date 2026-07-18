# Celery, Background Jobs & RabbitMQ

**Scope:** Celery + an in-house task wrapper is used in the content-moderation service and the multi-brand platform, plus the traffic-analysis service. **The recommendation service uses RQ** (not Celery). **The analytics/ingestion service has no async task system.** The single source of truth for every convention below is the shared **in-house Celery task wrapper** — consumers only *declare* tasks and routes; the library enforces behavior.

## Use the custom `@shared_task` — never Celery's

```python
from app_tasks.task import shared_task   # NOT: from celery import shared_task
```

- **Why:** new Celery releases keep breaking task-invocation semantics. The wrapper pins one predictable behavior across versions and gives every task retries, idempotency, structured logging, dead-lettering, chain semantics, and APM naming for free.
- **Internal facts:** the wrapper always forces `bind=True` + a custom base task class internally; `apply()` re-raises everything except `Retry` (retries execute, not propagate) so eager/test mode matches prod. `OperationalError` (DB) is always retryable.
- **Bad alternative:** vanilla `@celery.shared_task` → version-dependent eager behavior, no structured logging, no DLQ, no idempotency.

```python
@shared_task(
    bind=True,
    name="publishing.publish_record",     # explicit, dot-separated — this name IS the routing key
    max_retries=8,
    retry_on_exceptions=[ObjectDoesNotExist, RecordAlreadyLockedError, *SearchIndexErrors],
    skip_if_running=True,
    reject_on_failure=True,
)
def publish_record(self, record_id: int, trigger: str = "unspecified"):
    self.set_logger_context(id=record_id, trigger=trigger)
    ...
```

**Naming:** always give an explicit dot-separated `name`. A deploy-time Django check fails the build if a task's name isn't routed. **`bind=True`:** used whenever the task needs `self` (almost always, for `set_logger_context`).

## Retries & exponential backoff

- **What:** declarative. `retry_on_exceptions=<exc|list>` + `max_retries=N`. Backoff is automatic and exponential: `countdown = 2 ** request.retries`. **Never call `self.retry()` manually.**
- **Where:** transient I/O (HTTP, search index, DB locks, FTP). Scale `max_retries` to how long you want to keep trying under `2**n`: webhooks `5–8`, search-index repair up to `14`.
- **Facts:** unpicklable retry exceptions are wrapped in a `RetryError`; on exhaustion `on_failure` runs, the exception hits the error-tracking service, and if `reject_on_failure` the message is dead-lettered.
- **Bad alternative:** scattered manual `try/except: self.retry(countdown=...)` → inconsistent backoff, forgotten caps, retries that silently break in eager/test mode.

## Idempotency / run-at-most-one — `skip_if_running=True`

Acquires a **PostgreSQL advisory lock keyed on the function name** (`advisory_lock(func.__name__, wait=False)` on the `direct` DB alias). If the lock is held, the task is **skipped** (state `"SKIPPED"`), not queued/blocked. **When:** singleton periodic/maintenance jobs where a second concurrent run is wasteful or unsafe and "skip this tick, next tick catches up" is fine (index reconciliation, external sync — `ensure_*_index_consistency`, all the `sync_*` jobs). **Tradeoff:** advisory locks auto-release on crash (better than Redis locks or DB status rows) but are per-function-name only (no per-argument granularity), and a skipped run is silently dropped until the next schedule.

## Dead-lettering — `reject_on_failure=True`

After retries are exhausted, raises `Reject(exc, requeue=False)` so RabbitMQ routes the message to the destination's dead-letter exchange (`{destination}_dlx`, requires `with_dead_letter_queue=True`). Alerts go on DLQs. Tuning: DLQ TTL 4 days ("postpone the Friday problem till Monday"), max length 1M, overflow `reject-publish` (keep oldest). **When:** work whose loss is unacceptable with a bounded external effect — webhooks, submissions, moderation, indexing.

## Task routing to worker queues — declarative, central

Routing is **declared in settings, never on the task**. A central route config maps task names → destination (exchange) via `.add_task_route(name, routing_key=...)`; the library generates Celery `task_routes`/`task_queues`. Auto-create is disabled (`task_create_missing_queues = False`) and every task must be routed (deploy check).

**Standard routing keys:** `priority`, `low_priority`, `maintenance`, `quick`, `memory_bound_1x`. A destination opts into a sub-queue per key with flags (`with_priority_queue`, `with_maintenance_queue`, `with_dead_letter_queue`, `with_memory_bound_1x_queue`, …); if a flag is off, that key falls back to the destination's default queue — so it's always safe to send any routing key to any task. A `"*"` route sends unregistered tasks to the `celery` queue.

**The caller chooses the routing key at send time** (QoS decision), not the task:
```python
routing_key = RoutingKey.PRIORITY if high_priority else RoutingKey.MAINTENANCE
publish_record.apply_async(args=[record_id], routing_key=routing_key)
```

**Workers subscribe to explicit queue lists** (`runcelery -Q ...`), one worker type per concern (submissions vs maintenance), deployed as separate process types. Isolates slow maintenance from latency-sensitive work. **Bad alternative:** Celery auto-creating queues → drift, orphans, unroutable tasks reaching prod.

## Concurrency & memory per worker

Run workers via `python manage.py runcelery`:
- `--cpu <cores-per-task>` → `concurrency = ceil(usable_cpu_cores / cpu_per_task)`, where `usable_cpu_cores()` reads **cgroup limits** (not `psutil`) so containers aren't over-subscribed.
- `--max-tasks-per-child=5000` always → bounds memory-leak impact.
- `-P solo` when concurrency==1 (avoid loading code twice); `-Ofair` + prefetch multiplier 1 (a slow worker doesn't hoard messages).
- NumPy/BLAS thread-pool limited per worker so children don't each spawn one thread per core.
- **`memory_bound_1x` routing key + queue** for memory-heavy tasks (>200MB) → run on a `--cpu 1` worker so each task gets full memory.
- **Lazy queues** (`x-queue-mode: lazy`) for long, slow, non-critical maintenance/cleanup backlogs → messages to disk, lower broker memory, at the cost of throughput.

**Budget:** default worker has ~512MB, ~200MB is the app itself; at 1x concurrency a task can use ~300MB, at 4x only ~75MB. Keep well below the ceiling — OOM makes a worker swap and crawl.

## RabbitMQ broker config & release provisioning

Broker settings (all in the wrapper's `app.py`): URL from `RABBITMQ_URL`; **JSON-only** serialization (no pickle); `broker_heartbeat=None` with TCP keep-alive instead; connect/read timeouts 20/30s; `broker_pool_limit=1`; timezone UTC. Reliability trio: `task_acks_late=True`, `task_reject_on_worker_lost=True`, `worker_cancel_long_running_tasks_on_connection_loss=True` — a killed worker never silently loses a task.

**Release-time provisioning** (the differentiator):
- `manage.py applyceleryconfig` declares all exchanges/queues from `TASK_ROUTES` *before* release, and can safely re-create queues without message loss (temp-queue + shovel), refusing if a queue is longer than the threshold.
- `manage.py applybrokerconfig` applies the managed message-broker provider's server settings, notably `{"rabbit.consumer_timeout": "false"}` (RabbitMQ's consumer timeout interferes with Celery ETA/countdown).

Standard release order: `check --deploy` → `migrate` → `applyceleryconfig` → `applybrokerconfig`.

## Structured logging & observability

`self.set_logger_context(**kwargs)` attaches structured context; the library auto-logs a `"Celery task post run"` event per task with `task_state`, `task_name`, `task_id`, retry count, `task_queue`, `task_routing_key`, and **`task_queue_time`** (computed from a publish-time header). Exceptions go to the error-tracking service with a `called: sync|async` tag; the APM tool names transactions via an instrumentation hook. **Reuse existing log keys** (`id`, `count`, `status`, `trigger`, `total`) — the downstream key budget (the log-aggregation platform) is limited.

## Exception handling in tasks

**Let tasks fail and retry naturally — don't wrap task bodies in `try/except`.** Celery's retry only fires on raised exceptions; the error-tracking service only captures uncaught ones; swallowing errors makes a failed task look successful and backs up the queue. Use `retry_on_exceptions` for transient errors. `try/except` is acceptable only for: expected business-logic exceptions (e.g. optional data not found), cleanup in `finally` (release locks), and partial-failure batch processing.

## Scheduling (Celery Beat)

Run via `manage.py runceleryscheduler` (wraps `celery beat -S django` with graceful SIGTERM→SIGQUIT). Two styles:
- **In-code `beat_schedule`** — simple, needs a deploy to change.
- **DB-backed 4-file convention:** (1) a `PeriodicTask` subclass in `periodic_tasks/tasks/` with `task_name`/`enabled`/`schedule`; (2) the domain `@shared_task(bind=True, name=<task_name>)`; (3) register in `ALL_PERIODIC_TASKS`; (4) add a route in the central route config. Schedules use a small set of staggered named crontabs to avoid on-the-hour stampedes.

Periodic tasks are almost always `routing_key=MAINTENANCE` (lazy, low-priority) and frequently `skip_if_running=True`.

## Eager execution & one-off runs

`ALWAYS_EAGER=True` makes `.delay()` synchronous (dev/tests); even then args are round-tripped through JSON so a non-serializable arg fails locally, not in prod. `manage.py runcelerytask <name> --task-kwargs '{...}' [--sync]` runs a single task without a worker — the standard for release-time data jobs. Chains don't break on a failed task by default (opposite of Celery); pass `error_breaks_chain=True` for strict all-or-nothing pipelines.

## Decorator flags — quick reference

| Flag | Effect | Use |
|------|--------|-----|
| `bind=True` | task gets `self` | nearly always |
| `name="..."` | routing name; required by route check | always |
| `max_retries=N` | cap under `2**n` backoff | any fallible I/O |
| `retry_on_exceptions=[...]` | which excs retry (DB `OperationalError` always added) | HTTP/index/DB/FTP |
| `skip_if_running=True` | PG advisory lock on func name; skip if held | singleton periodic jobs |
| `reject_on_failure=True` | dead-letter after retries | must-not-lose work |
| `routing_key=...` (call-time) | pick priority/maintenance/memory_bound queue | caller-driven QoS |
| `error_breaks_chain=True` (call-time) | abort chain on error | strict pipelines |
