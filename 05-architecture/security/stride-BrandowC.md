# STRIDE - Sales to Catalog internal stock endpoints

Author: Brandow Claros (@BrandowC). Claim issue: #34.

## Boundary

The trust boundary between `sales-api` and `catalog-api` on the `platform` Docker network. `sales-api` reserves stock with
`POST /internal/products/reserve-stock` and gives it back with `POST /internal/products/release-stock`. The gateway never
routes `/internal`, and these routes ask for no credential: decision OD-3 of `specs/checkout.md` trusts the Docker network.
The security policy (`00-governance/security-policy.md`, principle 5) asks for Zero Trust inside the internal network, so
this boundary is where the two documents disagree.

## Components on each side

- Caller side: `sales-api` (`CatalogStockHttpAdapter`) and its PostgreSQL schema `sales`.
- Called side: `catalog-api` (`internal.py`, `ReserveStockService`, `ReleaseStockService`) and its MongoDB `catalog.product`.

## Data that crosses

A stock reservation (`productId`, `quantity`) from Sales to Catalog; the reserved lines (`productName`, `unitPriceAmount`,
`quantity`) back; the `X-Correlation-Id` header.

## Entry points

| Entry | Caller | Authentication today |
|---|---|---|
| `POST /internal/products/reserve-stock` | `sales-api` | none, only the Docker network |
| `POST /internal/products/release-stock` | `sales-api` | none, only the Docker network |

## STRIDE

Risk = probability (1-3) x impact (1-3).

| | Threat | P | I | Risk | Control that exists | Missing mitigation |
|---|---|---|---|---|---|---|
| S | Any container on the `platform` network can call the internal routes as if it were `sales-api`, because no token and no secret is checked | 2 | 3 | 6 | the gateway never publishes `/internal` (404, covered by the gateway smoke test) | a shared-secret header checked by `catalog-api` (the upgrade named in OD-3) |
| T | `release-stock` credits any `productId` with any `quantity`, unconditionally and not tied to a reservation; a caller can inflate a stock and a replayed request credits it twice | 2 | 3 | 6 | `quantity` is a whole number from 1 (`StrictInt`), `extra=forbid`, a product only once per request | a reservation id with an idempotent release (the upgrade named in the spec) and a ceiling per call |
| R | `catalog-api` cannot say which caller or which order moved a stock: the body carries neither | 2 | 2 | 4 | the `X-Correlation-Id` travels on the call and is on every access-log line of `catalog-api` | log the caller and the reservation id on every reserve and release |
