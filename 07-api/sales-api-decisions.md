# Sales API — closed decisions and declared gaps (Brandow Claros)

> **Author:** Brandow Stivent Claros Polania (`BrandowC`) — individual contribution, Week 08.
> Scope: `07-api/`, Sales bounded context only.

> **What this is.** `sales-service.yaml` describes the *shape* of the Sales endpoints. This
> document closes the decisions that shape alone does not answer — what error a client actually
> gets, and why — verified against `apps/api/src/sales/` and `apps/api/src/shared/`, not against
> what `guidelines.md` says should happen. Where the two disagree, that disagreement is recorded
> here as a gap, not silently resolved.
>
> **Scope.** Sales only (`POST /orders`, `POST /orders/in-store`, `POST /orders/{id}/mark-paid`,
> `GET /orders`). Verified by reading `checkout.use-case.ts`, `order.entity.ts`,
> `mark-order-as-paid.use-case.ts`, `catalog-stock.adapter.ts`, `reserve-stock.use-case.ts` (in
> Catalog, called in-process), `domain-exception.filter.ts`, `main.ts`, `roles.guard.ts`,
> `jwt-auth.guard.ts` and `optional-jwt-auth.guard.ts` in the `amaranta` code repo, `develop`
> branch, working tree as of `2026-09-24` — **not yet committed there** at the time of writing.
> The guest-checkout guard, `ShopHours`, and the other behavior this document describes do not
> exist yet at any pushed commit (`a3962e0` is the latest one on `origin/develop`); this document
> should not be merged before that code lands on `amaranta`, and its citation should be updated
> to the real commit hash once it does.

---

## 1. There are three error response pipelines today, not one

`guidelines.md` documents a single `ErrorResponse` — `{error, message, details}` — and
`_shared.yaml` gives it a schema. The code does not produce that shape for any error that touches
Sales. Three different pipelines answer, and none of them match it.

### 1.1 Class-validator rejects the request body (`ValidationPipe`)

`main.ts` registers `new ValidationPipe({ whitelist: true, transform: true })` with **no**
`exceptionFactory`. Nest's own default shape is returned as-is:

```json
{ "statusCode": 400, "message": ["quantity must not be less than 1"], "error": "Bad Request" }
```

`message` is an **array of strings**, one per failing field-rule — not the `string` the
`ErrorResponse` schema declares. There is no `error` code a client can branch on other than the
literal string `"Bad Request"`, and no `details` array.

### 1.2 A domain rule is violated (`DomainExceptionFilter`)

Every `DomainException` thrown anywhere the request reaches — inside `Order.checkout()`,
`Order.markAsPaid()`, or `ReserveStockUseCase` in Catalog, called in-process from
`CatalogStockAdapter` — is caught by one global filter and always answered as **400**:

```json
{ "statusCode": 400, "code": "INV-PRODUCT-002", "message": "\"Dog food 3kg\" only has 2 unit(s) left (requested 5)" }
```

The field is `code`, not `error`; there is no `details`; and — this is the part worth stopping
on — **it is always 400**, even when the violation is not a validation problem at all:

- `markAsPaid()` on an order that is not `PENDING` (`INV-ORDER-005`) is a **state conflict**, not
  bad input — the same request body would have succeeded an hour earlier.
- `PRODUCT_NOT_FOUND` inside `ReserveStockUseCase` — the product does not exist or was
  deactivated — is a **missing resource**, not bad input.

`guidelines.md` itself lists `409` for "Conflict (e.g. email already registered)". The filter does
not use it for anything. This is a real inconsistency, not a documentation oversight — see
[G-1](#g-1--domain-conflicts-and-missing-resources-are-reported-as-400).

### 1.3 Nest's own built-in exceptions — auth, role, and "not found" all share one shape

Three different failures go through three different guards/use-cases, but they all end up as the
same structural shape, because none of them build a custom body — they just call a Nest built-in
exception with a message (or none) and let Nest's default `HttpException` serialization handle it:

- **401** — `JwtAuthGuard` (`passport-jwt`'s own `handleRequest`) on a missing or invalid token.
- **403** — `RolesGuard` (`roles.guard.ts`), `throw new ForbiddenException(...)`, when the caller's
  role isn't in the endpoint's required list.
- **404** — `MarkOrderAsPaidUseCase` (`mark-order-as-paid.use-case.ts`), `throw new
  NotFoundException(...)` directly — **not** a `DomainException`, so it never reaches the filter
  from §1.2.

```json
{ "statusCode": 404, "message": "Order \"3f9e...\" was not found", "error": "Not Found" }
```

`error` here is the HTTP status's own reason phrase ("Unauthorized", "Forbidden", "Not Found") —
**not empty**, and not the same `error` field `ErrorResponse` describes (that one is meant to be a
stable machine-readable code, e.g. `VALIDATION_ERROR`; this one is just English text tied to the
status code).

**Consequence for anyone integrating against Sales today:** branch on `statusCode`, not on a
stable error code — `error` (§1.3), `code` (§1.2), and the absence of either (§1.1's `message`
array) are three different contracts, not one.

---

## 2. Six decisions, closed against the code as it runs today

### D-S1 · `POST /orders` accepts anonymous checkout — and silently degrades a bad token to anonymous

**Decision.** `POST /orders` is guarded by `OptionalJwtAuthGuard`, not `JwtAuthGuard`. A request
with no token succeeds; the order is attributed to `WALK_IN_CUSTOMER_ID`, the same sentinel
customer used for in-store sales. A request with a token that verifies attributes the order to
that customer instead, so it appears under "my orders" (`GET /orders`).

**Verified, not assumed: an invalid token is not the same as a missing one — except here.**
`OptionalJwtAuthGuard.handleRequest` (`optional-jwt-auth.guard.ts`) receives whatever
`passport-jwt` produces on failure and does `return user || undefined` — it never re-throws. An
expired token, a malformed one, and no token at all all reach the use-case the same way:
`user === undefined`. This is the **only** place in the controller where that's true; every other
guarded route on `JwtAuthGuard` alone turns the same failure into a 401.

**Why it's a real decision, not an oversight.** `sales.controller.ts` line 32 chooses
`OptionalJwtAuthGuard` deliberately over the `JwtAuthGuard` every other write endpoint in this
controller uses — someone had to pick that guard on purpose. Whether the silent degrade on an
*invalid* (vs. absent) token was the same deliberate choice, or a side effect of reusing the guard
as-is, is not something the code alone can answer — see [G-5](#g-5--an-expired-token-on-checkout-fails-silently-not-loudly).

**Consequence, and the gap it exposes.** `sales-service.yaml` currently declares
`security: [bearerAuth: []]` at the document root with no per-operation override on `POST
/orders`, which — read on its own — says a token is required. It is not. `authentication.md` §
"Enforcement" lists the public endpoints as `POST /auth/register`, `POST /auth/login`, and every
`GET /health` — `POST /orders` is missing from that list even though it behaves the same way.
**Fixed in this PR:** `sales-service.yaml` now overrides `security` on `POST /orders` to
`[{}, {bearerAuth: []}]` (either no auth or a valid token), and `authentication.md` now lists it.

### D-S2 · `GET /orders` is deliberately unpaginated

**Decision.** `ListMyOrdersUseCase` returns every order for the calling customer, sorted
newest-first, as a plain array — no `page`/`limit`, no `PaginatedMeta`.

**Why.** This is not the catalog-wide listing `guidelines.md`'s pagination convention targets. The
result set is bounded by construction: one customer's own purchase history. Paginating a
collection that is realistically in the tens, not the thousands, would add a `PaginatedMeta` no
client needs and no test would ever exercise past page 1.

**Scope of this decision.** It applies to `GET /orders` specifically, not to Sales as a whole — a
future admin-facing "all orders" endpoint would need real pagination and is not exempt by this
precedent.

### D-S3 · `deliveryAddress` is required twice, and the DTO answers first

**Decision.** Both layers check it: `CheckoutRequestDto` (`@ValidateIf` +
`@IsNotEmpty` when `fulfillmentMethod === DELIVERY`) and `Order.checkout()`
(`INV-ORDER-007`). For any request that goes through the HTTP controller, the DTO layer answers
first — the 400 a client actually sees is the shape from [§1.1](#11-class-validator-rejects-the-request-body-validationpipe),
with message `"deliveryAddress is required when fulfillmentMethod is DELIVERY"`, not the
`DomainException` shape from §1.2.

**Why the domain check isn't redundant to delete.** `Order.checkout()` is the aggregate's public
factory — any future caller that isn't `CheckoutRequestDto` (a message consumer, a script, a
different adapter) still gets the invariant enforced. The DTO validator only protects the one
entry point that has it today.

### D-S4 · Whitespace-only contact fields are accepted by the DTO and rejected by the domain

**Decision, verified, not assumed.** `class-validator`'s `@IsNotEmpty()` checks string length, not
content — `"   "` (three spaces) passes it. `Order.checkout()` then does
`input.contactName.trim()` and, finding it empty, throws `INV-ORDER-006`. So `contactName: "   "`
**does** reach the domain layer and **is** rejected — as the §1.2 shape (400, `code:
"INV-ORDER-006"`), not the §1.1 shape a reviewer skimming only the DTO would expect.

**Why this is worth writing down.** It is the one case in this controller where the "DTO answers
first" assumption from D-S3 does not hold, and it would be easy to assume otherwise without
reading both files side by side.

### D-S5 · Store hours gate both channels, on server wall-clock time

**Decision.** `AGGR-INV-ORDER-008` — no order, online or in-store — can be created outside
`ShopHours` (08:00–12:00 and 14:00–18:00, Monday–Saturday; closed all day Sunday).
`ShopHours.isWithinShopHours` reads `Date.getHours()`/`getDay()` — the **server process's** local
time zone, with no explicit `TZ` handling anywhere in `shop-hours.vo.ts`.

**Why closing this matters.** It is currently correct only because the one environment this has
been verified in runs with the store's own time zone. It is not a bug today; it is an assumption
that becomes a bug the day this container runs anywhere else — see
[G-3](#g-3--shophours-has-no-explicit-time-zone).

### D-S6 · A rejected checkout gives its stock reservation back — best-effort, not transactional

**Decision.** `CheckoutUseCase` calls `ProductStockPort.reserve()` (decrements stock in Catalog,
in-process) **before** building and saving the `Order`, because the order's lines need the
product name and price that only Catalog can answer. Everything that can fail between that
reservation and the saved order — `Order.checkout()` rejecting the request (`INV-ORDER-003`,
`-006`, `-007`, `-008`) or `orderRepository.save()` throwing — is caught, and the reservation is
returned through `ProductStockPort.release()` before the error propagates. A rejected checkout
leaves every product's stock unchanged. Publishing `OrderCreated` is **outside** that block on
purpose: once the order is saved its stock is spoken for, so a failing event bus must not hand
the units back.

**Why it was not always like this.** The first version had no `release`: the three
`Order.checkout()` rejections above — the likeliest ones, a closed store on a Sunday among them —
silently consumed the stock of every line of the rejected request. Found by reading the use case
while writing `specs/checkout.md`, covered now by the five tests under *"a rejected checkout leaves
the stock of every product unchanged"* in `checkout.use-case.spec.ts`.

**What it still is not.** The two calls — reserve in Catalog, save in Sales — are separate
transactions, so this is a compensation, not atomicity. If `release()` itself fails, or the process
dies between the reservation and the save, the units stay reserved with no order behind them. That
residue is what `path-to-microservices.md` already declares as the Saga / outbox work this MVP
defers — see [G-4](#g-4--a-reservation-can-still-outlive-its-order-d-s6).

---

## 3. Endpoint error tables (as verified, not as intended)

### `POST /orders` · checkout (public or authenticated — see D-S1)

| Code | When | Body shape |
|---|---|---|
| 400 | Missing/invalid field (`items` empty, `quantity < 1`, malformed `productId`, missing `deliveryAddress` on a `DELIVERY` order) | §1.1 — Nest `ValidationPipe` default |
| 400 | `AGGR-INV-ORDER-008` — store closed right now | §1.2 — `code: "INV-ORDER-008"` |
| 400 | `AGGR-INV-ORDER-003/004/006` — duplicate product line, empty item list past DTO validation, or whitespace-only contact fields (D-S4) | §1.2 |
| 400 | `PRODUCT_NOT_FOUND` — a `productId` doesn't exist or the product is inactive | §1.2 — **not a 404**, see [G-1](#g-1--domain-conflicts-and-missing-resources-are-reported-as-400) |
| 400 | `INV-PRODUCT-002` — insufficient stock for a requested quantity | §1.2 — **not a 409**, see G-1 |

### `POST /orders/in-store` · receptionist-assisted checkout

Same table as above, plus:

| Code | When | Body shape |
|---|---|---|
| 401 | No token, or a token that fails verification | §1.3 — `NestDefaultErrorResponse` (`JwtAuthGuard`) |
| 403 | Token belongs to a non-`RECEPTIONIST` role | §1.3 — `NestDefaultErrorResponse` (`RolesGuard`) |

### `POST /orders/{orderId}/mark-paid`

| Code | When | Body shape |
|---|---|---|
| 400 | Order exists but isn't `PENDING` (`INV-ORDER-005`) | §1.2 — **semantically a 409**, see G-1 |
| 401 | No token, or a token that fails verification | §1.3 |
| 403 | Non-`RECEPTIONIST` role | §1.3 |
| 404 | No order with that id | §1.3 — thrown directly, not via `DomainException` |

### `GET /orders`

| Code | When | Body shape |
|---|---|---|
| 401 | No token, or a token that fails verification | §1.3 |

No pagination — see D-S2.

---

## 4. Gaps declared, with an owner

Everything above is decided and closed. What follows is what would still need to change in the
*code* for the contract to be internally consistent — none of it is ambiguous, all of it is a
task without an owner yet.

### G-1 · Domain conflicts and missing resources are reported as 400

`DomainExceptionFilter` cannot tell a validation failure from a state conflict
(`INV-ORDER-005`) or a missing resource (`PRODUCT_NOT_FOUND`) apart — it only knows it caught a
`DomainException`. Fixing this means either giving `DomainException` a `kind` (`VALIDATION` |
`CONFLICT` | `NOT_FOUND`) the filter can switch on, or splitting it into subclasses. **Owner:
unassigned** — it's a `shared/` change, so it belongs to whoever picks up cross-cutting
infrastructure next, not to Sales alone; Catalog's `PRODUCT_NOT_FOUND` hits the same filter.

### G-2 · No shared schema for the shapes in §1.1–§1.3

`_shared.yaml#ErrorResponse` documents a shape nothing in Sales actually returns. This PR does
**not** change `_shared.yaml` to match, because the honest fix depends on G-1 first — reshaping
the filter's output is a bigger, cross-service change than one PR under a same-day deadline should
make silently. What this PR *does* do: `sales-service.yaml` now points at the real shapes
documented in §1, instead of at `_shared.yaml#ErrorResponse`, so the contract stops promising a
body no endpoint sends. **Owner: unassigned.**

### G-3 · `ShopHours` has no explicit time zone

`shop-hours.vo.ts` should take a zone (config or fixed to the store's) instead of reading the
process's local time. **Owner: unassigned.**

### G-4 · A reservation can still outlive its order (D-S6)

Narrowed, not closed: the ordinary rejections now release the reservation (D-S6), so what is left
is the unrecoverable remainder — `release()` failing, or the process dying between reserve and
save. Tracked here for Sales specifically; the general version of this gap is already declared in
`path-to-microservices.md` under "What this MVP deliberately does not solve yet" — Saga /
distributed transactions. **Owner: unassigned**, and correctly so until the team commits to
extracting `sales-service`/`catalog-service` — building a Saga for a call that is still
in-process would be solving a problem this stage of the project doesn't have yet.

### G-5 · An expired token on checkout fails silently, not loudly

A customer whose session expired mid-cart, or who sends a corrupted token by accident, gets a
**201 Created** — a *successful* anonymous checkout — instead of a 401 telling them to log back
in. Their order is real, but it is attached to the walk-in customer, not to their account: it will
never show up under `GET /orders` for them, with no error at any point indicating why. This is a
direct product-facing consequence of D-S1, worth a deliberate answer (re-validate the token
explicitly and 401 on a *present-but-invalid* one, while still allowing a *missing* one) rather
than inheriting whatever `OptionalJwtAuthGuard` happens to do. **Owner: unassigned.**

---

## 5. What this PR changes, concretely

- `authentication.md` — documented `POST /orders`'s optional auth and the silent-degrade-to-anonymous
  behavior on an invalid token (D-S1, G-5).
- `_shared.yaml` — added three schemas for the shapes verified in §1
  (`ValidationErrorResponse`, `DomainErrorResponse`, `NestDefaultErrorResponse`), replacing the
  generic `ErrorResponse` nothing in Sales actually returns.
- `sales-service.yaml` — per-operation `security` override on `POST /orders` (optional auth);
  every error response across all four Sales operations now points at the real schema from §1
  instead of the generic one, with a description explaining why; added the 400 row on `mark-paid`
  (§3) that was missing from the contract entirely.
- This document.

Nothing in `apps/api/src/` changes in this PR — this is a documentation contract catching up to
already-running behavior, not a behavior change. G-1 through G-4 are exactly the items that
*would* require a code change, and are left open on purpose.
