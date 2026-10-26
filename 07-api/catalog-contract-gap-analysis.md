# Catalog contract gap analysis and decisions

## Scope

This record closes the contract gaps identified in the public catalog endpoint of the Amaranta MVP. It is intentionally limited to the portion that is backed by live code in `apps/api/src/catalog/*` and to the conventions that affect integration with Sales.

## Decision log

| ID | Problem | Evidence | Alternatives | Decision proposed | Justification | Consequences | Files affected | Acceptance criteria | Status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| CAT-01 | Route and versioning are inconsistent across the docs and the monolith. | `apps/api/src/main.ts` does not call `app.setGlobalPrefix('api/v1')`; controllers are mounted directly as `/products`, `/auth`, `/orders`. | Keep the future `/api/v1` contract as-is, or document the runtime reality. | The current contract is documented as the monolith's runtime route (`/products`) and `/api/v1` is treated as a future service-boundary target only. | It keeps the contract honest and avoids telling frontend or other teams to call a route the app does not expose. | Service consumers need to know the current monolith and the future extraction target are not the same thing. | `07-api/guidelines.md`, `07-api/contracts/openapi/catalog-service.yaml` | A reader can tell which URL is implemented today and which is a target state. | Vigente |
| CAT-02 | Pagination is described as `{ data, meta }` but the code returns `{ items, page, pageSize, total }`. | `ListProductsUseCase.execute()` returns `items`, `page`, `pageSize`, `total`; repository uses `findActivePaginated()`. | Keep the shared `data/meta` envelope or document the actual runtime envelope. | Use the runtime envelope and document it explicitly as the current contract. | This matches the actual code path used by the frontend and sales integration. | Any future shift to `data/meta` requires a deliberate contract version change. | `07-api/guidelines.md`, `07-api/contracts/openapi/catalog-service.yaml` | Response examples and schema match the code. | Vigente |
| CAT-03 | The catalog OpenAPI file includes admin CRUD endpoints that do not exist in the current codebase. | `CatalogController` contains only `GET /products`; no admin controller or product mutation flow is present under `apps/api/src/catalog`. | Remove the admin operations from the catalog contract or explicitly mark them as future. | Keep only the implemented route in the OpenAPI contract and document admin product management as pending work. | This prevents false requirements from being treated as implemented features. | The team must create a separate work item for product management before claiming the catalog is fully CRUD-capable. | `07-api/contracts/openapi/catalog-service.yaml` | The file contains only implemented operations for the current monolith. | Vigente |
| CAT-04 | Error handling is documented as a single shared error schema, but the code emits a thin domain error payload instead. | `DomainExceptionFilter` returns `{ statusCode, code, message }` with `HttpStatus.BAD_REQUEST`. | Standardize the domain filter to the shared `ErrorResponse` contract or keep the current contract and document the mismatch. | Document the current runtime error shape as the code behavior and register the standardization gap as pending. | The solution is contract-first: document what actually happens before changing the exception contract globally. | A future shared error schema change must be coordinated across auth, sales and catalog. | `07-api/guidelines.md`, `07-api/contracts/openapi/catalog-service.yaml` | The error contract makes the difference between implemented behavior and desired target behavior explicit. | Propuesto |
| CAT-05 | Query validation is undocumented. | The controller receives strings and converts them with `Number(...)` and defaults to `1`/`50`; there is no custom validation in code. | Reject invalid values or accept runtime coercion. | Document the current behavior: invalid numeric values are coerced and defaulted, with no schema-level validation error. | This reflects the actual user experience and avoids inventing a validation rule the implementation does not enforce. | Frontend must rely on defaults rather than a strict server-side 400 for bad query params. | `07-api/contracts/openapi/catalog-service.yaml` | The OpenAPI description states the exact default and fallback behavior. | Vigente |

## Endpoint fiche: GET /products

- ID: CAT-GET-01
- Purpose: Return the active product catalog for customers.
- Method and route: `GET /products`
- Domain owner: Catalog
- Authentication: public (`security: []`)
- Roles allowed: none required
- Parameters:
  - `page` (optional, integer, default `1`)
  - `pageSize` (optional, integer, default `50`, maximum `50`)
- Request body: none
- Validations in code: values are coerced with `Number(...)`; no `class-validator` schema exists for this endpoint.
- Success response:

```json
{
  "items": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "sku": "FOOD-DOG-001",
      "name": "Adult dog kibble 3kg",
      "description": "Balanced feed for adult dogs of any breed",
      "category": "FOOD",
      "price": 45000,
      "priceFormatted": "$45.000,00",
      "stockQuantity": 20,
      "inStock": true,
      "imageUrl": "https://example.com/products/food-dog-001.jpg"
    }
  ],
  "page": 1,
  "pageSize": 50,
  "total": 27
}
```

- Error responses:
  - Current runtime: `400` with `{ statusCode, code, message }` from `DomainExceptionFilter` only for domain errors, not for query validation.
  - Pending standardization: unify with the `ErrorResponse` schema in `07-api/contracts/openapi/_shared.yaml`.
- Edge cases:
  - Empty result set returns `items: []` and `total: 0`.
  - Invalid numeric query values are defaulted instead of rejected.
  - `pageSize` is capped at `50` in the use case.
- Dependencies:
  - `apps/api/src/catalog/application/list-products.use-case.ts`
  - `apps/api/src/catalog/domain/ports/out/product-repository.port.ts`
- Contract tests:
  - GET /products?page=1&pageSize=50 returns a 200 with the envelope above.
  - GET /products with no params returns the default page and pageSize.
  - Empty catalog returns `total: 0`.
- Pending gaps:
  - Admin catalog operations remain unimplemented.
  - Error schema standardization across the monolith remains pending.
  - Service extraction into `/api/v1` will require a versioned gateway strategy.

## Gap register

| Gap | Owner | Closure criterion |
| --- | --- | --- |
| Standardize error payloads across Catalog, Identity and Sales | Team / API owner | One shared error schema is implemented and documented in code and OpenAPI | 
| Add admin catalog CRUD endpoints and auth requirements | Catalog owner | CRUD routes are implemented, security is enforced, and contract tests pass |
| Establish a stable versioning strategy for the service boundary | Architecture owner | `/api/v1` target is defined at the gateway or service boundary and documented | 
