# C1 execution and release register

**Traceability:** Refs: code-corhuila/amaranta-shop-docs#42.
**Purpose:** keep the Clinical timeline increment, its repositories and its future
release evidence synchronized without claiming that a proposed component is deployed.

## Release tracks

| Product reference | Scope | Current rule |
|---|---|---|
| `amaranta-v2.0.0-rc.1` | Existing Catalog + Sales minimum journey | Still HOLD. It is the baseline candidate and does not include C1. |
| `amaranta-v2.0.0` | Approved baseline release | Cannot be created until the current evidence, topology, identity, recovery and release gates are resolved or explicitly excluded. |
| `amaranta-v2.1.0-rc.1` | Proposed Scheduling + Clinical timeline increment | A future candidate only. It cannot be created until every C1 slice has merged and its target-topology evidence is recorded. |

These are product-manifest references in Docs, not a mandate to apply the same tag
to every repository. Database repositories retain their own component-version
history. No tag may be created on an open PR branch.

## C1 repository register

| Area | Current status | C1 disposition | First allowed change |
|---|---|---|---|
| Docs | C1 plan is PR #43 into `docs/cross-repository-verification` | Ready for architecture review only | Merge #43 after independent review; record decisions and evidence links |
| Security / Security DB | B3 plan is PR #40; `amaranta-shop-security-db` is not available | Blocking dependency | Resolve B3.0 data owner, then B3.1 contract |
| Scheduling DB/API/Portal | Official repositories requested in #19 do not exist yet | Not provisioned | Create protected repositories; then C1.2a begins only after C1.1 contract |
| Clinical DB/API/Portal | Official repositories requested in #19 do not exist yet | Not provisioned | Create protected repositories; then C1.1a starts |
| RabbitMQ / platform | ADR exists; target environment inputs are not accepted | Design only | Approve broker endpoint, secret delivery, TLS, DLQ retention and alert owner |
| Gateway / Front | Existing baseline components | No C1 code now | C1.4c only after Scheduling and Clinical API contracts are merged |
| Catalog / Sales and their DBs/portals | Existing baseline components | Explicitly unchanged by C1 | Remain pinned compatibility evidence; no duplicate implementation |
| Workflow / Worker | Not in the current candidate | Explicitly excluded from C1 | Start only through a separate accepted event/business requirement |
| Infra PostgreSQL / Mongo | Local B2 evidence exists | No schema ownership transfer | Support target topology only after platform inputs and recovery gate |

## Entry checklist for implementation

Before the first executable C1 PR, the delivery owner records all of the following
as true with a URL or commit:

- [ ] PR #43 is independently reviewed and merged into the Docs parent branch.
- [ ] The professor has decided the Docs parent PR #23 and the resulting canonical
      documentation is on `main`.
- [ ] B3.0 names the Security schema owner, migration repository and runtime role.
- [ ] B3.1 freezes claims, issuer, audience, public-key distribution and service identity.
- [ ] The six Scheduling/Clinical repositories from #19 exist, have `develop`, `qa`
      and `main`, and follow the branching policy.
- [ ] The approved environment has broker, secret-delivery, TLS, backup and restore owners.
- [ ] Synthetic datasets and clinical-data redaction rules are accepted for all tests.

Any unchecked item is a stop condition. It is not replaced by a local mock, a
shared schema, or a direct database connection.

## Evidence ledger to fill during C1

| Slice | PR / commit | Executable changed lines | Verification | Reviewer | Result |
|---|---|---:|---|---|---|
| C1.0 boundary | #43 / pending merge | 0 | Docs review | Brandow | Pending |
| C1.1a Clinical schema | Not started | <=250 | Clean migration + role test | Clinical reviewer | Blocked by repos/B3 |
| C1.1b eligibility contract | Not started | <=450 | Contract + negative authorization test | Clinical reviewer | Blocked by C1.1a |
| C1.1c timeline read | Not started | <=450 | Indexed query + owner-cross-read rejection | Clinical reviewer | Blocked by C1.1a/B3 |
| C1.2a Scheduling schema | Not started | <=300 | Overlap + business-hour tests | Scheduling reviewer | Blocked by repos/C1.1 |
| C1.2b Clinical adapter | Not started | <=450 | Timeout, valid pet and unavailable service tests | Scheduling reviewer | Blocked by C1.1b/C1.2a |
| C1.3a–d projection | Not started | <=500 each | Outbox, duplicate, retry and DLQ tests | Integration reviewer | Blocked by C1.1/C1.2/platform |
| C1.4 UI/routing | Not started | <=450 each | Role, route and remote-loading tests | Front reviewer | Blocked by API contracts |
| C1.5 operational proof | Not started | <=500 each | E2E, metrics, DLQ drill, restore rehearsal | Release reviewer | Blocked by all prior slices |

The line budget follows [pr-sizing-policy.md](./pr-sizing-policy.md). A completed
row means a merged, independently reviewed commit with its evidence URL; it never
means only that a branch was created or that a local result was described.

## Promotion decision

1. Complete and merge C1.0 through C1.5 in the listed order.
2. Pin the resulting immutable SHAs in a new Docs release matrix.
3. Re-run the authorized C1 end-to-end journey against that matrix.
4. Record CI disposition, projection metrics, DLQ drill, rollback/restore evidence
   and residual risk.
5. Obtain the release review required by the branching policy.
6. Only then may the release owner create `amaranta-v2.1.0-rc.1` in the agreed
   Docs release context. Promotion to `amaranta-v2.1.0` requires the same matrix
   plus target-topology approval.

## Non-negotiable exclusions

- No clinical diagnosis, treatment, free text or contact details in event payloads,
  dashboard labels or structured logs.
- No shared database, cross-domain foreign key or global analytics store.
- No “temporary” direct Gateway-to-database route.
- No new code in Catalog or Sales merely to make C1 appear more complete.
- No production, release or tag claim from a branch, a mock broker or an unreviewed PR.
