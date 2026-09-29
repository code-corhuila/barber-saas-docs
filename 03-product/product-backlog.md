# Product Backlog — BarberSaaS

> Epic-level backlog, derived from the MVP feature list already documented in
> `01-context/scope.md` and the completion status in `01-context/overview.md`.
> **What is deliberately NOT in this document:** Gherkin acceptance criteria, story points and
> sprint assignment. The acceptance criteria live in the 12 refined user stories of
> `04-requirements/user-stories.md` (`HU-AUTH-001` … `HU-TENANT-001`), linked per epic below.
> Story points and sprints are not agreed yet, so they are not invented here either; a story
> gets them when it passes the Definition of Ready (`00-governance/definition-of-ready.md`).

---

## How to read the status column

| Status | Meaning |
|--------|---------|
| ✅ Done | Implemented in the first-cut **prototype** (`code-corhuila/barber-saas`, category A), per `01-context/overview.md` → "Completed so far" |
| 🔄 In progress | Implementation started in the prototype but not complete, per the same source |
| ⛔ Not started | Documented as planned (Phase 2+) but no implementation exists yet |

> **Status is the prototype's, not the polyrepo's.** Since ADR-004 every epic is rebuilt in
> its domain repositories (`barber-saas-<domain>-{db,api,app}`), which today hold only
> README and CODEOWNERS. "Done" here means the business rules are proven and are the source
> to port — not that the microservice exists. Progress in the polyrepo is tracked on the
> board through each story's **Environment** field (Dev / QA / Main).

---

## Epics

| ID | Epic | Description | Owning domain (ADR-004) | User stories | Status (prototype) |
|----|------|-------------|-------------------------|--------------|--------------------|
| EP-001 | Identity & Access | Registration, login, JWT sessions, password recovery, self-registration wizard, 4-role RBAC | identity-auth (+ workflow for owner onboarding) | HU-AUTH-001, HU-AUTH-002, HU-AUTH-003 | ✅ Done |
| EP-002 | Barbershop & Staff Management | Service catalog configuration, employee (barber) management, weekly schedules and exceptions | barbershop, schedule | HU-SHOP-001 | ✅ Done |
| EP-003 | Appointment Booking | Real-time availability, anti-double-booking, 6-state lifecycle, cancellation/reschedule, reminders | appointment | HU-APPT-001, HU-APPT-002 | ✅ Done (core booking) / 🔄 walk-in tracking |
| EP-004 | Loyalty & Rewards | Sticker-based loyalty card, reward configuration, redemption, automatic coupon application | loyalty | HU-LOY-001 | ✅ Done |
| EP-005 | Financial Tracking | Manual income/expense records, revenue dashboard | finance-inventory | HU-FIN-001 | ✅ Done |
| EP-006 | Inventory Management | Product stock tracking, minimum-stock alerts | finance-inventory | HU-INV-001 | ✅ Done |
| EP-007 | Notifications | In-app + push (FCM) + email notifications for appointment events and password recovery | notifications | HU-NOTIF-001 | ✅ Done (FCM in development build) |
| EP-008 | Platform Administration | Super Admin dashboard, barbershop account lifecycle, subscription plan management | platform-admin | HU-SADMIN-001 | ✅ Done (dashboard, plans) / 🔄 trial expiration automation |
| EP-009 | Multi-tenancy & Security | `barbershop_id` tenant isolation from the JWT — the cross-cutting foundation every other epic depends on | every domain `-api` | HU-TENANT-001 | ✅ Done |

---

## Features per epic

Numbering follows the MVP feature table in `01-context/scope.md` — feature `#N` there maps
to `F-N` here, so both documents stay traceable to each other.

### EP-001 — Identity & Access
| # | Feature | Status |
|---|---------|--------|
| F-2 | Authentication & roles (JWT, 4 roles, password recovery via 6-digit email code) | ✅ Done |
| F-11 | Self-registration & 60-day trial wizard | ✅ Done |

### EP-002 — Barbershop & Staff Management
| # | Feature | Status |
|---|---------|--------|
| F-4 | Service catalog (per-barbershop services, prices, duration) | ✅ Done |
| F-5 | Staff & schedule management (barbers, weekly schedules, exceptions) | ✅ Done |

### EP-003 — Appointment Booking
| # | Feature | Status |
|---|---------|--------|
| F-3 | Appointment booking (availability, anti-double-booking, state machine) | ✅ Done |
| F-13 | Walk-in client tracking | 🔄 In progress |

### EP-004 — Loyalty & Rewards
| # | Feature | Status |
|---|---------|--------|
| F-6 | Loyalty program (stickers, reward coupons, 100% discount application) | ✅ Done |

### EP-005 — Financial Tracking
| # | Feature | Status |
|---|---------|--------|
| F-7 | Financial tracking (manual income/expense, revenue dashboard) | ✅ Done |

### EP-006 — Inventory Management
| # | Feature | Status |
|---|---------|--------|
| F-8 | Inventory management (stock, minimum-stock alerts) | ✅ Done |

### EP-007 — Notifications
| # | Feature | Status |
|---|---------|--------|
| F-9 | Push (FCM), in-app, and email notifications | ✅ Done |

### EP-008 — Platform Administration
| # | Feature | Status |
|---|---------|--------|
| F-10 | Super Admin dashboard (barbershops, metrics) | ✅ Done |
| F-12 | Subscription plan management (Starter/Profesional/Premium) | ✅ Done |
| F-14 | Trial expiration automation | 🔄 In progress |

### EP-009 — Multi-tenancy & Security
| # | Feature | Status |
|---|---------|--------|
| F-1 | Multi-tenant architecture (`barbershop_id` + `TenantContext`) | ✅ Done |

---

## Not yet on this backlog (Phase 3+, out of current scope)

Per `01-context/scope.md` → Out of Scope. Listed here only so nobody re-proposes them as
"missing" MVP work — they are deliberately deferred, not forgotten:

- Automated payment processing (Stripe/PSE/Nequi) — Phase 3
- Client-facing web app (QR-based booking) — Phase 3
- Advanced analytics / PDF-Excel exports — Phase 3
- Multi-location barbershop chains — Phase 4 (post-MVP, no committed date)
- Marketplace / discovery platform — not currently planned
- Google/Apple Calendar integration — not currently planned

---

## Next step to make this backlog sprint-ready

This document stops at the epic/feature level on purpose. Every epic already has its refined
user stories (column above); what they still lack before a sprint is an estimate and the rest
of the Definition of Ready (`00-governance/definition-of-ready.md`) — in particular the owning
repositories and the contract/data they need, now that each epic is rebuilt per domain. A new
🔄 or future item goes through `spec-forge` (or manual refinement) into a new HU first.

---

## Correlations

- MVP feature source of truth → `01-context/scope.md`
- Implementation status source → `01-context/overview.md` → "Current status"
- Product vision these epics serve → `03-product/vision.md`
- Refined user stories per epic → `04-requirements/user-stories.md`
