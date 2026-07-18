# Observability & Disaster Recovery

Everything here is **as code** — dashboards, alarms, monitors, heartbeats, and error-tracking
projects are versioned Terraform, not console clicks. Referencing shared threshold/notification
profiles by name keeps alerting policy DRY.

## Metrics

- **In-cluster Prometheus** with long-term storage offloaded to **object storage via Thanos** — a
  private, name-prefixed bucket + an IAM policy granting Thanos access. A node-exporter and
  CloudWatch dashboards/alerts modules round it out. Grafana sits on top for visualization.
- **Why:** Prometheus data survives cluster loss (durable in object storage); metrics history isn't
  tied to a pod's lifetime.
- **Dashboards as code:** CloudWatch (and Grafana) dashboards are checked-in per subsystem —
  separate definitions for database, cache, search, load balancer — so a dashboard is a versioned
  artifact, diffable in review.
- **Common mistake:** dashboards clicked together in a console (lost on account rebuild, no review
  trail); Prometheus with only local/ephemeral storage so history dies with the pod.

## Alerting

- **CloudWatch alarms** generated per ingress/service, with a separate notification-receivers
  module.
- **Uptime / synthetic monitoring (SaaS, as code):** website / REST-API / page-speed / keyword
  monitors referencing **named threshold & notification profiles** (managed once, reused).
  Real health checks are encoded — home-page keyword match, `/version`, `/healthcheck`, a
  websocket health endpoint, and **search-index health monitors** (regex-match on cluster health
  JSON).
- **On-call / alerting (SaaS, as code):** **dead-man's-switch heartbeats** — a job must check in
  every interval; a miss raises an alert against the owning team.
- **Error tracking (SaaS, as code):** a project per team with a client key that carries **rate
  limits**; the DSN is parsed into components for app injection.
- **Why:** every monitor/alarm/heartbeat is versioned and reviewable; heartbeats catch *silent*
  failures (a cron/queue-consumer that simply stopped) that request-based uptime checks never see.
- **Tradeoff:** alerting spread across several SaaS providers fragments the on-call surface;
  regex-referenced shared profiles create a hidden dependency.
- **Common mistake:** monitoring only request uptime and omitting **heartbeats** for batch/cron
  jobs, so a nightly job can die unnoticed; hard-coding thresholds per-monitor instead of shared
  profiles.

## Logging

- **Two-hop pipeline:** apps log to **stdout/stderr** (structured JSON); a cluster log shipper
  forwards to durable **object storage**, which a **log-analytics SaaS** pulls from via
  cross-account bucket access. The buckets are fully locked down (`block_public_*`).
- **Edge log-push:** HTTP-request logs are pushed from the CDN/edge provider straight to a bucket,
  after an ownership-challenge handshake, with an **explicit field allow-list** and a URL filter to
  bound volume and limit sensitive-data capture at the source.
- **Why:** cheap durable buffering decoupled from the analytics vendor; field allow-listing controls
  cost and PII.
- **Common mistake:** apps logging to files inside the container instead of stdout/stderr;
  log-push jobs with no field allow-list/filter → huge egress bills and inadvertent PII capture; a
  public logs bucket.

## Tracing / APM

- APM/tracing is wrapped via a launcher script around the app process (agent log file pre-created
  and permissioned in the image); the APM sidecar drops all Linux capabilities and adds back only
  what it needs.
- A standalone APM stack (in docker-compose) exists for local trace analysis without a full
  environment.
- **Common mistake:** running an APM agent as root / with broad capabilities; enabling verbose
  tracing everywhere in prod (cost + overhead) instead of sampling.

## Disaster recovery

DR is a *set of consistent choices*, tiered to the data's value, not one module:

- **RTO/RPO by tier:** primary RDBMS → multi-AZ + automated backups + PITR + final snapshot +
  deletion protection; read replicas for read-scaling/standby (retention deliberately 0 on the
  replica); cache → **no snapshots** (reconstructable); broker → `prevent_destroy`.
- **Cross-region / cross-account replication** is first-class in the object-storage module
  (replication-follower IAM, owner-override); versioning + KMS support point-in-time restore.
- **Guardrails against destructive ops:** `prevent_destroy` on zones/brokers, `force_destroy=false`
  on zones, deletion protection on DBs, `create_before_destroy` on parameter groups/certs/policies
  to avoid gaps during replacement.
- **Metrics durability** via Thanos-in-object-storage; the edge/DNS/cert tier is fully reproducible
  from code, so its "restore" is `terraform apply`.
- **State/inventory as a recovery aid:** because every connection fact lives in the parameter store
  and every resource in code, rebuilding a consumer is just re-running its attachment module.
- **Why:** tiering backup posture to data value balances cost against RPO; `prevent_destroy` +
  deletion protection defend against the most common real outage cause — human error.
- **Tradeoff:** multi-AZ + PITR + cross-region replication cost real money; `prevent_destroy` can
  block legitimate teardowns and force manual state surgery.
- **Common mistake:** assuming **multi-AZ is a backup** (it's HA, not PITR); treating a
  snapshot-less cache/session store as durable; leaving cross-region replication off for the one
  bucket that holds irreplaceable data.

### A minimal DR posture to copy
- Every stateful resource: encryption at rest, automated backups with retention matched to value,
  deletion protection / `prevent_destroy`.
- The irreplaceable data: versioning + cross-region replication + tested restore.
- Metrics/logs: durable off-cluster storage.
- Everything else: reproducible from code (`terraform apply`) + config in a secret store.
- Write down the RTO/RPO per tier and actually rehearse a restore.
