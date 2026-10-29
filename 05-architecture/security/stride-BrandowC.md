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
| I | The answer of `reserve-stock` gives the product name, the price and the exact units left ("only N unit(s) left") to any caller | 1 | 1 | 1 | the same data is already public in `GET /api/v1/products` | none: accepted, see the residual risk |
| D | `items` has no maximum and `quantity` has no ceiling, and these calls skip the gateway rate limit and body limit; a loop can reserve every unit and the shop shows "Agotado" | 2 | 2 | 4 | reserve takes every line or none; `sales-api` declares request timeouts and answers 503 when Catalog is down | a maximum of items and quantity per call, a body-size limit and a rate limit on `/internal` |
| E | A foothold on the network gives the power over stock that only an `ADMIN` has through `PATCH /api/v1/admin/products/{id}/stock` | 1 | 3 | 3 | the admin routes require `ADMIN`; `/internal` is not routed | same root cause as S: authenticating the callers closes both |

## Ranking by risk

| Position | Letter | Risk |
|---|---|---|
| 1 | S | 6 |
| 2 | T | 6 |
| 3 | R | 4 |
| 4 | D | 4 |
| 5 | E | 3 |
| 6 | I | 1 |

S and T come first because each one lets a caller on the network change the stock. R and D follow with the same value; D goes
after R only because the audit trail is needed to detect the abuse that D describes.

## Residual risk

I accept the Information disclosure row (I, risk 1): the answer of `reserve-stock` shows the name, the price and the units
left, but `GET /api/v1/products` already shows the same data to anyone, so a caller learns nothing new. I also accept, for
now, that the Docker network is the only protection of these routes. The `platform` network is not reachable from outside and
the gateway never routes `/internal`, so the exposure needs a foothold inside the network first. That is acceptable for the
current demo environment, and not for a production one: S, T and E share one root cause, and the shared secret closes them.

## References

The physical library book consulted in class, with its pages, is added in a later commit.
