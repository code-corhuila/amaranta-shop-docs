# Domain Map — Bounded Contexts

---

## 1. Domain overview

```
Amaranta runs a single physical location that combines a veterinary clinic ("Amaranta la
Clínica") and a pet products store ("Amaranta Shop"). A customer can register their pets,
buy products, and book clinic appointments — either from home, or in person, where a
receptionist uses the same system on their behalf. The system must keep appointment slots
inside real business hours, prevent double-booking a professional, keep product stock
consistent between online and in-store sales, and remember each pet's medical history
across visits.
```

---

## 2. Identified Bounded Contexts

### Bounded Context: Identity & Access

| Field | Value |
|-------|-------|
| **Name** | Identity & Access |
| **Responsibility** | Owns the `User` aggregate (customers and staff), authentication, and role assignment |
| **Owning team** | Full-stack developer (solo project) |
| **Microservice(s)** | `identity-service` |
| **Database** | PostgreSQL |
| **Ubiquitous Language** | User, Role, Credential, Access Token |

**Context-specific terms (Ubiquitous Language):**

| Term | Meaning in THIS context | Different in another context? |
|------|------------------------|-------------------------------|
| User | Any registered person: customer or staff member | Yes — Sales calls this same person "Customer"; Scheduling calls a staff member with role VETERINARIAN or GROOMER a "Professional" |
| Role | One of `CUSTOMER`, `RECEPTIONIST`, `VETERINARIAN`, `GROOMER`, `ADMIN` | No |

---

### Bounded Context: Catalog

| Field | Value |
|-------|-------|
| **Name** | Catalog |
| **Responsibility** | Owns the `Product` aggregate: what is for sale, its price in COP, and its stock level |
| **Owning team** | Full-stack developer (solo project) |
| **Microservice(s)** | `catalog-service` |
| **Database** | PostgreSQL |
| **Ubiquitous Language** | Product, Category, Stock, Price |

**Context-specific terms:**

| Term | Meaning in THIS context | Different in another context? |
|------|------------------------|-------------------------------|
| Product | An item for sale, with its own price and stock | In Sales, a product referenced inside an order is a frozen snapshot (`OrderItem`), not the live Catalog record |

---

### Bounded Context: Sales

| Field | Value |
|-------|-------|
| **Name** | Sales |
| **Responsibility** | Owns the `Order` aggregate: cart-to-checkout flow, order totals, and order status, for both online and in-store (receptionist-assisted) purchases |
| **Owning team** | Full-stack developer (solo project) |
| **Microservice(s)** | `sales-service` |
| **Database** | PostgreSQL |
| **Ubiquitous Language** | Order, OrderItem, Cart, Checkout, Channel |
| **Subdomain type** | **Core** — the shop's conversion experience (cart, checkout, the "fly to cart" interaction) is where Amaranta Shop differentiates |

---

### Bounded Context: Scheduling

| Field | Value |
|-------|-------|
| **Name** | Scheduling |
| **Responsibility** | Owns the `Appointment` aggregate: booking, business-hours validation, and conflict-free assignment of a professional to a time slot |
| **Owning team** | Full-stack developer (solo project) |
| **Microservice(s)** | `scheduling-service` |
| **Database** | PostgreSQL |
| **Ubiquitous Language** | Appointment, Time Slot, Business Hours, Service Type |
| **Subdomain type** | **Core** — the appointment-hours and double-booking logic is the clinic's hardest, most valuable business rule |

---

### Bounded Context: Clinical Records

| Field | Value |
|-------|-------|
| **Name** | Clinical Records |
| **Responsibility** | Owns the `Pet` aggregate and the `MedicalRecord` aggregate: pet identity and its accumulated medical history |
| **Owning team** | Full-stack developer (solo project) |
| **Microservice(s)** | `clinical-service` |
| **Database** | PostgreSQL |
| **Ubiquitous Language** | Pet, Patient, Species, Medical Record, Diagnosis, Treatment |
| **Subdomain type** | **Core** — the medical history is Amaranta's most valuable long-term data asset |

---

### Bounded Context: Notifications

| Field | Value |
|-------|-------|
| **Name** | Notifications |
| **Responsibility** | Consumes domain events from Sales and Scheduling and sends confirmation/reminder emails. Owns no business data of its own beyond a delivery log. |
| **Owning team** | Full-stack developer (solo project) |
| **Microservice(s)** | `notification-service` |
| **Database** | PostgreSQL (thin — delivery log only, for idempotency and audit) |
| **Ubiquitous Language** | Notification, Delivery, Template |
| **Subdomain type** | **Generic** — a commodity capability, deliberately kept simple |

---

## 3. Context Map

```
┌────────────────────┐   REST (validate    ┌────────────────────┐
│   Sales            │   price & stock)     │   Catalog          │
│   (Core)           │──────────────────────▶│   (Supporting)    │
│                     │◀──────────────────────│                    │
└─────────┬───────────┘  event: StockReserved,└────────────────────┘
          │              StockDepleted
          │ event: OrderCreated, OrderPaid, OrderCancelled
          │
          ▼
┌────────────────────┐
│  Notifications      │◀── event: AppointmentScheduled, AppointmentCancelled ──┐
│  (Generic)          │                                                        │
└────────────────────┘                                                        │
                                                                                │
┌────────────────────┐   REST (verify pet exists)   ┌────────────────────┐    │
│   Scheduling        │──────────────────────────────▶│  Clinical Records │────┘
│   (Core)            │◀──────────────────────────────│  (Core)           │
└─────────┬───────────┘   event: AppointmentCompleted └────────────────────┘
          │
          │  JWT (published by Identity, verified at the Gateway)
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Identity & Access (Generic)                 │
│           Open Host Service — issues JWT consumed by all         │
└─────────────────────────────────────────────────────────────────┘
```

### Context relationship types

| Type | Symbol | Description | Example |
|------|--------|-------------|---------|
| **Upstream → Downstream** | `U → D` | U provides, D consumes. D depends on U. | Catalog → Sales |
| **Open Host Service** | `OHS` | U publishes a published protocol | Identity → everyone (JWT) |
| **Published Language** | `PL` | Explicit shared language | Domain events (`02-domain/domain-events.md`) |

### Relationships table

| Context A | Relationship | Context B | Communication channel | Contract |
|-----------|-------------|-----------|----------------------|---------|
| Sales | D ← U | Catalog | REST (sync, validate price/stock at checkout) | OpenAPI |
| Catalog | U → D | Sales | Event (`StockReserved`, `StockDepleted`) | AsyncAPI-style event schema |
| Scheduling | D ← U | Clinical Records | REST (sync, verify the pet exists before booking) | OpenAPI |
| Scheduling | U → D | Clinical Records | Event (`AppointmentCompleted`) | Event schema |
| Sales, Scheduling | U → D | Notifications | Event (`OrderCreated`, `OrderPaid`, `AppointmentScheduled`, `AppointmentCancelled`) | Event schema |
| Identity | OHS | All contexts | JWT (validated at the Kong API Gateway, re-checked per service) | JWT claims contract |

---

## 4. Core Domain, Supporting, Generic

| Type | Description | Investment | Example |
|------|-------------|-----------|---------|
| **Core Domain** | Where Amaranta's competitive advantage lies | MAXIMUM — build, don't buy | Sales, Scheduling, Clinical Records |
| **Supporting Subdomain** | Necessary for the core but not differentiating | MEDIUM | Catalog |
| **Generic Subdomain** | Commodity, off-the-shelf solution exists | MINIMUM — buy/use OSS in a real company, built simply here for the course | Identity & Access, Notifications |

### Classification of this project's bounded contexts

| Bounded Context | Type | Justification |
|----------------|------|---------------|
| Sales | Core | The cart-to-checkout experience (and its "fly to cart" animation) is the shop's conversion driver |
| Scheduling | Core | Enforcing real business hours and preventing double-booking is the clinic's hardest, most business-critical rule |
| Clinical Records | Core | A pet's accumulated medical history is Amaranta's long-term data asset and trust driver |
| Catalog | Supporting | Necessary to run the shop, but a plain product/stock list is not differentiating by itself |
| Identity & Access | Generic | Authentication is a solved problem (a real company would use Auth0/Keycloak); built simply here to keep the course project self-contained |
| Notifications | Generic | Email delivery is a commodity capability; kept intentionally thin |

---

## 5. Modeling decisions

- **Event Storming session:** Solo design session, 2026-08-25, using the domain vocabulary already spoken by Amaranta's staff (appointments, orders, pets, products)
- **Tool used:** Written directly into this document
- **Map iterations:** v1 (2026-08-25) — initial 6-context split

### Key decisions and discarded alternatives

| Decision | Discarded alternative | Reason |
|----------|----------------------|--------|
| Split Scheduling and Clinical Records into two contexts | One combined "Clinic" context | Scheduling is about *time slots and availability*; Clinical Records is about *medical history*. They evolve at different rates and a receptionist needs Scheduling without needing write access to medical notes |
| Split Catalog and Sales into two contexts | One combined "Shop" context | Catalog owns product truth (price, stock); Sales owns transactional state (an order). Database-per-service would otherwise force stock and orders into the same schema, coupling two very different write patterns |
| Keep `User` in a single Identity context instead of a per-context "Customer"/"Professional" duplicate | Duplicating user data into Sales/Scheduling | A person is one identity across the whole business; duplicating it would create sync problems for a two-person team to maintain |

---

## 6. How to update this map

1. Before adding a new microservice, verify whether it belongs to an existing bounded context.
2. If a context's ubiquitous language is changing, review whether the context should be split.
3. Run a mini Event Storming session every time the domain changes significantly (e.g. adding a second Amaranta location).
4. The context map MUST be synchronized with the C4 system-level diagram (`05-architecture/overview.md`).

> **Important correlation:** The bounded contexts in this document →
> Microservices in `09-microservices/service-catalog.md` →
> C4 diagrams in `08-uml/` →
> Service separation ADRs in `05-architecture/decisions/`
