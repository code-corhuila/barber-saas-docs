# 07 — API Contracts

> **What is this?** The formal specification of how microservices communicate with each other
> and with the outside world. The contract is the promise a service makes to its consumers.

## Why contracts are critical in microservices

In a monolith, changing a function is easy: the compiler tells you what you break.
In microservices, a change to an API can silently break other services in production.

**Contract-first development:** design the API before implementing it. This way the team
can work in parallel (frontend, backend, other services) with an agreed contract.

---

## What is here and how to fill it in

### `guidelines.md` ⭐
The project's REST standards. **The entire team must follow this before designing an endpoint.**

**Key topics to define:**
```markdown
## Versioning
- URL: /api/v1/resources (version in the URL)
- Header: Accept: application/vnd.api+json;version=1

## Endpoint naming
- Plural for collections: GET /users
- Nouns, not verbs: GET /users/{id}/orders (not: GET /getUserOrders)
- snake_case or kebab-case for URLs

## Pagination
- ?page=1&limit=20 (offset-based)
- ?cursor=xyz (cursor-based for large volumes)

## Standard responses
| Code | When to use it |
|------|----------------|
| 200 | Success with body |
| 201 | Successful creation |
| 204 | Success without body |
| 400 | Client error (validation) |
| 401 | Not authenticated |
| 403 | Not authorized |
| 404 | Resource not found |
| 422 | Unprocessable entity |
| 500 | Server error |

## Error format
{
  "error": "VALIDATION_ERROR",
  "message": "The email field is required",
  "details": [{"field": "email", "message": "required"}]
}
```

### `authentication.md` ⭐
The system's authentication and authorization strategy.
**Fill in:** what mechanism (JWT, OAuth2, API Key), authentication flow, expiration policies,
refresh token handling, RBAC (roles and permissions).

### `contracts/openapi/` ⭐⭐
One `.yaml` file per microservice with the OpenAPI 3.0 contract.

**File name:** `service-name.yaml`

**Minimum structure for each contract:**
```yaml
openapi: 3.0.3
info:
  title: [Service Name] API
  version: 1.0.0
  description: [What this service does]

servers:
  - url: http://localhost:8080/api/v1
    description: Local

paths:
  /resources:
    get:
      summary: List resources
      tags: [Resources]
      parameters:
        - name: page
          in: query
          schema:
            type: integer
      responses:
        '200':
          description: List of resources
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ResourceList'

components:
  schemas:
    Resource:
      type: object
      required: [id, name]
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
  
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

### `_shared.yaml`
Reusable schemas and components across all contracts (pagination, errors, etc.)

### `open-questions.md`
What this section deliberately leaves open — each gap with its evidence, why it isn't
closed yet, an owner, and the condition that closes it. Not a TODO list: a gap only belongs
here once there's a real reason it can't be closed today.

---

## Contract index

One contract per domain `-api` (ADR-004), all reached through `barber-saas-api-gateway` under
`/api/v1`. Shared components live in [`_shared.yaml`](contracts/openapi/_shared.yaml);
`_template-service.yaml` is the starting point for a new one. The gateway itself is NGINX
configuration, not a contract (`guidelines.md`).

| Domain | Repo | Contract | Tables (`06-data/models.md`) | FRs | Notes |
|---|---|---|---|---|---|
| identity-auth | `barber-saas-identity-auth-api` | [`auth-service.yaml`](contracts/openapi/auth-service.yaml) | `users` | FR-001–FR-004 | Still on the monolith's paths and `409` — OQ-05 |
| barbershop | `barber-saas-barbershop-api` | [`barbershop-service.yaml`](contracts/openapi/barbershop-service.yaml) | `barbershops`, `services`, `barber_profiles`, `barber_specialties` | FR-005, FR-023, FR-027, FR-028 | OQ-06, OQ-07, OQ-08 |
| appointment | `barber-saas-appointment-api` | [`appointment-service.yaml`](contracts/openapi/appointment-service.yaml) | `appointments` | FR-008–FR-012 | Still on the monolith's paths and `409` — OQ-05 |
| schedule | `barber-saas-schedule-api` | [`schedule-service.yaml`](contracts/openapi/schedule-service.yaml) | `barber_schedules`, `schedule_exceptions` | FR-006, FR-007, FR-008 | OQ-09 |
| loyalty | `barber-saas-loyalty-api` | [`loyalty-service.yaml`](contracts/openapi/loyalty-service.yaml) | `loyalty_rewards_config`, `loyalty_cards`, `loyalty_transactions`, `reward_coupons` | FR-010, FR-013–FR-015 | OQ-02 |
| notifications | `barber-saas-notifications-api` | [`notification-service.yaml`](contracts/openapi/notification-service.yaml) | `notifications` | FR-020–FR-022 | Placeholder — OQ-03, OQ-05 |
| finance-inventory | `barber-saas-finance-inventory-api` | [`finance-inventory-service.yaml`](contracts/openapi/finance-inventory-service.yaml) | `finance_records`, `inventory_products`, `inventory_movements` | FR-016–FR-019 | OQ-06 |
| platform-admin | `barber-saas-platform-admin-api` | [`platform-admin-service.yaml`](contracts/openapi/platform-admin-service.yaml) | `subscription_plans` (+ `barbershops` via barbershop) | FR-004, FR-023–FR-026 | OQ-10, OQ-11 |

Lint every contract before a PR:

```bash
npx @redocly/cli lint 07-api/contracts/openapi/<service>.yaml
```

---

## Recommended tools

- **Swagger Editor** — Online editor for OpenAPI
- **Redocly** — Generates HTML documentation from the YAML (already configured in `redocly.yaml`)
- **Postman / Insomnia** — For testing the contracts
- **OpenAPI Generator** — Generates client/server code from the YAML

---

## Correlations with other sections

| This section is fed by... | And feeds into... |
|---------------------------|-------------------|
| `04-requirements/functional.md` → what it must do | Endpoints implementing each FR |
| `06-data/models.md` → what data exists | Schemas in the contracts |
| `02-domain/domain-events.md` | Event publishing endpoints |
| Contracts here | `09-microservices/[service]/` that implements them |
| Contracts here | `11-quality/testing-strategy.md` → contract tests |

---

## Questions this section must answer

- What is each endpoint called and what does it do?
- What data does it receive and what does it return?
- How does the client authenticate?
- What errors can the service return?
- How do I version the API without breaking consumers?
