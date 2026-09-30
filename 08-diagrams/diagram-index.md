# Diagram Index

> Registry of every BarberSaaS diagram. Adding, renaming or removing a diagram updates this
> table in the same Pull Request. Conventions: [`README.md`](README.md).

## Architecture — C4

| ID | Name | Level | File | Derived from | Updated |
|---|---|---|---|---|---|
| C4-01 | System context | L1 | [`c4/c1-system-context.md`](c4/c1-system-context.md) | `05-architecture/overview.md` §2 · `07-api/authentication.md` | 2026-09-30 |
| C4-02 | Containers | L2 | [`c4/c2-containers.md`](c4/c2-containers.md) | `05-architecture/overview.md` §3–4 · `deployment.md` · ADR-004/006/009 | 2026-09-30 |
| C4-03 | appointment-api components | L3 | [`c4/c3-appointment-api.md`](c4/c3-appointment-api.md) | `05-architecture/hexagonal-architecture.md` | 2026-09-30 |

## Behavior — UML

| ID | Name | Type | File | Derived from | Updated |
|---|---|---|---|---|---|
| SEQ-01 | Login and token validation | Sequence | [`uml/seq-login.md`](uml/seq-login.md) | `07-api/authentication.md` · `auth-service.yaml` | 2026-09-30 |
| SEQ-02 | Book an appointment without double-booking | Sequence | [`uml/seq-book-appointment.md`](uml/seq-book-appointment.md) | `appointment-service.yaml` · `06-data/models.md` §5, §10 | 2026-09-30 |
| SEQ-03 | Complete an appointment (outbox → loyalty, notifications) | Sequence | [`uml/seq-complete-appointment.md`](uml/seq-complete-appointment.md) | `appointment-service.yaml` · `02-domain/domain-events.md` · `06-data/models.md` §6–7 | 2026-09-30 |
| SEQ-04 | Owner onboarding saga (proposed) | Sequence | [`uml/seq-owner-onboarding-saga.md`](uml/seq-owner-onboarding-saga.md) | `07-api/open-questions.md` OQ-12 · ADR-009 · annex E | 2026-09-30 |
| ST-01 | Appointment lifecycle | State machine | [`uml/state-appointment.md`](uml/state-appointment.md) | `02-domain/entities-and-rules.md` · `appointment-service.yaml` | 2026-09-30 |
| ST-02 | Barbershop subscription lifecycle | State machine | [`uml/state-barbershop.md`](uml/state-barbershop.md) | `02-domain/entities-and-rules.md` · `06-data/models.md` §3 | 2026-09-30 |

## Data — ER

| ID | Database | Engine | File | Derived from | Updated |
|---|---|---|---|---|---|
| ERD-01 | identity-auth-db | PostgreSQL | [`er/erd-domain-databases.md`](er/erd-domain-databases.md#erd-01--identity-auth-db-postgresql-schema-identity_auth) | `06-data/models.md` §2 | 2026-09-30 |
| ERD-02 | barbershop-db | PostgreSQL | same file | §3 | 2026-09-30 |
| ERD-03 | schedule-db | PostgreSQL | same file | §4 | 2026-09-30 |
| ERD-04 | appointment-db | PostgreSQL | same file | §5 | 2026-09-30 |
| ERD-05 | loyalty-db | PostgreSQL | same file | §6 | 2026-09-30 |
| ERD-06 | notifications-db | MongoDB | same file | §7 | 2026-09-30 |
| ERD-07 | finance-inventory-db | PostgreSQL | same file | §8 | 2026-09-30 |
| ERD-08 | platform-admin-db | PostgreSQL | same file | §9 | 2026-09-30 |
| ERD-09 | `idempotency_key`, `outbox_event` (every domain) | both | same file | §10 | 2026-09-30 |

## Divergences found while drawing (rule P2)

Drawing forced a few questions the documents do not answer the same way. The diagrams follow
the most specific source and flag the point; none is resolved here.

| # | Diagram | Divergence | Documents involved |
|---|---|---|---|
| D-1 | SEQ-04 | OQ-12 creates the user first and compensates by deactivating it, but `chk_app_user_tenant` forbids an `ADMIN_BARBERSHOP` without `barbershop_id`; the diagram creates the barbershop first | `07-api/open-questions.md` OQ-12 · `06-data/models.md` §2 |
| D-2 | C4-02, SEQ-02 | The booking contract checks the slot against the barber's schedule, but the appointment-api structure lists no port toward schedule-api (only `BarbershopCatalog`) | `appointment-service.yaml` · `05-architecture/hexagonal-architecture.md` |
| D-3 | SEQ-03 | `domain-events.md` still describes the prototype (in-process calls, income registered in finance on completion); finance-inventory's contract declares no consumer of `AppointmentCompleted` | `02-domain/domain-events.md` · `finance-inventory-service.yaml` |
| D-4 | C4-02, SEQ-03 | Event transport between outboxes and consumers is undecided | `05-architecture/overview.md` AT-004 |

## Not drawn yet

- **C4-03 for the other seven services:** same shape as appointment-api; drawn when a service's
  ports differ enough to matter.
- **Business process view of the sagas:** belongs in `16-bpmn/`, which does not exist yet.
