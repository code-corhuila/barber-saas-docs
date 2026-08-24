# Domain Events — Luxury Barber

> **What to fill in here:** A domain event is a fact that occurred in the business.
> They are the backbone of asynchronous communication between bounded contexts.
> The name is ALWAYS in the past tense and in the ubiquitous language of the domain.

---

## What is a domain event?

A **Domain Event** communicates that something important occurred in the business.
It is an immutable message that describes the fact in the past tense.

```
✓ AppointmentCreated
✓ PaymentProcessed
✓ PaymentFailed
✓ AppointmentConfirmed

✗ CreateAppointment (this is a command, not an event)
✗ AppointmentUpdated (too generic, what changed?)
✗ PaymentEvent (does not indicate what happened)
```

### Difference between Command and Event

| Concept | Intention | Time | Can it fail? |
|----------|-----------|--------|---------------|
| **Command** | Instruction to do something | Present | Yes |
| **Event** | Notification of something that happened | Past | No (it already happened) |

```
Customer → [BookAppointment] → Appointment Service → [AppointmentCreated] → Payment Service
            (REST Command)                                (AMQP Event)
```

---

## Event Catalog

### Event: AppointmentCreated

| Field | Value |
|-------|-------|
| **Name** | `AppointmentCreated` |
| **Bounded Context** | Appointment Service |
| **Aggregate** | Appointment (Cita) |
| **Trigger** | The customer successfully books an appointment (validating schedule and barber availability). |
| **Consumers** | Payment Service (initiates payment flow or amount reservation). |
| **Channel (topic)** | `appointment.events.appointment_created` |
| **Schema Version** | `v1` |
| **Delivery Guarantee** | At-least-once (requires idempotency in Payment Service). |

**Payload (JSON schema):**

```json
{
  "eventId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "eventType": "AppointmentCreated",
  "aggregateId": "appt-8849-a1b2",
  "aggregateType": "Appointment",
  "occurredAt": "2026-08-23T10:30:00Z",
  "version": 1,
  "payload": {
    "customer_id": "usr-9901-xyz",
    "barber_id": "bar-002-abc",
    "service_id": "srv-01-haircut-beard",
    "date": "2026-08-25",
    "time": "15:00",
    "initial_status": "pending_payment"
  },
  "metadata": {
    "correlationId": "req-11223344",
    "causationId": "cmd-book-appointment-998",
    "userId": "usr-9901-xyz"
  }
}
```

**What do consumers do with this event?**

| Consumer Service | Action | Idempotent? |
|--------------------|--------|--------------|
| Payment Service | Creates a pending payment record associated with the appointment. | Yes — uses eventId as idempotency key. |

---

### Event: PaymentProcessed

| Field | Value |
|-------|-------|
| **Name** | `PaymentProcessed` |
| **Bounded Context** | Payment Service |
| **Aggregate** | Payment (Pago) |
| **Trigger** | The payment gateway (mocked) or the receptionist registers a successful payment. |
| **Consumers** | Appointment Service, Notification Service |
| **Channel (topic)** | `payment.events.payment_processed` |
| **Schema Version** | `v1` |
| **Delivery Guarantee** | At-least-once |

**Payload (JSON schema):**

```json
{
  "eventId": "e98df21a-67dd-4372-b768-1f13c3d4e580",
  "eventType": "PaymentProcessed",
  "aggregateId": "pay-5511-xyz",
  "aggregateType": "Payment",
  "occurredAt": "2026-08-23T10:35:00Z",
  "version": 1,
  "payload": {
    "appointment_id": "appt-8849-a1b2",
    "amount": 45000,
    "method": "transfer",
    "status": "successful"
  }
}
```

---

## Event Flow: Saga Appointment → Payment → Notification

This is the central distributed flow (star process) using choreography.

```
[Customer]
  │
  │  Book Appointment (REST)
  ▼
[Appointment Service]
  │
  │  AppointmentCreated (Event via RabbitMQ)
  ├─────────────────────────────────────────┐
  │                                         ▼
  │                                [Payment Service]
  │                                Processes payment (mock)
  │                                         │
  │  PaymentProcessed / PaymentFailed       │
  │◄────────────────────────────────────────┘
  │
  │  Updates appointment status
  │  AppointmentConfirmed (Event via RabbitMQ)
  ├─────────────────────────────────────────┐
                                            ▼
                                   [Notification Service]
                                   Applies Circuit Breaker
                                   Sends SMTP email to customer
```

---

## Schema Evolution Strategy

Events are contracts. Changing them in an incompatible way breaks consumers.

### What is a compatible change (does not break)?

```
✓ Add a new optional field to the payload (e.g., discount_applied)
✓ Add a new event type (e.g., AppointmentRescheduled)
✓ Make a mandatory field → optional
```

### What is an incompatible change (breaks)?

```
✗ Remove a field from the payload (e.g., remove barber_id)
✗ Change the type of a field (string → number)
✗ Make an optional field → mandatory
✗ Change the event name
```

---

## Event Summary Table

| Event | Origin Context | Topic | Consumers | Version |
|--------|----------------|-------|-------------|---------|
| `AppointmentCreated` | Appointment Service | `appointment.events.appointment_created` | Payment Service | v1 |
| `PaymentProcessed` | Payment Service | `payment.events.payment_processed` | Appointment Svc, Notification Svc | v1 |
| `PaymentFailed` | Payment Service | `payment.events.payment_failed` | Appointment Svc | v1 |
| `AppointmentConfirmed` | Appointment Service | `appointment.events.appointment_confirmed` | Notification Svc | v1 |

---

## Policies — Reactions to events (Saga Choreography)

A **Policy** (or Saga step) describes what happens automatically when an event arrives. In Luxury Barber, the saga is managed through the following distributed policies:

| Triggering Event | Policy | Internally Issued Command | Executing Service |
|------------------|--------|-----------------|---------|
| `AppointmentCreated` | Whenever an appointment is created, register a pending payment. | `CreatePendingPayment` | Payment Service |
| `PaymentProcessed` | Whenever a payment is successful, confirm the associated appointment. | `ConfirmAppointment` | Appointment Service |
| `PaymentFailed` | Whenever a payment fails, compensate by leaving the appointment in 'pending payment' or canceling it. | `RevertPendingAppointment` | Appointment Service |
| `AppointmentConfirmed` | Whenever an appointment is confirmed, send an email to the customer. | `SendConfirmationEmail` | Notification Service |

---

## Resiliency Patterns for Events (Luxury Barber)

### At-least-once delivery + Idempotency

RabbitMQ guarantees that the event is delivered **at least once**. Consumers must be **idempotent**, validating the event's unique ID to avoid processing duplicate payments or sending repeated emails.

```typescript
// Idempotent consumer - Appointment Service
async function processSuccessfulPayment(event: PaymentProcessed): Promise<void> {
  // 1. Check if it was already processed using eventId
  if (await eventAlreadyProcessed(event.eventId)) {
    logger.info(`Payment event ${event.eventId} already processed, ignoring`);
    return;
  }

  // 2. Process the event (Update MongoDB 'appointmentdb')
  await updateAppointmentStatus(event.payload.appointment_id, 'confirmed');
  await publishAppointmentConfirmedEvent(event.payload.appointment_id);

  // 3. Mark as processed (in the same persistence transaction)
  await markEventProcessed(event.eventId);
}
```

### Fault Tolerance: Circuit Breaker

If `Notification Service` goes down, the `Payment Service` or `Appointment Service` must not be blocked. Resilience4j will handle the fallback.

### Dead Letter Queue (DLQ)

When an event (e.g., sending a notification) fails after N retries, it goes to the DLQ.

| RabbitMQ Configuration | Defined Value |
|--------------|------------------|
| Retries before DLQ | 3 |
| Backoff | Exponential (Spring AMQP default) |
| DLQ Retention | 7 days for analysis |
