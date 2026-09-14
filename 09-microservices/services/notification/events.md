# Notification — Communication (Current and Planned)

## Current (in-process, within the monolith)

| From module | To module | Trigger | Channel |
|---|---|---|---|
| `appointment` | `notification` | `create()`, `confirm()`, `cancel()`, daily reminder job | In-process Java method call (`notificationService.notify(...)`) |
| `loyalty` | `notification` | Sticker granted / reward redeemed — **planned, not implemented** | In-process (planned) |

Not called today: `appointment.complete()` and the NO_SHOW auto-marking job do not trigger
a notification (see `decisions.md` — known drift, not addressed by this document).

## Planned (once extracted to `notification-service`)

Per `PDR-BarberSaaS.md` §10.4: the in-process call becomes either an HTTP POST to
`notification-service` or an asynchronous event publish/subscribe. The specific mechanism
(synchronous REST vs. message broker) is a decision for the extraction work itself, not
fixed by `ADR-003`.

| From module | To `notification-service` | Channel (once extracted) |
|---|---|---|
| `appointment` | Notify on create/confirm/cancel/reminder | HTTP POST or event publish |
| `loyalty` | Notify on sticker granted / reward redeemed | HTTP POST or event publish |

---

## Correlations

- Extraction decision → `05-architecture/decisions/records/ADR-003-academic-microservice-extraction.md`
- Domain events and relationship table → `02-domain/domain-map.md`, `02-domain/domain-events.md`
