# User Stories — Backlog

> Formalized from the epic-level backlog in `03-product/product-backlog.md`. Each HU's
> acceptance criteria are grounded in the business invariants already documented and
> verified against the real backend code in `02-domain/entities-and-rules.md` and
> `02-domain/domain-map.md` — no new business rule is invented here.
>
> **2026-09-27 (SPEC-008):** the 10 epic-sized HUs originally formalized here were split
> into 18 sprint-ready HUs, per `00-governance/definition-of-ready.md` and rule #4 below
> ("One HU = one unit of value"). Each split HU carries a `Split from` field pointing to its
> original epic-sized parent. Story Points are now filled in for every `✅ Done` HU using a
> **retroactive estimation methodology** (see `00-governance/agile-conventions.md` § "Team
> velocity") — these are not planning-poker consensus, they size already-built work so
> historical throughput is visible. HUs that are not yet done (gaps) are marked
> "pending real estimation" where a retroactive size cannot be honestly derived.

---

## Backlog status

| Cut | Epics covered | Total HUs | Status |
|-----|--------|-----------|--------|
| 2026-09-17 (first pass) | EP-001 … EP-009 (all) | 10 (epic-sized) | 9 Done, 1 partially Done |
| 2026-09-27 (SPEC-008 split) | EP-001 … EP-009 (all) | 18 (sprint-ready) | 17 Done, 1 not implemented (HU-SADMIN-001-B) |

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

### HU-AUTH-001-A — Register a role-based account {#HU-AUTH-001-A}

**Epic:** EP-001 · **Split from:** HU-AUTH-001

> **As** a new user (client, barber, or barbershop admin)
> **I want** to register an account
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
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.auth`
**Endpoint(s) implemented:** `POST /api/auth/register`
**Required permissions:** None (public endpoint)

**Definition of Done:** See `00-governance/definition-of-done.md`.

| Field | Value |
|-------|-------|
| Split from | HU-AUTH-001 |
| Story Points | 3 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred, see `15-project-control/sprint-status.md`) |
| Status | ✅ Done |
| Dependencies | None |
| Affected service(s) | `auth` |

---

### HU-AUTH-001-B — Log in with a role-based account {#HU-AUTH-001-B}

**Epic:** EP-001 · **Split from:** HU-AUTH-001

> **As** a registered user (client, barber, or barbershop admin)
> **I want** to log in
> **so that** I can access the features available to my role

**Acceptance Criteria:**

```gherkin
Scenario 1: Successful login
  Given valid credentials for an existing account
  When the user logs in
  Then the system returns a JWT access token (24h expiration) and a refresh token (7-day expiration)
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.auth`
**Endpoint(s) implemented:** `POST /api/auth/login`
**Required permissions:** None (public endpoint)

| Field | Value |
|-------|-------|
| Split from | HU-AUTH-001 |
| Story Points | 2 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-AUTH-001-A |
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
| Story Points | 3 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-AUTH-001-A |
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
| Story Points | 3 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | None |
| Affected service(s) | `auth`, `barbershop` |

---

### HU-SHOP-001-A — Configure the barbershop's service catalog {#HU-SHOP-001-A}

**Epic:** EP-002 · **Split from:** HU-SHOP-001

> **As** an `ADMIN_BARBERSHOP`
> **I want** to configure my barbershop's service catalog
> **so that** clients see accurate pricing and duration when booking

**Acceptance Criteria:**

```gherkin
Scenario 1: Create a service
  Given I am authenticated as ADMIN_BARBERSHOP
  When I create a service with a name, price, and duration
  Then it is scoped to my barbershopId and visible only within my tenant
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.barberservice`
**Endpoint(s) implemented:** `GET/POST /api/admin/services`, `PUT/PATCH /api/admin/services/{id}`
**Required permissions:** `ADMIN_BARBERSHOP`

| Field | Value |
|-------|-------|
| Split from | HU-SHOP-001 |
| Story Points | 2 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-AUTH-003 |
| Affected service(s) | `barberservice` |

---

### HU-SHOP-001-B — Manage staff weekly schedules and exceptions {#HU-SHOP-001-B}

**Epic:** EP-002 · **Split from:** HU-SHOP-001

> **As** an `ADMIN_BARBERSHOP`
> **I want** to configure my barbers' weekly schedules, with exceptions for specific dates
> **so that** clients see accurate availability when booking

**Acceptance Criteria:**

```gherkin
Scenario 1: Overlapping schedule rejected
  Given a barber already has a schedule slot for a given day
  When I attempt to save an overlapping schedule slot for that same barber/day
  Then the system rejects it

Scenario 2: Schedule exception overrides, never merges
  Given a barber has a regular weekly schedule for a given day of week
  When I register a schedule exception for a specific date (day off or modified hours)
  Then that exception overrides the weekly schedule for that date, it does not merge with it
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.schedule`
**Endpoint(s) implemented:** (schedule management endpoints under `/api/admin/schedule`)
**Required permissions:** `ADMIN_BARBERSHOP`

| Field | Value |
|-------|-------|
| Split from | HU-SHOP-001 |
| Story Points | 3 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-SHOP-001-A |
| Affected service(s) | `schedule` |

---

### HU-APPT-001-A — Book an appointment without double-booking {#HU-APPT-001-A}

**Epic:** EP-003 (Core Domain) · **Split from:** HU-APPT-001

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
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.appointment`
**Endpoint(s) implemented:** `POST /api/client/appointments`
**Required permissions:** `CLIENT`

> **Known gap (traceability-matrix.md FR-008):** highest-risk untested path — the pessimistic
> lock has no automated concurrency test.

| Field | Value |
|-------|-------|
| Split from | HU-APPT-001 |
| Story Points | 5 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-SHOP-001-B |
| Affected service(s) | `appointment` |

---

### HU-APPT-001-B — Apply an active loyalty coupon automatically at booking {#HU-APPT-001-B}

**Epic:** EP-003 · **Split from:** HU-APPT-001

> **As** a `CLIENT`
> **I want** my active reward coupon to be applied automatically when I book
> **so that** I don't have to remember to redeem it manually

**Acceptance Criteria:**

```gherkin
Scenario 1: Active reward coupon applied automatically
  Given I have an ACTIVE RewardCoupon for this barbershop
  When I book an appointment at that barbershop
  Then the coupon is applied and marked USED in the same transaction
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.appointment`, `com.barbersaas.loyalty`
**Endpoint(s) implemented:** `POST /api/client/appointments`
**Required permissions:** `CLIENT`

| Field | Value |
|-------|-------|
| Split from | HU-APPT-001 |
| Story Points | 2 (retroactive) |
| Priority | Should Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-APPT-001-A, HU-LOY-001 |
| Affected service(s) | `appointment`, `loyalty` |

---

### HU-APPT-002-A — Cancel an appointment within policy {#HU-APPT-002-A}

**Epic:** EP-003 · **Split from:** HU-APPT-002

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
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.appointment`
**Endpoint(s) implemented:** `PATCH /api/client/appointments/{id}/cancel`, `PATCH /api/admin/appointments/{id}/cancel`
**Required permissions:** `CLIENT`, `ADMIN_BARBERSHOP`

| Field | Value |
|-------|-------|
| Split from | HU-APPT-002 |
| Story Points | 3 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-APPT-001-A |
| Affected service(s) | `appointment`, `notification` |

---

### HU-APPT-002-B — Automatically mark past appointments as no-show {#HU-APPT-002-B}

**Epic:** EP-003 · **Split from:** HU-APPT-002

> **As** the BarberSaaS platform
> **I want** to automatically mark CONFIRMED appointments as NO_SHOW once their date passes uncompleted
> **so that** barbers and admins have an accurate record without manual bookkeeping

**Acceptance Criteria:**

```gherkin
Scenario 1: Automatic no-show marking
  Given a CONFIRMED appointment's date has passed without being completed
  When the daily 01:00 scheduled job runs
  Then the appointment is automatically marked NO_SHOW, with no notification sent (documented gap)
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.appointment`
**Endpoint(s) implemented:** scheduled job `AppointmentReminderJob#markPastAppointmentsAsNoShow`
**Required permissions:** N/A (system job)

> **Known gap:** no notification is sent to the client when this transition happens.

| Field | Value |
|-------|-------|
| Split from | HU-APPT-002 |
| Story Points | 2 (retroactive) |
| Priority | Should Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-APPT-001-A |
| Affected service(s) | `appointment` |

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
| Story Points | 3 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 2 (inferred — see `feat(loyalty)` commit `122b362`, 2026-09-13) |
| Status | ✅ Done |
| Dependencies | HU-SHOP-001-A |
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
| Story Points | 2 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
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
| Story Points | 2 (retroactive) |
| Priority | Should Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-AUTH-003 |
| Affected service(s) | `inventory` |

---

### HU-NOTIF-001-A — Notify client on appointment state changes {#HU-NOTIF-001-A}

**Epic:** EP-007 · **Split from:** HU-NOTIF-001

> **As** a `CLIENT`
> **I want** to receive a notification when my appointment is booked, confirmed, or cancelled
> **so that** I don't have to check the app constantly to know my appointment status

**Acceptance Criteria:**

```gherkin
Scenario 1: Notification on state change
  Given my appointment is created, confirmed, or cancelled
  When the state change happens
  Then an in-app notification is persisted for me, regardless of whether FCM push delivery succeeds

Scenario 2 (known gap, not yet implemented): Completion notification
  Given my appointment is marked COMPLETED
  When the status changes
  Then — as of this review, no notification is sent (see 02-domain/domain-events.md, traceability-matrix.md FR-022)
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.notification`, triggered from `com.barbersaas.appointment`
**Endpoint(s) implemented:** N/A (triggered server-side); `GET /api/notifications` to read them
**Required permissions:** `CLIENT` (to read own notifications)

| Field | Value |
|-------|-------|
| Split from | HU-NOTIF-001 |
| Story Points | 3 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done (booking/confirm/cancel) — 🔴 gap on completion, see Scenario 2 |
| Dependencies | HU-APPT-001-A |
| Affected service(s) | `notification`, `appointment` |

---

### HU-NOTIF-001-B — Send a day-before appointment reminder {#HU-NOTIF-001-B}

**Epic:** EP-007 · **Split from:** HU-NOTIF-001

> **As** a `CLIENT`
> **I want** to receive a reminder the day before my appointment
> **so that** I don't forget it

**Acceptance Criteria:**

```gherkin
Scenario 1: Day-before reminder
  Given I have a CONFIRMED appointment for tomorrow
  When the daily 18:00 reminder job runs
  Then I receive a REMINDER notification
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.notification`, triggered from `com.barbersaas.appointment`
**Endpoint(s) implemented:** scheduled job (daily 18:00)
**Required permissions:** N/A (system job)

| Field | Value |
|-------|-------|
| Split from | HU-NOTIF-001 |
| Story Points | 2 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-APPT-001-A |
| Affected service(s) | `notification`, `appointment` |

---

### HU-SADMIN-001-A — View barbershops and manage subscription/trial status {#HU-SADMIN-001-A}

**Epic:** EP-008 · **Split from:** HU-SADMIN-001

> **As** a `SUPER_ADMIN`
> **I want** to view all registered barbershops and manually transition their subscription/trial status
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
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.barbershop.SuperAdminBarbershopController`, `com.barbersaas.plan`, `com.barbersaas.dashboard`
**Endpoint(s) implemented:** `GET /api/super-admin/barbershops`, `PATCH /api/super-admin/barbershops/{id}/status`, `/api/super-admin/plans`, `/api/super-admin/dashboard`
**Required permissions:** `SUPER_ADMIN`

| Field | Value |
|-------|-------|
| Split from | HU-SADMIN-001 |
| Story Points | 3 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
| Status | ✅ Done |
| Dependencies | HU-AUTH-003 |
| Affected service(s) | `barbershop`, `plan`, `dashboard` |

---

### HU-SADMIN-001-B — Automatically expire a barbershop's trial {#HU-SADMIN-001-B}

**Epic:** EP-008 · **Split from:** HU-SADMIN-001

> **As** the BarberSaaS platform
> **I want** to automatically transition a barbershop from TRIAL to SUSPENDED when its 60-day trial expires without conversion
> **so that** unpaid tenants lose access without requiring a Super Admin to check manually every day

**Acceptance Criteria:**

```gherkin
Scenario 1 (not yet implemented): Automatic trial expiration
  Given a barbershop's 60-day trial reaches its expiration date without conversion
  When the expiration date passes
  Then — as of this review — there is no automatic transition to SUSPENDED
  (see 01-context/scope.md, feature F-14, "in progress")
```

**Technical notes:**

**Responsible service(s):** `com.barbersaas.barbershop` (planned: a scheduled job analogous to `AppointmentReminderJob`)
**Endpoint(s) implemented:** None yet
**Required permissions:** N/A (system job, planned)

> This is real, currently-planned MVP work (F-14, "in progress" per `01-context/scope.md`
> and `04-requirements/traceability-matrix.md` FR-026) — **it is not MVP2 scope.** Do not
> confuse it with `03-product/mvp2-backlog.md`.

| Field | Value |
|-------|-------|
| Split from | HU-SADMIN-001 |
| Story Points | 3 (proposed — pending real estimation, not retroactive since it isn't built yet) |
| Priority | Must Have |
| Target sprint | Not yet scheduled |
| Status | 🔴 Not implemented |
| Dependencies | HU-SADMIN-001-A |
| Affected service(s) | `barbershop` |

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

> This HU is the foundation every other HU in this backlog depends on. Deliberately kept
> unsplit despite touching every module — splitting a single cross-cutting security
> invariant into per-module pieces would fragment something that must be verified as one
> whole. See the multi-tenancy rule in each repo's `CLAUDE.md` and
> `00-governance/security-policy.md`.

| Field | Value |
|-------|-------|
| Story Points | 5 (retroactive) |
| Priority | Must Have |
| Target sprint | Sprint 1 (inferred) |
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
As of 2026-09-27 (SPEC-008), the backlog above is already split to sprint-ready
granularity. When adding a new HU, keep it small enough to fit in one sprint from the
start — do not create another epic-sized HU that needs splitting later.

---

## Correlations

- Functional requirements these HUs implement → `04-requirements/functional.md`
- Traceability to tests and services → `04-requirements/traceability-matrix.md`
- Epic-level backlog → `03-product/product-backlog.md`
- Full HU template with DoD checklist → `04-requirements/_template-hu.md`
- Sprint numbering and dates → `15-project-control/sprint-status.md`
- Story map → `03-product/story-map.md`
- MVP2 backlog (Phase 3+ items, out of current scope) → `03-product/mvp2-backlog.md`
