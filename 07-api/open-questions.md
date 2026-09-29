# Open Questions

> `guidelines.md` and `authentication.md` close the contract's baseline decisions
> (versioning, pagination, error format, JWT mechanism, roles, tenant isolation). This file
> declares what the contract layer still leaves open on purpose — each with the evidence
> that it's a real gap (not a guess), an owner, and the condition that closes it.

## OQ-01 — No rate limiting contract on `/api/auth/**`

**Status:** partially closed (2026-09-28). The `429 TOO_MANY_REQUESTS` response with
`Retry-After` now exists in `_shared.yaml#/components/responses/TooManyRequests` and in
`guidelines.md`'s status table; per course norm 5.6 rate limiting is enforced by
`barber-saas-api-gateway`. Still open: reference it from `auth-service.yaml`.

**Evidence:** `00-governance/security-rules.md` (line 94) and `05-architecture/overview.md`
(risk `AT-002`, line 185) already flag brute-force exposure on login as a known risk,
"not yet implemented," targeted "before production." But nothing in `07-api/` reflects
this at the contract level: `guidelines.md`'s status-code table has no `429`, and
`auth-service.yaml` documents no rate-limit response for `POST /auth/login` or
`POST /auth/refresh`.

**Why it's still open:** the mechanism (Redis-backed, per overview.md's cache layer) isn't
built yet, so there is no real response shape to document — writing one now would be a
guess, not a contract.

**Responsible:** Daniel Cerquera — track alongside the Redis rate-limiting implementation;
once the throttling middleware exists, add the `429` response (with `Retry-After` header)
to `_shared.yaml#/components/responses/` and reference it from `auth-service.yaml`.

**Closing criterion:** a `429 TOO_MANY_REQUESTS` response, using the existing
`ErrorResponse` schema, is documented for every rate-limited endpoint, and
`guidelines.md`'s status-code table includes it.

---

## OQ-02 — `POST /api/admin/loyalty/grant` has no idempotency contract

**Status:** open, known non-drift gap. **Strategy decided (2026-09-28):** course norm 5.3.8
makes the `Idempotency-Key` header mandatory on every creating operation
(`_shared.yaml#/components/parameters/IdempotencyKeyHeader`). What remains is applying it to
the loyalty contract once it exists.

**Evidence:** `02-domain/domain-events.md` (lines 113–116) documents this explicitly: the
manual grant endpoint "is not idempotent against an appointment that already triggered an
automatic grant. If staff also grant a sticker manually for the same completed appointment,
the client can be credited twice." It's flagged there as "a candidate for a short SPEC, not
fixed here" — but no contract file declares it, and no one owns closing it.

**Why it's still open:** fixing it means picking an idempotency strategy (an
`Idempotency-Key` header, or a server-side check against the appointment's existing grants)
and that decision hasn't been made — `02-domain/domain-events.md` only names the symptom.

**Responsible:** Daniel Cerquera — bring to the next SPEC round as a short, scoped SPEC
(candidate: SPEC-009) that picks one strategy and updates both `contracts/openapi/` (once
a `loyalty-service.yaml` or equivalent exists) and `02-domain/domain-events.md`.

**Closing criterion:** the endpoint's contract states its idempotency guarantee explicitly
(key-based or check-based), and `domain-events.md`'s "known gap" note is removed or
resolved to point at the closing SPEC.

---

## OQ-03 — `notification-service.yaml` has no extraction timeline or owner

**Status:** open, contract is a placeholder by design.

**Evidence:** the contract's own `info.description` says it is a "planned contract for
when notification is extracted from the modular monolith per ADR-003. Not yet an
independently deployed service" — today it's `com.barbersaas.notification` inside the
monolith, reached in-process, not through this OpenAPI file. ADR-003 (per
`05-architecture/decisions/records/`) justifies *why* it will be extracted, but neither the
ADR nor the contract says *when*, or who drives the extraction.

**Why it's still open:** the extraction is correctly sequenced after the monolith is stable
(per ADR-002/003's incremental-extraction rationale), so committing to a date now would be
speculative.

**Naming is open too, for the same reason.** The course convention names an extracted
component `<abbr>-<domain>-<piece>`, but assigning that name now would mean picking a side
of a question the team hasn't formally closed: ADR-002/003 (currently in effect) describe
one service extracted incrementally from the monolith, while a full-polyrepo split
(one repo per domain, with notification split further into separate api/app/db pieces) has
already started being explored outside this repo but isn't recorded as a decision here.
Naming the component today would silently pick the second model without the ADR to back
it — the name is a consequence of that decision, not a substitute for making it.

**Responsible:** Daniel Cerquera — flag for the team's next MVP-boundary planning session
(see `00-governance/branching-policy.md`'s MVP cadence) so a target MVP (2 or 3) gets
assigned, the monolith-extraction-vs-polyrepo question gets resolved with its own ADR, and
the component name follows from whichever wins.

**Closing criterion:** an ADR resolves whether notification extracts as one service or a
polyrepo split, `notification-service.yaml`'s `info.description` names the resulting
component per `<abbr>-<domain>-<piece>` and a target MVP milestone, and that milestone is
tracked in `15-project-control/`.

---

## OQ-04 — JWT is signed with HS512; the course norm requires RS256

**Status:** open, conflicts with course norm 5.3.7.

**Evidence:** `authentication.md` documents "HS512 (symmetric, shared-secret HMAC — not
RS256)", with the secret "shared only between the components that issue and validate
tokens". Course norm 5.3.7 requires every service to validate the token itself with
**RS256 and the identity service's public key**, to reject any other algorithm (including
`none` and `HS256`), to require `exp` and `sub`, and states that **no service holds the
private key or a shared key**. With ADR-004 every domain is its own service, so a shared
HS512 secret would have to be copied into each of them.

**Why it's still open:** changing the signing mechanism changes `auth-service.yaml`
(`/jwks` is already declared there), `authentication.md` and `05-architecture/overview.md`
(which `authentication.md` names as its source of truth). That is a decision for an ADR,
not an edit to this file.

**Responsible:** team — ADR in the SPEC round that follows ADR-004.

**Closing criterion:** an ADR adopts RS256 per norm 5.3.7; `authentication.md` and
`overview.md` describe it; `_shared.yaml`'s `bearerAuth` note stops pointing here.

---

## OQ-05 — Service contracts do not follow the common contract yet

**Status:** open (found 2026-09-28 while aligning `_shared.yaml` with norm 5.3.5–5.3.9).

**Evidence:**

| Contract | Gap |
|---|---|
| `appointment-service.yaml` | Answers `409` for state transitions (`confirm`, `start`, `complete`) and double booking — must be `422 INVALID_STATUS_TRANSITION` / `422 BUSINESS_RULE_VIOLATION`; `priceAtBooking` is `type: number` — money must be `schemas/Money` (`priceCents`) |
| `auth-service.yaml` | Answers `409` for a duplicate email — must be `422 BUSINESS_RULE_VIOLATION` |
| all four (incl. `_template-service.yaml`) | No `Idempotency-Key` on creating operations; no `X-Correlation-Id` header; `servers` point at a service port instead of the api-gateway; paths still use the monolith's role prefixes |
| all four | Error examples written before 1.1.0 lack `traceId`, now required by `ErrorResponse` |
| `appointment-service.yaml`, `auth-service.yaml` | Error examples use codes outside the closed `ErrorCode` list, so they no longer validate against `_shared.yaml` 1.1.0 (see the mapping below) |

Codes outside the closed list and the code of norm 5.3.5 that replaces each one:

| Contract | Current code | Replace with |
|---|---|---|
| `appointment-service.yaml` | `INVALID_TRANSITION` | `INVALID_STATUS_TRANSITION` (422) |
| `appointment-service.yaml` | `SLOT_ALREADY_BOOKED` | `BUSINESS_RULE_VIOLATION` (422) |
| `appointment-service.yaml` | `CANCELLATION_WINDOW_CLOSED` | `BUSINESS_RULE_VIOLATION` (422) |
| `auth-service.yaml` | `EMAIL_ALREADY_EXISTS` | `BUSINESS_RULE_VIOLATION` (422) |
| `auth-service.yaml` | `INVALID_CREDENTIALS` | `UNAUTHORIZED` (401) |

The specific cause stays readable in `message` and, when useful, in `details`.

**Why it's still open:** each contract is its own change, reviewed with its service's
owner; bundling them with the shared components would exceed the 400-line PR limit
(norm 9.2).

**Responsible:** team — one PR per contract, starting with `_template-service.yaml` so new
contracts start compliant.

**Closing criterion:** every contract under `contracts/openapi/` references
`IdempotencyKeyHeader`, `CorrelationIdHeader`, `PaginatedList`, `Money` and the shared
responses, and uses no status code outside `guidelines.md`'s table.

---

## OQ-06 — Contracts use UUID and cents; `06-data/models.md` still says BIGINT and DECIMAL

**Status:** closed on the data side (2026-09-28): ADR-010 adopts UUID ids and `bigint` cents, and `06-data/models.md` was rewritten per domain with them. Still open on the contract side: `priceAtBooking` in `appointment-service.yaml` moves to `priceAtBookingCents`.

**Original status:** open (found 2026-09-28 while writing the five domain contracts).

**Evidence:** `06-data/models.md` ("ID strategy") documents every table as
`BIGINT AUTO_INCREMENT PRIMARY KEY` and every monetary column (`services.price`,
`subscription_plans.price`, `finance_records.amount`) as `DECIMAL(10,2)` in COP. The course
norm (5.3.5) and `guidelines.md` require UUID identifiers and money as integer minor units
(`priceCents`), and `barbershop-service.yaml`, `schedule-service.yaml`,
`loyalty-service.yaml`, `finance-inventory-service.yaml` and `platform-admin-service.yaml`
follow the norm. The monolith's DTOs (`ServiceResponse`, `PlanResponse`,
`FinanceRecordResponse`) still expose `Long id` and `BigDecimal price`.

**Why it's still open:** the ID/money types are a data-model decision owned by `06-data/`
(each `-db`), being reworked in parallel with the ADR that adopts UUID. The contracts were not
bent back to BIGINT/DECIMAL, and 06 was not edited from here.

**Responsible:** Carlos Leal — with the UUID ADR and the per-domain `-db` schemas.

**Closing criterion:** each `-db` schema (and `06-data/models.md`) uses UUID keys and integer
cents for the columns the contracts expose as `*Cents`, or the contracts are amended to match
whatever the ADR decides.

---

## OQ-07 — How a `CLIENT` token gets bound to a barbershop

**Status:** open.

**Evidence:** `authentication.md` says the JWT embeds `barbershop_id` for `ADMIN_BARBERSHOP`,
`BARBER` **and `CLIENT`**, and every tenant-scoped contract (appointment, barbershop,
schedule, loyalty) resolves the tenant from the token. But `06-data/models.md` stores
`users.barbershop_id = NULL` for `CLIENT` (a client is platform-wide and can visit several
barbershops), and the monolith passed the barbershop explicitly in client paths
(`/api/client/loyalty/{barbershopId}`, `/api/public/barbershops/{barbershopId}/services`).
`barbershop-service.yaml` keeps an anonymous discovery catalog under
`/api/v1/barbershops/{id}` (`DEC-SHOP-02`) for the step before a client picks a shop.

**Why it's still open:** choosing between "one token per selected barbershop" (a tenant
selection call in identity-auth) and "a claim listing the client's barbershops" changes
`auth-service.yaml` and `authentication.md`; that is an identity-auth decision, not a detail
of these contracts.

**Responsible:** team — identity-auth owner.

**Closing criterion:** `authentication.md` states how a `CLIENT` token carries its
barbershop, and `auth-service.yaml` exposes the call that issues it.

---

## OQ-08 — Barber name and photo live in another domain

**Status:** open.

**Evidence:** the monolith's `BarberPublicResponse` joins `barber_profiles` with `users` to
return `fullName` and `profilePhotoUrl`. With ADR-004 `users` belongs to identity-auth and
`barber_profiles` to barbershop, and golden rule 8 forbids one domain from querying another's
database, so `barbershop-service.yaml`'s `BarberProfile` exposes only `userId`
(`DEC-SHOP-04`). The same applies to the monolith's loyalty client search
(`/api/admin/loyalty/clients/search`) and the employee/commission/payroll endpoints
(`EmployeeController`), which read `users` and are not in any of the five new contracts;
`commission_percentage` doesn't exist in `06-data/models.md` either.

**Why it's still open:** the composition strategy (the `-app` calls both services, the
workflow composes, or barbershop keeps a read replica fed by an identity-auth event) is an
architecture decision.

**Responsible:** team — next architecture SPEC round.

**Closing criterion:** an ADR or `05-architecture/` section picks the composition strategy,
and the barber and loyalty contracts reference it.

---

## OQ-09 — Circular dependency between schedule and appointment

**Status:** open.

**Evidence:** `schedule-service.yaml`'s `GET /api/v1/availability` subtracts the barber's
booked appointments (owned by appointment) from the working hours (`DEC-SCHED-03`), while
`appointment-service.yaml`'s `POST /appointments` must check the requested slot against the
barber's schedule and exceptions (owned by schedule). In the monolith both lived in one
process (`AvailabilityService` read `appointments` directly); golden rule 8 now forbids either
domain from reading the other's database, so each would call the other's `-api`.

**Why it's still open:** breaking the cycle (appointment publishes booking events and schedule
keeps a busy-slot projection, or `barber-saas-workflow` composes availability) is an
architecture decision for `05-architecture/`, not for the contracts.

**Responsible:** team — next architecture SPEC round, alongside OQ-08.

**Closing criterion:** a decision records which service owns the availability computation and
how it learns about bookings, and both contracts reference it.

---

## OQ-10 — platform-admin changes rows that barbershop owns

**Status:** open.

**Evidence:** `platform-admin-service.yaml` creates barbershops, changes their `status` and
assigns their `planId`, but `barbershops` is barbershop's table (`06-data/models.md` groups it
under Barbershop Management, and ADR-004 gives each domain its own `-db`). Golden rule 8 forbids
platform-admin from writing that database, so `DEC-PLAT-01` routes the change through
`barber-saas-barbershop-api` — an internal, service-to-service interface that no contract
declares yet. The same applies in reverse to `DEC-PLAT-02`: refusing to deactivate a plan
still assigned to barbershops needs barbershop's data.

**Why it's still open:** whether this is a synchronous internal endpoint, an event
(`BarbershopStatusChanged`) or a workflow saga, and how the service authenticates as itself
(not with a user's token), is an architecture decision.

**Responsible:** team — together with OQ-08 and OQ-09.

**Closing criterion:** the mechanism is recorded in `05-architecture/`, and either
`barbershop-service.yaml` declares the internal operation or `02-domain/domain-events.md`
declares the event.

---

## OQ-11 — `trialEndsAt` has no column

**Status:** closed on the data side (2026-09-28): `barbershop.trial_ends_at` is stored (`06-data/models.md` §3). Still open: update `DEC-PLAT-03` in `platform-admin-service.yaml` to read it instead of deriving it.

**Original status:** open (already flagged from the data side in `06-data/models.md`, under
`barbershops`).

**Evidence:** `INV-SHOP-001` defines `trialEndsAt = createdAt + 60 days`, immutable, and FR-026's
expiration job needs to query it efficiently. `barbershops` has no `trial_ends_at` column.
`platform-admin-service.yaml` exposes it as a derived, read-only field (`DEC-PLAT-03`) instead
of inventing a column.

**Why it's still open:** storing it or keeping it derived is a `06-data/` decision, and FR-026's
job (in `barber-saas-worker`) doesn't exist yet.

**Responsible:** whoever implements FR-026, with the `barbershop-db` owner.

**Closing criterion:** the barbershop schema either adds `trial_ends_at` or documents the
derivation, and `DEC-PLAT-03` is updated to match.
