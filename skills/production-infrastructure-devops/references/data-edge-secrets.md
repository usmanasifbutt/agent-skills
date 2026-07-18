# Data Services, Edge, DNS, Certificates & Secrets

## The control-plane / data-plane split (read this first)

Terraform provisions **all stateful infrastructure** (databases, caches, brokers, search, buckets,
CDN, DNS, certs) and all third-party SaaS. Kubernetes runs **only stateless workloads** — no
in-cluster databases/queues/search, no StatefulSets or PVCs for data. Data services are consumed as
**external managed endpoints**.

The glue is a repeated **instance → inventory → attachment** idiom:
1. **instance** — creates the managed resource.
2. **inventory** — the instance writes canonical connection facts (`host`, `port`, `user`,
   `password`, `schema`, `token`) into a hierarchical **parameter store** at a well-known path.
3. **attachment** — each *consumer* reads that inventory, mints its **own scoped least-privilege
   credentials** (its own DB role / broker vhost+user), and emits app-facing config vars
   (`<NAME>_URL`, marked sensitive).

- **Why:** decouples the data store's lifecycle from the many apps using it; N apps attach to one
  instance, each with its own credentials; connection strings never live in app repos.
- **Common mistake:** hard-wiring one shared "god" connection string instead of the
  instance→inventory→attachment indirection with per-consumer roles.

## Relational database (RDS)

- One wrapper over the community RDS module supports Postgres and MySQL via engine-keyed lookup
  maps. Concerns split into files (`main`, `parameters`, `security`, `inventory`).
- **Guardrails baked in:** `storage_encrypted`, `deletion_protection` and IAM DB auth default on;
  `publicly_accessible` default off; pinned CA cert; enhanced monitoring + CloudWatch log exports.
- **Multi-AZ** is a single toggle. **Read replicas** inherit the master's SG/param group and
  deliberately set `backup_retention_period=0` + `skip_final_snapshot=true`.
- **Backups/PITR:** retention default 7 (validated range), backup/maintenance windows, a
  `final_snapshot` prefix, restore via `restore_from_snapshot_id`.
- **Validation as code:** `locals` use `tobool("message")` to fail the plan on invalid storage
  (e.g. IOPS-to-GiB ratios) — fail at plan, not at the cloud API.
- **Parameter groups as code** with named feature bundles toggled by flags (logical replication for
  CDC; a bulk-import "maintenance" tuning bundle), deduped, `create_before_destroy`.
- **Common mistake:** enabling a `pending-reboot` parameter (e.g. logical replication) without a
  reboot window so it never takes effect; assuming multi-AZ is a backup.

## Postgres roles / databases / extensions as code

A dedicated module set operates *inside* a provisioned DB (connection sourced from the parameter
store):
- **role** — creates a role + optional db/schema and maps a coarse `permission` enum
  (`read-only` / `read-write` / `full-access` / `custom` / `owner`) to concrete grants; static
  access generates a password, IAM access does `GRANT rds_iam`.
- **db-roles** — stamps a **standard role topology** on every database: `master`, group roles
  (readonly/readwrite/fullaccess), IAM login roles (readwrite capped at a low `connection_limit` to
  protect the primary), plus toggled CDC/BI/maintenance/metrics roles. Each role's URL is written
  back to the parameter store as a `SecureString`.
- **db-extensions** — declares extensions and logical **replication slots** as first-class
  resources.
- **Why:** roles/grants/extensions become reviewable, drift-detected code with a consistent
  least-privilege ladder; app credentials are per-consumer.
- **Common mistake:** granting `full-access`/`owner` to app roles for convenience — reserve `owner`
  for DB creation, put apps on read-write with a low connection limit.

## Cache (managed in-memory store)

- Managed replication group; topology (`num_cache_clusters`, `automatic_failover_enabled`) computed
  from a `(read_replica, multi_az)` truth table.
- **TLS + auth token** when enabled; a sane default parameter group (`maxmemory-policy=allkeys-lru`).
- **Snapshots default off** — treated as a pure cache, not a datastore. The attachment is
  **type-aware** (cache / sessions / tasks) and emits the correctly-named env var per usage —
  encoding "one instance per role."
- **Common mistake:** reusing an evicting (`allkeys-lru`) instance for sessions or a task broker,
  where eviction silently drops data; treating a snapshot-less cache as durable.

## Message broker — managed SaaS vs in-cluster

- Consumed via a **managed message-broker SaaS**, provisioned as code with `prevent_destroy=true`
  (guard against queue loss). Dedicated plans enable a metrics/Prometheus plugin, a
  delayed-message plugin, and a critical override (**disable the broker's consumer timeout** when it
  conflicts with the task library's delayed/retry semantics).
- **Alerts** are a separate module: CPU/memory/disk in **sustained + spike pairs**, disk as
  percent-free, queue-depth by class (all / maintenance / dead-letter) routed to critical vs warning
  recipients. Disable the vendor's default alarms before layering custom ones.
- Attachment mints a **per-consumer vhost + user** and emits an `amqps://…/<vhost>` URL.
- **Managed vs in-cluster tradeoff:** managed offloads clustering/HA/upgrades and gives built-in
  metrics; in-cluster (Helm) gives cost control and locality but forces you to own quorum queues,
  disk, and failover — which this estate avoids (no broker in the cluster).
- **Common mistake:** sharing one vhost across services instead of one per consumer; leaving both
  default and custom alarms on (duplicate/no-op paging).

## Search — self-managed cluster vs hosted SaaS

- **Managed cluster (OpenSearch/Elasticsearch):** zone-awareness/dedicated-masters toggles;
  fine-grained logging opt-ins (each creates a log group + resource policy); an advanced-security
  bundle (encrypt at rest, node-to-node encryption, internal user DB) behind one flag. Note: the
  access policy relies on **network placement**, so a `Principal:*` policy is a footgun if the
  domain is ever exposed.
- **Hosted search SaaS:** the attachment creates **two scoped keys per index prefix** — an **admin**
  key (index management) and a **search-only** key — so the frontend only ever gets the search key.
- **Common mistake:** shipping the **admin** key to the browser instead of the search-only key
  (the two-key split exists precisely to prevent this).

## Object storage (S3)

Two modules for two intents:
- **Public static-asset bucket:** `acl=private` bucket with a policy granting anonymous
  `s3:GetObject` under conventional prefixes — for assets only.
- **Governed general-purpose bucket:** declarative per-prefix access (`read`/`write`/`full_access`
  → least-privilege statements), **versioning**, **encryption** (SSE or a customer-managed KMS key
  with rotation), `BucketOwnerEnforced` ownership (ACLs disabled), **lifecycle** rules (expire /
  transition to archive), and cross-account/-region **replication**.
- **Common mistake:** using the public static-asset module for user uploads; enabling versioning
  without lifecycle expiry of noncurrent versions (unbounded cost).

## CDN & edge

- **Intent presets, not knobs:** a `kind` abstraction (`default` / `images` / `custom`) selects
  bundles of allowed/cached methods, TTLs, and forwarded values. Cache/origin-request/
  response-headers policies are reusable named objects; the response-headers policy encodes CORS +
  security headers (CSP, HSTS, referrer-policy) declaratively; a WAF/web-ACL attaches.
- A **third-party CDN/edge provider** is also fully code-managed (zone, records, page-rules,
  firewall, log-push) with **per-endpoint page rules** by intent (`cache_everything` for
  assets/images with a host-header override; `bypass` for API/auth).
- **Common mistake:** `cache_everything` on an authenticated path; forgetting a host-header override
  when caching a proxied origin (origin rejects the request); page-rule priority tied to list index
  so reordering silently reshuffles precedence.

## DNS

- Zones and records fully as code (native provider and/or a third-party provider with a per-record
  `proxied` flag). The zone module explicitly manages **NS and SOA** records; the records module
  supports **alias** records (with `evaluate_target_health`) and returns FQDNs by re-suffixing with
  the zone name.
- **Common mistake:** managing a zone's records in code but leaving NS/SOA unmanaged (or vice versa)
  so registrar delegation drifts from the zone.

## Certificates

- **ACM with a two-call split:** call #1 creates the cert (`validate_certificate=false`); call #2
  creates the DNS validation records and waits — avoids a chicken-and-egg dependency when the
  validating zone lives in another account/provider.
- Edge-provider certificate packs (wildcard + apex) use TXT or DCV-delegation validation.
- **Common mistake:** creating cert + validation records in one shot across an account boundary
  (cycle/permission failure) instead of create-then-validate.

## Secrets management

Three tiers, always **structure not values** (git/images hold references only):
- **Parameter-store wrapper** — the workhorse. Modes `write` / `read` / `read_all`.
  **Auto-classifies** each var as `SecureString` vs `String` by testing sensitivity (with
  force overrides), and emits a **checksum** (sha256 of a scrubbed JSON) so consumers detect
  rotation without exposing the value in a diff. Everything under a hierarchical path.
- **Generic secret store** — provider-agnostic secrets that live only in (encrypted) state.
- **In-cluster consumption** — the **External Secrets Operator** with a `ClusterSecretStore`
  backed by the parameter store; `ExternalSecret` objects declare *which path* to pull with a
  refresh interval, and the operator materializes real K8s Secrets. ArgoCD's own cluster/repo
  registrations are themselves ExternalSecrets. Ephemeral environments additionally use **SOPS**
  (KMS-encrypted files) merged at render time.
- **Why:** no secrets in git; rotation in the store; least-privilege via IAM; the checksum lets apps
  redeploy on rotation without reading the secret.
- **Tradeoff:** rendering/deploy depends on live access to the secret store + correct IAM (a store
  outage blocks *rendering*, not just running); secrets still land in Terraform state, so state must
  be encrypted and access-controlled.
- **Common mistake:** committing plaintext secrets or baking them into chart values/image layers;
  pulling each variable individually (N API calls) instead of a recursive path fetch; passing a real
  secret without marking it sensitive so it's written as plaintext and shows in plan output.
