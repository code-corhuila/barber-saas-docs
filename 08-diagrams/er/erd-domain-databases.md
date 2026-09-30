# ERD-01…08 · One Entity-Relationship Diagram per Domain Database

> **Type:** ER · **Derived from:** `06-data/models.md` §1–10 · **Decisions:** ADR-004, ADR-006,
> ADR-010 · **Rule:** norm 7.1–7.4

Each diagram is one database. Lines are real foreign keys **inside** that database. A column
marked `ref <domain>` points to another domain: it is a UUID with **no foreign key**, checked
through that domain's contract (norm 7.4). Every creating domain also owns `idempotency_key`,
and identity-auth, appointment and loyalty own `outbox_event` (§ERD-09); they are not repeated.

## ERD-01 · identity-auth-db (PostgreSQL, schema `identity_auth`)

```mermaid
erDiagram
  app_user ||--o{ refresh_token : "has"
  app_user ||--o{ password_reset_token : "has"
  app_user {
    uuid id PK
    uuid barbershop_id "ref barbershop · only ADMIN_BARBERSHOP, BARBER"
    text full_name
    text email UK "unique on lower(email)"
    text password_hash "BCrypt"
    text role "SUPER_ADMIN | ADMIN_BARBERSHOP | BARBER | CLIENT"
    boolean is_active
    timestamptz created_at
  }
  refresh_token {
    uuid id PK
    uuid user_id FK
    text token_hash UK
    timestamptz expires_at "+7 days"
    timestamptz revoked_at
  }
  password_reset_token {
    uuid id PK
    uuid user_id FK
    text code_hash "6 digits"
    timestamptz expires_at "+15 minutes"
    timestamptz used_at "single use"
  }
```

## ERD-02 · barbershop-db (PostgreSQL, schema `barbershop`)

```mermaid
erDiagram
  barbershop ||--o{ service : "offers"
  barbershop ||--o{ barber_profile : "employs"
  barber_profile ||--o{ barber_specialty : "has"
  barbershop {
    uuid id PK
    text name
    text city
    text status "TRIAL | ACTIVE | SUSPENDED | CANCELLED"
    uuid plan_id "ref platform-admin"
    text timezone "America/Bogota"
    integer cancellation_policy_hours
    timestamptz trial_ends_at "set once"
  }
  service {
    uuid id PK
    uuid barbershop_id FK
    text name
    integer duration_minutes ">= 5"
    bigint price_cents ">= 0"
    boolean is_active
  }
  barber_profile {
    uuid id PK
    uuid barbershop_id FK
    uuid user_id UK "ref identity-auth"
    integer experience_years
    numeric rating_avg "0..5"
  }
  barber_specialty {
    uuid id PK
    uuid barber_profile_id FK
    text specialty_name
  }
```

## ERD-03 · schedule-db (PostgreSQL, schema `schedule`)

```mermaid
erDiagram
  barber_schedule {
    uuid id PK
    uuid barbershop_id "ref barbershop (tenant)"
    uuid barber_profile_id "ref barbershop"
    smallint day_of_week "0 Sunday .. 6 Saturday"
    time start_time
    time end_time "> start_time"
    boolean is_active
  }
  schedule_exception {
    uuid id PK
    uuid barbershop_id "ref barbershop (tenant)"
    uuid barber_profile_id "ref barbershop · unique with date"
    date exception_date
    boolean is_day_off
    time start_time "only when not day off"
    time end_time
  }
```

No foreign key inside this database: both tables hang from a barber that lives in
barbershop-db. Availability is computed, not stored.

## ERD-04 · appointment-db (PostgreSQL, schema `appointment`)

```mermaid
erDiagram
  appointment {
    uuid id PK
    uuid barbershop_id "ref barbershop (tenant)"
    uuid client_id "ref identity-auth · null = walk-in"
    uuid barber_id "ref barbershop (barber_profile)"
    uuid service_id "ref barbershop"
    date appointment_date
    time start_time
    time end_time
    text status "PENDING .. COMPLETED | CANCELLED | NO_SHOW"
    bigint price_at_booking_cents "snapshot"
    uuid created_by "ref identity-auth"
  }
```

`ex_appointment_no_double_booking`: an exclusion constraint on `(barber_id, time range)` for
every status except `CANCELLED` and `NO_SHOW` — the database guarantee behind INV-APPT-001.

## ERD-05 · loyalty-db (PostgreSQL, schema `loyalty`)

```mermaid
erDiagram
  loyalty_card ||--o{ loyalty_transaction : "logs"
  loyalty_rewards_config {
    uuid id PK
    uuid barbershop_id UK "ref barbershop · one rule per shop"
    integer stickers_required ">= 1"
    text reward_description
    boolean is_active
  }
  loyalty_card {
    uuid id PK
    uuid barbershop_id "ref barbershop"
    uuid client_id "ref identity-auth · unique with barbershop"
    integer stickers_count
    integer total_rewards_redeemed
  }
  loyalty_transaction {
    uuid id PK
    uuid loyalty_card_id FK
    uuid appointment_id "ref appointment · one sticker per appointment"
    text type "STICKER_EARNED | REWARD_REDEEMED"
    uuid granted_by_user_id "ref identity-auth"
  }
  reward_coupon {
    uuid id PK
    uuid barbershop_id "ref barbershop"
    uuid client_id "ref identity-auth"
    text status "ACTIVE | USED"
    uuid appointment_id "ref appointment · set when used"
    timestamptz used_at
  }
```

## ERD-06 · notifications-db (MongoDB rs0, database `notifications`)

```mermaid
erDiagram
  notification ||--o{ delivery_attempt : "embeds (max 10)"
  notification {
    string _id PK "UUID"
    string userId "ref identity-auth"
    string barbershopId "ref barbershop · nullable"
    string title
    string body
    string type "APPOINTMENT_CONFIRMATION | REMINDER | PROMOTION | SYSTEM"
    bool read
    string sourceEventId UK "unique when present"
  }
  delivery_attempt {
    string channel "PUSH | EMAIL"
    string status "SENT | FAILED"
    date attemptedAt
  }
  device_token {
    string _id PK "UUID"
    string userId "ref identity-auth"
    string token UK "FCM token"
    string platform "ANDROID | IOS"
  }
```

Collections, not tables: `delivery_attempt` is an embedded array, not a collection. Validation
is `$jsonSchema` with `additionalProperties: false` (annex B).

## ERD-07 · finance-inventory-db (PostgreSQL, schema `finance_inventory`)

```mermaid
erDiagram
  inventory_product ||--o{ inventory_movement : "moves"
  finance_record {
    uuid id PK
    uuid barbershop_id "ref barbershop (tenant)"
    text type "INCOME | EXPENSE"
    text category
    bigint amount_cents "> 0"
    date record_date
    uuid related_appointment_id "ref appointment"
  }
  inventory_product {
    uuid id PK
    uuid barbershop_id "ref barbershop (tenant)"
    text name
    text unit
    numeric current_stock ">= 0"
    numeric min_stock_alert
  }
  inventory_movement {
    uuid id PK
    uuid product_id FK
    text movement_type "IN | OUT"
    numeric quantity "> 0"
    uuid created_by_user_id "ref identity-auth"
  }
```

## ERD-08 · platform-admin-db (PostgreSQL, schema `platform_admin`)

```mermaid
erDiagram
  subscription_plan {
    uuid id PK
    text name UK
    bigint price_cents
    integer max_barbers ">= 1"
    jsonb features_json
    boolean is_active
  }
```

platform-admin owns no barbershop rows; it changes them through barbershop-api (OQ-10).

## ERD-09 · Tables every domain repeats

```mermaid
erDiagram
  idempotency_key {
    text key PK "8..128 chars"
    text operation PK "e.g. POST /api/v1/appointments"
    uuid resource_id
    text request_hash "same key, other body = 422"
  }
  outbox_event {
    uuid id PK
    text aggregate_type
    uuid aggregate_id
    text event_type "e.g. AppointmentCompleted"
    jsonb payload
    text correlation_id
    timestamptz published_at "null = pending"
  }
```

Both are written in the **same transaction** as the resource they belong to (norm 5.3.8,
5.3.11). Their use across services is drawn in [SEQ-02](../uml/seq-book-appointment.md) and
[SEQ-03](../uml/seq-complete-appointment.md).
