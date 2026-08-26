# ADR-004 — Service Discovery

- **ID:** ADR-004
- **Date:** 2026-08-25
- **Status:** Accepted
- **Authors:** Francy Claros — Tech Lead / Full-stack developer

---

## Context

Amaranta is deployed as 6 independent microservices plus an API Gateway. The Gateway (and any
service that makes a direct REST call to another, per the context map) needs to find the
current network location of the target service without a hardcoded address — the whole point
of the course's "discovery" requirement. Docker Compose and Kubernetes both offer *some*
built-in DNS-based discovery, but that alone doesn't give explicit health-check-aware discovery
or a visible registry a student can inspect and reason about.

**Known constraints:**
- Must work identically in local Docker Compose and in the target Kubernetes manifests
- Needs to be inspectable (a UI or API to see what's registered) for demonstration purposes
- Zero budget, self-hosted only

---

## Decision

**We decided:** Use **Consul** (HashiCorp) as the service registry. Every microservice
registers itself on startup with a health check (`GET /health`); Kong resolves upstream targets
through Consul instead of a static list of addresses.

**Justification:** Consul is the textbook example of the service-registry pattern taught in
distributed-systems courses — it's language-agnostic (any of the 6 Node.js services registers
over a simple HTTP call), ships a web UI to inspect the registry live during a demo, and
integrates cleanly with both Docker Compose (a single `consul` container) and Kubernetes (via
the official Consul Helm chart) without changing how services register.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| Consul (CHOSEN) | Explicit registry pattern, health-checked, inspectable UI, works the same in Compose and K8s | One more container to run locally | — (chosen) |
| Kubernetes-native DNS (`ClusterIP` + CoreDNS) | Zero extra component in a K8s cluster, "just works" | Only exists once deployed to K8s — provides nothing in the local Docker Compose environment this project actually ships and demos; also hides the discovery mechanism instead of teaching it explicitly | Doesn't satisfy the course's requirement to demonstrate discovery explicitly, and doesn't help local development |
| Netflix Eureka | Same textbook pattern as Consul | Java/Spring-centric tooling and client libraries; weaker fit for the Node.js/NestJS stack chosen in ADR-006 | Consul's Node.js integration (plain HTTP calls) is a better match for the stack |

---

## Decision detail

| Aspect | Choice |
|--------|--------|
| Registration | Each service calls Consul's HTTP Agent API on `main.ts` bootstrap, registering its name, address, port, and a `/health` check URL |
| Discovery consumer | Kong Gateway (via its Consul-backed upstream configuration) for external traffic; `sales-service` and `scheduling-service` for their two internal REST calls (to Catalog and Clinical Records respectively) |
| Deregistration | TTL/health-check based — Consul marks a service unhealthy and removes it from rotation if `/health` stops responding |

---

## Consequences

**Positive:**
- No hardcoded service addresses anywhere in configuration
- The Consul UI gives a live, demonstrable view of "which services are up" — directly useful for a course delivery
- Same registration code path works unchanged whether running under Docker Compose or Kubernetes

**Negative / Trade-offs:**
- One additional moving part to run and reason about locally
- Consul itself becomes a dependency other services need at startup (mitigated by Docker Compose's `depends_on` + health checks)

**Impact on the system:**
- Affected services: all 6, plus the API Gateway
- Documents that must be updated: `10-devops/local-setup.md` (once filled), each service's `README.md`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Consul itself is a single point of failure locally | Low (acceptable for local/demo scope) | Medium | Out of scope to run a Consul cluster for the academic MVP; documented as a known limitation |

---

## References

- [Consul service discovery docs](https://developer.hashicorp.com/consul/docs)
- [Kong + Consul integration](https://docs.konghq.com/)
- Related to: ADR-005 (API Gateway)
