# ADR-013 — Monolith First: Strangler Fig with Extraction Triggers

- **ID:** ADR-013
- **Date:** 2026-10-07
- **Status:** Accepted
- **Authors:** Brandow Claros — Tech Lead / Product Owner / Full-stack developer

---

## Context

The documentation describes two different systems. `05-architecture/overview.md` shows six or
seven services behind a gateway; the API contracts in `07-api/`, the navigation map in
`12-ux-ui/` and the code that actually runs describe **one NestJS monolith** with no gateway
and no service discovery. The teacher's review of 2026-10-02 flagged exactly this as the
largest inconsistency in the repository, and asked for an ADR that records the monolith stage
and says when each service is extracted.

The monolith was built first on purpose: it was the fastest way to have a working product and
to learn the domain boundaries. Its modules already talk to each other only through ports, and
the course now requires the services to be separate (Annex J).

What the code says about the boundaries today:

| Module | Calls out to | Is called by |
|---|---|---|
| `catalog` | nobody | `sales` (stock reservation) |
| `identity` | nobody | `scheduling` (professional lookup), `clinical` (owner lookup) |
| `sales` | `catalog` | nobody |
| `scheduling` | `identity`, `clinical` | `clinical` (appointment status) |
| `clinical` | `identity`, `scheduling` | `scheduling` (pet verification) |

**Known constraints:**
- A working product must exist at every step; there is no period in which the shop is down
- One developer, course-graded delivery
- The first cut must deliver two independent, working services, not a plan

---

## Decision

**We decided:** the system is built **monolith first** and extracted by the **Strangler Fig**
pattern, one capability at a time, behind a gateway. The monolith is the current stage, not a
mistake to hide; the services in `05-architecture/overview.md` are the **target**, and that
document marks which stage each one is in.

**Order of extraction**, from the table above — the fewest cut dependencies first:

1. **Catalog and Sales** (this cut). Catalog calls nobody; Sales reaches it through a port
   written to be replaced by an HTTP adapter.
2. **Security.** It calls nobody, and two adapters (in `scheduling` and `clinical`) become
   calls to it. See [ADR-011](./ADR-011-identity-design-with-revocation.md).
3. **Scheduling and Clinical.** They depend on each other, so the mutual dependency is cut
   first, or they leave together.
4. **Notifications** as a worker, when events travel over a broker.

**A module may be extracted when all of these hold:**

1. It has a spec whose rules are traced to tests (spec-driven development).
2. Every call it makes to another module already goes through a driven port.
3. A contract suite — HTTP-level tests — exists and passes against the monolith.
4. Its repositories exist (`-api`, `-db`) and their protections are in place.
5. Its data sits in its own schema or database, with no foreign key to another domain.

**Cutover rules:**
- It happens **by gateway route**. At any moment exactly one of the two implementations
  receives a route, so exactly one of them writes that domain's data.
- The monolith's module is deleted only after the spec's definition of done passes. Until then,
  going back is one route.
- The same contract suite runs against both implementations; any difference it tolerates is
  written in the spec as an intentional contract change.
- Writes are never mirrored to both: a copy of live checkouts would reserve every unit twice.

**Justification:** it keeps a working product at every step, it turns the monolith into the
reference that the new services are tested against, and it lets each extraction be judged by a
checklist instead of by confidence. The order follows measured dependencies, not preference.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| Monolith first, Strangler Fig (CHOSEN) | A working system throughout; the contract suite is reusable; each step is reversible | The two implementations coexist for a while; documents must mark the stage | — (chosen) |
| Rewrite everything as services at once | A clean result, nothing legacy | Nothing works until everything works; no reference to test against; the whole risk lands at the end | Too risky for one developer and one cut |
| Microservices from day one | Matches the target from the start | It would have meant guessing the boundaries before the domain was understood | Superseded by what the monolith taught us |
| Keep the monolith | Least work | Fails the course's requirement for separate services | Not a valid option under Annex J |

---

## Consequences

**Positive:**
- The 05 versus 07 contradiction is resolved by saying which one is the stage and which the target
- Every extraction has the same entry conditions and the same exit test
- The monolith keeps earning its place as the source of the contract suite

**Negative / Trade-offs:**
- For a while two implementations of a capability exist; the monolith's copy must be frozen —
  bug fixes only — once its contract suite is written
- Some calls stay in-process in the monolith and become HTTP in the new service, so each cut has
  one adapter to write and one to retire
- The documents carry an extra column (stage) that someone has to keep honest

**Impact on the system:**
- Affected services: all; the order above is the roadmap
- Documents that must be updated: `05-architecture/overview.md` (mark target versus built),
  `07-api/contracts/openapi/*.yaml` (the server of each contract per stage),
  `12-ux-ui/navigation-map.md`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| The monolith's copy keeps changing while its replacement is built | Medium | High | Freeze the extracted module once its contract suite exists |
| The stage table goes stale and the documents contradict each other again | Medium | Medium | The definition of done of each extraction includes updating it |
| A service is extracted before its dependencies are behind ports | Low | High | Condition 2 of the extraction checklist; nothing is cut that fails it |
| The new service and the monolith both write the same schema | Low | High | One writer per schema at a time, enforced by routing at the gateway |

---

## References

- Annex J of the course norm — J.1, J.3
- [`specs/checkout.md`](../../../specs/checkout.md) — section 8, and CA-X2.1 and CA-X2.2
- [ADR-010](./ADR-010-engine-per-domain-sales-postgres-catalog-mongo.md), [ADR-011](./ADR-011-identity-design-with-revocation.md), [ADR-012](./ADR-012-stack-of-extracted-services.md)
- `path-to-microservices.md` in the monolith repository (`BrandowC/amaranta`)
