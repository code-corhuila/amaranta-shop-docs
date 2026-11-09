# v2.0.0-rc.1 local E2E evidence

**Execution date:** 2026-11-09

**Scope:** the existing Catalog + Sales minimum journey in a disposable local Docker topology.
**Result:** **PASS (local evidence only)**

This record is deliberately narrower than a release approval. It records one clean
execution of the repository-owned E2E harness at the revisions below. It does not
replace the QA topology, a review, service-to-service identity, CI disposition, or
backup/restore evidence required for a candidate tag.

## Immutable component revisions exercised

| Component | Branch at execution | Commit |
|---|---|---|
| Sales Portal (E2E harness) | `develop` | `d9ff5cfa039877e98faf1343f03a374f030defa9` |
| Front shell | `develop` | `6376219cec29bee97b18d25e04d661ca9347ef72` |
| Catalog Portal | `develop` | `76fd431c5dc67898cfac2acbdabfdcfbc923384c` |
| API Gateway | `develop` | `ebbe265e38de7af39f20288196f6b719745388dc` |
| Catalog API | `develop` | `87a98f3ed3eef45a806f42397ae227286774b02f` |
| Sales API | `develop` | `016f56a4bba41bf46799af67c61f6b838e567d53` |
| Catalog DB | `develop` | `070946d6ee7b3ed888899e71aaf76fdc5ccca524` |
| Sales DB | `develop` | `949976a36251e3e86621d33c4f63fa664150780d` |

The worktrees were clean before the execution. The harness generated a throwaway
signing key, test tokens and database credentials in its ignored `.stack/`
directory; none of that material is included in this document.

## Reproducible command and result

From `amaranta-shop-sales-portal`, with all listed repositories as siblings:

```bash
bash scripts/e2e-real-stack.sh up
bash scripts/e2e-real-stack.sh check
```

The first command completed the Sales Flyway migration (11 migrations), Catalog
Liquibase update (8 changes), image builds and startup of PostgreSQL, MongoDB,
Gateway, Catalog API, Sales API, Front, Catalog Portal and Sales Portal.

The check command completed with:

```text
OK: the whole system does what it must
```

The assertions that passed cover:

- Front loading both remote portals and the purchase preflight through the Gateway;
- guest purchase and the computed total;
- retry with the same idempotency key returning the same order without a second stock reservation;
- invalid quantity, contact data, delivery address and idempotency key responses;
- rejection of invalid credentials and unauthenticated order listing;
- customer order isolation and pagination validation;
- closed-shop rejection without changing stock; and
- one supplied `X-Correlation-Id` appearing in Gateway, Sales API and Catalog API logs.

## Release interpretation

This PASS satisfies the **local minimum-journey rerun** portion of the
`amaranta-v2.0.0-rc.1` readiness work. It does **not** change the release decision
from **HOLD** to **READY FOR TAG**.

The remaining gates are factual and must stay visible:

1. Docs PR #23 remains open and requires review before its release evidence can be
   considered merged.
2. The Browser-to-Gateway security work in Docs PR #30 remains open and requires
   review; the documented service-to-service identity gap is not closed by a local
   JWT test key.
3. An approved QA topology, secret-delivery method, backup/restore proof and CI
   disposition are still required by the release-readiness record.
4. The release owner must freeze the reviewed, merged SHAs in the release matrix
   and obtain the required approval.

Only after those gates are complete may the release owner create the annotated
`amaranta-v2.0.0-rc.1` tag in the agreed Docs release context. No component tag,
open PR branch, local-only result, or this document alone is a release candidate.
