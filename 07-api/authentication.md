# Authentication & Authorization

> How every contract under `contracts/openapi/` authenticates callers. Architecture:
> `05-architecture/overview.md` §5 (P2, P3) and §7. Rule: course norm 5.3.7 and 5.6.2.
> Endpoints: `contracts/openapi/auth-service.yaml` (`barber-saas-identity-auth-api`).

## Mechanism

JWT signed with **RS256**. identity-auth holds the only private key; every other service
validates with the public key published at `GET /api/v1/auth/jwks`. No service holds the
private key or a shared secret.

- **Access token:** expires in **24 hours** (`expiresIn: 86400`).
- **Refresh token:** opaque, **7 days**, single use — rotated on every
  `POST /api/v1/auth/refresh`; stored hashed in `identity_auth.refresh_token`.
- Authentication is stateless: no server-side session and no revocation list. Logout revokes
  refresh tokens; an access token lives until it expires.

### Claims

| Claim | Content | Required |
|---|---|---|
| header `alg` | `RS256` — any other value is rejected | Yes |
| header `kid` | Key id, matched against the JWKS | Yes |
| `sub` | User id (UUID), or the service name for a service token | Yes |
| `exp`, `iat` | Expiration and issue time | Yes |
| `iss` | `barber-saas-identity-auth-api` | Yes |
| `role` | One of the four roles below, or `SERVICE` | Yes |
| `barbershopId` | Tenant (UUID) for `ADMIN_BARBERSHOP` and `BARBER`; absent otherwise | By role |

## Validation in every service (norm 5.3.7)

The api-gateway only checks that a protected route carries a credential (norm 5.6.2).
**Each service validates the token itself**, because internal calls do not pass through the
gateway:

1. Algorithm exactly `RS256`; `none`, `HS256` and anything else → `401`.
2. Signature checked with the JWKS key whose `kid` matches; the key set is cached and reloaded
   when an unknown `kid` arrives (key rotation).
3. `exp` and `sub` present and `exp` in the future → otherwise `401`.
4. The role is checked per operation → `403` when not allowed.

In `develop` the key pair comes from `barber-saas-infra/scripts/dev-keys.sh`
(`JWT_PUBLIC_KEY` in `.env`); in `qa` and `main` it comes from identity-auth as an environment
secret (norm 5.9.2, `05-architecture/deployment.md` §6).

## Service tokens

`barber-saas-worker` and `barber-saas-workflow` call domain services with their own token
(`role: SERVICE`, `sub` = service name), never with a user's token — a user's token can
expire in the middle of a saga compensation (norm 5.8.2). Service tokens are issued by
identity-auth and delivered as environment secrets; in `develop`, `dev-keys.sh` writes a
`SERVICE_TOKEN`. An operation that accepts service tokens says so in its contract.

## Roles

BarberSaaS has exactly four user roles, one per user (`identity_auth.app_user.role`):

| Role | Who |
|------|-----|
| `SUPER_ADMIN` | Platform administrator — operates the SaaS, not a single barbershop |
| `ADMIN_BARBERSHOP` | Barbershop owner — manages one tenant (catalog, staff, schedules, finance) |
| `BARBER` | Barbershop employee — manages their own appointments and schedule |
| `CLIENT` | End customer — books and manages their own appointments |

## Multi-tenancy: the tenant comes from the token

Every tenant-scoped table carries `barbershop_id` (`06-data/models.md`, ADR-010). In every
service:

1. The HTTP adapter reads `barbershopId` from the validated token and passes it to the use
   case as an explicit argument. **No operation accepts the tenant in the path, query or
   body.**
2. Every query and write of tenant-scoped data filters by it. A resource of another tenant
   answers `404`, never `403`, so its existence is not confirmed (`DEC-APPT-01`).
3. `SUPER_ADMIN` carries no tenant and is refused by tenant-scoped operations; platform-wide
   work goes through `platform-admin-service.yaml`.

With ADR-004 this filter is repeated in eight services instead of living once in the
prototype's `TenantContext`; a cross-tenant test is required in every `-api`
(`06-data/migration-strategy.md` §4).

**Open:** a `CLIENT` is platform-wide (`barbershop_id` is `NULL`) and visits several
barbershops. How a client's request is bound to one barbershop is `open-questions.md` OQ-07.
