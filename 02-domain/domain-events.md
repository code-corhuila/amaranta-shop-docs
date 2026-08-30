# Domain Events

---

## What is a domain event?

See `02-domain/README.md` for the general definition. All Amaranta events travel through
**RabbitMQ** (see ADR-003) as JSON messages on topic exchanges.

---

## Event catalog

### Event: OrderCreated

| Field | Value |
|-------|-------|
| **Name** | `OrderCreated` |
| **Bounded Context** | Sales |
| **Aggregate** | Order |
| **Trigger** | A customer or receptionist submits a cart at checkout |
| **Consumers** | `catalog-service` (reserve stock), `notification-service` (send confirmation email) |
| **Channel (topic)** | `sales.order.created` |
| **Schema version** | `v1` |
| **Delivery guarantee** | At-least-once |

**Payload:**

```json
{
  "eventId": "uuid",
  "eventType": "OrderCreated",
  "aggregateId": "order-uuid",
  "aggregateType": "Order",
  "occurredAt": "2026-08-25T15:10:00Z",
  "version": 1,
  "payload": {
    "customerId": "uuid",
    "channel": "ONLINE",
    "items": [{ "productId": "uuid", "quantity": 2 }],
    "totalCOP": 85000
  }
}
```

**What do consumers do?**

| Consuming service | Action | Idempotent? |
|------------------|--------|-------------|
| catalog-service | Decrements stock for each item; emits `StockReserved` or, if insufficient, `StockDepleted` | Yes — uses `eventId` as idempotency key |
| notification-service | Sends an order confirmation email | Yes — checks the delivery log before sending |

---

### Event: StockReserved

| Field | Value |
|-------|-------|
| **Name** | `StockReserved` |
| **Bounded Context** | Catalog |
| **Trigger** | Stock was successfully decremented for all items of an `OrderCreated` event |
| **Consumers** | `sales-service` (moves order forward / confirms it can be paid) |
| **Channel (topic)** | `catalog.stock.reserved` |
| **Schema version** | `v1` |

### Event: StockDepleted

| Field | Value |
|-------|-------|
| **Name** | `StockDepleted` |
| **Bounded Context** | Catalog |
| **Trigger** | One or more items in the order no longer have enough stock |
| **Consumers** | `sales-service` (cancels the order and notifies the customer), `notification-service` |
| **Channel (topic)** | `catalog.stock.depleted` |
| **Schema version** | `v1` |

---

### Event: OrderPaid

| Field | Value |
|-------|-------|
| **Name** | `OrderPaid` |
| **Bounded Context** | Sales |
| **Trigger** | Staff confirms payment was received at the counter |
| **Consumers** | `notification-service` (payment receipt email) |
| **Channel (topic)** | `sales.order.paid` |
| **Schema version** | `v1` |

### Event: OrderCancelled

| Field | Value |
|-------|-------|
| **Name** | `OrderCancelled` |
| **Bounded Context** | Sales |
| **Trigger** | Order cancelled by staff, or automatically after `StockDepleted` |
| **Consumers** | `catalog-service` (restores stock for orders that had already reserved it), `notification-service` |
| **Channel (topic)** | `sales.order.cancelled` |
| **Schema version** | `v1` |

---

### Event: AppointmentScheduled

| Field | Value |
|-------|-------|
| **Name** | `AppointmentScheduled` |
| **Bounded Context** | Scheduling |
| **Aggregate** | Appointment |
| **Trigger** | An appointment passes all invariants (business hours, no double-booking) and is persisted |
| **Consumers** | `notification-service` (confirmation email) |
| **Channel (topic)** | `scheduling.appointment.scheduled` |
| **Schema version** | `v1` |

**Payload:**

```json
{
  "eventId": "uuid",
  "eventType": "AppointmentScheduled",
  "aggregateId": "appointment-uuid",
  "aggregateType": "Appointment",
  "occurredAt": "2026-08-25T15:12:00Z",
  "version": 1,
  "payload": {
    "petId": "uuid",
    "ownerId": "uuid",
    "professionalId": "uuid",
    "serviceType": "VACCINATION",
    "scheduledAt": "2026-08-27T09:00:00Z",
    "durationMinutes": 30
  }
}
```

### Event: AppointmentCancelled

| Field | Value |
|-------|-------|
| **Name** | `AppointmentCancelled` |
| **Bounded Context** | Scheduling |
| **Trigger** | Customer, receptionist, or professional cancels a `PENDING`/`CONFIRMED` appointment |
| **Consumers** | `notification-service` |
| **Channel (topic)** | `scheduling.appointment.cancelled` |
| **Schema version** | `v1` |

### Event: AppointmentCompleted

| Field | Value |
|-------|-------|
| **Name** | `AppointmentCompleted` |
| **Bounded Context** | Scheduling |
| **Trigger** | The professional marks the appointment as finished |
| **Consumers** | `clinical-service` (prompts/opens a `MedicalRecord` entry tied to this appointment) |
| **Channel (topic)** | `scheduling.appointment.completed` |
| **Schema version** | `v1` |

---

## Standard fields for all events

Same envelope defined in the framework's `02-domain/README.md` template
(`eventId`, `eventType`, `aggregateId`, `aggregateType`, `occurredAt`, `version`, `payload`,
`metadata.correlationId`, `metadata.causationId`, `metadata.userId`) — every Amaranta event
uses this envelope without exception, enforced by a shared Zod schema in each service.

---

## Event flow: Checkout

```
Customer / Receptionist
  │
  │  SubmitCart (command)
  ▼
[Aggregate: Order] (sales-service)
  │
  │  OrderCreated (event) — topic: sales.order.created
  ├───────────────────────────────────────────────┐
  │                                                ▼
  │                                       [catalog-service]
  │                                       decrementStock() per item
  │                                       StockReserved  or  StockDepleted
  │◀───────────────────────────────────────────────┘
  │  (on StockReserved: order stays PENDING, ready to be marked PAID at the counter)
  │  (on StockDepleted: order auto-cancels → OrderCancelled)
  │
  │  OrderCreated (event, fan-out)
  └───────────────────────────────────────────────▶ [notification-service]
                                                     sends confirmation email
```

## Event flow: Appointment booking

```
Customer / Receptionist
  │
  │  BookAppointment (command)
  ▼
[AppointmentAvailabilityService] (scheduling-service)
  │  1. validate BusinessHours (INV-APPT-001)
  │  2. validate no overlap for professionalId (INV-APPT-002)
  ▼
[Aggregate: Appointment]
  │
  │  AppointmentScheduled (event) — topic: scheduling.appointment.scheduled
  └───────────────────────────────────────────────▶ [notification-service]
                                                     sends confirmation email

... later, after the visit ...

[Aggregate: Appointment].complete()
  │
  │  AppointmentCompleted (event) — topic: scheduling.appointment.completed
  └───────────────────────────────────────────────▶ [clinical-service]
                                                     opens a MedicalRecord entry
```

---

## Schema evolution strategy

Same rule as the framework default (`02-domain/README.md`): additive/optional changes are
compatible; removing a field, changing its type, or renaming an event is not — a new versioned
event (`AppointmentScheduledV2`) is published alongside the old one during migration.

---

## Event summary table

| Event | Origin context | Topic | Consumers | Version |
|-------|---------------|-------|-----------|---------|
| OrderCreated | Sales | `sales.order.created` | catalog-service, notification-service | v1 |
| StockReserved | Catalog | `catalog.stock.reserved` | sales-service | v1 |
| StockDepleted | Catalog | `catalog.stock.depleted` | sales-service, notification-service | v1 |
| OrderPaid | Sales | `sales.order.paid` | notification-service | v1 |
| OrderCancelled | Sales | `sales.order.cancelled` | catalog-service, notification-service | v1 |
| AppointmentScheduled | Scheduling | `scheduling.appointment.scheduled` | notification-service | v1 |
| AppointmentCancelled | Scheduling | `scheduling.appointment.cancelled` | notification-service | v1 |
| AppointmentCompleted | Scheduling | `scheduling.appointment.completed` | clinical-service | v1 |

---

## Policies — Reactions to events

| Trigger event | Policy | Emitted command | Service |
|--------------|--------|----------------|---------|
| StockDepleted | Whenever an order's stock reservation fails, cancel the order automatically | `CancelOrder` | sales-service |
| AppointmentCompleted | Whenever an appointment finishes, open a draft medical record for the assigned veterinarian to fill in | `OpenMedicalRecordDraft` | clinical-service |

---

## Resilience patterns for events

Same as the framework default: **at-least-once delivery + idempotent consumers** (each
consumer stores processed `eventId`s), and a **Dead Letter Queue** per service queue with 3–5
retries, exponential backoff, and 7-day DLQ retention, configured in RabbitMQ (see
`05-architecture/decisions/records/ADR-003-message-broker.md`).
