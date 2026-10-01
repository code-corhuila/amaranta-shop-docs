# ADR-009 — JWT Signing: RS256, Per the Course Norm

- **ID:** ADR-009
- **Date:** 2026-09-27
- **Status:** Accepted
- **Authors:** Brandow Claros — Tech Lead / Product Owner / Full-stack developer
- **Supersedes:** [ADR-008](./ADR-008-jwt-signing-and-token-lifecycle.md)

---

## Context

[`ADR-008`](./ADR-008-jwt-signing-and-token-lifecycle.md) adopted HS256 (a shared
secret) for JWT signing, reasoning that a single monolith process both signs and
verifies tokens today, so an asymmetric key bought nothing yet.

The course's official governing document,
`1-Norma-Repositorios-Sistemas-Distribuidos-2026B.pdf`, numeral 5.3.7, states:

> "Cada servicio valida el JWT por sí mismo... algoritmo RS256 con la clave pública
> del servicio de identidad, rechazo explícito de cualquier otro algoritmo —incluidos
> `none` y `HS256`—... Ningún servicio posee la clave privada ni una clave
> compartida."

This norm prevails over any internal team decision (numeral 1.2: "Prevalece sobre
cualquier convención interna del equipo. Un equipo puede añadir reglas más
estrictas; no puede relajar las de este documento."). HS256 is not a stricter rule
than RS256 here — it is a different, explicitly forbidden one. ADR-008's reasoning
was sound engineering judgment for the monolith's current shape, but it is not a
choice this project is free to make.

**Known constraints:**
- The rule is non-negotiable and applies starting now, not only after the
  microservices split described in `docs/path-to-microservices.md`
- Still a single monolith process today — the private key currently signs and the
  public key currently verifies inside the same codebase, but they must already be
  handled as two distinct secrets with two distinct exposure levels
- Zero budget, one developer, course-graded delivery

---

## Decision

**We decided:** Use **RS256** (RSA-2048, asymmetric) for JWT signing, as the norm
requires. The identity module holds `JWT_PRIVATE_KEY` and signs with it. Every
guard that verifies a token — today only inside the same monolith, later in
independently deployed services — uses only `JWT_PUBLIC_KEY`, and the verifier
explicitly restricts accepted algorithms to `['RS256']`, rejecting `none`, `HS256`,
or any other algorithm a forged token might declare.

**Justification:** The norm's own rationale is the justification: a service that
only *verifies* tokens must never be able to *mint* one, which is exactly what a
shared HS256 secret would allow if it ever leaked or was reused carelessly outside
the identity module. Enforcing RS256 and an explicit algorithm allow-list today —
even while everything still runs in one process — means the security boundary is
already correct in code before it becomes load-bearing at the microservices split,
instead of being retrofitted under deadline pressure later.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| RS256, explicit algorithm allow-list (CHOSEN) | Matches the course norm exactly (5.3.7); verifiers never hold signing capability; ready for the microservices split with no further change | Slightly more setup: generate and manage a key pair instead of one string | — (chosen) |
| HS256 (ADR-008's original choice) | Simplest possible setup for a single process | **Explicitly forbidden** by the course norm; a verifier that also holds the secret can forge tokens | Not a valid option under this project's governing document |
| Accept any algorithm declared by the token (no allow-list) | Zero extra code | The exact vulnerability the norm calls out by name: `alg: none` or an HS256 token signed with the public key as if it were a secret | Explicitly named as the classic exploit in the norm's own Anexo C |

---

## Consequences

**Positive:**
- Compliant with a non-negotiable, graded requirement (dimension 5 of the rubric,
  "Aislamiento de datos y arquitectura")
- The identity module is the only holder of `JWT_PRIVATE_KEY`; every other module
  and every future extracted service only ever needs `JWT_PUBLIC_KEY`
- No further JWT-related change needed when `identity-service` is extracted per
  `docs/path-to-microservices.md`

**Negative / Trade-offs:**
- Local development now requires generating an RSA key pair (`scripts/dev-keys.sh`)
  instead of picking one random string
- `JWT_PRIVATE_KEY` and `JWT_PUBLIC_KEY` must both be handled as multi-line PEM
  values encoded as single-line environment variables (literal `\n`), which is more
  error-prone to configure by hand than a plain secret

**Impact on the system:**
- Code changed: `apps/api/src/identity/identity.module.ts` (JwtModule now uses
  `privateKey`/`publicKey` + `signOptions.algorithm: 'RS256'` +
  `verifyOptions.algorithms: ['RS256']`, replacing `secret`),
  `apps/api/src/identity/infrastructure/security/jwt.strategy.ts` (verifies with
  `JWT_PUBLIC_KEY`, restricted to `algorithms: ['RS256']`)
- Config changed: `.env.example` (root and `apps/api/`), `docker-compose.yml`
  (`JWT_PRIVATE_KEY`/`JWT_PUBLIC_KEY` replace `JWT_SECRET`)
- New: `scripts/dev-keys.sh` generates a development-only RSA key pair and writes
  both env files; `keys/` and `*.pem` added to `.gitignore`
- Docs changed: `07-api/contracts/openapi/auth-service.yaml`,
  `07-api/authentication.md`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| `JWT_PRIVATE_KEY` is accidentally read or logged by a module other than identity | Low (only the identity module's `JwtModule.registerAsync` reads it) | High | Code review checks that only `identity.module.ts` references `JWT_PRIVATE_KEY`; every other verifier uses `JWT_PUBLIC_KEY` only |
| A developer regenerates keys with `dev-keys.sh` and forgets existing tokens are now invalid | Medium | Low | Expected behavior for a dev-only script; documented in the script's own header comment |

---

## References

- Course norm: `1-Norma-Repositorios-Sistemas-Distribuidos-2026B.pdf`, numeral 5.3.7
  and Anexo C ("Cómo se valida el token")
- Superseded decision → [`ADR-008`](./ADR-008-jwt-signing-and-token-lifecycle.md)
- Corrected contract → `07-api/contracts/openapi/auth-service.yaml`
- Extraction plan (why verifiers must not hold the private key) →
  `docs/path-to-microservices.md` (`amaranta-shop-mvp`)
