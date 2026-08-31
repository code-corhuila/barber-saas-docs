# Project Glossary — BarberSaaS

> **Instructions:** Define here all technical and business terms used in the project.
> This is the official dictionary — if there is ambiguity, this document wins.
> Add terms throughout the project, not just at the beginning.

---

## How to use this glossary

1. Before using a technical or business term in code, docs, or conversations: look it up here.
2. If it is not here: add it with its definition.
3. If there is disagreement about the definition: discuss it with the team and update this document.

---

## Domain Terms

| Term | Definition | Notes / Synonyms |
|---------|------------|------------------|
| **Barbershop** | A registered business tenant on the platform — the unit of data isolation (`barbershop_id`). Has its own status, subscription plan, staff, and service catalog. | *Synonyms:* shop, tenant |
| **Appointment (or Booking)** | A scheduled service between a client and a specific barber, at a specific barbershop, date, and time slot. | *Synonyms to AVOID: Turn, Event.* "Slot" has a distinct technical meaning — see below |
| **Appointment Status** | Phase of an appointment's lifecycle: `PENDING` → `CONFIRMED` → `IN_PROGRESS` → `COMPLETED`, or `CANCELLED` / `NO_SHOW`. | Full state machine documented in `02-domain/entities-and-rules.md` |
| **Client** | A platform user with role `CLIENT`. Searches barbershops, books/cancels/reschedules appointments, earns and redeems loyalty rewards. | |
| **Barber** | An employee user with role `BARBER`. Checks their daily agenda, performs services, grants loyalty stickers, registers walk-in clients. | *Synonyms to AVOID: Stylist, Hairdresser* |
| **Admin (Barbershop Admin)** | The owner of a barbershop, role `ADMIN_BARBERSHOP`. Manages employees, schedules, service catalog, finances, loyalty program, and inventory for their own shop only. | *Synonym: shop owner* |
| **Super Admin** | Internal BarberSaaS platform team member, role `SUPER_ADMIN`. Creates barbershop accounts, manages subscription plans, activates/suspends/cancels accounts, monitors platform-wide metrics. | Operates across all tenants — the only role not scoped to one barbershop |
| **Walk-in** | An appointment created by an admin directly on a barber's agenda for a client who arrived without a prior booking, without requiring that client to register an account. | Marked "in progress" in `01-context/scope.md` |
| **Service Catalog** | The list of services a barbershop offers (name, price, duration), configured per barbershop by its admin. | Entity: `BarberService` |
| **Loyalty Card** | A client's accumulated sticker count and redemption history at one specific barbershop. | One card per client per barbershop |
| **Sticker** | A loyalty point granted by a barber or admin after a completed service. | |
| **Reward / Reward Coupon** | The benefit a client earns after accumulating the stickers required by their barbershop. Redemption issues an `ACTIVE` Reward Coupon, applied as a 100% discount on the client's next booking. | |
| **Subscription Plan** | One of three paid tiers a barbershop is on — **Starter** (up to 2 barbers), **Profesional** (up to 5), **Premium** (unlimited) — differing in monthly COP price and max barbers. | See `01-context/overview_en.md` → Monetization model |
| **Trial** | The 60-day free period a barbershop gets at self-registration before it must convert to a paid plan. | Status `TRIAL` in the barbershop lifecycle (`TRIAL` → `ACTIVE` → `SUSPENDED` → `CANCELLED`) |

---

## Technical Project Terms

| Term | Definition |
|---------|------------|
| **Modular Monolith** | The backend's chosen architecture (**ADR-002**): a single Spring Boot 3 / Java 21 deployable unit, internally organized into bounded-context packages (`auth`, `barbershop`, `appointment`, `schedule`, `loyalty`, `finance`, `inventory`, `notification`, `plan`, `dashboard`), with no direct cross-context repository access. Chosen over microservices-from-day-one given the 1-developer team size and current load. |
| **Bounded Context** | A boundary within which a domain model has one consistent meaning — e.g., "Barber" means an `Employee` in the Barbershop Management context but a `BarberProfile` performing the service in the Appointment context. Full map in `02-domain/domain-map.md`. |
| **Multi-tenancy** | The isolation model that keeps one barbershop's data invisible to another: a `barbershop_id` discriminator column on every tenant-scoped table, enforced in the service layer — **not** a separate database or schema per tenant. |
| **TenantContext** | A `ThreadLocal` populated from the JWT by `JwtAuthenticationFilter` on every request, holding `userId`, `barbershopId`, and `role` for the duration of that request. The mechanism every query relies on to enforce multi-tenancy. |
| **RBAC** | Role-Based Access Control, enforced via Spring Security `@PreAuthorize("hasRole(...)")` at the controller layer. BarberSaaS has exactly 4 fixed roles (`SUPER_ADMIN`, `ADMIN_BARBERSHOP`, `BARBER`, `CLIENT`) — no custom or extensible permission model. |
| **Aggregate / Entity / Value Object** | DDD tactical building blocks used to describe the domain model (e.g., `Appointment` and `LoyaltyCard` as aggregate roots, `Money` and `TimeSlot` as value objects). Full breakdown in `02-domain/entities-and-rules.md`. |
| **Extraction trigger** | A documented, measurable condition (e.g., a module's p95 latency or a barbershop-count threshold) that would justify pulling a bounded-context module out of the monolith into its own service later. Referenced in **ADR-002**; the modular structure exists specifically to keep this path open. |

---

## Acronyms

| Acronym | Meaning |
|----------|-------------|
| **PRD** | Product Requirements Document — the BarberSaaS PRD v1.0 (August 2026) is the source document cited throughout `01-context`, `02-domain`, and `05-architecture` |
| **MVP** | Minimum Viable Product |
| **RBAC** | Role-Based Access Control |
| **SaaS** | Software as a Service |
| **JWT** | JSON Web Token (used for authentication — see `00-governance/security-policy.md`) |
| **FCM** | Firebase Cloud Messaging — push notification delivery to the mobile app |
| **API** | Application Programming Interface |
| **CRUD** | Create, Read, Update, Delete |
| **DTO** | Data Transfer Object |
| **FR** | Functional Requirement |
| **NFR** | Non-Functional Requirement |
| **CI/CD** | Continuous Integration / Continuous Delivery |
| **PR** | Pull Request |
| **COP** | Colombian Peso — the only currency supported in the MVP (see `Money` value object, `02-domain/entities-and-rules.md`) |
