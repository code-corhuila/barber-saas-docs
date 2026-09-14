# Module & Service Catalog

> BarberSaaS runs today as a **modular monolith** (`ADR-002-modular-monolith.md`): a single
> Spring Boot deployable (`barbersaas-backend`, port `8080`), internally organized into 11
> bounded-context modules under `com.barbersaas.*`. This catalog documents those modules —
> not independent microservices — plus the trigger-based extraction roadmap that decides
> when (if ever) each one becomes a real service.
>
> One module, `notification`, is the exception: it is extracted to a real
> `notification-service` as a scheduled academic deliverable, ahead of its production
> trigger — see `ADR-003-academic-microservice-extraction.md`. Its detail lives in
> `services/notification/`.

---

## Current deployable

| Property | Value |
|---|---|
| Type | Modular Monolith |
| Service name | `barbersaas-backend` |
| Language / Framework | Java 21 / Spring Boot 3.3.4 |
| Deployment | Single JAR |
| Database | PostgreSQL (shared schema, tenant-isolated by `barbershop_id`) |
| Port | 8080 |
| API prefix | `/api/**` |

---

## Module registry

| # | Module | Package | Endpoint prefix | Bounded context | Extraction status |
|---|--------|---------|------------------|------------------|--------------------|
| 01 | `auth` | `com.barbersaas.auth` | `/api/auth/**` | Identity & Auth | Trigger-based — Phase 3 candidate (`auth-service`) |
| 02 | `barbershop` | `com.barbersaas.barbershop` | `/api/admin/barbershop` · `/api/super-admin/barbershops` | Barbershop Management | No extraction planned |
| 03 | `employee` | `com.barbersaas.employee` | `/api/admin/employees` | Barbershop Management | No extraction planned |
| 04 | `appointment` | `com.barbersaas.appointment` | `/api/client/appointments` · `/api/barber/appointments` · `/api/admin/appointments` | Appointment (Core Domain) | Trigger-based — Phase 3 candidate (`appointment-service`) |
| 05 | `schedule` | `com.barbersaas.schedule` | `/api/admin/schedules` | Schedule | No extraction planned — Shared Kernel with `appointment` (same DB tables); discarded as extraction candidate in `02-domain/domain-map.md` §5 until >10,000 barbershops |
| 06 | `loyalty` | `com.barbersaas.loyalty` | `/api/client/loyalty` · `/api/admin/loyalty` | Loyalty & Rewards (Core Domain) | Trigger-based — Phase 4 candidate (`loyalty-service`) |
| 07 | `finance` | `com.barbersaas.finance` | `/api/admin/finance` | Finance & Inventory | No extraction planned |
| 08 | `inventory` | `com.barbersaas.inventory` | `/api/admin/inventory` | Finance & Inventory | No extraction planned |
| 09 | `notification` | `com.barbersaas.notification` | `/api/notifications` | Notifications | **Phase 2 — anticipated for course delivery, weeks 5-7 (see `ADR-003`)** — see `services/notification/` |
| 10 | `plan` | `com.barbersaas.plan` | `/api/super-admin/plans` · `/api/public/plans` | Platform Administration | No extraction planned |
| 11 | `dashboard` | `com.barbersaas.dashboard` | `/api/admin/dashboard` · `/api/super-admin/dashboard` | Supporting | Trigger-based — Phase 4 candidate (`analytics-service`) |

Detail per module lives in `services/NN-module-name/` once it is documented individually.
Today that exists for `auth` (`services/02-auth-service/`) and `notification`
(`services/notification/`); the rest are covered only at the level of this catalog until
their own extraction is under discussion.

---

## Extraction roadmap

> All extractions are **trigger-based, not schedule-based**: no module is pulled out of the
> monolith until its trigger condition is measured in production — **except
> `notification-service`, an explicit, scoped exception documented in `ADR-003`.**

| Service | Extract from | Trigger condition | Phase |
|---|---|---|---|
| **`notification-service`** | `com.barbersaas.notification` | Anticipated for course delivery (weeks 5-7) — **not waiting for the production trigger** (FCM/email calls add >200ms to p95 appointment-creation latency, OR >5,000 active barbershops) | Phase 2 — scheduled (`ADR-003`) |
| `appointment-service` | `com.barbersaas.appointment` | CPU >70% sustained during peak hours in production, OR concurrent booking failures under load | Phase 3 — trigger-based, not yet scheduled |
| `auth-service` | `com.barbersaas.auth` | A second client application (web app, external API) needs to authenticate against the same identity store | Phase 3 — trigger-based, not yet scheduled |
| `loyalty-service` | `com.barbersaas.loyalty` | Loyalty program sold as a standalone product to businesses outside the barbershop sector | Phase 4 — trigger-based, not yet scheduled |
| `analytics-service` | `com.barbersaas.dashboard` | Advanced reporting, PDF exports, data warehouse needs — dashboards require queries >2s | Phase 4 — trigger-based, not yet scheduled |

---

## Inter-module communication

| From module | To module | Communication today | Communication once extracted |
|---|---|---|---|
| `appointment` | `notification` | In-process (`notificationService.notify(...)`) | HTTP POST or event publish |
| `appointment` | `loyalty` | In-process (`rewardCouponRepository` check in `create()`) | HTTP GET or event subscribe |
| `loyalty` | `notification` | In-process (`notificationService.notify(...)`) — **planned, not yet wired**, see `02-domain/domain-map.md` drift note | HTTP POST or event publish |
| `auth` | All modules | `TenantContext` (ThreadLocal) | JWT propagation via HTTP header |

---

## How to add a new module or extract one to a service

1. Assign the next available number in the module registry above.
2. If documenting the module in detail: `cp -r 09-microservices/_template/service
   09-microservices/services/NN-module-name`. Fill in the README, data-model, events, and
   decisions.
3. If extracting the module into a real service: write the extraction ADR first (see
   `ADR-003` as the template for how to justify an early/exception extraction, or cite the
   trigger condition if it was met naturally).
4. Create the OpenAPI contract in `07-api/contracts/openapi/NN-module-name.yaml`.
5. Update this catalog's module registry and extraction roadmap.
6. Update the diagram in `05-architecture/overview.md`.

---

## Correlations

- Architecture and modular monolith decision → `05-architecture/decisions/records/ADR-002-modular-monolith.md`
- Academic extraction exception → `05-architecture/decisions/records/ADR-003-academic-microservice-extraction.md`
- Bounded contexts and domain classification → `02-domain/domain-map.md`
- API contracts per module → `07-api/contracts/openapi/`
- System events → `02-domain/domain-events.md`
- Template for documenting a module in detail → `09-microservices/_template/service/`
- Full detail per module → `09-microservices/services/NN-module-name/`
- Source PRD (module table, extraction roadmap) → `PDR-BarberSaaS.md` §10 (repo WEEKLY)
