# ADR-002 — Architectural Style: Modular Monolith

| Field | Value |
|-------|-------|
| **ID** | ADR-002 |
| **Date** | 2026-08-31 |
| **Status** | Accepted |
| **Authors** | Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia |

---

## Context

BarberSaaS is a multi-tenant SaaS platform digitizing barbershop operations in Colombia
(appointment scheduling, staff management, loyalty programs, financial tracking, inventory,
and push notifications) across four user roles: `SUPER_ADMIN`, `ADMIN_BARBERSHOP`, `BARBER`,
and `CLIENT`.

At the start of implementation, the team had to decide how to structure the backend: as a
distributed set of microservices from day one, or as a single deployable unit organized by
bounded context, with the option to extract services later. This decision shapes team
velocity, operational cost, and the platform's ability to scale into the target market
(~15,000 barbershops in mid-sized Colombian cities, with a Year-1 target of 50 paying shops).

**Known constraints:**
- Team size: 1 developer for the initial build (single-developer bottleneck is a registered
  project risk)
- Current and near-term load: tens of barbershops, well under any threshold that would
  require independent scaling of individual capabilities
- Time-to-market pressure: 60-day free trial model requires a working MVP quickly
- Long-term ambition: the domain is expected to grow (multi-location chains, marketplace,
  standalone loyalty product), so the architecture must not preclude future decomposition

---

## Decision

**We decided:** Implement the backend as a **Modular Monolith** — a single Spring Boot 3 /
Java 21 deployable unit, internally organized into explicit bounded-context packages (auth,
barbershop, employee, appointment, schedule, loyalty, finance, inventory, notification,
plan, dashboard), each with its own controller, service, and DTOs, and no direct
cross-context imports except through domain entities and shared security utilities
(`TenantContext`).

**Justification:** At the current team size and load, the operational overhead of a
distributed system — service discovery, distributed tracing, versioned contracts, network
partial-failure handling — would consume all available engineering capacity without a
corresponding scalability payoff. A well-structured modular monolith extracts into services
faster than a poorly structured one, so enforcing bounded-context boundaries now keeps the
extraction path open while deferring its cost until a measured, production-observed trigger
condition justifies it (documented per module in Section "Planned evolution" of
`overview.md`).

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Modular Monolith (chosen)** | Single deployment (Railway, one JAR); no network hop between modules; fast local development and testing; in-process calls are simpler to reason about; extraction path is explicit and low-risk if bounded contexts are respected | Requires discipline to avoid contexts leaking into each other; shared PostgreSQL schema means DB-level isolation relies on `barbershop_id`, not physical separation | — (chosen) |
| Microservices from day one | Independent scaling and deployment per capability; natural team ownership boundaries at scale | Distributed-systems overhead (service discovery, distributed tracing, network failures, eventual consistency) is unsustainable for a 1-developer team; premature extraction was explicitly flagged as a project risk | Discarded — no current load or team-size justification; overhead would delay MVP delivery and the 60-day-trial go-to-market timeline |
| Traditional layered monolith (no bounded-context discipline) | Simplest possible starting point; fastest initial velocity | No clear extraction seams later; business logic tends to leak into controllers/services; harder to onboard a second developer or extract a service under load | Discarded — would create technical debt that contradicts the documented extraction roadmap and risks the "single developer bottleneck" mitigation plan, which depends on clear module boundaries |

---

## Consequences

**Positive:**
- Single deployable unit simplifies operations on Railway (one JAR, automatic TLS, daily
  backups) and keeps the 1-developer team's operational load manageable
- Explicit bounded-context packages document and enforce the domain boundaries that will
  become service boundaries later, without paying distributed-systems tax today
- In-process module communication (direct Java calls) gives low latency and simple
  debugging while the platform is small
- A documented, trigger-based extraction roadmap (Section 10.3 of the PRD) removes guesswork
  about *when* to split a module out

**Negative / Trade-offs:**
- All modules currently share a single PostgreSQL database and schema; tenant isolation
  relies entirely on `barbershop_id` filtering enforced in the service layer via
  `TenantContext`, rather than physical database-per-service separation
- A bug that breaks context-boundary discipline (e.g., a module directly querying another
  module's repository) is not prevented by infrastructure — it depends on code review and
  the checklist in `hexagonal-architecture.md`
- Scaling is currently vertical/whole-application; a single hot module (e.g., appointment
  booking under peak load) cannot be scaled independently until it is extracted

**Impact on the system:**
- Affected services: all — this decision defines the shape of `barbersaas-backend` as a
  whole
- Documents that must be updated: `overview.md` (Section 1, 6, 9), `09-microservices/service-catalog.md`,
  `02-domain/domain-map.md`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Bounded-context boundaries erode over time (direct cross-module DB access) | Medium | High | Enforce the hexagonal/bounded-context checklist in code review; no direct repository access across module packages |
| Premature or delayed microservice extraction | Medium | Medium | Extraction triggers are documented and measurable per module (PRD §10.3); no extraction without production data confirming the trigger |
| Single shared schema limits per-module scaling under load | Low (at current scale) | Medium | Monitor p95 latency and CPU per module; `appointment-service` extraction trigger explicitly covers this scenario |
| Single-developer bottleneck slows onboarding a second developer | High | High | Modular structure plus PRD/ADR/domain-map documentation is designed to let a second developer onboard within one sprint |

---

## References

- Source requirements: `PDR-BarberSaaS.md`, Section 9 (System Architecture Overview) and
  Section 10 (Microservices & Modular Design)
- Related to: `ADR-001-idioma-documentacion.md`
- Internal module structure guidance → `05-architecture/hexagonal-architecture.md`
- Applied patterns → `05-architecture/pattern-guide.md`
