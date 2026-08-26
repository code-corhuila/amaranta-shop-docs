# ADR-002 — Relational Database Engine and ORM

- **ID:** ADR-002
- **Date:** 2026-08-25
- **Status:** Accepted
- **Authors:** Francy Claros — Tech Lead / Full-stack developer

---

## Context

Amaranta needs a relational database per microservice (Database per Service principle, see
`05-architecture/overview.md` §5 P2). PostgreSQL was the requested default, but the project explicitly
needs the freedom to move a service to MySQL or SQL Server later without a rewrite — for
example, if a future infrastructure provider only offers managed MySQL, or the course wants to
demonstrate portability across relational engines.

**Known constraints:**
- Zero budget — must run entirely in free/open-source Docker containers locally
- One developer — migration tooling and type safety matter more than raw performance tuning
- The chosen ORM must support PostgreSQL, MySQL, and SQL Server without switching libraries

---

## Decision

**We decided:** Use **PostgreSQL 16** as the default engine for every service, accessed through
**Prisma ORM**, so the database engine is a configuration choice rather than a code dependency.

**Justification:** Prisma's `datasource` block has a single `provider` field
(`"postgresql" | "mysql" | "sqlserver" | ...`). Moving a service to a different relational
engine means changing that one line, re-running `prisma migrate`, and regenerating the typed
client — no query code, no repository implementation, and no domain code changes, because the
Hexagonal Architecture already isolates persistence behind a repository port
(`05-architecture/hexagonal-architecture.md`). Prisma's official docs confirm first-class
support for all three target engines, and its type-safe generated client removes a whole class
of SQL-injection and typo bugs, directly supporting NFR-004 (Security) and NFR-006
(Maintainability).

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| PostgreSQL + Prisma (CHOSEN) | Type-safe client, first-class Postgres/MySQL/SQL Server support, built-in migrations, great local DX with Docker | Prisma's SQL Server support historically lagged Postgres/MySQL slightly | — (chosen) |
| PostgreSQL + TypeORM | Also supports Postgres/MySQL/SQL Server; more "raw" control over queries | Weaker type inference, decorator-heavy, migration tooling is less consistent across engines than Prisma's | Prisma's DX and multi-engine parity are stronger for a small team that needs to move fast |
| PostgreSQL + raw `pg` driver (no ORM) | Maximum control, zero abstraction overhead | Every engine swap would mean rewriting SQL by hand; loses type safety; contradicts the explicit "easy to swap engines" requirement | Defeats the entire purpose of this decision |
| MongoDB (document store) | Flexible schema | The domain is fundamentally relational (orders, appointments with time-range constraints, foreign keys between pets/owners/appointments); document modeling would fight the invariants in `02-domain/entities-and-rules.md` | Requested stack is explicitly relational |

---

## Consequences

**Positive:**
- Any service can move from PostgreSQL to MySQL or SQL Server with a config change + migration, not a rewrite
- Type-safe queries reduce runtime errors and SQL injection risk
- Prisma Migrate gives every service a clean, versioned migration history from day 1

**Negative / Trade-offs:**
- Prisma adds a build step (`prisma generate`) to each service's setup
- Very complex raw SQL (rare in this domain) still needs `$queryRaw`, which is engine-specific and would need re-writing on a swap

**Impact on the system:**
- Affected services: all 6 (identity, catalog, sales, scheduling, clinical, notification)
- Documents that must be updated: `06-data/models.md` (once filled), each service's future `data-model.md` in `09-microservices/`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A future engine swap exposes a Postgres-specific feature in use (e.g. JSONB operators) | Low | Medium | Keep schema design portable — avoid engine-specific column types unless justified and documented |

---

## References

- [Prisma multi-provider docs](https://www.prisma.io/docs/orm/overview/databases)
- Hexagonal Architecture (persistence isolated behind a port) → `05-architecture/hexagonal-architecture.md`
- Related to: ADR-006 (backend framework)
