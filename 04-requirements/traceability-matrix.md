# Traceability Matrix

> Traceability connects every line of code to its business justification.
> It allows answering: "Why does this function exist?" and "Which HU covers this part of the system?"
> It also identifies: unimplemented requirements and code without a requirement (possible technical debt).

---

## How to use this matrix

```
Requirement → HU → Test Case → Implementation → Service

If a requirement has no HU: it is not planned
If a HU has no test case: it has no completeness criterion
If a test case has no implementation: there is test technical debt
If there is code without an HU: possible gold-plating or bug introduced without a story
```

**Reading this matrix honestly:** `barbersaas-backend/src/test/java` contains **zero test
files** as of this review (verified by direct file count, not assumed). Every row below has
"Tests that verify it" = **None**. This is not a template gap to fill in later — it's the
real, current state, and the single largest item in "Identified gaps" below.

---

## FR → HU → Test → Service matrix

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR-001 | User registration | HU-AUTH-001 | None | `auth` | ✅ Implemented, untested |
| FR-002 | Login, JWT issuance | HU-AUTH-001 | None | `auth` | ✅ Implemented, untested |
| FR-003 | Password recovery via email | HU-AUTH-002 | None | `auth`, `notification` | ✅ Implemented, untested |
| FR-004 | Barbershop self-registration + trial | HU-AUTH-003 | None | `auth`, `barbershop` | ✅ Implemented, untested |
| FR-005 | Service catalog configuration | HU-SHOP-001 | None | `barberservice` | ✅ Implemented, untested |
| FR-006 | Barber weekly schedule, no overlap | HU-SHOP-001 | None | `schedule` | ✅ Implemented, untested |
| FR-007 | Schedule exceptions override | HU-SHOP-001 | None | `schedule` | ✅ Implemented, untested |
| FR-008 | Appointment booking, anti-double-booking | HU-APPT-001 | None | `appointment` | ✅ Implemented, untested — **highest-risk untested path**: the pessimistic lock has no automated concurrency test |
| FR-009 | Price snapshot immutability | HU-APPT-001 | None | `appointment` | ✅ Implemented, untested |
| FR-010 | Reward coupon auto-applied at booking | HU-APPT-001 | None | `appointment`, `loyalty` | ✅ Implemented, untested |
| FR-011 | Cancellation within policy window | HU-APPT-002 | None | `appointment` | ✅ Implemented, untested |
| FR-012 | Automatic NO_SHOW marking | HU-APPT-002 | None | `appointment` | ✅ Implemented, untested |
| FR-013 | Grant loyalty sticker | HU-LOY-001 | None | `loyalty` | ✅ Implemented, untested |
| FR-014 | Reject redemption with insufficient stickers | HU-LOY-001 | None | `loyalty` | ✅ Implemented, untested |
| FR-015 | Issue reward coupon on redemption | HU-LOY-001 | None | `loyalty` | ✅ Implemented, untested |
| FR-016 | Record income/expense | HU-FIN-001 | None | `finance` | ✅ Implemented, untested |
| FR-017 | Reject non-positive finance amount | HU-FIN-001 | None | `finance` | ✅ Implemented, untested |
| FR-018 | Track stock, register movements | HU-INV-001 | None | `inventory` | ✅ Implemented, untested |
| FR-019 | Low-stock alert flag | HU-INV-001 | None | `inventory` | ✅ Implemented, untested |
| FR-020 | Notify on booking/confirm/cancel | HU-NOTIF-001 | None | `notification`, `appointment` | ✅ Implemented, untested |
| FR-021 | Day-before reminder job | HU-NOTIF-001 | None | `notification`, `appointment` | ✅ Implemented, untested |
| FR-022 | Notify on completion | HU-NOTIF-001 | None | `notification`, `appointment` | 🔴 Not implemented |
| FR-023 | Super Admin dashboard & barbershop list | HU-SADMIN-001 | None | `dashboard`, `barbershop` | ✅ Implemented, untested |
| FR-024 | Subscription plan management | HU-SADMIN-001 | None | `plan` | ✅ Implemented, untested |
| FR-025 | Only Super Admin transitions billing status | HU-SADMIN-001 | None | `barbershop` | ✅ Implemented, untested |
| FR-026 | Automatic trial expiration | HU-SADMIN-001 | None | `barbershop` | 🔴 Not implemented |
| FR-027 | Tenant resolved from JWT into TenantContext | HU-TENANT-001 | None | `security` | ✅ Implemented, untested — **second highest-risk untested path**: no automated test proves cross-tenant isolation holds |
| FR-028 | Cross-tenant access rejected | HU-TENANT-001 | None | `security` | ✅ Implemented, untested |

---

## NFR → Validation matrix

| NFR ID | Description | How it is validated | Tool | Status |
|--------|-------------|-------------------|------|--------|
| NFR-001 | Performance targets (p95/p99 latency) | Not validated | None configured | 🔴 Pending — see `04-requirements/non-functional.md` |
| NFR-002 | 99.9% availability SLO | Not validated | None configured (no health checks exist) | 🔴 Pending |
| NFR-004 | JWT authentication, RBAC | Manual code review only (this document + `00-governance/security-policy.md`) | None automated | 🟡 Partially validated (manual only) |
| NFR-005 | Observability (logs/metrics/traces/alerts) | Not validated | None configured | 🔴 Pending |
| NFR-006 | Test coverage ≥ 80% | Not validated | None configured | 🔴 Pending — actual coverage is 0% |

---

## Inverse traceability: HU → FR

| HU | Title | FR(s) it implements | Sprint |
|----|-------|---------------------|--------|
| HU-AUTH-001 | Register and log in | FR-001, FR-002 | Not yet scheduled |
| HU-AUTH-002 | Password recovery | FR-003 | Not yet scheduled |
| HU-AUTH-003 | Barbershop self-registration | FR-004 | Not yet scheduled |
| HU-SHOP-001 | Service catalog & schedules | FR-005, FR-006, FR-007 | Not yet scheduled |
| HU-APPT-001 | Book an appointment | FR-008, FR-009, FR-010 | Not yet scheduled |
| HU-APPT-002 | Cancel / auto no-show | FR-011, FR-012 | Not yet scheduled |
| HU-LOY-001 | Earn and redeem loyalty | FR-013, FR-014, FR-015 | Not yet scheduled |
| HU-FIN-001 | Track income/expenses | FR-016, FR-017 | Not yet scheduled |
| HU-INV-001 | Track inventory | FR-018, FR-019 | Not yet scheduled |
| HU-NOTIF-001 | Appointment notifications | FR-020, FR-021, FR-022 | Not yet scheduled |
| HU-SADMIN-001 | Manage barbershops & plans | FR-023, FR-024, FR-025, FR-026 | Not yet scheduled |
| HU-TENANT-001 | Tenant isolation | FR-027, FR-028 | Not yet scheduled |

> "Sprint" is left unscheduled across the board — see `00-governance/agile-conventions.md`:
> no formal sprint-planning cycle with dates has been run yet for this backlog.

---

## Status legend

| Status | Meaning |
|--------|---------|
| ✅ Implemented, untested | Feature works in the code, verified by direct code review, but has zero automated test coverage |
| 🟡 Partially validated | Some manual/non-automated check exists, no automated gate |
| 🔴 Not implemented / Pending | Feature does not exist yet, or validation mechanism does not exist yet |

---

## Identified gaps (requirements without coverage)

| Gap type | Description | Required action | Owner | Date |
|----------|-------------|----------------|-------|------|
| **Systemic — no tests at all** | Every implemented FR (26 of 28) has zero automated test coverage | Write unit/integration tests, starting with FR-008 (double-booking lock) and FR-027/028 (tenant isolation) — these are the two paths where a silent bug has the worst blast radius | Whole team | Not yet scheduled — flag for next `00-governance/agile-conventions.md` sprint planning |
| FR without full implementation | FR-022 (notify on completion) has no HU-side implementation | Either implement `NotificationService` call in `AppointmentService.complete()`, or explicitly re-scope FR-022 out of MVP | Carlos Leal (Tech Lead) | Not yet scheduled |
| FR without full implementation | FR-026 (automatic trial expiration) has no scheduled job implemented | Add a scheduled job analogous to `AppointmentReminderJob` for barbershop trial expiration | Carlos Leal (Tech Lead) | Tracked as "in progress" in `01-context/scope.md`, feature F-14 |
| NFR without validation infrastructure | NFR-001, 002, 005, 006 have no tooling at all (no CI, no load test, no coverage report, no observability stack) | Prioritize NFR-006 (testing) first — the others depend less on a live production environment that doesn't exist yet | Whole team | Before Phase 2 (Controlled Launch), per `03-product/vision.md` |

---

## How to maintain this matrix

1. When an HU is created: add the row in the FR → HU → Test → Service section
2. When a test is written: replace "None" in the "Tests that verify it" column with the actual test file path, and update Status to ✅ Validated
3. When an HU is completed: status here reflects implementation, not sprint completion — keep it in sync with `04-requirements/user-stories.md`
4. At each Sprint Planning: review gaps and assign actions

---

## Correlations

- User Stories → `04-requirements/user-stories.md`
- Functional Requirements → `04-requirements/functional.md`
- Non-Functional Requirements → `04-requirements/non-functional.md`
- DoD that determines when an HU is Done → `00-governance/definition-of-done.md`
