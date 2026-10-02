# Migration Work Split — Prototype to the 29 Repositories

> **What is this?** Who builds which repository while the prototype (`code-corhuila/barber-saas`)
> is migrated to the polyrepo, and the rules that keep three developers from blocking or
> overwriting each other. Decided by the team on 2026-10-02.
>
> **Architecture it follows:** ADR-004 (29 repositories), ADR-011 (one database instance per
> engine, one schema per domain), ADR-012 (Java 21 for eight services, Python for
> `notifications-api` and `worker`), ADR-013 (one hybrid Ionic + Capacitor app, Angular shell,
> four Angular and four React domain apps), course norm 2026-B and its Annex J.

---

## 1. Owners

Every repository has **one owner**. Only the owner opens pull requests on it, except for the three
shared points of §3.

| Developer | GitHub | Phase 1 — first functional app | Phase 2 |
|---|---|---|---|
| Daniel Felipe Cerquera Idrobo | `Pipecerquera` | `infra` (single PostgreSQL), `identity-auth-db`, `identity-auth-api`, `identity-auth-app` (Ionic React), `api-gateway`, `front` (Angular shell) | `worker` (Python), `platform-admin-db/api/app` (Ionic Angular), `workflow` |
| Carlos Mauricio Leal Medina | `carlosleal16` | `barbershop-db/api/app` (Ionic React), `schedule-db/api/app` (Ionic React) | `notifications-db` (MongoDB), `notifications-api` (Python, FastAPI), `notifications-app` (Ionic Angular) |
| Juan Pablo Borrero Morales | `JUANDAX233` | `appointment-db/api/app` (Ionic React) | `loyalty-db/api/app`, `finance-inventory-db/api/app` (Ionic Angular) |

**Phase 1 is the vertical slice** the app needs to work end to end: register and log in, browse
barbershops and services, see a barber's availability, and book, list and cancel an appointment.
Phase 2 adds the remaining domains one at a time.

### Phase 1 order and dependencies

| Step | Who | Delivers | Unblocks |
|---|---|---|---|
| 1 | Daniel | `infra` with the single PostgreSQL instance, `identity-auth` db + api (**done 2026-10-02**) | Everyone can run the platform and get tokens |
| 2 | Carlos, Juan Pablo | Their `-db` schemas and the core (domain, use cases, tests) of their `-api` — needs nobody | — |
| 3 | Daniel | `api-gateway` with its routes, `front` shell with `shell/apiClient` and the mount contract for React apps | Domain apps can be mounted |
| 4 | Carlos | `barbershop-api` and `schedule-api` HTTP endpoints (catalog, availability) | `appointment-api` can check price, duration and slots |
| 5 | Juan Pablo | `appointment-api` HTTP endpoints, calling barbershop and schedule with a service token | Booking works |
| 6 | Each owner | Their Ionic React domain app, mounted by the shell | The app works |

---

## 2. Reference implementation

`barber-saas-infra`, `barber-saas-identity-auth-db` and `barber-saas-identity-auth-api` (on
`develop`) are the first repositories built under the norm and Annex J. **Copy their shape**:

- `-db`: Liquibase layout of annex A, one changeset per object with its rollback, `deploy/compose.yml`
  with **only** the migration runner and its own changelog tables
  (`databasechangelog_<domain>`), `03_dcl` granting `<domain>_writer` to `<domain>_app`, and
  `db-ci.yml` (build from empty, second run applies nothing, full rollback, apply again).
- `-api` (Java): three Maven modules — `<domain>-core` (no Spring), `<domain>-adapters`,
  `<domain>-app` — package `co.edu.corhuila.barbersaas.<domain>`; the shared error envelope,
  `X-Correlation-Id`, RS256 validation in every service, `Idempotency-Key` on creations, explicit
  pool limits, `ci.yml` with `mvn -B verify`; it connects as `<domain>_app`, never as the
  administrator, and never runs migrations.
- Models and contracts come from `06-data/models.md` and `07-api/contracts/openapi/`; if the code
  needs something different, the document is changed in the same week through a `docs/` branch.

---

## 3. Shared points

Three repositories are touched by more than one developer. Each has a rule that avoids conflicts:

| Repository | What others change | Rule |
|---|---|---|
| `barber-saas-infra` | One `include` line per repository in `compose.yml` | A small pull request with **only** that line, opened right after `git pull` |
| `barber-saas-api-gateway` | `nginx/routes/<domain>.conf` | One file per domain: nobody edits another domain's file |
| `barber-saas-front` | One entry in `federation.manifest.json` and one route in the shell | A small pull request with only that entry, right after `git pull` |

After merging into a shared repository, the author tells the team so everyone pulls.

---

## 4. Daily workflow (every repository)

```bash
git checkout develop && git pull                 # always start from the updated develop
git checkout -b feat/<short-description>        # one child branch per piece of work
# … one commit per logical step …
git push -u origin feat/<short-description>
gh pr create --base develop                     # body: Refs: code-corhuila/barber-saas-docs#NN
# merge into develop when CI is green, keeping the commits (merge, not squash)
git checkout develop && git pull
```

- **Never** commit directly to `develop`, `qa` or `main` (norm 6.2.2); `develop` only accepts pull
  requests, which the team merges without the teacher.
- **Many small commits.** One commit per logical step — the aggregate, its tests, the use case, the
  adapter, the endpoint — never everything at once. Format of norm 8:
  `feat(appointment): add the booking use case` (types `feat|fix|docs|style|refactor|test|chore|perf`,
  lowercase, imperative, no final period).
- **Small pull requests:** at most 400 changed lines without tests; a feature is split into several
  pull requests, each with several commits.
- **User stories:** HU-AUTH-001 `#3`, HU-APPT-001 `#4`, HU-TENANT-001 `#13`, and the story of each
  domain in this repository.
- `CODEOWNERS` is never touched (norm 13.7). Everything written in a repository is in English;
  only the text a user sees on screen is in Spanish (ADR-001).

---

## 5. Definition of done per repository

- `-db`: `db-ci.yml` green; Annex J's query (J.10) shows `<domain>_app` writing only to its schema.
- `-api`: `ci.yml` green; every operation of its contract answers with the shared envelope; a
  cross-tenant test (HU-TENANT-001); it starts with `./scripts/up.sh dev` from `barber-saas-infra`.
- `-app`: mounted by the shell; uses only `shell/apiClient`; every view has its four states
  (loading, error with retry, empty, data).
- README answers: what it is, how to start it, where the data is, how to test it, what is missing.
