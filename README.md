# agent-skills

Reusable [Agent Skills](https://www.skills.sh) for AI coding agents (Claude Code, Cursor, and any
tool that reads `SKILL.md` files).

Each skill lives in its own directory under [`skills/`](skills/) with a `SKILL.md` entry point and
optional `references/` for progressive disclosure.

## Skills

| Skill | What it does |
|-------|--------------|
| [`django-backend-engineering`](skills/django-backend-engineering/SKILL.md) | Engineering standards for building Django/DRF backend services — a multi-tenant class-registry (factory) architecture, a validate→persist→publish pipeline, Keycloak-based authentication and a principal/permission model, structured Celery tasks, PostgreSQL-native upserts, structured logging, and safe migrations. |
| [`production-safe-django-migrations`](skills/production-safe-django-migrations/SKILL.md) | Writing zero-downtime Django/PostgreSQL migrations — a timeout-aware migration base class, concurrent indexes, lock/statement timeouts, NOT VALID→VALIDATE, expand/contract multi-release changes, batched backfills, a per-operation lock reference, and checklists. |
| [`production-react-engineering`](skills/production-react-engineering/SKILL.md) | Building production React frontends — a multi-brand layered structure, CSS-Module component primitives, a custom SSR router, plain Redux + a custom data-fetcher framework (instead of React Query), Keycloak-via-BFF-cookie auth, a Redux-integrated search abstraction, Formik+Yup forms, lightweight charts, code splitting, checklists, anti-patterns, and snippets. |
| [`production-infrastructure-devops`](skills/production-infrastructure-devops/SKILL.md) | Running production infrastructure and delivery — Terraform (versioned modules consumed by PR-automated live configs), AWS networking/VPC/EKS/IAM, Kubernetes via Helm + Helmfile, GitOps with ArgoCD, managed data services wired in via an instance→inventory→attachment pattern, secrets via a parameter store + External Secrets, observability & DR, plus multi-stage Docker images, cached CI/CD, content-addressed image tags, GitOps rolling deploys, and revert-based rollback. |
| [`production-testing`](skills/production-testing/SKILL.md) | Writing and verifying tests — pytest + DRF (real-DB unit/integration/API tests, factories/fixtures, boundary-only mocking, snapshots, coverage) and Jest + React Testing Library (provider-wrapped renders, restorable config mocks), an E2E suite with layered page objects and DB-backed flakiness quarantine, contract & performance testing, when-to-mock guidance, and a mandatory run→report-bug→rerun verification loop. |

## Get them on Agensi

All five skills are also listed on [Agensi](https://www.agensi.io/creators/usman-asif), free to download:

[![Django Backend Engineering Standards on Agensi](https://img.shields.io/badge/Agensi-Django%20Backend%20Engineering%20Standards-1f6feb)](https://www.agensi.io/skills/django-backend-engineering-standards)  
[![Zero-Downtime Django Migrations on Agensi](https://img.shields.io/badge/Agensi-Zero--Downtime%20Django%20Migrations-1f6feb)](https://www.agensi.io/skills/zero-downtime-django-migrations)  
[![Production React Engineering on Agensi](https://img.shields.io/badge/Agensi-Production%20React%20Engineering-1f6feb)](https://www.agensi.io/skills/production-react-engineering)  
[![Production Infrastructure & DevOps on Agensi](https://img.shields.io/badge/Agensi-Production%20Infrastructure%20%26%20DevOps-1f6feb)](https://www.agensi.io/skills/production-infrastructure-devops)  
[![Production Testing for Django and React on Agensi](https://img.shields.io/badge/Agensi-Production%20Testing%20for%20Django%20and%20React-1f6feb)](https://www.agensi.io/skills/production-testing-for-django-and-react)

## Install

Using the [`skills` CLI](https://github.com/vercel-labs/skills):

```bash
# install every skill in this repo
npx skills add usmanasifbutt/agent-skills

# or a single skill
npx skills add usmanasifbutt/agent-skills/django-backend-engineering
```

Or install manually by copying a skill directory into your agent's skills folder
(e.g. `.claude/skills/` for Claude Code).

## Layout

```
skills.sh.json                       # repo page / grouping config for skills.sh
skills/
  django-backend-engineering/
    SKILL.md                         # entry point (name + description frontmatter)
    references/                      # detailed topic docs, loaded on demand
```

## Authoring notes

- Every skill directory contains a `SKILL.md` with YAML frontmatter (`name`, `description`).
  The `description` is what an agent matches against to decide when to load the skill — keep it
  specific about *when* to use it.
- Keep `SKILL.md` as a concise router; push depth into `references/*.md` so agents only pull in
  what they need.
