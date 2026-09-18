# Summary of the Traceability Matrix — BarberSaaS

> Executive summary connecting Functional Requirements (FR), User Stories (HU), Tests, and Services.

---

## 📊 Overview

| Category | Status / Count | Details |
| :--- | :--- | :--- |
| **Total FRs** | 28 | FR-001 to FR-028 covering core barbershop operations |
| **Implemented FRs** | 26 / 28 (92.8%) | Code exists in backend microservices |
| **Unimplemented FRs** | 2 / 28 (7.2%) | FR-022 (completion notification) & FR-026 (trial expiration) |
| **Test Coverage** | 0% | Zero test files in `barbersaas-backend/src/test/java` |
| **NFR Validation** | 0% Automated | All NFRs (NFR-001 to NFR-006) are pending automated tooling |

---

## 🎯 Highest-Risk Untested Paths

* **FR-008 (Anti-double-booking):** Pessimistic lock logic lacks automated concurrency testing.
* **FR-027 / FR-028 (Tenant Isolation):** No automated tests verify cross-tenant data boundaries in `TenantContext`.

---

## 🚨 Identified Gaps & Required Actions

1. **Systemic Test Deficit:** Write unit/integration tests starting with FR-008 and FR-027.
2. **Missing Features:**
   * **FR-022:** Call `NotificationService` inside `AppointmentService.complete()`.
   * **FR-026:** Implement scheduled job for trial expiration.
3. **NFR Tooling:** Set up CI/CD pipeline and automated coverage targeting $\ge 80\%$ (NFR-006).

---

## 🔗 Related Documentation

* User Stories: `04-requirements/user-stories.md`
* Requirements: `04-requirements/functional.md` & `04-requirements/non-functional.md`
