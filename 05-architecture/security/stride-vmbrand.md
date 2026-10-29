# STRIDE - Browser to API Gateway public edge

## Scope

This analysis owns the trust boundary between an untrusted browser or Internet
client and `amaranta-shop-api-gateway`. It does not analyze the separate
gateway-to-service or service-to-database boundaries.

| Side | Components | Trust level |
|---|---|---|
| Source | Browser, public Internet, unauthenticated visitor, authenticated customer or administrator | Untrusted |
| Boundary | nginx public listener in `amaranta-shop-api-gateway` | Enforcement point |
| Destination | `sales-api`, `catalog-api`, and `workflow` through named public routes | Internal platform |

The gateway is configuration, not a business service: it routes public paths,
requires a present `Authorization` header for protected paths, limits requests,
applies CORS, and preserves or creates `X-Correlation-Id`. Token signature,
expiry, issuer, audience, and role checks remain responsibilities of the
destination service. This division follows ADR-012 and the Gateway contract.

## Data and entry points

| Asset crossing this boundary | Classification | Entry point | Current control |
|---|---|---|---|
| Catalog product name, price, stock availability | Public | `GET /api/v1/products*` | Explicit public locations only |
| Order line items, delivery/contact data, idempotency key | Confidential | `POST /api/v1/orders` | Explicit public checkout location; Sales validates an optional token |
| Customer order history and state | Confidential | `GET /api/v1/orders*` | Gateway rejects a missing credential |
| Product administration and stock changes | Restricted | `/api/v1/admin/products*` | Gateway rejects a missing credential; Catalog checks `ADMIN` |
| Correlation and trace identifiers | Internal | `X-Correlation-Id`, `traceparent` | Syntax validation, propagation, structured gateway logs |

`/internal/products/reserve-stock` and `/internal/products/release-stock` are
not entry points at this boundary. No Gateway route publishes them; direct,
path-traversal, and encoded-path variants must return `404`.

## STRIDE - first threats

Risk is likelihood (L) multiplied by impact (I), both on a 1-3 scale.

| Type | Concrete threat | L | I | Risk | Existing control | Missing mitigation / verification |
|---|---|---:|---:|---:|---|---|
| S | A caller sends `Authorization: Bearer not-a-token` to `GET /api/v1/orders` or an admin Catalog route. The Gateway deliberately checks only that the header exists, so accepting it would depend on a missing or incorrect service verifier. | 2 | 3 | 6 | The Gateway blocks a missing header; its routing suite proves a malformed token is forwarded for the service to reject. | Sales, Catalog, and Workflow must independently reject invalid signature, issuer, audience, expiry, and unauthorized role claims; add negative tests per protected endpoint. |
| T | A caller uses `../`, `%2e%2e`, or encoded slashes below `/api/v1/products/` to reach `internal/products/reserve-stock` and alter stock outside the Sales workflow. | 1 | 3 | 3 | The Gateway uses exact locations or prefixes ending in `/`; current routing tests assert `404` for direct and encoded traversal attempts. | Keep these cases in Gateway CI whenever routes change; the Catalog internal endpoint must also require a service-to-service identity when that boundary is implemented. |
| R | A caller disputes an order creation or status change and supplies an arbitrary but syntactically valid `X-Correlation-Id`, leaving no durable actor-to-order evidence at the edge. | 2 | 2 | 4 | The Gateway validates or generates the correlation identifier and returns it in the response and error envelope. | Sales must record order id, authenticated subject when present, idempotency key, action, outcome, and correlation id in an audit record; a test must prove rejected attempts are recorded without logging tokens or bodies. |

## Initial ranking

1. **S - risk 6:** service-side verification is the final authorization control after the Gateway's cheap header filter.
2. **R - risk 4:** trace correlation exists, but accountability requires a durable Sales audit event.
3. **T - risk 3:** the Gateway currently blocks the known traversal forms; regression coverage must remain mandatory.

## Evidence consulted

- `05-architecture/decisions/records/ADR-012-stack-of-extracted-services.md`
- `07-api/contracts/openapi/api-gateway.yaml`
- `amaranta-shop-api-gateway` route configuration and `tests/routing-checks.sh`

## References

The mandatory physical-library reference and the cited pages must be completed
by the author from the genuine library consultation record before this analysis
is considered complete.
