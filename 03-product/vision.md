# Product Vision

---

## Vision statement

**For** pet owners and staff at Amaranta,
**who** need to buy pet products and book clinic appointments without phone calls or paper
agendas,
**the** Amaranta platform
**is a** unified web application combining a pet store and a veterinary clinic booking system
**that** lets a customer browse products, buy them, register their pets, and book a real,
conflict-free appointment — the exact same way whether they're at home or being helped by a
receptionist in-store.
**Unlike** running the shop's register and the clinic's paper agenda as two separate, disconnected
processes,
**our product** gives every pet, order, and appointment one consistent digital record, with
appointment-hours and double-booking rules enforced by the system itself, not by whoever is
holding the pen.

---

## Team mission

Give a small, real veterinary + pet-shop business the same quality of software experience a
much larger company would build — a fast, vivid, trustworthy digital storefront and booking
system — while using the project as a hands-on demonstration of Domain-Driven Design and
microservices for the Sistemas Distribuidos course.

---

## Strategic pillars

| Pillar | Description | Success metrics |
|--------|-------------|----------------|
| Reliability of the schedule | Business hours and double-booking rules are enforced as domain invariants, not UI validation, so they can never be silently bypassed | 0 double-booking incidents (see `03-product/problem-framing.md`) |
| One unified experience | Shop and clinic feel like one branded product, not two apps bolted together, and staff use the exact same screens customers do | Single design system (`12-ux-ui/design-system.md`), shared auth/cart/session across both modes |
| Distributed-systems correctness | Cross-service consistency (stock vs. orders, appointments vs. medical records) is handled through domain events and idempotent consumers, matching real microservices practice | All 6 bounded contexts deployed as independent services, communicating via REST (sync) + RabbitMQ (async) per the context map |

---

## High-level roadmap

```
Sprint 1-2 ──── Sprint 3-4 ──── Sprint 5-6 ──── Beyond MVP
     │               │               │               │
  [MVP core]   [Admin/POS panel] [Polish & QA]  [Payments, delivery,
  Catalog,      Receptionist-      Notifications,  multi-location]
  Sales,        assisted flows     full RBAC
  Scheduling,
  Clinical
```

| Horizon | Period | Objective | Epics / Features | Uncertainty |
|---------|--------|----------|----------------|-------------|
| H1 (Now) | Sprint 1–4 | Ship the MVP: browse/buy products, register pets, book appointments, staff-assisted flows | EP-001 Catalog & Sales, EP-002 Scheduling, EP-003 Clinical Records, EP-004 Admin/Receptionist Panel | Low |
| H2 (Next) | Sprint 5–6 | Polish the unified UX (cart animation, brand system), finish notifications and RBAC end-to-end | EP-005 Notifications, EP-006 Design System | Low |
| H3 (Later) | Post-course | Payments, delivery, multi-location support | Future-version candidates from `01-context/scope.md` | High |

---

## Product principles

1. **Domain rules live in the domain, not the UI.** Business hours, double-booking, and stock
   invariants are enforced in the aggregate itself — a UI bug or a future integration must
   never be able to create an impossible appointment or an inconsistent order.

2. **One brand, two modes.** The shop and the clinic share the same login, the same cart/booking
   session, and the same visual language — a customer should never feel like they left the app.

3. **Staff use the product, they don't get a separate back office.** The receptionist panel
   reuses the same Sales and Scheduling APIs a customer uses — no duplicated business logic
   between "customer-facing" and "internal" code paths.

---

## Product Definition of Done

**Objective:** Deliver a working documented design (phases 00–05) and, subsequently, a locally
runnable MVP that lets a customer buy a product and book an appointment end-to-end, and lets a
receptionist do the same on their behalf.

| Key Result | Baseline | Target | Date |
|------------|---------|--------|------|
| KR1: SDD phases 00–05 filled in and reviewed | Empty scaffold | Complete for Amaranta | Sprint 1 (2026-08-28) |
| KR2: Core user flows implemented (buy product, book appointment, staff-assisted flow) | 0 | 3/3 flows working locally via Docker Compose | End of MVP sprints |
| KR3: Zero double-booking or stock-mismatch defects in manual QA | N/A | 0 | Before final delivery |

---

## Correlations

- Problem framing (the why) → `03-product/problem-framing.md`
- Backlog that implements the vision → `04-requirements/user-stories.md`
