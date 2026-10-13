# ADR-005 — API Gateway

- **ID:** ADR-005
- **Date:** 2026-08-25
- **Status:** Superseded by [ADR-012](./ADR-012-stack-of-extracted-services.md)
- **Authors:** Brandow Claros — Tech Lead / Full-stack developer

---

## Context

The React web app must talk to 6 independent microservices without knowing their individual
addresses, without duplicating JWT validation logic 6 times, and without exposing every
service's port directly to the internet (or the in-store kiosk's network). Per
`05-architecture/pattern-guide.md`, an API Gateway is "always" needed in a microservices
architecture — the question is which one.

**Known constraints:**
- Must resolve upstream service targets through Consul (ADR-004), not a static config
- Zero budget, self-hosted, Docker-runnable
- Must support JWT validation and basic rate limiting out of the box (not custom-built)

---

## Decision

**We decided:** Use **Kong Gateway** (open-source) as the single entry point for all client
traffic, configured to resolve upstreams via Consul.

**Justification:** Kong is a mature, widely used, production-grade gateway with built-in JWT
validation, rate-limiting, and CORS plugins — meaning the Gateway's cross-cutting concerns
(`05-architecture/overview.md` §7) are configuration, not code the team has to write and
maintain. It has a documented Consul integration for dynamic upstreams, directly supporting
ADR-004, and it runs as a single Docker container locally.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| Kong Gateway (CHOSEN) | Built-in JWT/rate-limit/CORS plugins, Consul-aware, industry-standard, declarative config | Slightly heavier to configure than a hand-rolled proxy | — (chosen) |
| A custom NestJS gateway service (hand-written proxy + `http-proxy-middleware`) | Same language/stack as the rest of the system, full control | Reimplements JWT validation, rate limiting, and CORS by hand — exactly the cross-cutting work a real Gateway product exists to avoid; more code to maintain and test for no functional gain | Not worth building what Kong already provides correctly |
| NGINX (plain reverse proxy) | Extremely lightweight, well known | No native JWT validation or Consul-based dynamic routing without significant custom Lua scripting (OpenResty) | Kong is literally NGINX + these exact missing features, pre-built |

---

## Consequences

**Positive:**
- JWT validation, rate limiting, and CORS are configuration, not custom code, reducing surface
  area for security bugs (directly supports NFR-004)
- One declarative Kong config file documents the entire routing table for all 6 services

**Negative / Trade-offs:**
- One more piece of infrastructure to learn and run locally
- Kong's plugin configuration has its own learning curve distinct from the rest of the Node.js stack

**Impact on the system:**
- Affected services: all 6 (routed through Kong), plus the React frontend (which only ever calls the Gateway's address)
- Documents that must be updated: `10-devops/environments.md` (once filled)

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Kong becomes a single point of failure | Low (acceptable for local/demo scope) | High | Kubernetes manifests express Kong with multiple replicas for a future non-local deployment |

---

## References

- [Kong Gateway docs](https://docs.konghq.com/)
- [Kong + Consul upstreams](https://docs.konghq.com/hub/kong-inc/consul/)
- Related to: ADR-004 (service discovery)
