# Traceability Matrix

---

## FR → HU → Test → Service matrix

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR-001 | Self-registration as CUSTOMER | HU-IAM-001 | `user.register.spec.ts` | identity-service | 🔴 Pending |
| FR-002 | Login issuing a JWT | HU-IAM-002 | `user.login.spec.ts` | identity-service | 🔴 Pending |
| FR-003 | Receptionist registers a walk-in customer/staff | HU-IAM-003 | `user.createByStaff.spec.ts` | identity-service | 🔴 Pending |
| FR-004 | Browse the product catalog | HU-CAT-001 | `product.list.spec.ts` | catalog-service | 🔴 Pending |
| FR-005 | Admin manages products | HU-CAT-002 | `product.create.spec.ts`, `product.priceInvariant.spec.ts` | catalog-service | 🔴 Pending |
| FR-006 | Stock decrement/restore on order events | HU-CAT-002 | `product.stockConsumer.spec.ts` | catalog-service | 🔴 Pending |
| FR-007 | Cart checkout | HU-SALES-001 | `order.checkout.spec.ts` | sales-service | 🔴 Pending |
| FR-008 | View own order history | HU-SALES-002 | `order.listOwn.spec.ts` | sales-service | 🔴 Pending |
| FR-009 | Receptionist creates in-store order | HU-SALES-003 | `order.inStore.spec.ts` | sales-service | 🔴 Pending |
| FR-010 | Staff marks order PAID/FULFILLED/CANCELLED | HU-SALES-003 | `order.statusTransition.spec.ts` | sales-service | 🔴 Pending |
| FR-011 | Book appointment inside business hours | HU-SCHED-001 | `appointment.businessHours.spec.ts` | scheduling-service | 🔴 Pending |
| FR-012 | Reject overlapping appointment | HU-SCHED-002 | `appointment.noDoubleBooking.spec.ts` | scheduling-service | 🔴 Pending |
| FR-013 | View/cancel own appointment | HU-SCHED-003 | `appointment.cancelOwn.spec.ts` | scheduling-service | 🔴 Pending |
| FR-014 | Receptionist manages appointments for a customer | HU-SCHED-004 | `appointment.staffAssisted.spec.ts` | scheduling-service | 🔴 Pending |
| FR-015 | Professional views agenda / completes appointment | HU-SCHED-004 | `appointment.complete.spec.ts` | scheduling-service | 🔴 Pending |
| FR-016 | Register a pet | HU-CLIN-001 | `pet.register.spec.ts` | clinical-service | 🔴 Pending |
| FR-017 | Vet adds medical record | HU-CLIN-002 | `medicalRecord.create.spec.ts` | clinical-service | 🔴 Pending |
| FR-018 | Owner views pet medical history | HU-CLIN-003 | `medicalRecord.listOwn.spec.ts` | clinical-service | 🔴 Pending |
| FR-019 | Confirmation email on order/appointment | HU-NOTIF-001 | `notification.confirmation.spec.ts` | notification-service | 🔴 Pending |
| FR-020 | Cancellation email | HU-NOTIF-001 | `notification.cancellation.spec.ts` | notification-service | 🔴 Pending |

---

## NFR → Validation matrix

| NFR ID | Description | How it is validated | Tool | Status |
|--------|-------------|-------------------|------|--------|
| NFR-001 | P95 < 300ms on critical endpoints | Manual load test | k6 | 🔴 Pending |
| NFR-004 | JWT authentication + RBAC | Manual security review against `security-rules.md` | Manual + Jest | 🔴 Pending |
| NFR-006 | ≥ 80% test coverage, ≥ 90% in domain layer | Coverage report | Jest --coverage | 🔴 Pending |
| NFR-007 | Portable Docker images, DB-engine swappable via Prisma | Manual verification (build image, swap `provider` in one service) | Docker, Prisma | 🔴 Pending |

---

## Inverse traceability: HU → FR

| HU | Title | FR(s) it implements | Sprint |
|----|-------|---------------------|--------|
| HU-IAM-001 | Customer self-registration | FR-001 | Sprint 1 |
| HU-IAM-002 | Login | FR-002 | Sprint 1 |
| HU-IAM-003 | Receptionist registers a walk-in customer | FR-003 | Sprint 3 |
| HU-CAT-001 | Browse product catalog | FR-004 | Sprint 1 |
| HU-CAT-002 | Admin manages the catalog | FR-005, FR-006 | Sprint 1 |
| HU-SALES-001 | Add to cart and checkout | FR-007 | Sprint 2 |
| HU-SALES-002 | View my order history | FR-008 | Sprint 2 |
| HU-SALES-003 | Receptionist creates an in-store order | FR-009, FR-010 | Sprint 3 |
| HU-SCHED-001 | Book an appointment within business hours | FR-011 | Sprint 2 |
| HU-SCHED-002 | Prevent double-booking | FR-012 | Sprint 2 |
| HU-SCHED-003 | Cancel my appointment | FR-013 | Sprint 3 |
| HU-SCHED-004 | Receptionist manages appointments | FR-014, FR-015 | Sprint 3 |
| HU-CLIN-001 | Register a pet | FR-016 | Sprint 2 |
| HU-CLIN-002 | Veterinarian adds a medical record | FR-017 | Sprint 4 |
| HU-CLIN-003 | Owner views pet medical history | FR-018 | Sprint 4 |
| HU-NOTIF-001 | Confirmation and cancellation emails | FR-019, FR-020 | Sprint 5 |

---

## Status legend

| Status | Meaning |
|--------|---------|
| ✅ Done | Implemented, tested, and in production |
| 🟡 In progress | Under development in the current sprint |
| 🔴 Pending | In the backlog, not started |
| ⏸ Blocked | Has an external blocker |
| ❌ Cancelled | Removed from scope |

---

## Identified gaps (requirements without coverage)

None at the documentation stage — every FR maps to exactly one HU and every HU maps to at
least one FR. This will be re-checked at the end of each sprint once implementation starts,
and updated to ✅/🟡 as tests are written.

---

## Correlations

- User Stories → `04-requirements/user-stories.md`
- Functional Requirements → `04-requirements/functional.md`
- Non-Functional Requirements → `04-requirements/non-functional.md`
- DoD that determines when an HU is Done → `00-governance/definition-of-done.md`
