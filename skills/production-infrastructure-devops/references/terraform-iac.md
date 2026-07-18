# Terraform & Infrastructure-as-Code

## Repo topology — library vs. wiring vs. bootstrap

Three cleanly-layered repos/areas:

1. **Modules library** (reusable) — provider wrappers + composite "high-level" modules. **No state,
   no accounts, no environment values.** Pure library, released on its own cadence, semver-tagged.
2. **Live / root-config repos** — thin root modules (one directory = one Terraform state) that
   consume versioned modules and wire in real account/environment values. Driven by PR automation.
3. **`setup/` bootstrap** — a tiny project (with **local** state, committed) that creates the state
   backend itself (the bucket + lock table), solving the chicken-and-egg problem. Keep it
   secret-free.

- **Why:** modules change on a library release cadence (tested, pinned); live config changes
  constantly and must be safe to auto-apply. Separation lets many consumers depend on a stable,
  versioned library.
- **Tradeoff:** every module change needs a release + a version bump in consumers (more PRs, slower
  propagation) — accepted for reproducibility and blast-radius control.
- **Common mistake:** reusable modules in the same repo/branch as live state, referenced by relative
  path — an in-flight edit instantly mutates every environment's plan with no version gate.

**Consumption is always by pinned git ref:**
```hcl
module "network" {
  source = "git::git@github.com:<org>/<modules-repo>.git//platform/network?ref=v<MAJOR>.<MINOR>.<PATCH>"
}
```
Different root modules may sit on different tags, so upgrades roll out incrementally.

**Folder hierarchy encodes isolation** — the path *is* the unit of state/governance:
```
<account-or-tenant>/<environment>/<component>/     # e.g. tenant-a/production/network/
master/ (org/account management)   global/ or ops/ (shared/dev services)   setup/ (backend bootstrap)
```
The modules library is organized **provider-first, then resource** (`aws/`, one dir per SaaS
provider, a `platform/` namespace for composite modules, a `null/` namespace for pure computation).

**Common mistake:** encoding environment differences inside one directory with `count`/conditionals
instead of separate directories — it couples dev and prod into one state and plan.

## PR-automated workflow (Atlantis or equivalent)

A repo-root metadata file customizes auto-discovered projects:
- Enable `parallel_plan`/`parallel_apply`, `delete_source_branch_on_merge`.
- **Disable autoplan on nested `modules/`** (they aren't projects) and on accounts the automation
  identity can't reach.
- Map path globs → **owning teams** so only that team can apply a given project; a catch-all maps to
  the platform team.
- Tune `parallelism` per project (serialize a provider with a concurrent-update bug; crank
  config-only projects that are I/O-bound on API calls).
- `when_modified` so a plan also fires when a non-`.tf` data file changes.

**Apply is gated** by branch protection + CODEOWNERS (devops on all `*.tf`, an extra reviewer group
on `**/production/**`) + human role membership; applying is a PR comment; success auto-merges.

- **Why:** every change is planned, reviewed, and applied through an auditable PR with team-scoped
  permissions — no laptop `terraform apply`.
- **Tradeoff:** path-regex governance is brittle — renaming a directory can silently drop its
  protection.
- **Common mistake:** autoplan running on nested `modules/` (phantom projects); config-only projects
  left at parallelism 1 so plans hang on serial API calls.

## State backend

- **Remote state with locking, one state per leaf directory**; the state **key mirrors the folder
  path** (directory layout *is* the state namespace).
- **Directories, not workspaces**, for environment/region/account separation.
- Chain projects via `terraform_remote_state`: a `base`/`accounts` project publishes an account
  registry (`id`, `region`, `cidr`) consumed everywhere to derive provider `assume_role` ARNs and
  CIDRs.
- **Common mistake:** one giant state per environment (slow plans, huge blast radius) — or the
  opposite, workspaces faking environments so prod/dev share code paths.

## Module design

Every module has a fixed skeleton: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`,
auto-generated `README.md` (via a docs pre-commit hook), plus topic-split files (`iam.tf`, `vpc.tf`,
`moves.tf` for `moved {}` blocks).

- **Small, composable modules with a high-level (HL) composition layer.** Low-level modules wrap one
  provider concept; HL modules *don't touch the cloud directly* — they call sub-modules (each pinned
  by ref). Reuse reputable upstream community modules for commodities (VPC, EKS, RDS, IAM, SGs, ACM).
- **One namespace per provider; wrap SaaS like cloud.** ~20 providers each wrapped so app teams see
  one uniform interface regardless of vendor.
- **Provider pinning:** floor+ceiling appropriate to stability; root configs pin tighter;
  `configuration_aliases` for required aliased providers.
- **Semver on the library:** patch = fixes only; minor = back-compat (breaking only with a migration
  file + guide); major = structural. Ship `moved {}` blocks so refactors are state-safe for
  consumers.

**Variable interface conventions:** every var typed, `description` prefixed `(Required)`/
`(Optional)`, almost always `nullable=false`; sane defaults; heavy `validation{}`; a `create`
boolean to toggle a whole module; structured object/map inputs with `try()`/`coalesce()` for
back-compat.

- **Common mistake:** unpinned/floor-only provider constraints; refactoring resource addresses
  without `moved {}` → destroy/recreate in every consumer.

## Derived identity — constants / attributes / computation modules

- A provider-less **`constants`** module exposes canonical maps/lists via outputs (env→short-name,
  the supported service-type list, code→name maps) so naming is centralized and typo-proof.
- An **`attributes`** module computes normalized identity (`app_name`, short names, cluster maps,
  `is_production` flag) from raw inputs, and runs **static validation** beyond a single
  `validation{}` block (e.g. `alltrue([... ? true : tobool("error msg")])` to fail fast on
  unsupported values).
- **`null/*`** pure-computation modules do side-effect-free math (CIDR/subnet decomposition, account
  decomposition, string templating) — independently testable, keeps arithmetic out of resource
  modules.

**Tagging & naming are derived, not typed:**
```hcl
tags = merge(var.base.tags, {
  MaintainedBy = "Terraform"
  Module       = "<namespace>/<module>"   # trace any cloud resource back to the module that made it
}, var.tags)
```
Names are composed centrally (`<app>-<environment>` / `<app>-<env-short>`); use a `name_prefix`
toggle to avoid create-before-destroy name clashes.

- **Common mistake:** hand-writing names/tags per resource (drift, un-attributable resources)
  instead of deriving them; nullable inputs that silently `coalesce` to empty names.

## IAM

- **Roles over users; assume-role for humans and automation.** A fixed role taxonomy
  (Guest/Developer/Operator/Terraform/Admin) trusts the org root and **enforces MFA on assume**. The
  automation identity assumes a dedicated `Terraform` role per target account.
- **OIDC/IRSA for in-cluster workloads** — each controller (autoscaler, LB controller, CSI drivers)
  gets a *scoped* role bound to a specific `namespace:serviceaccount` via the cluster OIDC provider.
  No node-wide shared credentials.
- **Least-privilege policy composition** with `aws_iam_policy_document`, gated by `count`, scoped to
  specific ARNs/regions/accounts rather than `*`.
- **Common mistake:** broad managed policies (or `Action:*`) on the node instance role instead of
  per-service OIDC roles; standing admin on the automation role.

## Environment separation & promotion

- **Per-account, per-environment, directory-based isolation** — each tenant has `global/ + staging/
  + production/` sibling dirs, each mapping to a distinct account via the registry. **Promotion =
  applying the same module version to the next environment's directory via a PR** (not copying and
  editing).
- A central **account registry** (from a YAML file + templates, since Terraform can't dynamically
  generate providers) drives provider targeting; adding an account = adding a YAML entry.
- **Production gets extra guardrails:** an `is_production` attribute raises HA/retention defaults;
  CODEOWNERS demands an extra reviewer on any `**/production/**` path.
- **Pre-commit as the local gate:** credential detection, `fmt`, docs, `tflint` (with a cloud
  ruleset) — mirroring CI.
- **Common mistake:** copy-pasting an environment directory to "promote" then editing it, so staging
  and prod drift into different module versions/inputs.
