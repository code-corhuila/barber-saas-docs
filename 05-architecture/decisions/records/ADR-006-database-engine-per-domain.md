# ADR-006 — Database Engine per Domain: PostgreSQL for Seven Domains, MongoDB for Notifications

- **ID:** ADR-006
- **Date:** 2026-09-28
- **Status:** Accepted
- **Authors:** Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia

---

## Context

The course norm (4.2.2) requires choosing, per domain, PostgreSQL or MongoDB. Each domain has its own
database, in its own instance, with its own volume (norm 7.1). The prototype uses MySQL 8 with one
shared schema filtered by `barbershop_id`; neither MySQL nor a shared schema is allowed.

---

## Decision

**We decided:** `identity-auth`, `barbershop`, `appointment`, `schedule`, `loyalty`,
`finance-inventory` and `platform-admin` use **PostgreSQL**; `notifications` uses **MongoDB**
(single-node replica set). Each domain runs its own instance, defined in its `-db` repository.

For `notifications` (annex B):
- **Embed or reference:** the delivery attempts of a notification are embedded; the recipient, the
  barbershop and the source event are referenced by identifier.
- **Array limit:** embedded delivery attempts have `maxItems: 10`; a notification that exceeds it is
  marked failed instead of growing.
- **Validation:** `$jsonSchema` validator with `additionalProperties: false`,
  `validationLevel: strict` and `validationAction: error` from the first migration.

---

## Evaluated alternatives (options)

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| **PostgreSQL ×7 + MongoDB for notifications (chosen)** | relational integrity where money, schedules and appointments live; a document store where each notification carries a channel-specific payload | two engines and two annexes (A and B) | — (chosen) |
| PostgreSQL in all eight | one annex, one pipeline | a variable payload per channel forced into JSON columns | the notification payload is a document by nature |
| MongoDB in several domains | schema flexibility | loses the CHECK constraints and transactions that appointment and finance need | consistency required by the transactional domains |

---

## Dominant criterion

**The shape of each domain's data:** appointments, schedules, finance and loyalty need constraints
and transactions (PostgreSQL); notifications are append-mostly documents whose payload changes per
channel (push, e-mail) and are never joined with other data (MongoDB).

## Accepted cost

Two engines to operate and to test, two structures (annexes A and B), a single-node replica set and
the Liquibase MongoDB extension for one domain, and eight database instances in development.

---

## Consequences

- `barber-saas-notifications-db` follows annex B; the other seven `-db` repositories follow annex A.
- No foreign key crosses domains; references are identifiers checked through contracts (norm 7.4).
- Money is stored in minor units (`bigint` / `long`) and states with `CHECK`, never `ENUM` (norm 5.2.5).

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| The team treats MongoDB like a relational schema | Medium | Medium | annex B checklist in review; validator from day one |
| Too many instances for a laptop | Medium | Medium | resource limits in each `deploy/compose.yml`; start only what is needed |

## References

- Course norm 4.2.2, 5.2, 5.2.4, 7.1–7.4, annexes A and B
- Related to: ADR-004, ADR-007
