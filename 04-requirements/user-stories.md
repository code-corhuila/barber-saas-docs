# User Stories — Backlog

> Formalized from the epic-level backlog in `03-product/product-backlog.md`. Each HU's
> acceptance criteria are grounded in the business invariants already documented and
> verified against the real backend code in `02-domain/entities-and-rules.md` and
> `02-domain/domain-map.md` — no new business rule is invented here.
>
> **What is deliberately left blank:** Story Points and Target Sprint. No real estimation
> session has happened yet (see `00-governance/agile-conventions.md` → Estimation). Marking
> these as a specific number would misrepresent planning work that hasn't occurred. Status
> reflects real implementation state, verified against the codebase.

---

## Backlog status

| Cut | Epics covered | Total HUs | Status |
|-----|--------|-----------|--------|
| This pass | EP-001 … EP-009 (all) | 10 | 9 Done, 1 partially Done (see HU-APPT gaps below) |

This is a first formalization pass — one representative HU per epic (two for Appointment,
the core domain). Each HU below is large enough that, per
`00-governance/definition-of-ready.md`, it should be **split into smaller HUs during
refinement** before being scheduled into a sprint — it is documented here at
feature-granularity, not sprint-ready granularity.

---

## Epics

| ID | Epic | Description |
|----|------|-------------|
| EP-001 | Identity & Access | See `03-product/product-backlog.md` |
| EP-002 | Barbershop & Staff Management | See `03-product/product-backlog.md` |
| EP-003 | Appointment Booking | See `03-product/product-backlog.md` |
| EP-004 | Loyalty & Rewards | See `03-product/product-backlog.md` |
| EP-005 | Financial Tracking | See `03-product/product-backlog.md` |
| EP-006 | Inventory Management | See `03-product/product-backlog.md` |
| EP-007 | Notifications | See `03-product/product-backlog.md` |
| EP-008 | Platform Administration | See `03-product/product-backlog.md` |
| EP-009 | Multi-tenancy & Security | See `03-product/product-backlog.md` |

---

## User Stories

### HU-AUTH-001 — Register and log in with a role-based account {#HU-AUTH-001}

**Epic:** EP-001

> **As** a new user (client, barber, or barbershop admin)
> **I want** to register an account and log in
> **so that** I can access the features available to my role without sharing credentials with anyone else

**Acceptance Criteria:**

```gherkin
Scenario 1: Successful registration
  Given a unique, valid email and the required fields
  When the user submits registration
  Then an account is created with the assigned role
  And the email is normalized to lowercase before being stored

Scenario 2: Duplicate email rejected
  Given an email that is already registered on the platform
  When a user submits registration with that email
  Then the system rejects it with a validation error

Scenario 3: Successful login
  Given valid credentials for an existing account
  When the user logs in
  Then the system returns a JWT access token (24h expiration) and a refresh token (7-day expiration)
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.auth`
**Endpoint(s) implemented:** `POST /api/auth/register`, `POST /api/auth/login`
**Required permissions:** None (public endpoints)

**Definition of Done:** See `00-governance/definition-of-done.md`.

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled — split into smaller HUs during refinement first |
| Status | ✅ Done |
| Dependencies | None |
| Affected service(s) | `auth` |

---

### HU-AUTH-002 — Recover a forgotten password via email {#HU-AUTH-002}

**Epic:** EP-001

> **As** a registered user who forgot their password
> **I want** to reset it using a code sent to my email
> **so that** I can regain access without contacting support

**Acceptance Criteria:**

```gherkin
Scenario 1: Request a reset code
  Given a registered email address
  When the user requests a password reset
  Then a 6-digit code is generated and emailed via Gmail SMTP

Scenario 2: Reset with a valid code
  Given a valid, unexpired reset code
  When the user submits it with a new password
  Then the password is updated and the code cannot be reused
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.auth`, `com.barbersaas.notification.EmailService`
**Endpoint(s) implemented:** `POST /api/auth/forgot-password`, `POST /api/auth/reset-password`
**Required permissions:** None (public endpoints)

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done |
| Dependencies | HU-AUTH-001 |
| Affected service(s) | `auth`, `notification` |

---

### HU-AUTH-003 — Self-register a barbershop with a 60-day trial {#HU-AUTH-003}

**Epic:** EP-001

> **As** a prospective barbershop owner
> **I want** to register my barbershop myself, without waiting for a sales process
> **so that** I can start a free trial immediately and evaluate the platform

**Acceptance Criteria:**

```gherkin
Scenario 1: Successful self-registration
  Given valid barbershop and owner details
  When the owner completes self-registration
  Then a Barbershop is created with status TRIAL
  And trialEndsAt is set to createdAt + 60 days, and is never recalculated afterward
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.auth`, `com.barbersaas.barbershop`
**Endpoint(s) implemented:** `POST /api/auth/register-barbershop`
**Required permissions:** None (public endpoint)

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done |
| Dependencies | None |
| Affected service(s) | `auth`, `barbershop` |

---

### HU-SHOP-001 — Configure service catalog and staff schedules {#HU-SHOP-001}

**Epic:** EP-002

> **As** an `ADMIN_BARBERSHOP`
> **I want** to configure my barbershop's service catalog and my barbers' weekly schedules
> **so that** clients see accurate availability and pricing when booking

**Acceptance Criteria:**

```gherkin
Scenario 1: Create a service
  Given I am authenticated as ADMIN_BARBERSHOP
  When I create a service with a name, price, and duration
  Then it is scoped to my barbershopId and visible only within my tenant

Scenario 2: Overlapping schedule rejected
  Given a barber already has a schedule slot for a given day
  When I attempt to save an overlapping schedule slot for that same barber/day
  Then the system rejects it

Scenario 3: Schedule exception overrides, never merges
  Given a barber has a regular weekly schedule for a given day of week
  When I register a schedule exception for a specific date (day off or modified hours)
  Then that exception overrides the weekly schedule for that date, it does not merge with it
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.barberservice`, `com.barbersaas.schedule`
**Endpoint(s) implemented:** `GET/POST /api/admin/services`, `PUT/PATCH /api/admin/services/{id}`
**Required permissions:** `ADMIN_BARBERSHOP`

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done |
| Dependencies | HU-AUTH-003 |
| Affected service(s) | `barberservice`, `schedule` |

---

### HU-APPT-001 — Book an appointment without double-booking {#HU-APPT-001}

**Epic:** EP-003 (Core Domain)

> **As** a `CLIENT`
> **I want** to book an appointment with a specific barber, service, date, and time
> **so that** I have a guaranteed slot without having to call or message the barbershop

**Acceptance Criteria:**

```gherkin
Scenario 1: Successful booking
  Given the barber has no other appointment overlapping the requested time slot
  When I book the appointment
  Then it is created with status PENDING
  And priceAtBooking is snapshotted from the service's current price

Scenario 2: Double-booking rejected
  Given the barber already has an appointment overlapping the requested slot
  When a second client attempts to book the same slot
  Then the booking is rejected (pessimistic database lock on the barber's schedule)

Scenario 3: Past-date booking rejected
  Given a requested date/time is in the past
  When I attempt to book
  Then the system rejects it

Scenario 4: Active reward coupon applied automatically
  Given I have an ACTIVE RewardCoupon for this barbershop
  When I book an appointment at that barbershop
  Then the coupon is applied and marked USED in the same transaction
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.appointment`
**Endpoint(s) implemented:** `POST /api/client/appointments`
**Required permissions:** `CLIENT`

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done |
| Dependencies | HU-SHOP-001 |
| Affected service(s) | `appointment`, `loyalty` (coupon consumption) |

---

### HU-APPT-002 — Cancel an appointment within policy, and auto-mark no-shows {#HU-APPT-002}

**Epic:** EP-003

> **As** a `CLIENT`
> **I want** to cancel my appointment with enough notice
> **so that** I free up the slot for someone else and am not penalized

**Acceptance Criteria:**

```gherkin
Scenario 1: Cancellation within the policy window
  Given the current time is before (appointmentStartTime - barbershop.cancellationPolicyHours)
  When I cancel a PENDING or CONFIRMED appointment
  Then it transitions to CANCELLED
  And I receive a notification

Scenario 2: Cancellation inside the policy window rejected for clients
  Given the current time is inside the cancellation policy window
  When a CLIENT attempts to cancel
  Then the system rejects it
  (Admins and barbers are exempt from this restriction)

Scenario 3: Automatic no-show marking
  Given a CONFIRMED appointment's date has passed without being completed
  When the daily 01:00 scheduled job runs
  Then the appointment is automatically marked NO_SHOW, with no notification sent (documented gap)
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.appointment`
**Endpoint(s) implemented:** `PATCH /api/client/appointments/{id}/cancel`, `PATCH /api/admin/appointments/{id}/cancel`, scheduled job `AppointmentReminderJob#markPastAppointmentsAsNoShow`
**Required permissions:** `CLIENT`, `ADMIN_BARBERSHOP`

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done |
| Dependencies | HU-APPT-001 |
| Affected service(s) | `appointment`, `notification` |

---

### HU-LOY-001 — Earn and redeem loyalty rewards {#HU-LOY-001}

**Epic:** EP-004

> **As** a `CLIENT`
> **I want** to accumulate loyalty stickers and redeem them for a reward
> **so that** I'm rewarded for being a repeat customer at this barbershop

**Acceptance Criteria:**

```gherkin
Scenario 1: Sticker granted
  Given a barber or admin grants me a sticker (optionally linked to a completed visit)
  When the grant is processed
  Then my LoyaltyCard.stickersCount increases by 1
  And a LoyaltyTransaction of type STICKER_EARNED is recorded

Scenario 2: Redemption rejected when insufficient stickers
  Given my stickersCount is below the barbershop's configured stickersRequired
  When staff attempts to redeem a reward for me
  Then the system rejects it

Scenario 3: Successful redemption
  Given my stickersCount meets or exceeds stickersRequired
  When staff redeems my reward
  Then stickersCount decreases by stickersRequired, totalRewardsRedeemed increases by 1
  And a new ACTIVE RewardCoupon is created for me
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.loyalty`
**Endpoint(s) implemented:** `POST /api/admin/loyalty/grant`, `POST /api/admin/loyalty/redeem/{clientId}`
**Required permissions:** `ADMIN_BARBERSHOP`, `BARBER`

> **Known gap:** neither `grantSticker()` nor `redeemReward()` sends a client notification
> today, unlike appointment state changes. See `02-domain/domain-events.md`.

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done |
| Dependencies | HU-SHOP-001 |
| Affected service(s) | `loyalty` |

---

### HU-FIN-001 — Track barbershop income and expenses {#HU-FIN-001}

**Epic:** EP-005

> **As** an `ADMIN_BARBERSHOP`
> **I want** to record income and expense entries
> **so that** I can see my barbershop's financial performance in one place instead of manual bookkeeping

**Acceptance Criteria:**

```gherkin
Scenario 1: Record a valid entry
  Given a FinanceRecord with type INCOME or EXPENSE and a positive amount
  When I save it
  Then it appears in my barbershop's financial summary

Scenario 2: Invalid amount rejected
  Given an amount of zero or negative
  When I attempt to save a FinanceRecord
  Then the system rejects it
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.finance`
**Endpoint(s) implemented:** `POST /api/admin/finance/records`, `GET /api/admin/finance/records`, `GET /api/admin/finance/summary`
**Required permissions:** `ADMIN_BARBERSHOP`

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done |
| Dependencies | HU-AUTH-003 |
| Affected service(s) | `finance` |

---

### HU-INV-001 — Track product stock and low-stock alerts {#HU-INV-001}

**Epic:** EP-006

> **As** an `ADMIN_BARBERSHOP`
> **I want** to track my product inventory and get alerted when stock is low
> **so that** I don't run out of supplies mid-service

**Acceptance Criteria:**

```gherkin
Scenario 1: Stock alert triggered
  Given a product's stock is updated
  When currentStock <= minStockAlert
  Then the product is flagged as a stock alert

Scenario 2: Negative stock rejected
  Given a stock movement that would make stock negative
  When I attempt to save it
  Then the system rejects it
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.inventory`
**Endpoint(s) implemented:** `GET/POST /api/admin/inventory/products`, `POST /api/admin/inventory/products/{id}/movement`
**Required permissions:** `ADMIN_BARBERSHOP`

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Should Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done |
| Dependencies | HU-AUTH-003 |
| Affected service(s) | `inventory` |

---

### HU-NOTIF-001 — Receive appointment notifications {#HU-NOTIF-001}

**Epic:** EP-007

> **As** a `CLIENT`
> **I want** to receive a notification when my appointment is booked, confirmed, cancelled, or about to happen
> **so that** I don't have to check the app constantly to know my appointment status

**Acceptance Criteria:**

```gherkin
Scenario 1: Notification on state change
  Given my appointment is created, confirmed, or cancelled
  When the state change happens
  Then an in-app notification is persisted for me, regardless of whether FCM push delivery succeeds

Scenario 2: Day-before reminder
  Given I have a CONFIRMED appointment for tomorrow
  When the daily 18:00 reminder job runs
  Then I receive a REMINDER notification

Scenario 3 (known gap, not yet implemented): Completion notification
  Given my appointment is marked COMPLETED
  When the status changes
  Then — as of this review, no notification is sent (see 02-domain/domain-events.md)
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.notification`, triggered from `com.barbersaas.appointment`
**Endpoint(s) implemented:** N/A (triggered server-side, not a client-called endpoint); `GET /api/notifications` to read them
**Required permissions:** `CLIENT` (to read own notifications)

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done (booking/confirm/cancel/reminder) — 🔴 gap on completion, see AC3 |
| Dependencies | HU-APPT-001 |
| Affected service(s) | `notification`, `appointment` |

---

### HU-SADMIN-001 — Manage barbershop accounts and subscription plans {#HU-SADMIN-001}

**Epic:** EP-008

> **As** a `SUPER_ADMIN`
> **I want** to view all registered barbershops and manage their subscription/trial status
> **so that** I can operate the BarberSaaS business (billing, support, account lifecycle) across every tenant

**Acceptance Criteria:**

```gherkin
Scenario 1: Manual trial-to-paid conversion
  Given a barbershop's 60-day trial and a confirmed manual payment
  When a SUPER_ADMIN transitions it from TRIAL to ACTIVE
  Then that barbershop gains full access

Scenario 2: Only Super Admin can change billing status
  Given a non-SUPER_ADMIN user
  When they attempt to change a barbershop's status
  Then the system rejects it

Scenario 3 (known gap, not yet implemented): Automatic trial expiration
  Given a barbershop's 60-day trial reaches its expiration date without conversion
  When the expiration date passes
  Then — as of this review — there is no automatic transition to SUSPENDED (see 01-context/scope.md, feature F-14, "in progress")
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.barbershop.SuperAdminBarbershopController`, `com.barbersaas.plan`, `com.barbersaas.dashboard`
**Endpoint(s) implemented:** `GET /api/super-admin/barbershops`, `PATCH /api/super-admin/barbershops/{id}/status`, `/api/super-admin/plans`, `/api/super-admin/dashboard`
**Required permissions:** `SUPER_ADMIN`

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done (manual transitions) — 🔴 gap on automatic expiration, see AC3 |
| Dependencies | HU-AUTH-003 |
| Affected service(s) | `barbershop`, `plan`, `dashboard` |

---

### HU-TENANT-001 — Tenant data isolation across every operation {#HU-TENANT-001}

**Epic:** EP-009 (Cross-cutting foundation)

> **As** the BarberSaaS platform
> **I want** every request scoped to the authenticated user's `barbershopId`
> **so that** one barbershop can never read or modify another barbershop's data

**Acceptance Criteria:**

```gherkin
Scenario 1: Tenant resolved from JWT
  Given a valid JWT
  When any tenant-scoped endpoint is called
  Then JwtAuthenticationFilter resolves barbershopId into TenantContext
  And the service layer filters every query by that barbershopId

Scenario 2: Cross-tenant access rejected
  Given a user attempts to access or modify a resource (e.g., an appointment or loyalty card) belonging to a different barbershopId than their own
  When the request is processed
  Then the system rejects it
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.security` (cross-cutting, applies to every module)
**Endpoint(s) implemented:** N/A — enforced as a filter + service-layer pattern across all endpoints, not a single endpoint
**Required permissions:** N/A (applies regardless of role)

> This HU is the foundation every other HU in this backlog depends on. See the
> multi-tenancy rule in each repo's `CLAUDE.md` and `00-governance/security-policy.md`.

| Field | Value |
|-------|-------|
| Story Points | Not yet estimated |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | ✅ Done |
| Dependencies | None (foundational) |
| Affected service(s) | `security`, all tenant-scoped modules |

---

## Rules for writing HUs

### 1. The role matters
Do not write "As a user" — use the specific role (`CLIENT`, `BARBER`, `ADMIN_BARBERSHOP`, `SUPER_ADMIN`).

### 2. The benefit justifies the work
The "so that" must describe a business benefit, not redescribe the action.

### 3. ACs are verifiable
Each AC must be verifiable manually or automatable as a test — every AC above traces to a
documented invariant (`02-domain/entities-and-rules.md`) or a verified code path.

### 4. One HU = one unit of value
The HUs above are epic-sized on purpose (see "Backlog status" note). Split each into
smaller HUs during refinement before scheduling into a sprint.

---

## Correlations

- Functional requirements these HUs implement → `04-requirements/functional.md`
- Traceability to tests and services → `04-requirements/traceability-matrix.md`
- Epic-level backlog → `03-product/product-backlog.md`
- Full HU template with DoD checklist → `04-requirements/_template-hu.md`
