# Technical Decisions — Auth Service

> **Framework example, not BarberSaaS's identity-auth.** BarberSaaS's authentication decisions
> are in `07-api/authentication.md` and `07-api/contracts/openapi/auth-service.yaml`
> (`DEC-AUTH-*`).

> Design decisions of the authentication service that complement the global ADRs.

---

## Decision: JWT signing algorithm — RS256 vs HS256

**Date:** [date]
**Context:** A JWT can be signed with HMAC-SHA256 (HS256, shared symmetric key) or
RSA-SHA256 (RS256, public/private key pair).

**Decision:** RS256 — private key only in auth-service, public key available at
`GET /api/v1/auth/jwks` so any service can verify tokens without calling auth-service.

**Consequences:**
- Each service can verify tokens independently (no network latency)
- Key rotation is operationally more complex (the new public key must be distributed)
- The JWKS endpoint allows gradual rotation with several active keys at the same time

---

## Decision: Refresh tokens in the database vs. stateless

**Date:** [date]
**Context:** Refresh tokens can be stateless (long-lived JWT) or stateful (id in the DB).

**Decision:** Stateful — the refresh token is a UUID stored in the `refresh_tokens` table.
Only its SHA-256 hash is persisted (never the real value).

**Consequences:**
- Immediate revocation is possible (logout of a specific device, account suspension)
- Requires a PostgreSQL lookup on every use of the refresh token
- The refresh_tokens table can become a bottleneck with millions of active sessions

---

## Decision: Where per-resource permission checks live

**Date:** [date]
**Context:** Fine-grained permissions (e.g. "can only see their own appointments") can live
in auth-service or in each business service.

**Decision:** Auth-service manages roles (ADMIN, USER, VIEWER). Fine-grained per-resource
permissions are the responsibility of each business microservice.

**Reason:** Auth-service does not know each service's domain. Centralizing fine-grained
permissions would create a two-way coupling and make auth-service depend on all the others.

---

## Decision: Account lockout policy

**Date:** [date]
**Decision:** [N] failed attempts → locked for [M] minutes. The counter resets on a successful
login. The lock is stored in Redis (TTL = M minutes, self-cleaning).

**Consequences:**
- An attacker can DoS a specific user by forcing the lock
- Mitigation: gradual locking (5 min → 30 min → 24 hours) reduces the impact

---

## Correlations

- Global architecture ADRs → `05-architecture/decisions/records/`
- Technical security rules → `00-governance/security-rules.md`
- Data model → `data-model.md`
