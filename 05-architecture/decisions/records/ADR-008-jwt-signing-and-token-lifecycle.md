# ADR-008 — JWT Signing Algorithm and Token Lifecycle

- **ID:** ADR-008
- **Date:** 2026-09-23
- **Status:** Superseded by [ADR-009](./ADR-009-jwt-rs256-per-course-norm.md) (signing) and [ADR-011](./ADR-011-identity-design-with-revocation.md) (token lifecycle)
- **Authors:** Brandow Claros — Tech Lead / Product Owner / Full-stack developer

---

> **Superseded:** the course norm (`1-Norma-Repositorios-Sistemas-Distribuidos-2026B.pdf`,
> numeral 5.3.7) mandates RS256 and explicitly forbids HS256. This decision is kept
> for the record — see [`ADR-009`](./ADR-009-jwt-rs256-per-course-norm.md) for the
> corrected decision and why this one no longer applies.

## Context

An earlier draft of `07-api/contracts/openapi/auth-service.yaml` documented RS256
(asymmetric) signing and a rotating refresh token, before any of the identity code
existed. When `07-api/contracts/openapi/auth-service.yaml` was corrected to match
the actual implementation (`apps/api/src/identity/`), it turned out the real
`JwtTokenIssuer` signs with a shared secret (HS256, `@nestjs/jwt`'s default) and
there is no refresh-token endpoint at all — the token simply expires and the user
logs in again. That gap between the originally-documented design and the
implemented one was never recorded as a decision, so a course review flagged it as
an unexplained, security-relevant change.

**Known constraints:**
- Single deployable today: the monolith (`amaranta-shop-mvp`) is the only process
  that ever verifies a token — no other service needs to check a signature
  independently yet
- Zero budget, one developer, MVP-scope delivery for the course
- `docs/path-to-microservices.md` already plans splitting Identity & Access into
  its own `identity-service`, at which point every other service (`catalog`,
  `sales`, `scheduling`, `clinical`) will need to verify tokens **without** calling
  back into `identity-service` for every request

---

## Decision

**We decided:** Use **HS256** (shared-secret HMAC) for token signing and **no
refresh token** for the MVP. The access token's lifetime is configurable via
`JWT_EXPIRES_IN` (default `3600s` / 1 hour); when it expires, the client calls
`POST /auth/login` again.

**Justification:** HS256 requires only one secret (`JWT_SECRET`), known solely to
the process that both issues and verifies tokens. That is exactly today's
topology — one monolith does both. RS256 buys nothing yet: its entire point is
letting a party that only *verifies* tokens do so with a public key, without ever
holding the private signing key — valuable once `identity-service` is a separate
process from `catalog`/`sales`/`scheduling`/`clinical`, worthless before that
split exists. Similarly, a refresh-token flow adds a second token type, a
revocation/rotation strategy, and a second persisted entity, to solve a UX problem
(avoiding re-login every hour) that is not a stated requirement for the MVP
delivery.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| HS256, no refresh token (CHOSEN) | Minimal moving parts; matches the current single-process topology exactly | Every future verifier needs the shared secret; user must re-login every hour | — (chosen) |
| RS256, no refresh token | Verifiers don't need the signing secret, ready for the microservices split | Asymmetric key management (generation, rotation, distribution) with no verifier that needs it yet | Solves a problem this deployment doesn't have yet — see "Revisit" below |
| HS256 + rotating refresh token | Better UX (no re-login every hour) | A second token type, storage, and revocation logic to build and secure, for a UX gap that is not a course requirement | Not worth the added surface at MVP scope |

---

## Consequences

**Positive:**
- Nothing to build or configure beyond a single `JWT_SECRET` environment variable
- The security model matches the actual deployment: one process signs, the same
  process verifies

**Negative / Trade-offs:**
- A user must log in again every time the token expires (default: hourly) — no
  "stay signed in" experience
- `JWT_SECRET` must be shared with every process that verifies a token, the moment
  a second process needs to do so — this becomes a real constraint at the point
  described below

**Impact on the system:**
- Affected: `07-api/contracts/openapi/auth-service.yaml`, `07-api/authentication.md`
- Must be revisited when: `identity-service` is extracted per
  `docs/path-to-microservices.md`. At that point, re-evaluate RS256 (or JWKS-based
  verification, matching the `/jwks`-style endpoint an earlier contract draft
  assumed) so `catalog-service`, `sales-service`, `scheduling-service`, and
  `clinical-service` can verify tokens without holding `JWT_SECRET`. A
  refresh-token flow can be revisited independently, if the re-login-hourly UX
  turns out not to be acceptable.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| `JWT_SECRET` leaks or is reused carelessly once more services need it | Medium (grows as more processes need the secret) | High | Revisit RS256/JWKS before the first service extraction, not after |
| A user is logged out mid-task when the token expires | Medium | Low | Acceptable for MVP scope; revisit only if user feedback flags it |

---

## References

- Identity implementation → `apps/api/src/identity/` (`amaranta-shop-mvp`)
- Extraction plan → `docs/path-to-microservices.md` (`amaranta-shop-mvp`)
- Corrected contract → `07-api/contracts/openapi/auth-service.yaml`
- Related to: ADR-005 (API Gateway)
