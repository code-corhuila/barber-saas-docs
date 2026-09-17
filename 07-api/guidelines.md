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
- BarberSaaS is currently a single modular monolith exposing one version (`v1`) under
  role-prefixed paths (`/api/public`, `/api/auth`, `/api/client`, `/api/barber`,
  `/api/admin`, `/api/super-admin`) — see `05-architecture/overview.md` §5 (P5).

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
| 400 | Client error (validation failure) |
| 401 | Not authenticated (missing or invalid JWT) |
| 403 | Authenticated but not authorized for this resource/action |
| 404 | Resource not found |
| 409 | Conflict with current state (e.g. duplicate email, double booking) |
| 422 | Semantically invalid request the schema alone can't reject |
| 500 | Server error |

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
- `details`: optional, present for validation errors with multiple field-level causes.
- `traceId`: optional, present when correlating with server-side logs.

The reusable `BadRequest` / `Unauthorized` / `Forbidden` / `NotFound` / `InternalError`
responses in `_shared.yaml#/components/responses/` already wrap this schema — reference
those instead of writing the response object out again in a new contract.
