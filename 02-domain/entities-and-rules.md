# Entities, Value Objects, and Business Rules

> **Stack note:** The concepts of Entity, Value Object, and Aggregate are language-independent.
> BarberSaaS's backend is implemented in Java 21 / Spring Boot; code examples below are written
> in pseudo-TypeScript to illustrate the domain model, per the project's DDD convention.

---

## Tactical DDD concepts

### Entity
An **Entity** is an object defined by its identity, not its attributes.
Two entities are equal if they have the same ID, even if all their other attributes differ.

### Value Object (VO)
A **Value Object** is an object defined by its attributes; it has no identity of its own.
It is immutable — if an attribute changes, it is a new VO.

### Aggregate
An **Aggregate** is a cluster of entities and VOs treated as a unit, with an **Aggregate Root**
as its only entry point.

**Golden rule of the Aggregate:** Transactions do not cross aggregate boundaries.
In BarberSaaS this shows up concretely: a loyalty redemption (Loyalty context) creates a
`RewardCoupon`, but the `Appointment` aggregate (Appointment context) only ever references that
coupon by ID — it never reaches into the `LoyaltyCard` aggregate directly.

### Business Rules
**Business Rules** (invariants) are the constraints the domain must always satisfy. They live in
the Aggregate Root and are validated on every operation.

---

## System entities

### Entity: Appointment

**Context:** Appointment

**Description:** A scheduled service between a client and a barber at a specific barbershop, for a specific service, date, and time slot.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| id | UUID | Unique identifier | Yes | Auto-generated on creation |
| barbershopId | UUID | Owning tenant | Yes | Immutable after creation; enforced by `TenantContext` |
| clientId | UUID | Client the appointment belongs to | Yes (nullable for walk-ins) | Nullable when created as a walk-in by an admin |
| barberId | UUID | Barber assigned to the appointment | Yes | Must belong to the same `barbershopId` |
| serviceId | UUID | Service being performed | Yes | Must belong to the same `barbershopId`'s service catalog |
| date | Date | Appointment date | Yes | Must not be in the past at creation time |
| startTime / endTime | Time | Time slot (VO: `TimeSlot`) | Yes | `endTime` derived from service duration; must not overlap another appointment for the same barber |
| status | Enum | PENDING / CONFIRMED / IN_PROGRESS / COMPLETED / CANCELLED / NO_SHOW | Yes | Only valid transitions allowed (see state machine) |
| priceAtBooking | Money | Price snapshot at the moment of booking | Yes | Immutable once set — protects against later service price changes |
| createdAt | DateTime | Creation date | Yes | Immutable, set on creation |
| updatedAt | DateTime | Last modification | Yes | Updated automatically |

**Lifecycle / States:**

```
PENDING ──(confirm)──▶ CONFIRMED ──(start service)──▶ IN_PROGRESS ──(finish)──▶ COMPLETED
   │                        │
   │                        ├──(client/admin cancels, within policy window)──▶ CANCELLED
   │                        └──(client doesn't show up)──▶ NO_SHOW
   └──(client/admin cancels)──▶ CANCELLED
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| PENDING | Just booked, awaiting confirmation | → CONFIRMED, → CANCELLED |
| CONFIRMED | Confirmed and scheduled | → IN_PROGRESS, → CANCELLED, → NO_SHOW |
| IN_PROGRESS | Barber has started the service | → COMPLETED |
| COMPLETED | Service finished | Terminal state |
| CANCELLED | Cancelled by client or admin | Terminal state |
| NO_SHOW | Client did not show up | Terminal state |

**Invariants (Business rules that MUST ALWAYS hold):**

```
INV-APPT-001: No double-booking
  - Rule: A barber cannot have two appointments with overlapping time slots on the same date.
  - Violation: Booking attempt is rejected at the database level via pessimistic locking.
  - Implementation: Pessimistic lock acquired on the barber's schedule row during appointment creation.

INV-APPT-002: Price snapshot immutability
  - Rule: priceAtBooking is fixed at the moment of booking and never recalculated, even if the
    barbershop later changes the service's price.
  - Violation: Attempting to mutate priceAtBooking after creation throws a DomainException.
  - Implementation: priceAtBooking has no public setter; only set in the factory method.

INV-APPT-003: Cancellation window
  - Rule: A client can only cancel an appointment if the current time is before
    (appointmentStartTime - barbershop.cancellationPolicyHours).
  - Violation: DomainException thrown if a client attempts to cancel inside the policy window
    (admins/barbers are exempt from this restriction).
  - Implementation: Validated in the cancel() method against the barbershop's configured policy.

INV-APPT-004: Valid state transitions only
  - Rule: Status can only move along the edges defined in the state machine above.
  - Violation: DomainException thrown on any invalid transition (e.g., COMPLETED → PENDING).
  - Implementation: Each transition method (confirm(), start(), complete(), cancel(), markNoShow())
    validates the current status before mutating it.

INV-APPT-005: Tenant consistency
  - Rule: barberId and serviceId referenced by an appointment must belong to the same
    barbershopId as the appointment itself.
  - Violation: DomainException thrown if a cross-tenant reference is attempted.
  - Implementation: Validated in the factory method using TenantContext.
```

**Code example (TypeScript/Java):**

```typescript
// TypeScript — Entity with invariants
class Appointment {
  private constructor(
    private readonly id: AppointmentId,
    private readonly barbershopId: BarbershopId,
    private readonly clientId: ClientId | null,
    private readonly barberId: BarberId,
    private readonly serviceId: ServiceId,
    private readonly slot: TimeSlot,
    private status: AppointmentStatus,
    private readonly priceAtBooking: Money,
  ) {}

  static book(barberId: BarberId, serviceId: Service, slot: TimeSlot, clientId: ClientId | null): Appointment {
    if (slot.isInThePast()) {
      throw new DomainException('INV-APPT-000: Cannot book an appointment in the past');
    }
    return new Appointment(
      AppointmentId.new(), serviceId.barbershopId, clientId, barberId,
      serviceId.id, slot, AppointmentStatus.PENDING, serviceId.currentPrice(),
    );
  }

  cancel(requestedBy: Role, cancellationPolicyHours: number): void {
    if (this.status !== AppointmentStatus.PENDING && this.status !== AppointmentStatus.CONFIRMED) {
      throw new DomainException('INV-APPT-004: Only PENDING or CONFIRMED appointments can be cancelled');
    }
    if (requestedBy === Role.CLIENT && this.slot.hoursUntilStart() < cancellationPolicyHours) {
      throw new DomainException('INV-APPT-003: Cancellation window has passed');
    }
    this.status = AppointmentStatus.CANCELLED;
    this.addEvent(new AppointmentCancelledEvent(this.id));
  }
}
```

---

### Entity: Barbershop

**Context:** Barbershop (tenant)

**Description:** A barbershop business registered on the platform; the tenant root for all barbershop-scoped data.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| id | UUID | Unique identifier, used as `barbershop_id` tenant key | Yes | Auto-generated on creation |
| name | String | Barbershop's display name | Yes | Non-empty |
| city | String | City where the barbershop operates | Yes | Used in client search |
| status | Enum | TRIAL / ACTIVE / SUSPENDED / CANCELLED | Yes | Managed by Super Admin (except TRIAL, set at registration) |
| trialEndsAt | DateTime | End of the 60-day free trial | Yes | Set to `createdAt + 60 days` at registration; immutable |
| subscriptionPlanId | UUID | Currently assigned plan | Yes | Must reference an `isActive` `SubscriptionPlan` |
| cancellationPolicyHours | Integer | Hours before an appointment that a client can still cancel | No | Configured by the admin; defaults if unset |
| createdAt | DateTime | Registration date | Yes | Immutable |
| updatedAt | DateTime | Last modification | Yes | Updated automatically |

**Lifecycle / States:**

```
TRIAL ──(payment confirmed by Super Admin)──▶ ACTIVE ──(non-payment / manual action)──▶ SUSPENDED
  │                                                │                                       │
  └──(trial expires without conversion)──▶ SUSPENDED                                       │
                                                    └──(reactivation)──▶ ACTIVE              │
                                                                                             ▼
                                                                                        CANCELLED
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| TRIAL | Within the 60-day free trial period | → ACTIVE, → SUSPENDED |
| ACTIVE | Paying, fully operational | → SUSPENDED, → CANCELLED |
| SUSPENDED | Trial expired without conversion, or payment lapsed | → ACTIVE, → CANCELLED |
| CANCELLED | Account permanently closed | Terminal state |

**Invariants (Business rules that MUST ALWAYS hold):**

```
INV-SHOP-001: Trial period is fixed
  - Rule: trialEndsAt = createdAt + 60 days, and is never recalculated after registration.
  - Violation: Any attempt to change trialEndsAt after creation is rejected.
  - Implementation: trialEndsAt has no setter; computed once in the registration factory method.

INV-SHOP-002: Only Super Admin transitions billing status
  - Rule: TRIAL → ACTIVE and any → SUSPENDED/CANCELLED transitions can only be triggered by a
    user with the SUPER_ADMIN role (manual billing in the MVP).
  - Violation: DomainException thrown if a non-SUPER_ADMIN caller attempts the transition.
  - Implementation: Role check performed at the application-service layer before invoking the
    entity's transition method.

INV-SHOP-003: Plan must be active
  - Rule: subscriptionPlanId must always reference a SubscriptionPlan with isActive = true.
  - Violation: DomainException thrown if assigning an inactive plan.
  - Implementation: Validated in assignPlan().
```

---

### Entity: LoyaltyCard

**Context:** Loyalty

**Description:** Tracks a client's accumulated loyalty stickers at a specific barbershop.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| id | UUID | Unique identifier | Yes | Auto-generated on creation |
| clientId | UUID | Owning client | Yes | Immutable |
| barbershopId | UUID | Tenant scope | Yes | Immutable; a client has one card per barbershop |
| stickersCount | Integer | Current sticker count | Yes | ≥ 0; reset to 0 (or decremented) on redemption |
| totalRewardsRedeemed | Integer | Historical count of redemptions | Yes | Only increments |

**Invariants (Business rules that MUST ALWAYS hold):**

```
INV-LOYAL-001: Cannot redeem without enough stickers
  - Rule: A reward can only be redeemed when stickersCount >= barbershop's configured
    stickersRequired.
  - Violation: DomainException thrown if redeem() is called with insufficient stickers.
  - Implementation: Validated in redeem() against LoyaltyRewardsConfig.

INV-LOYAL-002: Stickers only granted by staff
  - Rule: Stickers can only be granted by a user with role ADMIN_BARBERSHOP or BARBER belonging
    to the same barbershop as the card.
  - Violation: DomainException thrown on a cross-tenant or cross-role grant attempt.
  - Implementation: Validated at the application-service layer via TenantContext + role check.

INV-LOYAL-003: Redemption is atomic with coupon issuance
  - Rule: Every successful redemption must produce exactly one ACTIVE RewardCoupon (via a domain
    event), and stickersCount must be decremented in the same operation.
  - Violation: A redemption that doesn't emit a coupon-issued event is invalid.
  - Implementation: redeem() both mutates stickersCount and raises LoyaltyRedeemedEvent, consumed
    by the Loyalty context to create the RewardCoupon within the same aggregate transaction.
```

**Code example (TypeScript/Java):**

```typescript
// TypeScript — Entity with invariants
class LoyaltyCard {
  redeem(stickersRequired: number): void {
    if (this.stickersCount < stickersRequired) {
      throw new DomainException('INV-LOYAL-001: Not enough stickers to redeem');
    }
    this.stickersCount -= stickersRequired;
    this.totalRewardsRedeemed += 1;
    this.addEvent(new LoyaltyRedeemedEvent(this.id, this.clientId, this.barbershopId));
  }
}
```

---

### Other domain entities (abbreviated)

These entities follow the same tactical patterns but don't require a full state machine; see the
summary table for their bounded context and aggregate placement.

| Entity | Key attributes | Key invariant |
|---|---|---|
| `User` | id, role, email (VO), phone, barbershopId (nullable for CLIENT/SUPER_ADMIN) | Email must be unique platform-wide; barbershopId is null only for CLIENT and SUPER_ADMIN roles |
| `BarberProfile` | id, userId, bio, photoUrl | userId must reference a User with role BARBER |
| `BarberService` | id, barbershopId, name, price (VO: Money), durationMinutes | durationMinutes > 0; price >= 0 |
| `BarberSchedule` | id, barberProfileId, dayOfWeek, startTime, endTime (VO: TimeSlot) | No two schedule slots for the same barber/day may overlap |
| `ScheduleException` | id, barberProfileId, exceptionDate, isDayOff | Overrides the weekly schedule for a specific date |
| `RewardCoupon` | id, clientId, barbershopId, status (ACTIVE/USED), appointmentId (nullable) | Can only transition ACTIVE → USED once, when applied to an appointment at the same barbershop |
| `FinanceRecord` | id, barbershopId, type (INCOME/EXPENSE), amount (VO: Money), description, date | amount must be > 0 regardless of type; sign is derived from `type`, not stored |
| `InventoryProduct` | id, barbershopId, name, stock, minStockAlert | stock >= 0; alert triggered when stock <= minStockAlert |
| `SubscriptionPlan` | id, name, price (VO: Money), maxBarbers, featuresJson, isActive | maxBarbers must be enforced when a barbershop assigns barbers under this plan (`null`/unlimited for Premium) |
| `Notification` | id, userId, title, body, type, isRead | Always persisted regardless of FCM delivery success (INV per NFR 8.2) |

> **Deliberately excluded from the domain model:** `device_tokens` (FCM push token per user/device)
> and `password_reset_tokens` (6-digit recovery code, expiry, used flag) exist as database tables
> per the PRD, but are technical/infrastructure records with no business behavior or invariants of
> their own — they don't participate in any aggregate and are better modeled as plain persistence
> entities in the Auth/Notification application layer, not as DDD tactical building blocks.

---

## System Value Objects

### Value Object: Email

**Description:** A validated, normalized email address used for authentication and password recovery.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| value | String | The normalized (lowercased) email address |

**Validation rules:**

```
- Must match a valid email format: text@domain.extension
- Normalized to lowercase before comparison or storage
- Must be unique across all Users on the platform (uniqueness is a repository-level check, not
  a VO-level rule, but the VO guarantees the value is always well-formed before that check runs)
```

**Example:**

```typescript
class Email {
  private readonly value: string;

  constructor(email: string) {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      throw new DomainException(`Invalid email: ${email}`);
    }
    this.value = email.toLowerCase();
  }

  toString(): string { return this.value; }
  equals(other: Email): boolean { return this.value === other.value; }
}
```

---

### Value Object: Money

**Description:** A monetary amount in Colombian Pesos (COP), used for service prices, appointment
price snapshots, subscription plan prices, and finance records.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| amount | Decimal | Numeric value, always >= 0 |
| currency | String | Fixed to `"COP"` for the MVP (no multi-currency support) |

**Validation rules:**

```
- amount must be >= 0 (no negative prices or balances)
- currency is always "COP" in the MVP; the field exists to avoid a future breaking change
- Display formatting uses es-CO locale (e.g., "$39.900")
```

**Example:**

```typescript
class Money {
  private constructor(private readonly amount: number, private readonly currency: string) {}

  static of(amount: number, currency: string = 'COP'): Money {
    if (amount < 0) {
      throw new DomainException('Money amount cannot be negative');
    }
    return new Money(amount, currency);
  }

  static zero(currency: string = 'COP'): Money { return new Money(0, currency); }

  add(other: Money): Money {
    return Money.of(this.amount + other.amount, this.currency);
  }

  format(): string {
    return this.amount.toLocaleString('es-CO', { style: 'currency', currency: this.currency });
  }
}
```

---

### Value Object: TimeSlot

**Description:** A date + start/end time range used for appointments and barber schedules.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| date | Date | The calendar date |
| startTime | Time | Start of the slot |
| endTime | Time | End of the slot |

**Validation rules:**

```
- startTime must be strictly before endTime
- All times are interpreted in the America/Bogota (UTC-5) timezone
- Two TimeSlots overlap if one's startTime falls strictly before the other's endTime and vice versa
```

**Example:**

```typescript
class TimeSlot {
  constructor(
    private readonly date: string,
    private readonly startTime: string,
    private readonly endTime: string,
  ) {
    if (startTime >= endTime) {
      throw new DomainException('TimeSlot startTime must be before endTime');
    }
  }

  overlaps(other: TimeSlot): boolean {
    return this.date === other.date && this.startTime < other.endTime && other.startTime < this.endTime;
  }

  hoursUntilStart(): number { /* compares to now() in America/Bogota */ return 0; }
  isInThePast(): boolean { /* ... */ return false; }
}
```

---

## System Aggregates

### Aggregate: Appointment

**Aggregate Root:** `Appointment`

**Internal entities:**
- None — `Appointment` is a simple aggregate with no owned child entities.

**Value Objects:**
- `TimeSlot` (the booked date/time range)
- `Money` (`priceAtBooking`)

**Aggregate invariants:**

```
AGGR-INV-APPT-001: A barber cannot hold two overlapping Appointments (INV-APPT-001)
AGGR-INV-APPT-002: priceAtBooking never changes after the aggregate is created (INV-APPT-002)
AGGR-INV-APPT-003: Status only advances through valid transitions (INV-APPT-004)
```

**Why do these objects form an aggregate?**
> An `Appointment` and its time slot/price must always be consistent as a unit — a `TimeSlot`
> cannot exist detached from an `Appointment`, and the `priceAtBooking` snapshot only makes sense
> in the context of that specific booking. Cross-aggregate concerns (e.g., "does the client have
> an active coupon to apply?") are resolved by referencing another aggregate's ID
> (`RewardCoupon.id`) rather than loading it inside the same transaction.

---

### Aggregate: Barbershop

**Aggregate Root:** `Barbershop`

**Internal entities:**
- None directly — `BarberProfile`, `BarberService`, and inventory/finance records reference
  `barbershopId` but are modeled as separate aggregates within the same bounded context, since
  they change independently and at different rates than the `Barbershop` record itself (e.g.,
  adding a `FinanceRecord` shouldn't require locking the whole `Barbershop` row).

**Value Objects:**
- None beyond scalar fields; `trialEndsAt` is a plain `DateTime`.

**Aggregate invariants:**

```
AGGR-INV-SHOP-001: status transitions only follow the defined state machine (INV-SHOP-002)
AGGR-INV-SHOP-002: subscriptionPlanId always references an active plan (INV-SHOP-003)
```

**Why is `Barbershop` its own (small) aggregate?**
> `Barbershop` is the tenant root — every other aggregate in the system carries its
> `barbershopId` as a reference, not as an owned child, precisely so that high-frequency
> operations (booking appointments, granting stickers) never need to lock the tenant record
> itself.

---

### Aggregate: BarberProfile

**Aggregate Root:** `BarberProfile`

**Internal entities:**
- `BarberSchedule[]` — the barber's recurring weekly availability; only meaningful in the context
  of one barber.
- `ScheduleException[]` — day-specific overrides (day off, modified hours); must always be
  reconciled against the same barber's `BarberSchedule`.

**Value Objects:**
- `TimeSlot` (used inside each `BarberSchedule` entry and each `ScheduleException`)

**Aggregate invariants:**

```
AGGR-INV-BARBER-001: No two BarberSchedule entries for the same barberProfileId/dayOfWeek overlap
AGGR-INV-BARBER-002: A ScheduleException for a given date always overrides, never merges with,
                     the corresponding BarberSchedule entry for that day of week
```

**Why do these objects form an aggregate?**
> A barber's schedule and its exceptions must be evaluated together to compute real-time
> availability (see `AvailabilityService` below) — they change together, are always read
> together, and an orphaned `BarberSchedule` or `ScheduleException` (without its `BarberProfile`)
> has no business meaning.

---

### Aggregate: LoyaltyCard

**Aggregate Root:** `LoyaltyCard`

**Internal entities:**
- `LoyaltyTransaction[]` — the audit trail of GRANT/REDEEM events for the card, always created
  and read as part of the card's history.

**Value Objects:**
- None beyond scalar counters.

**Aggregate invariants:**

```
AGGR-INV-LOYAL-001: stickersCount must equal the net sum of GRANT/REDEEM LoyaltyTransactions
AGGR-INV-LOYAL-002: A REDEEM transaction can only be recorded if it satisfies INV-LOYAL-001 at
                     the time it is created
```

**Why do these objects form an aggregate?**
> `LoyaltyCard` and its `LoyaltyTransaction` history must stay consistent — the running
> `stickersCount` is a derived value that must always match the transaction log, so both are
> written in the same transaction. `RewardCoupon` is deliberately **excluded** from this
> aggregate (it's created via a domain event instead) because it is later read and mutated by the
> Appointment context when a client applies it at booking time — keeping it in a separate
> aggregate avoids a cross-context transactional dependency.

---

## Summary table of tactical building blocks

| Name | Type | Bounded Context | Aggregate Root? |
|------|------|----------------|----------------|
| `Appointment` | Entity | Appointment | Yes |
| `User` | Entity | Auth (shared reference elsewhere) | Yes |
| `Barbershop` | Entity | Barbershop | Yes |
| `BarberProfile` | Entity | Barbershop | Yes |
| `BarberSchedule` | Entity | Barbershop | No (inside `BarberProfile`) |
| `ScheduleException` | Entity | Barbershop | No (inside `BarberProfile`) |
| `BarberService` | Entity | Barbershop | Yes (simple, single-entity aggregate) |
| `LoyaltyCard` | Entity | Loyalty | Yes |
| `LoyaltyTransaction` | Entity | Loyalty | No (inside `LoyaltyCard`) |
| `RewardCoupon` | Entity | Loyalty | Yes (simple, single-entity aggregate) |
| `FinanceRecord` | Entity | Finance | Yes (simple, single-entity aggregate) |
| `InventoryProduct` | Entity | Barbershop | Yes (simple, single-entity aggregate) |
| `SubscriptionPlan` | Entity | Super Admin | Yes |
| `Notification` | Entity | Notification | Yes (simple, single-entity aggregate) |
| VO: `Email` | Value Object | Shared | N/A |
| VO: `Money` | Value Object | Shared | N/A |
| VO: `TimeSlot` | Value Object | Shared (Appointment, Barbershop) | N/A |
| `AvailabilityService` | Domain Service | Appointment | N/A |
| `LoyaltyRedemptionService` | Domain Service | Loyalty | N/A |

---

## Domain Services

A **Domain Service** is business logic that does not naturally belong to any single entity.
Use it when the operation spans multiple entities/aggregates, or the logic doesn't need its own
state.

```typescript
// Domain Service — checks a barber's real-time availability
// Spans the BarberProfile aggregate (schedule + exceptions) and the Appointment aggregate
// (existing bookings) — this is exactly why it can't live inside either aggregate root.
class AvailabilityService {
  isAvailable(barberId: BarberId, slot: TimeSlot, schedule: BarberSchedule[],
              exceptions: ScheduleException[], existingAppointments: Appointment[]): boolean {
    const exception = exceptions.find(e => e.date === slot.date);
    if (exception?.isDayOff) return false;

    const withinWorkingHours = schedule.some(s => s.dayOfWeek === slot.dayOfWeek() && s.contains(slot));
    if (!withinWorkingHours) return false;

    return !existingAppointments.some(appt => appt.slot.overlaps(slot));
  }
}
```

```typescript
// Domain Service — orchestrates a loyalty redemption across LoyaltyCard and RewardCoupon
// (two different aggregates, hence a Domain Service + Domain Event rather than a direct call)
class LoyaltyRedemptionService {
  redeem(card: LoyaltyCard, config: LoyaltyRewardsConfig): RewardCoupon {
    card.redeem(config.stickersRequired); // raises LoyaltyRedeemedEvent internally
    return RewardCoupon.issue(card.clientId, card.barbershopId);
  }
}
```

---

## Correlation with code

| Domain artifact | Package / folder in code | File |
|----------------|--------------------------|------|
| Aggregate Root `Appointment` | `src/main/java/.../appointment/domain/` | `Appointment.java` |
| Aggregate Root `Barbershop` | `src/main/java/.../barbershop/domain/` | `Barbershop.java` |
| Aggregate Root `BarberProfile` | `src/main/java/.../barbershop/domain/` | `BarberProfile.java` |
| Aggregate Root `LoyaltyCard` | `src/main/java/.../loyalty/domain/` | `LoyaltyCard.java` |
| Entity `RewardCoupon` | `src/main/java/.../loyalty/domain/` | `RewardCoupon.java` |
| Entity `FinanceRecord` | `src/main/java/.../finance/domain/` | `FinanceRecord.java` |
| Value Object `Money` | `src/main/java/.../shared/domain/valueobject/` | `Money.java` |
| Value Object `Email` | `src/main/java/.../shared/domain/valueobject/` | `Email.java` |
| Value Object `TimeSlot` | `src/main/java/.../shared/domain/valueobject/` | `TimeSlot.java` |
| Domain Service `AvailabilityService` | `src/main/java/.../appointment/domain/service/` | `AvailabilityService.java` |
| Domain Service `LoyaltyRedemptionService` | `src/main/java/.../loyalty/domain/service/` | `LoyaltyRedemptionService.java` |
| Repository `AppointmentRepository` | `src/main/java/.../appointment/domain/port/` | `AppointmentRepository.java` |

> Package paths follow the modular monolith structure described in the PRD (ADR-001):
> `com.barbersaas.{context}.domain`, one bounded context per module (auth, barbershop,
> appointment, loyalty, finance, notification, super-admin).

---

*Source: derived from the BarberSaaS PRD v1.0 (August 2026) — sections 7 (Functional
Requirements) and 10 (Data Model & Database). This is an inferred domain model, not a literal
transcription: the PRD documents database tables and API-level requirements, not tactical DDD
artifacts, so aggregate boundaries and invariants above are a reasonable design proposal and
should be validated with the engineering team against the actual codebase (ADR-001).*
