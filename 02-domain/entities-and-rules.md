# Entities, Value Objects, and Business Rules

> **Stack note:** Amaranta implements this in Node.js + TypeScript. Full folder structure and
> examples: [`_stacks/node-typescript.md`](../_stacks/node-typescript.md).

---

## Tactical DDD concepts

See `05-architecture/hexagonal-architecture.md` for the general definitions of Entity, Value
Object, Aggregate, and Business Rule. Below is how they apply specifically to Amaranta.

---

## System entities

### Entity: User

**Context:** Identity & Access

**Description:** A person registered in the system — either a customer or a staff member.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| id | UUID | Unique identifier | Yes | Auto-generated on creation |
| fullName | string | Person's full name | Yes | 2–100 characters |
| email | Email (VO) | Login identifier | Yes | Unique across all users |
| phone | string | Contact phone | No | Colombian format, 10 digits |
| passwordHash | string | bcrypt hash, cost ≥ 12 | Yes | Never returned in any API response |
| role | UserRole (enum) | `CUSTOMER \| RECEPTIONIST \| VETERINARIAN \| GROOMER \| ADMIN` | Yes | Defaults to `CUSTOMER` on self-registration; only `ADMIN` can assign other roles |
| createdAt | DateTime | Creation date | Yes | Immutable, set on creation |
| updatedAt | DateTime | Last modification | Yes | Updated automatically |

**Lifecycle / States:** `User` has no state machine — it is either active or, in a future
version, `DEACTIVATED` (out of MVP scope).

**Invariants (Business rules that MUST ALWAYS hold):**

```
INV-USER-001: Unique email
  - Rule: No two users may share the same email address
  - Violation: DomainException on registration with a duplicate email
  - Implementation: Unique constraint at the database + check in the use case

INV-USER-002: Self-registration is always CUSTOMER
  - Rule: A user registering themselves (not created by a RECEPTIONIST/ADMIN) always
    receives role CUSTOMER
  - Violation: The registration endpoint ignores any role sent in the request body
  - Implementation: The role is hardcoded in the CreateUserUseCase for the public
    registration flow; only the "create staff/walk-in customer" use case (RECEPTIONIST/ADMIN-only)
    accepts a role parameter
```

---

### Entity: Product

**Context:** Catalog

**Description:** An item Amaranta Shop sells (food, accessory, hygiene item, toy, medication).

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| id | UUID | Unique identifier | Yes | Auto-generated |
| sku | string | Stock keeping unit | Yes | Unique |
| name | string | Product name | Yes | 2–120 characters |
| description | string | Product description shown to the customer | Yes | Max 500 characters |
| category | ProductCategory (enum) | `FOOD \| ACCESSORIES \| HYGIENE \| MEDICATION \| TOYS` | Yes | |
| price | Money (VO) | Price in COP | Yes | > 0 |
| stockQuantity | integer | Units available | Yes | ≥ 0 |
| imageUrl | string | Product photo | Yes | Required — the shop UI always shows a photo |
| isActive | boolean | Whether the product is visible in the catalog | Yes | Defaults to `true` |

**Invariants:**

```
INV-PRODUCT-001: Non-negative price
  - Rule: price.amount must always be > 0
  - Violation: Cannot be saved with price <= 0
  - Implementation: Validated in the Money value object constructor

INV-PRODUCT-002: Non-negative stock
  - Rule: stockQuantity must never go below 0
  - Violation: DomainException if a decrement would take it below 0
  - Implementation: Validated in the decrementStock() domain method, not in the caller
```

**Behaviors (domain methods):**
- `decrementStock(quantity)`: reduces stock; throws if insufficient
- `restoreStock(quantity)`: used when an order is cancelled

---

### Entity: Order (Aggregate Root)

**Context:** Sales

**Description:** A confirmed or in-progress purchase of one or more products.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| id | UUID | Unique identifier | Yes | Auto-generated |
| customerId | UUID | Reference to the `User` who owns the order (cross-context, by ID only) | Yes | |
| channel | OrderChannel (enum) | `ONLINE \| IN_STORE` | Yes | `IN_STORE` set when created by a `RECEPTIONIST` |
| createdBy | UUID | The `User` id who created the order (may differ from `customerId` when a receptionist assists) | Yes | |
| items | OrderItem[] | Line items (internal entity, see below) | Yes | At least 1 item |
| total | Money (VO) | Order total in COP | Yes | Always equals the sum of item subtotals |
| status | OrderStatus (enum) | `PENDING \| PAID \| CANCELLED \| FULFILLED` | Yes | See lifecycle below |
| createdAt | DateTime | Creation date | Yes | Immutable |

**Internal entity: OrderItem**

| Attribute | Type | Description |
|-----------|------|-------------|
| productId | UUID | Reference to the `Product` at time of purchase |
| productName | string | Snapshot of the product name (survives future renames) |
| unitPrice | Money (VO) | Snapshot of the price at time of purchase |
| quantity | integer | Units purchased (1–50) |
| subtotal | Money (VO) | `unitPrice * quantity` |

**Lifecycle / States:**

```
PENDING ──(payment confirmed)──▶ PAID ──(handed to customer)──▶ FULFILLED
   │
   └──(cancelled)──▶ CANCELLED
```

| State | Description | Allowed transitions |
|-------|-------------|---------------------|
| PENDING | Created, awaiting payment confirmation at the counter | → PAID, → CANCELLED |
| PAID | Payment confirmed | → FULFILLED, → CANCELLED |
| FULFILLED | Customer has taken the products | Terminal state |
| CANCELLED | Order will not be completed | Terminal state |

**Invariants:**

```
AGGR-INV-ORDER-001: Total consistency
  - Rule: order.total must always equal the sum of items[].subtotal
  - Implementation: total is recalculated inside the aggregate every time an item changes,
    never set directly from outside

AGGR-INV-ORDER-002: No items after PAID
  - Rule: Items cannot be added or removed once the order is PAID, FULFILLED, or CANCELLED
  - Violation: DomainException "INV-ORDER-002: cannot modify a non-PENDING order"

AGGR-INV-ORDER-003: No duplicate product
  - Rule: No two OrderItems in the same order may share the same productId — increment
    quantity on the existing item instead

AGGR-INV-ORDER-004: At least one item
  - Rule: An order cannot be confirmed with zero items
```

---

### Entity: Appointment (Aggregate Root)

**Context:** Scheduling

**Description:** A scheduled block of clinic time for one pet, one service type, and one
professional.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| id | UUID | Unique identifier | Yes | Auto-generated |
| petId | UUID | Reference to the `Pet` (Clinical Records context) | Yes | Verified to exist via REST call before creation |
| ownerId | UUID | Reference to the `User` who owns the pet | Yes | |
| professionalId | UUID | Reference to the `User` (role `VETERINARIAN` or `GROOMER`) assigned | Yes | Must hold a role compatible with `serviceType` |
| serviceType | ServiceType (enum) | `MEDICAL_CONSULT \| VACCINATION \| GROOMING \| SURGERY \| EMERGENCY` | Yes | Determines `durationMinutes` default and which role can be assigned |
| scheduledAt | DateTime | Start date and time of the appointment | Yes | See business-hours invariant below |
| durationMinutes | integer | Length of the appointment | Yes | 15–120, defaulted by `serviceType` |
| status | AppointmentStatus (enum) | `PENDING \| CONFIRMED \| IN_PROGRESS \| COMPLETED \| CANCELLED \| NO_SHOW` | Yes | |
| createdBy | UUID | `User` id who booked it (self or a `RECEPTIONIST`) | Yes | |
| notes | string | Optional free-text note from the customer at booking time | No | Max 300 characters |

**Value Object: BusinessHours** (shared logic used by the invariant below)
- Operating window: **07:00–12:00** and **14:00–18:00**, Monday to Saturday
- `isWithinBusinessHours(scheduledAt, durationMinutes): boolean` — the appointment's full
  span, start to end, must fit inside one of the two windows (it may never straddle the
  12:00–14:00 lunch break or run past 18:00)

**Lifecycle / States:**

```
PENDING ──(staff confirms)──▶ CONFIRMED ──(appointment begins)──▶ IN_PROGRESS ──▶ COMPLETED
   │                              │
   └──(cancelled)──▶ CANCELLED    └──(cancelled)──▶ CANCELLED
                                   └──(customer doesn't show)──▶ NO_SHOW
```

**Invariants:**

```
AGGR-INV-APPT-001: Business hours
  - Rule: scheduledAt and scheduledAt + durationMinutes must both fall inside BusinessHours
  - Violation: DomainException "INV-APPT-001: appointment must be between 07:00-12:00 or
    14:00-18:00" — enforced in Appointment.create(), not only in the UI
  - Why in the domain and not just the UI: the receptionist-assisted booking flow and any
    future integration must not be able to bypass this rule

AGGR-INV-APPT-002: No double-booking
  - Rule: No two appointments for the same professionalId may have overlapping
    [scheduledAt, scheduledAt + durationMinutes) ranges, unless one is CANCELLED
  - Violation: DomainException "INV-APPT-002: professional already booked in this time range"
  - Implementation: Checked by the ScheduleAppointmentUseCase via a repository query
    (findOverlapping) before the aggregate is persisted; enforced additionally by a
    database exclusion constraint as a second line of defense

AGGR-INV-APPT-003: No past bookings
  - Rule: scheduledAt must be in the future relative to the moment of booking

AGGR-INV-APPT-004: Role-service compatibility
  - Rule: MEDICAL_CONSULT, VACCINATION, SURGERY, and EMERGENCY require a VETERINARIAN;
    GROOMING requires a GROOMER
```

---

### Entity: Pet (Aggregate Root)

**Context:** Clinical Records

**Description:** An animal registered under an owner.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| id | UUID | Unique identifier | Yes | Auto-generated |
| ownerId | UUID | Reference to the `User` (cross-context, by ID only) | Yes | |
| name | string | Pet's name | Yes | 1–50 characters |
| species | Species (enum) | `DOG \| CAT \| OTHER` | Yes | |
| breed | string | Breed | No | |
| birthDate | Date | Date of birth | No | Used to compute age shown in the clinic UI |
| weightKg | decimal | Latest known weight | No | > 0 if present |
| photoUrl | string | Pet photo | No | |

**Invariants:**

```
INV-PET-001: Positive weight
  - Rule: weightKg, if present, must be > 0
```

---

### Entity: MedicalRecord (Aggregate Root)

**Context:** Clinical Records

**Description:** A single diagnosis/treatment entry written by a veterinarian, tied to a
completed appointment.

**Attributes:**

| Attribute | Type | Description | Required | Rules |
|-----------|------|-------------|---------|-------|
| id | UUID | Unique identifier | Yes | Auto-generated |
| petId | UUID | Reference to the `Pet` | Yes | |
| appointmentId | UUID | Reference to the originating `Appointment` (cross-context) | Yes | |
| veterinarianId | UUID | Reference to the `User` who wrote it | Yes | |
| diagnosis | string | Diagnosis text | Yes | Max 1000 characters |
| treatment | string | Treatment prescribed | No | Max 1000 characters |
| notes | string | Additional notes | No | |
| createdAt | DateTime | Entry date | Yes | Immutable |

**Invariants:**

```
INV-MEDREC-001: Immutability
  - Rule: Once created, diagnosis/treatment/notes cannot be edited — a correction is a new
    entry that references the one it amends
  - Why: medical history must be an audit-safe record, not a mutable note
```

---

## System Value Objects

### Value Object: Money

**Description:** An amount in Colombian Pesos. Every price, subtotal, and total in the system
is a `Money` value object — never a raw number.

**Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| amount | integer | Amount in whole COP (no decimal subunits are used in everyday COP pricing) |
| currency | string | Always `"COP"` for the MVP |

**Validation rules:**

```
- amount must be a non-negative integer
- currency must equal "COP" (multi-currency is out of scope)
```

**Example:**

```typescript
class Money {
  private constructor(private readonly amount: number, private readonly currency: 'COP') {}

  static cop(amount: number): Money {
    if (!Number.isInteger(amount) || amount < 0) {
      throw new DomainException('Money amount must be a non-negative integer');
    }
    return new Money(amount, 'COP');
  }

  add(other: Money): Money { return Money.cop(this.amount + other.amount); }
  multiply(factor: number): Money { return Money.cop(this.amount * factor); }

  format(): string {
    return new Intl.NumberFormat('es-CO', { style: 'currency', currency: 'COP', maximumFractionDigits: 0 }).format(this.amount);
  }
}
```

### Value Object: Email

Same shape as the generic example in `05-architecture/hexagonal-architecture.md`.

### Value Object: BusinessHours

Described above under `Appointment` — encapsulates the 07:00–12:00 / 14:00–18:00 rule so it
is defined in exactly one place and reused by both the booking use case and any future
reporting/analytics code.

---

## System Aggregates

### Aggregate: Order

**Aggregate Root:** `Order`
**Internal entities:** `OrderItem[]` — cannot exist or be queried independently of their order
**Value Objects:** `Money` (unitPrice, subtotal, total)
**Aggregate invariants:** AGGR-INV-ORDER-001 through 004 (see above)

**Why do these objects form an aggregate?**
> An `OrderItem` has no meaning outside its `Order`, and `order.total` must always reflect the
> current set of items — so both must be saved and validated together, in one transaction.

### Aggregate: Appointment

**Aggregate Root:** `Appointment`
**Value Objects:** `BusinessHours`
**Aggregate invariants:** AGGR-INV-APPT-001 through 004 (see above)

**Why an aggregate of its own (not nested under Pet or User)?**
> An appointment references a pet, an owner, and a professional, but none of those own its
> lifecycle — its own scheduling state (PENDING → CONFIRMED → ... ) and the business-hours/
> double-booking rules are self-contained and must be enforced independently of any single
> pet's history.

---

## Summary table of tactical building blocks

| Name | Type | Bounded Context | Aggregate Root? |
|------|------|----------------|----------------|
| User | Entity | Identity & Access | Yes |
| Product | Entity | Catalog | Yes |
| Order | Entity | Sales | Yes |
| OrderItem | Entity | Sales | No (inside Order) |
| Appointment | Entity | Scheduling | Yes |
| Pet | Entity | Clinical Records | Yes |
| MedicalRecord | Entity | Clinical Records | Yes |
| VO: Money | Value Object | Shared | N/A |
| VO: Email | Value Object | Shared | N/A |
| VO: BusinessHours | Value Object | Scheduling | N/A |
| Service: StockValidationService | Domain Service | Sales (calls Catalog) | N/A |
| Service: AppointmentAvailabilityService | Domain Service | Scheduling | N/A |

---

## Domain Services

### AppointmentAvailabilityService

Stateless service used by `ScheduleAppointmentUseCase` to check business hours and
professional availability before an `Appointment` is created — it spans a repository query
(existing appointments) and the `BusinessHours` value object, which is why it does not belong
to a single entity.

```typescript
class AppointmentAvailabilityService {
  constructor(private readonly appointmentRepo: IAppointmentRepository) {}

  async assertAvailable(professionalId: string, scheduledAt: Date, durationMinutes: number): Promise<void> {
    if (!BusinessHours.isWithinBusinessHours(scheduledAt, durationMinutes)) {
      throw new DomainException('INV-APPT-001: appointment must be between 07:00-12:00 or 14:00-18:00');
    }
    const overlapping = await this.appointmentRepo.findOverlapping(professionalId, scheduledAt, durationMinutes);
    if (overlapping.length > 0) {
      throw new DomainException('INV-APPT-002: professional already booked in this time range');
    }
  }
}
```

---

## Correlation with code

| Domain artifact | Package / folder in code | File |
|----------------|--------------------------|------|
| Aggregate Root `Order` | `sales-service/src/domain/order/` | `Order.ts` |
| Aggregate Root `Appointment` | `scheduling-service/src/domain/appointment/` | `Appointment.ts` |
| Aggregate Root `Pet` | `clinical-service/src/domain/pet/` | `Pet.ts` |
| Value Object `Money` | `*/src/domain/shared/value-objects/` | `Money.ts` (duplicated per service — no shared runtime dependency across services) |
| Domain Service `AppointmentAvailabilityService` | `scheduling-service/src/domain/appointment/services/` | `AppointmentAvailabilityService.ts` |

> See hexagonal structure in `05-architecture/hexagonal-architecture.md`
