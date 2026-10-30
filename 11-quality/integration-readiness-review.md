# Amaranta integration-readiness review

This is the sequential review record for the current integration increment. It gives
the reviewer a reproducible basis to decide whether the project has reached the
80-percent pre-release integration threshold. It is not a deployment approval and it
does not treat a blocked check as a passing check.

## Scope

The repository inventory contains 14 official Amaranta repositories:

- Gateway and Docs;
- Catalog API, DB and Portal;
- Sales API, DB and Portal;
- Front;
- Mongo and PostgreSQL infrastructure;
- Security, Workflow and Worker.

The organization interface displayed 15 search results, while the authenticated GitHub
repository search returned 14. A fifteenth repository must be added only after its
official URL is known.

## Meaning of 80 percent

The threshold means that the integrated delivery path is defined, reviewed and
reproducible for its included scope. It is not a claim that every repository is
complete or production-ready. Security, Worker, observability, backup/restore and
production deployment remain release gates after this threshold.

All mandatory rows below must be accepted by the reviewer before the threshold is
reported as reached.

| Gate | Required evidence | Current state | Reviewer decision |
|---|---|---|---|
| Inventory | Fourteen official clones, clean revisions and responsibility map | Ready locally | Pending |
| Gateway boundary | Catalog/Sales routes, non-public path protection and W3C trace handling | Gateway [#5](https://github.com/code-corhuila/amaranta-shop-api-gateway/pull/5) is open; local suite passed | Pending |
| Portal quality | Cross-platform portal tests, typecheck and build | Catalog Portal [#16](https://github.com/code-corhuila/amaranta-shop-catalog-portal/pull/16) and Sales Portal [#12](https://github.com/code-corhuila/amaranta-shop-sales-portal/pull/12) are open; local suites passed | Pending |
| Cross-repository contract | Trace, correlation, routing and release rules | This contract and Docs [#23](https://github.com/code-corhuila/amaranta-shop-docs/pull/23) | Pending |
| CI confirmation | Required Actions checks execute successfully, or a reviewer accepts the evidence of an external CI incident | Blocked: Actions jobs stopped before execution because of an organization billing lock | Pending |
| Topology baseline | Approved network, ports, persistence, health checks and secret names | Not received | Blocked |
| Complete critical journey | Front to Gateway to Catalog/Sales to ephemeral databases, including an unavailable dependency | Not yet executable without topology | Blocked |
| Security and Worker scope | Confirmed write access plus minimal IdP/broker contract, or documented exclusion from the release candidate | Access/contract pending | Blocked |

## Review sequence for Brandow

1. Review each open PR independently and confirm its source branch, target branch and
   local validation evidence.
2. Confirm that Gateway [#5](https://github.com/code-corhuila/amaranta-shop-api-gateway/pull/5)
   does not expose /internal/*, issue tokens, access databases or implement domain
   rules.
3. Confirm that Catalog Portal [#16](https://github.com/code-corhuila/amaranta-shop-catalog-portal/pull/16)
   and Sales Portal [#12](https://github.com/code-corhuila/amaranta-shop-sales-portal/pull/12)
   are limited to the Vitest alias/build correction and do not add service URLs or
   token handling.
4. Review the contract in
   [cross-repository-verification.md](./cross-repository-verification.md) and confirm
   that its commands and limitations match the PR evidence.
5. Record a decision for the CI billing incident. Do not merge a PR merely because
   local tests passed.
6. After the topology is approved, run the complete critical journey and append its
   exact commands, commit SHAs and outcome to the release evidence.

## Current conclusion

The preparation and verification structure is ready for review, but the project must
not yet be described as 80-percent integrated. That statement becomes supportable only
after the reviewer accepts the four PRs, the CI limitation is resolved or formally
accepted, and the topology-backed critical journey passes.

## Reviewer sign-off

| Decision | Name | Date | Evidence links or requested changes |
|---|---|---|---|
| Inventory accepted |  |  |  |
| PR evidence accepted |  |  |  |
| CI incident disposition |  |  |  |
| 80-percent threshold accepted or rejected |  |  |  |
