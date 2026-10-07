# ADR-010 — Engine per Domain: Sales on PostgreSQL, Catalog on MongoDB

- **ID:** ADR-010
- **Date:** 2026-10-07
- **Status:** Accepted
- **Authors:** Brandow Claros — Tech Lead / Product Owner / Full-stack developer
- **Refines:** [ADR-002](./ADR-002-database-engine-and-orm.md) — PostgreSQL stays the default engine; this ADR records the exception

---

## Context

Annex J of the course norm (J.1.2) makes two engines mandatory — `<abbr>-infra-postgres` and
`<abbr>-infra-mongo` — and requires that **at least one domain uses MongoDB**. J.3 fixes the
rest: one instance per engine, defined in its `-infra-<engine>` repository; each `-db`
repository feeds its own schema (PostgreSQL) or its own database (MongoDB); no foreign keys
between domains.

Both infrastructure repositories now exist (`amaranta-shop-docs#13`, closed by the teacher).
In that issue the teacher recommends recording "the engine per domain … in an ADR before
building". [ADR-002](./ADR-002-database-engine-and-orm.md) chose PostgreSQL 16 "as the default
engine for every service"; it did not foresee the MongoDB requirement.

The first two services to leave the monolith are Sales and Catalog
([`specs/checkout.md`](../../../specs/checkout.md)). Their engine decides which template each
`-db` repository starts from (`db-postgres-flyway` or `db-mongo`), so it has to be decided
before those repositories are created.

**Known constraints:**
- At least one domain on MongoDB, ideally through a service we were going to extract anyway
- Sales writes an order, its lines and its idempotency key in **one transaction** (spec RN-15)
- Stock reservation must stay safe under concurrency (spec CA-C3.6)
- Existing orders reference products by id only, with no foreign key — product ids must
  survive the move

---

## Decision

**We decided:** each domain uses the engine below. PostgreSQL remains the default; Catalog is
the exception that satisfies Annex J's MongoDB requirement.

| Domain | Engine | Where it lives | Fed by |
|---|---|---|---|
| `sales` | PostgreSQL 16 | schema `sales`, user `sales_app`, in `amaranta-shop-infra-postgres` | `amaranta-shop-sales-db` (Flyway) |
| `catalog` | **MongoDB 7** | database `catalog`, in `amaranta-shop-infra-mongo` | `amaranta-shop-catalog-db` (Liquibase, MongoDB extension) |
| `security` | PostgreSQL 16 | schema `security` — see [ADR-011](./ADR-011-identity-design-with-revocation.md) | `amaranta-shop-security-db` |
| `scheduling`, `clinical` | PostgreSQL 16 | their own schemas, when each is extracted | revisited at that time |

**How Catalog is modelled in MongoDB** — the decisions the `db-mongo` template asks each team
to record:

- **One collection, `product`, no embedding.** A product has no child that is read with it
  and nothing grows with use; images are stored outside the database and the document keeps a
  URL.
- **Ids are preserved.** The `_id` is the product's existing UUID, so every historical order
  line still points at a real product.
- **Fields:** `sku`, `name`, `description`, `category`, `price`, `stockQuantity`, `imageUrl`,
  `isActive`, `createdAt`, `updatedAt`. `price` and `stockQuantity` are **`long`** — whole
  Colombian pesos and whole units. A plain JSON number arrives as a double, so the validator
  requires the type explicitly.
- **A `$jsonSchema` validator replaces the PostgreSQL `CHECK`s:** `additionalProperties: false`,
  `price` minimum 1, `stockQuantity` minimum 0, `category` limited to the five known values.
  `validationLevel: strict`, `validationAction: error`.
- **A unique index on `sku`**, and indexes for the category filter and the name search.
- **Reservation is one atomic conditional update on one document:** filter
  `{_id, isActive: true, stockQuantity: {$gte: qty}}`, update `$inc: {stockQuantity: -qty}`.
  If no document matches, the reservation failed. An order with several lines reserves them
  one by one and releases the earlier ones if a later one fails, exactly as the monolith does
  today. **No multi-document transaction is needed**, so no replica set is required.

**Justification:** Catalog is the cheapest way to meet J.1.2. It is already one of the two
services being extracted; a product is naturally a document; the one operation that needs
atomicity touches a single document; and it has about a dozen rows to move. The alternative —
keeping Catalog relational — would force a third service whose only reason to exist is the
engine.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| Catalog on MongoDB, Sales on PostgreSQL (CHOSEN) | Meets J.1.2 with a planned service; each engine fits its data | The catalog data model must be rewritten; one more engine to operate | — (chosen) |
| Everything on PostgreSQL | One engine to learn | **Violates Annex J (J.1.2):** no domain on MongoDB | Not a valid option under the norm |
| Sales on MongoDB | Orders with embedded lines are a classic document | The order, its lines and the idempotency key must commit together; money and relational integrity are the core of the business; the teacher's Java template is relational | Puts the core domain on the engine with the weakest guarantees for it |
| Clinical on MongoDB | Medical records are document-like | Clinical is the most coupled domain (it depends on Scheduling and Identity, and they on it); it would be the last extraction | Pushes the requirement to the end of the project |
| A notifications log on MongoDB | Natural fit for an append-only log | There is no notification service yet; building one only to hold a collection | A service created for the engine's sake |

---

## Consequences

**Positive:**
- Annex J's two-engine requirement is met by services that were already planned
- Each domain keeps one engine and one owner; no foreign key crosses domains
- Stock safety does not depend on transactions, so a standalone MongoDB instance is enough

**Negative / Trade-offs:**
- The Catalog section of `06-data/models.md` (DDL, `CHECK`, indexes) must be rewritten as a
  collection, a validator and indexes
- The monolith keeps its PostgreSQL `catalog` schema until the cutover; the dozen products are
  copied once, with their ids, by a `02_dml` changeset in `amaranta-shop-catalog-db`
- No joins: Sales never reads Catalog's data directly. It snapshots the name and the price into
  the order line, which it already does (spec RN-12)
- The concurrency test must run against a real MongoDB; a test double would only model it

**Impact on the system:**
- Affected services: Catalog (new, MongoDB), Sales (unchanged), Security (see ADR-011)
- Documents that must be updated: `06-data/models.md`, `05-architecture/overview.md`,
  `07-api/contracts/openapi/catalog-service.yaml` if the id or the stock fields change shape

---

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| A price or a stock is written as a double | Medium | Medium | The validator requires `long`; an integration test sends a plain JSON number and expects a rejection |
| A migration is applied twice | Medium | Low | MongoDB migrations are not transactional: every changeset must be idempotent on its own |
| A product document grows without bound | Low | Medium | Nothing in a product grows with use; any history becomes a collection of its own |
| The team lacks MongoDB operational experience | Medium | Low | One collection, one validator, three indexes — the smallest useful surface |

---

## References

- Annex J of the course norm — J.1.2, J.3, J.8
- `db-mongo` template — "Decisions the team records in an ADR"
- [`specs/checkout.md`](../../../specs/checkout.md) — RN-08, RN-12, RN-15
- [ADR-002](./ADR-002-database-engine-and-orm.md), [ADR-011](./ADR-011-identity-design-with-revocation.md)
