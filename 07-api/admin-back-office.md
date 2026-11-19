# Admin back office: every order, every account, and a menu for each role

> Records what was added so that the administrator can run the shop: see **who placed each order** and **who has
> registered**, and be offered a menu that fits their role. It is written after the fact, from what was built and
> tested, and says plainly what it does not do.

| | |
|---|---|
| **Status** | Built and verified in the real stack (develop). Not yet promoted to `qa` / `main`. |
| **Author** | Brandow Claros |
| **Touches** | `amaranta-shop-sales-api`, the identity module of the monolith, `amaranta-shop-api-gateway`, `amaranta-shop-front`, `amaranta-shop-sales-portal` |
| **Does not touch** | `amaranta-shop-catalog-api`, any database schema (`sales-db`, `catalog-db`) |

## 1. The problem

The administrator could manage the catalog and nothing else. The checkout spec (`specs/checkout.md`) gives the
administrator "nothing in this capability", and `sales-api-decisions.md` (D-S2) already warned that an admin-facing "all
orders" endpoint "would need real pagination". Nobody could answer the two questions a shop owner asks first: *who ordered?*
and *who signed up?* The header also offered every person the same links, including a "Ventas" link to a visitor who had
nothing to look at there.

## 2. What was added

### 2.1 Two read-only endpoints, both for the `ADMIN` role

| Endpoint | Owner | Answers |
|---|---|---|
| `GET /api/v1/admin/orders?page&limit` | `sales-api` | Every order of the shop, newest first |
| `GET /api/v1/admin/users?page&limit` | identity (the monolith, served at `/auth/users`) | Every registered account, newest first |

Both return `{data, meta: {page, limit, total, totalPages}}`, `limit` is 1 to 100 (20 by default), and an invalid value is a
`400` naming the field. The gateway only checks that a credential **exists** (`401` envelope otherwise); the owning service
checks that it is an administrator's and answers `403` to anyone else (norm 5.3.7: each service validates for itself).

**An order says who placed it.** Next to what the customer already sees, the administrator gets `customerId`,
`registeredCustomer` and `units`. `registeredCustomer` is `false` for a visitor who bought without an account and for a
counter sale: both belong to the walk-in customer (spec DP-1), so `contactName` and `contactPhone` are the only identity there
is. For a registered customer the name shown is the one given at checkout, **not** looked up in identity: Sales knows a customer
by id only, and joining across services would add a runtime dependency that no other call has (ADR-010).

**An account never shows a secret.** `UserView` carries `userId`, `fullName`, `email`, `phone`, `role` and `createdAt`; the
password hash is never mapped, and a test fails if it ever appears in the answer.

### 2.2 Gateway routes (`nginx/routes/admin.conf`)

Two **exact** paths, so nothing else under `/api/v1/admin` is routed from there. `/api/v1/admin/users` is mapped to the
`identity` alias at `/auth/users` and keeps the page asked for; identity's own CORS headers are dropped on that route (a
browser refuses an answer with two `Access-Control-Allow-Origin` values). The rest of the monolith's `/auth` (staff list,
customer search, walk-in registration) is still not published.

### 2.3 The shell: a menu for each person

| Who | Header offers | Why |
|---|---|---|
| Visitor | Catálogo, **Iniciar sesión** (a pill with the brand gradient) and the floating cart | Browsing and buying need no account |
| Customer | Catálogo, **Mis pedidos**, a "Cliente" badge, Cerrar sesión | Their own history |
| Administrator | Catálogo, **Administración**, an "Administrador" badge, Cerrar sesión | They run the shop; they do not buy from a cart |

`/admin` is the front door: three cards (catalog, orders, users), each with how many there are. A count that cannot be had
says "No se pudo contar" instead of a wrong number. `/admin/users` (shell) and `/sales/admin/orders` (sales portal) are the two
new screens. All are behind `RequireRole`: a customer who types the address is turned away **without a single request**, and a
visitor is sent to sign in. An administrator who signs in lands on `/admin`.

The orders screen shows, for each order, **what it asked for** (every product with how many), who bought, how it is
delivered and paid, its status and its total. The users screen shows name, email, phone, role and the day.

**The phone is collected at sign-up.** The users screen listed a phone for nobody because the form never asked for one.
Creating an account now offers "Celular (opcional)": ten digits, no spaces or signs, sent only when given. Identity refuses
anything else with a `400` (`phone must be exactly 10 digits`), because the column holds ten characters and a longer value used
to fail in the database as a `500`. An account created before this change has no phone and shows "—".

## 3. Decisions

| # | Decision | Why | Cost |
|---|---|---|---|
| A-1 | Orders are listed by Sales, accounts by identity, each in its own service | One owner per datum (ADR-010); no cross-service join | The orders screen shows the name given at checkout, not the account's name |
| A-2 | Both lists are paginated and bounded (max 100), unlike `GET /orders` (D-S2) | D-S2 said an all-orders list "is not exempt by this precedent" | Two more parameters to validate |
| A-3 | `/api/v1/admin/...` for both | Same prefix as the catalog's administration | None |
| A-4 | Users are served by the monolith's identity module for now | ADR-011 point 9: until the security service exists, identity stays in the monolith | When Security is extracted only the `identity` alias in the gateway changes, not the route file nor any client |
| A-5 | The role check is repeated in the service and the screen | The screen spares a person a page that cannot work; the service is the one that protects | Two places say "ADMIN" |

## 4. How it was proven

| What | Where | Result |
|---|---|---|
| The use case is bounded, newest first, over every customer | `OrderQueriesTest` (core) | pass |
| PostgreSQL and the in-memory repository return the same thing | `OrderRepositoryContract` (the PostgreSQL run is the integration test) | in-memory pass; PostgreSQL verified through the running service |
| Only `ADMIN` reads; a customer, a receptionist and no token do not; pages are validated | `AdminOrdersHttpTest` (sales-api) | pass |
| The users use case never leaks a hash and refuses an unbounded list | `list-users.use-case.spec.ts` (identity) | pass |
| Only the two exact paths are routed, with the page, the `401`, and one CORS header | `tests/routing-checks.sh` (gateway) | pass |
| Each role sees its own menu; a customer is turned away from `/admin*` with no request | `App.test.tsx`, `Admin.test.tsx` (shell), `AdminOrdersPage.test.tsx` (portal) | pass |
| The whole thing in a browser, through a public HTTPS URL | real stack | administrator sees 2 orders and 5 users; a customer sees "Mis pedidos" and is refused `/admin` |

## 5. What this does not do

- **No search or filter** on either list: newest first and pages. A shop with thousands of orders will want them.
- **It cannot mark an order as paid or change anything.** Both screens only read. Marking an order paid stays with the
  receptionist (spec DP-5).
- **The walk-in account is listed.** `Cliente de mostrador` is the placeholder account that owns visitor and counter orders; it
  is real data, so it appears among the users.
- **Staff accounts are not created from the web.** The demo administrator is created by the real-stack script with a random
  password; creating staff from the administration is future work.
- **The orders screen cannot open one order's detail**, only its summary.
- **`sales-api` on `develop` publishes an outbox that the stack's database did not have**, so for the local demo the change was
  re-applied on the version of `sales-api` the stack runs. The pull request itself targets `develop`.

## 6. Related

`specs/checkout.md` (DP-1, DP-5), `07-api/sales-api-decisions.md` (D-S2), `07-api/authentication.md` (roles),
ADR-009 (RS256, each service validates), ADR-010 (a datum has one owner), ADR-011 (identity until the security service).
