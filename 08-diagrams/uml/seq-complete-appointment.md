# SEQ-03 · Complete an Appointment (Outbox → Loyalty, Notifications)

> **Type:** UML sequence · **Derived from:** `07-api/contracts/openapi/appointment-service.yaml`
> (`completeAppointment`), `06-data/models.md` §6, §7, §10, `02-domain/domain-events.md`
> (`AppointmentCompleted`, `StickerGranted`) · **Rule:** norm 5.3.11, 7.5

A change in one domain reaching others without a distributed transaction.

```mermaid
sequenceDiagram
  autonumber
  actor B as Barber (mobile app)
  participant GW as api-gateway
  participant APPT as appointment-api
  participant ADB as appointment-db
  participant R as relay (transport pending, AT-004)
  participant LOY as loyalty-api
  participant NOTIF as notifications-api
  participant FCM as Firebase FCM

  B->>GW: POST /api/v1/appointments/{id}/complete
  GW->>APPT: forward
  APPT->>ADB: BEGIN · UPDATE status IN_PROGRESS → COMPLETED<br/>INSERT outbox_event AppointmentCompleted · COMMIT
  APPT-->>B: 200 appointment (COMPLETED)

  Note over R,ADB: Later, outside the request
  R->>ADB: read outbox_event WHERE published_at IS NULL
  R->>LOY: AppointmentCompleted {appointmentId, clientId, barbershopId}
  LOY->>LOY: active program? INSERT loyalty_transaction STICKER_EARNED<br/>unique per appointment · outbox StickerGranted
  R->>ADB: SET published_at
  R->>NOTIF: StickerGranted
  NOTIF->>NOTIF: store notification (unique sourceEventId)
  NOTIF->>FCM: push to the client's device tokens
  FCM-->>NOTIF: SENT or FAILED (recorded in deliveryAttempts)
```

- **At-least-once delivery is safe:** a redelivered event hits
  `uq_loyalty_transaction_sticker_per_appointment` or `uq_notification_source_event` and does
  nothing twice.
- A walk-in (`client_id` null) earns no sticker. A barbershop without an active loyalty
  program ignores the event; completing never fails because of loyalty.
- Registering the income in finance-inventory from the same event is the prototype's
  behavior (`registerServiceIncome`); its contract does not declare it yet.
