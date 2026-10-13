# ADR-011 — Identity Design: the Security Service Owns Accounts, Tokens and Revocation

- **ID:** ADR-011
- **Date:** 2026-10-07
- **Status:** Accepted
- **Authors:** Brandow Claros — Tech Lead / Product Owner / Full-stack developer
- **Supersedes:** the refresh-token and revocation decision of [ADR-008](./ADR-008-jwt-signing-and-token-lifecycle.md), which was already superseded for signing by [ADR-009](./ADR-009-jwt-rs256-per-course-norm.md)

---

## Context

Annex J of the course norm (J.1.2) requires a transversal security microservice that "manages
identity and issues the JWT". In `amaranta-shop-docs#13` the teacher recommends that it be the
owner of identity — "accounts, sign-in, tokens and revocation" — and warns against a design
where it only signs what another service asks for: whoever held that service's credentials
could obtain any signed token.

Where the project stands:

- The monolith's identity module already owns accounts, credentials and login, and it is the
  only module that holds the private key ([ADR-009](./ADR-009-jwt-rs256-per-course-norm.md)).
  It is not a signing oracle.
- What it lacks is **revocation** and any way to stay signed in. [ADR-008](./ADR-008-jwt-signing-and-token-lifecycle.md)
  left both out "for the MVP": a token cannot be invalidated before it expires, and the user
  logs in again every hour.
- `amaranta-shop-security` now exists, and the teacher asks that the identity design be
  recorded before it is built.

**Known constraints:**
- Every service validates the JWT by itself with the public key (norm 5.3.7); no verifier may
  call the security service on each request
- The gateway only filters requests that carry no credentials; it does not validate tokens
- One developer, course-graded delivery; the design must be small enough to build

---

## Decision

**We decided:** the security service is the only place that creates accounts, checks
credentials, issues tokens and revokes them. Access tokens are short-lived; sessions are kept
alive and revoked through a server-side refresh token.

1. **Ownership.** The private key exists only in the security service. It has **no endpoint
   that signs caller-supplied claims**: a token is issued only as the result of a successful
   login or refresh.
2. **Access token.** JWT, RS256 ([ADR-009](./ADR-009-jwt-rs256-per-course-norm.md)), **15
   minutes** (configurable), with the claims `sub`, `role`, `sid` (session id), `jti`, `iat`,
   `exp`. Every service verifies it locally.
3. **Refresh token.** An opaque random value of at least 256 bits, shown once and **stored only
   as a hash** in `security.refresh_token` (session id, user id, hash, `issued_at`,
   `expires_at` — 7 days —, `rotated_at`, `revoked_at`). Each use **rotates** it: the old one is
   marked rotated and a new one is issued in the same session.
4. **Reuse detection.** Presenting a refresh token that was already rotated revokes the whole
   session. A legitimate client never does that; a thief holding a copy will.
5. **Revocation.** Three operations set `revoked_at` and stop refresh immediately: log out of
   the current session, log out of all sessions of the caller, and an administrator revoking
   all sessions of a user. A disabled account can neither log in nor refresh.
6. **The window, declared.** An access token already issued stays valid until it expires — at
   most 15 minutes — because verifiers do not call the security service. Refresh is cut at
   once; access is cut within the window. If a route ever needs immediate revocation, the
   upgrade is an introspection endpoint or a distributed list of denied `sid`s for that route
   alone — not for the whole system.
7. **One role per user**, as the monolith has today. The token carries a single `role`; the
   many-to-many `user_roles` table in `06-data/models.md` is not adopted.
8. **Data.** PostgreSQL, schema `security`, user `security_app` ([ADR-010](./ADR-010-engine-per-domain-sales-postgres-catalog-mongo.md)),
   migrated by `amaranta-shop-security-db`. Passwords stay hashed with bcrypt.
9. **Stage.** Until the service exists, the monolith's identity module keeps issuing tokens
   without revocation. That gap is declared, and it is closed by building revocation **in the
   security service**, not by retrofitting a module that is about to be extracted.

**Justification:** it satisfies the teacher's three words — accounts, tokens, revocation —
without breaking the rule that each service validates tokens by itself. A short access token
plus a revocable refresh token is the smallest design in which revoking something has an effect,
and its one weakness (the window) is measurable and written down instead of hidden.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| Short access token + rotating, revocable refresh token (CHOSEN) | Revocation takes effect; theft of a refresh token is detected; services stay autonomous | A second token type, a table and endpoints; a window of up to 15 minutes | — (chosen) |
| Keep ADR-008: no refresh, no revocation | Nothing to build | A leaked token is valid until it expires; contradicts the teacher's guidance | Leaves the identity service without the third thing it is meant to own |
| A denylist of `jti`/`sid` checked by every service on every request | Immediate revocation | Every verifier needs shared state or a network call; the services stop being autonomous | Kept as the upgrade path for specific routes, not as the base design |
| Opaque tokens, validated by calling the security service each time | Immediate revocation, trivial to reason about | A runtime dependency on every request; **contradicts norm 5.3.7** ("each service validates the JWT by itself") | Not allowed by the course norm |
| A very short access token (5 minutes) with no refresh | Almost no window | The user logs in again every five minutes | Unusable for the web shop |
| The security service signs tokens on behalf of other services | Convenient for service-to-service calls | **The exact design the teacher warns against** | Any holder of a service credential could obtain any token |

---

## Consequences

**Positive:**
- Meets Annex J (J.1.2) and the teacher's recommendation point by point
- A stolen refresh token is detected instead of silently accepted
- `06-data/models.md` already models a `refresh_tokens` table; the earlier inconsistency
  between 06 and 07 disappears

**Negative / Trade-offs:**
- New endpoints (refresh, logout, logout-all, revoke), a new table, and rotation logic to build
  and test
- Revocation of an access token is not immediate (≤ 15 minutes)
- The web client must keep two tokens, call refresh when it receives a `401`, and make sure only
  one refresh is in flight at a time
- Tokens now expire in 15 minutes instead of an hour, so refresh must be reliable

**Impact on the system:**
- Affected services: `security` (new), the web client; every other service verifies tokens
  exactly as before
- Documents that must be updated: `07-api/authentication.md` (it says "there is no
  refresh-token endpoint"), `07-api/contracts/openapi/auth-service.yaml`,
  `06-data/models.md` (align `refresh_token`, drop `user_roles`), and a note in ADR-008

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A refresh token is stolen | Medium | High | Stored hashed, rotated on every use, reuse revokes the session, HTTPS only |
| Two browser tabs refresh at once and trip the reuse detection | Medium | Medium | A single in-flight refresh in the client; a short grace window is a decision for the service's spec |
| A revoked user keeps acting for up to 15 minutes | Medium | Medium | The window is short and declared; introspection is the upgrade for sensitive routes |
| The private key leaks | Low | High | It lives only in the security service; the `kid` header leaves room for rotation later |

---

## References

- Annex J of the course norm — J.1.2; norm numeral 5.3.7
- `amaranta-shop-docs#13` — the teacher's recommendations
- [ADR-008](./ADR-008-jwt-signing-and-token-lifecycle.md), [ADR-009](./ADR-009-jwt-rs256-per-course-norm.md), [ADR-010](./ADR-010-engine-per-domain-sales-postgres-catalog-mongo.md)
