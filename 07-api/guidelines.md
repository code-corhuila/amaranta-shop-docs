# REST Guidelines

> The whole team must follow this before designing an endpoint. It reflects the
> conventions already implemented in `apps/api/src/` — this document describes what
> the code does, and new endpoints must match it.

## Versioning

The current Amaranta monolith does not mount a global `/api/v1` prefix in code.
Controllers are exposed directly at `/auth`, `/products`, and `/orders` by their
resource routes (see `apps/api/src/.../*.controller.ts`). When the system is later
split into independent services, the target contract for the service boundary can
still be `/api/v1/...`, but the implementation contract must describe the current
runtime accurately before requiring a gateway versioning layer.

## Endpoint naming

- Plural nouns for collections: `GET /products`, `GET /orders`, `GET /appointments`,
  `GET /pets`
- Nouns, not verbs, for the resource itself — action-style endpoints are the
  documented exception, not the rule (`POST /orders/{id}/mark-paid`,
  `POST /appointments/{id}/cancel`): used only when the operation is a state
  transition on an existing resource, not a CRUD op
- camelCase for path parameters (`{productId}`, `{appointmentId}`, `{petId}`)

## Pagination

The current implementation uses offset-based pagination with numeric query params:
`?page=1&pageSize=50`.

- `page` is 1-indexed and defaults to `1`
- `pageSize` defaults to `50`
- the runtime caps `pageSize` at `50` (`MAX_PAGE_SIZE = 50`)
- empty result sets still return the paginated envelope, e.g. `{ items: [], page: 1,
  pageSize: 50, total: 0 }`
- the delivery contract currently uses `{ items, page, pageSize, total }` instead of
  `{ data, meta }` because that is the shape produced by `ListProductsUseCase` in
  the monolith today

This is the behavior to document for the current code. Any future shift to a
`data/meta` envelope should be treated as an explicit cross-service contract
change, not an implicit monolith convention.

## Standard responses

| Code | When to use it |
|------|-----------------|
| 200 | Success with body |
| 201 | Successful creation |
| 204 | Success without body |
| 400 | Client error (validation) |
| 401 | Not authenticated |
| 403 | Not authorized for this role |
| 404 | Resource not found |
| 409 | Conflict (e.g. email already registered) |
| 500 | Server error |

## Error format

The current implementation does not yet use the shared ErrorResponse schema
uniformly. Domain-level exceptions are filtered by
`DomainExceptionFilter`, which returns a minimal payload:

```json
{
  "statusCode": 400,
  "code": "INV-PRODUCT-003",
  "message": "quantity to decrement must be positive"
}
```

This is what the code emits today; it is not yet aligned with the shared
`ErrorResponse` contract in `_shared.yaml` (`error`, `message`, `details`). The
contract should treat this as a pending standardization task before the service is
split into independent microservices.

## Authorization

Role checks use `@Roles(...)` + `RolesGuard` on top of `@ApiBearerAuth()` +
`JwtAuthGuard`. See `authentication.md` for the full role list and flow.
