# ADR-009 — Saga State Store: PostgreSQL Owned by the Workflow

- **ID:** ADR-009
- **Date:** 2026-09-28
- **Status:** Proposed — pending the teacher's answer (see "Open question")
- **Authors:** Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia

---

## Context

`barber-saas-workflow` orchestrates processes that cross domains (sagas) and must persist the state
of every saga after each step (norm 5.8, annex E). The topology gives no `-db` repository to
cross-cutting services, so where saga state lives in production is a team decision recorded in an ADR
(norm 5.8.4). An in-memory store only serves tests: it does not survive a restart.

## Open question (to the teacher)

"For the saga state, may the `-workflow` repository version its own schema (with Liquibase) and run
its own PostgreSQL instance composed by `-infra`, or should that schema live in an additional
repository `barber-saas-workflow-db`, registered with an ADR as required by norm 4.2.1?"

---

## Decision (proposed)

**We propose:** a **PostgreSQL instance owned by the workflow**, composed by `-infra`, with the
saga schema versioned with Liquibase (ADR-007). The state is written in the same transaction after
each step and on each compensation.

---

## Evaluated alternatives (options)

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| **PostgreSQL owned by the workflow (proposed)** | durable and transactional; same engine, tool and CI as the domains | one more instance; where its schema lives needs the teacher's confirmation | — (proposed) |
| Redis with persistence (teacher's support material) | simple and fast | durability depends on configuration; no transactions with the saga logic | weaker guarantees for compensation |
| Camunda 7 process engine | real BPMN orchestration | a large new component seven weeks before the end; needs its own database | cost and time |

---

## Dominant criterion

**Transactional durability of the saga state after every step**, with the same engine, migration
tool and CI already chosen for the domains.

## Accepted cost

One more PostgreSQL instance on the platform, and either a justified exception (schema inside
`-workflow`) or one additional repository with its ADR — whichever the teacher accepts.

---

## Consequences

- The workflow can resume or compensate a saga after a restart; a failed compensation leaves the saga `FAILED` for a person to decide (annex E).
- Each saga must be documented as a process flow (`16-bpmn`, to be created).
- On acceptance: status → Accepted; if an additional repository is required, a new ADR is written first (4.2.1).

## References

- Course norm 4.2.1, 5.8, 5.8.4, annex E
- Related to: ADR-004, ADR-006, ADR-007
