# Product Backlog — BarberSaaS

> Epic-level backlog, derived from the MVP feature list already documented in
> `01-context/scope.md` and the completion status in `01-context/overview_en.md`.
> **What is deliberately NOT in this document:** story points, Gherkin acceptance
> criteria, and sprint assignment. None of those exist yet as real, agreed data —
> `04-requirements/user-stories.md` is still the unfilled template. Inventing them here
> would misrepresent estimation/planning work that hasn't actually happened. Once real
> HUs are refined (see `00-governance/definition-of-ready.md`), they belong in
> `04-requirements/user-stories.md`, and this file should be updated to reference them.

---

## How to read the status column

| Status | Meaning |
|--------|---------|
| ✅ Done | Implemented in the current codebase, per `01-context/overview_en.md` → "Completed so far" |
| 🔄 In progress | Implementation started but not complete, per the same source |
| ⛔ Not started | Documented as planned (Phase 2+) but no implementation exists yet |

---

## Epics

| ID | Epic | Description | Owning module(s) | Status |
|----|------|-------------|-------------------|--------|
| EP-001 | Identity & Access | Registration, login, JWT sessions, password recovery, self-registration wizard, 4-role RBAC | `auth`, `security` | ✅ Done |
| EP-002 | Barbershop & Staff Management | Service catalog configuration, employee (barber) management, weekly schedules and exceptions | `barbershop`, `employee`, `schedule` | ✅ Done |
| EP-003 | Appointment Booking | Real-time availability, anti-double-booking, 6-state lifecycle, cancellation/reschedule, reminders | `appointment` | ✅ Done (core booking) / 🔄 walk-in tracking |
| EP-004 | Loyalty & Rewards | Sticker-based loyalty card, reward configuration, redemption, automatic coupon application | `loyalty` | ✅ Done |
| EP-005 | Financial Tracking | Manual income/expense records, revenue dashboard | `finance` | ✅ Done |
| EP-006 | Inventory Management | Product stock tracking, minimum-stock alerts | `inventory` | ✅ Done |
| EP-007 | Notifications | In-app + push (FCM) + email notifications for appointment events and password recovery | `notification` | ✅ Done (FCM in development build) |
| EP-008 | Platform Administration | Super Admin dashboard, barbershop account lifecycle, subscription plan management | `dashboard`, `plan`, `barbershop` (super-admin scope) | ✅ Done (dashboard, plans) / 🔄 trial expiration automation |
| EP-009 | Multi-tenancy & Security | `barbershop_id` tenant isolation, `TenantContext`, JWT filter — the cross-cutting foundation every other epic depends on | `security`, `domain` | ✅ Done |

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

This document stops at the epic/feature level on purpose. To move any 🔄 or future item
into a sprint, it needs to go through `spec-forge` (or manual refinement) into a real HU in
`04-requirements/user-stories.md` — with a specific role, Given/When/Then acceptance
criteria, an estimated size, and a Definition of Ready check
(`00-governance/definition-of-ready.md`) — before it can be picked up.

---

## Correlations

- MVP feature source of truth → `01-context/scope.md`
- Implementation status source → `01-context/overview_en.md` → "Current status"
- Product vision these epics serve → `03-product/vision.md`
- Where refined HUs eventually live → `04-requirements/user-stories.md`
