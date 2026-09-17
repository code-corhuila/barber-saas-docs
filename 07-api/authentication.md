# Authentication & Authorization

> Source of truth: `05-architecture/overview.md` §5 (Architectural Principles, P4:
> Stateless Authentication) and §7 (Cross-cutting concerns). This document restates that
> decision for the API contracts under `contracts/openapi/` — if the two ever disagree,
> `05-architecture/overview.md` wins and this file is out of date.

## Mechanism

JWT (JSON Web Token), signed with **HS512** (symmetric, shared-secret HMAC — not RS256).
There is no public/private key pair and no JWKS endpoint: the signing secret is a server
side value shared only between the components that issue and validate tokens.

- **Access token**: expires in **24 hours** (86400 seconds).
- **Refresh token**: expires in **7 days**, single use (rotated on every
  `POST /auth/refresh` — the previous refresh token is invalidated).

There is no server-side session. Authentication is fully stateless, which is what allows
the backend to scale horizontally behind a load balancer without session affinity
(overview.md §5, P4).

## Authentication flow

1. `POST /auth/register` or `POST /auth/login` — returns `accessToken`, `refreshToken`,
   `expiresIn` (86400) and the authenticated user's summary.
2. Send `Authorization: Bearer <accessToken>` on every subsequent request.
3. When the access token expires, `POST /auth/refresh` with the current `refreshToken` to
   get a new pair. The old refresh token stops being valid the moment this succeeds.
4. `POST /auth/logout` revokes the current session's refresh token (and optionally all of
   the user's refresh tokens, via `allDevices: true`).

See `contracts/openapi/auth-service.yaml` for the exact request/response schemas.

## Roles

BarberSaaS defines exactly four roles (overview.md lines 47-48). There is no generic
`ADMIN` / `USER` / `VIEWER` set — any contract or document using those names is describing
a different, unrelated system:

| Role | Who |
|------|-----|
| `SUPER_ADMIN` | Platform administrator — operates the SaaS itself, not a single barbershop |
| `ADMIN_BARBERSHOP` | Barbershop owner — manages one tenant (employees, schedules, finance) |
| `BARBER` | Barbershop employee — manages their own appointments and schedule |
| `CLIENT` | End customer — books and manages their own appointments |

Roles are carried in the JWT and enforced per endpoint via Spring Security's
`@PreAuthorize` (overview.md §7, "Authentication / Authorization").

## Multi-tenancy: how `barbershop_id` is derived from the JWT

BarberSaaS is multi-tenant, isolated by a `barbershop_id` column on every tenant-scoped
table (overview.md §5, P1: Tenant Isolation by Design). This isolation is not optional
per-endpoint behavior — it applies to every request that touches barbershop-scoped data:

1. The JWT issued at login embeds the user's `barbershop_id` (for `ADMIN_BARBERSHOP`,
   `BARBER` and `CLIENT` — `SUPER_ADMIN` operates across tenants and is not bound to one).
2. `JwtAuthenticationFilter` validates the token on every request and populates
   `TenantContext`, a request-scoped (`ThreadLocal`) holder for the resolved
   `barbershop_id`.
3. Every service method that reads or writes barbershop-scoped data validates against
   `TenantContext` before touching the database — so a request authenticated for one
   barbershop cannot read or mutate another barbershop's data through the API layer, even
   if it guesses a valid resource ID belonging to a different tenant.
4. `TenantContext` is cleared in a `finally` block at the end of the request, so no state
   leaks between requests handled by the same thread.

Any endpoint added to a contract in `contracts/openapi/` that returns or accepts
barbershop-scoped data must be assumed to go through this same tenant check — a contract
that omits `barbershop_id` from its request/response shapes is relying on the JWT-derived
`TenantContext`, not asking the client to supply the tenant explicitly.
