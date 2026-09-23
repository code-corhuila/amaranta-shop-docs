# REST Guidelines

> The whole team must follow this before designing an endpoint. It reflects the
> conventions already implemented in `apps/api/src/` — this document describes what
> the code does, and new endpoints must match it.

## Versioning

URL-based: `/api/v1/...`. The monolith mounts every controller under this prefix
(see `apps/api/src/main.ts`).

## Endpoint naming

- Plural nouns for collections: `GET /products`, `GET /orders`, `GET /appointments`,
  `GET /pets`
- Nouns, not verbs, for the resource itself — action-style endpoints are the
  documented exception, not the rule (`POST /orders/{id}/mark-paid`,
  `POST /appointments/{id}/cancel`, `POST /admin/products/{id}/activate`): used only
  when the operation is a state transition on an existing resource, not a CRUD op
- camelCase for path parameters (`{productId}`, `{appointmentId}`, `{petId}`)

## Pagination

Query params: `?page=1&pageSize=50` (1-indexed, default `pageSize=50`). Every
paginated response returns `{ data: [...], meta: PaginatedMeta }` —
see `_shared.yaml#/components/schemas/PaginatedMeta`.

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

Every error response follows `_shared.yaml#/components/schemas/ErrorResponse`:

```json
{
  "error": "VALIDATION_ERROR",
  "message": "The email field is required",
  "details": [{ "field": "email", "message": "required" }]
}
```

`error` is a `SCREAMING_SNAKE_CASE` code consumers can branch on; `message` is for
humans; `details` is only present for validation errors.

## Authorization

Role checks use `@Roles(...)` + `RolesGuard` on top of `@ApiBearerAuth()` +
`JwtAuthGuard`. See `authentication.md` for the full role list and flow.
