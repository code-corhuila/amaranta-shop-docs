# System Scope

---

## In Scope

### MVP Features

| # | Feature | Description | Responsible service |
|---|---------|-------------|---------------------|
| 1 | Account registration & login | Customers and staff register/authenticate with email + password; JWT issued with role claim | identity-service |
| 2 | Product catalog browsing | Product grid with photo, name, description, category, and price in COP | catalog-service |
| 3 | Shopping cart & checkout | Add/remove products, view cart total, place an order (online or in-store) | sales-service |
| 4 | Stock tracking | Stock is decremented on order confirmation; out-of-stock products cannot be added to the cart | catalog-service |
| 5 | Appointment booking | Book a clinic appointment (medical consult, vaccination, grooming/bath, surgery) inside business hours only | scheduling-service |
| 6 | Double-booking prevention | Two appointments cannot be assigned to the same professional at overlapping times | scheduling-service |
| 7 | Pet registration | A customer or receptionist registers a pet (species, breed, birth date) linked to an owner | clinical-service |
| 8 | Medical history | Vet records diagnosis/treatment notes per appointment, visible to the pet's owner | clinical-service |
| 9 | Order & appointment confirmations | Email notification sent when an order is placed or an appointment is booked/cancelled | notification-service |
| 10 | Admin / receptionist panel | Staff-only screens to register walk-in customers, sell products, and book appointments on their behalf | sales-service, scheduling-service (frontend routes gated by role) |
| 11 | Role-based access control | Five roles (`CUSTOMER`, `RECEPTIONIST`, `VETERINARIAN`, `GROOMER`, `ADMIN`) restrict what each screen and endpoint allows | identity-service (issued in JWT), enforced per service |

### Included integrations

| External system | Integration type | Purpose |
|----------------|-----------------|---------|
| SMTP provider (e.g. a transactional email service) | Outbound email | Order and appointment confirmations/reminders sent by `notification-service` |

> No payment gateway integration is in scope for the MVP — orders are marked `PAID` manually by
> staff at checkout (cash/card handled physically at the counter), since Amaranta operates from a
> single physical location today. Online payment is a future-version candidate.

### Environments being built

| Environment | Purpose |
|-------------|---------|
| Local | Development on the developer's machine via Docker Compose — this is the **only** environment built for this course delivery |
| Development (dev) | Future environment once the project moves beyond the academic MVP |
| Staging | Future environment |
| Production | Future environment |

---

## Out of Scope

| # | What is out of scope | Reason | Future version? |
|---|---------------------|--------|----------------|
| 1 | Online payment gateway (Stripe/PSE/Wompi) | Amaranta is a single physical location; MVP checkout is cash/card in person | Yes — once a delivery/shipping model exists |
| 2 | Delivery / shipping of products | No logistics operation exists yet; MVP is pickup-in-store only | Yes |
| 3 | Native mobile app | React web app covers both desktop and the in-store kiosk use case | Possible — H2, as a wrapper around the same APIs |
| 4 | Multi-location / multi-tenant support | Amaranta operates one physical location; multi-branch adds a whole context (inventory transfer, branch-scoped appointments) | Yes — if Amaranta opens a second location |
| 5 | Recurring/subscription orders (e.g. monthly food delivery) | Not part of the current business model | Possible |
| 6 | SMS notifications | Email is enough to demonstrate the async notification flow for the course; adding a second channel is not additional distributed-systems learning value | Possible |
| 7 | Push notifications / real-time chat with the vet | Out of the appointment + e-commerce domain this course project targets | Not planned |
| 8 | Automated CI/CD pipelines to a cloud environment | Everything stays local for this delivery, per explicit instruction — no deploy target exists yet | Yes — once the project leaves the academic phase |

### What another system / team handles (and why not us)

| Feature | Who builds it | Why not us |
|---------|--------------|-----------|
| Payment processing | A future payment gateway integration (Stripe/Wompi) | Out of MVP scope — no online payment yet |
| Accounting / tax reporting | Amaranta's existing accounting process | Outside the core domain of catalog + appointments |

---

## Scope assumptions

| # | Assumption | Consequence if false |
|---|-----------|---------------------|
| 1 | Amaranta operates a single physical location with one set of business hours (07:00–12:00 and 14:00–18:00) | The `Appointment` invariant that validates business hours would need to become location-aware |
| 2 | Initial catalog is under 500 products and appointment volume is under 50/day | The chosen NFR performance targets (see `04-requirements/non-functional.md`) remain realistic without extra caching/scaling work |
| 3 | Users access the system via a modern desktop or mobile browser (no legacy browser support required) | UI/animation choices (Framer Motion, CSS Grid) would need fallbacks |
| 4 | Prices are always in Colombian Pesos (COP), no multi-currency needed | Money value object and formatting logic would need to change |

---

## Constraints

| Type | Description |
|------|-------------|
| **Time** | Academic delivery for the Sistemas Distribuidos course — documentation phases (00–05) must be ready before any implementation starts |
| **Budget** | $0 — all technology choices must be free/open-source and runnable locally (Docker) |
| **Technology** | Must use React (frontend), Node.js (backend), a relational database with an easy-to-swap engine, a message broker, and Docker + Kubernetes, per course requirements |
| **Regulatory** | No formal regulatory certification required for the academic MVP; PII handling still follows the security policy (`00-governance/security-policy.md`) as good practice |
| **Team** | 1 developer acting as architect, designer, data analyst, and full-stack developer |

---

## External dependencies

| Dependency | Team / Provider | Required date | Status |
|-----------|----------------|--------------|--------|
| Docker Desktop / Docker Engine on the development machine | Local environment | Before Sprint 1 implementation | 🟢 Available |
| SMTP test account (e.g. Mailtrap or a free provider) for `notification-service` | External provider | Before the notifications HU is implemented | 🟡 In progress |

---

## Correlations

- Vision and roadmap → `03-product/vision.md`
- Term glossary → `01-context/glossary.md`
- System overview → `01-context/overview.md`
- Scope-related risks → `15-project-control/risks.md`
