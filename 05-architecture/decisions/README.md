# ADRs — Architecture Decision Records

ADRs document important architectural decisions. Each file = one decision.

## How to create an ADR

1. Copy `_template-adr.md`
2. Name it with the next free number and a short English title: `ADR-NNN-short-title.md`
   (e.g. the next one is `ADR-011-…`; check the register below first — numbers are never reused)
3. Fill in the sections the course norm requires (4.2.3): Context, Options, Dominant criterion,
   Accepted cost, Consequences — see `00-governance/documentation-rules.md` § "ADR format"
4. Open it through a `docs/NNN-slug` Pull Request and add its row to the register in the same PR
5. Once accepted, the status is **permanent** (it is not deleted, it is "Superseded" by another ADR)

## Possible statuses

- `Proposed` — under discussion
- `Accepted` — approved by the team
- `Rejected` — evaluated and discarded (document why)
- `Superseded` — superseded by ADR-NNN (indicate which one)

## ADR register

| # | Title | Status | Date |
|---|-------|--------|------|
| [ADR-001](records/ADR-001-idioma-documentacion.md) | Documentation Language (English for code and docs) | Accepted | 2026-08-20 |
| [ADR-002](records/ADR-002-modular-monolith.md) | Architectural Style: Modular Monolith | Superseded by ADR-004 | 2026-08-31 |
| [ADR-003](records/ADR-003-academic-microservice-extraction.md) | Academic Microservice Extraction: notification-service | Superseded by ADR-004 | 2026-09-03 |
| [ADR-004](records/ADR-004-full-microservice-decomposition.md) | Full Microservice Decomposition (29 repositories) | Accepted | 2026-09-20 |
| [ADR-005](records/ADR-005-language-per-service.md) | Language per Service: Java 21 and Spring Boot 3.5 | Superseded by ADR-012 | 2026-09-28 |
| [ADR-006](records/ADR-006-database-engine-per-domain.md) | Database Engine per Domain: PostgreSQL ×7, MongoDB for notifications | Accepted — instance topology superseded by ADR-011 | 2026-09-28 |
| [ADR-007](records/ADR-007-migration-tool-per-domain.md) | Migration Tool per Domain: Liquibase | Accepted | 2026-09-28 |
| [ADR-008](records/ADR-008-interface-framework.md) | Interface Framework: React (React Native for mobile) | Superseded by ADR-013 | 2026-09-28 |
| [ADR-009](records/ADR-009-saga-state-store.md) | Saga State Store: PostgreSQL owned by the workflow | Proposed | 2026-09-28 |
| [ADR-010](records/ADR-010-data-conventions-per-domain.md) | Data Conventions per Domain: UUID ids, money in cents, no cross-domain FK | Accepted | 2026-09-28 |
| [ADR-011](records/ADR-011-single-instance-per-engine.md) | Single Database Instance per Engine, One Schema per Domain (Annex J) | Accepted | 2026-10-02 |
| [ADR-012](records/ADR-012-two-backend-languages.md) | Two Backend Languages: Java 21 for eight services, Python for notifications-api and worker | Accepted | 2026-10-02 |
| [ADR-013](records/ADR-013-hybrid-mobile-interface.md) | Interface: one hybrid mobile app (Ionic + Capacitor), Angular shell, four Angular and four React domain apps | Accepted | 2026-10-02 |

## Decisions the course norm requires

| Required decision (norm) | Recorded in | Status |
|---|---|---|
| Topology beyond the mandatory repositories (4.2.1) | ADR-004 — no extra repository: exactly the 29 mandatory ones | Accepted |
| Language of each service (4.2.2; Annex J: two or more) | ADR-012 (supersedes ADR-005) | Accepted |
| Database engine of each domain (4.2.2) | ADR-006 | Accepted |
| Database topology: one instance per engine, one schema per domain (Annex J) | ADR-011 | Accepted |
| Migration tool of each domain (4.2.2) | ADR-007 | Accepted |
| Interface framework (4.2.2; Annex J: React and Angular) | ADR-013 (supersedes ADR-008) | Accepted |
| Where saga state is persisted (5.8.4) | ADR-009 | Proposed — awaiting the teacher |
| Identifier type and money representation (5.3.5) | ADR-010 — UUID for every entity, `bigint` cents | Accepted |

> The identifier type is **not** a separate ADR: it is decided in ADR-010 together with money and
> cross-domain references, because the three change the same tables. ADR-005 is the first language
> decision; its number is not free.
