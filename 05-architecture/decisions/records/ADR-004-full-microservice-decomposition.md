# ADR-004 — Architectural Style: Full Microservice Decomposition (Course Requirement Update)

| Field | Value |
|-------|-------|
| **ID** | ADR-004 |
| **Date** | 2026-09-20 |
| **Status** | Accepted |
| **Authors** | Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia |
| **Supersedes** | ADR-002, ADR-003 |

---

## Context

ADR-002 established BarberSaaS's backend as a **modular monolith** with extraction defined
as strictly **trigger-based**: no module is pulled out until a measured production condition
(latency, load, a second consuming client) justifies it. ADR-003 carved out exactly one,
tightly scoped exception — `notification-service` — as an academic deliverable ahead of its
trigger, and was explicit that this exception did not extend further: *"any future request to
extract another module early needs its own ADR, not a ride on this one"* and *"none of
[appointment/auth/loyalty/analytics] is extracted ahead of its measured trigger because of
this ADR."*

Since ADR-003 (2026-09-03), the course requirement itself changed: as of this ADR
(2026-09-20), the Distributed Systems course (Sistemas Distribuidos, CORHUILA, 2026-B)
requires the deliverable to be built as a **full microservices architecture**, not a monolith
with a single extracted service. This confirmation came directly from Daniel in session; there
is not yet a cited written professor/syllabus source for it (see "Open questions" below).

Independent evidence corroborates the requirement change: a polyrepo of **29 repositories**
already exists under the course organization `code-corhuila`, following a
one-repo-per-bounded-context-per-layer pattern (`<domain>-db`, `<domain>-api`, `<domain>-app`)
for eight business domains, plus four cross-cutting repositories. These 29 repositories share
a branching policy — documented course-side in `00-governance/branching-policy.md` — requiring
**1 approval from `ariel5253`** (the professor's account, per that same policy file) on `main`,
identical to the gate this DOCS repository itself uses. This is the same course-level
governance pattern, not an ad-hoc team setup, which supports reading the polyrepo as official
course infrastructure rather than a parallel experiment.

**Known constraints:**
- Team size: unchanged (up to 4 for the course team), but now distributed across up to 29
  independently deployable units instead of 1 monolith + 1 service — a materially larger
  operational surface than ADR-003 anticipated.
- Course timeline: further along than ADR-003's "week 5 of 16" baseline; the original
  trigger-based extraction glide path assumed most of the semester to absorb one exception,
  not a full decomposition.
- The production-scale triggers defined in `PDR-BarberSaaS.md` §10.3 still won't fire at
  academic scale — the same problem ADR-003 already identified for `notification`, now
  generalized to every remaining module.

---

## Decision

**We decide:** adopt full microservice decomposition as BarberSaaS's course deliverable
architecture, matching the already-provisioned 29-repository polyrepo under `code-corhuila`.
Each of the eight business domains — `identity-auth`, `barbershop`, `appointment`, `schedule`,
`loyalty`, `notifications`, `finance-inventory`, `platform-admin` — is split into three
repositories: `-db` (schema/migrations/seeds), `-api` (backend service), `-app` (mobile/UI
remote for that domain). Four cross-cutting repositories complete the platform: `front`
(micro-frontend shell composing each domain's `-app` as a remote), `api-gateway` (single entry
point: auth, routing, rate limiting), `workflow` (business process/saga orchestration), and
`worker` (async/background job processing) — plus `infra` (compose/IaC, observability,
environments, secrets).

This **supersedes** both ADR-002's default (modular monolith, trigger-based extraction) and
ADR-003's scoped single-exception model. It does not merely stack another exception on top of
ADR-003 — it replaces the underlying premise, because the course requirement, not measured
production load, is now the driver of the architecture.

**Justification:** ADR-002/003 optimized for production-realistic engineering discipline —
extract only when a measured trigger justifies the operational cost. That discipline assumed
the course would accept, and even reward, staying a monolith except for one demonstrated
extraction. That assumption no longer holds: the course now requires the full distributed-
systems surface. Continuing to develop `barbersaas-backend` (CODE, `code-corhuila/barber-saas`)
as the primary deliverable under the old assumption would mean building toward an architecture
that does not satisfy the actual requirement, and would leave the 29 already-provisioned
repositories unexplained and unused.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| Keep ADR-002/003 as-is; treat the polyrepo as a parallel, non-canonical exercise | No rework; preserves the trigger-based discipline documented so far; avoids the operational overhead of 29 deployables for a 4-person team | Leaves the 29 course-provisioned repos unused and unexplained on the record; contradicts the confirmed course requirement; if grading targets the polyrepo, the monolith becomes irrelevant work | Discarded — the course requirement itself changed; this is not scope drift to resist, it is an update to the actual grading target |
| Extend ADR-003's exception model — a few more scoped extractions instead of full decomposition | Incremental; preserves most of the monolith's operational simplicity; lower risk per step | Doesn't match what's already provisioned (29 full repositories, not a handful); the confirmed requirement is "full microservices," not "a few more services" | Discarded — doesn't match the actual scope of the requirement or the already-created infrastructure |
| **Full microservice decomposition matching the 29-repo polyrepo (chosen)** | Matches the deliverable already provisioned by the course's own tooling (`branching-policy.md`, CODEOWNERS, `ariel5253` gate — same governance pattern as DOCS); satisfies the confirmed requirement directly; the bounded-context boundaries ADR-002 already enforced inside the monolith become directly reusable service boundaries, which was explicitly ADR-002's stated benefit ("extraction path is explicit and low-risk if bounded contexts are respected") | Full distributed-systems operational cost (service discovery, network partial failure, 29 CI/CD pipelines) lands on a small team all at once — a much larger jump than ADR-003's single scoped exception; multi-tenant isolation must now be reimplemented and verified per service instead of centrally; the previously planned incremental trigger-based glide path is abandoned | — (chosen) |

---

## Consequences

**Positive:**
- Satisfies the confirmed, current course requirement directly, instead of the
  notification-only assumption ADR-003 made, which no longer matches it.
- The bounded-context discipline ADR-002 already enforced inside the monolith gives each new
  service a natural, already-documented boundary (`02-domain/domain-map.md`) — extraction is
  not starting from a tangled codebase.
- Unblocks productive use of the 29 already-created repositories instead of leaving them idle.

**Negative / Trade-offs:**
- Full distributed-systems operational cost lands on a small team all at once: up to 29 CI/CD
  pipelines, service discovery/routing through `api-gateway`, network partial-failure
  handling, and per-service deployment — a much bigger jump than the single
  `notification-service` exception ADR-003 scoped.
- Multi-tenant isolation, previously centralized in one `TenantContext`/
  `JwtAuthenticationFilter` inside the monolith, must now be correctly reimplemented (or
  shared as a library) in every `-api` service individually. A missed `barbershop_id` filter
  in any one of eight services reproduces the exact cross-tenant leak risk ADR-002 already
  flagged — now multiplied across eight surfaces instead of one.
- The relationship between `barbersaas-backend`/`barbersaas-frontend` (CODE,
  `code-corhuila/barber-saas`) and the new polyrepo is **not resolved by this ADR**: whether
  CODE is migrated into the 29 repos, kept as a reference, or retired needs its own follow-up
  decision with Daniel — see "Open questions."
- The 29 repositories currently carry unedited template residue ("LMS Library" / team
  `lms-library` / `library-docs`) in their READMEs from the course scaffolding tool. Not a
  defect introduced by this ADR, but a known gap it inherits and that must be cleaned up
  before those READMEs count as real per-repo documentation.

**Impact on the system:**
- Affected: the entire backend architecture — supersedes the shape ADR-002 gave to
  `barbersaas-backend` as a whole, and the single-exception shape ADR-003 gave to
  `notification`.
- Documents that must be updated: `09-microservices/service-catalog.md` (currently documents
  only the single `notification-service` exception, must be rewritten for the 29-service
  reality), `02-domain/domain-map.md` (confirm domain boundaries still match the polyrepo's
  eight-domain split), `overview.md` (Sections 1, 6, 9).

---

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| The "full microservices" requirement was misunderstood or will change again before delivery | Medium | High | Get a written confirmation (syllabus excerpt, professor message) and attach it as a reference to this ADR; if the requirement changes, it needs its own superseding ADR, same discipline ADR-003 asked of future exceptions |
| Multi-tenant isolation regresses because each service reimplements `barbershop_id` filtering independently | Medium | High | Extract the `TenantContext`/JWT-filter logic into a shared library or document the pattern per service before any service starts handling real tenant data; treat this as a review-gate checklist item per repo |
| Team capacity cannot sustain 29 independently deployable services (CI/CD, monitoring, versioned contracts) for a 4-person course team | Medium | Medium | `infra`/`workflow`/`worker`/`api-gateway` centralize the cross-cutting operational concerns; prioritize which domains actually need to be functional for grading versus scaffolded-only |
| CODE (`barber-saas` monolith) and the polyrepo drift into two competing "real" implementations | High until resolved | Medium | Resolve the open question below before further feature work lands in either repository |

---

## Open questions (not resolved by this ADR)

1. Does `code-corhuila/barber-saas` (CODE) get migrated/retired in favor of the polyrepo, or
   does it continue as a separate deliverable? Needs its own decision with Daniel before
   further feature work is planned in either place.
2. Is there a written professor/syllabus source for the "full microservices" requirement to
   cite here, beyond this session's confirmation from Daniel?

---

## References

- `ADR-002-modular-monolith.md` — superseded by this ADR
- `ADR-003-academic-microservice-extraction.md` — superseded by this ADR
- `00-governance/branching-policy.md` — course-mandated branching policy; same `ariel5253`
  gate pattern found in the 29-repo polyrepo, used here as corroborating evidence
- `02-domain/domain-map.md` — bounded-context boundaries reused as service boundaries
- Course syllabus (Sistemas Distribuidos, 2026-B) — cited pending the written source in
  "Open questions" above
