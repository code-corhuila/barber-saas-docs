# Functional Requirements (FR)

> Derived from the MVP feature list in `01-context/scope.md`, the bounded contexts in
> `02-domain/domain-map.md`, and the business invariants already verified against the
> real backend code in `02-domain/entities-and-rules.md`. Each FR is grounded in either
> a documented invariant or a controller endpoint that exists in
> `barbersaas-backend/src/main/java/com/barbersaas` — nothing here describes a feature
> that isn't real. Where an FR describes a documented gap (something not yet implemented),
> its Status column says so explicitly instead of claiming it works.
>
> **Source (HU)** links to the epic-level user stories in `04-requirements/user-stories.md`.

---

## Identity & Access (EP-001)

| ID | Module | Description | Source (HU) | Priority | Status |
|----|--------|-------------|------------|---------|--------|
| FR-001 | `auth` | The system must allow a new user to register with a unique, normalized (lowercase) email and an assigned role | HU-AUTH-001 | Must Have | ✅ Done |
| FR-002 | `auth` | The system must allow a registered user to log in and receive a JWT access token (24h expiration) and a refresh token (7-day expiration) | HU-AUTH-001 | Must Have | ✅ Done |
| FR-003 | `auth`, `notification` | The system must allow a user to recover their password via a 6-digit code sent by email | HU-AUTH-002 | Must Have | ✅ Done |
| FR-004 | `auth`, `barbershop` | The system must allow a prospective barbershop owner to self-register (`POST /api/auth/register-barbershop`) and receive a 60-day free trial | HU-AUTH-003 | Must Have | ✅ Done |

## Barbershop & Staff Management (EP-002)

| ID | Module | Description | Source (HU) | Priority | Status |
|----|--------|-------------|------------|---------|--------|
| FR-005 | `barberservice` | The system must allow `ADMIN_BARBERSHOP` to configure a service catalog (name, price, duration) scoped to their own barbershop | HU-SHOP-001 | Must Have | ✅ Done |
| FR-006 | `schedule` | The system must allow `ADMIN_BARBERSHOP` to define a barber's recurring weekly schedule, rejecting overlapping slots for the same barber/day | HU-SHOP-001 | Must Have | ✅ Done |
| FR-007 | `schedule` | The system must allow registering a schedule exception (day off / modified hours) that overrides — never merges with — the regular weekly schedule for that date | HU-SHOP-001 | Should Have | ✅ Done |

## Appointment Booking (EP-003)

| ID | Module | Description | Source (HU) | Priority | Status |
|----|--------|-------------|------------|---------|--------|
| FR-008 | `appointment` | The system must allow a `CLIENT` to book an appointment with a specific barber, service, date, and time, rejecting any booking that overlaps an existing appointment for the same barber (pessimistic DB lock) | HU-APPT-001 | Must Have | ✅ Done |
| FR-009 | `appointment` | The system must snapshot the service price at booking time (`priceAtBooking`) and never recalculate it, even if the service price later changes | HU-APPT-001 | Must Have | ✅ Done |
| FR-010 | `appointment`, `loyalty` | The system must automatically apply and consume the client's `ACTIVE` reward coupon (if any) at booking time | HU-APPT-001 | Should Have | ✅ Done |
| FR-011 | `appointment` | The system must allow a `CLIENT` to cancel a `PENDING`/`CONFIRMED` appointment, enforcing the barbershop's configured `cancellationPolicyHours` window (admins/barbers are exempt) | HU-APPT-002 | Must Have | ✅ Done |
| FR-012 | `appointment` | The system must automatically mark past `CONFIRMED` appointments as `NO_SHOW` via a daily scheduled job (01:00) | HU-APPT-002 | Should Have | ✅ Done |

## Loyalty & Rewards (EP-004)

| ID | Module | Description | Source (HU) | Priority | Status |
|----|--------|-------------|------------|---------|--------|
| FR-013 | `loyalty` | The system must allow `ADMIN_BARBERSHOP`/`BARBER` to grant a loyalty sticker to a client, optionally linked to a specific appointment, creating the client's `LoyaltyCard` on first grant if none exists | HU-LOY-001 | Must Have | ✅ Done |
| FR-014 | `loyalty` | The system must reject reward redemption when the client's `stickersCount` is below the barbershop's configured `stickersRequired` | HU-LOY-001 | Must Have | ✅ Done |
| FR-015 | `loyalty` | The system must issue an `ACTIVE` `RewardCoupon` on successful redemption, consumable automatically on the client's next booking (FR-010) | HU-LOY-001 | Must Have | ✅ Done |

## Financial Tracking (EP-005)

| ID | Module | Description | Source (HU) | Priority | Status |
|----|--------|-------------|------------|---------|--------|
| FR-016 | `finance` | The system must allow `ADMIN_BARBERSHOP` to record income and expense entries (`POST /api/admin/finance/records`) with amount, date, and description | HU-FIN-001 | Must Have | ✅ Done |
| FR-017 | `finance` | The system must reject finance records with an amount ≤ 0 regardless of type (`INCOME`/`EXPENSE`) | HU-FIN-001 | Should Have | ✅ Done |

## Inventory Management (EP-006)

| ID | Module | Description | Source (HU) | Priority | Status |
|----|--------|-------------|------------|---------|--------|
| FR-018 | `inventory` | The system must allow `ADMIN_BARBERSHOP` to track product stock and register stock movements (`POST /api/admin/inventory/products/{id}/movement`) | HU-INV-001 | Should Have | ✅ Done |
| FR-019 | `inventory` | The system must flag a product as a stock alert when `currentStock <= minStockAlert` | HU-INV-001 | Should Have | ✅ Done |

## Notifications (EP-007)

| ID | Module | Description | Source (HU) | Priority | Status |
|----|--------|-------------|------------|---------|--------|
| FR-020 | `notification` | The system must persist an in-app notification for appointment booking, confirmation, and cancellation, regardless of FCM push delivery outcome | HU-NOTIF-001 | Must Have | ✅ Done |
| FR-021 | `notification` | The system must send a reminder notification the day before a `CONFIRMED` appointment via a daily scheduled job (18:00) | HU-NOTIF-001 | Should Have | ✅ Done |
| FR-022 | `notification` | The system must send a notification when an appointment is marked `COMPLETED` | HU-NOTIF-001 | Should Have | 🔴 **Not implemented** — `AppointmentService.complete()` does not call `NotificationService` as of this review; see `02-domain/domain-events.md` |

## Platform Administration (EP-008)

| ID | Module | Description | Source (HU) | Priority | Status |
|----|--------|-------------|------------|---------|--------|
| FR-023 | `dashboard`, `barbershop` | The system must allow `SUPER_ADMIN` to view all registered barbershops and platform-wide metrics (`/api/super-admin/dashboard`, `/api/super-admin/barbershops`) | HU-SADMIN-001 | Must Have | ✅ Done |
| FR-024 | `plan` | The system must allow `SUPER_ADMIN` to manage subscription plans (Starter/Profesional/Premium) | HU-SADMIN-001 | Must Have | ✅ Done |
| FR-025 | `barbershop` | The system must allow only `SUPER_ADMIN` to transition a barbershop's status (`TRIAL` → `ACTIVE` → `SUSPENDED` → `CANCELLED`) via `PATCH /api/super-admin/barbershops/{id}/status` | HU-SADMIN-001 | Must Have | ✅ Done |
| FR-026 | `barbershop` | The system must automatically transition a barbershop from `TRIAL` to `SUSPENDED` when the 60-day trial expires without conversion | HU-SADMIN-001 | Should Have | 🔴 **Not implemented** — "trial expiration automation" is listed as in progress in `01-context/scope.md` (feature F-14) |

## Multi-tenancy & Security (EP-009 — cross-cutting)

| ID | Module | Description | Source (HU) | Priority | Status |
|----|--------|-------------|------------|---------|--------|
| FR-027 | `security` | Every request to a tenant-scoped endpoint must resolve `barbershopId` from the JWT via `JwtAuthenticationFilter` into `TenantContext` before reaching the service layer | HU-TENANT-001 | Must Have | ✅ Done |
| FR-028 | `security` | The system must reject any operation where the requested resource's `barbershopId` does not match the caller's `TenantContext` | HU-TENANT-001 | Must Have | ✅ Done |

---

## Correlations

- Non-functional requirements → `04-requirements/non-functional.md`
- User Stories these FRs trace to → `04-requirements/user-stories.md`
- Traceability to tests and services → `04-requirements/traceability-matrix.md`
- Source feature list → `01-context/scope.md`
- Business invariants these FRs encode → `02-domain/entities-and-rules.md`
