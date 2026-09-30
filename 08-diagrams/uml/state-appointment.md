# ST-01 · Appointment Lifecycle

> **Type:** UML state machine · **Derived from:** `02-domain/entities-and-rules.md`
> (Appointment, INV-APPT-004), `07-api/contracts/openapi/appointment-service.yaml`,
> `06-data/models.md` §5 (`chk_appointment_status`)

```mermaid
stateDiagram-v2
  [*] --> PENDING : book (POST /appointments)
  PENDING --> CONFIRMED : confirm — ADMIN_BARBERSHOP, BARBER
  PENDING --> CANCELLED : cancel
  CONFIRMED --> IN_PROGRESS : start — ADMIN_BARBERSHOP, BARBER
  CONFIRMED --> CANCELLED : cancel — CLIENT only within the policy window
  CONFIRMED --> NO_SHOW : no-show — staff, or the worker's daily job
  IN_PROGRESS --> COMPLETED : complete — writes AppointmentCompleted to the outbox
  COMPLETED --> [*]
  CANCELLED --> [*]
  NO_SHOW --> [*]
```

| Rule | Effect on the diagram |
|---|---|
| INV-APPT-004 | Any transition not drawn answers `422 INVALID_STATUS_TRANSITION` |
| INV-APPT-003 | A `CLIENT` cancels only while more than `cancellation_policy_hours` remain before the start |
| INV-APPT-001 | `CANCELLED` and `NO_SHOW` free the slot: the no-double-booking constraint ignores them |
| Terminal states | `COMPLETED`, `CANCELLED`, `NO_SHOW` never change again |
