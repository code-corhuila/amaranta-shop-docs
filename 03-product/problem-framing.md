# Problem Framing — Problem Definition

---

## 1. The problem in one sentence

**Amaranta's staff and customers** who **manage pet purchases and clinic appointments through
phone calls, a paper agenda, and a manual counter register** struggle with **double-booked
appointments, forgotten confirmations, and no shared view of product stock between in-store and
future online sales** because **there is no single system tracking a pet, its owner, its
purchases, and its medical history together**, resulting in **wasted staff time resolving
scheduling conflicts and no digital record customers can return to**.

---

## 2. Affected users

| Segment | Description | Estimated size | Priority |
|---------|-------------|---------------|---------|
| Pet owners (customers) | People who buy products and/or bring their pets for clinical services at Amaranta | Local neighborhood customer base | High |
| Receptionist | Front-desk staff who currently juggles a phone, a paper agenda, and a manual register | 1 role, on-site every business hour | High |
| Veterinarians / Groomers | Clinical staff who need a reliable, conflict-free agenda | Small clinical team | High |
| Business owner (Admin) | Wants visibility into sales and appointment volume without manual reconciliation | 1 | Medium |

### Jobs-to-be-done (JTBD)

**When** I need to buy food or book a vaccination appointment for my pet,
**I want** to do it myself, at any hour, without calling and waiting for someone to check a
paper agenda,
**so that** I know immediately whether the slot or the product is actually available.

**When** a customer walks into the store,
**I want** the receptionist to serve them from the same system a remote customer would use,
**so that** there is one source of truth for stock and appointment availability, not two.

---

## 3. Evidence of the problem

| Evidence type | Source | Date | Key finding |
|--------------|--------|------|------------|
| Direct observation | Amaranta's current phone- and paper-based booking process | 2026-08 | Appointments are tracked on paper with no automatic conflict check; double-booking is possible whenever two people write on the same page |
| Direct observation | Manual counter register for shop sales | 2026-08 | No link between what is recorded as sold and how much stock physically remains, since online sales don't exist yet but are a near-term goal |

> This is an academic project modeling a real small business; formal user interviews and
> support-ticket data are not available. Evidence here is based on direct observation of how
> the business currently operates.

---

## 4. Current user solution (and its problems)

| Current solution | Limitations | Cost/Friction |
|-----------------|------------|--------------|
| Phone calls + paper agenda for appointments | No automatic conflict check, no confirmation record, error-prone under time pressure | Staff time resolving conflicts when two bookings overlap |
| Manual counter register for sales | No shared stock visibility across channels, no digital order history for a customer to check later | Manual reconciliation, no way to look up "what did I buy last month" |

---

## 5. Solution hypothesis

**We believe that** a single web platform where customers self-serve product purchases and
appointment booking — with the appointment logic enforcing real business hours and
double-booking prevention at the domain level, not just the UI —
**for** Amaranta's pet-owner customers and front-desk staff,
**will achieve** fewer scheduling conflicts, a shared and accurate stock count, and a digital
history customers and staff can both rely on.
**We will know we succeeded when** every appointment booked through the system respects
business hours and never collides with another booking for the same professional, and every
order placed reflects real, current stock.

---

## 6. Success metrics (North Star)

| Metric | Current baseline | 6-month target | How to measure it |
|--------|----------------|---------------|-------------------|
| Appointment double-booking incidents | Unknown (not tracked on paper) | 0 — enforced structurally by `AGGR-INV-APPT-002` | Domain invariant test coverage + manual QA |
| Orders with inaccurate stock at fulfillment | Unknown (no shared stock system exists) | 0 — enforced by the Catalog/Sales event flow | Manual QA of the `OrderCreated` → `StockReserved`/`StockDepleted` flow |
| Time to book an appointment | Phone call, variable wait | Self-service, under 1 minute | Manual UX walkthrough |

**North Star Metric:** Percentage of appointments and orders that complete without a manual
staff intervention to resolve a conflict or stock mismatch.

---

## 7. Hypothesis risks

| Risk | Probability | Impact | Experiment to validate |
|------|------------|--------|----------------------|
| Non-technical customers/staff find the self-service flow harder than a phone call | Medium | Medium | Usability walkthrough with the actual receptionist before final delivery |
| Business hours change seasonally and the hardcoded 07:00–12:00/14:00–18:00 rule becomes wrong | Low (for the MVP timeframe) | Medium | Keep `BusinessHours` as a single value object so the rule can move to configuration later without touching the invariant logic |

---

## 8. Out of scope (we do not solve)

- Online payment processing — out of MVP scope, see `01-context/scope.md`
- Multi-location scheduling — Amaranta operates a single physical location today
- Inventory purchasing/supplier management — this system tracks sellable stock, not
  supplier purchase orders

---

## Correlations

- Product vision → `03-product/vision.md`
- HUs that implement this solution → `04-requirements/user-stories.md`
