# ADRs — Architecture Decision Records

ADRs document important architectural decisions. Each file = one decision.

## How to create an ADR

1. Copy `_template-adr.md`
2. Name it `ADR-NNN-short-title.md` (e.g.: `ADR-001-message-broker.md`)
3. Fill it in completely — especially the evaluated alternatives
4. Once accepted, the status is **permanent** (it is not deleted, it is "Superseded" by another ADR)

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
| [ADR-005](records/ADR-005-language-per-service.md) | Language per Service: Java 21 and Spring Boot 3.5 | Accepted | 2026-09-28 |
| [ADR-006](records/ADR-006-database-engine-per-domain.md) | Database Engine per Domain: PostgreSQL ×7, MongoDB for notifications | Accepted | 2026-09-28 |
| [ADR-007](records/ADR-007-migration-tool-per-domain.md) | Migration Tool per Domain: Liquibase | Accepted | 2026-09-28 |
| [ADR-008](records/ADR-008-interface-framework.md) | Interface Framework: React (React Native for mobile) | Proposed | 2026-09-28 |
| [ADR-009](records/ADR-009-saga-state-store.md) | Saga State Store: PostgreSQL owned by the workflow | Proposed | 2026-09-28 |

> Add rows here as you create ADRs in `records/`
