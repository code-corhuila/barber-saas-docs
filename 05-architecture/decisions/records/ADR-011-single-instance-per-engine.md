# ADR-011 — Single Database Instance per Engine, One Schema per Domain

- **ID:** ADR-011
- **Date:** 2026-10-02
- **Status:** Accepted
- **Supersedes:** ADR-006 **in part** — only its instance topology ("each domain runs its own
  instance, defined in its `-db` repository"). The engine choice of ADR-006 stays in force.
- **Authors:** Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia

---

## Context

ADR-006 followed norm 7.1 as published: one database per domain, each in its own instance with its
own volume, defined in its `-db` repository. On 2026-10-01 the teacher issued **Annex J** of the
course norm, which prevails over the norm and over annexes A, B and G where it corrects them. It
states that the requirement was the opposite:

- There is **one database per engine, in one instance, with one volume**, defined in the
  infrastructure repository of that engine (`<abbr>-infra-postgres`, `<abbr>-infra-mongo`). No
  `-db` repository defines an instance (J.2, J.3.1).
- Each `-db` feeds that instance with the DDL, DML, DCL and TCL of its domain, **inside its own
  schema**; in MongoDB the equivalent of the schema is one database per domain (J.2).
- Each `-db` keeps, in `deploy/`, only the runner that applies its migrations, with **its own
  changelog table**: with one shared database, two `-db` repositories using the default Liquibase
  table collide and the second one fails (J.6).
- Each service connects with its domain's user, `<domain>_app`, never with the administrator (J.7).
- A domain writes only to its own schema; reading another domain's tables is accepted only as
  `SELECT`, with a grant issued by the owner's `-db` and a technical-debt ADR (J.3.2–J.3.3).
- Both engines are mandatory and at least one domain uses MongoDB (J.1.2).

ADR-006 already chose PostgreSQL for seven domains and MongoDB for `notifications`, and
`06-data/models.md` already gives every domain its own schema (`identity_auth`, `barbershop`, …).
What no longer holds is the topology: eight instances, eight volumes, a database service inside
each `-db`.

**Known constraints:**
- The 29 repositories are still empty (`README.md` and `CODEOWNERS` only), so no data and no
  deployed instance has to be moved.
- `barber-saas-infra-mongo` does not exist and `barber-saas-infra` keeps its name; creating and
  renaming infrastructure repositories is done by the teacher on request (J.8.3).

---

## Decision

> One PostgreSQL instance and one MongoDB instance for the whole platform; every domain lives in its
> own schema and connects with its own user.

**We decided:**

1. **PostgreSQL:** a single PostgreSQL 16 instance with one volume, defined in `barber-saas-infra`
   (to be renamed `barber-saas-infra-postgres` by the teacher). It holds seven schemas named after
   the domain, without suffixes: `identity_auth`, `barbershop`, `schedule`, `appointment`,
   `loyalty`, `finance_inventory`, `platform_admin` — the recommended form of J.4.
2. **MongoDB:** a single replica-set instance (rs0) with one volume, defined in
   `barber-saas-infra-mongo`, holding the `notifications` database.
3. **Roles and users per domain:** `<domain>_reader`, `<domain>_writer` and the login user
   `<domain>_app`, whose `search_path` points to its schema. The infrastructure creates the
   extensions (`pgcrypto`) and the `<domain>_app` users in an idempotent
   `postgres/init/01-instance.sh`; each `-db` grants `<domain>_writer` to `<domain>_app` in
   `03_dcl/`.
4. **`-db` repositories:** migrations plus a `deploy/compose.yml` with **only** the
   `<domain>-db-migrate` runner (no database service, no volume), pointing at the shared instance
   and using its own changelog tables, `databasechangelog_<domain>` and
   `databasechangeloglock_<domain>` (Liquibase, ADR-007).
5. **Environments:** each infrastructure repository versions `env/dev.env.example`,
   `env/qa.env.example` and `env/main.env.example`, each setting its own `COMPOSE_PROJECT_NAME`
   (`barber-saas-dev`, `barber-saas-qa`, `barber-saas-main`), so each environment has its own
   container and volume. In `qa` and `main` the volume is never removed and the database is
   backed up before migrating (J.5).
6. **Cross-domain reads:** none today. Data from another domain is obtained through its API or its
   events. Any future direct read needs a `GRANT <owner>_reader TO <domain>_app` in the owner's
   `-db` and its own technical-debt ADR (J.3.3).

**Justification:** Annex J makes this mandatory; the only real choice it leaves is the schema
form (J.4), and a schema per domain is the form our data model already uses.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **One instance per engine, one schema per domain (chosen)** | Required by Annex J; matches the schemas already in `06-data/models.md`; one PostgreSQL container instead of seven on a laptop | Every `-db` needs its own changelog table; a broken migration can affect the shared instance | — (chosen) |
| One instance per engine, everything in `public` with a domain prefix per table (`appointment_appointment`) | Accepted by J.4 for teams that already built it | `GRANT … ON ALL TABLES IN SCHEMA public` hands every domain's tables to everyone; permissions table by table; long names | We have no tables yet, and our model is already schema-per-domain |
| Keep one instance per domain (ADR-006 as written) | Physical isolation | Contradicts Annex J (J.3.1: no `-db` defines an instance); eight PostgreSQL containers in development | Forbidden by Annex J |

---

## Consequences

**Positive:**
- Development runs one PostgreSQL and one MongoDB container instead of eight database containers.
- The isolation the course evaluates moves from the network to **permissions**: a `<domain>_app`
  user can only write to its own schema, and Annex J's verification query (J.10) proves it.
- The data model in `06-data/models.md` and the ER diagrams in `08-diagrams/er/` stay valid: they
  were already drawn one schema at a time.

**Negative / Trade-offs:**
- One failing instance takes every domain's data down at the same time; the domains are no longer
  isolated against an engine crash, only against each other's writes.
- Every `-db` must configure its own changelog table names; forgetting it makes the second `-db`
  fail on its first run (J.6).
- Roles live at instance level, so every role name must start with the domain (`appointment_app`,
  never `app`), and adding a domain means editing `01-instance.sh` and running it once by hand in
  every environment that already exists (J.5.5).
- The engine is shared, so a heavy query in one domain can slow the others; there is no resource
  limit per schema.

**Impact on the system:**
- Affected repositories: the eight `-db`, the eight `-api`, `-workflow` (saga state, ADR-009),
  `barber-saas-infra`, and the requested `barber-saas-infra-mongo`.
- Documents that must be updated: `05-architecture/overview.md`, `05-architecture/deployment.md`,
  `06-data/models.md`, `06-data/migration-strategy.md`, `02-domain/domain-map.md`,
  `05-architecture/pattern-guide.md`, `08-diagrams/c4/c2-containers.md`, `CLAUDE.md`.
- ADR-009 (*Proposed*): if saga state stays in PostgreSQL, it becomes a `workflow` schema in the
  shared instance, not a separate instance.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A service connects as the administrator user | Medium | High | `DATABASE_URL` always uses `<domain>_app`; review checks every `.env.example` and `application.yml` |
| Two `-db` repositories share the default changelog table | Medium | Medium | `--database-changelog-table-name=databasechangelog_<domain>` in every runner; `db-ci.yml` runs it |
| A domain gains write access to another schema through a broad grant | Low | High | Grants only `<domain>_writer TO <domain>_app`; run the J.10 query before each cut |
| `docker compose down -v` wipes `qa` or `main` | Low | High | Forbidden in `qa` and `main` (J.5.4); back up before migrating |
| `barber-saas-infra-mongo` is not created in time | Medium | Medium | Request it from the teacher with an issue in this repository (J.8.3) |

---

## References

- Course norm 2026-B, **Annex J** (J.1.2, J.2–J.10), numerals 4.5.4, 5.2, 5.9, 7, 12 (rule 8), 13
- The teacher's repository templates for `infra` and `db-postgres` (annexes G and A), as corrected
  by Annex J
- Related to: ADR-004, ADR-006 (engine choice, still in force), ADR-007, ADR-009, ADR-010
- Supersedes: the instance topology of ADR-006
