# Events — Auth Service

> **Framework example, not BarberSaaS's identity-auth.** BarberSaaS's identity event is
> `PasswordResetRequested` (`02-domain/domain-events.md`), written to the `outbox_event` table
> of `identity-auth-db` (`06-data/models.md` §10).

> The Auth Service publishes domain events when significant changes happen to a user's
> identity. Events are how other services react without querying auth-service directly.

---

## Published events

### `user.registered`

**When it is emitted:** When a new user's registration completes successfully.
**Topic / Exchange:** `auth.events` (or `user.registered`, depending on the project's broker)
**Typical consumers:** notification-service (send a welcome email), analytics-service

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440001",
  "eventType": "user.registered",
  "aggregateId": "550e8400-e29b-41d4-a716-446655440000",
  "occurredAt": "2024-01-15T10:30:00.000Z",
  "version": 1,
  "payload": {
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "roles": ["USER"]
  },
  "metadata": {
    "correlationId": "req-abc-123",
    "causationId": "cmd-register-456"
  }
}
```

---

### `user.login_failed`

**When it is emitted:** When a login attempt fails because of invalid credentials.
**Topic / Exchange:** `auth.events`
**Typical consumers:** security-monitoring-service (brute-force attack detection)

```json
{
  "eventId": "...",
  "eventType": "user.login_failed",
  "aggregateId": "user-email-hash",
  "occurredAt": "2024-01-15T10:31:00.000Z",
  "version": 1,
  "payload": {
    "email": "user@example.com",
    "failedAttempts": 3,
    "ipAddress": "192.168.1.100",
    "userAgent": "Mozilla/5.0 ..."
  }
}
```

**Privacy note:** The email is included to correlate attempts, but the consuming service must
not log it in plain text. Consider hashing it with HMAC before publishing.

---

### `user.password_changed`

**When it is emitted:** When the password is changed successfully.
**Topic / Exchange:** `auth.events`
**Typical consumers:** notification-service (alert the user about the change)

```json
{
  "eventType": "user.password_changed",
  "aggregateId": "[userId]",
  "payload": {
    "userId": "[userId]",
    "changedAt": "2024-01-15T10:35:00.000Z"
  }
}
```

---

### `user.account_locked`

**When it is emitted:** When an account is locked because of too many failed attempts.
**Topic / Exchange:** `auth.events`
**Typical consumers:** notification-service (alert the user), security-monitoring

---

## Consumed events

**None at the moment.** The Auth Service does not react to other services' events.

If in the future it needs business data (e.g. suspending an account for non-payment), it
must subscribe to the corresponding event — record that decision in `decisions.md`.

---

## Delivery guarantees

**At-least-once:** Events are published after the database transaction commits (Outbox Pattern recommended — see `05-architecture/pattern-guide.md`).

**Idempotency:** Consumers must be idempotent — they may receive the same event more than once. The `eventId` is the idempotency key.

---

## Correlations

- Outbox pattern → `05-architecture/pattern-guide.md#outbox-pattern`
- Standard event format → `02-domain/domain-events.md`
- User data (model) → `data-model.md`
