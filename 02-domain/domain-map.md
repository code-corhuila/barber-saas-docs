# Domain Map — Bounded Contexts

> **BarberSaaS** · v1.0 · August 2026
> Built with Event Storming sessions and validated against the working implementation.

---

## 1. Business Domain Overview

BarberSaaS is a multi-tenant SaaS platform that digitizes the full operational cycle of barbershops in Colombia. The system covers every stage of a barbershop's day-to-day business: a client discovers and books an appointment with a specific barber, the barber manages their daily agenda and marks services as completed, the shop owner tracks revenue, controls staff schedules, and runs a loyalty program to retain clients. A platform administrator oversees all registered barbershops, manages subscription plans, and controls trial and billing cycles. Every barbershop operates in complete isolation — one shop can never see the data of another.

---

## 2. Bounded Contexts

---

### Bounded Context: Identity & Auth

| Field | Value |
|---|---|
| **Name** | Identity & Auth |
| **Responsibility** | Registration, login, JWT issuance, tenant resolution, and password recovery for all user roles |
| **Owner** | Carlos Leal (backend: `com.barbersaas.auth`) |
| **Service** | `barber-saas-identity-auth-api` (ADR-004); prototype source `com.barbersaas.auth` |
| **Database** | `identity-auth-db` (PostgreSQL) — `app_user`, `refresh_token`, `password_reset_token` |
| **Ubiquitous language** | User, Role, JWT, Tenant claim, JWKS, RefreshToken, PasswordResetToken |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **User** | Any authenticated person with a role and an account | Yes — in Appointment it becomes `client`, `barber` |
| **Role** | `CLIENT`, `BARBER`, `ADMIN_BARBERSHOP`, `SUPER_ADMIN` | No — role is a platform-wide concept |
| **Tenant claim** | The `barbershopId` claim of the JWT; every service filters tenant data by it (the prototype kept it in a ThreadLocal `TenantContext`) | No |
| **Token** | A JWT signed with RS256 containing claims: `sub`, `role`, `barbershopId` | Yes — in Loyalty, `token` is a 6-digit password reset code |

---

### Bounded Context: Barbershop Management

| Field | Value |
|---|---|
| **Name** | Barbershop Management |
| **Responsibility** | Lifecycle of a barbershop tenant: creation, plan assignment, trial management, status transitions (TRIAL → ACTIVE → SUSPENDED → CANCELLED), and employee (barber) administration |
| **Owner** | Carlos Leal (backend: `com.barbersaas.barbershop`, `com.barbersaas.employee`) |
| **Service** | `barber-saas-barbershop-api` (ADR-004); prototype source `com.barbersaas.barbershop`, `com.barbersaas.employee` |
| **Database** | `barbershop-db` (PostgreSQL) — `barbershop`, `service`, `barber_profile`, `barber_specialty`. Staff accounts live in identity-auth (`app_user`), plans in platform-admin; referenced by id |
| **Ubiquitous language** | Barbershop, SubscriptionPlan, BarberProfile, BarbershopStatus, TrialPeriod |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **Barbershop** | A registered business tenant with its own plan, status, and data scope | Yes — in Appointment it is just the `barbershopId` FK |
| **Employee** | A `BARBER` or `ADMIN_BARBERSHOP` user belonging to a barbershop | Yes — in Appointment it is called `barber` |
| **BarberProfile** | Extended profile of a barber: bio, photo, linked to a `User` | No |
| **Plan** | A `SubscriptionPlan` with price (COP), max barbers, and features | Yes — in Super Admin it is managed; in Barbershop it is consumed |
| **Trial** | A 60-day free period starting at `created_at`, tracked via `trial_ends_at` | No |

---

### Bounded Context: Appointment

| Field | Value |
|---|---|
| **Name** | Appointment |
| **Responsibility** | The complete lifecycle of a service booking: availability calculation, anti-double-booking, state machine (PENDING → CONFIRMED → IN_PROGRESS → COMPLETED / CANCELLED / NO_SHOW), rescheduling, and walk-in client tracking |
| **Owner** | Carlos Leal (backend: `com.barbersaas.appointment`) |
| **Service** | `barber-saas-appointment-api` (ADR-004); prototype source `com.barbersaas.appointment` |
| **Database** | `appointment-db` (PostgreSQL) — `appointment`. Schedules belong to Schedule and services to Barbershop; read through their APIs (OQ-09) |
| **Ubiquitous language** | Appointment, AppointmentStatus, Slot, AvailabilityWindow, CancellationPolicy, WalkIn |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **Appointment** | A confirmed time slot between a client and a barber for a specific service | No |
| **Slot** | A computed available time window for a barber on a given date | No |
| **Client** | A `User` with role `CLIENT` who books the appointment | Yes — in Identity it is just a `User` |
| **Barber** | A `BarberProfile` who performs the service | Yes — in Barbershop Management it is an `Employee` |
| **WalkIn** | An appointment created by the admin for a client who arrived without prior booking, tracked without requiring client registration | No |
| **CancellationPolicy** | Number of hours before the appointment within which cancellation is allowed (configurable per barbershop) | No |
| **PriceAtBooking** | The service price captured at reservation time — may be `0` if a reward coupon is applied | Yes — in Loyalty it becomes the coupon redemption signal |

---

### Bounded Context: Schedule

| Field | Value |
|---|---|
| **Name** | Schedule |
| **Responsibility** | Definition and management of a barber's recurring weekly working hours and one-off exceptions (days off, modified hours). Input to the availability algorithm in Appointment. |
| **Owner** | Carlos Leal (backend: `com.barbersaas.schedule`) |
| **Service** | `barber-saas-schedule-api` (ADR-004); prototype source `com.barbersaas.schedule` |
| **Database** | `schedule-db` (PostgreSQL) — `barber_schedule`, `schedule_exception` |
| **Ubiquitous language** | WeeklySchedule, DayOfWeek, TimeSlot, ScheduleException, DayOff |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **Schedule** | A barber's recurring availability definition (day + start/end times) | No |
| **ScheduleException** | A date where the barber deviates from the regular schedule (full day off or modified hours) | No |

---

### Bounded Context: Loyalty & Rewards

| Field | Value |
|---|---|
| **Name** | Loyalty & Rewards |
| **Responsibility** | Sticker-based loyalty card per client per barbershop, reward redemption, automatic coupon generation and application on the client's next booking |
| **Owner** | Carlos Leal (backend: `com.barbersaas.loyalty`) |
| **Service** | `barber-saas-loyalty-api` (ADR-004); prototype source `com.barbersaas.loyalty` |
| **Database** | `loyalty-db` (PostgreSQL) — `loyalty_card`, `loyalty_rewards_config`, `loyalty_transaction`, `reward_coupon` |
| **Ubiquitous language** | LoyaltyCard, Sticker, Reward, Redemption, RewardCoupon, CouponStatus |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **Sticker** | A loyalty point granted by a barber or admin after a completed service | No |
| **LoyaltyCard** | A client's sticker count and redemption history within a specific barbershop | No |
| **Reward** | The benefit a client earns after accumulating the required stickers (defined by the barbershop) | No |
| **RewardCoupon** | An `ACTIVE` coupon generated on redemption — automatically applied as a 100% discount on the client's next booking | No |
| **Redemption** | The act of exchanging accumulated stickers for a reward, which creates a `RewardCoupon` | No |

---

### Bounded Context: Notifications

| Field | Value |
|---|---|
| **Name** | Notifications |
| **Responsibility** | Delivery of in-app, push (FCM), and email notifications triggered by domain events. Persists all notifications to DB regardless of delivery outcome (graceful degradation). |
| **Owner** | Carlos Leal (backend: `com.barbersaas.notification`) |
| **Service** | `barber-saas-notifications-api` (ADR-004); prototype source `com.barbersaas.notification` |
| **Database** | `notifications-db` (MongoDB, ADR-006) — collections `notification`, `device_token` |
| **Ubiquitous language** | Notification, NotificationType, DeviceToken, PushDelivery, EmailDelivery |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **Notification** | A persisted in-app message (title, body, type, read status) for a specific user | No |
| **DeviceToken** | The FCM registration token for a user's Android or iOS device | No |
| **PushDelivery** | An attempt to send a push notification via Firebase Admin SDK — may fail without blocking the triggering operation | No |

---

### Bounded Context: Finance & Inventory

| Field | Value |
|---|---|
| **Name** | Finance & Inventory |
| **Responsibility** | Manual recording of income and expenses per barbershop, product stock management, and restock alerts |
| **Owner** | Carlos Leal (backend: `com.barbersaas.finance`, `com.barbersaas.inventory`) |
| **Service** | `barber-saas-finance-inventory-api` (ADR-004); prototype source `com.barbersaas.finance`, `com.barbersaas.inventory` |
| **Database** | `finance-inventory-db` (PostgreSQL) — `finance_record`, `inventory_product`, `inventory_movement` |
| **Ubiquitous language** | FinanceRecord, RecordType (INCOME/EXPENSE), InventoryProduct, StockAlert, Movement |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **FinanceRecord** | A manually registered income or expense entry with amount, date, and description | No |
| **RecordType** | `INCOME` or `EXPENSE` — determines the financial sign of the record | No |
| **InventoryProduct** | A physical product tracked by quantity (e.g., hair gel, clippers) | No |
| **StockAlert** | Triggered when `currentStock <= minStockAlert` for a product | No |

---

### Bounded Context: Platform Administration (Super Admin)

| Field | Value |
|---|---|
| **Name** | Platform Administration |
| **Responsibility** | Cross-tenant oversight: creating and managing barbershops, defining subscription plans, monitoring platform-wide metrics (total barbershops, clients, revenue), and managing trial/billing states |
| **Owner** | Carlos Leal (backend: `com.barbersaas.barbershop.SuperAdminBarbershopController`, `com.barbersaas.plan`) |
| **Service** | `barber-saas-platform-admin-api` (ADR-004); prototype source `SuperAdminBarbershopController`, `com.barbersaas.plan` |
| **Database** | `platform-admin-db` (PostgreSQL) — owns `subscription_plan`; reads and changes barbershops only through Barbershop's API (OQ-10), never another database |
| **Ubiquitous language** | PlatformDashboard, SubscriptionPlan, BarbershopStatus, TrialExpiry |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **Platform** | The entire BarberSaaS system viewed as a product sold to barbershop owners | No |
| **Plan** | A `SubscriptionPlan` with price (COP/month), max barbers, and features — managed by Super Admin | Yes — in Barbershop Management it is an assigned contract |
| **Suspension** | Setting a barbershop's `status` to `SUSPENDED`, blocking all operations for that tenant | No |

---

### Prototype entities not yet in the MVP scope — proposed placement

The prototype has four tables with working code (`reviews`, `promotions`,
`client_favorites`, `gallery_images`, see the prototype schema in the history of
`06-data/models.md`) that are not in `01-context/scope.md` and had no context here. None is
ported to a `-db` until a user story brings it into scope (`00-governance/definition-of-ready.md`).
When that happens, this is where each one belongs:

| Entity | Proposed context | Why | Ubiquitous language |
|---|---|---|---|
| **Review** | Barbershop Management | A client rates a barber and the barbershop after a `COMPLETED` appointment; it feeds `barber_profile.rating_avg` / `rating_count`, which Barbershop already exposes. It learns about completion from `AppointmentCompleted` | Review, Rating (1–5) |
| **Promotion** | Barbershop Management | A discount on the barbershop's own catalog (`PERCENTAGE`, `FIXED_AMOUNT`, `TWO_FOR_ONE`), valid for a date range; the price a booking snapshots comes from Barbershop | Promotion, DiscountType, ValidityPeriod |
| **ClientFavorite** | Barbershop Management | A client's bookmark of a barbershop in the discovery catalog (`DEC-SHOP-02`) | Favorite |
| **GalleryImage** | Barbershop Management | The barbershop's and its barbers' showcase photos, shown with the public profile | GalleryImage, Caption |

All four are **supporting** subdomains (§4): they help a barbershop sell, but no core rule
(booking, loyalty) depends on them. Placing them in Barbershop keeps them next to the data
they decorate and adds no new service. This placement is a proposal for the team to confirm
when the first of these stories is refined.

## 3. Context Map

```
┌─────────────────────────┐
│   Identity & Auth       │  ← Upstream to ALL contexts
│   /api/v1/auth          │    (JWT RS256 claims carry
│   Role, JWT, Tenant     │     sub, barbershopId, role)   
└────────────┬────────────┘
             │ U → D (JWT claims)
             ▼
┌────────────────────────────────────────────────────────────────────┐
│                    Barbershop Management                           │
│   /api/v1/barbershops · /services · /barbers                       │
│   Barbershop, BarberProfile, SubscriptionPlan, BarbershopStatus    │
└──────┬──────────────────────────┬──────────────────────────────────┘
       │ U → D (barberProfileId)  │ U → D (barberId, serviceId)
       ▼                          ▼
┌──────────────────┐    ┌──────────────────────────┐
│    Schedule      │    │       Appointment        │
│  barber_schedules│───▶│  availability algorithm  │
│  exceptions      │API │  state machine (6 states)│
└──────────────────┘    │  walk-in support         │
                        └──────────┬───────────────┘
                                   │ U → D (event AppointmentCompleted
                                   │ through the outbox; see relationship
                                   │ table below)
                        ┌──────────▼───────────────┐
                        │    Loyalty & Rewards      │
                        │  stickers, coupons        │
                        │  RewardCoupon hooks       │
                        │  back into Appointment    │
                        │  (priceAtBooking = 0)     │
                        └──────────┬───────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
            ┌──────────┐  ┌──────────────┐  ┌──────────────────┐
            │Notifica- │  │  Finance &   │  │    Platform      │
            │tions     │  │  Inventory   │  │  Administration  │
            │FCM+email │  │  COP records │  │  Super Admin     │
            │in-app    │  │  stock alerts│  │  cross-tenant    │
            └──────────┘  └──────────────┘  └──────────────────┘
```

### Relationship table

> Since ADR-004 every context is its own service and database, so no relation is an
> in-process call or a shared table anymore. The prototype's mechanism is kept in the last
> column because it is the behaviour being ported.

| Context A | Relation | Context B | Channel (target) | Contract | Prototype mechanism |
|---|---|---|---|---|---|
| Identity & Auth | U → D | All other contexts | JWT RS256 in `Authorization`, validated by each service with the JWKS | `07-api/authentication.md` (claims `sub`, `role`, `barbershopId`) | `TenantContext` (ThreadLocal) |
| Barbershop Management | U → D | Appointment | `barberId` / `serviceId` as UUIDs; price and duration read from Barbershop's API at booking | `barbershop-service.yaml` | JPA FK |
| Barbershop Management | U → D | Schedule | `barberProfileId` as UUID, no FK | `barbershop-service.yaml` | JPA FK |
| Schedule | U → D | Appointment | Appointment asks Schedule whether a slot is inside working hours; Schedule learns bookings from Appointment's events (direction still open, OQ-09) | `schedule-service.yaml`, `appointment-service.yaml` | Shared Kernel on the same tables |
| Appointment | U → D | Loyalty & Rewards | Event `AppointmentCompleted` through Appointment's outbox; Loyalty grants the sticker once per appointment | `02-domain/domain-events.md`, `uq_loyalty_transaction_sticker_per_appointment` | In-process `grantStickerForCompletedAppointment()` |
| Loyalty & Rewards | U → D | Appointment | Appointment checks and consumes an `ACTIVE` coupon through Loyalty's API (`/api/v1/loyalty/coupons/{id}/use`) | `loyalty-service.yaml` | In-process `RewardCoupon` check |
| Appointment | U → D | Notifications | Events `AppointmentConfirmed`, `AppointmentCancelled`, `AppointmentCompleted`; reminder produced by the worker | `notification-service.yaml` (created from events) | In-process `NotificationService.notify()` |
| Loyalty & Rewards | U → D | Notifications | Events `StickerGranted`, `RewardRedeemed` | `02-domain/domain-events.md` | In-process `notify()` |
| Identity & Auth | U → D | Notifications | Event `PasswordResetRequested` (e-mail with the code) | `auth-service.yaml` | In-process mail call |
| Barbershop Management | U → D | Platform Administration | Platform admin changes a barbershop's status and plan through Barbershop's API with a service token (OQ-10) | `platform-admin-service.yaml` | Same tables |
| All contexts | U → D | Finance & Inventory | Manual registration by the admin; appointment revenue linked by `relatedAppointmentId` | `finance-inventory-service.yaml` | Same tables |

---

## 4. Core Domain, Supporting, Generic

| Bounded Context | Type | Justification |
|---|---|---|
| **Appointment** | **Core Domain** | The anti-double-booking mechanism, the 6-state machine, and walk-in support are the primary competitive differentiator. No off-the-shelf solution handles Colombian barbershop workflows. |
| **Loyalty & Rewards** | **Core Domain** | The sticker → coupon → automatic discount pipeline is a key retention feature specific to BarberSaaS. Drives repeat visits and differentiates from WhatsApp-based competitors. |
| **Barbershop Management** | **Supporting** | Essential but not unique — multi-tenant CRUD with plan management. Could eventually be handled by a generic SaaS platform, but is built in-house to keep full control over the onboarding flow and trial logic. |
| **Schedule** | **Supporting** | Barber schedule configuration supports the Appointment core but is not itself a differentiator. |
| **Finance & Inventory** | **Supporting** | Necessary for shop owner visibility but not the reason barbershops choose BarberSaaS. |
| **Platform Administration** | **Supporting** | Internal tooling for the BarberSaaS team — no direct user-facing value, but critical for operations. |
| **Identity & Auth** | **Generic** | Standard JWT authentication. Uses BCrypt + Spring Security — commodity patterns. Will never be a competitive advantage. |
| **Notifications** | **Generic** | FCM + Gmail SMTP — uses off-the-shelf services. The integration layer is custom but the capability itself is commodity. |

---

## 5. Modeling decisions

### How this map was built

- **Method:** Solo architectural walkthrough + incremental refinement as each bounded context was implemented (Phases 1–16 of the BarberSaaS development plan)
- **Tool:** Implementation-driven — bounded contexts map directly to Java packages (`com.barbersaas.[context]`)
- **Iterations:** v1 (Phase 1 — Auth + Barbershop), v2 (Phase 5 — Appointment added), v3 (Phase 6 — Loyalty added), current v4 (Phase 16 — full system including walk-in and password recovery)

### Key decisions and discarded alternatives

| Decision | Discarded alternative | Reason |
|---|---|---|
| Schedule is its own service and database, not a Shared Kernel with Appointment | Keep the shared tables (prototype) | ADR-004 and norm 7.3: no service reads another domain's database. How availability learns about bookings is OQ-09 |
| Loyalty reacts to `AppointmentCompleted` as an event | In-process call inside the booking transaction (prototype) | Two databases cannot share a transaction (norm 7.5); the outbox plus a unique sticker per appointment make delivery safe to repeat |
| One database per bounded context (ADR-004, ADR-006) | Single PostgreSQL schema with `barbershop_id` (prototype, ADR-002 — superseded) | Course requirement and norm 7.1; tenant isolation now repeated in each service (`07-api/authentication.md`) |

---

## 6. How to update this map

1. Before adding a new feature, identify which bounded context it belongs to.
2. If a term starts meaning different things in different places — that is a signal that a context needs to split.
3. This map must stay synchronized with:
   - Microservice catalog → `09-microservices/service-catalog.md`
   - C4 system diagram → `05-architecture/overview.md`
   - ADRs for service extraction → `05-architecture/decisions/`
4. Run an Event Storming session any time the domain changes significantly (e.g., adding a payments context, splitting Appointment from Availability).

> **Correlation:** Bounded contexts here →
> Modules in `09-microservices/service-catalog.md` →
> C4 diagrams in `08-uml/` →
> Extraction ADRs in `05-architecture/decisions/`
