# ADR-012 — Stack of the Extracted Services: Java, Python and nginx

- **ID:** ADR-012
- **Date:** 2026-10-07
- **Status:** Accepted
- **Authors:** Brandow Claros — Tech Lead / Product Owner / Full-stack developer
- **Supersedes:** [ADR-005](./ADR-005-api-gateway.md) (Kong)
- **Partially supersedes:** [ADR-006](./ADR-006-backend-framework.md) — it stays valid for the monolith, not for the extracted services

---

## Context

[ADR-005](./ADR-005-api-gateway.md) chose Kong, and discarded plain NGINX, for two reasons: Kong
validates JWTs at the gateway, and it routes through Consul. Both reasons have disappeared:

- [ADR-009](./ADR-009-jwt-rs256-per-course-norm.md) and norm 5.3.7 moved validation into each
  service. The gateway only rejects requests that carry no credentials.
- The teacher's `api-gateway-nginx` template is **configuration, not code**: one route file per
  domain, rate limiting, CORS, a correlation id, its own errors in the shared envelope, and
  per-request DNS resolution on the Docker network. Its README says that logic a proxy cannot
  express "is a decision for an ADR".

[ADR-006](./ADR-006-backend-framework.md) chose NestJS "for all 6 microservices". Sales will be
written in Java, from the teacher's `api-java` template, and Catalog in Python. Annex J
(J.1.2, and numeral 4.2.2 as corrected in J.9) requires at least two different languages among
the backend services, and J.1.2 names **Go, Java, Python or C#**.

**Known constraints:**
- The teacher provides templates for Java only; Python has none
- The NestJS monolith keeps serving traffic until its last module is extracted
  ([ADR-013](./ADR-013-monolith-first-strangler-fig.md))
- The language rule must hold without depending on how an ambiguous list is read
- One developer, course-graded delivery

---

## Decision

**We decided:**

1. **Sales — Java 21 with Spring Boot**, as the three-module Maven build of `api-java`:
   a `core` module with no Spring dependency, an `adapters` module and an `app` module.
2. **Catalog — Python 3.12 or later**, with a hexagonal structure that mirrors the Java
   template: a core with no framework imports, adapters, and one composition root. The web
   framework is chosen in the service's plan, not here.
3. **The monolith stays NestJS/TypeScript** until its last module leaves. We do **not** count it
   toward Annex J's two languages, because it is not among the languages the Annex names.
   Java and Python alone satisfy the rule.
4. **The security service's language** is decided in its own spec.
5. **Gateway — nginx**, from `api-gateway-nginx`, one route file per domain. It keeps the cheap
   "no credentials, no entry" filter, with **one public exception**: `POST /api/v1/orders`
   accepts a request without an `Authorization` header, so a visitor can buy without an account
   (spec `specs/checkout.md`, DP-1). Each service still validates its own tokens.
6. **Services find each other through Docker DNS** on the shared `platform` network, as the
   template does, not through Consul.

**Justification:** each choice follows a teacher-provided artefact where one exists (Java
template, nginx template), and the one place without a template — Python — is the cheapest way
to guarantee the two-language rule on our own, since Java plus Python are both on the Annex's
list. Not counting NestJS costs nothing today and removes the risk of a wrong reading.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| Java + Python, nginx (CHOSEN) | Two named languages; both teacher templates used where they exist | No Python template; two toolchains in CI | — (chosen) |
| Every extracted service in Java | Templates for all; one toolchain | Meets "two languages" only if NestJS counts, which the Annex's list does not say | Relies on a reading we cannot confirm |
| Catalog in Go or C# | Both are on the Annex's list | No template, and less familiar than Python | No advantage over Python |
| Keep Kong (ADR-005) | Already decided; plugins for JWT, rate limiting | Heavier; its two original reasons no longer apply; departs from the teacher's template | The reasoning of ADR-005 no longer holds |
| Keep NestJS for every service (ADR-006) | One stack, no new language | Fails Annex J's language rule outright | Not a valid option under the norm |

---

## Consequences

**Positive:**
- The two-language rule is met by two services, not by an interpretation
- The gateway is a configuration file anyone can read, not a product to operate
- The tokens' trust boundary stays in the services, where the norm puts it

**Negative / Trade-offs:**
- Python has no template: the hexagonal structure is built by hand and kept consistent with the
  Java one by convention and by tests written first
- Two ecosystems to build, test and release
- The gateway's CORS list must include the web origin (`http://localhost:3000`); the template
  allows only `4200` and `5173`
- [ADR-004](./ADR-004-service-discovery.md) (Consul) is now stale, because the template resolves
  services through Docker DNS. It needs its own review; this ADR does not rewrite it

**Impact on the system:**
- Affected services: Sales, Catalog, the gateway, and the monolith (unchanged until cutover)
- Documents that must be updated: `05-architecture/overview.md` (services and gateway),
  `09-microservices/`, `07-api/contracts/openapi/api-gateway.yaml`; ADR-005 is marked
  superseded and ADR-006 partially superseded

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| The Python service drifts from the hexagonal rules because nothing enforces them | Medium | Medium | Mirror the Java layout, write the tests first, and add an import-boundary check |
| The public exception on `POST /api/v1/orders` weakens "a token everywhere" | Low | Medium | Rate limiting stays on; a present but invalid token is still `401` (spec CA-C1.4) |
| The teacher accepts NestJS as a language after all | Low | Low | Nothing changes: we already satisfy the rule without it |

---

## References

- Annex J of the course norm — J.1.2, J.9
- Templates `api-java` and `api-gateway-nginx` (README of each)
- [ADR-005](./ADR-005-api-gateway.md), [ADR-006](./ADR-006-backend-framework.md), [ADR-009](./ADR-009-jwt-rs256-per-course-norm.md), [ADR-013](./ADR-013-monolith-first-strangler-fig.md)
- [`specs/checkout.md`](../../../specs/checkout.md) — DP-1, DP-6, DP-12
