# Security Policy

> Security is not a feature — it is a system property built from day one. This document
> defines the mandatory practices.
> Any deviation must be explicitly approved by the Tech Lead.

---

## Security principles

1. **Defense in Depth:** Multiple security layers. If one fails, the others contain the damage.
2. **Least Privilege:** Each component has only the minimum necessary permissions.
3. **Fail Secure:** In case of error, the system denies access, does not allow it.
4. **Security by Design:** Security controls are designed from the start, not added at the end.
5. **Zero Trust:** Always verify, never implicitly trust, even within the internal network.

---

## Authentication

### JWT (JSON Web Tokens)

| Property | Required value |
|----------|---------------|
| Signing algorithm | HS512 (HMAC-SHA, symmetric secret) — `JwtTokenProvider` uses `Keys.hmacShaKeyFor(jwtSecret.getBytes())`, which selects the HMAC algorithm from key length; the current secret (65 bytes / 520 bits) resolves to HS512. A shorter secret would silently downgrade to HS384/HS256 — keep any replacement secret at 64+ bytes |
| Access token expiration | 24 hours (`expiration-ms: 86400000` in `application.yml`) |
| Refresh token expiration | 7 days (`refresh-expiration-ms: 604800000` in `application.yml`) |
| Required claims | `sub` (userId), `iat`, `exp`; role and `barbershop_id` are resolved server-side via `TenantContext`, not trusted from arbitrary claims |
| Client storage | Mobile app (Expo/React Native) — Zustand-backed secure storage; no `httpOnly cookie` path exists yet, there is no web client in scope |

> **Known gap (tracked, not yet closed):** the `JWT_SECRET` used locally is a placeholder value committed in `application.yml` for dev convenience (`CHANGE_THIS_SECRET_IN_PRODUCTION_...`). Migrating it to a secrets manager (AWS Secrets Manager / Railway secrets) is a documented pre-production blocker — see `01-context/scope.md` → External dependencies.

**Prohibited in the payload:**
- Passwords
- Card data
- Full PII (only the user ID)

### Refresh Token

**Target policy (team commitment):**
- Stored in the database (with bcrypt hash)
- Mandatory rotation on each use (one refresh token = one use)
- Invalidated on logout and on password change
- ALL active tokens invalidated if use of a revoked token is detected

> **Known gap (tracked, not yet closed):** the current implementation issues the refresh
> token as a second, longer-lived stateless JWT (7-day expiration) — it is not yet persisted,
> rotated, or revocable. A Redis-backed revocation list is a documented pre-production
> blocker — see `01-context/scope.md` → External dependencies (`Token revocation list (Redis)
> for logout invalidation`). Do not build features that assume revocation works today.

---

## Authorization

### RBAC (Role-Based Access Control)

BarberSaaS has exactly four roles — no custom/extensible role model. Each controller
endpoint is annotated `@PreAuthorize("hasRole('ROLE_NAME')")` (Spring Security method
security, enabled via `@EnableMethodSecurity` in `SecurityConfig`); there is no separate
`resource:action` permission table.

| Role | Description | What it can do |
|------|-------------|------------|
| `SUPER_ADMIN` | Internal BarberSaaS platform team | Create/activate/suspend/cancel barbershop accounts, manage subscription plans, view platform-wide metrics (`SuperAdminDashboardController`, `SuperAdminBarbershopController`) |
| `ADMIN_BARBERSHOP` | Barbershop owner | Manage employees, schedules, service catalog, finances, loyalty program, inventory for their own barbershop (`BarberServiceController`, `BarbershopDashboardController`, etc.) |
| `BARBER` | Barbershop employee | View their daily agenda and upcoming appointments, grant loyalty stickers, register walk-in clients (`BarberStatsController`, appointment confirm/complete flows) |
| `CLIENT` | Barbershop customer | Search barbershops, book/cancel/reschedule appointments, earn and redeem loyalty rewards (`AppointmentController` client-facing endpoints) |

**Two layers of authorization, both mandatory:**
1. **Role check** — `@PreAuthorize("hasRole('X')")` on the controller: does this user's role
   allow calling this endpoint at all?
2. **Tenant check** — inside the service layer, every query is filtered by
   `barbershop_id` resolved from `TenantContext` (populated from the JWT by
   `JwtAuthenticationFilter`). A role check alone is **not** sufficient: an `ADMIN_BARBERSHOP`
   passing the role check must still be blocked from reading or writing another barbershop's
   data. See the multi-tenancy rule in each repo's `CLAUDE.md`.

**Validation:**
- `JwtAuthenticationFilter` validates the JWT (signature and expiration) on every request — there is no separate API Gateway; the Spring Boot app is the single deployable unit (ADR-002)
- `@PreAuthorize` enforces the role check at the controller
- The service layer enforces the tenant check via `TenantContext` — this is the check that actually protects cross-tenant data and must never be skipped

---

## Secure communication

### Transmission

- **HTTPS mandatory** in all environments except local
- TLS 1.2 minimum; TLS 1.3 recommended
- Certificates: Let's Encrypt (staging) / Corporate CA (production)
- HSTS enabled in production

### Internal module-to-module communication

- Not applicable in the current architecture: BarberSaaS is a **modular monolith**
  (ADR-002) — a single Spring Boot deployable unit. Bounded-context modules
  (`auth`, `barbershop`, `appointment`, `loyalty`, `finance`, …) call each other via direct
  in-process Java calls, not network requests, so there is no mTLS/API-key surface between
  them today.
- If a module is later extracted into its own service (see the trigger conditions in
  `overview.md` and the extraction roadmap referenced by ADR-002), this section must be
  revisited and a real service-to-service auth mechanism (mTLS or signed internal tokens)
  defined before that extraction ships.

---

## Secret management

```
✗ NEVER in source code
✗ NEVER in committed .env
✗ NEVER in logs
✗ NEVER in client error messages
✓ Environment variables (injected by the orchestrator)
✓ Vault (HashiCorp Vault, AWS Secrets Manager, etc.)
✓ Kubernetes Secrets (encrypted with KMS)
```

**Secret rotation:**
- API keys: every 90 days
- TLS certificates: 60 days before expiration
- DB passwords: every 6 months or immediately if compromise is suspected

---

## Input validation and sanitization

### General rules

1. **Never trust user input.** Validate at the edge (controller) before processing.
2. **Whitelist, not blacklist.** Define what is allowed, not only what is prohibited.
3. **Reject early.** If input is invalid, respond 400 and do not process further.

### SQL Injection — Prevention

```typescript
// ✗ VULNERABLE
const result = await db.query(`SELECT * FROM users WHERE email = '${userInput}'`);

// ✓ SAFE — always use prepared parameters
const result = await db.query('SELECT * FROM users WHERE email = $1', [userInput]);
```

### XSS — Prevention

```typescript
// ✗ VULNERABLE — rendering HTML without escaping
element.innerHTML = userProvidedContent;

// ✓ SAFE — use textContent or sanitize
element.textContent = userProvidedContent;
// or with library: DOMPurify.sanitize(userProvidedContent)
```

### Validation with Jakarta Bean Validation (backend) / Zod (mobile, where applicable)

```java
// Explicit validation annotations on the request DTO — backend (Spring Boot)
public record CreateAppointmentRequest(
    @NotNull UUID barberId,
    @NotNull UUID serviceId,
    @NotNull @Future LocalDateTime startTime
) {}

// @Valid on the controller parameter triggers validation before the method body runs
@PostMapping
public ResponseEntity<AppointmentDto> create(@Valid @RequestBody CreateAppointmentRequest request) { ... }
```

---

## OWASP Top 10 — Review checklist

| Vulnerability | Implemented control |
|---------------|-------------------|
| A01: Broken Access Control | `@PreAuthorize("hasRole(...)")` per endpoint + mandatory `barbershop_id` tenant filtering via `TenantContext` in the service layer |
| A02: Cryptographic Failures | TLS 1.2+, bcrypt for passwords, secrets in vault |
| A03: Injection | Prepared parameters in SQL, schema validation |
| A04: Insecure Design | Threat modeling in design, Security review |
| A05: Security Misconfiguration | IaC for configuration, review of defaults |
| A06: Vulnerable Components | Dependabot / Snyk for automatic updates |
| A07: Authentication Failures | JWT with rotation, brute-force protection |
| A08: Software Integrity Failures | Verify dependency checksums, SBOM |
| A09: Logging Failures | Logs without PII, centralized, with alerts |
| A10: SSRF | Whitelist of external URLs, do not follow redirects automatically |

---

## Audit and security logs

### Events that are ALWAYS recorded

```typescript
// Security events — store in a separate log, with retention > 1 year
const SECURITY_EVENTS = [
  'auth.login.success',
  'auth.login.failure',
  'auth.login.brute_force_detected',
  'auth.password.changed',
  'auth.token.revoked',
  'auth.unauthorized_access_attempt',
  'data.pii.accessed',
  'admin.role.changed',
  'admin.user.deleted',
];
```

**Required fields in security logs:**
- `userId` (or `ANONYMOUS` if not authenticated)
- `sourceIp`
- `action`
- `resource`
- `result` (SUCCESS / FAILURE)
- `timestamp`

---

## Vulnerability process

### What to do if you find a vulnerability

1. **Do not commit it to the public repo** or discuss it in open channels
2. Immediately notify the Tech Lead via a private channel
3. Create a private issue or a restricted repository issue
4. Severity is assigned (CVSS score or internal classification)
5. Remediated in the current sprint if critical, in the next sprint if high

### Remediation SLAs

| Severity | Remediation time |
|----------|----------------|
| Critical (CVSS 9-10) | 24 hours |
| High (CVSS 7-8.9) | 1 week |
| Medium (CVSS 4-6.9) | 1 month |
| Low (CVSS < 4) | Next security review |

---

## Correlations

- Security non-functional requirements → `04-requirements/non-functional.md`
- ADR on authentication → `05-architecture/decisions/`
- Security event observability → `13-operations/observability.md`
- RBAC and JWT implemented in → `com.barbersaas.auth` and `com.barbersaas.security` (backend), see `05-architecture/hexagonal-architecture.md` for the module boundary rules
- Multi-tenancy rule (`TenantContext`, `barbershop_id`) → each repo's `CLAUDE.md`, "Producto: BarberSaaS" section
