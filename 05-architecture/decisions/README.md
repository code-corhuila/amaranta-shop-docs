# ADRs — Architecture Decision Records

ADRs document important architectural decisions. Each file = one decision.

## How to create an ADR

1. Copy `_template-adr.md`
2. Name it `ADR-NNN-short-title.md` (e.g.: `ADR-001-message-broker.md`)
3. Fill it in completely — especially the evaluated alternatives
4. Once accepted, the status is **permanent** (it is not deleted, it is "Superseded" by another ADR)

## Possible statuses

- `Proposed` — under discussion
- `Accepted` — approved by the team
- `Rejected` — evaluated and discarded (document why)
- `Superseded` — superseded by ADR-NNN (indicate which one)

## ADR register

| # | Title | Status | Date |
|---|-------|--------|------|
| ADR-001 | Documentation Language | Accepted | 2024-01-10 |
| ADR-002 | Relational Database Engine and ORM | Accepted | 2026-08-25 |
| ADR-003 | Asynchronous Message Broker | Accepted | 2026-08-25 |
| ADR-004 | Service Discovery | Accepted | 2026-08-25 |
| ADR-005 | API Gateway | Accepted | 2026-08-25 |
| ADR-006 | Backend Framework | Accepted | 2026-08-25 |
| ADR-007 | Brand Color Palette: Fiery Ocean | Accepted | 2026-09-20 |

> Add rows here as you create ADRs in `records/`
