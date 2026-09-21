# ADR-006 — Backend Framework

- **ID:** ADR-006
- **Date:** 2026-08-25
- **Status:** Accepted
- **Authors:** Brandow Claros — Tech Lead / Full-stack developer

---

## Context

The course requires Node.js on the backend. The Hexagonal Architecture standard used across
this repo (`05-architecture/hexagonal-architecture.md`, `_stacks/node-typescript.md`) needs a
framework that supports clean dependency injection between `domain/`, `application/`, and
`infrastructure/` layers without fighting the framework's own conventions, and that has a
native way to speak RabbitMQ (ADR-003), since most of Amaranta's inter-service communication is
event-based rather than REST.

**Known constraints:**
- Must produce OpenAPI-documented REST endpoints (API-First principle, `05-architecture/overview.md` §5 P1)
- One developer maintaining 6 services — consistency and built-in tooling matter more than raw minimalism

---

## Decision

**We decided:** Use **NestJS** with TypeScript for all 6 microservices.

**Justification:** NestJS's module system and constructor-based dependency injection map
directly onto the Ports & Adapters structure this repo already documents (controllers as
primary/HTTP adapters, repositories as secondary/persistence adapters, use cases injected
through interfaces) — the framework enforces the same boundaries the architecture already
requires, instead of leaving it to developer discipline alone. It ships a first-class RabbitMQ
microservice transporter (matching ADR-003) and generates OpenAPI/Swagger documentation from
decorators, satisfying the API-First principle with less hand-written contract work.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| NestJS (CHOSEN) | DI container matches Hexagonal Architecture, native RabbitMQ transporter, built-in OpenAPI generation, opinionated structure keeps 6 services consistent | More upfront structure/boilerplate than a minimal framework | — (chosen) |
| Express (plain) | Minimal, exactly what `_stacks/node-typescript.md` shows as the baseline example | No built-in DI — hexagonal boundaries would rely entirely on manual discipline across 6 services; RabbitMQ integration and OpenAPI generation would be hand-wired per service, repeating the same boilerplate 6 times | The manual discipline required to keep 6 services consistently hexagonal without framework support is a real risk for a one-person team on a deadline |
| Fastify | Faster raw throughput than Express/NestJS | Same lack-of-DI issue as Express; performance is not a real constraint at Amaranta's scale (NFR-001 targets 50 RPS) | Raw throughput is not the bottleneck this project needs to solve |

---

## Consequences

**Positive:**
- The same module/DI pattern is copy-pasted consistently across all 6 services, reducing the chance of one service silently breaking the hexagonal dependency rule
- OpenAPI docs stay close to the code (`@ApiProperty()` decorators) instead of drifting out of sync
- RabbitMQ consumers are declared with `@EventPattern()`, matching the domain-event catalog in `02-domain/domain-events.md` almost 1:1

**Negative / Trade-offs:**
- Slightly steeper learning curve (decorators, modules) than plain Express for a newcomer to the codebase
- More generated boilerplate per service than a minimal framework would produce

**Impact on the system:**
- Affected services: all 6
- Documents that must be updated: `_stacks/node-typescript.md` examples remain valid conceptually; NestJS-specific folder conventions will be documented per service in `09-microservices/`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Decorator-heavy code tempts putting business logic in controllers instead of the domain | Medium | Medium | Enforced by the Hexagonal Architecture Checklist in `05-architecture/hexagonal-architecture.md` during code review |

---

## References

- [NestJS documentation](https://docs.nestjs.com/)
- [NestJS RabbitMQ transporter](https://docs.nestjs.com/microservices/rabbitmq)
- Hexagonal Architecture → `05-architecture/hexagonal-architecture.md`
- Node.js/TypeScript stack guide → `_stacks/node-typescript.md`
- Related to: ADR-002 (database/ORM), ADR-003 (message broker)
