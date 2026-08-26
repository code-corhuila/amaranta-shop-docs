# User Stories — Backlog

---

## Backlog status

| Cut | Sprint | Total HUs | Refined | In progress | Completed |
|-----|--------|-----------|---------|-------------|-----------|
| Cut 1 — Core MVP | Sprint 1-4 | 13 | 13 | 0 | 0 |
| Cut 2 — Notifications & polish | Sprint 5-6 | 3 | 3 | 0 | 0 |

---

## Epics

| ID | Epic | Description |
|----|------|-------------|
| EP-001 | Identity & Access | Registration, login, role-based staff creation |
| EP-002 | Catalog & Sales | Product browsing, cart, checkout, in-store sales |
| EP-003 | Scheduling | Appointment booking with business-hours and double-booking rules |
| EP-004 | Clinical Records | Pet registration and medical history |
| EP-005 | Notifications | Order/appointment confirmation and cancellation emails |

---

## User Stories

### HU-IAM-001 — Customer self-registration {#HU-IAM-001}

**Epic:** EP-001

> **As** a visitor
> **I want** to create my own account with name, email, phone, and password
> **so that** I can buy products and book appointments under my own identity

**Acceptance Criteria:**

```gherkin
Scenario 1: Successful registration
  Given I am not registered
  When  I submit a valid name, unique email, and password (min. 8 characters)
  Then  a User with role CUSTOMER is created
  And   I receive a JWT access token

Scenario 2: Duplicate email
  Given a user already exists with email "ana@example.com"
  When  I try to register with that same email
  Then  the system rejects the request with error "EMAIL_ALREADY_REGISTERED"
```

| Field | Value |
|-------|-------|
| Story Points | 3 |
| Priority | Must Have |
| Target sprint | Sprint 1 |
| Status | Backlog |
| Dependencies | — |
| Affected service(s) | identity-service |

---

### HU-IAM-002 — Login {#HU-IAM-002}

**Epic:** EP-001

> **As** a registered user (customer or staff)
> **I want** to log in with my email and password
> **so that** I can access the features allowed by my role

**Acceptance Criteria:**

```gherkin
Scenario 1: Successful login
  Given I am registered with email "ana@example.com" and a valid password
  When  I submit correct credentials
  Then  I receive a JWT containing my userId and role, valid for 1 hour

Scenario 2: Wrong password
  Given I am registered
  When  I submit an incorrect password
  Then  the system responds 401 with error "INVALID_CREDENTIALS"
  And   the attempt is logged per `00-governance/security-rules.md` (A07)
```

| Field | Value |
|-------|-------|
| Story Points | 2 |
| Priority | Must Have |
| Target sprint | Sprint 1 |
| Status | Backlog |
| Dependencies | HU-IAM-001 |
| Affected service(s) | identity-service |

---

### HU-IAM-003 — Receptionist registers a walk-in customer {#HU-IAM-003}

**Epic:** EP-001

> **As** a receptionist
> **I want** to register a walk-in customer's basic info without them creating a password
> **so that** I can immediately sell them a product or book their appointment in person

**Acceptance Criteria:**

```gherkin
Scenario 1: Quick walk-in registration
  Given I am logged in as RECEPTIONIST
  When  I submit a name and phone for a new walk-in customer
  Then  a User with role CUSTOMER is created without requiring a password
  And   I can immediately use that customer's id to create an order or appointment

Scenario 2: Non-receptionist cannot use this endpoint
  Given I am logged in as CUSTOMER
  When  I call the walk-in registration endpoint
  Then  the system responds 403 "FORBIDDEN"
```

| Field | Value |
|-------|-------|
| Story Points | 3 |
| Priority | Must Have |
| Target sprint | Sprint 3 |
| Status | Backlog |
| Dependencies | HU-IAM-001 |
| Affected service(s) | identity-service |

---

### HU-CAT-001 — Browse product catalog {#HU-CAT-001}

**Epic:** EP-002

> **As** any visitor
> **I want** to browse products with their photo, name, description, and price in COP
> **so that** I can decide what to buy

**Acceptance Criteria:**

```gherkin
Scenario 1: List active products
  Given there are active products in the catalog
  When  I open the shop
  Then  I see a paginated grid (max 50 per page) showing photo, name, price formatted as COP, and category

Scenario 2: Out-of-stock product
  Given a product has stockQuantity = 0
  When  I view it
  Then  it is shown as "out of stock" and cannot be added to the cart
```

| Field | Value |
|-------|-------|
| Story Points | 3 |
| Priority | Must Have |
| Target sprint | Sprint 1 |
| Status | Backlog |
| Dependencies | — |
| Affected service(s) | catalog-service |

---

### HU-CAT-002 — Admin manages the catalog {#HU-CAT-002}

**Epic:** EP-002

> **As** an admin
> **I want** to create, edit, deactivate, and adjust the stock of a product
> **so that** the shop always reflects what Amaranta actually has for sale

**Acceptance Criteria:**

```gherkin
Scenario 1: Create a product
  Given I am logged in as ADMIN
  When  I submit a valid product (name, description, category, price > 0, image)
  Then  the product is created with the given stock and isActive = true

Scenario 2: Reject invalid price
  Given I am logged in as ADMIN
  When  I submit a product with price = 0
  Then  the system rejects it with "INV-PRODUCT-001"
```

| Field | Value |
|-------|-------|
| Story Points | 3 |
| Priority | Must Have |
| Target sprint | Sprint 1 |
| Status | Backlog |
| Dependencies | HU-IAM-002 |
| Affected service(s) | catalog-service |

---

### HU-SALES-001 — Add to cart and checkout {#HU-SALES-001}

**Epic:** EP-002

> **As** a customer
> **I want** to add products to a cart and submit them as an order
> **so that** I can buy what I need without calling the store

**Acceptance Criteria:**

```gherkin
Scenario 1: Successful checkout
  Given my cart has 2 units of a product with enough stock
  When  I submit checkout
  Then  an Order is created with status PENDING and channel ONLINE
  And   an OrderCreated event is published

Scenario 2: Insufficient stock at checkout
  Given my cart has more units than are currently in stock
  When  I submit checkout
  Then  the order is rejected (or created and then auto-cancelled via StockDepleted)
  And   I see a clear message naming which product ran out
```

| Field | Value |
|-------|-------|
| Story Points | 5 |
| Priority | Must Have |
| Target sprint | Sprint 2 |
| Status | Backlog |
| Dependencies | HU-CAT-001, HU-IAM-002 |
| Affected service(s) | sales-service, catalog-service |

---

### HU-SALES-002 — View my order history {#HU-SALES-002}

**Epic:** EP-002

> **As** a customer
> **I want** to see my past and current orders
> **so that** I can track what I've bought and its status

**Acceptance Criteria:**

```gherkin
Scenario 1: View own orders only
  Given I have 3 orders
  When  I open "My orders"
  Then  I see exactly those 3 orders, never another customer's orders
```

| Field | Value |
|-------|-------|
| Story Points | 2 |
| Priority | Should Have |
| Target sprint | Sprint 2 |
| Status | Backlog |
| Dependencies | HU-SALES-001 |
| Affected service(s) | sales-service |

---

### HU-SALES-003 — Receptionist creates an in-store order {#HU-SALES-003}

**Epic:** EP-002

> **As** a receptionist
> **I want** to add products to a cart and check out on behalf of a walk-in customer, then mark the order as paid
> **so that** in-store sales use the same stock and order records as online sales

**Acceptance Criteria:**

```gherkin
Scenario 1: In-store sale
  Given I am logged in as RECEPTIONIST and have selected an existing customer
  When  I check out a cart on their behalf
  Then  an Order is created with channel IN_STORE and createdBy = my userId

Scenario 2: Mark order as paid
  Given a PENDING order exists
  When  I mark it as PAID at the counter
  Then  its status changes to PAID and an OrderPaid event is published
```

| Field | Value |
|-------|-------|
| Story Points | 5 |
| Priority | Must Have |
| Target sprint | Sprint 3 |
| Status | Backlog |
| Dependencies | HU-SALES-001, HU-IAM-003 |
| Affected service(s) | sales-service |

---

### HU-SCHED-001 — Book an appointment within business hours {#HU-SCHED-001}

**Epic:** EP-003

> **As** a customer
> **I want** to book an appointment for my pet, choosing service type, date, and time
> **so that** I get a confirmed slot without calling the clinic

**Acceptance Criteria:**

```gherkin
Scenario 1: Valid booking
  Given I choose a date/time between 07:00-12:00 or 14:00-18:00
  When  I submit the booking
  Then  an Appointment is created with status PENDING
  And   an AppointmentScheduled event is published

Scenario 2: Outside business hours
  Given I choose 13:00 (lunch break) or 19:00 (after closing)
  When  I submit the booking
  Then  the system rejects it with "INV-APPT-001: appointment must be between 07:00-12:00 or 14:00-18:00"
```

| Field | Value |
|-------|-------|
| Story Points | 5 |
| Priority | Must Have |
| Target sprint | Sprint 2 |
| Status | Backlog |
| Dependencies | HU-CLIN-001, HU-IAM-002 |
| Affected service(s) | scheduling-service, clinical-service |

---

### HU-SCHED-002 — Prevent double-booking {#HU-SCHED-002}

**Epic:** EP-003

> **As** the clinic
> **I want** the system to reject an appointment that overlaps another appointment for the same professional
> **so that** no veterinarian or groomer is ever double-booked

**Acceptance Criteria:**

```gherkin
Scenario 1: Overlapping booking rejected
  Given Dr. Ruiz has a CONFIRMED appointment from 09:00 to 09:30
  When  someone tries to book Dr. Ruiz from 09:15 to 09:45
  Then  the system rejects it with "INV-APPT-002: professional already booked in this time range"

Scenario 2: Back-to-back booking allowed
  Given Dr. Ruiz has an appointment from 09:00 to 09:30
  When  someone books Dr. Ruiz from 09:30 to 10:00
  Then  the booking succeeds (adjacent slots do not overlap)
```

| Field | Value |
|-------|-------|
| Story Points | 5 |
| Priority | Must Have |
| Target sprint | Sprint 2 |
| Status | Backlog |
| Dependencies | HU-SCHED-001 |
| Affected service(s) | scheduling-service |

---

### HU-SCHED-003 — Cancel my appointment {#HU-SCHED-003}

**Epic:** EP-003

> **As** a customer
> **I want** to view and cancel my upcoming appointments
> **so that** I can free the slot if my plans change

**Acceptance Criteria:**

```gherkin
Scenario 1: Cancel a future appointment
  Given I have a PENDING appointment scheduled for tomorrow
  When  I cancel it
  Then  its status changes to CANCELLED and an AppointmentCancelled event is published

Scenario 2: Cannot cancel someone else's appointment
  Given another customer's appointment exists
  When  I try to cancel it
  Then  the system responds 403 "FORBIDDEN"
```

| Field | Value |
|-------|-------|
| Story Points | 2 |
| Priority | Should Have |
| Target sprint | Sprint 3 |
| Status | Backlog |
| Dependencies | HU-SCHED-001 |
| Affected service(s) | scheduling-service |

---

### HU-SCHED-004 — Receptionist manages appointments for a customer {#HU-SCHED-004}

**Epic:** EP-003

> **As** a receptionist
> **I want** to book, cancel, or reassign an appointment on behalf of a customer, and let professionals mark their own appointments complete
> **so that** walk-in and phone-assisted booking uses the exact same rules as self-service booking

**Acceptance Criteria:**

```gherkin
Scenario 1: Receptionist books for a customer
  Given I am logged in as RECEPTIONIST and have selected an existing customer and pet
  When  I book an appointment on their behalf inside business hours
  Then  the same INV-APPT-001/002 invariants apply exactly as they would for a self-service booking

Scenario 2: Professional completes an appointment
  Given I am logged in as VETERINARIAN and have a CONFIRMED appointment happening now
  When  I mark it as COMPLETED
  Then  an AppointmentCompleted event is published
```

| Field | Value |
|-------|-------|
| Story Points | 5 |
| Priority | Must Have |
| Target sprint | Sprint 3 |
| Status | Backlog |
| Dependencies | HU-SCHED-001, HU-SCHED-002, HU-IAM-003 |
| Affected service(s) | scheduling-service |

---

### HU-CLIN-001 — Register a pet {#HU-CLIN-001}

**Epic:** EP-004

> **As** a customer (or a receptionist on their behalf)
> **I want** to register my pet's basic information
> **so that** I can book appointments for it and build its medical history

**Acceptance Criteria:**

```gherkin
Scenario 1: Register a pet
  Given I am logged in as CUSTOMER
  When  I submit a pet's name, species, and optional breed/birth date
  Then  a Pet is created linked to my userId as ownerId

Scenario 2: Invalid weight
  Given I am registering or updating a pet
  When  I submit weightKg = -1
  Then  the system rejects it with "INV-PET-001"
```

| Field | Value |
|-------|-------|
| Story Points | 2 |
| Priority | Must Have |
| Target sprint | Sprint 2 |
| Status | Backlog |
| Dependencies | HU-IAM-002 |
| Affected service(s) | clinical-service |

---

### HU-CLIN-002 — Veterinarian adds a medical record {#HU-CLIN-002}

**Epic:** EP-004

> **As** a veterinarian
> **I want** to record the diagnosis and treatment after a completed appointment
> **so that** the pet's medical history stays accurate over time

**Acceptance Criteria:**

```gherkin
Scenario 1: Add a medical record after completion
  Given an Appointment I performed is COMPLETED
  When  I submit a diagnosis and treatment
  Then  a MedicalRecord is created, linked to that appointment and pet, and cannot be edited afterward (only amended with a new entry)

Scenario 2: Cannot record for a non-completed appointment
  Given an Appointment is still PENDING
  When  I try to add a medical record for it
  Then  the system rejects it with "appointment must be COMPLETED first"
```

| Field | Value |
|-------|-------|
| Story Points | 3 |
| Priority | Must Have |
| Target sprint | Sprint 4 |
| Status | Backlog |
| Dependencies | HU-SCHED-004 |
| Affected service(s) | clinical-service |

---

### HU-CLIN-003 — Owner views their pet's medical history {#HU-CLIN-003}

**Epic:** EP-004

> **As** a customer
> **I want** to view my pet's full medical history
> **so that** I have a reliable record across visits and veterinarians

**Acceptance Criteria:**

```gherkin
Scenario 1: View own pet's history
  Given my pet has 2 medical records
  When  I open its profile
  Then  I see both records in chronological order

Scenario 2: Cannot view another owner's pet
  Given another customer's pet exists
  When  I try to access its medical history
  Then  the system responds 403 "FORBIDDEN"
```

| Field | Value |
|-------|-------|
| Story Points | 2 |
| Priority | Should Have |
| Target sprint | Sprint 4 |
| Status | Backlog |
| Dependencies | HU-CLIN-002 |
| Affected service(s) | clinical-service |

---

### HU-NOTIF-001 — Confirmation and cancellation emails {#HU-NOTIF-001}

**Epic:** EP-005

> **As** a customer
> **I want** to receive an email when my order or appointment is created or cancelled
> **so that** I have written confirmation without checking the app

**Acceptance Criteria:**

```gherkin
Scenario 1: Order confirmation
  Given an OrderCreated event is published
  When  notification-service consumes it
  Then  a confirmation email is sent to the customer's email within 30 seconds

Scenario 2: Idempotent delivery
  Given the same OrderCreated event is redelivered by RabbitMQ (at-least-once)
  When  notification-service consumes it again
  Then  only one email is actually sent (checked against the delivery log by eventId)
```

| Field | Value |
|-------|-------|
| Story Points | 3 |
| Priority | Should Have |
| Target sprint | Sprint 5 |
| Status | Backlog |
| Dependencies | HU-SALES-001, HU-SCHED-001 |
| Affected service(s) | notification-service |

---

## Rules for writing HUs

See `04-requirements/README.md` for the general rules (specific role, benefit ≠ action,
verifiable ACs, one HU = one unit of value).

---

## Correlations

- Full template with DoD checklist → `04-requirements/_template-hu.md`
- Functional requirements these HUs implement → `04-requirements/functional.md`
- Non-functional requirements → `04-requirements/non-functional.md`
- Traceability matrix → `04-requirements/traceability-matrix.md`
