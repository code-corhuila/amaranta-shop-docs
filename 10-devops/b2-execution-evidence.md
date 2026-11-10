# B2 persistent-topology execution evidence

This record distinguishes completed executable work from the QA deployment work that
remains blocked. It is the evidence companion to b2-persistent-topology-contract.md.

## Completed increments

| Increment | Repository and integrated commit | Pull request | Verified result |
|---|---|---|---|
| PostgreSQL runtime preflight | amaranta-shop-infra-postgres, 07e4351 | [#6](https://github.com/code-corhuila/amaranta-shop-infra-postgres/pull/6) | Checks health, platform network, named volume, localhost-only port, sales_app and pgcrypto before a migration or API deployment. |
| MongoDB runtime preflight | amaranta-shop-infra-mongo, 75bd7f8 | [#4](https://github.com/code-corhuila/amaranta-shop-infra-mongo/pull/4) | Checks health, platform network, named volume, localhost-only port, catalog_app and Catalog authentication. |
| Sales migration composition | amaranta-shop-infra-postgres, 68338bf | [#7](https://github.com/code-corhuila/amaranta-shop-infra-postgres/pull/7) | Composes the existing Flyway runner, applies all 11 Sales migrations, and verifies sales schema access for sales_app. |
| Catalog migration composition | amaranta-shop-infra-mongo, a7385cc | [#5](https://github.com/code-corhuila/amaranta-shop-infra-mongo/pull/5) | Composes the existing Liquibase runner, verifies the Catalog product collection and catalog_app access, and preserves idempotence. |

## Reproducible local validation

For each engine, the validation used a temporary ignored environment file, a distinct
Compose project and a localhost-only port. It then removed the temporary container
and named volume after the proof.

| Check | PostgreSQL and Sales | MongoDB and Catalog |
|---|---|---|
| Compose configuration | Passed with docker compose config --quiet. | Passed with docker compose config --quiet. |
| Runtime contract | scripts/up.sh followed by verify-runtime.sh dev passed. | scripts/up.sh followed by verify-runtime.sh dev passed. |
| Migration contract | The Flyway runner applied 11 migrations on a clean volume; sales.orders and sales_app access were verified. | The Liquibase runner applied the Catalog changesets; a second run found the changesets already applied; product and catalog_app access were verified. |
| Shell quality | ShellCheck passed in the pinned Docker image. | ShellCheck passed in the pinned Docker image. |
| Secrets and cleanup | No environment file was committed; the test volume was removed after the test. | No environment file was committed; the test volume was removed after the test. |

## CI composition contract

Both infrastructure repositories now run the preflight after booting a disposable
database in pull-request CI. They also check out a pinned database-contract revision:

| Infrastructure repository | Pinned database contract | Reason |
|---|---|---|
| Infra PostgreSQL | Sales DB 949976a | Keeps the verified Flyway composition reproducible until an intentional contract update. |
| Infra Mongo | Catalog DB 070946d | Keeps the verified Liquibase composition reproducible until an intentional contract update. |

The database repositories retain ownership of migrations, validators, roles and
grants. Infrastructure only starts the engine and invokes the existing runner over
the shared platform network.

## B2.3 status: blocked, not omitted

The QA critical journey has not been claimed as complete. Its remaining external
inputs are:

1. Approved QA DNS, ports and access model.
2. Secret-delivery mechanism and owner for database, API and identity credentials.
3. Backup destination, retention rule, recovery owner and a successful restore
   rehearsal on persistent storage.
4. The executable identity contract required for production-grade protected-route
   evidence.

When these inputs exist, the owner can run the existing real-stack journey from
Sales Portal against the approved topology and append redacted evidence for browse,
order, idempotency, stock reservation, internal-route protection, correlation and a
dependency-failure path.

## Release interpretation

This completes the executable local runtime and migration scope of B2. It does not
create a v2.0.0 tag, approve a QA deployment or replace B3 identity, B4 asynchronous
processing, or B5 operations. Those conditions remain governed by
v1-implementation-backlog.md.
