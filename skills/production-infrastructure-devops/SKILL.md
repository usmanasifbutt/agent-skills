---
name: production-infrastructure-devops
description: How to run production infrastructure and delivery the disciplined way — Terraform (a versioned reusable-modules repo consumed by thin PR-automated live configs), AWS networking/VPC/EKS/IAM, Kubernetes via Helm + Helmfile, GitOps with ArgoCD, managed data services (RDS/Redis/broker/search) wired in through an instance→inventory→attachment pattern, secrets via a parameter store + External Secrets, observability (Prometheus/Thanos/CloudWatch + uptime/on-call/error-tracking as code), and the delivery side — multi-stage Docker images, cached GitHub Actions pipelines, content-addressed image tags, GitOps-driven rolling deploys, and revert-based rollback. Load this BEFORE writing or reviewing ANY infrastructure or delivery code — Terraform modules/live config, VPC/IAM/security groups, EKS/node groups, Helm charts/Helmfile values, ArgoCD Applications, secrets wiring, DNS/CDN/certificates, Dockerfiles, docker-compose, CI/CD workflows, or deployment/rollback plans — and when bootstrapping a new service, environment, or account.
---

# Production Infrastructure & DevOps

Standards for provisioning cloud infrastructure and delivering software to it, reverse-engineered
from a real platform: a Terraform **modules** library, PR-automated Terraform **live** repos, a
**Helmfile** app-definition repo, an **ArgoCD** GitOps control-plane, and application build/deploy
pipelines. Everything below is expressed generically — apply it to any AWS + Kubernetes estate.

## The philosophy in one screen

Two structural splits define everything:

1. **Reusable library vs. live wiring.** A versioned **modules** repo (pure library — no state, no
   accounts) is consumed by thin **live/root** configs (one directory = one state) that supply real
   account/environment values. Modules propagate by **pinned git ref**, never local path.
2. **Control plane vs. data plane.** Terraform owns *stateful* things (databases, caches, brokers,
   search, buckets, CDN, DNS, certs) **and** all third-party SaaS as code. Kubernetes runs *only
   stateless* workloads — no in-cluster databases/queues/search. The two planes meet through a
   parameter store: infra publishes connection facts, apps consume scoped credentials.

Everything else follows from these "north stars":

- **Directory layout = state layout = governance layout.** The path drives the state key, the
  remote-state wiring, PR autoplan, and who is allowed to apply.
- **Define once, render many.** One generic template per service/module; environments contribute
  only their *differences* through layered defaults (`all → type → instance`).
- **Everything derived, nothing hand-typed.** Names, tags, subnets, ARNs flow from
  constants/attributes/computation modules so resources are consistent and traceable.
- **Git is the single source of truth; pull, don't push.** ArgoCD reconciles continuously; CI
  holds no cluster credentials. Rollback is `git revert`.
- **Version-pin everything** — Terraform CLI, every provider, every internal module ref, every
  base image (by tag *and* digest), every chart (semver, never `latest`).
- **Secrets never live in git or image layers.** They live in a secret store; git/images hold only
  references, and secrets are injected at render/run time.
- **Fail fast, fail in review.** `validation{}` + static assertions in Terraform, JSON-schema
  validation of Helm values in CI, Dockerfile `RUN test` assertions, `atomic` releases.
- **Don't fight the platform.** GitOps ignores autoscaler-owned replica counts; declare desired
  state and let HPA/ALB/EKS own the rest.
- **Scope the blast radius.** Separate accounts/clusters per environment; least-privilege IAM;
  `prevent_destroy` + deletion protection on irreplaceable resources.
- **Immutable, content-addressed artifacts.** Image tag = `<code-version>-<config-hash>`, never
  reused; promotion and rollback are git operations on that tag.

## Reference map — read the file for the area you touch

| Working on… | Read |
|-------------|------|
| Terraform repo topology, Atlantis PR flow, state, module design, IAM, provider strategy, env separation | `references/terraform-iac.md` |
| VPC, subnets, NAT, security groups, EKS, node groups, autoscaling, load balancing | `references/networking-compute.md` |
| Helm, Helmfile, environment values, review apps, ArgoCD/GitOps, ingress, rollout/rollback | `references/kubernetes-helm-gitops.md` |
| RDS/Postgres, Redis, message broker, search, S3, CDN, DNS, certificates, secrets | `references/data-edge-secrets.md` |
| Prometheus/Grafana/Thanos, CloudWatch, uptime/on-call/error-tracking, logging, disaster recovery | `references/observability-dr.md` |
| Dockerfiles, docker-compose, build caching, GitHub Actions CI/CD, deploy strategy, health checks, container security | `references/docker-cicd.md` |

---

## Architecture principles

1. **Layer the estate.** modules (library) → live/root (wiring) → a tiny `setup/` that bootstraps
   the state backend with local state. App workloads live in a separate Helm/GitOps repo.
2. **One state per leaf directory**, keyed to mirror the path; chain projects with remote-state
   outputs (a central account registry feeds provider targeting everywhere). Directories, not
   workspaces, separate environments.
3. **Wrap SaaS like cloud.** Every third-party service (DB/cache/broker/search/CDN/DNS/monitoring/
   alerting/error-tracking) is a Terraform module with a uniform interface, so consumers don't care
   which vendor is behind it.
4. **Stateless cluster.** Managed data services out of cluster; consume them via
   instance→inventory→attachment with per-consumer least-privilege credentials.
5. **GitOps delivery.** Build → push an immutable image → open a PR bumping the tag in the deploy
   repo → ArgoCD auto-syncs. No imperative `kubectl` against production.
6. **Environments are isolation boundaries**, ideally separate accounts/clusters; production carries
   extra guardrails (mandatory extra review, higher HA/retention defaults).
7. **Cost is a first-class constraint** — shared ALB groups, ephemeral environments that sleep
   off-hours, aggressive-but-disciplined build caching, spot runners off the critical path.

---

## Infrastructure checklist

- [ ] New infra goes in a **module** (reusable, versioned) if others will use it; only *wiring* goes
      in the live/root config.
- [ ] Resource is **multi-AZ** where it must survive a zone loss; single-AZ only for
      dev/disposable.
- [ ] **Least-privilege IAM**: roles not users, assume-role/OIDC (IRSA for pods), scoped ARNs, no
      `Action: *` on shared roles.
- [ ] **Encryption at rest** on, **deletion protection** / `prevent_destroy` on irreplaceable
      resources, backups/PITR configured with a retention that matches the data's value.
- [ ] Networking: private subnets for workloads/data, DB subnets closed to the internet by default,
      one **NAT per AZ** with stable egress IPs if partners allowlist you, default SG/ACL/route
      table left unmanaged-and-empty.
- [ ] Tags derived from the standard tag set (`MaintainedBy`, `Module`, env, owner) — no
      hand-typed names/tags.
- [ ] SaaS provisioned as code, not clicked in a console; its alerts/keys/DSNs managed in the same
      module.
- [ ] DR posture explicit: know the RTO/RPO for this resource; cross-region replication for the
      truly irreplaceable; don't mistake multi-AZ for a backup.

## Terraform checklist

- [ ] Module reference is a **pinned git ref** (`?ref=vX.Y.Z`), never a local path from a live repo.
- [ ] Terraform CLI pinned (`.terraform-version`); every provider pinned with a sane floor+ceiling;
      internal modules bumped deliberately via PR.
- [ ] Module has the full skeleton: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`,
      auto-generated `README.md`; every variable typed, described, `nullable=false`, with
      `validation{}` where it matters; sane defaults; a `create` toggle if it can be optional.
- [ ] Refactors ship `moved {}` blocks so consumers don't destroy/recreate.
- [ ] Remote state backend with locking; one state per leaf dir; state key mirrors the path; no
      secrets in state you can avoid, and state is encrypted.
- [ ] Semver discipline on the module library: patch = fixes only, minor = back-compat (breaking
      only with a migration guide), major = structural.
- [ ] PR automation (Atlantis or equivalent): autoplan scoped to real projects (not nested
      `modules/`), team-based apply permissions by path, extra reviewer on `**/production/**`,
      parallelism tuned per project.
- [ ] `terraform fmt`/`validate`/`tflint` + secret-detection run pre-commit and in CI.

## Helm checklist

- [ ] One versioned chart per service type, pulled from a registry by **pinned semver** (never
      `latest`); no giant umbrella chart / multi-thousand-line values file.
- [ ] Deployment repo stores **only values** (inputs to charts), layered `default → type →
      instance` via merge; environment files carry only differences.
- [ ] Values validated against a **JSON schema** in CI (`additionalProperties:false`, enum
      allow-lists) so typos fail in review.
- [ ] A booby-trapped `default` environment (or equivalent) so a command missing its `--environment`
      fails loudly instead of deploying somewhere unintended.
- [ ] Releases are `atomic:true` + `cleanupOnFail` with a timeout so a failed upgrade auto-rolls
      back the release.
- [ ] Probes set as chart values: **readiness ≠ liveness**, a `startupProbe` for slow boots,
      `terminationGracePeriodSeconds`, and a PodDisruptionBudget.
- [ ] Workload identity via IRSA (service-account role annotations), never static cloud keys in pods.
- [ ] `nodeSelector` + matching `tolerations` for node-group pinning; pod anti-affinity for AZ
      spread.

## GitOps checklist

- [ ] Pull-based: ArgoCD watches git; **no cluster credentials in CI**.
- [ ] App-of-apps bootstrap; ArgoCD manages itself; clusters/repos registered declaratively (as
      External Secrets, not plaintext).
- [ ] `syncPolicy.automated` with `prune` + retry/backoff; `CreateNamespace=true`; **`selfHeal`**
      where drift must be corrected.
- [ ] **`ignoreDifferences` on `Deployment.spec.replicas`** for any autoscaled workload (or ArgoCD
      and the HPA will fight).
- [ ] `manifest-generate-paths` scoped to the files that actually affect a render, so a commit
      doesn't re-render every app.
- [ ] ArgoCD **Projects** restrict allowed source repos and destinations; ApplicationSets for
      fan-out (per-cluster, PR-based review apps).
- [ ] Promotion = commit a new immutable image tag; **rollback = `git revert`** — never mutate
      cluster state imperatively, never re-push a tag.
- [ ] Review/ephemeral environments are isolated (own namespace, throwaway branch), auto-pruned,
      and sleep off-hours to cut cost.

## Docker checklist

- [ ] **Multi-stage** build; fat build stage discarded; slim runtime.
- [ ] Base image pinned by **tag *and* digest** (and the `# syntax` frontend too); tools/lockfiles
      pinned/frozen.
- [ ] Dependency manifests copied and installed **before** app source (cache-friendly layer order).
- [ ] Runs as a **non-root** user; entrypoint only steps *down* from root; files `--chown`ed.
- [ ] `.dockerignore` keeps the build context small (VCS, caches, `node_modules`, build output).
- [ ] **No secrets in layers** — build-time config via BuildKit `--mount=type=secret`, runtime
      config injected at start from a mounted file/env.
- [ ] Build args validated with `RUN test -n "$X"` fail-fast assertions.
- [ ] Health delegated to the orchestrator (K8s probes) rather than baked in — or a `HEALTHCHECK`
      for compose-only services.
- [ ] Consider an image-vulnerability **scan gate** in CI (a common gap — pinning + a dependency
      bot is not a substitute).

## CI checklist

- [ ] Pipeline stages: **lint → test → build → push → deploy → post-deploy smoke/e2e**.
- [ ] Reusable workflows (`workflow_call`) + composite actions; **no copy-pasted** per-environment
      logic; matrices computed in a prep job.
- [ ] **OIDC role-assumption** for cloud auth (no long-lived keys); registry login via standard
      action; secrets never passed between jobs as plaintext artifacts (encrypt + short retention).
- [ ] Layered build caching: BuildKit cache mounts + registry layer cache; **fresh cache written
      only from the default branch**; PRs restore only; image-existence check skips rebuilding
      unchanged images.
- [ ] `concurrency` group per workflow+ref with `cancel-in-progress` so superseded runs stop.
- [ ] Spot/cheap runners off the critical path; stable runners for protected branches.
- [ ] Infra CI actually plans/applies changed modules in a throwaway context (real unit tests),
      plus `fmt`/`tflint`.

## Production deployment checklist

- [ ] Image is **immutable + content-addressed** (`<code>-<confighash>`); config is a build secret,
      not a layer; a config snapshot is archived with retention + release metadata.
- [ ] Deploy is a **PR against the GitOps repo** bumping the tag; ArgoCD auto-syncs; release is
      `atomic`.
- [ ] Rollout strategy set per service (rolling with `maxSurge`/`maxUnavailable` tuned for
      zero-capacity-loss); traffic "formations" scaled/routed independently where needed.
- [ ] **Migrations run as a gated one-shot release step**, only if there are unapplied ones, with a
      timeout monitor — never unconditionally on every pod start.
- [ ] Feature toggled via a **config var / running-mode flag**, not an image rebuild.
- [ ] Health/readiness verified; **post-deploy smoke/e2e** gates the release; observability
      (metrics/tracing/log shipping) confirmed live.
- [ ] Rollback path rehearsed: `git revert` the tag bump → re-sync; know that data-migration side
      effects are **not** reverted by git.
- [ ] Production change had the extra required review; blast radius understood.

---

## Common mistakes

**Terraform / IaC**
- Referencing modules by local path from a live repo → an in-flight edit mutates every environment's
  plan with no version gate.
- Faking environments with `terraform workspace` or `count` conditionals instead of separate
  directories/states → prod and dev share code paths and drift silently.
- Floor-only (`>=`) provider pins → an unrelated PR silently upgrades a provider mid-plan.
- Refactoring resource addresses without `moved {}` → destroy/recreate in every consumer.
- Hand-typed names/tags → un-attributable, drifting resources.
- Standing admin on the automation role instead of scoped, MFA-gated assume-role per account.

**Networking**
- A single shared NAT gateway (SPOF + a moving egress IP that breaks partner allowlists).
- Leaving the default SG/route-table/NACL managed; opening DB subnets to the internet by default.
- Hard-coding one instance architecture so arm/GPU nodes break.

**Kubernetes / Helm / GitOps**
- Umbrella chart or one enormous values file → one service's change redeploys everything.
- Floating chart versions → a re-sync silently pulls a breaking chart.
- Auto-sync + prune while *also* managing `replicas` in git for an autoscaled workload → ArgoCD and
  the HPA thrash pods.
- `nodeSelector` without a matching `toleration` → pods never schedule.
- Rolling back with `kubectl rollout undo` on a cluster ArgoCD immediately re-syncs (fighting GitOps).

**Data / secrets**
- Hard-wiring a shared "god" connection string instead of instance→inventory→attachment with
  per-consumer roles.
- Reusing an evicting cache (`allkeys-lru`) for sessions/queues; treating a snapshot-less cache as
  durable; mistaking multi-AZ for a backup.
- Shipping an admin search key to the browser instead of the search-only key.
- Committing plaintext secrets or baking them into image layers / chart values.

**Delivery**
- Copying whole source before installing deps (busts the cache); running as root; floating base tags.
- Writing build cache from every branch (poisons trunk's baseline); rebuilding images that already
  exist for the same input hash.
- Reusing a single `latest` tag → can't tell or roll back what's running.
- Unconditional migrations on every pod start; coupling a feature toggle to an image rebuild.
- One probe for both liveness and readiness; no startup probe; no grace period/PDB → dropped
  requests and crash loops.

## Production best practices (the short list)

Immutable content-addressed images · one image, config injected at runtime (12-factor) ·
dev↔prod parity through identical images/entrypoints · GitOps single source of truth with
revert-based rollback · atomic Helm releases · gated one-shot migrations · least-privilege OIDC/IRSA
short-lived credentials · secrets in a store, never in git or layers · pin & digest everything ·
fail fast in review (schema/validation/assertions) · separate accounts/clusters per environment ·
tier backups to data value · alert on heartbeats (silent cron death) not just uptime · scope the
blast radius and the re-render/apply scope · design for cost (shared ALBs, sleeping ephemeral envs,
disciplined caching).
