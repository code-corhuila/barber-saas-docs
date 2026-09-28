# ADR-007 — Migration Tool per Domain: Liquibase for All Eight Domains

- **ID:** ADR-007
- **Date:** 2026-09-28
- **Status:** Accepted
- **Authors:** Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia

---

## Context

The course norm (4.2.2) requires choosing, per domain, Liquibase or Flyway with PostgreSQL, and
Liquibase with MongoDB. ADR-006 puts `notifications` on MongoDB. Every `-db` repository must prove in
CI that its schema is built from an empty database, fully rolled back and re-applied (norm 5.2.8).

---

## Decision

**We decided:** all eight `-db` repositories use **Liquibase** (version pinned), with
`changelog/changelog-master.yaml` as the single entry point and one `changelog.yaml` per folder
(annex A); `notifications` adds the MongoDB extension (`liquibase-mongodb` and `mongodb`, annex B).

---

## Evaluated alternatives (options)

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| **Liquibase in all eight (chosen)** | rollback declared per changeset and run by the tool (`rollback-count`); the only option for MongoDB; one tool for the whole platform | one `changelog.yaml` per folder; strict YAML (`comment` always quoted) | — (chosen) |
| Flyway for PostgreSQL + Liquibase for MongoDB | Flyway is simpler to read | two tools; Flyway Community does not roll back: a hand-written `U` script per migration | two tools and manual rollbacks |
| Flyway in all eight | simplest naming | not allowed with MongoDB (norm 4.2.2) | incompatible with ADR-006 |

---

## Dominant criterion

**Rollback verifiable in CI without hand-written scripts**, with a single tool for the whole
platform (MongoDB requires Liquibase anyway).

## Accepted cost

More configuration files (a changelog per folder), a stricter YAML syntax, and a custom Liquibase
image for the MongoDB domain.

---

## Consequences

- `db-ci.yml` in each `-db`: `update` → `update` (zero changesets) → `rollback-count 999` → `update`.
- Every changeset declares its rollback; an applied changeset is never edited (norm 5.2.3).
- `labels` in each changeset link the user story; `author` is the GitHub user.

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Unquoted `comment` breaks the changelog | Medium | Low | rule in review; `db-ci.yml` fails fast |
| Only one of the two MongoDB packages installed | Low | Medium | `lpm add liquibase-mongodb mongodb --global` in `deploy/liquibase.Dockerfile` |

## References

- Course norm 4.2.2, 5.2.3, 5.2.8, annexes A, B and G
- Related to: ADR-006
