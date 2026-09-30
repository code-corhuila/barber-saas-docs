# Per-Microservice Documentation Standard

> Defines exactly what documents each microservice must have, who writes them,
> when they are created, and when they must be updated. Non-compliance blocks the merge.

---

## Scope — the 29 repositories

The system is built as a polyrepo of 29 mandatory code repositories (norm 4.1: 5 + N × (2 + C),
with N = 8 domains and C = 1 channel): five cross-cutting repositories (`barber-saas-api-gateway`,
`-worker`, `-workflow`, `-infra`, `-front`) and three per domain (`barber-saas-<domain>-db`, `-api`,
`-app`). The documents below live in this repository; each code repository also follows the annex of
the course norm for its type and has a `README.md` with its purpose, how to run it and what it
depends on (norm 5.1).

| Repository type | Course norm annex |
|---|---|
| `-db` | A (PostgreSQL) or B (MongoDB) |
| `-api` | C (hexagonal service) |
| `-worker` | D |
| `-workflow` | E (sagas) |
| `-api-gateway` | F |
| `-infra` | G |
| `-front`, `-app` | H (the `-app` repositories apply it to the mobile channel) |
| all code repositories | I (CODEOWNERS, PR template, CI, board tracking) |

---

## The eight domain services

The folder of each service is named after its domain, exactly as in its repositories
(`barber-saas-<domain>-api`). State on 2026-09-30:

| Domain | Repositories | Contract (`07-api/contracts/openapi/`) | Data model | Docs folder `09-microservices/services/<domain>/` |
|---|---|---|---|---|
| identity-auth | `-db`, `-api`, `-app` | `auth-service.yaml` ✅ | `06-data/models.md` §2 ✅ | ❌ not created |
| barbershop | `-db`, `-api`, `-app` | `barbershop-service.yaml` ✅ | §3 ✅ | ❌ not created |
| schedule | `-db`, `-api`, `-app` | `schedule-service.yaml` ✅ | §4 ✅ | ❌ not created |
| appointment | `-db`, `-api`, `-app` | `appointment-service.yaml` ✅ | §5 ✅ | ❌ not created |
| loyalty | `-db`, `-api`, `-app` | `loyalty-service.yaml` ✅ | §6 ✅ | ❌ not created |
| notifications | `-db`, `-api`, `-app` | `notification-service.yaml` ✅ | §7 (MongoDB) ✅ | ❌ — `services/notification/` is the prototype's ADR-003 extraction (superseded), not this service |
| finance-inventory | `-db`, `-api`, `-app` | `finance-inventory-service.yaml` ✅ | §8 ✅ | ❌ not created |
| platform-admin | `-db`, `-api`, `-app` | `platform-admin-service.yaml` ✅ | §9 ✅ | ❌ not created |

`services/02-auth-service/` is the governance framework's worked example (it says so in its
header), not identity-auth. Until a domain gets its folder, its contract and its section of
`06-data/models.md` are its documentation; the folder becomes mandatory with the first story that
implements the domain (README) and before its first migration (`data-model.md`).

The cross-cutting repositories (`-api-gateway`, `-workflow`, `-worker`, `-infra`, `-front`) are
documented in `05-architecture/overview.md` §4.2 and `05-architecture/deployment.md`, not here.

---

## Required structure for each service

Each domain service lives in `09-microservices/services/<domain>/` and MUST have:

```
09-microservices/services/<domain>/
├── README.md         ⭐ REQUIRED from Sprint 1
├── data-model.md     ⭐ REQUIRED before creating migrations
├── events.md         ⭐ REQUIRED if the service emits/consumes events
├── decisions.md      🔵 RECOMMENDED — internal technical decisions of the service
└── runbook.md        🟢 REQUIRED before first deploy to qa
```

And its OpenAPI contract in:
```
07-api/contracts/openapi/service-name.yaml   ⭐ REQUIRED if it exposes REST endpoints
```

---

## README.md — Service technical sheet

**When to create it:** At the start of the sprint where the service is created
**Owner:** Developer assigned to the service
**Update when:** Responsibility, ports, or dependencies between services change

Minimum content (use `_template/service/README.md`):

| Section | What it must say |
|---------|-----------------|
| Responsibility | One sentence: what it does and what data it is the authoritative owner of |
| Architecture location | Port, repository, DB engine, who it communicates with |
| Responsibilities (what it DOES) | List of concrete responsibilities |
| Out of scope (what it does NOT do) | What it delegated and to whom |
| How to run locally | Exact commands, must work |
| Related documents | Links to the other files of the service |

---

## data-model.md — Service data model

**When to create it:** Before the first migration script
**Owner:** Developer assigned to the service
**Update when:** A table/collection is created or modified

Minimum content:
- ER diagram (Mermaid) of the service's tables
- Description of each table with its columns, types, constraints, and purpose
- The database engine (PostgreSQL or MongoDB) and its ADR (norm 4.2.2)
- The migration tool (Liquibase or Flyway with PostgreSQL; Liquibase with MongoDB) and its ADR — the schema lives only in the domain's `-db` repository, never in the `-api` (norm 5.2.1)

**Rule:** A field whose reason for existing is not obvious MUST have a comment in the diagram.

---

## events.md — Service event catalog

**When to create it:** When the service publishes or consumes its first domain event
**Owner:** Developer assigned to the service
**Update when:** An event is added, modified, or removed

Minimum content:
- Table of published events: name, topic/exchange, when it is emitted, schema
- Table of consumed events: name, which service it comes from, what action it triggers
- Payload schema (can reference the OpenAPI or Avro contract)

See standard event structure: `02-domain/domain-events.md`

---

## decisions.md — Service technical decisions

**When to create it:** When the team makes a non-obvious technical decision about the service
**Owner:** Whoever made the decision
**Update when:** A new decision is made or a previous one is revoked

Recommended format: miniADR (without the full rigor of an architecture ADR):
```markdown
### Decision: [short name]
**Date:** [date]
**Context:** [what problem was being solved]
**Decision:** [what was decided]
**Consequences:** [known trade-offs]
```

---

## runbook.md — Service operations manual

**When to create it:** Before the first deploy to `qa`
**Owner:** Responsible developer + DevOps
**Update when:** A new operational issue is discovered or a procedure changes

Minimum content:
- How to verify the service is healthy (health check, key metrics)
- Known symptoms and their causes: "If you see X, the problem is Y, the solution is Z"
- How to perform a service rollback
- How to run DB migrations
- Configured alerts and what to do when they fire

---

## OpenAPI Contract

**When to create it:** Before implementing the service's first endpoint (API-first)
**Owner:** Developer assigned to the service
**Update when:** An endpoint is added, modified, or removed

**API-First Rule:** The contract is written BEFORE the code. Contract tests validate
that the code fulfills the contract, not the other way around.

Use the template: `07-api/contracts/openapi/_template-service.yaml`

---

## How to add a new microservice

The same steps apply when one of the eight domains gets its docs folder for the first time.

1. If it is not one of the 29 mandatory repositories, record an ADR **before** creating it (norm 4.2.1)
2. Copy `09-microservices/_template/service/` → `09-microservices/services/<domain>/`
3. Update `09-microservices/service-catalog.md` with the service's entry — the catalog still
   describes the prototype's modules (`05-architecture/overview.md`, AT-006), so the first domain to
   do this also replaces the monolith description
4. Update `09-microservices/dependency-map.md` (it does not exist yet; until then the dependencies
   are in `05-architecture/overview.md` §3 and in the index of `07-api/README.md`)
5. Copy `07-api/contracts/openapi/_template-service.yaml` → `07-api/contracts/openapi/<domain>-service.yaml`
   and register it in the index of `07-api/README.md`
6. Create a PR with at least the README.md and the sketched API contract

---

## Correlations

- Service template → `09-microservices/_template/service/`
- Service catalog → `09-microservices/service-catalog.md`
- API contracts → `07-api/contracts/openapi/`
- General documentation rules → `00-governance/documentation-rules.md`
