# STRIDE - Gateway and Sales APIs to Catalog API

## Scope

This analysis owns the inbound API boundary of `amaranta-shop-catalog-api`.
It covers the Gateway-routed product and administrator routes plus the Sales
calls that reserve and release stock. It does not analyze the browser-to-Gateway
or Catalog-to-MongoDB boundaries.

| Side | Components | Trust level |
|---|---|---|
| Source | `amaranta-shop-api-gateway` on public product/admin routes; `amaranta-shop-sales-api` on stock routes | Distinct public and internal callers |
| Boundary | Catalog HTTP adapter and its authentication/authorization filter | Enforcement point |
| Destination | Catalog Product aggregate and Catalog repository port | Catalog bounded context |

The Gateway is configuration, not an authorization decision point: it makes the
public route available and filters a missing credential on protected paths.
Catalog owns the decisive checks: valid RS256 JWT for administrator operations,
`ADMIN` authorization for mutations, Product invariants, and stock reservation
semantics. The two `/internal/products/*` routes are deliberately absent from
the Gateway; today their trust is the platform network, which is a declared
residual risk until a service credential is implemented.

## Data and entry points

| Asset crossing this boundary | Classification | Entry point | Current control |
|---|---|---|---|
| Catalog product name, price, and availability | Public | `GET /api/v1/products*` from Gateway | Catalog exposes active products only |
| Inactive products, product mutation, and stock set | Restricted | `/api/v1/admin/products*` from Gateway | Catalog must verify JWT and `ADMIN` |
| Stock reservation/release lines | Restricted | `/internal/products/reserve-stock`, `/internal/products/release-stock` from Sales | Not published by Gateway; Product aggregate enforces non-negative stock |
| JWT claims and correlation identifier | Internal | Protected admin requests and service calls | Catalog verifies token; HTTP adapter carries correlation id |
| Product records and stock quantity | Confidential at rest | MongoDB through Catalog repository adapter | `catalog_app` owns only the Catalog database |

The internal endpoints are entry points for Sales, not for a browser. Gateway
tests prove a browser cannot reach them through direct, traversal, or
encoded-path variants; the missing service-to-service credential remains a
Catalog API risk rather than a reason to call them public.

## STRIDE - first threats

Risk is likelihood (L) multiplied by impact (I), both on a 1-3 scale.

| Type | Concrete threat | L | I | Risk | Existing control | Missing mitigation / verification |
|---|---|---:|---:|---:|---|---|
| S | A process on the platform network sends a forged request to `/internal/products/reserve-stock`, impersonating Sales because the current internal endpoint has no service credential. | 2 | 3 | 6 | The Gateway never publishes the path; Catalog runs on the private platform network. | Authenticate Sales with a short-lived service token or mTLS and reject internal calls without the verified service identity. |
| T | A caller manipulates reservation quantities or repeats a request so stock becomes inconsistent or negative. | 2 | 3 | 6 | The Product aggregate validates quantity and Catalog uses an atomic conditional decrement; the database validator is a final defense. | Bind a reservation identity to an idempotent Sales operation and reject malformed or replayed internal requests with a named test. |
| R | Sales cannot prove which service action reserved or released stock when an internal request later needs investigation. | 2 | 2 | 4 | Correlation id reaches Catalog and HTTP errors use one envelope. | Record reservation id, caller service identity, affected product ids, outcome, and correlation id without logging any token. |
| I | A public Catalog error or inactive-product lookup discloses internal stock, validation details, or administrative product state. | 2 | 2 | 4 | Public reads return active products; errors pass through the Catalog error adapter. | Keep `details` safe for public responses and verify inactive products return `404` to non-admin callers. |
| D | Public reads or high-concurrency reservations exhaust Catalog's MongoDB pool and deny stock checks to Sales. | 2 | 3 | 6 | Gateway limits public traffic; Catalog declares MongoDB pool and socket limits and has concurrent reservation tests. | Add overload evidence for internal calls and set an explicit capacity alert before enabling Workflow retries. |
| E | A valid customer token reaches `/api/v1/admin/products/*` and a weak Catalog role filter permits product or stock mutation. | 2 | 3 | 6 | Gateway requires a header and Catalog owns the `ADMIN` decision. | Prove `403` for every non-admin mutation family and run the negative authorization suite in Catalog CI. |

## Initial ranking

1. **S - risk 6:** network reachability is not a verifiable Sales identity for a stock mutation.
2. **T - risk 6:** reservation correctness protects the Product aggregate and every order that depends on it.
3. **D - risk 6:** public and internal callers can compete for the same Catalog capacity.
4. **E - risk 6:** a valid token is insufficient without the Catalog `ADMIN` policy.
5. **R - risk 4:** correlation alone does not identify the service action that changed stock.
6. **I - risk 4:** public read and error contracts must remain deliberately minimal.

## Residual risk

The boundary accepts that the Gateway blocks browser access to `/internal/*`,
but this does not authenticate Sales to Catalog. Network-only trust is retained
only while the Workflow and Security services are still skeletons; it is a P0
release blocker for an end-to-end checkout. Public product browsing remains
unauthenticated by product design, so rate limits, MongoDB timeouts, and bounded
queries must contain abuse without turning a catalogue read into a login flow.

## Evidence consulted

- `05-architecture/decisions/records/ADR-012-stack-of-extracted-services.md`
- `amaranta-shop-catalog-api` README, API contract, and authentication tests
- `amaranta-shop-api-gateway` route configuration and `tests/routing-checks.sh`
- `amaranta-shop-sales-api` README and Catalog stock adapter contract

## References

The mandatory physical-library reference and the cited pages must be completed
by the author from the genuine library consultation record before this analysis
is considered complete.
