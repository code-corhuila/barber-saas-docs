# Data Model — Auth Service

> **Framework example, not BarberSaaS's identity-auth.** This folder is the worked example
> seeded by the governance framework. BarberSaaS's real identity data model is
> `06-data/models.md` §2 (`app_user`, `refresh_token`, `password_reset_token`, one role per
> user, no Redis) and its contract is `07-api/contracts/openapi/auth-service.yaml`.

> The Auth Service is the **authoritative owner** of user identity.
> No other service reads these tables directly. If they need user data,
> they get it from the JWT or query it through the API.

---

## Database engine

**Primary engine:** PostgreSQL — for users, roles and refresh tokens.
**Cache engine:** Redis — for the token blacklist and failed-attempt counters.

**PostgreSQL rationale:** ACID transactions are critical: a user cannot exist without an
assigned role, and failed login attempt records cannot be lost.

**Redis rationale:** The token blacklist check happens on every request to the gateway.
Redis supports O(1) lookups with automatic TTL, with no cleanup jobs needed.

---

## PostgreSQL schema

### Table: `users`

**Purpose:** Authentication credentials. Only the data needed to authenticate — the application profile belongs to the corresponding business service.

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| id | UUID | No | Unique identifier | PK, gen_random_uuid() |
| email | VARCHAR(255) | No | User email | UNIQUE, NOT NULL |
| password_hash | VARCHAR(255) | No | bcrypt hash (cost 12) | NOT NULL |
| email_verified | BOOLEAN | No | Whether the email was verified | DEFAULT false |
| failed_attempts | INT | No | Failed login attempts | DEFAULT 0 |
| locked_until | TIMESTAMPTZ | Yes | Locked until this date | NULL = not locked |
| created_at | TIMESTAMPTZ | No | Registration date | DEFAULT NOW() |
| updated_at | TIMESTAMPTZ | No | Last modification | DEFAULT NOW() |
| deleted_at | TIMESTAMPTZ | Yes | Soft delete | NULL = active |

**Indexes:**
| Name | Fields | Rationale |
|------|--------|-----------|
| idx_users_email | email | Lookup by email at login (main lookup) |
| idx_users_deleted_at | deleted_at | Filter active users |

### Table: `roles`

**Purpose:** Catalog of the system's roles.

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| id | UUID | No | PK |
| name | VARCHAR(50) | No | Role name (UNIQUE): ADMIN, USER, VIEWER |
| description | TEXT | Yes | Human-readable role description |
| created_at | TIMESTAMPTZ | No | DEFAULT NOW() |

### Table: `user_roles`

**Purpose:** Many-to-many relation between users and roles.

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| user_id | UUID | No | FK → users.id ON DELETE CASCADE |
| role_id | UUID | No | FK → roles.id ON DELETE RESTRICT |
| assigned_at | TIMESTAMPTZ | No | When the role was assigned |
| assigned_by | UUID | Yes | FK → users.id — who assigned the role |

**Composite PK:** (user_id, role_id)

### Table: `refresh_tokens`

**Purpose:** Active refresh tokens. Allows rotation and revocation.

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| id | UUID | No | PK |
| user_id | UUID | No | FK → users.id ON DELETE CASCADE |
| token_hash | VARCHAR(255) | No | SHA-256 hash of the token (never the real token) |
| expires_at | TIMESTAMPTZ | No | Refresh token expiration |
| revoked_at | TIMESTAMPTZ | Yes | NULL = active, date = revoked |
| created_at | TIMESTAMPTZ | No | DEFAULT NOW() |
| user_agent | TEXT | Yes | To show active sessions to the user |

---

## Redis schema

| Key pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `blacklist:{jti}` | String | Until the JWT expires | Revoked tokens (logout) |
| `attempts:{email}` | String (int) | 5 minutes | Failed attempt counter |
| `locked:{email}` | String | Until unlock | Temporarily locked email |

---

## Migrations

**Tool:** [Flyway / Alembic / golang-migrate — depending on the project's stack]
**Script location:** `src/migrations/` (see the guide in `_stacks/[your-stack].md`)

**Policy:** All migrations are forward-only in production. Data rollbacks are done with additional migrations, not by reverting scripts.

---

## Correlations

- Service runbook → `runbook.md`
- Events it emits → `events.md`
- Security and JWT policy → `00-governance/security-policy.md`
