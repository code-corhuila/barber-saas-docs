# Notification — Data Model

> Current state: tables live in the shared PostgreSQL schema of the monolith, tenant-scoped
> like every other table. This is the schema as of the modular-monolith phase — it does not
> yet reflect a separate database for `notification-service` (that decision is made at
> extraction time, per `ADR-003-academic-microservice-extraction.md`).

## `notifications`

An in-app message for a specific user.

| Field | Meaning |
|---|---|
| `title` | Notification headline |
| `body` | Notification message |
| `type` | `NotificationType` — category of the notification (e.g. appointment confirmed, reminder) |
| `read` | Whether the user has opened/dismissed it |
| `user_id` | Owner of the notification |

## `device_tokens`

The FCM registration token for a user's Android or iOS device, used for push delivery.

| Field | Meaning |
|---|---|
| `token` | FCM device token |
| `user_id` | Owner of the device |
| `platform` | Android / iOS |

---

## Correlations

- Ubiquitous language and bounded-context classification → `02-domain/domain-map.md`
  (Bounded Context: Notifications)
- Full data dictionary for the system → `06-data/data-dictionary.md`
