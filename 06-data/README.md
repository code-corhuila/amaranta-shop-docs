# 06 - Data

This section defines how Amaranta Shop stores, protects, migrates, and relates its data.
The MVP focuses on **relational SQL per microservice**, using PostgreSQL as the default
engine according to `05-architecture/decisions/records/ADR-002-database-engine-and-orm.md`.

The main rule is simple: **each microservice owns its own database**. No service performs
`JOIN`s or direct queries against another service's tables. When a service needs external
information, it gets it through an API or through events.

---

## Files in this section

| File | Purpose | Status |
|------|---------|--------|
| `models.md` | SQL schema per microservice, tables, relationships, indexes, basic security, and data ownership | Complete for the MVP |
| `data-dictionary.md` | Detailed dictionary of critical fields | Pending |
| `modeling-conventions.md` | Naming, data types, auditing, and soft-delete conventions | Pending |
| `normalization-assessment.md` | Normalization and denormalization justification | Pending |
| `migration-strategy.md` | Detailed migration policy | Pending |

Until the pending files exist, `models.md` is the main source of truth for the data model.

---

## Questions this section must answer

| Question | Amaranta answer |
|----------|-----------------|
| What data does each microservice handle? | Identity & Access (`identity-service` in architecture, `auth-service` in the current folder structure) handles users, credentials, roles, and tokens; `catalog-service` handles products and stock; `sales-service` handles orders and order items; `scheduling-service` handles appointments; `clinical-service` handles pets and medical records; `notification-service` handles delivery logs. |
| Why was this database engine chosen? | PostgreSQL was chosen because the domain is relational and requires ACID transactions, constraints, indexes, versioned migrations, and solid Prisma support. |
| How is the schema updated without breaking the system? | Through versioned, forward-only SQL migrations, with compatible changes first and destructive changes handled in two phases. |
| Who owns each piece of data? | Each data item has a single owner service. Other services store only UUID references or audit snapshots when needed. |

---

## Data map by microservice

| Microservice | SQL database | Owned data | Allowed external references |
|--------------|--------------|------------|-----------------------------|
| `api-gateway` | No persistent DB | No business data | Validates JWT and applies entry policies |
| `identity-service` / `auth-service` | PostgreSQL | Users, credentials, roles, tokens, security audit | None |
| `catalog-service` | PostgreSQL | Products, prices, stock, inventory movements | `created_by`, `source_order_id` as external UUIDs |
| `sales-service` | PostgreSQL | Orders, order items, order states | `customer_id`, `created_by`, `product_id` as external UUIDs |
| `scheduling-service` | PostgreSQL | Appointments, agenda, status history | `pet_id`, `owner_id`, `professional_id` as external UUIDs |
| `clinical-service` | PostgreSQL | Pets, medical records | `owner_id`, `appointment_id`, `veterinarian_id` as external UUIDs |
| `notification-service` | PostgreSQL | Notification templates and delivery log | `source_event_id`, `aggregate_id`, recipients |

---

## Correlations with other sections

| This section is fed by... | And feeds into... |
|---------------------------|-------------------|
| `02-domain/entities-and-rules.md` - entities, invariants, and value objects | SQL tables, mandatory columns, `CHECK` constraints, and internal relationships |
| `02-domain/domain-events.md` - events and consumers | `outbox_messages` and `processed_events` tables |
| `05-architecture/` - engine decisions, database per service, outbox, and saga | Engine choice and consistency rules in `models.md` |
| `models.md` | `07-api/contracts/` - data exposed by each service API |
| `models.md` | `08-uml/` - ER diagrams per service |
| `models.md` | `09-microservices/[service]/data-model.md` - operational detail per service |

---

## Basic SQL security

The model applies defensive security at the database layer:

- separate roles for migrations and runtime access: `*_migrator` and `*_app`;
- passwords and refresh tokens stored only as hashes;
- `NOT NULL`, `UNIQUE`, `CHECK`, and internal FK constraints;
- soft delete through `deleted_at`;
- audit and idempotency tables;
- no permissions between databases owned by different services;
- public queries through views or endpoints, never through direct access to sensitive tables.

Business authorization is validated in each service with RBAC, according to
`00-governance/security-policy.md`.

---

## How to append new information

1. Identify the owner of the data in the matrix above.
2. Add the table or column in `models.md` under the owning microservice.
3. If another service needs to read that data, update the contract in `07-api/contracts/`
   or publish/consume an event in `02-domain/domain-events.md`.
4. Create a new SQL migration named `V###__description.sql`.
5. Do not modify migrations that already ran.
6. Update the ER diagram in `08-uml/` and the service `data-model.md` file in
   `09-microservices/services/`.
