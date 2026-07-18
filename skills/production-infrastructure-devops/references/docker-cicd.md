# Docker, CI/CD & Deployment Strategy

## Dockerfile patterns

- **Multi-stage** with named stages (`base → deps → build → runtime`). A shared pinned base declared
  once and reused (`FROM base AS ...`). The fat build stage (compilers, dev deps) is discarded; the
  runtime stage copies only the built artifact (virtualenv / compiled assets).
- **Pin by tag *and* digest:** base images `image:tag@sha256:...`, the `# syntax=` frontend
  digest-pinned, tools version-pinned, lockfiles frozen (`--frozen`/`--frozen-lockfile`). Pull
  through a registry mirror rather than a public registry directly.
- **Non-root:** runtime stages create an unprivileged user and `USER` it before `CMD`; the entrypoint
  refuses to run as an unknown UID and only steps *down* from root (e.g. via `gosu`); files copied
  with `--chown`.
- **Cache-friendly layer order:** copy dependency manifests and install **before** copying app
  source, so a source-only change doesn't bust the dependency layer.
- **`.dockerignore`** excludes VCS/IDE/caches/`node_modules`/build output/logs to keep the build
  context small and builds reproducible.
- **Fail-fast build args:** `ARG` for per-build config, validated with `RUN test -n "$X"` / probes so
  a misconfigured build breaks immediately with a clear message.
- **Health** delegated to the orchestrator (K8s probes) rather than a baked-in `HEALTHCHECK` — keeps
  the image orchestrator-agnostic (compose-only infra services still get a compose `healthcheck`).

```dockerfile
# syntax=docker/dockerfile:1.x@sha256:<digest>
FROM <registry>/debian:bookworm-<date>-slim@sha256:<digest> AS base
FROM base AS deps
RUN --mount=type=cache,target=/root/.cache/pkg \
    --mount=type=bind,source=lock,target=lock \
    install-deps --frozen        # deps installed before source copy
COPY . /build && build-app
FROM base AS runtime
RUN adduser --system app
COPY --from=deps --chown=app:app /opt/venv /opt/venv
USER app
ENTRYPOINT ["/entrypoint.sh"]
```
- **Common mistake:** copying the whole source before installing deps (busts cache every change);
  running as root; floating base tags (not reproducible, vulnerable to tag mutation).

## Image optimization & build caching

Three complementary layers:
1. **BuildKit cache mounts** in the Dockerfile (package caches, webpack/node cache).
2. **Registry-backed layer cache** in CI (`cache-from`/`cache-to type=registry,...,mode=max`)
   pointed at a shared cache registry, so layers are reused across runs and branches.
3. **A `deps`-target pre-build** run first to populate/reuse the dependency layer.

**Cache-key discipline:** write fresh cache **only from the default branch**; PR branches restore
only. An **image-existence check** queries the registry for content-hash tags in parallel and skips
building images whose inputs are unchanged. A "rebuild-without-cache" escape hatch exists.

- **Common mistake:** writing build cache from every branch (thrash + poisons trunk's baseline);
  `mode=min` losing intermediate layers; rebuilding an image that already exists for the same input
  hash; no `ignore-errors` on `cache-to` so a cache hiccup fails the build.

## docker-compose — one file per purpose

Compose files are split by intent, DRY'd with YAML anchors, not duplicated:
- **Base** — full production-parity dev stack (proxy + app + workers + DB + broker + cache + search
  + object-storage mock) with `healthcheck:` on infra services and `depends_on: { condition:
  service_healthy }`.
- **`*.local`** — laptop-friendly (bind-mounted source, optional services via `profiles:`,
  `${VAR:-default}` indirection).
- **`*.dev`** — override adding verbose logging / a file-watch reloader.
- **`*.lite`** — infra-only (datastores + broker + search) to run the app outside containers.
- **`*.k8s`** — parity overrides rewriting service URLs to in-cluster DNS and secret paths.
- **Add-on stacks** (e.g. an APM stack) brought up alongside on their own network.

**Parity mechanism:** the *same* entrypoint wrapper and the *same* images dev→prod; only the
injected config differs.
- **Common mistake:** one monolithic compose file with hand-toggled commented sections; dev using
  different images/entrypoints than prod (parity drift); `depends_on` without health conditions so
  the app races its database.

## CI/CD (GitHub Actions)

- **Stages:** `lint → test (sharded matrix) → build (per image, per tenant) → push → deploy →
  post-deploy smoke/e2e`. Infra CI additionally runs `fmt`/`tflint` and real `init/validate/plan/
  apply/destroy` on changed modules in a throwaway context.
- **DRY, computed pipelines:** heavy `workflow_call` reusable sub-workflows (setup → build → push)
  and composite actions; matrices **computed in a prep job** so one dispatch input expands into the
  right per-tenant/per-image matrix (`fail-fast:false`).
- **Per-environment deploy workflows** differ only in trigger + tenant list + which environment's
  role/KMS ARNs are passed — all delegating to the same shared sub-workflows.
- **Auth = OIDC role-assumption** (`id-token: write`, no long-lived keys), often chained
  (automation role → target-account role); registry login via the standard action.
- **Concurrency** group per workflow+ref with `cancel-in-progress` so superseded runs stop.
- **Runner cost:** stable runners for protected branches, cheaper **spot** runners elsewhere (chosen
  by expression).
- **Artifacts between jobs** (resolved config, computed tag) are **KMS-encrypted, 1-day retention**.
- **Common mistake:** copy-pasted per-environment logic instead of `workflow_call`; no `concurrency`
  group (stacked runs); long-lived cloud keys in secrets instead of OIDC; secrets passed between
  jobs as plaintext artifacts.

## Deployment strategy — build → version → GitOps

1. **Setup** resolves layered build-time config and computes a **version = `<code-version>-<config
   -hash>`** (code = tag in prod / short SHA in dev; config = SHA of the resolved config). Snapshots
   the config to object storage with retention + release metadata.
2. **Build** builds each image with that tag, pushes to a per-tenant registry repo, and passes
   config as a **build secret** (`--mount=type=secret`), never a layer.
3. **Push/deploy** opens a PR against the **GitOps deploy repo** bumping the image `tag:` for that
   tenant/env.
4. **ArgoCD auto-syncs** and renders; releases are `atomic:true`.

- **Trigger model:** production by semver git tags; staging by `rc` tags; dev on a cron + manual
  dispatch. A tag encoding a tenant restricts the matrix (targeted hotfix).
- **Rolling updates** via the orchestrator, tuned per service (e.g. `maxSurge:100% /
  maxUnavailable:0` for zero-capacity-loss). Traffic is segmented into **formations** each
  independently scaled/routed — workload isolation rather than canary. (No blue/green or canary
  machinery here — the primitives are K8s rolling updates + ArgoCD sync with retry/backoff.)
- **Promotion** is rebuild-per-environment (because the config hash — and thus the image — is
  environment-specific), not artifact promotion of a single pre-built image.
- **Common mistake:** mutating cluster state imperatively instead of committing the tag; reusing a
  `latest` tag (can't tell/rollback what's running); baking environment config into the image at a
  fixed tag so it can't be promoted safely.

## Rollback

**GitOps revert.** Because the running version is a declared, never-overwritten content-addressed
tag, reverting the commit (or setting the tag back) makes ArgoCD re-sync to the prior image. Helm
releases are `atomic` so a *failed* apply auto-rolls-back; archived config snapshots make any past
release's exact build-time config recoverable.
- **Common mistake:** `kubectl rollout undo` on a cluster ArgoCD immediately re-syncs (fighting
  GitOps); overwriting image tags so the previous artifact no longer exists.

## Smoke tests / e2e gating

A scheduled smoke-test workflow runs the e2e suite against the trunk build on a cron and on demand,
reusing the same cached build pipeline, and supports **re-running only the failed jobs** from a
prior run. Builds feeding e2e are gated on image-existence.
- **Common mistake:** re-running the whole multi-hour matrix to retry one flaky shard; rebuilding
  all images for every smoke run.

## Health checks

- **Infra (compose):** `healthcheck:` per backing service (`pg_isready`, broker ping, cache ping,
  search HTTP) with `interval/timeout/retries/start_period`; apps gate startup on them.
- **App (K8s, via chart values):** **readiness ≠ liveness**; a **`startupProbe`** (HTTP GET on a
  `/health` path with a generous `failureThreshold`) lets slow cold starts finish before liveness
  takes over; `terminationGracePeriodSeconds` + a **PodDisruptionBudget** for controlled disruption.
- **Ordered startup:** the entrypoint runs an ordered hook directory and a **sanity-check** script
  that validates required env/config and fails fast before launching the process.
- **Common mistake:** one probe for both liveness and readiness (a slow dependency → crash loops);
  no startup probe so a slow boot trips liveness; no grace period/PDB so rollouts drop in-flight
  requests.

## Feature flags & release hooks

- A container **running mode** (`release` / `web` / `worker`) toggles startup behavior via env
  flags. **`release` mode runs migrations then exits** — a one-shot release/migrate pod — and only
  if there are unapplied migrations, wrapped in a timeout monitor, skippable via a flag, and skipped
  for ephemeral/maintenance modes.
- Config vars come from the secret store **at deploy time**, functioning as **runtime feature flags
  without an image rebuild**; a `maintenanceMode` value gates whether web/workers run.
- **Common mistake:** running migrations unconditionally on every pod start (races, downtime)
  instead of a gated one-shot step; coupling a feature toggle to an image rebuild.

## Observability hooks in deploys

Error-tracking wired at **build time** (source-map upload, toggleable and disabled for prod bundles)
and **runtime** (a DSN config var). APM via a launcher wrapper. Logs to **stdout/stderr** for the
cluster shipper. Config snapshots + release object tags (`ref`, `version`, `checksum`, `created-by`)
provide deploy provenance.
- **Common mistake:** logging to files inside the container; shipping browser source maps in prod
  (leaks source).

## Container security

- **Non-root** everywhere; entrypoint only steps down, never up. **Minimal base**
  (`--no-install-recommends`, clean apt lists), fat build stage discarded. **Pinned digests** +
  frozen lockfiles.
- **Secrets never baked in:** build-time via BuildKit `--mount=type=secret` (never `COPY`/layer);
  runtime injected at start from a mounted file; CI artifacts carrying config are KMS-encrypted;
  ephemeral-env secrets SOPS-encrypted at rest in git.
- **Least privilege:** OIDC federated short-lived role-chained credentials; per-tenant isolated
  registry repos and accounts; APM sidecar drops all capabilities; workloads pinned to node groups.
- **Supply chain:** a dependency auto-update bot on a weekly cadence.
- **Notable gap to close:** no explicit **image-vulnerability scan** gate was present — base-image
  pinning + a dependency bot is *not* a substitute. Add a scan (e.g. Trivy/Grype) to the build.
- **Common mistake:** `COPY .env` / secrets into a layer (recoverable from image history); running
  as root "to make it work"; floating base tags; plaintext secrets in the deploy repo; long-lived
  static cloud keys in CI.
