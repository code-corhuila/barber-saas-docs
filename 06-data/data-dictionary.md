# Data Dictionary — BarberSaaS

> Exact meaning of fields whose purpose or valid values aren't obvious from the column name
> alone. Sourced from `db/init.sql`, the JPA entities in `com.barbersaas.domain.entity`, and
> the business invariants in `02-domain/entities-and-rules.md`. This is not a repeat of every
> column in `06-data/models.md` — only the ones where a reader could reasonably get it wrong.

---

## Enums and their valid values

| Field | Table | Type | Valid values | Meaning |
|-------|-------|------|--------------|---------|
| `role` | `users` | ENUM | `SUPER_ADMIN`, `ADMIN_BARBERSHOP`, `BARBER`, `CLIENT` | Fixed 4-role RBAC — see `00-governance/security-policy.md`. Not extensible without a schema migration |
| `status` | `barbershops` | ENUM | `TRIAL`, `ACTIVE`, `SUSPENDED`, `CANCELLED` | Default `TRIAL` on insert. Only `SUPER_ADMIN` may transition it (INV-SHOP-002) |
| `status` | `appointments` | ENUM | `PENDING`, `CONFIRMED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`, `NO_SHOW` | Default `PENDING`. See the state machine in `02-domain/entities-and-rules.md` — not every transition is legal from every state |
| `type` | `loyalty_transactions` | ENUM | `STICKER_EARNED`, `REWARD_REDEEMED` | One row per grant or redemption — this table is the audit log, not a config table |
| `status` | `reward_coupons` | ENUM | `ACTIVE`, `USED` | Transitions `ACTIVE` → `USED` exactly once, when applied to a booking (`appointment_id` gets set at that point) |
| `type` | `finance_records` | ENUM | `INCOME`, `EXPENSE` | Determines the financial sign; `amount` itself is always stored positive (see below) |
| `movement_type` | `inventory_movements` | ENUM | `IN`, `OUT` | Direction of a stock change; `quantity` is always positive, direction comes from this field |
| `type` | `notifications` | ENUM | `APPOINTMENT_CONFIRMATION`, `REMINDER`, `PROMOTION`, `SYSTEM` | See `06-data/models.md` note: cancellations use `SYSTEM`, not a dedicated type; `PROMOTION` is defined but not currently wired to any code path |
| `discount_type` | `promotions` | ENUM | `PERCENTAGE`, `FIXED_AMOUNT`, `TWO_FOR_ONE` | Undocumented feature (see `06-data/models.md`) — meaning of `discount_value` depends on this: a percentage (0–100) if `PERCENTAGE`, a COP amount if `FIXED_AMOUNT`, ignored if `TWO_FOR_ONE` |

---

## Fields with a non-obvious default or business meaning

| Field | Table | Default | Why it matters |
|-------|-------|---------|-----------------|
| `barbershop_id` | `users` | `NULL` | `NULL` is not "missing data" — it is the correct, expected value for `SUPER_ADMIN` and `CLIENT` roles, both of which are platform-wide, not barbershop-scoped. A `NOT NULL` constraint here would be a bug |
| `cancellation_policy_hours` | `barbershops` | `2` | Real seeded/default value (verified in `init.sql`) — a client can cancel a `PENDING`/`CONFIRMED` appointment up to 2 hours before its start time by default. Each barbershop's admin can override this per-shop, but there is no override at the individual-service or individual-appointment level |
| `price_at_booking` | `appointments` | *(no default — always set at insert)* | A **snapshot**, not a live reference to `services.price`. If a barbershop later changes a service's price, every existing appointment keeps the price it was booked at (INV-APPT-002). Never join to `services.price` to determine what a past appointment cost |
| `stickers_required` | `loyalty_rewards_config` | `10` | Per-barbershop configurable threshold — do not assume 10 is universal; always read this table, never hardcode |
| `min_stock_alert` | `inventory_products` | `0` | A product with the default (`0`) effectively has alerting disabled — `current_stock` would have to go negative (which nothing prevents at the DB level) to trigger it. A barbershop must explicitly set a positive threshold to get real alerts |
| `is_active` (on `services`, `barber_schedules`, `subscription_plans`, `loyalty_rewards_config`, `promotions`) | multiple | `TRUE` | This project's soft-disable pattern — see "Deletion model" below. It is **not** the same concept as `deleted_at` (which doesn't exist in this schema at all) |
| `used_at` | `reward_coupons` | `NULL` | `NULL` = still `ACTIVE`. Gets a timestamp exactly when `status` flips to `USED`. Do not rely on `status = 'USED'` alone if you need *when* it was used — check this column |

---

## Deletion model (there is no soft delete)

This schema has **no `deleted_at` column anywhere** — a deliberate departure from the
generic scaffold's original recommendation (soft delete on every table), confirmed by
reading every `CREATE TABLE` statement in `init.sql`. Two different mechanisms are used
instead, and a reader needs to know which applies to which table:

| Mechanism | Applies to | Effect |
|-----------|-----------|--------|
| `ON DELETE CASCADE` (hard delete) | Almost every child table hanging off `barbershops` or `users` | Deleting a `barbershops` row physically deletes every `services`, `appointments`, `loyalty_cards`, etc. row for that tenant. There is no recovery path at the DB level |
| `is_active` boolean flag | `services`, `barber_schedules`, `subscription_plans`, `loyalty_rewards_config`, `promotions` | "Deleting" from the user's point of view is really deactivation — the row stays, historical references (e.g., an old appointment's `service_id`) keep resolving |
| Status enum, not deletion | `barbershops` (`status = 'CANCELLED'`), `appointments` (`status = 'CANCELLED'`), `reward_coupons` (implicitly via `USED`) | The lifecycle itself has a terminal state; there is no separate delete concept for these |

**Implication:** there is currently no recovery path if a `barbershops` row is ever deleted
by mistake — everything cascades. If this project needs an "undo," it would need to be
added (soft delete or an audit/backup mechanism), it doesn't exist today.

---

## Money and currency

Every monetary field (`price`, `amount`, `discount_value`, `price_at_booking`) is
`DECIMAL(10,2)`, always in Colombian Pesos (COP) — there is no `currency` column anywhere in
this schema, unlike the `Money` value object described in
`02-domain/entities-and-rules.md`, which models `currency` as a field "fixed to COP for the
MVP." At the database level, COP is not stored, only assumed. If multi-currency is ever
needed, every monetary column needs a migration to add a `currency` column — there's no
schema hook for it today.

---

## Time and timezone

- `barbershops.timezone` defaults to `'America/Bogota'` and is stored per barbershop — but
  every `appointments.appointment_date`/`start_time`/`end_time` is a plain `DATE`/`TIME`
  with **no timezone attached at the column level** (`TIME`, not `TIMESTAMPTZ`). Correctness
  depends entirely on the application always interpreting these against the owning
  barbershop's `timezone`, never against server-local or UTC time. This is a real
  correctness risk worth a dedicated test (see the untested-paths list in
  `04-requirements/traceability-matrix.md`).
- `created_at`/`updated_at` use `TIMESTAMP` (which in MySQL is timezone-aware, stored as UTC
  and converted on read using the connection's timezone), not `TIMESTAMPTZ`/`DATETIME`. This
  distinction matters for the Phase 2 PostgreSQL migration — see
  `06-data/migration-strategy.md`.

---

## Correlations

- Full schema and DDL → `06-data/models.md`
- Business invariants these fields encode → `02-domain/entities-and-rules.md`
- Migration to PostgreSQL → `06-data/migration-strategy.md`
