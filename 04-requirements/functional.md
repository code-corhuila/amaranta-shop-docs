# Functional Requirements

> Each FR belongs to exactly one microservice (owning service), traces back to a User Story in
> `04-requirements/user-stories.md`, and is checked off in `04-requirements/traceability-matrix.md`.

| ID | Module | Description | Source (HU) | Priority |
|----|--------|-------------|------------|---------|
| FR-001 | identity-service | The system must allow a visitor to self-register as a `CUSTOMER` with name, email, phone, and password | HU-IAM-001 | High |
| FR-002 | identity-service | The system must allow a registered user to log in and receive a JWT containing their role | HU-IAM-002 | High |
| FR-003 | identity-service | The system must allow a `RECEPTIONIST` or `ADMIN` to register a walk-in customer or staff member with an explicit role | HU-IAM-003 | High |
| FR-004 | catalog-service | The system must let any visitor browse the product catalog with photo, name, description, category, and price in COP, paginated | HU-CAT-001 | High |
| FR-005 | catalog-service | The system must let an `ADMIN` create, edit, deactivate, and adjust stock for a product | HU-CAT-002 | High |
| FR-006 | catalog-service | The system must decrement product stock when it consumes an `OrderCreated` event, and restore it on `OrderCancelled` | HU-CAT-002 | High |
| FR-007 | sales-service | The system must let a `CUSTOMER` build a cart and submit it as an order (checkout) | HU-SALES-001 | High |
| FR-008 | sales-service | The system must let a `CUSTOMER` view their own order history | HU-SALES-002 | Medium |
| FR-009 | sales-service | The system must let a `RECEPTIONIST` create an order on behalf of a walk-in customer (channel `IN_STORE`) | HU-SALES-003 | High |
| FR-010 | sales-service | The system must let `RECEPTIONIST`/`ADMIN` mark an order as `PAID`, `FULFILLED`, or `CANCELLED` | HU-SALES-003 | High |
| FR-011 | scheduling-service | The system must let a `CUSTOMER` book an appointment (service type, professional, date/time) only inside business hours (07:00–12:00, 14:00–18:00) | HU-SCHED-001 | High |
| FR-012 | scheduling-service | The system must reject an appointment that overlaps an existing appointment for the same professional | HU-SCHED-002 | High |
| FR-013 | scheduling-service | The system must let a `CUSTOMER` view and cancel their own upcoming appointments | HU-SCHED-003 | Medium |
| FR-014 | scheduling-service | The system must let a `RECEPTIONIST` book or cancel an appointment on behalf of a customer | HU-SCHED-004 | High |
| FR-015 | scheduling-service | The system must let a `VETERINARIAN`/`GROOMER` view their own agenda and mark an appointment as `COMPLETED` | HU-SCHED-004 | Medium |
| FR-016 | clinical-service | The system must let a `CUSTOMER` or `RECEPTIONIST` register a pet (species, breed, birth date) linked to an owner | HU-CLIN-001 | High |
| FR-017 | clinical-service | The system must let a `VETERINARIAN` add a medical record entry (diagnosis, treatment, notes) tied to a completed appointment | HU-CLIN-002 | High |
| FR-018 | clinical-service | The system must let a `CUSTOMER` view their own pet's medical history | HU-CLIN-003 | Medium |
| FR-019 | notification-service | The system must send a confirmation email when an order is created or an appointment is scheduled | HU-NOTIF-001 | Medium |
| FR-020 | notification-service | The system must send a notification email when an order or an appointment is cancelled | HU-NOTIF-001 | Low |

---

## Correlations

- Non-functional requirements → `04-requirements/non-functional.md`
- User Stories → `04-requirements/user-stories.md`
- Traceability → `04-requirements/traceability-matrix.md`
- Domain events these FRs rely on → `02-domain/domain-events.md`
