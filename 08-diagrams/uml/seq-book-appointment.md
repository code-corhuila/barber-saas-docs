# SEQ-02 · Book an Appointment Without Double-Booking

> **Type:** UML sequence · **Derived from:** `07-api/contracts/openapi/appointment-service.yaml`
> (`bookAppointment`), `06-data/models.md` §5 and §10, `02-domain/entities-and-rules.md`
> (INV-APPT-001/002) · **User story:** HU-APPT-001 (#4)

```mermaid
sequenceDiagram
  autonumber
  actor C as Client (mobile app)
  participant GW as api-gateway
  participant APPT as appointment-api
  participant SHOP as barbershop-api
  participant SCH as schedule-api
  participant DB as appointment-db

  C->>GW: POST /api/v1/appointments<br/>Idempotency-Key · {barberId, serviceId, date, startTime}
  GW->>APPT: forward + X-Correlation-Id
  APPT->>APPT: validate JWT · tenant = barbershopId from the token
  APPT->>DB: find idempotency_key (key, operation)
  alt key already used with the same body
    DB-->>APPT: resource_id
    APPT-->>C: 200 existing appointment
  else new key
    APPT->>SHOP: GET service (service token)
    SHOP-->>APPT: durationMinutes, priceCents (404 if not in the tenant)
    APPT->>SCH: GET /api/v1/availability (service token)
    SCH-->>APPT: barber works in that slot?
    APPT->>APPT: endTime = start + duration · priceAtBookingCents = priceCents
    APPT->>DB: BEGIN · INSERT appointment (PENDING) · INSERT idempotency_key
    alt overlapping active appointment
      DB-->>APPT: exclusion constraint violated
      APPT-->>C: 422 BUSINESS_RULE_VIOLATION
    else slot free
      DB-->>APPT: COMMIT
      APPT-->>C: 201 Created · Location · appointment
    end
  end
```

- The **database is the guarantee** against double booking: `ex_appointment_no_double_booking`
  rejects two active appointments of one barber that overlap in time, even if two requests pass
  the checks at the same instant.
- Outside the barber's schedule also answers `422` (`DEC-APPT-03`); a `barberId` or
  `serviceId` of another tenant answers `404` (`DEC-APPT-01`).
- The price is a snapshot: later changes to the service never touch the appointment
  (INV-APPT-002).
