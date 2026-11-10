# Cross-repository verification contract

This document defines the evidence required when one Amaranta change crosses repository boundaries. It is a PR verification contract, not a claim that a change is deployed or approved.

## Scope and responsibility

| Area | Responsibility | Required evidence | Boundary that must remain true |
|---|---|---|---|
| API Gateway | Routes public requests, protects non-public paths, preserves request context | Gateway route and smoke checks | No database access, domain rule or token issuance |
| Catalog API | Owns catalog HTTP behavior and its data boundary | API unit/integration tests and trace-context tests | `/internal/*` remains unavailable through the Gateway |
| Front shell | Loads portal remotes and calls services through the Gateway | Typecheck, build and shell/container tests | Portals do not own service URLs or tokens |
| Docs | Defines the contract, evidence and release conditions | Reviewed document links and commit references | Documentation does not substitute executable validation |

## Context propagation contract

The platform keeps the existing `X-Correlation-Id` convention and uses W3C `traceparent` for distributed tracing. They have different roles:

- `X-Correlation-Id` identifies the request in error envelopes and compatibility logs.
- `traceparent` carries a valid W3C trace ID and span relationship across HTTP boundaries.
- A receiving component preserves a valid incoming `traceparent`; a malformed, all-zero or absent header is replaced with a new valid context.
- Responses expose the resulting `traceparent` only where the caller needs it. No token, request body or personal data is added to tracing headers or logs.

The Gateway PR [#5](https://github.com/code-corhuila/amaranta-shop-api-gateway/pull/5) is the current review point for this behavior. Catalog API trace-context support was already merged into `develop` through [PR #15](https://github.com/code-corhuila/amaranta-shop-catalog-api/pull/15).

## Verification sequence

### 1. Repository gate

Every PR must record the source commit, target branch, command and result. At minimum:

```text
git diff --check
<repository documented test command>
<repository documented build command when available>
```

The output must distinguish a passed command from a blocked command. A missing Docker daemon, unavailable dependency or disabled CI is a blocker; it is never reported as a pass.

### 2. Contract gate

For the critical catalog journey, validate all of the following against stubs or real services as the repository supports:

1. `GET /api/v1/products*` reaches Catalog API without requiring a credential.
2. `/api/v1/admin/products*` reaches Catalog API only with the required credential.
3. `/api/v1/orders*` reaches Sales API under its documented authentication policy.
4. `/internal/*` cannot be reached through the public Gateway.
5. A valid `traceparent` survives Gateway to Catalog API; invalid values are not echoed.
6. The same request can be found by `X-Correlation-Id` and trace ID without inspecting a token or request body.

### 3. Shell gate

The Front shell must build its remote configuration and prove that unavailable remotes fail in isolation: navigation and the remaining portal stay usable. The shell calls the public Gateway address only; service container addresses are deployment details, not frontend configuration.

### 4. Release gate

Before a release candidate is tagged, the team records a clean commit for each included repository, successful CI where the organization runner is available, an end-to-end result, and the reviewer approval. A Git tag marks an approved immutable release commit; it must not be used to mark unmerged work or a local experiment.

## Evidence record format

Use this table in the PR description or the release note.

| Field | Required value |
|---|---|
| Component | Repository and responsibility |
| Source / target | Feature branch and target branch |
| Commit | Immutable short SHA and link |
| Commands | Exact validation commands |
| Result | Pass, fail or blocked with the reason |
| Dependencies | Services, runtime or secret names only; never values |
| Reviewer | Required owner or team |
| Release status | PR open, approved, merged or tagged |

## Suggested PR labels

Labels make the integration queue searchable; they do not replace review:

- `area:gateway`, `area:catalog`, `area:front`, `area:docs`
- `type:integration`
- `cross-cutting:observability`
- `blocked:ci` or `blocked:access` when evidence shows an external blocker

Apply only labels that exist in the repository. Creating or changing organization labels requires maintainer agreement.

## Current integration record

| Component | State | Evidence / limitation |
|---|---|---|
| Gateway W3C propagation | PR open | [Gateway #5](https://github.com/code-corhuila/amaranta-shop-api-gateway/pull/5); local Gateway suite passed. The remote Actions jobs did not start because the organization account reported a billing lock. |
| Catalog API trace context | Merged to `develop` | [Catalog API #15](https://github.com/code-corhuila/amaranta-shop-catalog-api/pull/15). |
| Front container shell | Existing work pending owner review | The current remote branch is owned by its original author; it must be reviewed under that ownership rather than copied into another contributor's PR. |
| Documentation | Contract-gap review already open | [Docs #11](https://github.com/code-corhuila/amaranta-shop-docs/pull/11) is separate from this verification contract. |

## Review checklist

- [ ] The PR changes one repository responsibility and does not duplicate merged work.
- [ ] Tests include the changed boundary, not only isolated code.
- [ ] The trace and correlation headers preserve no secrets or payloads.
- [ ] Documentation matches the actual commit and commands.
- [ ] CI is green, or the external blocker is visible and accepted by the reviewer.
- [ ] A release tag is deferred until the reviewed commit is merged.
