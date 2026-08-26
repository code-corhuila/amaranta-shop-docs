# ADR-003 — Asynchronous Message Broker

- **ID:** ADR-003
- **Date:** 2026-08-25
- **Status:** Accepted
- **Authors:** Francy Claros — Tech Lead / Full-stack developer

---

## Context

The context map (`02-domain/domain-map.md`) requires asynchronous, event-driven communication
between bounded contexts for every reaction that is not an immediate query: stock updates after
checkout, cancellation compensations, and email notifications (`02-domain/domain-events.md`).
The course explicitly asks for a message broker and rewards keeping synchronous REST calls to a
minimum, so the async layer is not optional — it is where most inter-service communication
actually happens.

**Known constraints:**
- Amaranta's real event volume is small (tens of orders/appointments per day) — this is not a
  high-throughput streaming problem
- Must run locally in Docker with no paid managed service
- Needs native Dead Letter Queue support (per-consumer retry/backoff is required by
  `02-domain/domain-events.md`)

---

## Decision

**We decided:** Use **RabbitMQ** with topic exchanges (one exchange per bounded context,
e.g. `sales.events`, `scheduling.events`) as the message broker for all domain events.

**Justification:** RabbitMQ's AMQP model — exchanges, routing keys, per-consumer queues, and
native Dead Letter Exchanges — maps directly onto the "one event, several independent
consumers, some of which may fail and need retry" pattern every event in
`02-domain/domain-events.md` follows. It is lighter to operate in Docker/Kubernetes than Kafka
(no ZooKeeper/KRaft cluster, no partition/replication tuning) and NestJS ships a first-class
RabbitMQ microservice transporter, which matches ADR-006.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| RabbitMQ (CHOSEN) | Native DLQ, flexible routing (topic exchanges), simple single-node Docker setup, first-class NestJS transporter | Not built for very high-throughput log streaming or long-term event replay | — (chosen) |
| Apache Kafka | Excellent for very high-throughput streaming and long log retention/replay | Built for a scale Amaranta doesn't have; needs a broker cluster + coordination service to run correctly, meaning more operational complexity than the events in this domain justify; DLQ and per-consumer retry semantics require extra tooling (Kafka doesn't have them natively) | Overkill for tens of events/day; the operational cost isn't paid back by any benefit at this scale |
| Redis Streams | Very lightweight, already useful as a cache if one were needed | Consumer groups and DLQ semantics are less mature/standard than RabbitMQ's; no built-in flexible routing (topic exchanges) needed for the multi-consumer fan-out in `02-domain/domain-events.md` | RabbitMQ's routing and DLQ model fit the domain's needs more directly |

---

## Consequences

**Positive:**
- Fits the domain's actual shape: moderate volume, routing-heavy, needs reliable retry/DLQ
- Simple to run locally: one RabbitMQ container in `docker-compose.yml`, no cluster coordination
- The RabbitMQ management UI (port 15672) gives free visibility into queue depth and DLQ
  contents during development and demoing — useful for the course delivery

**Negative / Trade-offs:**
- If Amaranta's event volume ever grows by orders of magnitude (well beyond a single physical
  location), Kafka would become the more defensible choice
- No built-in long-term event replay (not needed — Event Sourcing was explicitly rejected in
  `05-architecture/pattern-guide.md`)

**Impact on the system:**
- Affected services: all 6 publish and/or consume events through RabbitMQ
- Documents that must be updated: each service's future `events.md` in `09-microservices/`

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A consumer processes the same event twice (at-least-once delivery) | Medium (normal AMQP behavior under retry) | Medium | Every consumer is idempotent, keyed by `eventId` — see `02-domain/domain-events.md` "Resilience patterns" |
| Messages pile up in the DLQ unnoticed | Low | Medium | RabbitMQ management UI checked during manual QA; a Prometheus alert on DLQ depth > 0 is planned per NFR-005 |

---

## References

- [RabbitMQ AMQP concepts](https://www.rabbitmq.com/tutorials)
- [NestJS microservices — RabbitMQ transporter](https://docs.nestjs.com/microservices/rabbitmq)
- Event catalog and resilience patterns → `02-domain/domain-events.md`
- Related to: ADR-006 (backend framework)
