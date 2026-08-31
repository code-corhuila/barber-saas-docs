# Non-Functional Requirements (NFR)

> NFRs define the **qualities of the system** — not what it does but how well it does it.
> The golden rule: every NFR must have a metric. "The system must be fast" is not an NFR.
> "The P99 latency of the /orders endpoint must be < 200ms under 500 RPS load" is.

---

## How to write a measurable NFR?

| Bad | Good |
|-----|------|
| "The system must be fast" | "P95 latency must be < 300ms under 1000 concurrent RPS" |
| "The system must be secure" | "All endpoints require a valid JWT; tokens expire in 1 hour" |
| "The system must scale" | "The system must support up to 5000 concurrent users without degradation" |
| "The system must be available" | "Availability SLO: 99.9% monthly (maximum 44 min downtime/month)" |

---

## NFR-001: Performance

| Attribute | Metric | Test condition |
|-----------|--------|---------------|
| P95 latency — critical endpoints | < 300ms | **Target only — not yet measured.** No load test has been run against this codebase as of this review |
| P99 latency — critical endpoints | < 500ms | **Target only — not yet measured** |
| P95 latency — non-critical endpoints | < 1000ms | **Target only — not yet measured** |
| Minimum throughput | Not yet defined | No expected-load figure has been agreed with the team; define before Phase 2 (production) launch |
| Service startup time | < 30 seconds | Cold start — reasonable target for a single Spring Boot JAR, not yet benchmarked |

**Defined critical endpoints (grounded — these carry direct revenue/trust impact):**
- `POST /api/client/appointments` — booking is the core domain (ADR-002); a slow or failed booking directly costs the barbershop a client
- `POST /api/auth/login` — gateway to every other operation; every session starts here

**Load testing tools:** Not yet chosen. k6, Apache JMeter, Locust, and Gatling are reasonable
candidates, but none is wired into this project today — there is no CI/CD pipeline yet
(see `10-devops/`, currently template).

**Where is it validated?** Nowhere yet. There is no staging pipeline or CI/CD configured
for this repo. This is a real gap, not a "not applicable" — it should be closed before
Phase 2 (Controlled Launch, per `03-product/vision.md`).

---

## NFR-002: Availability

| Environment | SLO | Maintenance window | Max downtime/month |
|------------|-----|-------------------|-------------------|
| Production | 99.9% | Sundays 2am-4am | 44 minutes |
| Staging | 95% | No restriction | 36 hours |

**Monthly error budget in production:** 44 minutes
**Error Budget policy:** If > 50% of the error budget is consumed in the first half of the month,
feature deploys are frozen until the next month and stability is prioritized.

**Health checks:**
- **Not implemented.** `spring-boot-starter-actuator` is not a dependency in `pom.xml`, so
  there is no `/actuator/health` (or equivalent) endpoint today. Adding it is a one-dependency
  change and should happen before relying on Railway (or any platform) health-check-based
  restarts in Phase 2.

---

## NFR-003: Scalability

> The scenarios below (Kubernetes-style horizontal auto-scaling) do **not** match the
> actual deployment target. BarberSaaS deploys as a single Spring Boot JAR to **Railway**
> (per `01-context/overview_en.md` → Technology stack), not Kubernetes. Rewritten to
> reflect that target instead of a generic microservices assumption.

| Scenario | Expected behavior |
|---------|------------------|
| Gradual load growth | Railway vertical scaling (more CPU/RAM to the single instance) — Railway does not orchestrate Kubernetes-style horizontal pod autoscaling |
| Sudden spike | Not yet handled — no autoscaling policy is configured; this is an open item for Phase 2 hardening |
| Horizontal scaling limit | N/A today — the app runs as one instance. Horizontal scaling would require the app to first be made properly stateless (see below) and a load balancer added in front of multiple Railway instances, neither of which exists yet |

**Strategy (current, and gap):** The backend already avoids in-memory session state — JWT is
stateless and Redis is a listed dependency (`spring-boot-starter-data-redis`) for caching —
but `TenantContext` is a per-request `ThreadLocal`, which is safe for horizontal scaling
*if* each instance is stateless otherwise. This has not been load-tested with multiple
instances behind a load balancer, so treat "horizontally scalable" as an architectural
intent (ADR-002 keeps this door open), not a validated capability today.

---

## NFR-004: Security

### Authentication and Authorization
- All private endpoints require a valid JWT in the `Authorization: Bearer <token>` header
- JWT access tokens expire in **24 hours** (`expiration-ms: 86400000` in `application.yml` — corrected from an earlier draft of this document that said 1 hour)
- Refresh tokens valid for **7 days** (`refresh-expiration-ms: 604800000`) — currently a second stateless JWT, not yet DB-backed/revocable; see the known gap noted in `00-governance/security-policy.md`
- RBAC (Role-Based Access Control): 4 fixed roles (`SUPER_ADMIN`, `ADMIN_BARBERSHOP`, `BARBER`, `CLIENT`) enforced via `@PreAuthorize("hasRole(...)")` — full detail in `00-governance/security-policy.md`

### Data transmission
- HTTPS mandatory in production (TLS 1.2+)
- HTTP only in local development

### Sensitive data
- Passwords: hashing with bcrypt (cost factor ≥ 12) or Argon2id
- PII (personal data): encrypted at rest
- Secrets/keys: only in environment variables or vault, **never in code**

### OWASP Top 10
Code must be reviewed against the OWASP Top 10 on each release.
Tools: SAST (SonarQube/Snyk), dependency scanning, DAST in staging.

### Regulatory compliance
- **Habeas Data** (Ley 1581 de 2012, Colombia) applies — BarberSaaS's target market is
  exclusively Colombian (per `01-context/overview_en.md`). Whether client data must be
  physically stored within Colombia is an explicit **open legal question**, owned by
  Legal, to be resolved before production — see `01-context/scope.md` → Constraints
- GDPR and PCI-DSS are not currently applicable: no EU user base is targeted, and no card
  data is processed in the MVP (billing is manual, per `01-context/scope.md`)

---

## NFR-005: Observability

> **None of this is implemented today.** The stack has no Actuator, no metrics exporter,
> no tracing library, and no log aggregation — this table is a target for Phase 2
> hardening, not current behavior. `13-operations/observability.md` is still the empty
> template.

| Pillar | Requirement (target) | Current state |
|--------|------------|------|
| Logs | Structured JSON format + Correlation ID | Plain SLF4J/Logback console logging (default Spring Boot), no structured format, no correlation ID propagation |
| Metrics | RED (Rate, Errors, Duration) per endpoint | None — no metrics exporter configured (no Micrometer/Prometheus dependency in `pom.xml`) |
| Traces | End-to-end distributed traces | Not applicable to a single-process modular monolith today (ADR-002); would matter once/if a module is extracted |
| Alerts | Alert in < 5 min when SLI violates SLO | None — no monitoring stack exists to alert from |

**Correlation ID:** Not implemented. If added, the standard approach (a UUID generated per
external request and propagated through logs via MDC) is a reasonable target, but no code
implements it as of this review.

---

## NFR-006: Maintainability

| Metric | Target | Current state |
|--------|--------|----------------|
| Test coverage | ≥ 80% of lines (≥ 90% in the domain) | **0%** — `src/test/java` in `barbersaas-backend` contains zero test files as of this review. This is the single largest gap against this NFR and should be a near-term priority, not a Phase-2 item |
| Cyclomatic complexity | ≤ 10 per function | Not measured — no static analysis tool (SonarQube/Checkstyle/etc.) is configured |
| Technical debt | Resolution time < 1 sprint from registration | No technical-debt log exists yet — `15-project-control/technical-backlog.md` is not present in this repo |
| Onboarding time | A new dev can deploy locally in < 1 hour | Plausible given `docker-compose.yml` exists, but `10-devops/local-setup.md` is still the empty template — not verified end-to-end |
| Average build time | < 5 minutes in CI | Not applicable yet — there is no CI pipeline configured for this project |

---

## NFR-007: Portability

- The backend is packaged as a Docker image (real: `Dockerfile` + `docker-compose.yml` exist in `barbersaas-backend/`) and fronted by Nginx locally
- **Kubernetes is not the target** — deployment is planned on Railway (`01-context/overview_en.md`), which runs the container directly. Remove/ignore any K8s-specific assumption until an actual K8s deployment is scoped
- No service depends on the host operating system (standard for a containerized JVM app)
- Environment variables are the source of environment-specific configuration — confirmed in `application.yml` (`${JWT_SECRET:...}`, `${DB_URL:...}`-style placeholders)

---

## NFR-008: Disaster Recovery (DR / Recovery)

> Not yet defined for the real deployment target. The table below (K8s restarts, DB
> failover, multi-AZ, multi-region) assumes infrastructure BarberSaaS does not have —
> there is no Kubernetes cluster and no database replica configured. Treat this whole
> section as **pending**, to be defined once Railway production deployment is scoped
> (Phase 2, per `03-product/vision.md`), rather than as an existing guarantee.

| Scenario | RTO / RPO |
|---------|------------------------------|
| Single instance failure | Not yet defined — depends on Railway's own restart behavior, not evaluated |
| Primary database failure | Not yet defined — no replica or automated failover configured (single MySQL instance in dev, PostgreSQL migration still pending per `01-context/scope.md`) |
| Full-region disaster | Out of scope for a single-region MVP deployment; revisit only if/when multi-region is actually planned |

---

## NFR priority matrix

| NFR | Priority (P1/P2/P3) | Validated in CI? | Owner |
|-----|---------------------|-----------------|-------|
| Performance | P2 | No — no CI/CD pipeline exists yet | Carlos Leal (Tech Lead) |
| Availability | P2 | No — no health checks or CI exist yet | Carlos Leal (Tech Lead) — no dedicated DevOps role on the team |
| Security | P1 | No — no SAST/OWASP scanning configured; see `00-governance/security-policy.md` for manual checklist | Carlos Leal (Tech Lead) |
| Scalability | P3 | No | Carlos Leal (Tech Lead) |
| Observability | P2 | No | Carlos Leal (Tech Lead) |
| Maintainability | **P1** | No — test coverage is currently 0%, this is the most urgent gap on this list | Whole team (Carlos Leal, Daniel Cerquera, Juan Pablo Borrero, Carolay Arraut) |

> Priorities were re-ranked from the original generic template: **Maintainability moved to
> P1** because 0% test coverage on a system with financial records, appointment locking, and
> multi-tenant isolation is a correctness risk, not a nice-to-have. Performance/Availability/
> Scalability were lowered from P1 because there is no production traffic yet to protect —
> they matter more once Phase 2 (Controlled Launch) begins.

---

## Correlations

- Detailed SLOs and SLAs → `13-operations/README.md`
- Pipeline that validates NFRs → `10-devops/README.md`
- Incidents related to NFR violations → `13-operations/incident-management.md`
- Security checklist → `00-governance/security-policy.md`
