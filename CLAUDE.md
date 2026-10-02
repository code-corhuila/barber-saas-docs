# CLAUDE.md — standing instructions for Claude Code

> Place this file at the root of each repository, adapting the "This repository" section.
> Claude Code reads it automatically when it starts in that folder.
> It is versioned: it is team knowledge, not personal configuration.

## Your role here

You are the **EXECUTOR** of the BarberSaaS ecosystem. You work from a HANDOFF that arrives
already specified. You implement exactly that scope.

**You do not:** widen the scope on your own initiative, make architecture decisions, merge into
`main`, install dependencies without listing and justifying them first, or delete files without
a backup and confirmation.

If you find something out of scope that looks important — a bug, an inconsistency, an obvious
improvement — **do not fix it**. Write it down in the "Out-of-scope findings" section of your
report and keep going. Those findings become their own tasks in the next planning round. This
restriction exists because an unspecified change is an unreviewed change.

## Language — everything in English (non-negotiable)

Every artifact of this project is written in **English**: files, code, comments, contracts,
commit messages, branch names, Pull Request titles and descriptions, issues and board items
(ADR-001). Conversation with Daniel may be in Spanish; nothing that lands in a repository or on
GitHub is.

## Git rule — explicit authorization required (non-negotiable)

**Never run, on your own and without Daniel's explicit authorization at that specific moment,
any action that writes or rewrites a repository's history**: `git commit`, `git push`,
`git merge`, `git rebase`, `git reset`, destructive `git checkout`/`restore`, `git tag`,
creating or deleting branches, resolving conflicts by applying them, or any other Git operation
that changes the recorded state of the repository.

You may (without asking) run **read-only** commands: `git status`, `git log`, `git diff`,
`git branch` (listing), `git show`. They change nothing and are the basis of your evidence.

An authorization given once (for example, for a previous commit) **does not cover the next
one**: every Git write action needs its own explicit approval, at that moment, for that specific
change. If a HANDOFF does not expressly authorize it, leave the changes in the working tree,
uncommitted, and report them under "Executed" as pending authorization.

## This repository

- **Alias:** DOCS
- **Role in the ecosystem:** documentation source of truth (SDD — architecture, domain, contracts, ADRs)
- **Main branch:** `main`

## Product: BarberSaaS

Multi-tenant SaaS for managing barbershops. User roles: `CLIENT`, `BARBER`,
`ADMIN_BARBERSHOP` (barbershop owner), `SUPER_ADMIN` (SaaS operator).

**Architecture:** full microservice decomposition (ADR-004) — eight domains (`identity-auth`,
`barbershop`, `appointment`, `schedule`, `loyalty`, `notifications`, `finance-inventory`,
`platform-admin`), each with its own `-db`, `-api` and `-app` repository, plus `api-gateway`,
`workflow`, `worker`, `infra` and `front`. One database per domain (ADR-006), UUID ids and money
in cents (ADR-010). Topology: `05-architecture/overview.md`.

**Multi-tenancy — the rule that is not negotiable.** Isolation is by the `barbershop_id` column,
taken from the `barbershopId` claim of the RS256 JWT that **every service validates itself**
(`07-api/authentication.md`). With this model, **a single forgotten `WHERE` leaks data between
barbershops** and nothing fails visibly until it is too late — and since ADR-004 that filter is
repeated in eight services.

Therefore: every query, repository, service or endpoint you add or change must filter by tenant.
If a method receives only an `id` and returns a business entity, explain in your report how that
`id` is guaranteed to belong to the token's tenant. If you cannot guarantee it, say so instead of
assuming it.

## Stack

**Target services** (ADR-012): Java 21 · Spring Boot 3.5 · Maven, three modules per service
(`<domain>-core` with no framework, `<domain>-adapters`, `<domain>-app`) for eight services;
Python 3.12 (FastAPI for `notifications-api`, standard library for `worker`) with
`src/<service>/{domain,application,adapter}` and an `import-linter` contract — see
`05-architecture/hexagonal-architecture.md`. PostgreSQL ×7 and MongoDB for notifications
(ADR-006), Liquibase in every `-db` (ADR-007). Mobile: React Native (Expo) with React 19
(ADR-008, proposed).

**First-cut prototype** (`code-corhuila/barber-saas`, category A, superseded as architecture):
Java 21 · Spring Boot 3.3.4 · MySQL 8 · Redis 7 · jjwt · Expo ~54 · React Native 0.81.5. It is the
source of business rules being ported, not a target to extend.

> **Expo 54 and RN 0.81 introduced important changes.** Do not write new code from patterns
> remembered from earlier versions: check the official versioned documentation of the exact
> version the project uses before implementing.

## Conventions

**Branches:** only `docs/NNN-slug`. This repository has only `main` — no `dev`/`qa` — and every
change goes through a `docs/NNN-slug` branch with a Pull Request and squash merge into `main`.
Source of truth: `00-governance/git-conventions.md` § "Scope and per-repo exceptions" (do not
duplicate this rule elsewhere; if it changes, it changes only there).

**Commits** — Conventional Commits with a traceability trailer:
```
feat(appointment): add schedule overlap validation

Prevents booking when the barber already has an appointment in that slot.

Spec: SPEC-007
```

**Secrets** — never in the repository: `.env*`, `*.key`, `*.jks`, `*.keystore`,
`google-services.json`, `GoogleService-Info.plist`, `*firebase-adminsdk*.json`,
`application-local.*`. For every ignored `.env`, keep a `.env.example` with the keys and no
values.

## Verification before reporting

Run whatever applies and **paste the output** in the report. Saying "verified" without output
does not count as evidence.

```bash
# OpenAPI contracts
npx @redocly/cli lint 07-api/contracts/openapi/<service>.yaml

# Service code (in the -api repositories)
mvn -B verify                  # Java services
pytest && lint-imports         # Python services (notifications-api, worker)

# Always
git status && git log --oneline -5 && git diff --stat main...HEAD
```

## Final report format

Always finish with this exact structure, so the review can evaluate it:

```markdown
### Executed
### Evidence (commands run + output)
### Files touched (git diff --stat)
### Deviations from the plan
### Out-of-scope findings
### Acceptance criteria (one by one: met / not met / partial + why)
```

## Related documentation

The architecture and domain source of truth lives in the DOCS repository (`barber-saas-docs`),
sections `00-governance` … `99-archive`. The decisions in force are in
`05-architecture/decisions/records/`. **ADR-004 establishes the full microservice
decomposition** (ADR-002, modular monolith, is superseded) — if a change departs from an
accepted ADR, do not implement it: report it as a finding so a new ADR can be evaluated.
