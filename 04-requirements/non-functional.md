# Non-Functional Requirements (NFR)

> Scale note: Amaranta is a single physical location. These NFRs are sized for that reality —
> a few hundred products, tens of appointments per day — not for a national e-commerce
> platform. The goal is to demonstrate correct distributed-systems practice at a realistic,
> honest scale, not to over-engineer for traffic that will never happen.

---

## NFR-001: Performance

| Attribute | Metric | Test condition |
|-----------|--------|---------------|
| P95 latency — critical endpoints | < 300ms | Under 50 concurrent RPS |
| P99 latency — critical endpoints | < 600ms | Under 50 concurrent RPS |
| P95 latency — non-critical endpoints | < 1000ms | Normal load |
| Minimum throughput | 50 RPS | Without degradation |
| Service startup time | < 30 seconds | Cold start (Docker Compose) |

**Defined critical endpoints:**
- `POST /orders` (sales-service) — checkout is the shop's conversion moment; slow checkout directly loses a sale
- `POST /appointments` (scheduling-service) — must validate business hours and check for overlaps synchronously; this is the most invariant-heavy write path in the system
- `GET /products` (catalog-service) — the first screen every customer sees

**Load testing tools:** k6 (run manually against the local Docker Compose environment — no
CI pipeline exists for this academic delivery)

**Where is it validated?** Manually, before final delivery, against the local environment.

---

## NFR-002: Availability

| Environment | SLO | Notes |
|------------|-----|-------|
| Local (the only environment built for this delivery) | Best-effort — no formal SLO | Availability targets apply once/if the project is deployed beyond the local Docker Compose environment |

**Health checks:**
- `GET /health` — Liveness: responds 200 if the process is alive
- `GET /health/ready` — Readiness: responds 200 only if it can process traffic (DB connected, RabbitMQ connected, Consul-registered)

> Formal SLO/error-budget tracking (as described in the framework's generic NFR-002) is
> deferred to a future, non-academic deployment — see `01-context/scope.md`.

---

## NFR-003: Scalability

| Scenario | Expected behavior |
|---------|------------------|
| Normal load (single-location traffic) | A single instance per service comfortably handles Amaranta's real volume |
| Kubernetes manifests still express replica counts and HPA thresholds | So the same deployment definitions would scale correctly if Amaranta opened a second location — this is what the course evaluates, even though local load never approaches it |
| Horizontal scaling limit | Up to 3 replicas per service in the local K8s manifests (demonstration value, not a real capacity need) |

**Strategy:** Stateless horizontal scaling — each service instance stores no in-memory session
state. Session/auth state is in the JWT itself; nothing else needs Redis for the MVP.

---

## NFR-004: Security

### Authentication and Authorization
- All private endpoints require a valid JWT in the `Authorization: Bearer <token>` header
- JWT tokens expire in **1 hour**; refresh tokens in **7 days** with rotation on each use
- RBAC roles and permissions: `00-governance/security-policy.md`

### Data transmission
- HTTPS in any non-local environment; HTTP is acceptable only for the local Docker Compose environment this project ships

### Sensitive data
- Passwords: bcrypt, cost factor ≥ 12
- Secrets (DB credentials, JWT signing key, RabbitMQ credentials): environment variables only,
  loaded from a local `.env` file that is git-ignored — never committed

### OWASP Top 10
See the full checklist and code-level controls in `00-governance/security-rules.md`.

---

## NFR-005: Observability

| Pillar | Requirement | Tool |
|--------|------------|------|
| Logs | Structured JSON format + Correlation ID | Pino (matches the Node.js/TypeScript stack guide) |
| Metrics | RED (Rate, Errors, Duration) per endpoint | Prometheus (local, via Docker Compose) |
| Traces | End-to-end distributed traces across the checkout and booking flows | OpenTelemetry + Jaeger (local) |
| Dashboards | Basic Grafana dashboard for the two critical flows (checkout, booking) | Grafana (local) |

**Correlation ID:** Each external request generates a UUID `correlationId`, propagated through
the API Gateway into every downstream REST call and event `metadata.correlationId`, so a single
checkout or booking can be traced across `sales-service` → `catalog-service` →
`notification-service` in the logs.

---

## NFR-006: Maintainability

| Metric | Target |
|--------|--------|
| Test coverage | ≥ 80% of lines, ≥ 90% in each service's `domain/` layer (aggregates and invariants) |
| Cyclomatic complexity | ≤ 10 per function |
| Onboarding time | A new developer can run the full system locally in < 1 hour following `10-devops/local-setup.md` |
| Average build time | < 5 minutes per service (`npm run build`) |

---

## NFR-007: Portability

- All 6 microservices + API Gateway are packaged as Docker images and orchestrated with Docker
  Compose locally
- Kubernetes manifests (Helm charts) express the same topology for a future non-local deployment
- No service depends on the host operating system
- Environment variables are the only source of environment-specific configuration
- **Database portability:** every service uses Prisma ORM against PostgreSQL by default; moving
  a single service to MySQL or SQL Server is a `datasource.provider` change plus a
  re-generated migration — no application code changes (see ADR-002)

---

## NFR-008: Disaster Recovery

Out of scope for this academic, local-only delivery — Amaranta runs a single Docker Compose
environment with no production traffic. This section is intentionally deferred to a future,
non-academic deployment (see `01-context/scope.md`, "Out of Scope").

---

## NFR priority matrix

| NFR | Priority (P1/P2/P3) | Validated how? | Owner |
|-----|---------------------|-----------------|-------|
| Performance | P2 | Manual k6 run before delivery | Full-stack developer |
| Availability | P3 | Health checks only (local) | Full-stack developer |
| Security | P1 | Manual OWASP checklist review + code review against `security-rules.md` | Full-stack developer |
| Scalability | P3 | K8s manifests reviewed, not load-tested at scale | Full-stack developer |
| Observability | P2 | Manual trace inspection of the checkout/booking flow | Full-stack developer |
| Maintainability | P1 | Jest coverage report | Full-stack developer |

---

## Correlations

- Detailed SLOs → deferred, see NFR-002/NFR-008 above
- Pipeline that validates NFRs → `10-devops/README.md`
- Security checklist → `00-governance/security-policy.md`, `00-governance/security-rules.md`
