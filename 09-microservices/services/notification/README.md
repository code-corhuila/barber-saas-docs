# Notification

## Responsibility

Delivery of in-app, push (Firebase Cloud Messaging), and email notifications triggered by
domain events elsewhere in the system. Persists every notification to the database
regardless of delivery outcome — graceful degradation, so a failed push or email never
blocks the operation that triggered it (`02-domain/domain-map.md`).

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Current state | Internal module of the monolith — `com.barbersaas.notification` |
| Planned state | Independently deployable `notification-service` (see `decisions.md`) |
| Endpoint prefix | `/api/notifications` |
| DB engine | PostgreSQL — tables `notifications`, `device_tokens` (shared schema today, see `data-model.md`) |
| Communicates with (today) | Called in-process by `appointment` and (planned) `loyalty` |
| Consumed by (planned) | `appointment`, `loyalty`, via HTTP or event once extracted |

---

## Responsibilities (what this module DOES)

- Persist in-app notifications (title, body, type, read status) per user.
- Register and store FCM device tokens per user device (Android/iOS).
- Attempt push delivery via Firebase Admin SDK and email delivery via Spring Mail /
  Gmail SMTP.
- Record every notification regardless of whether the push/email attempt succeeded.

## Out of scope (what it does NOT do)

- Decide *when* a notification should be sent — that is triggered by the calling module
  (`appointment`, `loyalty`) via an in-process call today.
- Own any business data other than notifications and device tokens.

---

## How to run it locally

Today it is not a separate deployable — it runs as part of the monolith:

```bash
# From the backend project root
./mvnw spring-boot:run
```

Once extracted (`ADR-003-academic-microservice-extraction.md`), this section will document
its own `docker compose up -d notification-service` and health-check endpoint.

---

## Related documents

- [data-model.md](./data-model.md) — current DB schema
- [events.md](./events.md) — current in-process communication and the planned HTTP/event
  communication once extracted
- [decisions.md](./decisions.md) — the extraction decision (`ADR-003`) and its rationale
- `ADR-003-academic-microservice-extraction.md` — `05-architecture/decisions/records/`
