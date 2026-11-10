# Amaranta v2.0.0-rc.1 readiness

**Decision date:** 2026-10-29  
**Current decision:** **HOLD - do not create the tag yet.**

This record reconciles the demonstrated local customer journey with the release
gates that still require review or an external topology decision. It does not
replace the executable evidence in
[v1-minimal-e2e-baseline.md](./v1-minimal-e2e-baseline.md) or the implementation
order in [v1-implementation-backlog.md](../10-devops/v1-implementation-backlog.md).

**Traceability:** Refs: code-corhuila/amaranta-shop-docs#41.

## What `amaranta-v2.0.0-rc.1` means

`amaranta-v2.0.0-rc.1` is a **product release-candidate manifest**, not an
identical component version. It pins one tested revision of every participant
in the minimal customer journey. It belongs in Docs as an annotated release
reference after review; it must not be created on an open PR branch or inferred
from independently tagged repositories.

The database repositories already use `v2.0.0`, while several application
repositories do not have a component tag. The release manifest preserves those
independent histories rather than forcing all repositories to a misleading
shared version.

## Candidate matrix

| Component | Candidate revision | Role in the demonstrated journey |
|---|---:|---|
| API Gateway | `ebbe265` on `develop` | Only public API boundary; routes, CORS, rate limit, correlation, internal-path block |
| Catalog API | `87a98f3` on `develop` | Product read, `ADMIN` mutation, stock reserve/release |
| Catalog DB | `070946d` on `develop`; `v2.0.0` | MongoDB collection, validator, seed, `catalog_app` access |
| Catalog Portal | `76fd431` on `develop` | Catalog remote consumed by Front |
| Sales API | `016f56a` on `develop` | Guest checkout, idempotency, order persistence, Catalog stock call |
| Sales DB | `949976a` on `develop`; `v2.0.0` | PostgreSQL schema and `sales_app` access |
| Sales Portal | `d9ff5cf` on `develop` | Sales remote consumed by Front |
| Front | `6376219` on `develop` | Shell that hosts Catalog and Sales remotes |
| Infra PostgreSQL | `68338bf` on `develop` | Local persistent runtime preflight and Sales migration composition |
| Infra Mongo | `a7385cc` on `develop` | Local persistent runtime preflight and Catalog migration composition |

## Gate ledger

| Gate | State | Evidence or remaining decision |
|---|---|---|
| Minimal customer journey | Passed locally | The exact Front -> Gateway -> Catalog/Sales -> databases journey passed on 2026-10-27; see the baseline record. |
| Gateway public-edge regression | Passed | The Gateway suite checks protected paths, CORS, correlation, rate limit, path traversal and downstream isolation. |
| Catalog implementation | Passed at component level | Catalog PRs cover RS256 verification, internal stock routes, `ADMIN` routes, trace context, logs and container checks. |
| Sales-Catalog contract | Passed locally | The real-stack check proves order persistence, stock reservation and idempotent retry. |
| Persistent database preparation | Passed locally | B2 preflight and migration composition are recorded in `b2-execution-evidence.md`. |
| Docs evidence accepted | Pending | Docs parent PR #23 still requires the required review and merge to `main`. |
| Official CI disposition | Pending | A reviewer must resolve or formally accept the organization-level CI limitation recorded in the baseline. |
| QA topology and recovery | Blocked | Approved DNS/ports, secret delivery, backup target and restore rehearsal are not yet supplied. |
| Executable identity boundary | Blocked | Security service, public-key distribution, issuer and service-to-service credential contract remain B3 work. |
| Workflow and Worker | Excluded from this candidate | They are not in the verified purchase journey; any later inclusion is B4 work. |

## Release-candidate procedure

1. An independent reviewer confirms the candidate matrix against the referenced
   repository revisions and the real-stack evidence.
2. Docs evidence is merged through the documented review gate; no open child
   branch is tagged.
3. Re-run the exact real-stack journey against the matrix above and append the
   command, date, result, and any intentional substitutions to the baseline.
4. Record the official CI decision: green remote checks or an accepted incident
   with owner and expiry.
5. Create the annotated Docs reference `amaranta-v2.0.0-rc.1`, whose message
   links this matrix and the successful evidence. Do not create matching tags in
   every repository merely for naming symmetry.
6. Promote to `amaranta-v2.0.0` only after the target topology, identity
   boundary, recovery evidence, and release approval are complete or formally
   excluded with recorded risk acceptance.

## Next engineering increment

`HU-E2E-001` is the next integration increment: run the existing real Catalog
and Sales services through the Gateway from the exact matrix, preserve the
idempotency and correlation assertions, and record the redacted result. This is
the only work that can change the candidate from **HOLD** to **READY FOR TAG**;
it does not implement Catalog API work already delivered by its owner.
