# B3 identity boundary - implementation plan

## Purpose and decision

B3 makes the already accepted identity design executable without changing the
ownership of Catalog or Sales. It is the prerequisite for a Security-capable
release candidate and for the later Workflow service.

The target is deliberately narrow:

```text
Front shell -> Security -> short-lived RS256 access token
                         -> rotating opaque refresh token
Catalog / Sales -> verify with public key only
Gateway -> route and reject a missing credential only
```

Security owns accounts, credentials, sessions, token issuance, refresh rotation
and revocation. It never acts as a signing oracle for another service. Catalog
and Sales continue to own their domains and verify tokens locally; the Gateway
does not acquire a database, private key, business rule, or token-verification
responsibility.

## Decision gate B3.0 - unblock data ownership

**State: blocked.** ADR-011 assigns migrations to `amaranta-shop-security-db`,
schema `security`, and database user `security_app`. No official repository with
that name is currently available to this delivery workspace.

Before implementation branches are opened, the team must record one of these
decisions in an approved PR:

1. create and grant access to the official `amaranta-shop-security-db` repository;
2. explicitly assign the schema and migration ownership to an existing official
   database repository; or
3. formally defer executable session persistence and exclude B3 from the release
   candidate.

No Security service may silently create its own schema at application startup.

## Immutable contract before code

| Concern | Contract |
|---|---|
| Signing | RS256 only; the private key is mounted into Security and never versioned or exposed to another service |
| Verification | Catalog, Sales and later Workflow use only the public key and reject an unexpected algorithm, issuer, audience, expiry, or role |
| Access token | Maximum 15 minutes; claims `sub`, `role`, `sid`, `jti`, `iat`, `exp`; `kid` supports key rotation |
| Refresh token | Opaque, at least 256 bits, displayed once, stored only as a hash, rotated on use |
| Reuse | A previously rotated refresh token revokes its session; a test covers the replay |
| Revocation | Logout current session, logout all sessions, and administrator revocation stop refresh immediately; access expiry bounds the remaining window |
| Browser session | Front performs one refresh in flight and retries the original request only after a successful refresh |
| Service identity | Not part of the first Security login cut; it is a separate contract for Workflow-to-domain calls and must not reuse a user-token minting endpoint |

The API contract PR must reconcile `07-api/authentication.md` with ADR-011 before
the first HTTP implementation PR. The current document still describes the
monolith's one-hour token and no refresh flow; it is not the B3 runtime contract.

## Sequential pull-request plan

| Increment | Repositories | Change | Exit evidence |
|---|---|---|---|
| B3.0 data decision | Docs, Security DB owner | Record data repository, schema, roles, migration runner, backup owner | Approved decision; no ambiguous persistence owner |
| B3.1 contract | Docs, Security | OpenAPI, error envelope, environment placeholders, health contract, test matrix | API review; no private key or real secret committed |
| B3.2 core and data | Security, Security DB | Account/session domain, password hash port, refresh-token hash, migrations and least-privilege roles | Unit tests plus clean migration and privilege test |
| B3.3 adapters | Security | Login, refresh, logout and revoke HTTP adapters; RS256 signer and key loader | Negative tests for wrong password, bad algorithm, expiry, refresh reuse and revoked session |
| B3.4 consumers | Front, Catalog, Sales | Public-key configuration, issuer/audience checks, single-flight refresh in Front | Valid token accepted; invalid token rejected; no service receives a private key |
| B3.5 real journey | Gateway, Front, Security, Catalog, Sales, databases, Docs | Run login, protected call, refresh, logout and expired-token journey | Redacted E2E evidence and release-matrix update |

Each increment enters `develop` through its own child PR. Promotion to `qa` is by
`cherry-pick -x` into a `qa/` child branch; final release work enters `main` only
through a `release/` PR with the approval required by the branching policy.

## Bottlenecks and preventive controls

| Bottleneck | Preventive control | Stop condition |
|---|---|---|
| Missing Security DB owner | Resolve B3.0 before service persistence code | Do not start B3.2 |
| Private-key leak | File mount or secret store only; secret scan blocks `.env` and PEM files | Any key in Git history blocks the PR |
| Contract drift | Freeze OpenAPI and error cases in B3.1; consumers test the same claims | Consumer and issuer tests disagree |
| Refresh race between browser tabs | Single-flight refresh and replay test | Reuse detection logs out legitimate concurrent sessions |
| Token validation moved to Gateway | Keep verifier tests in Catalog/Sales; Gateway route tests only check header presence | Gateway begins parsing or storing JWT data |
| Unbounded revocation expectation | Declare the access-token window and test refresh revocation separately | A story promises immediate access-token revocation without a design |
| Workflow starts too early | Workflow accepts only the published Security contract and a separately approved service identity | No B3.5 evidence |
| Migration rollback ambiguity | Migrations are additive; restore rehearsal is part of the topology gate | Destructive rollback is proposed without recovery evidence |

## Definition of done for B3

B3 is complete only when the following are all true:

- the Security database owner, schema, user and migration history are explicit;
- no private key, password or refresh token is versioned;
- Security issues and rotates tokens according to the contract;
- Catalog and Sales independently reject invalid tokens and enforce their roles;
- Front refreshes once per concurrent expiry and logout prevents another refresh;
- the real journey runs through Gateway, Security, Catalog and Sales with redacted
  evidence; and
- Docs records exact revisions, tests, known residual risk and rollback or restore
  evidence.

## Explicit non-goals

- Reimplementing Catalog API, Sales API, or Gateway JWT verification.
- Starting Worker before an accepted event, broker, retry and dead-letter contract.
- Treating Workflow as ready before B3 publishes its identity contract.
- Creating `amaranta-v1.0.0-rc.2` before B3.5 evidence is accepted.
