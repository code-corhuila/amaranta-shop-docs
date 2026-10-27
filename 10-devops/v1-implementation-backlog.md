# Amaranta v1.0.0 implementation backlog

This is the reverse-engineered implementation order after the minimal end-to-end
journey passed on 2026-10-27. It separates the demonstrated v1 customer path from
the work needed to make that path releasable and operable.

## Requirements already demonstrated

| ID | Requirement | Evidence |
|---|---|---|
| FR-01 | A visitor can browse the seeded catalog through the public Gateway. | Real-stack check passed. |
| FR-02 | A visitor can create a pickup order and Sales calculates the total. | Real-stack check passed. |
| FR-03 | Sales reserves Catalog stock once for an idempotent order. | Real-stack check passed. |
| FR-04 | Customer order history is private and invalid credentials are rejected. | Real-stack check passed. |
| FR-05 | Front loads Catalog and Sales as isolated remotes. | Real-stack check passed. |
| NFR-01 | Only the Gateway exposes APIs; internal routes are not public. | Gateway and real-stack checks passed. |
| NFR-02 | A correlation ID is visible in Gateway, Sales API and Catalog API logs. | Real-stack check passed. |

## Ordered development blocks

| Block | Objective | Repositories | Done when |
|---|---|---|---|
| B1: release evidence | Make the demonstrated path auditable and reviewable. | Docs | Docs child PRs are merged, main has CODEOWNER approval, matrix contains the tested SHAs and CI disposition. |
| B2: persistent topology | Replace throwaway containers with approved dev or QA infrastructure. | Infra Mongo, Infra PostgreSQL, Catalog DB, Sales DB, APIs, Gateway | Network, ports, health checks, secret names, migrations, persistent volumes and cleanup policy are approved and tested. |
| B3: identity boundary | Replace throwaway signing keys with the executable identity contract. | Security, Gateway, APIs, Front | Public-key distribution, token issuer, roles, refresh/logout policy and health checks are documented and tested. |
| B4: asynchronous boundary | Add only the workflow and worker capabilities required by an accepted business event. | Workflow, Worker, Sales, Catalog | Event schema, broker, retry, idempotency, dead-letter policy and compensation are tested. |
| B5: operational release | Make v1 observable, recoverable and promotable. | Infra PostgreSQL observability, Gateway, APIs, Docs | Metrics, traces, alert ownership, backup/restore, rollback and load evidence are accepted. |

## Rule of expansion

Complete and validate one block before opening the next. A block may use child branches
for distinct, necessary changes, but each child must merge into its parent before the
parent is promoted. Do not create a branch merely to represent a future requirement.

## v1.0.0 release decision

The current customer journey is the functional core of v1.0.0. The release tag remains
blocked until B1 and B2 are complete and B3 through B5 are either complete or formally
excluded by the project owner with their risk recorded.
