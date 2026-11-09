# C1 Clinical timeline - production increment plan

**Traceability:** Refs: code-corhuila/amaranta-shop-docs#42.
**Status:** proposed for architecture review; no implementation code is authorized by this document.

## Decision and value

C1 is not an independent analytics, search, or reporting microservice. It is the
production-ready extraction path for the existing **Clinical Records** bounded
context: a completed appointment becomes an idempotent, protected and indexed
entry in the pet's clinical timeline.

Clinical owns `Pet`, `MedicalRecord`, the timeline query, and every physical index
used by that query. Scheduling owns appointment state and publishes the fact that
an appointment was completed. This preserves database-per-service ownership and
adds a useful processed-data capability without a shared database, a cross-service
join, or a new repository that has no business owner.

```text
Owner / receptionist -> Gateway -> Scheduling -> appointment completed
                                             -> AppointmentCompleted v1
                                                         |
                                                         | RabbitMQ, at-least-once
                                                         v
                                               Clinical processed_events
                                                         |
                                                         v
                                           draft medical record + timeline index
                                                         |
                                                         v
                                  owner / assigned veterinarian / admin read model
```

The timeline is the first read model. An analytical dashboard, data lake, full-text
search engine, or recommendation model is explicitly deferred until this source
of truth and its event evidence exist.

## Preconditions: do not bypass these gates

| Gate | Required fact | Why it blocks C1 implementation |
|---|---|---|
| C1.0 identity | B3 has an approved Security data owner and B3.1 publishes the issuer, audience, role and service-identity contracts. | Clinical data is sensitive; a role name in a mock header is not production authorization. |
| C1.1 official repositories | The professor creates and protects `scheduling-db`, `scheduling-api`, `scheduling-portal`, `clinical-db`, `clinical-api`, and `clinical-portal`, requested in issue #19. | The extraction checklist requires a protected `-db` and `-api` owner before code. |
| C1.2 platform | RabbitMQ endpoint, credential delivery, TLS policy, DLQ retention and alert owner are accepted for the target environment. | An event-driven clinical projection cannot be demonstrated only by a local in-memory event. |
| C1.3 data protection | Data classification, backup owner, restore rehearsal and retention decision exist for clinical data. | A medical timeline cannot be called production-ready if it cannot be recovered or access controlled. |

If a gate is not met, work may stay at contract/document level only. It must not
add application-startup migrations, seed medical data into shared environments, or
pretend that a local event is a deployed integration.

## Ownership and data boundaries

| Concern | Scheduling owns | Clinical owns | Forbidden shortcut |
|---|---|---|---|
| Appointment | slot, professional assignment, status and completion time | reference only (`appointmentId`) | Clinical writing an appointment status |
| Pet | only an external `petId` reference | pet profile and eligibility lookup | Scheduling querying the Clinical database |
| Record | no diagnosis, treatment or clinical notes | draft, signed/amended medical record and timeline | publishing diagnosis or notes through RabbitMQ |
| Query | appointment views | pet timeline and clinical record detail | joining Scheduling and Clinical tables |
| Identity | receives verified claims | authorizes owner, assigned veterinarian and admin | trusting a browser-supplied owner or role id |

The existing target model already defines the bounded indexes:

```text
pets(owner_id) WHERE active
medical_records(pet_id, created_at DESC) WHERE active
medical_records(veterinarian_id, created_at DESC) WHERE active
```

`clinical-db` owns their migrations and the `clinical_app` role. Those migrations
must be additive, reviewable, and tested on a clean volume before any API deploy.
The initial projection is therefore an indexed transactional read model, not a
copy of all Sales, Catalog, or Scheduling data.

## Minimal contracts

### Synchronous contract: Scheduling verifies a pet

Before accepting a booking, Scheduling calls a Clinical internal endpoint through
an adapter, for example `GET /internal/pets/{petId}/booking-eligibility`. The
response contains only `petId`, `ownerId`, and an eligibility state; it does not
return medical history. The caller uses the B3 service-identity contract, not a
customer access token.

Clinical evaluates the same owner relationship that is later enforced on reads.
The adapter has bounded timeouts and fails closed for a new booking if eligibility
cannot be verified; it never falls back to a direct database connection.

### Asynchronous contract: `AppointmentCompleted` v1

Scheduling persists `COMPLETED` and its outbox record in the same transaction.
The outbox publisher is the only component that publishes to RabbitMQ.

```json
{
  "eventId": "uuid",
  "eventType": "AppointmentCompleted",
  "aggregateId": "appointment-uuid",
  "aggregateType": "Appointment",
  "occurredAt": "RFC-3339 timestamp",
  "version": 1,
  "payload": {
    "appointmentId": "uuid",
    "petId": "uuid",
    "ownerId": "uuid",
    "professionalId": "uuid",
    "completedAt": "RFC-3339 timestamp"
  },
  "metadata": {
    "correlationId": "uuid",
    "causationId": "uuid",
    "userId": "uuid"
  }
}
```

Topic and consumer names are proposed as `scheduling.appointment.completed.v1`
and `clinical.appointment-completed.v1`. Their final spelling belongs in the
versioned event contract PR. No diagnosis, treatment, free text, email, phone,
or document URL may enter this event.

Clinical persists `eventId` before or atomically with creating the draft record.
A uniqueness constraint on the source event (or appointment plus the event type)
makes redelivery harmless. A duplicate acknowledges without creating a second
draft. An event whose pet is absent is retried, then sent to the Clinical DLQ with
only identifiers and an error code for operations to investigate.

## Authorization, privacy and audit

| Actor | Allowed action | Denied action |
|---|---|---|
| Owner | Read only their own pet timeline and permitted summary | Another owner's pets; create or amend diagnosis |
| Assigned veterinarian | Read and complete records for an assigned completed appointment | Unassigned pet records; changing identity or appointment state |
| Receptionist | Register and view the minimum pet identity needed for booking | Diagnosis, treatment and clinical notes |
| Administrator | Operational access explicitly audited | Silent bulk export or bypass of record immutability |

All authorization is performed inside Clinical after B3 claim verification. Gateway
routes but does not decide record access. Structured logs retain `correlationId`,
`eventId`, record identifier and authorization outcome; they redact diagnosis,
treatment, notes, owner contact details and credentials. Demo and automated tests
use synthetic animal, owner and appointment data only.

## Reliability and observability contract

| Signal | Purpose | Initial acceptance target |
|---|---|---|
| `clinical_projection_lag_seconds` | Time from `occurredAt` to committed draft | p95 under 60 seconds in the approved test topology |
| `clinical_event_duplicates_total` | Confirms redelivery is handled without duplicating records | Any duplicate leaves one draft only |
| `clinical_event_dlq_depth` | Detects events that cannot be projected | Alert when greater than zero for the agreed window |
| `clinical_timeline_query_duration_seconds` | Protects the indexed read path | p95 target agreed from an approved load scenario |
| correlation trace | Links completion, outbox, broker and Clinical consumer | One redacted trace can reconstruct the journey |

These are proposed acceptance targets, not claims of current production SLOs. Their
thresholds, dashboard owner and alert receiver are finalized with the platform
topology. A consumer outage must not lose the event: messages retry with bounded
exponential backoff and then enter the DLQ; replay is an explicit, audited action.

## Sequential implementation PRs

The [PR sizing policy](./pr-sizing-policy.md) applies to every slice below: each
code PR stays at or below 500 executable changed lines, including tests,
migrations, scripts and executable configuration. The listed budgets are ceilings,
not targets; a smaller independently verifiable PR is preferred.

| Slice | Repository and branch pattern | Focused change | Maximum executable changed lines | Exit evidence |
|---|---|---|---:|---|
| C1.0 | Docs `docs/c1-clinical-read-model` | Freeze data, event, security and recovery gates | 0 | Architecture review; no ambiguous owner |
| C1.1a | Clinical DB `feat/clinical-pets-schema` | `clinical` role, pets table and owner index only | 250 | Clean migration and least-privilege test |
| C1.1b | Clinical API `feat/pet-booking-eligibility` | Read-only internal eligibility port, adapter and negative authorization test | 450 | Contract test with synthetic pet data |
| C1.1c | Clinical API `feat/clinical-timeline-read` | Bounded timeline query, pagination and owner-cross-read rejection | 450 | Index-backed query and authorization tests |
| C1.2a | Scheduling DB `feat/appointment-state-schema` | Appointment states, business-hours and no-overlap constraint | 300 | Migration, overlap and invalid-hours tests |
| C1.2b | Scheduling API `feat/booking-clinical-adapter` | Clinical adapter, timeout, fail-closed behavior and contract test | 450 | Unavailable-Clinical and valid-pet tests |
| C1.3a | Scheduling DB `feat/appointment-outbox` | Outbox table and atomic completion/outbox persistence | 350 | Transaction rollback test |
| C1.3b | Scheduling API `feat/publish-appointment-completed` | Versioned event schema and outbox publisher | 450 | Published payload schema and correlation test |
| C1.3c | Clinical DB `feat/clinical-processed-events` | Idempotency table and unique event source protection | 250 | Duplicate event migration test |
| C1.3d | Clinical API `feat/consume-appointment-completed` | Consumer, draft creation, bounded retry and DLQ hand-off | 500 | Duplicate, delayed and failing-consumer tests |
| C1.4a | Scheduling Portal `feat/appointment-booking` | Authorized booking remote only | 450 | Role and form validation tests |
| C1.4b | Clinical Portal `feat/pet-timeline` | Authorized timeline remote only | 450 | Owner/veterinarian authorization tests |
| C1.4c | Front or Gateway `feat/clinical-routes` | Remote registration or route declaration, never clinical rules | 200 | Route and module-loading tests |
| C1.5 | One repository per evidence PR | E2E harness, metrics, DLQ drill or restore proof; split again if needed | 500 | Exact command, redacted result and commit matrix |

Every increment starts from the target branch in a child branch and enters through
a reviewed PR. Promotion follows the repository branching policy by applying
accepted commits with `cherry-pick -x`; permanent branches are not merged into
each other. C1.3 does not start until C1.1 and C1.2 have their independent tests.
No slice may include a second repository merely to keep an end-to-end scenario in
one PR; the integration record links the already-reviewed repository slices.

## Explicit bottlenecks and stop conditions

| Bottleneck | Prevention | Stop condition |
|---|---|---|
| Security is incomplete | Enforce B3.0/B3.1 before role-protected endpoints | Claims or service identity are still provisional |
| Scheduling and Clinical form a cycle | One synchronous eligibility query; completion returns asynchronously by event | A second synchronous callback is proposed |
| Sensitive data leaks into events | Minimal payload and schema validation in producer and consumer | Event/log contains medical free text or contact data |
| Duplicate or reordered delivery | Outbox, `processed_events`, unique source reference, idempotent consumer | A replay creates a second draft or changes a signed record |
| Query slows as history grows | Bounded pagination and owned indexes | Timeline requires an unbounded scan or cross-domain join |
| Hidden recovery gap | Backup, restore and DLQ replay drill before release | No successful restore or replay evidence |
| Scope turns into analytics | Keep read model inside Clinical; require a separate ADR for any later warehouse | A new shared data store is proposed without owner and retention policy |

## Definition of done for C1

C1 is complete only when a real, authorized user can book an eligible pet, the
appointment completion reaches Clinical exactly once despite redelivery, the
appropriate actor can read the indexed timeline, an unauthorized actor is
rejected, and the team can show a redacted trace, metrics, DLQ/replay drill and
restore evidence from the approved topology.

## References

- `02-domain/domain-map.md` — bounded-context ownership and Clinical as the long-term data asset.
- `02-domain/domain-events.md` — `AppointmentCompleted` and at-least-once delivery model.
- `06-data/models.md` — target Scheduling and Clinical schemas and indexes.
- `05-architecture/decisions/records/ADR-003-message-broker.md` — RabbitMQ and DLQ decision.
- `05-architecture/decisions/records/ADR-013-monolith-first-strangler-fig.md` — extraction order.
- `10-devops/b3-identity-implementation-plan.md` in PR #40 — B3 prerequisite and service-identity boundary.
- Issue #19 — requested official repositories for Scheduling and Clinical.
