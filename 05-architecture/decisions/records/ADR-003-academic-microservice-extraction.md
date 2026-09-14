# ADR-003 — Academic Microservice Extraction: `notification-service`

- **ID:** ADR-003
- **Date:** 2026-09-03
- **Status:** Accepted
- **Authors:** Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia

---

## Context

ADR-002 established BarberSaaS's backend as a **modular monolith**, with microservice
extraction defined as **trigger-based, not schedule-based**: no module is pulled out of the
monolith until a measured production condition (latency, load, or a second consuming
client) justifies it (`PDR-BarberSaaS.md` §10.3, repo WEEKLY).

This project is developed for the **Distributed Systems** course (Sistemas Distribuidos,
CORHUILA, 2026-B). The course syllabus requires hands-on work with distributed
architectures and builds toward it progressively: week 2 covers architectural styles
including microservices, weeks 5-6 cover containerization and orchestration (the current
week), and week 7 covers inter-service communication (REST, gRPC, messaging). The course is
explicitly about the contrast and trade-offs between monoliths and microservices — it is
not satisfied by staying a monolith for its full duration.

The production triggers defined in ADR-002/PRD §10.3 are calibrated for real business
scale — for example, `notification-service` extracts when FCM/email calls add more than
200ms to the p95 latency of appointment creation, OR the platform passes 5,000 active
barbershops. At the academic MVP's actual scale ("tens of barbershops"), **neither
condition will ever fire on its own**. Without an explicit decision, the project would
reach the end of the course still as a pure monolith, unable to demonstrate the
microservices competency the course requires.

**Known constraints:**
- Course timeline: currently week 5 of 16. The syllabus's own progression (containerization
  now, inter-service communication in week 7) is the natural implementation window.
- Team size: 1 active developer (same constraint as ADR-002) — the extraction must stay
  small in scope.
- The extraction must not compromise the trigger-based discipline ADR-002 established for
  the rest of the roadmap.

---

## Decision

**We decided:** extract exactly one module, `notification` (`com.barbersaas.notification`),
into a real, independently deployable `notification-service`, as a **scheduled academic
deliverable during weeks 5-7 of the course** — regardless of whether the production trigger
condition defined in PRD §10.3 has been measured or met.

**Justification:** `notification` is classified as a **Generic** subdomain in
`02-domain/domain-map.md` (commodity capability — FCM/email delivery — not a competitive
differentiator), owns its own tables (`notifications`, `device_tokens`) with **no Shared
Kernel** dependency on any other bounded context. This is unlike `Schedule`, which shares
tables with `Appointment` and was explicitly discarded as an extraction candidate in
`domain-map.md` §5 (trigger: ">10,000 barbershops"). The domain map already anticipated
this exact move: *"Event-driven will be introduced when Notifications moves to its own
service."* It is also already listed as **Phase 2** in the production roadmap
(`PDR-BarberSaaS.md` §10.3) — this decision does not invent a new candidate, it moves up
the timeline of one the team had already agreed on.

This is an **explicit, scoped exception** to the trigger-based extraction criterion of
ADR-002. It applies **only** to `notification-service`. The remaining roadmap
(`appointment-service`, `auth-service`, `loyalty-service`, `analytics-service`) remains
strictly trigger-based per PRD §10.3 and ADR-002 — none of them is extracted ahead of its
measured trigger because of this ADR.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| **Extract `notification-service` now, ahead of its production trigger (chosen)** | Satisfies the course's microservices requirement without touching the core domain; low blast radius; module already has isolated tables; aligns with syllabus weeks 5-7 | Extraction happens without the operational proof (measured latency/scale) ADR-002 normally requires | — (chosen) |
| Wait for the production trigger to fire naturally | Fully consistent with ADR-002's discipline | Will almost certainly never happen within an academic MVP at this scale — the course requirement would go unmet | Discarded — conflicts with a hard course deliverable |
| Extract `appointment-service` (the core domain) instead | Appointment is the most business-critical module — a more visible demo | Highest risk: shares tables with `Schedule` (Shared Kernel), touches the anti-double-booking state machine — a broken extraction directly breaks the core product for a 1-developer team | Discarded — risk/effort disproportionate to a course proof-of-concept |
| Extract all remaining modules into full microservices now | Maximizes demonstrated distributed-systems surface | Reintroduces exactly the operational overhead ADR-002 was written to avoid, this early, for a 1-developer team | Discarded — contradicts ADR-002 without a corresponding trigger for any of the other modules |

---

## Consequences

**Positive:**
- Satisfies the course's requirement to demonstrate a real microservice (independent
  deployable, network communication, containerization) without touching the core domain.
- Validates in practice the extraction seam ADR-002 and the domain map already designed for
  (`notification` boundary, in-process call → HTTP/event).
- Low blast radius: if the extraction has issues, the core booking/loyalty/finance
  functionality of the monolith is unaffected.

**Negative / Trade-offs:**
- Introduces the first real distributed-systems operational cost (two deployables, network
  calls, partial-failure handling) earlier than the production trigger would have
  justified.
- Creates a documented precedent of "extraction ahead of trigger" that must stay scoped to
  this one exception — any future request to extract another module early needs its own
  ADR, not a ride on this one.

**Impact on the system:**
- Affected: `com.barbersaas.notification` module (becomes a separate deployable);
  `Appointment` and `Loyalty` modules (their in-process calls to Notification become
  HTTP/event calls).
- Documents that must be updated: `09-microservices/service-catalog.md`,
  `09-microservices/services/notification/` (new), `05-architecture/overview.md`
  (reference only).

---

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| The instructor specifies a different module or timeline than assumed here | Medium | Medium | This ADR is written as "first candidate, subject to instructor confirmation," not a closed date; tracking the confirmation is a follow-up open question, out of scope of this ADR |
| This exception is read as weakening ADR-002's trigger-based discipline for other modules | Low | Medium | This ADR explicitly limits itself to `notification-service`; ADR-002's criterion is unchanged for the rest of the roadmap |
| Extraction breaks notification delivery for Appointment/Loyalty during the transition | Medium | Low | Notification is already designed for graceful degradation (persists regardless of delivery outcome, per `domain-map.md`) — the extraction must preserve that property |

---

## References

- `ADR-002-modular-monolith.md`
- `PDR-BarberSaaS.md` §10 (repo WEEKLY, `04-week/hu-status/PDR-BarberSaaS.md`) — source of
  the module table and the extraction roadmap; read-only reference, not modified by this
  ADR
- `02-domain/domain-map.md` — Notifications bounded context classification (Generic) and
  the anticipated event-driven note
- Course syllabus (Sistemas Distribuidos, 2026-B) — weeks 2, 5-6, 7
