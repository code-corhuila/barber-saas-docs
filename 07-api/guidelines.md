# API Guidelines

> REST standards for BarberSaaS. Every endpoint, in every service contract under
> `contracts/openapi/`, must follow this document. The error and pagination formats
> defined here are not a description — they are the exact schemas `ErrorResponse` and
> `PaginatedMeta` in [`contracts/openapi/_shared.yaml`](contracts/openapi/_shared.yaml).
> Do not define a parallel or slightly different format anywhere else.

## Versioning

- Version goes in the URL path: `/api/v1/resources`.
- The version increments only on a breaking change (removed field, changed type, removed
  endpoint). Additive changes (new optional field, new endpoint) do not require a bump.
- Since [ADR-004](../05-architecture/decisions/records/ADR-004-full-microservice-decomposition.md), each domain is its own service
  (`barber-saas-<domain>-api`) exposing `/api/v1/<domain-resources>`. Every contract uses these
  paths, with `servers` pointing at the gateway (`http://localhost:8000` locally); the
  monolith's role-prefixed paths (`/api/client`, `/api/admin`, …) are gone. Role checks happen
  per operation, not per path prefix.
- **`barber-saas-api-gateway` is the only entry point** (course norm 5.6.1): clients never call
  a domain service directly, and only the gateway is published to the host. The gateway is
  NGINX configuration (one routes file per domain), not an OpenAPI contract: its own errors
  (`401`, `404`, `429`, `503`) use the shared `ErrorResponse`. Services still validate the
  token themselves (5.6.2).

## Common contract (course norm 5.3.5 – 5.3.9)

Every `-api` and the `-workflow` follow these rules, so a consumer cannot tell which
language a service is written in. The reusable pieces live in `_shared.yaml`.

| Aspect | Rule | `_shared.yaml` component |
|---|---|---|
| Names | JSON in `camelCase` | — |
| Identifiers | UUID | `schemas/UUID`, `parameters/IdParam` |
| Money | Integer in minor units, suffix `Cents` (`totalCents`); never floating point | `schemas/Money` |
| Dates | RFC 3339, UTC | `schemas/Timestamp` |
| Errors | One envelope `{error, message, details?, traceId}`, also for unknown routes and malformed JSON | `schemas/ErrorResponse`, `schemas/ErrorCode` |
| Lists | Paginated, `{data, meta}`, stable order (most recent first) | `schemas/PaginatedList`, `PageParam`, `LimitParam` |
| Creation | `Idempotency-Key` header required (8–128 chars); same key → same resource with `200` | `parameters/IdempotencyKeyHeader` |
| Correlation | `X-Correlation-Id` reused or generated, returned, logged, used as `traceId` | `parameters/CorrelationIdHeader`, `headers/X-Correlation-Id` |
| Token | RS256 with the identity service's public key, validated by each service | `securitySchemes/bearerAuth` |

A creation answers `201` with a `Location` header (`headers/Location`) the first time and
`200` with the same resource on a retry with the same `Idempotency-Key`.

## Endpoint naming

- Plural nouns for collections: `GET /appointments`, not `GET /appointment`.
- Nouns, not verbs: `GET /users/{id}/orders`, not `GET /getUserOrders`.
- Sub-resources nest under their parent: `GET /barbershops/{id}/employees`.
- Use kebab-case for multi-word path segments: `/device-tokens`, not `/deviceTokens` or
  `/device_tokens`.
- Actions that don't map to a CRUD verb are modeled as a sub-resource, not a verb in the
  path: `POST /notifications/{id}/read`, not `POST /notifications/markAsRead`.

## Pagination

Offset-based, via the shared parameters `PageParam` and `LimitParam`
(`_shared.yaml#/components/parameters/`):

- `?page=1&limit=20` — `page` starts at 1, `limit` defaults to 20 with a maximum of 100.
  A `limit` out of range or an unknown filter value is `400 VALIDATION_ERROR`.
- Order is stable, most recent first. A list without a limit is a defect (norm 5.3.6).
- Every paginated list response returns a `meta` object matching
  `_shared.yaml#/components/schemas/PaginatedMeta` exactly:

```json
{
  "data": [ /* ... */ ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 142,
    "totalPages": 8
  }
}
```

Cursor-based pagination is not currently used by any contract in this project. If a
future endpoint needs it (very large, append-only collections), that is an addition to
this document, not a silent deviation from it.

## Standard status codes

| Code | When to use it |
|------|-----------------|
| 200 | Success with body |
| 201 | Successful creation |
| 204 | Success without body (e.g. `DELETE`, some `POST` actions) |
| 400 | `VALIDATION_ERROR` — invalid input shape; one `details` entry per field |
| 401 | `UNAUTHORIZED` — missing, invalid or expired JWT |
| 403 | `FORBIDDEN` — valid token without permission for this action |
| 404 | `NOT_FOUND` — resource or route does not exist |
| 422 | `INVALID_STATUS_TRANSITION` (state rules) or `BUSINESS_RULE_VIOLATION` (other invariant, e.g. double booking, duplicate email) |
| 429 | `TOO_MANY_REQUESTS` — gateway only, with `Retry-After` |
| 500 | `INTERNAL_ERROR` — neutral message; full detail only in the log |
| 503 | `SERVICE_UNAVAILABLE` — gateway only, target service down |

`409` is not part of the course's closed code list (norm 5.3.5): conflicts with the
current state are `422`. No contract answers `409`.

## Error format

Every error response — from every endpoint, in every contract — uses the `ErrorResponse`
schema from `_shared.yaml`, referenced as `$ref: '../_shared.yaml#/components/schemas/ErrorResponse'`
(or `'./_shared.yaml#/...'` from within `contracts/openapi/`). Never redefine this shape
locally:

```json
{
  "error": "VALIDATION_ERROR",
  "message": "The email field is required",
  "details": [
    { "field": "email", "message": "required" }
  ],
  "traceId": "550e8400-e29b-41d4-a716-446655440000"
}
```

- `error`: machine-readable code in `SCREAMING_SNAKE_CASE`.
- `message`: human-readable summary.
- `details`: optional, present for validation errors — one entry per invalid field,
  including headers such as `Idempotency-Key`.
- `traceId`: **always present**; it is the request's `X-Correlation-Id`.
- An error never exposes a driver message, a stack trace or an internal host name.

The reusable `BadRequest` / `Unauthorized` / `Forbidden` / `NotFound` / `InternalError`
responses in `_shared.yaml#/components/responses/` already wrap this schema — reference
those instead of writing the response object out again in a new contract.
