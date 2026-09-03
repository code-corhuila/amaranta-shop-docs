# Navigation Map

> Defines the screen structure of the system, how screens connect to each other, and what routes
> exist. It is the reference when frontend and backend discuss what endpoints exist
> or how to reach a feature.

> **Stage note:** Amaranta ships today as a **monolith MVP** (one NestJS backend, one Next.js
> frontend) implementing 3 of the 15+ HUs in the full backlog: HU-IAM-001 (register), HU-CAT-001
> (browse catalog), HU-SALES-001 (cart + checkout). There is no API Gateway or service discovery
> yet — the single frontend calls the single backend directly. Those pieces belong to the
> **target** microservices architecture (`09-microservices/`, ADR-004/ADR-005) and are added when
> a bounded context is actually extracted — see `amaranta-shop-mvp/docs/path-to-microservices.md`
> in the implementation repo for the extraction plan.

---

## Frontend route structure

```
/                           → Home / product catalog (public)
├── /login                  → Login form
├── /register                → Self-registration form (HU-IAM-001)
├── /cart                    → Cart + checkout wizard (HU-SALES-001)
│                              4 steps: Cart → Fulfillment → Details → Review
└── /orders                  → "My orders" — order history (authenticated)
```

There is no `/admin`, `/dashboard`, or role-gated area in the MVP: the only role that exists in
practice today is `CUSTOMER` (self-registration always assigns it — `INV-USER-002`). Roles
`RECEPTIONIST`, `VETERINARIAN`, `GROOMER`, `ADMIN` are modeled in the domain (`identity.users.role`)
but have no dedicated screens yet — they belong to HUs outside this MVP's 3-story scope
(HU-IAM-003, HU-SALES-003, HU-SCHED-*, HU-CLIN-*).

---

## Screen map

| Screen | Route | Public / Authenticated | Backend endpoint(s) |
|--------|-------|------------------------|----------------------|
| Catalog (home) | `/` | Public | `GET /products` |
| Login | `/login` | Public | `POST /auth/login` |
| Register | `/register` | Public | `POST /auth/register` |
| Cart & checkout | `/cart` | Public to browse; login required to confirm | `POST /orders` |
| My orders | `/orders` | Authenticated (redirects to `/login` if not) | `GET /orders` |

---

## Main user flows

### Flow 1 — Browse and buy (HU-CAT-001 → HU-SALES-001)

```
Catalog (/)
    │
    ▼ click "Agregar" on a product card
[fly-to-cart animation, cart badge updates on arrival]
    │
    ▼ click "Carrito"
Cart — step 1: Cart (/cart)
    │
    ├── not logged in ──► redirected to /login, then back to /cart
    │
    ▼ click "Continuar"
Cart — step 2: Fulfillment
    │  choose "A domicilio" or "Recoger en tienda"
    ▼
Cart — step 3: Details
    │  name, phone, (address if DELIVERY), payment method
    │
    ├── missing required field ──► inline validation error, stays on step 3
    │
    ▼ click "Continuar"
Cart — step 4: Review
    │
    ▼ click "Confirmar pedido"
    │
    ├── insufficient stock (checked against Catalog) ──► error shown, stays on step 4
    │
    ▼ success
Order confirmation screen
    │  shows order id, total, "Enviar pedido por WhatsApp" (pre-filled wa.me link)
    ▼
/orders (optional) or back to /
```

**Related HUs:** HU-CAT-001, HU-SALES-001

### Flow 2 — Registration and login (HU-IAM-001)

```
Catalog (/)
    │
    ▼ click "Crear cuenta"
Register (/register)
    │
    ├── email already registered ──► inline error "EMAIL_ALREADY_REGISTERED"
    │
    ▼ valid submission
JWT issued, stored client-side, redirected to /
```

```
Catalog (/)
    │
    ▼ click "Ingresar"
Login (/login)
    │
    ├── wrong credentials ──► inline error "INVALID_CREDENTIALS"
    │
    ▼ valid credentials
JWT issued, redirected to /
```

**Related HUs:** HU-IAM-001 (register). Login itself is supporting infrastructure for
HU-SALES-001 (checkout requires a JWT), not a separately counted HU in this MVP's 3-story scope.

---

## Navigation rules

| Rule | Description |
|------|--------------|
| Authentication | `/orders` redirects to `/login` if there is no session; `/cart`'s checkout step also redirects to `/login` when the user tries to continue past the cart step without one |
| Session storage | The JWT is kept client-side (browser storage); there is no server session — every request re-sends `Authorization: Bearer <token>` |
| Cart persistence | The cart is client-side (survives a page reload) and is independent of login — you can add products before authenticating |
| Out of stock | A product with `stockQuantity = 0` shows "Agotado" and its "Agregar" button is disabled — cannot enter the cart at all |
| Checkout validation | Each wizard step blocks "Continuar" until its required fields are valid; the delivery address is only required when "A domicilio" was chosen (`INV-ORDER-007`, enforced again in the backend domain, not only in this form) |
| 404 | Undefined routes fall back to Next.js's default not-found page |

---

## Correlations

- Design system (visual components) → `12-ux-ui/design-system.md`
- Wireframes → `12-ux-ui/wireframes.md`
- User stories these flows implement → `04-requirements/user-stories.md`
- Domain invariants referenced above → `02-domain/entities-and-rules.md`
- Path to the target microservices navigation (API Gateway in front of this same frontend) →
  `amaranta-shop-mvp/docs/path-to-microservices.md`
