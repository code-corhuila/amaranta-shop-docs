# Amaranta sequential integration promotion plan

This plan orders the integration work across the official Amaranta repositories. It
turns the current blockers into explicit gates, so a blocked dependency stops only its
dependent wave instead of producing unreviewable changes across the platform.

## Working model

Each change has one responsibility, one source branch, one validation record and one
reviewer. A child PR is merged into its parent branch before the parent PR is proposed
to its permanent target.

Branch names do not encode a filesystem hierarchy. Git cannot keep both a branch named
feat/gateway-trace-context and a branch beneath that name. Use a distinct child name
and select the parent as the PR target:

| Purpose | Source branch | PR target |
|---|---|---|
| Parent integration change | feat/gateway-trace-context | develop |
| Real child increment | feat/gateway-trace-context-health | feat/gateway-trace-context |
| Documentation parent | docs/cross-repository-verification | main |
| Real documentation increment | docs/integration-promotion-plan | docs/cross-repository-verification |

After a child PR is merged into its parent, GitHub may delete the child branch. Closing
and deleting an unmerged child PR discards its work; it is not a promotion step.

## Mandatory evidence per PR

1. Record the source and target branches, immutable commit SHA and responsibility.
2. Run git diff --check and the repository test/build commands that cover the change.
3. Mark unavailable CI, Docker, topology or access as blocked, with the exact reason.
4. Keep secrets, generated database volumes and environment values out of commits and
   PR descriptions.
5. Require independent review before promotion to develop or main.

## Promotion order

| Wave | Scope | Repositories | Entry gate | Exit evidence |
|---|---|---|---|---|
| 0 | Governance and evidence | Docs | Repository map and cross-repository contract | Reviewed verification, readiness and promotion documents |
| 1 | Public request boundary | Gateway, Catalog API, Sales API | Existing API contracts and route ownership | Routing, non-public path, correlation and trace tests |
| 2 | User experience boundary | Front, Catalog Portal, Sales Portal | Shell remotes and public Gateway contract | Typecheck, build, remote isolation and no direct service URLs/tokens |
| 3 | Persistence and runtime topology | Infra Mongo, Infra PostgreSQL, Catalog DB, Sales DB | Approved topology and ephemeral data policy | Containers, migrations, health checks, cleanup and restore rehearsal |
| 4 | Cross-cutting capabilities | Security, Workflow, Worker | Confirmed repository access and IdP/broker contracts | Authentication path, retry/compensation, DLQ or failure policy and health endpoints |
| 5 | Critical journey and release candidate | All included repositories | Waves 0 through 4 accepted for their included scope | Front to Gateway to APIs to databases, failure-path evidence, release matrix and rollback record |

The Catalog and Sales domains already exist. Wave 1 validates their public integration
without reimplementing their domain behavior.

## Current PR mapping

| Parent PR | Responsibility | Child PRs allowed only for |
|---|---|---|
| Gateway [#5](https://github.com/code-corhuila/amaranta-shop-api-gateway/pull/5) | W3C request context and public edge boundary | A real additional gateway check or configuration change |
| Catalog Portal [#16](https://github.com/code-corhuila/amaranta-shop-catalog-portal/pull/16) | Cross-platform test/build aliases | A test-only correction required by reviewer feedback |
| Sales Portal [#12](https://github.com/code-corhuila/amaranta-shop-sales-portal/pull/12) | Cross-platform test/build aliases | A test-only correction required by reviewer feedback |
| Docs [#23](https://github.com/code-corhuila/amaranta-shop-docs/pull/23) | Verification contract and readiness evidence | A missing plan, checklist or evidence correction |

## Blocker-to-action map

| Blocker | Effect | Required owner/action | Unblocks |
|---|---|---|---|
| GitHub Actions billing lock | Remote checks do not execute | Organization administrator restores Actions billing or reviewer records a temporary exception | PR review decision |
| No approved topology | No trustworthy runtime manifest or E2E environment | Professor/platform owner supplies network, ports, DNS, persistence, secret names and environment target | Wave 3 and Wave 5 |
| Security and Worker access or contracts incomplete | No accountable change can be made | Repository owner confirms write access and provides IdP/broker, health and failure contracts | Wave 4 |
| Front work owned by another contributor | Cannot duplicate or claim their branch | Original owner submits or updates that PR; reviewer evaluates it under the original authorship | Wave 2 |
| No release evidence | Tag would be meaningless | Release owner records merged SHAs, successful checks, critical journey and rollback result | Wave 5 |

## Stable execution checklist

1. Review the existing four parent PRs; merge only approved changes.
2. For review feedback, create a child branch from the affected parent, open its PR to
   that parent and merge it there after review.
3. Freeze the merged SHAs in the Docs release matrix.
4. Receive and record the topology inputs before changing infrastructure manifests.
5. Bring up databases and APIs in an ephemeral environment, then prove the Gateway
   boundary before adding Front remotes.
6. Add Security, Workflow and Worker only when their contracts are executable.
7. Run the critical journey, failure path and rollback rehearsal before creating a
   release candidate tag.

## Stop conditions

Do not advance a wave when its entry gate is missing, when a check has not actually run,
or when a child change has not been merged into its parent. The proper response is to
record the blocker and assign an owner, not to create an empty PR or declare a release.
