# Migration Strategy — MySQL → PostgreSQL

> This is the concrete, scheduled migration for BarberSaaS (AT-001 in
> `05-architecture/overview.md`, Phase 2 / Q4 2026 per `03-product/vision.md`), not a
> generic migration-tooling guide. Every item below is grounded in the actual
> `application.yml` config and `db/init.sql` script — read in full for this document.

---

## Current state (verified, not assumed)

| Aspect | Value | Source |
|--------|-------|--------|
| Engine | MySQL 8.x | `db/init.sql` header comment |
| JDBC driver | `com.mysql.cj.jdbc.Driver` | `application.yml` |
| Hibernate dialect | `org.hibernate.dialect.MySQLDialect` (hardcoded, not profile-specific) | `application.yml` |
| Schema creation | **Manual SQL script** (`db/init.sql`), not Hibernate auto-generation | `ddl-auto: validate` in both `dev` and `prod`... |
| ...except | `application-dev.yml` overrides to `ddl-auto: update` | Hibernate *can* alter dev schemas automatically; production never does |
| Migration tool | **None** — no Flyway/Liquibase dependency in `pom.xml` | Verified — schema changes today mean hand-editing `init.sql` |

**Why this matters for the migration plan:** since `ddl-auto: validate` in production, this
project already treats a hand-written SQL script as its real schema source of truth, not
Hibernate auto-DDL. The PostgreSQL migration should produce an equivalent hand-written
script, not rely on pointing Hibernate at an empty Postgres DB and hoping `update` produces
the right thing — that mode isn't even used in prod.

---

## Why this migration is low-risk right now

There is **no production data** to migrate yet — Phase 1 (Private Beta) runs on MySQL with
seed/test data only (per `01-context/overview_en.md`, production hasn't launched). This means
the migration can be a **clean cutover** (recreate schema + re-seed on PostgreSQL, or a
one-time data dump/reload if beta data needs to carry over) rather than a live, zero-downtime
migration with dual-write or replication. Treat this as the easy version of this problem —
if it slips past the first real paying barbershops going live, it becomes much harder.

---

## MySQL → PostgreSQL: construct-by-construct translation

Every MySQL-specific construct actually used in `db/init.sql`, and its PostgreSQL
equivalent:

| MySQL construct (in `init.sql`) | PostgreSQL equivalent | Notes |
|----------------------------------|------------------------|-------|
| `BIGINT AUTO_INCREMENT PRIMARY KEY` | `BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` | SQL-standard identity column (Postgres 10+). Avoid legacy `BIGSERIAL` — `GENERATED ALWAYS AS IDENTITY` is the modern, more portable choice |
| `ENUM('A','B',...)` inline column type | `VARCHAR(n) NOT NULL CHECK (col IN ('A','B',...))` — **recommended**, not a native Postgres `CREATE TYPE ... AS ENUM` | Every entity already uses `@Enumerated(EnumType.STRING)` (verified in `Appointment.java`, `Barbershop.java`, `FinanceRecord.java`, etc.) — Hibernate is *already* reading/writing these columns as plain strings. A native Postgres `ENUM` type adds ALTER-TYPE friction (adding a value requires `ALTER TYPE ... ADD VALUE`, which can't run inside a transaction in older Postgres versions) for no benefit here, since the app never relied on native enum semantics to begin with |
| `JSON` (on `subscription_plans.features_json`) | `JSONB` | Postgres's `JSONB` is indexable and generally preferred over plain `JSON`; MySQL's `JSON` type is closer in behavior to `JSONB` (validated, not preserving exact formatting) than to Postgres's plain `JSON` |
| `TIMESTAMP DEFAULT CURRENT_TIMESTAMP` | `TIMESTAMPTZ DEFAULT NOW()` | See "Timestamps and timezone" below — this is the one place where "just translate the syntax" isn't quite enough |
| `TIMESTAMP ... ON UPDATE CURRENT_TIMESTAMP` | **Drop it — do not replicate with a trigger** | Every entity with an `updated_at` column already has Hibernate's `@UpdateTimestamp` annotation (verified on `Appointment`, `Barbershop`, etc.), which sets the value from the application on every `UPDATE` through JPA. A DB-level trigger would be redundant. Keep it simple: `TIMESTAMPTZ` column, no default-on-update clause, let Hibernate manage it — exactly as it already effectively does today (MySQL's `ON UPDATE` clause is likely never even exercised, since JPA sets the value before the `UPDATE` statement reaches the DB) |
| `TINYINT` (`barber_schedules.day_of_week`, `reviews.rating`) | `SMALLINT` | Postgres has no `TINYINT`; `SMALLINT` (2 bytes) is the standard substitute, more than sufficient for a 0–6 day-of-week or 1–5 rating |
| `ENGINE=InnoDB` | *(remove entirely)* | Postgres has no pluggable storage engines — this clause has no equivalent and isn't needed |
| `CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci` (on `CREATE DATABASE`) | `CREATE DATABASE barbersaas ENCODING 'UTF8' LC_COLLATE 'en_US.UTF-8' LC_CTYPE 'en_US.UTF-8'` (or the server's default UTF-8 locale) | Postgres databases are UTF-8 by default in most modern installs; verify Railway's default Postgres image before assuming this needs to be explicit |
| `CHECK (end_time > start_time)`, `CHECK (rating BETWEEN 1 AND 5)` | Identical syntax | Postgres `CHECK` constraints use the same syntax — no change needed |
| `DECIMAL(10,2)` | Identical (`NUMERIC(10,2)` is the same type, `DECIMAL` is an accepted alias in Postgres too) | No change needed |
| `FOREIGN KEY ... ON DELETE CASCADE` / `ON DELETE SET NULL` | Identical syntax | No change needed |
| `UNIQUE KEY name (cols)` | `CONSTRAINT name UNIQUE (cols)` or a separate `CREATE UNIQUE INDEX` | Minor syntax difference, same guarantee |

---

## Timestamps and timezone — the one non-mechanical decision

`06-data/data-dictionary.md` already flags that `appointments.appointment_date`/
`start_time`/`end_time` are plain `DATE`/`TIME` with no timezone, and correctness depends on
always interpreting them against `barbershops.timezone` (`'America/Bogota'` by default) in
application code. **This migration does not fix that** — translating `TIME` → `TIME` doesn't
change the underlying design risk. If this gets addressed, it should be its own decision
(possibly an ADR), not bundled silently into the engine migration. Recommendation: treat it
as a separate backlog item, not a blocker for the MySQL → PostgreSQL cutover itself.

For `created_at`/`updated_at`: switch from `TIMESTAMP` to `TIMESTAMPTZ`. MySQL's plain
`TIMESTAMP` type is already timezone-aware in a specific way (stored as UTC internally,
converted using the connection's `serverTimezone` — set to `America/Bogota` in the current
JDBC URL). Postgres's `TIMESTAMPTZ` is the closer equivalent; a plain Postgres `TIMESTAMP`
(without time zone) would silently drop that behavior.

---

## Application-side changes required (beyond the SQL script)

| Change | File | Current value | New value |
|--------|------|---------------|-----------|
| JDBC driver dependency | `pom.xml` | `mysql-connector-j` | `org.postgresql:postgresql` |
| Datasource URL | `application.yml` | `jdbc:mysql://...` | `jdbc:postgresql://...` |
| Driver class | `application.yml` | `com.mysql.cj.jdbc.Driver` | `org.postgresql.Driver` |
| Hibernate dialect | `application.yml` | `org.hibernate.dialect.MySQLDialect` | `org.hibernate.dialect.PostgreSQLDialect` |
| `docker-compose.yml` | `barbersaas-backend/` | MySQL service definition | PostgreSQL service definition (base image, port 5432, volume) |

None of the JPA entity classes themselves (`@Enumerated(EnumType.STRING)`,
`@CreationTimestamp`, `@UpdateTimestamp`, `@GeneratedValue(strategy = IDENTITY)`) need to
change — they were already written in an engine-agnostic way. `GenerationType.IDENTITY`
maps cleanly to Postgres's `GENERATED ALWAYS AS IDENTITY` the same way it mapped to MySQL's
`AUTO_INCREMENT`.

---

## Rollout plan

1. **Write the PostgreSQL equivalent of `init.sql`** using the translation table above —
   produces `db/init-postgres.sql` (or convert in place once MySQL is fully retired; keep
   both during the transition so `docker-compose.yml` can still spin up the old environment
   if needed for comparison).
2. **Switch local/dev first** (`application-dev.yml` already tolerates `ddl-auto: update`,
   giving a safety net while validating the translated schema against real entity mappings).
3. **Run the full test suite** — this is also the forcing function for
   `04-requirements/traceability-matrix.md`'s biggest gap: there are currently **zero**
   automated tests, so there is no regression safety net for this migration today. Writing
   at least integration tests for the highest-risk paths (FR-008 double-booking lock,
   FR-027/028 tenant isolation) before or during this migration is strongly recommended —
   a lock-behavior bug introduced by a dialect change would otherwise go undetected.
4. **Validate `PESSIMISTIC_WRITE` locking behavior specifically** — lock semantics are one
   of the few areas where MySQL (InnoDB) and PostgreSQL genuinely differ under the hood
   (MVCC implementation, gap-locking behavior). The anti-double-booking mechanism
   (INV-APPT-001) is the single most business-critical piece of logic in this system — treat
   its behavior under concurrent load on Postgres as something to explicitly verify, not
   assume carries over identically.
5. **Staging cutover**, then **production** — per the existing environment plan in
   `01-context/scope.md`.

---

## Correlations

- Schema being migrated → `06-data/models.md`
- Field-level semantics to preserve → `06-data/data-dictionary.md`
- Tracked as technical debt AT-001 → `05-architecture/overview.md`
- Untested paths this migration should not regress → `04-requirements/traceability-matrix.md`
