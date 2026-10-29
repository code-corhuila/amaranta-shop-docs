# B2 persistent-topology contract

This is the reverse-engineered sequence that turns the demonstrated disposable
end-to-end stack into a reviewable persistent runtime. It is a delivery contract:
each increment has one owner, a small observable outcome, and evidence that becomes
the input of the next increment.

It does not authorize production deployment. Real QA or production values for DNS,
secret delivery, backup destination and access control still require approval from
the platform owner.

## Evidence used for the reverse engineering

| Existing fact | Source of truth | Consequence |
|---|---|---|
| PostgreSQL has one instance, an external platform network, per-environment volumes and local-only published ports. | amaranta-shop-infra-postgres at 07e4351, including [PR #6](https://github.com/code-corhuila/amaranta-shop-infra-postgres/pull/6) | Runtime readiness can be checked without exposing credentials or changing a migration. |
| MongoDB follows the same ownership model for Catalog, but has no equivalent runtime preflight. | amaranta-shop-infra-mongo at b6574c2 | Mongo is the next symmetric runtime gap; it must be closed before a combined topology is claimed. |
| Catalog and Sales already own migration runners, roles and rebuild checks. | amaranta-shop-catalog-db at 070946d; amaranta-shop-sales-db at 949976a | Infrastructure may invoke migrations but must not duplicate or edit their definitions. |
| The real-stack journey already composes Front, both portals, Gateway, APIs and both data engines. | amaranta-shop-sales-portal at d9ff5cf | The final B2 proof should reuse the real journey, replacing throwaway data only after the runtime and migrations are verified. |

## Contractual invariants

These rules remain true in all three increments:

1. Each infrastructure repository owns its engine container, network attachment,
   environment examples, health check and persistent volume.
2. Each -db repository owns schema, data rules, grants and migration history. It
   supplies a runner only; it never creates an engine container or volume.
3. APIs connect with catalog_app or sales_app, never with an administrator.
4. The Gateway remains the only public API boundary. Service and database addresses
   stay inside the platform network.
5. Real environment files, passwords, tokens and backups never enter Git or test
   output.
6. A failed check stops promotion. It does not trigger a destructive reset of QA or
   production volumes.

## The three auto-incremental developments

| Increment | Scope and owner | Input contract | Deliverable and acceptance evidence | Unlocks |
|---|---|---|---|---|
| B2.1 Runtime parity | Infra Mongo | PostgreSQL preflight merged at 07e4351; Mongo already has up.sh, down.sh, health check, platform, named volume and localhost-only port. | A verify-runtime.sh for dev, qa and main, executed in CI after boot. It proves the live Mongo health, platform attachment, named volume, localhost-only publication, catalog_app user and expected Catalog database. Local Docker run, invalid-environment rejection, Compose validation and shell lint are attached to the PR. | Both persistent engines expose the same safe, executable readiness boundary. |
| B2.2 Migration composition | Infra PostgreSQL, Infra Mongo, Sales DB, Catalog DB | B2.1 is accepted for both engines. Existing Flyway and Liquibase rebuild checks remain the authority for schema correctness. | A non-destructive composition and runbook that starts each engine, invokes the existing domain migration runner over platform, then verifies the domain role and schema or collection. CI proves composition syntax and uses a throwaway volume only. No applied migration changes and no QA volume deletion. | A fresh approved environment can be prepared in a known order: engine, migration, API. |
| B2.3 QA journey evidence | Sales Portal as real-stack harness; Gateway, APIs, Front, both portals and Docs | B2.2 proves both engines and migrations. The platform owner has supplied approved QA endpoint, secret-delivery and backup or restore inputs. | The existing real-stack critical journey is parameterized for the approved QA topology and produces a redacted evidence record: health checks, browse, order, idempotent retry, stock reservation, protected internal route, correlation search and one dependency-failure path. It never substitutes QA data with seed resets. | A candidate release manifest and a reviewable B2 completion decision. |

## Sequencing and review flow

Each row is a separate small PR. The preceding PR must be merged and its evidence
recorded before implementation of the next row starts.

1. Create feat/mongo-runtime-preflight from the current Mongo develop; target Mongo
   develop. Review its Docker proof and merge it before B2.2.
2. Create one child PR per repository for B2.2 because the migration runner belongs
   to its database repository and the invocation belongs to its infrastructure
   repository. Merge each child only after its own rebuild check and composition
   check are green.
3. Create test/qa-critical-journey in Sales Portal from the revision that owns
   scripts/e2e-real-stack.sh; target its develop. The Docs evidence change is a
   separate child PR, linked to that test PR.
4. Promote only accepted commits by cherry-pick -x from a qa child into qa. Do not
   merge permanent branches into each other.

## Tag and release rule

No B2.1 or B2.2 PR creates v1.0.0: they establish a capability, not a release.
After B2.3 is accepted, Docs records a release-candidate manifest with the exact
component SHAs and test evidence. Only when the v1 implementation backlog gates are
accepted may the release owner create the annotated v1.0.0-rc.1 candidate tag in
the agreed release repository. The final v1.0.0 tag remains blocked until the
release decision in v1-implementation-backlog.md is satisfied.

## Explicit blockers

| Blocker | Impact | Required resolution |
|---|---|---|
| No approved QA topology values | B2.3 cannot target a shared runtime responsibly. | Platform owner provides DNS, ports, secret-delivery mechanism, backup destination and access model. |
| No backup and restore rehearsal | Persistent volume cannot be considered recoverable. | Define an encrypted backup location, recovery owner, retention and a restore proof. |
| Missing executable identity contract | Protected-route evidence remains limited to the current test identity. | Complete B3 before treating the result as a production release. |
| No broker or event contract | Workflow and Worker stay outside the B2 journey. | Complete B4 only after an accepted event, retry and failure contract exists. |

## Definition of B2 complete

B2 is complete only when all three increments are merged, their evidence is linked
from the release manifest, the QA topology inputs are approved, and a restore
rehearsal plus the critical journey pass without destructive cleanup. Until then,
the completed PostgreSQL preflight is an accepted B2.1 slice, not a claim that the
full persistent topology is deployed.
