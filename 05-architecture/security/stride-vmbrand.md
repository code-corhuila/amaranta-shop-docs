# STRIDE - Browser to API Gateway public edge

## Scope

This analysis owns the trust boundary between an untrusted browser or Internet
client and `amaranta-shop-api-gateway`. It does not analyze the separate
Gateway-to-service or service-to-database boundaries.

| Side | Components | Trust level |
|---|---|---|
| Source | Browser, public Internet, unauthenticated visitor, authenticated customer, or administrator | Untrusted |
| Boundary | nginx public listener in `amaranta-shop-api-gateway` | Enforcement point |
| Destination | `sales-api`, `catalog-api`, and `workflow` through named public routes | Internal platform |

The Gateway is configuration, not a business service: it routes public paths,
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
| S | A caller sends `Authorization: Bearer not-a-token` to `GET /api/v1/orders` or an admin Catalog route. The Gateway deliberately checks only that the header exists, so accepting it would depend on a missing or incorrect service verifier. | 2 | 3 | 6 | The Gateway blocks a missing header. Catalog independently verifies RS256 tokens in Brandow's PR #7; its routing suite proves a malformed token is forwarded for the owner to reject. | Every protected service must reject invalid signature, issuer, audience, expiry, and unauthorized role claims; retain negative tests per endpoint. |
| T | A caller uses `../`, `%2e%2e`, or encoded slashes below `/api/v1/products/` to reach `internal/products/reserve-stock` and alter stock outside the Sales workflow. | 1 | 3 | 3 | The Gateway uses exact locations or prefixes ending in `/`; its routing tests assert `404` for direct and encoded traversal attempts. Catalog's internal endpoints are documented in Brandow's PR #8. | Keep the boundary-bypass cases in Gateway CI; authenticate Sales-to-Catalog calls separately rather than publishing the route. |
| R | A caller disputes an order creation or status change and supplies an arbitrary but syntactically valid `X-Correlation-Id`, leaving no durable actor-to-order evidence at the edge. | 2 | 2 | 4 | The Gateway validates or generates the correlation identifier. Brandow's Catalog trace-context and log-contract work (#15, #16) preserves downstream trace evidence. | Sales must record order id, authenticated subject when present, idempotency key, action, outcome, and correlation id without logging tokens or bodies. |
| I | A browser from an untrusted origin reads a credentialed error response or correlation data because a new route bypasses the Gateway's CORS allow-list. | 2 | 3 | 6 | CORS headers are produced centrally and the current test suite proves an unknown origin receives no grant. | Keep origin allow-list and exposed-header assertions in CI for every new route; production origins must be explicit configuration, not a reflected request header. |
| D | A client floods public catalog reads or anonymous `POST /api/v1/orders`, exhausting upstream connections and preventing legitimate checkout. | 2 | 3 | 6 | nginx limits proxied requests to 100 per minute per source address with burst 40; the smoke suite verifies a `429` response. | Measure rejected and upstream requests, set a reviewed capacity target, and test that a Sales outage affects only Sales routes while Catalog remains available. |
| E | A customer presents a valid non-admin token to `/api/v1/admin/products/*`; the Gateway forwards any present credential and an incomplete Catalog role check would grant stock or product administration. | 2 | 3 | 6 | The Gateway rejects a missing credential; Brandow's Catalog PR #11 implements the `ADMIN` route guard. | Add an end-to-end Gateway-to-Catalog test that a valid non-admin token receives `403` for every mutation family. |

## Initial ranking

1. **S - risk 6:** service-side verification is the final authentication control after the Gateway's cheap header filter.
2. **I - risk 6:** CORS is central today, but a route-level exception could expose data to an untrusted browser origin.
3. **D - risk 6:** the limit is configured and tested, but its capacity and operational signal are not yet measured.
4. **E - risk 6:** Catalog has the `ADMIN` control; an end-to-end role regression is still needed at the public edge.
5. **R - risk 4:** trace correlation exists, but accountability requires a durable Sales audit event.
6. **T - risk 3:** the Gateway currently blocks the known traversal forms; regression coverage must remain mandatory.

## Residual risk

The boundary accepts guest checkout and public product browsing by product
design, so abuse cannot be eliminated by authentication; rate limiting and
service input validation contain it while stronger abuse controls are deferred.
The Gateway also does not parse or validate JWTs by design. That reduces edge
complexity but makes independent verification and role checks in Sales, Catalog,
and Workflow non-negotiable release gates.

## Evidence consulted

- `05-architecture/decisions/records/ADR-012-stack-of-extracted-services.md`
- `07-api/contracts/openapi/api-gateway.yaml`
- `amaranta-shop-api-gateway` route configuration and `tests/routing-checks.sh`
- Brandow's Catalog API PRs [#7](https://github.com/code-corhuila/amaranta-shop-catalog-api/pull/7), [#8](https://github.com/code-corhuila/amaranta-shop-catalog-api/pull/8), [#11](https://github.com/code-corhuila/amaranta-shop-catalog-api/pull/11), [#13](https://github.com/code-corhuila/amaranta-shop-catalog-api/pull/13), [#15](https://github.com/code-corhuila/amaranta-shop-catalog-api/pull/15), [#16](https://github.com/code-corhuila/amaranta-shop-catalog-api/pull/16), and [#17](https://github.com/code-corhuila/amaranta-shop-catalog-api/pull/17)

## References

The mandatory physical-library reference and the cited pages must be completed
by the author from the genuine library consultation record before this analysis
is considered complete.
