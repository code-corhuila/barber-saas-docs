# SEQ-01 · Login and Token Validation

> **Type:** UML sequence · **Derived from:** `07-api/authentication.md`,
> `07-api/contracts/openapi/auth-service.yaml` (`login`, `jwks`) · **Rule:** norm 5.3.7, 5.6.2

How a user gets a token and how any other service trusts it without calling identity-auth on
every request.

```mermaid
sequenceDiagram
  autonumber
  actor U as User (mobile app)
  participant GW as api-gateway
  participant AUTH as identity-auth-api
  participant ADB as identity-auth-db
  participant SVC as any domain -api

  U->>GW: POST /api/v1/auth/login {email, password}
  GW->>AUTH: forward (public route) + X-Correlation-Id
  AUTH->>ADB: find app_user by lower(email)
  ADB-->>AUTH: user (password_hash, role, barbershop_id)
  AUTH->>AUTH: BCrypt check · sign JWT RS256 (sub, role, barbershopId, exp 24 h)
  AUTH->>ADB: INSERT refresh_token (hash, expires 7 days)
  AUTH-->>GW: 200 {accessToken, refreshToken, expiresIn: 86400}
  GW-->>U: 200

  Note over SVC,AUTH: Once, and again when an unknown kid arrives
  SVC->>AUTH: GET /api/v1/auth/jwks
  AUTH-->>SVC: public keys (kid)

  U->>GW: GET /api/v1/... Authorization: Bearer token
  GW->>GW: credential present? (no signature check here)
  GW->>SVC: forward
  SVC->>SVC: alg == RS256 · signature by kid · exp and sub · role
  alt token valid and role allowed
    SVC-->>U: 2xx (data filtered by barbershopId from the token)
  else invalid or expired
    SVC-->>U: 401 UNAUTHORIZED
  else role not allowed
    SVC-->>U: 403 FORBIDDEN
  end
```

Invalid credentials, an inactive account or a temporarily locked one answer `401` without
saying which case applies (`DEC-AUTH-02`). Refresh
(`POST /api/v1/auth/refresh`) rotates both tokens and revokes the old refresh token.
