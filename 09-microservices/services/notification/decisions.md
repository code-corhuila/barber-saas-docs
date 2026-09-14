# Notification — Internal Design Decisions

## Extraction to `notification-service`

Decided in `ADR-003-academic-microservice-extraction.md`: `notification` is extracted from
the monolith into a real, independently deployable service as a scheduled academic
deliverable (course weeks 5-7) — ahead of the production trigger defined in
`ADR-002-modular-monolith.md` / `PDR-BarberSaaS.md` §10.3 (FCM/email calls add >200ms to
p95 appointment-creation latency, OR >5,000 active barbershops).

**Why this module first:** classified as a **Generic** subdomain in
`02-domain/domain-map.md` (commodity capability, not a competitive differentiator), owns
its own tables with no Shared Kernel dependency on any other bounded context, and the
domain map already anticipated the move ("Event-driven will be introduced when
Notifications moves to its own service").

## In-process → HTTP/event migration

Today, `appointment` and (planned) `loyalty` call `notificationService.notify(...)`
directly as an in-process Java method call — no network hop, synchronous. Once extracted,
per `PDR-BarberSaaS.md` §10.4, this becomes either an HTTP POST to the new service or an
asynchronous event publish (Kafka/RabbitMQ — technology choice not yet made, out of scope
of `ADR-003`).

## Known drift to account for during extraction

Per `02-domain/domain-map.md` (context map, relationship table), as of the last domain
review:
- `appointment.complete()` does **not** call `notificationService.notify(...)` — only
  `create()`, `confirm()`, `cancel()`, and the daily reminder job do.
- The NO_SHOW auto-marking job does not trigger a notification either.
- `loyalty.grantSticker()` and `.redeemReward()` do **not** call the notification service —
  this integration is planned, not implemented.

These gaps exist in the monolith today; extracting the module does not fix them by itself
and should not silently paper over them — they should be resolved (or explicitly deferred)
before or during the extraction, not assumed to be pre-existing feature parity.

---

## Correlations

- Extraction decision → `05-architecture/decisions/records/ADR-003-academic-microservice-extraction.md`
- Base architectural style → `05-architecture/decisions/records/ADR-002-modular-monolith.md`
- Domain classification and relationship drift notes → `02-domain/domain-map.md`
