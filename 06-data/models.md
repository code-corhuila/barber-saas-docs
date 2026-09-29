# Data Models — BarberSaaS

> Target data model: **one database per domain** (ADR-004), PostgreSQL for seven domains and
> MongoDB for notifications (ADR-006), under the conventions of
> [ADR-010](../05-architecture/decisions/records/ADR-010-data-conventions-per-domain.md).
> Every table below backs a resource of `07-api/contracts/openapi/`: same fields
> (`camelCase` ↔ `snake_case`), same types, same sets of values.
>
> Each schema is versioned only in its own `barber-saas-<domain>-db` repository with Liquibase
> (ADR-007). The DDL here is the reference those changesets implement; in the `-db` repository
> tables and foreign keys go in separate folders (annex A).

> **History.** Until 2026-09-28 this file transcribed the prototype's `db/init.sql` (MySQL 8,
> one shared schema, `BIGINT` ids, `DECIMAL` money). That transcription is still available in
> the git history of this file and in `code-corhuila/barber-saas`. The business rules it
> surfaced are carried over below; its types are not.

---

## 1. Ownership map

| Domain | Database (instance) | Engine | Schema | Owns | Contract |
|---|---|---|---|---|---|
| Identity & Auth | `identity-auth-db` | PostgreSQL | `identity_auth` | `app_user`, `refresh_token`, `password_reset_token` | `auth-service.yaml` |
| Barbershop | `barbershop-db` | PostgreSQL | `barbershop` | `barbershop`, `service`, `barber_profile`, `barber_specialty` | `barbershop-service.yaml` |
| Schedule | `schedule-db` | PostgreSQL | `schedule` | `barber_schedule`, `schedule_exception` | `schedule-service.yaml` |
| Appointment | `appointment-db` | PostgreSQL | `appointment` | `appointment` | `appointment-service.yaml` |
| Loyalty | `loyalty-db` | PostgreSQL | `loyalty` | `loyalty_rewards_config`, `loyalty_card`, `loyalty_transaction`, `reward_coupon` | `loyalty-service.yaml` |
| Notifications | `notifications-db` | MongoDB (rs0) | `notifications` | `notification`, `device_token` | `notification-service.yaml` |
| Finance & Inventory | `finance-inventory-db` | PostgreSQL | `finance_inventory` | `finance_record`, `inventory_product`, `inventory_movement` | `finance-inventory-service.yaml` |
| Platform Admin | `platform-admin-db` | PostgreSQL | `platform_admin` | `subscription_plan` | `platform-admin-service.yaml` |

Every domain that creates resources over HTTP also owns an `idempotency_key` table (§10), and
every domain that publishes events owns an `outbox_event` table (§10): identity-auth,
appointment and loyalty.

### Cross-domain references (no foreign keys)

| Column | In | Points to | Checked through |
|---|---|---|---|
| `barbershop_id` | every tenant-scoped table | `barbershop.barbershop` | The JWT (tenant claim), never the body |
| `user_id`, `client_id`, `granted_by_user_id`, `created_by_user_id`, `created_by` | several | `identity_auth.app_user` | The JWT `sub`, or `auth-service` |
| `barber_profile_id`, `barber_id` | schedule, appointment | `barbershop.barber_profile` | `barbershop-service` |
| `service_id` | appointment | `barbershop.service` | `barbershop-service` |
| `appointment_id`, `related_appointment_id` | loyalty, finance | `appointment.appointment` | `appointment-service` |
| `plan_id` | barbershop | `platform_admin.subscription_plan` | `platform-admin-service` |

---

## 2. Identity & Auth — `identity_auth`

```sql
CREATE TABLE app_user (                       -- "user" is reserved in PostgreSQL
    id                 uuid        NOT NULL,
    barbershop_id      uuid        NULL,      -- no FK: barbershop domain
    full_name          text        NOT NULL,
    email              text        NOT NULL,
    password_hash      text        NOT NULL,
    phone              text        NULL,
    profile_photo_url  text        NULL,
    role               text        NOT NULL,
    is_active          boolean     NOT NULL DEFAULT true,
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_app_user PRIMARY KEY (id),
    CONSTRAINT chk_app_user_full_name CHECK (char_length(full_name) BETWEEN 1 AND 120),
    CONSTRAINT chk_app_user_email     CHECK (char_length(email) <= 150),
    CONSTRAINT chk_app_user_phone     CHECK (char_length(phone) <= 20),
    CONSTRAINT chk_app_user_role      CHECK (role IN ('SUPER_ADMIN','ADMIN_BARBERSHOP','BARBER','CLIENT')),
    CONSTRAINT chk_app_user_tenant    CHECK ((role IN ('ADMIN_BARBERSHOP','BARBER')) = (barbershop_id IS NOT NULL))
);
CREATE UNIQUE INDEX uq_app_user_email ON app_user (lower(email));
CREATE INDEX idx_app_user_barbershop_id ON app_user (barbershop_id);

CREATE TABLE refresh_token (
    id           uuid        NOT NULL,
    user_id      uuid        NOT NULL,
    token_hash   text        NOT NULL,
    expires_at   timestamptz NOT NULL,        -- issued + 7 days
    revoked_at   timestamptz NULL,            -- set on rotation or logout
    created_at   timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_refresh_token PRIMARY KEY (id),
    CONSTRAINT uq_refresh_token_hash UNIQUE (token_hash),
    CONSTRAINT fk_refresh_token_user FOREIGN KEY (user_id) REFERENCES app_user (id) ON DELETE CASCADE
);
CREATE INDEX idx_refresh_token_user_id ON refresh_token (user_id);

CREATE TABLE password_reset_token (
    id           uuid        NOT NULL,
    user_id      uuid        NOT NULL,
    code_hash    text        NOT NULL,        -- 6-digit code, stored hashed
    expires_at   timestamptz NOT NULL,        -- issued + 15 minutes
    used_at      timestamptz NULL,            -- single use
    created_at   timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_password_reset_token PRIMARY KEY (id),
    CONSTRAINT fk_password_reset_token_user FOREIGN KEY (user_id) REFERENCES app_user (id) ON DELETE CASCADE
);
CREATE INDEX idx_password_reset_token_user_id ON password_reset_token (user_id);
```

- **One role per user** (`role`, not an array). `SUPER_ADMIN` and `CLIENT` have no
  `barbershop_id`: a client is platform-wide and visits several barbershops (how a `CLIENT`
  token is bound to one is OQ-07). `chk_app_user_tenant` enforces this in the database.
- `refresh_token` and `password_reset_token` existed in the prototype only as JPA entities; here
  they are explicit tables because rotation (single-use refresh) and the 15-minute code need
  stored state.

---

## 3. Barbershop — `barbershop`

```sql
CREATE TABLE barbershop (
    id                         uuid          NOT NULL,
    name                       text          NOT NULL,
    address                    text          NULL,
    city                       text          NOT NULL,
    latitude                   numeric(10,7) NULL,
    longitude                  numeric(10,7) NULL,
    phone                      text          NULL,
    whatsapp_number            text          NULL,
    logo_url                   text          NULL,
    status                     text          NOT NULL DEFAULT 'TRIAL',
    plan_id                    uuid          NULL,          -- no FK: platform-admin domain
    timezone                   text          NOT NULL DEFAULT 'America/Bogota',
    cancellation_policy_hours  integer       NOT NULL DEFAULT 2,
    trial_ends_at              timestamptz   NOT NULL,      -- created_at + 60 days, never updated
    created_at                 timestamptz   NOT NULL DEFAULT now(),
    updated_at                 timestamptz   NOT NULL DEFAULT now(),
    CONSTRAINT pk_barbershop PRIMARY KEY (id),
    CONSTRAINT chk_barbershop_name     CHECK (char_length(name) BETWEEN 1 AND 120),
    CONSTRAINT chk_barbershop_address  CHECK (char_length(address) <= 255),
    CONSTRAINT chk_barbershop_city     CHECK (char_length(city) BETWEEN 1 AND 80),
    CONSTRAINT chk_barbershop_latitude CHECK (latitude BETWEEN -90 AND 90),
    CONSTRAINT chk_barbershop_longitude CHECK (longitude BETWEEN -180 AND 180),
    CONSTRAINT chk_barbershop_phone    CHECK (char_length(phone) <= 20 AND char_length(whatsapp_number) <= 20),
    CONSTRAINT chk_barbershop_logo_url CHECK (char_length(logo_url) <= 255),
    CONSTRAINT chk_barbershop_status   CHECK (status IN ('TRIAL','ACTIVE','SUSPENDED','CANCELLED')),
    CONSTRAINT chk_barbershop_timezone CHECK (char_length(timezone) <= 50),
    CONSTRAINT chk_barbershop_cancellation_policy CHECK (cancellation_policy_hours >= 0)
);
CREATE INDEX idx_barbershop_city_status ON barbershop (city, status);
CREATE INDEX idx_barbershop_trial_ends_at ON barbershop (trial_ends_at) WHERE status = 'TRIAL';

CREATE TABLE service (
    id                uuid        NOT NULL,
    barbershop_id     uuid        NOT NULL,
    name              text        NOT NULL,
    description       text        NULL,
    duration_minutes  integer     NOT NULL,
    price_cents       bigint      NOT NULL,
    is_active         boolean     NOT NULL DEFAULT true,
    created_at        timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_service PRIMARY KEY (id),
    CONSTRAINT fk_service_barbershop FOREIGN KEY (barbershop_id) REFERENCES barbershop (id) ON DELETE CASCADE,
    CONSTRAINT chk_service_name        CHECK (char_length(name) BETWEEN 1 AND 100),
    CONSTRAINT chk_service_description CHECK (char_length(description) <= 255),
    CONSTRAINT chk_service_duration    CHECK (duration_minutes >= 5),
    CONSTRAINT chk_service_price       CHECK (price_cents >= 0)
);
CREATE INDEX idx_service_barbershop_id ON service (barbershop_id);

CREATE TABLE barber_profile (
    id                uuid         NOT NULL,
    barbershop_id     uuid         NOT NULL,
    user_id           uuid         NOT NULL,   -- no FK: identity-auth domain
    experience_years  integer      NOT NULL DEFAULT 0,
    bio               text         NULL,
    rating_avg        numeric(3,2) NOT NULL DEFAULT 0,
    rating_count      integer      NOT NULL DEFAULT 0,
    CONSTRAINT pk_barber_profile PRIMARY KEY (id),
    CONSTRAINT uq_barber_profile_user UNIQUE (user_id),
    CONSTRAINT fk_barber_profile_barbershop FOREIGN KEY (barbershop_id) REFERENCES barbershop (id) ON DELETE CASCADE,
    CONSTRAINT chk_barber_profile_experience CHECK (experience_years >= 0),
    CONSTRAINT chk_barber_profile_bio        CHECK (char_length(bio) <= 500),
    CONSTRAINT chk_barber_profile_rating     CHECK (rating_avg BETWEEN 0 AND 5 AND rating_count >= 0)
);
CREATE INDEX idx_barber_profile_barbershop_id ON barber_profile (barbershop_id);

CREATE TABLE barber_specialty (
    id                 uuid NOT NULL,
    barber_profile_id  uuid NOT NULL,
    specialty_name     text NOT NULL,
    CONSTRAINT pk_barber_specialty PRIMARY KEY (id),
    CONSTRAINT fk_barber_specialty_profile FOREIGN KEY (barber_profile_id) REFERENCES barber_profile (id) ON DELETE CASCADE,
    CONSTRAINT chk_barber_specialty_name CHECK (char_length(specialty_name) BETWEEN 1 AND 80)
);
CREATE INDEX idx_barber_specialty_profile_id ON barber_specialty (barber_profile_id);
```

- `trial_ends_at` is **stored** (closes OQ-11): INV-SHOP-001 fixes it once at registration and
  the worker's trial-expiration job (FR-026) queries it through `idx_barbershop_trial_ends_at`.
- `status` and `plan_id` are changed by platform-admin through barbershop's API, never by
  writing this database (OQ-10).
- The barber's name and photo live in `identity_auth.app_user`; `barber_profile` keeps only
  `user_id` (OQ-08).

---

## 4. Schedule — `schedule`

```sql
CREATE TABLE barber_schedule (
    id                 uuid    NOT NULL,
    barbershop_id      uuid    NOT NULL,
    barber_profile_id  uuid    NOT NULL,       -- no FK: barbershop domain
    day_of_week        smallint NOT NULL,      -- 0 = Sunday … 6 = Saturday
    start_time         time    NOT NULL,
    end_time           time    NOT NULL,
    is_active          boolean NOT NULL DEFAULT true,
    CONSTRAINT pk_barber_schedule PRIMARY KEY (id),
    CONSTRAINT chk_barber_schedule_day  CHECK (day_of_week BETWEEN 0 AND 6),
    CONSTRAINT chk_barber_schedule_time CHECK (end_time > start_time)
);
CREATE INDEX idx_barber_schedule_barber_day ON barber_schedule (barber_profile_id, day_of_week);
CREATE INDEX idx_barber_schedule_barbershop_id ON barber_schedule (barbershop_id);

CREATE TABLE schedule_exception (
    id                 uuid    NOT NULL,
    barbershop_id      uuid    NOT NULL,
    barber_profile_id  uuid    NOT NULL,       -- no FK: barbershop domain
    exception_date     date    NOT NULL,
    is_day_off         boolean NOT NULL DEFAULT true,
    start_time         time    NULL,
    end_time           time    NULL,
    reason             text    NULL,
    CONSTRAINT pk_schedule_exception PRIMARY KEY (id),
    CONSTRAINT uq_schedule_exception_barber_date UNIQUE (barber_profile_id, exception_date),
    CONSTRAINT chk_schedule_exception_reason CHECK (char_length(reason) <= 150),
    CONSTRAINT chk_schedule_exception_hours  CHECK (
        (is_day_off AND start_time IS NULL AND end_time IS NULL)
        OR (NOT is_day_off AND start_time IS NOT NULL AND end_time > start_time))
);
CREATE INDEX idx_schedule_exception_barbershop_id ON schedule_exception (barbershop_id);
```

- `barbershop_id` is new on both tables: the prototype reached the tenant through a join to
  `barber_profiles`, which now lives in another database.
- `uq_schedule_exception_barber_date` keeps AGGR-INV-BARBER-002 (one exception per barber per
  date, overriding the weekly schedule). `chk_schedule_exception_hours` moves the "custom hours
  need both times" rule from application code into the database.
- `/api/v1/availability` is computed, not stored (how schedule learns about bookings is OQ-09).

---

## 5. Appointment — `appointment`

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE appointment (
    id                      uuid        NOT NULL,
    barbershop_id           uuid        NOT NULL,
    client_id               uuid        NULL,     -- NULL = walk-in created by staff
    barber_id               uuid        NOT NULL, -- barber_profile id, no FK
    service_id              uuid        NOT NULL, -- no FK: barbershop domain
    appointment_date        date        NOT NULL,
    start_time              time        NOT NULL,
    end_time                time        NOT NULL,
    status                  text        NOT NULL DEFAULT 'PENDING',
    price_at_booking_cents  bigint      NOT NULL,
    notes                   text        NULL,
    cancelled_reason        text        NULL,
    created_by              uuid        NOT NULL, -- user who booked (client or staff)
    created_at              timestamptz NOT NULL DEFAULT now(),
    updated_at              timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_appointment PRIMARY KEY (id),
    CONSTRAINT chk_appointment_status CHECK (status IN ('PENDING','CONFIRMED','IN_PROGRESS','COMPLETED','CANCELLED','NO_SHOW')),
    CONSTRAINT chk_appointment_time   CHECK (end_time > start_time),
    CONSTRAINT chk_appointment_price  CHECK (price_at_booking_cents >= 0),
    CONSTRAINT chk_appointment_notes  CHECK (char_length(notes) <= 500),
    CONSTRAINT chk_appointment_cancelled_reason CHECK (char_length(cancelled_reason) <= 255),
    CONSTRAINT ex_appointment_no_double_booking EXCLUDE USING gist (
        barber_id WITH =,
        tsrange(appointment_date + start_time, appointment_date + end_time) WITH &&
    ) WHERE (status NOT IN ('CANCELLED','NO_SHOW'))
);
CREATE INDEX idx_appointment_barbershop_date ON appointment (barbershop_id, appointment_date);
CREATE INDEX idx_appointment_client_id ON appointment (client_id);
CREATE INDEX idx_appointment_status ON appointment (status);
```

- **Walk-in:** `client_id` is nullable, as `appointment-service.yaml` declares (the prototype's
  `NOT NULL` blocked F-13). Only staff (`BARBER`, `ADMIN_BARBERSHOP`) may create an appointment
  with no client; the service checks the role of `created_by`, which is always set.
- **No double booking (INV-APPT-001)** is enforced by the database: two active appointments of
  the same barber cannot overlap in time. The prototype relied on a pessimistic lock in code;
  the lock remains useful to return a clean `422`, but the constraint is the guarantee.
- `price_at_booking_cents` is computed by the service from `service.price_cents` at booking
  time and never changes (INV-APPT-002). The contract still names it `priceAtBooking` as a
  `double`; it moves to `priceAtBookingCents` in the 07 alignment.

---

## 6. Loyalty — `loyalty`

```sql
CREATE TABLE loyalty_rewards_config (
    id                  uuid    NOT NULL,
    barbershop_id       uuid    NOT NULL,
    stickers_required   integer NOT NULL DEFAULT 10,
    reward_description  text    NOT NULL,
    is_active           boolean NOT NULL DEFAULT true,
    CONSTRAINT pk_loyalty_rewards_config PRIMARY KEY (id),
    CONSTRAINT uq_loyalty_rewards_config_barbershop UNIQUE (barbershop_id),
    CONSTRAINT chk_loyalty_rewards_config_stickers CHECK (stickers_required >= 1),
    CONSTRAINT chk_loyalty_rewards_config_description CHECK (char_length(reward_description) BETWEEN 1 AND 255)
);

CREATE TABLE loyalty_card (
    id                      uuid        NOT NULL,
    barbershop_id           uuid        NOT NULL,
    client_id               uuid        NOT NULL,   -- no FK: identity-auth domain
    stickers_count          integer     NOT NULL DEFAULT 0,
    total_rewards_redeemed  integer     NOT NULL DEFAULT 0,
    last_updated            timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_loyalty_card PRIMARY KEY (id),
    CONSTRAINT uq_loyalty_card_client_barbershop UNIQUE (client_id, barbershop_id),
    CONSTRAINT chk_loyalty_card_counts CHECK (stickers_count >= 0 AND total_rewards_redeemed >= 0)
);
CREATE INDEX idx_loyalty_card_barbershop_id ON loyalty_card (barbershop_id);

CREATE TABLE loyalty_transaction (
    id                  uuid        NOT NULL,
    loyalty_card_id     uuid        NOT NULL,
    appointment_id      uuid        NULL,       -- no FK: appointment domain
    type                text        NOT NULL,
    granted_by_user_id  uuid        NOT NULL,   -- no FK: identity-auth domain
    created_at          timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_loyalty_transaction PRIMARY KEY (id),
    CONSTRAINT fk_loyalty_transaction_card FOREIGN KEY (loyalty_card_id) REFERENCES loyalty_card (id) ON DELETE CASCADE,
    CONSTRAINT chk_loyalty_transaction_type CHECK (type IN ('STICKER_EARNED','REWARD_REDEEMED'))
);
CREATE INDEX idx_loyalty_transaction_card_id ON loyalty_transaction (loyalty_card_id);
CREATE UNIQUE INDEX uq_loyalty_transaction_sticker_per_appointment
    ON loyalty_transaction (appointment_id) WHERE type = 'STICKER_EARNED';

CREATE TABLE reward_coupon (
    id              uuid        NOT NULL,
    barbershop_id   uuid        NOT NULL,
    client_id       uuid        NOT NULL,
    status          text        NOT NULL DEFAULT 'ACTIVE',
    appointment_id  uuid        NULL,           -- set when the coupon pays an appointment
    created_at      timestamptz NOT NULL DEFAULT now(),
    used_at         timestamptz NULL,
    CONSTRAINT pk_reward_coupon PRIMARY KEY (id),
    CONSTRAINT chk_reward_coupon_status CHECK (status IN ('ACTIVE','USED')),
    CONSTRAINT chk_reward_coupon_used   CHECK ((status = 'USED') = (used_at IS NOT NULL))
);
CREATE INDEX idx_reward_coupon_client_barbershop_status ON reward_coupon (client_id, barbershop_id, status);
```

- `loyalty_transaction` stays the append-only log behind `StickerGranted` / `RewardRedeemed`.
  `uq_loyalty_transaction_sticker_per_appointment` makes "one sticker per completed
  appointment" a database rule, which matters now that the sticker arrives by event and can be
  delivered twice.
- `uq_loyalty_rewards_config_barbershop`: one active reward rule per barbershop, as the
  contract exposes a single `/loyalty/config`.

---

## 7. Notifications — `notifications` (MongoDB)

Collections follow annex B and ADR-006: `$jsonSchema` validator with
`additionalProperties: false`, `validationLevel: strict`, `validationAction: error`.

**`notification`**

| Field | BSON type | Rule |
|---|---|---|
| `_id` | string (UUID) | required |
| `userId` | string (UUID) | required; referenced, not embedded |
| `barbershopId` | string (UUID) or null | tenant, when the notification belongs to one |
| `title` | string | required, ≤ 150 |
| `body` | string | required, ≤ 500 |
| `type` | string | required, one of `APPOINTMENT_CONFIRMATION`, `REMINDER`, `PROMOTION`, `SYSTEM` |
| `read` | bool | required, default `false` |
| `sourceEventId` | string (UUID) or null | the event that produced it; unique when present |
| `deliveryAttempts` | array, `maxItems: 10` | embedded `{channel: PUSH\|EMAIL, status: SENT\|FAILED, attemptedAt, errorCode?}` |
| `createdAt`, `updatedAt` | date | required |
| `createdBy` | string (UUID) or null | null when created by an event |

Indexes: `idx_notification_user_read` on `{userId: 1, read: 1, createdAt: -1}`;
`uq_notification_source_event` unique on `sourceEventId` (partial, when it exists), so a
redelivered event does not notify twice.

**`device_token`**

| Field | BSON type | Rule |
|---|---|---|
| `_id` | string (UUID) | required |
| `userId` | string (UUID) | required |
| `token` | string | required, FCM token |
| `platform` | string | required, `ANDROID` or `IOS` |
| `createdAt`, `updatedAt` | date | required |

Index: `uq_device_token_token` unique on `token` (re-registering the same device updates it).

- The prototype had no `device_tokens` table in `init.sql` (it existed only as a JPA entity);
  here it is a declared collection backing `POST /device-tokens`.
- The `type` values are the prototype's four. The contract still describes `type` as free text;
  it is fixed to this enum in the 07 alignment.

---

## 8. Finance & Inventory — `finance_inventory`

```sql
CREATE TABLE finance_record (
    id                      uuid        NOT NULL,
    barbershop_id           uuid        NOT NULL,
    type                    text        NOT NULL,
    category                text        NOT NULL,
    amount_cents            bigint      NOT NULL,
    description             text        NULL,
    record_date             date        NOT NULL,
    related_appointment_id  uuid        NULL,   -- no FK: appointment domain
    created_at              timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_finance_record PRIMARY KEY (id),
    CONSTRAINT chk_finance_record_type        CHECK (type IN ('INCOME','EXPENSE')),
    CONSTRAINT chk_finance_record_category    CHECK (char_length(category) BETWEEN 1 AND 80),
    CONSTRAINT chk_finance_record_amount      CHECK (amount_cents > 0),
    CONSTRAINT chk_finance_record_description CHECK (char_length(description) <= 255)
);
CREATE INDEX idx_finance_record_barbershop_date ON finance_record (barbershop_id, record_date);

CREATE TABLE inventory_product (
    id               uuid          NOT NULL,
    barbershop_id    uuid          NOT NULL,
    name             text          NOT NULL,
    description      text          NULL,
    unit             text          NOT NULL DEFAULT 'unidad',
    current_stock    numeric(12,2) NOT NULL DEFAULT 0,
    min_stock_alert  numeric(12,2) NOT NULL DEFAULT 0,
    created_at       timestamptz   NOT NULL DEFAULT now(),
    CONSTRAINT pk_inventory_product PRIMARY KEY (id),
    CONSTRAINT chk_inventory_product_name  CHECK (char_length(name) BETWEEN 1 AND 120),
    CONSTRAINT chk_inventory_product_description CHECK (char_length(description) <= 255),
    CONSTRAINT chk_inventory_product_unit  CHECK (char_length(unit) BETWEEN 1 AND 20),
    CONSTRAINT chk_inventory_product_stock CHECK (current_stock >= 0 AND min_stock_alert >= 0)
);
CREATE INDEX idx_inventory_product_barbershop_id ON inventory_product (barbershop_id);

CREATE TABLE inventory_movement (
    id                  uuid          NOT NULL,
    product_id          uuid          NOT NULL,
    movement_type       text          NOT NULL,
    quantity            numeric(12,2) NOT NULL,
    reason              text          NULL,
    created_by_user_id  uuid          NOT NULL,  -- no FK: identity-auth domain
    created_at          timestamptz   NOT NULL DEFAULT now(),
    CONSTRAINT pk_inventory_movement PRIMARY KEY (id),
    CONSTRAINT fk_inventory_movement_product FOREIGN KEY (product_id) REFERENCES inventory_product (id) ON DELETE CASCADE,
    CONSTRAINT chk_inventory_movement_type     CHECK (movement_type IN ('IN','OUT')),
    CONSTRAINT chk_inventory_movement_quantity CHECK (quantity > 0),
    CONSTRAINT chk_inventory_movement_reason   CHECK (char_length(reason) <= 255)
);
CREATE INDEX idx_inventory_movement_product_created ON inventory_movement (product_id, created_at);
```

- `chk_finance_record_amount` closes the prototype gap where "amount must be positive" lived
  only in application code (FR-017).
- Stock is a quantity, not money: `numeric(12,2)` (millilitres, units), never floating point.
  `lowStock` in the contract is computed (`current_stock <= min_stock_alert`), not stored.
- `inventory_movement` reaches the tenant through its product; `chk_inventory_product_stock`
  rejects an `OUT` movement that would leave negative stock.

---

## 9. Platform Admin — `platform_admin`

```sql
CREATE TABLE subscription_plan (
    id             uuid        NOT NULL,
    name           text        NOT NULL,
    price_cents    bigint      NOT NULL,
    max_barbers    integer     NOT NULL,
    features_json  jsonb       NULL,     -- exposed by the contract as a JSON string
    is_active      boolean     NOT NULL DEFAULT true,
    created_at     timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_subscription_plan PRIMARY KEY (id),
    CONSTRAINT uq_subscription_plan_name UNIQUE (name),
    CONSTRAINT chk_subscription_plan_name  CHECK (char_length(name) BETWEEN 1 AND 50),
    CONSTRAINT chk_subscription_plan_price CHECK (price_cents >= 0),
    CONSTRAINT chk_subscription_plan_max_barbers CHECK (max_barbers >= 1)
);
```

**Seed (idempotent upsert on `name`), converted from the prototype:**

| name | price_cents | max_barbers |
|---|---|---|
| Basico | 4990000 | 2 |
| Pro | 9990000 | 6 |
| Premium | 17990000 | 999 |

> These names and prices come from the prototype's seed; `01-context/overview_en.md` lists
> different ones (Starter/Profesional/Premium). The team still has to confirm which is current.

platform-admin owns no barbershop rows: `/api/v1/platform/barbershops` reads and changes them
through barbershop's API (OQ-10).

---

## 10. Tables every creating domain repeats

```sql
CREATE TABLE idempotency_key (
    key            text        NOT NULL,     -- Idempotency-Key header, 8–128 chars
    operation      text        NOT NULL,     -- e.g. 'POST /api/v1/appointments'
    resource_id    uuid        NOT NULL,     -- the resource created with this key
    request_hash   text        NOT NULL,     -- same key + different body → 422
    created_at     timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_idempotency_key PRIMARY KEY (key, operation),
    CONSTRAINT chk_idempotency_key_length CHECK (char_length(key) BETWEEN 8 AND 128)
);

CREATE TABLE outbox_event (                  -- identity-auth, appointment, loyalty
    id              uuid        NOT NULL,
    aggregate_type  text        NOT NULL,     -- 'appointment', 'loyalty_card', 'app_user'
    aggregate_id    uuid        NOT NULL,
    event_type      text        NOT NULL,     -- e.g. 'AppointmentCompleted'
    payload         jsonb       NOT NULL,
    correlation_id  text        NOT NULL,
    occurred_at     timestamptz NOT NULL DEFAULT now(),
    published_at    timestamptz NULL,
    CONSTRAINT pk_outbox_event PRIMARY KEY (id)
);
CREATE INDEX idx_outbox_event_unpublished ON outbox_event (occurred_at) WHERE published_at IS NULL;
```

- The resource and its `idempotency_key` row are written in **one transaction** (norm 5.3.8).
- The change and its `outbox_event` row are written in **one transaction** (norm 5.3.11);
  publishing is done afterwards, by a separate process (transport pending, AT-004 in
  `05-architecture/overview.md`). Events per domain: `02-domain/domain-events.md`.
- In MongoDB (notifications) both are collections with the same fields.

---

## 11. Prototype tables not carried over

| Prototype table | Why not here |
|---|---|
| `reviews`, `promotions`, `client_favorites`, `gallery_images` | Not in `01-context/scope.md`, `02-domain/domain-map.md` or any contract. Their bounded context is a `02-domain` decision; until it is made, no `-db` owns them |

`rating_avg` / `rating_count` on `barber_profile` stay because the contract exposes them; they are
fed by reviews once that context is decided.

---

## Correlations

- Conventions (UUID, cents, `CHECK`, no cross-domain FK) → `ADR-010`
- Engine and migration tool per domain → `ADR-006`, `ADR-007`
- Resources these tables back → `07-api/contracts/openapi/`
- Invariants these constraints encode → `02-domain/entities-and-rules.md`
- Field-level meaning → `06-data/data-dictionary.md`
- Deployment of each instance → `05-architecture/deployment.md`
