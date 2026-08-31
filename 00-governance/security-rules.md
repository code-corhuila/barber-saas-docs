# Technical Security Rules

> Mandatory technical controls that apply to all project code.
> These rules complement the security policy (`security-policy.md`) with
> concrete implementation practices.

---

## OWASP Top 10 — Controls per category

### A01 — Broken Access Control

```java
// ❌ BAD — trusting a client-supplied id/tenant without verifying ownership
Long barbershopId = request.getBarbershopId(); // came from the request body

// ✅ GOOD — resolve tenant from the verified JWT, never from client input
Long barbershopId = TenantContext.getBarbershopId(); // populated by JwtAuthenticationFilter
```

**Rules:**
- Every protected endpoint MUST have `@PreAuthorize("hasRole('X')")` applied — no endpoint is
  reachable by an unauthenticated or wrong-role user
- The role check happens at the controller (`@PreAuthorize`); the **tenant check** happens in
  the service layer, filtering every query by `barbershop_id` from `TenantContext`
- A method that receives only an `id` and returns a business entity MUST verify that entity
  belongs to the caller's tenant before returning it — never trust that an id "looks right"
- Write actions (create/update/delete) require both the role check and the tenant check;
  neither one alone is sufficient (see `00-governance/security-policy.md` → RBAC)

### A02 — Cryptographic Failures

**Rules:**
- Passwords: use **bcrypt** with cost factor ≥ 12 (Spring Security `BCryptPasswordEncoder`). Never MD5 or SHA-1 for passwords
- JWTs: currently signed with **HS512** (`Keys.hmacShaKeyFor` over `JWT_SECRET` — the algorithm is derived from key length; the current 65-byte secret resolves to HS512) — acceptable only while `JWT_SECRET` is a high-entropy value held in a secrets manager, never the placeholder committed for local dev. Never a weak or committed secret in production
- Sensitive data in transit: HTTPS mandatory in all environments except local
- Sensitive data at rest: encrypt with AES-256-GCM the fields marked as PII (there is no field currently marked as needing column-level encryption beyond password hashing — flag any new PII field for review before adding it)
- Never log passwords, tokens, or credit card data (BarberSaaS does not currently handle card data — billing is manual, per `01-context/scope.md`)

### A03 — Injection

**SQL:**
```java
// ❌ BAD — string concatenation (do not do this even for "internal" queries)
String sql = "SELECT * FROM users WHERE id = " + userId;

// ✅ GOOD — Spring Data JPA repository method (parameterized by construction)
Optional<User> user = userRepository.findById(userId);

// ✅ GOOD — JPQL with named parameters, when a repository method isn't enough
@Query("SELECT a FROM Appointment a WHERE a.barbershopId = :barbershopId AND a.status = :status")
List<Appointment> findByTenantAndStatus(@Param("barbershopId") Long barbershopId, @Param("status") AppointmentStatus status);
```

**Rules:**
- Parameterized queries ALWAYS — Spring Data JPA repository methods or `@Query` with named
  parameters. Zero concatenated strings in SQL, including in native `@Query(nativeQuery = true)`
- Validate and sanitize all inputs with Jakarta Bean Validation (`@Valid` + `@NotNull`,
  `@Size`, `@Pattern`, etc. on request DTOs) — see `00-governance/security-policy.md` for a full example
- GraphQL is not part of the current stack (REST only); this rule does not apply today

### A04 — Insecure Design

- Every HU that exposes user data must undergo privacy review
- Bulk query endpoints have mandatory pagination — default page size 20, maximum 100 records per page
- Internal entity IDs are auto-increment `Long` (JPA `@GeneratedValue`), not UUIDs, in the current schema — this is an accepted trade-off for the MVP given tenant filtering already blocks cross-tenant access at the service layer; do not expose an endpoint that lets one tenant enumerate another tenant's IDs even if guessed

### A05 — Security Misconfiguration

```
# Verification checklist per environment
□ Stack traces NOT visible in production
□ Security headers configured (Helmet.js or equivalent):
  - X-Content-Type-Options: nosniff
  - X-Frame-Options: DENY
  - Content-Security-Policy defined
  - Strict-Transport-Security in production
□ Unnecessary ports closed
□ Development credentials NOT in production
```

### A06 — Vulnerable Components

**Rules:**
- Run `npm audit` (or equivalent) before each release
- **Critical/High** vulnerabilities block the deploy
- Renew dependencies each sprint (at least once)
- Do not use `latest` versions without pinning in `package.json`; use exact versions or conservative ranges

### A07 — Identification and Authentication Failures

- JWT access token expiration: **24 hours** (`expiration-ms: 86400000`, current implementation — see the "known gap" note in `00-governance/security-policy.md` about tightening this before production)
- Refresh token expiration: **7 days** (`refresh-expiration-ms: 604800000`); DB-backed rotation on each use is a target policy, not yet implemented (see `security-policy.md`)
- Rate limiting on `/auth/login`: target maximum **10** attempts per IP in 5 minutes — **not yet implemented**; listed as a pre-production dependency in `01-context/scope.md` (Redis-based)
- Account lockout after **5** consecutive failed attempts — target policy, not yet implemented

### A08 — Software and Data Integrity Failures

- Verify Docker image checksum before using in production (backend ships as a single image, `Dockerfile` + `docker-compose.yml`)
- Third-party webhooks must verify cryptographic signature (applies once automated billing via PayU/Stripe ships in Phase 3 — see `01-context/scope.md`; no webhook consumer exists today)
- No message broker (Kafka/RabbitMQ) is part of the current stack — module communication is in-process (ADR-002). This rule does not apply until a module is extracted into an independently deployed service

### A09 — Security Logging and Monitoring Failures

- Every failed authentication must be logged with IP, timestamp, and user-agent
- Log delete operations with who, when, and what was deleted
- Security logs are retained for a minimum of **90 days**
- Automatic alerts configured for:
  - More than **50** 401/403 errors in 5 minutes
  - Access to a resource from an unexpected country — not applicable at MVP scale (single-country, Colombia-only launch); revisit if geographic expansion is scoped

> Centralized log aggregation and alerting are not yet wired up — this section defines the
> target once observability tooling (`13-operations/observability.md`, currently template)
> is implemented.

### A10 — Server-Side Request Forgery (SSRF)

- URLs constructed from user input MUST be validated against an allowlist of permitted domains
- Do not fetch from private IPs (192.168.x.x, 10.x.x.x, 127.x.x.x) from the server

---

## User input handling

```java
// Backend — Jakarta Bean Validation on the request DTO, always validated at the controller
public record RegisterUserRequest(
    @NotBlank @Email @Size(max = 255) String email,
    @NotBlank @Size(min = 1, max = 100) String name,
    @NotNull Role role // enum: SUPER_ADMIN, ADMIN_BARBERSHOP, BARBER, CLIENT
) {}

@PostMapping("/register")
public ResponseEntity<UserDto> register(@Valid @RequestBody RegisterUserRequest request) { ... }
```

```typescript
// Mobile (Expo/React Native) — Zod for client-side form validation before the API call
const RegisterFormSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().min(1).max(100).trim(),
});
```

**Rule:** All external inputs (HTTP body, query params, path params) pass through a
validation schema before reaching the domain — Jakarta Bean Validation on the backend,
Zod on the mobile client. Client-side validation is a UX convenience only; the backend
validation is the one that is trusted.

---

## Secure error handling

```java
// ❌ BAD — exposes internal details (stack trace, SQL, entity structure) to the client
catch (Exception e) {
    return ResponseEntity.status(500).body(Map.of("error", e.getMessage(), "stack", e.getStackTrace()));
}

// ✅ GOOD — generic message via a centralized @ControllerAdvice / @ExceptionHandler
// (see com.barbersaas.exception.GlobalExceptionHandler)
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleGeneric(Exception e) {
    log.error("Unhandled exception", e); // full detail goes to the server log only
    return ResponseEntity.status(500).body(new ErrorResponse("INTERNAL_SERVER_ERROR", "Internal server error"));
}
```

---

## Correlations

- Security policy (management, access, vault) → `00-governance/security-policy.md`
- System threat model → `05-architecture/security-threat-model.md`
- Authentication and JWT → `07-api/authentication.md`
- Observability and security logs → `13-operations/observability.md`
