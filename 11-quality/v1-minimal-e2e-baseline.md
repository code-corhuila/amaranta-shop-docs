# Amaranta v2.0.0 minimal end-to-end baseline

Initially verified: 2026-10-26. Reconfirmed against merged application branches:
2026-10-27.

This document is reverse-engineered from the current repositories and their executable
Sales Portal real-stack script. It defines the smallest functioning v2.0.0 scope with
evidence from real services. It is a release-candidate baseline, not a production
approval.

## Minimal functional scope

The baseline contains the complete customer path:

1. the Front shell loads Catalog and Sales remotes;
2. a visitor reads the product catalog through the Gateway;
3. a visitor creates a pickup order through the Gateway;
4. Sales reserves stock through Catalog;
5. a retried order with the same idempotency key returns the same order without
   reserving stock again;
6. correlation crosses Gateway, Sales API and Catalog API.

The executable script starts this scope from:

- Catalog DB migration runner and MongoDB;
- Sales DB migration runner and PostgreSQL;
- Catalog API and Sales API;
- API Gateway;
- Front shell, Catalog Portal and Sales Portal.

## Executed verification

The exact command was run from the Sales Portal repository:

    bash scripts/e2e-real-stack.sh up
    bash scripts/e2e-real-stack.sh check
    bash scripts/e2e-real-stack.sh down

The check completed successfully with the final result:

    OK: the whole system does what it must

The stack used throwaway containers, network, keys, tokens and database files. The
cleanup command removed those Amaranta test resources after the check.

On 2026-10-27 the same three commands passed again after synchronizing Gateway,
Catalog Portal, Sales Portal and Front with their merged develop branches.

## Runtime boundary

| Component | Role in v2.0.0 | Address or dependency |
|---|---|---|
| Front shell | Hosts navigation, session and API client | Host port 4200 |
| Catalog Portal | Catalog remote consumed by Front | Host port 4201 |
| Sales Portal | Sales remote consumed by Front | Host port 4202 |
| API Gateway | Only public API boundary | Host port 8000 |
| Catalog API | Product and stock owner | Internal port 8082; MongoDB |
| Sales API | Order owner | Internal port 8080; PostgreSQL and Catalog API |
| MongoDB | Catalog persistence | Container network only in this baseline |
| PostgreSQL | Sales persistence | Container network only in this baseline |

Only the Gateway exposes application APIs to callers. The two portals obtain their
HTTP client from the shell; they do not own a service URL or token.

## Behaviour proven by the real-stack check

| Contract | Result |
|---|---|
| Shell and both remoteEntry files answer successfully | Passed |
| Gateway accepts the purchase CORS preflight and Idempotency-Key header | Passed |
| Product seed is visible and Catalog API returns the portal response shape | Passed |
| Guest pickup order is created and total is calculated by Sales API | Passed |
| Stock changes once, then remains unchanged on an idempotent retry | Passed |
| Invalid quantity, contact, address and idempotency key are rejected | Passed |
| Invalid credential is rejected and customer order history is isolated | Passed |
| Sunday purchase is rejected without changing stock | Passed |
| The same correlation ID appears in Gateway, Sales API and Catalog API logs | Passed |

## Tested repository matrix

| Repository | Tested revision | Release condition |
|---|---|---|
| amaranta-shop-api-gateway | ebbe265 on develop | Gateway PR #5 merged |
| amaranta-shop-catalog-api | 87a98f3 on develop | Keep contract compatible with Gateway |
| amaranta-shop-catalog-db | 070946d on develop | Preserve migration and seed compatibility |
| amaranta-shop-catalog-portal | 76fd431 on develop | Catalog Portal PR #16 merged |
| amaranta-shop-sales-api | 016f56a on develop | Keep order and stock contract compatible |
| amaranta-shop-sales-db | 949976a on develop | Preserve migration compatibility |
| amaranta-shop-sales-portal | d9ff5cf on develop | Sales Portal PR #12 merged |
| amaranta-shop-front | 6376219 on develop | Front PR #2 merged under its original authorship |

The tested application revisions are now merged in their target branches. They become
a taggable release matrix only after the documentation evidence is approved and the
remaining CI and topology gates are resolved or formally accepted.

## Exclusions from the v2.0.0 minimum

- The script generates a throwaway signing key and test tokens; it does not prove a
  deployed Security service or an identity-provider integration.
- Workflow and Worker are outside the purchase path verified here.
- Infra Mongo and Infra PostgreSQL persistent-volume environments are not used by the
  ephemeral test; their topology, backup and restore requirements remain separate
  release gates.
- GitHub Actions did not execute remote checks because of the organization billing
  lock. The local result does not remove that CI blocker.

## Sequential promotion to v2.0.0

1. Review and merge this child documentation PR into the Docs parent branch.
2. Obtain the required CODEOWNER approval for the Docs parent PR to main.
3. Record the merged immutable SHAs in a release matrix and rerun this exact E2E
   script against those SHAs.
4. Resolve or formally accept the GitHub Actions billing incident with the responsible
   reviewer.
5. Create an annotated release-candidate reference named amaranta-v2.0.0-rc.1 in the
   Docs release matrix only after the preceding evidence is accepted.
6. Promote the same matrix to amaranta-v2.0.0 only after independent review, rollback
   evidence and the approved target topology.

Do not tag an open PR branch, a local-only commit or a test run whose component SHAs
have not been recorded.
