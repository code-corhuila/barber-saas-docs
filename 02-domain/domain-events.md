# Domain Events — BarberSaaS

> **What to fill in here:** A domain event is a fact that occurred in the business.
> The name is ALWAYS in the past tense and in the ubiquitous language of the domain.

---

## Reality check first: there is no event bus

BarberSaaS is a **Modular Monolith** (ADR-002) — a single Spring Boot deployable unit.
There is **no message broker** (no Kafka/RabbitMQ) and **no Spring `ApplicationEvent`/
`@EventListener` mechanism** in the current codebase. What this document calls a "domain
event" is a **business-significant fact that triggers a synchronous, in-process Java method
call** (e.g., `AppointmentService.create()` calling `notificationService.notify(...)`
directly), all within the same database transaction where applicable.

This matches the explicit decision already recorded in `02-domain/domain-map.md` §5:
> *"Loyalty hooks directly into Appointment via in-process call... No message broker in MVP.
> In-process call is synchronous and easier to reason about. Event-driven will be introduced
> when Notifications moves to its own service."*

So the catalog below documents **what fact occurred and what it caused**, not a message
schema on a topic. Only `LoyaltyTransaction` rows are actually persisted as an append-only
event log today — everything else is an ephemeral side effect (e.g., a notification row is
created, but there is no separate "event" record of the fact that triggered it).

```
✓ AppointmentCreated       — a fact, past tense, describes what happened
✓ StickerGranted           — a fact, past tense

✗ CreateAppointment        (this is a command, not an event)
✗ AppointmentUpdated       (too generic, what changed?)
```

---

## Event Catalog

### Event: AppointmentCreated

| Field | Value |
|-------|-------|
| **Bounded Context** | Appointment |
| **Aggregate** | `Appointment` |
| **Trigger** | A client (or an admin, for a walk-in) successfully books an appointment via `AppointmentService.create()` — after the pessimistic-lock double-booking check passes and, if the client has an `ACTIVE` `RewardCoupon`, it is applied and marked `USED` in the same transaction |
| **Effect** | A `Notification` (type `APPOINTMENT_CONFIRMATION`, title "Cita reservada") is created for the client, in-process, in the same call |
| **Consumers** | Notification module only (no Loyalty involvement at booking time — coupon *consumption* happens here, but no sticker is granted yet) |
| **Persisted as its own record?** | No — only the resulting `Appointment` row (status `PENDING`) and `Notification` row exist; there is no separate event log entry |
| **Code reference** | `com.barbersaas.appointment.AppointmentService#create` |

---

### Event: AppointmentConfirmed

| Field | Value |
|-------|-------|
| **Bounded Context** | Appointment |
| **Aggregate** | `Appointment` |
| **Trigger** | `ADMIN_BARBERSHOP` or `BARBER` confirms a `PENDING` appointment via `AppointmentService.confirm()` |
| **Effect** | A `Notification` (type `APPOINTMENT_CONFIRMATION`, title "Cita confirmada") is created for the client |
| **Consumers** | Notification module |
| **Code reference** | `com.barbersaas.appointment.AppointmentService#confirm` |

---

### Event: AppointmentCancelled

| Field | Value |
|-------|-------|
| **Bounded Context** | Appointment |
| **Aggregate** | `Appointment` |
| **Trigger** | Client (within the barbershop's `cancellationPolicyHours` window) or admin cancels a `PENDING`/`CONFIRMED` appointment via `AppointmentService.cancel()` |
| **Effect** | A `Notification` (type `SYSTEM`, title "Cita cancelada") is created for the client |
| **Consumers** | Notification module |
| **Code reference** | `com.barbersaas.appointment.AppointmentService#cancel` |

---

### Event: AppointmentReminderDue

| Field | Value |
|-------|-------|
| **Bounded Context** | Appointment |
| **Trigger** | Not user-triggered — a scheduled job (`@Scheduled(cron = "0 0 18 * * *")`, daily at 18:00) finds all `CONFIRMED` appointments for the next calendar day |
| **Effect** | A `Notification` (type `REMINDER`, title "Recordatorio de cita") is created for each affected client |
| **Consumers** | Notification module |
| **Code reference** | `com.barbersaas.appointment.AppointmentReminderJob#sendTomorrowReminders` |

---

### Event: AppointmentMarkedNoShow

| Field | Value |
|-------|-------|
| **Bounded Context** | Appointment |
| **Trigger** | Not user-triggered — a scheduled job (`@Scheduled(cron = "0 0 1 * * *")`, daily at 01:00) finds `CONFIRMED` appointments dated yesterday that were never completed and sets their status to `NO_SHOW` |
| **Effect** | Status change only |
| **Consumers** | None — **no notification is sent today**, this is a silent status transition |
| **Code reference** | `com.barbersaas.appointment.AppointmentReminderJob#markPastAppointmentsAsNoShow` |

---

### Event: AppointmentCompleted

| Field | Value |
|-------|-------|
| **Bounded Context** | Appointment |
| **Trigger** | `BARBER` or `ADMIN_BARBERSHOP` marks an `IN_PROGRESS`/`CONFIRMED` appointment as `COMPLETED` via `AppointmentService.complete()` |
| **Effect** | Status change, income registered in Finance (`registerServiceIncome`), and — in the same transaction — `LoyaltyService.grantStickerForCompletedAppointment()` is called. If the barbershop has an active loyalty program, this grants a sticker (see `StickerGranted` below) using the completing staff member as `grantedBy`. If the barbershop has no active loyalty program, the call is a no-op: completing the appointment never fails because of this |
| **Consumers** | Loyalty (conditional — only when a loyalty program is active for the barbershop) |
| **Code reference** | `com.barbersaas.appointment.AppointmentService#complete`, `com.barbersaas.loyalty.LoyaltyService#grantStickerForCompletedAppointment` |

> **Known gap, not a drift:** the manual endpoint `POST /api/admin/loyalty/grant` still exists
> and is not idempotent against an appointment that already triggered an automatic grant. If
> staff also grant a sticker manually for the same completed appointment, the client can be
> credited twice. Documented as a follow-up (candidate for a short SPEC), not fixed here.

---

### Event: StickerGranted

| Field | Value |
|-------|-------|
| **Bounded Context** | Loyalty & Rewards |
| **Aggregate** | `LoyaltyCard` |
| **Trigger** | Either (a) `ADMIN_BARBERSHOP`/`BARBER` explicitly grants a sticker via `LoyaltyService.grantSticker()` (`POST /api/admin/loyalty/grant`), optionally linked to an appointment, or (b) automatically via `grantStickerForCompletedAppointment()` when an appointment is completed and the barbershop has an active loyalty program — which internally calls `grantSticker()`, so both paths share the same logic |
| **Effect** | `LoyaltyCard.stickersCount` is incremented by 1 (the card is created on first grant if the client has none yet at this barbershop); a `LoyaltyTransaction` row (`type = STICKER_EARNED`) is persisted as the audit record; a `Notification` (`type = LOYALTY`) is created for the client |
| **Persisted as its own record?** | **Yes** — `LoyaltyTransaction` is the one real append-only event log in the system today |
| **Consumers** | Notification module |
| **Code reference** | `com.barbersaas.loyalty.LoyaltyService#grantSticker` |

---

### Event: RewardRedeemed

| Field | Value |
|-------|-------|
| **Bounded Context** | Loyalty & Rewards |
| **Aggregate** | `LoyaltyCard` |
| **Trigger** | `ADMIN_BARBERSHOP` or `BARBER` redeems a reward on behalf of a client (at the moment the physical reward is handed over) via `LoyaltyService.redeemReward()`, once `stickersCount >= stickersRequired` |
| **Effect** | `LoyaltyCard.stickersCount` decremented by `stickersRequired`, `totalRewardsRedeemed` incremented, a new `RewardCoupon` (`status = ACTIVE`) is created, a `LoyaltyTransaction` row (`type = REWARD_REDEEMED`) is persisted, and a `Notification` (`type = LOYALTY`) is created for the client — all in the same transaction |
| **Persisted as its own record?** | Yes — `LoyaltyTransaction` row, plus the `RewardCoupon` itself as a first-class entity |
| **Consumers** | Notification module; the `RewardCoupon` is later consumed by `AppointmentService.create()` (see `AppointmentCreated` above) when the client books their next appointment |
| **Code reference** | `com.barbersaas.loyalty.LoyaltyService#redeemReward` |

---

### Event: PasswordResetRequested

| Field | Value |
|-------|-------|
| **Bounded Context** | Identity & Auth |
| **Trigger** | A user requests password recovery |
| **Effect** | A 6-digit reset code is generated and emailed via `EmailService.sendPasswordResetCode()` (Gmail SMTP / Spring Mail) |
| **Consumers** | Email only — this does **not** go through the in-app `Notification` module (unlike every event above) |
| **Code reference** | `com.barbersaas.auth.AuthService`, `com.barbersaas.notification.EmailService#sendPasswordResetCode` |

---

## Event Summary Table

| Event | Bounded Context | Channel | Consumers | Persisted as event? |
|--------|----------------|---------|-------------|---------|
| `AppointmentCreated` | Appointment | In-process method call | Notification | No |
| `AppointmentConfirmed` | Appointment | In-process method call | Notification | No |
| `AppointmentCancelled` | Appointment | In-process method call | Notification | No |
| `AppointmentReminderDue` | Appointment | Scheduled job (cron, daily 18:00) | Notification | No |
| `AppointmentMarkedNoShow` | Appointment | Scheduled job (cron, daily 01:00) | *(none — silent)* | No |
| `AppointmentCompleted` | Appointment | In-process method call | Loyalty (conditional — only if the barbershop has an active program; see known gap above) | No |
| `StickerGranted` | Loyalty & Rewards | In-process method call | Notification | **Yes** — `LoyaltyTransaction` |
| `RewardRedeemed` | Loyalty & Rewards | In-process method call | Notification; Appointment (coupon consumption, later) | **Yes** — `LoyaltyTransaction` + `RewardCoupon` |
| `PasswordResetRequested` | Identity & Auth | In-process method call | Email (Gmail SMTP) | No |

---

## When this document stops being accurate

The moment any bounded-context module is extracted into its own deployable service (see the
extraction triggers in ADR-002), the events above stop being synchronous method calls and
must become real asynchronous messages with a schema, a topic, a delivery guarantee, and an
idempotency strategy on the consumer side. At that point, re-introduce the standard
compatible/incompatible schema-evolution rules (additive fields only, never remove/rename/
change type, never make optional fields mandatory) that a distributed event catalog needs —
they don't apply yet because there is no wire format to evolve.

---

## Correlations

- Bounded contexts these events belong to → `02-domain/domain-map.md`
- Entities and invariants referenced above → `02-domain/entities-and-rules.md`
- Decision to stay in-process (no broker) for MVP → `05-architecture/decisions/records/ADR-002-modular-monolith.md`
- Notification delivery mechanism (FCM + email) → `01-context/scope.md` → Included integrations
