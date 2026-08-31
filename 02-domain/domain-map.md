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
| **Module** | `auth` (part of the modular monolith — not a separate service in MVP) |
| **Database** | PostgreSQL — tables: `users`, `password_reset_tokens` |
| **Ubiquitous language** | User, Role, JWT, TenantContext, PasswordResetToken |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **User** | Any authenticated person with a role and an account | Yes — in Appointment it becomes `client`, `barber` |
| **Role** | `CLIENT`, `BARBER`, `ADMIN_BARBERSHOP`, `SUPER_ADMIN` | No — role is a platform-wide concept |
| **TenantContext** | ThreadLocal store of `userId`, `barbershopId`, `role` for the current HTTP request | No |
| **Token** | A signed JWT containing claims: `userId`, `role`, `barbershopId` | Yes — in Loyalty, `token` is a 6-digit password reset code |

---

### Bounded Context: Barbershop Management

| Field | Value |
|---|---|
| **Name** | Barbershop Management |
| **Responsibility** | Lifecycle of a barbershop tenant: creation, plan assignment, trial management, status transitions (TRIAL → ACTIVE → SUSPENDED → CANCELLED), and employee (barber) administration |
| **Owner** | Carlos Leal (backend: `com.barbersaas.barbershop`, `com.barbersaas.employee`) |
| **Module** | `barbershop` + `employee` (modular monolith) |
| **Database** | PostgreSQL — tables: `barbershops`, `users` (ADMIN_BARBERSHOP / BARBER roles), `barber_profiles`, `subscription_plans` |
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
| **Module** | `appointment` (modular monolith) |
| **Database** | PostgreSQL — tables: `appointments`, `barber_services`, `barber_schedules`, `schedule_exceptions` |
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
| **Module** | `schedule` (modular monolith) |
| **Database** | PostgreSQL — tables: `barber_schedules`, `schedule_exceptions` |
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
| **Module** | `loyalty` (modular monolith) |
| **Database** | PostgreSQL — tables: `loyalty_cards`, `loyalty_rewards_config`, `loyalty_transactions`, `reward_coupons` |
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
| **Module** | `notification` (modular monolith) |
| **Database** | PostgreSQL — tables: `notifications`, `device_tokens` |
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
| **Module** | `finance` + `inventory` (modular monolith) |
| **Database** | PostgreSQL — tables: `finance_records`, `inventory_products`, `inventory_movements` |
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
| **Module** | Part of `barbershop` + `plan` (modular monolith, accessed via `/api/super-admin/**`) |
| **Database** | PostgreSQL — reads across all tenant tables; owns `subscription_plans` |
| **Ubiquitous language** | PlatformDashboard, SubscriptionPlan, BarbershopStatus, TrialExpiry |

**Terms in this context:**

| Term | Meaning in THIS context | Different in another context? |
|---|---|---|
| **Platform** | The entire BarberSaaS system viewed as a product sold to barbershop owners | No |
| **Plan** | A `SubscriptionPlan` with price (COP/month), max barbers, and features — managed by Super Admin | Yes — in Barbershop Management it is an assigned contract |
| **Suspension** | Setting a barbershop's `status` to `SUSPENDED`, blocking all operations for that tenant | No |

---

## 3. Context Map

```
┌─────────────────────────┐
│   Identity & Auth       │  ← Upstream to ALL contexts
│   /api/auth/**          │    (JWT + TenantContext resolve
│   Role, JWT, Tenant     │     userId, barbershopId, role)
└────────────┬────────────┘
             │ U → D (JWT claims)
             ▼
┌────────────────────────────────────────────────────────────────────┐
│                    Barbershop Management                           │
│   /api/admin/** · /api/super-admin/**                              │
│   Barbershop, BarberProfile, SubscriptionPlan, BarbershopStatus    │
└──────┬──────────────────────────┬──────────────────────────────────┘
       │ U → D (barbershopId FK)  │ U → D (barberProfileId FK)
       ▼                          ▼
┌──────────────────┐    ┌──────────────────────────┐
│    Schedule      │    │       Appointment        │
│  barber_schedules│───▶│  availability algorithm  │
│  exceptions      │ SK │  state machine (6 states)│
└──────────────────┘    │  walk-in support         │
                        └──────────┬───────────────┘
                                   │ U → D (planned — grantSticker() is a manual
                                   │ staff action today, not auto-triggered; see
                                   │ relationship table below)
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

| Context A | Relation | Context B | Channel | Contract |
|---|---|---|---|---|
| Identity & Auth | U → D | All other contexts | JWT (HTTP header) | `TenantContext` (ThreadLocal) |
| Barbershop Management | U → D | Appointment | `barbershopId` FK in `appointments` | JPA entity reference |
| Barbershop Management | U → D | Schedule | `barberProfileId` FK in `barber_schedules` | JPA entity reference |
| Schedule | Shared Kernel | Appointment | Shared `barber_schedules` / `schedule_exceptions` tables | JPA — same PostgreSQL schema |
| Appointment | U → D (target design, not yet wired) | Loyalty & Rewards | `grantSticker()` — **not actually called automatically on COMPLETED as of this review.** `AppointmentService.complete()` only changes status; sticker granting requires a separate explicit call to `POST /api/loyalty/grant-sticker` by staff. See `02-domain/domain-events.md` → `AppointmentCompleted` drift note | In-process Java method call (manual trigger today, not event-driven) |
| Loyalty & Rewards | U → D | Appointment | `RewardCoupon` check in `AppointmentService.create()` — confirmed in code | In-process Java method call |
| Appointment | U → D | Notifications | `NotificationService.notify()` after `create()`, `confirm()`, `cancel()`, and the daily reminder job — confirmed in code. **Not** called after `complete()` or the NO_SHOW auto-marking job | In-process Java method call |
| Loyalty & Rewards | U → D (target design, not yet wired) | Notifications | "Notify client on sticker granted / reward redeemed" — **not implemented.** `LoyaltyService.grantSticker()` and `.redeemReward()` do not call `NotificationService` as of this review | In-process Java method call (planned) |
| All contexts | U → D | Finance & Inventory | Manual registration by admin; appointment revenue logged | HTTP REST (admin screens) |
| All contexts | U → D | Platform Administration | Cross-tenant reads via Super Admin dashboard | HTTP REST (`/api/super-admin/**`) |

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
| Keep Schedule as a Shared Kernel with Appointment (same DB tables) | Extract Schedule as a separate microservice | At current scale, the shared table is simpler. No independent scaling need identified yet. Trigger for extraction: >10,000 barbershops. |
| Loyalty hooks directly into Appointment via in-process call | Event-driven (publish `AppointmentCompleted` event, Loyalty subscribes) | No message broker in MVP. In-process call is synchronous and easier to reason about. Event-driven will be introduced when Notifications moves to its own service. |
| Single PostgreSQL schema for all bounded contexts | One database per bounded context | Operational simplicity for MVP. The multi-tenancy discriminator (`barbershop_id`) provides logical isolation. Physical separation is a future ADR trigger. |

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
