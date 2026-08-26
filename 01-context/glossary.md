# Project Glossary

---

## How to use this glossary

1. Before using a technical or business term in code, docs, or conversations: look it up here.
2. If it's not there: add it with its definition.
3. If there is disagreement about the definition: discuss it as a team and update this document.

---

## Domain terms

| Term | Definition | Notes / Synonyms |
|------|-----------|-----------------|
| Pet / Patient | An animal registered under an owner (`User`). Called `Pet` in the Sales/Catalog vocabulary context, `Patient` in the clinical vocabulary — same entity, same table, two names depending on which bounded context is speaking | Avoid mixing "patient" into Catalog/Sales documents |
| Owner | A `User` with role `CUSTOMER` who has at least one registered `Pet` | Synonym: pet owner, customer |
| Appointment | A scheduled block of time for a specific service (medical consult, vaccination, grooming, surgery) between a professional and a pet | Synonyms to AVOID: "cita" (Spanish, do not use in code/docs), "reservation" (reserve for hotel/table booking analogies only) |
| Business hours | The clinic's operating window for appointments: 07:00–12:00 and 14:00–18:00, Monday to Saturday. Not the same as the shop's browsing hours (the online shop is available 24/7) | See `Appointment` invariants in `02-domain/entities-and-rules.md` |
| Service type | The category of clinical service an appointment is for: `MEDICAL_CONSULT`, `VACCINATION`, `GROOMING`, `SURGERY`, `EMERGENCY` | Each has its own default duration |
| Professional | A `User` with role `VETERINARIAN` or `GROOMER` who can be assigned to an `Appointment` | Not a separate entity — same `User` aggregate, different role |
| Product | An item sold in Amaranta Shop (food, accessory, hygiene item, toy, medication) | Not the same as a clinical treatment |
| Order | A confirmed or in-progress purchase of one or more products, placed either online by a customer or in-store by a receptionist on the customer's behalf | Synonym to AVOID: "cart" — a `Cart` is the pre-checkout state; an `Order` exists once checkout is submitted |
| Cart | The client-side, not-yet-submitted collection of products a customer intends to buy | Becomes an `Order` on checkout |
| Channel | How an `Order` or `Appointment` was created: `ONLINE` (customer self-service) or `IN_STORE` (receptionist-assisted) | Tracked for reporting, does not change business rules |
| Medical record | An entry describing the diagnosis, treatment, and notes a veterinarian writes after an appointment | Owned by the Clinical Records bounded context |
| Stock | The available quantity of a `Product` that can still be sold | Decremented on order confirmation, restored on cancellation |

---

## Technical terms of the project

| Term | Definition |
|------|-----------|
| Microservice | Independent service with a single responsibility, its own process, and its own database |
| Domain Event | A fact that occurred in the business that other services can observe. Name always in past tense. |
| Bounded Context | Boundary within which a particular domain model has consistent meaning |
| API Gateway | Single entry point to the system that routes requests to the corresponding microservices. Amaranta uses **Kong**. |
| Service Discovery | Mechanism by which a service finds the network location of another service at runtime instead of a hardcoded address. Amaranta uses **Consul**. |
| Circuit Breaker | Pattern that stops calls to a failing service, preventing failure cascades |
| Saga | Sequence of local transactions across different services with compensating transactions on failure |
| Dead Letter Queue (DLQ) | Queue where messages that could not be processed after several retries are sent |
| Idempotence | Property of an operation to produce the same result if executed multiple times |
| ORM | Object-Relational Mapper. Amaranta uses **Prisma**, chosen specifically because its `provider` field lets a service move between PostgreSQL, MySQL, and SQL Server with minimal code change |

---

## Acronyms

| Acronym | Meaning |
|---------|---------|
| IAM | Identity and Access Management |
| JWT | JSON Web Token |
| API | Application Programming Interface |
| CRUD | Create, Read, Update, Delete |
| DTO | Data Transfer Object |
| FR | Functional Requirement |
| NFR | Non-Functional Requirement |
| SLO | Service Level Objective |
| SLA | Service Level Agreement |
| ADR | Architecture Decision Record |
| PR | Pull Request |
| DoD | Definition of Done |
| CI/CD | Continuous Integration / Continuous Delivery |
| COP | Colombian Peso — the only currency the system handles |
| AMQP | Advanced Message Queuing Protocol — the protocol RabbitMQ implements |
