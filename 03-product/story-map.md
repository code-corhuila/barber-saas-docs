# Story Map — BarberSaaS

> Built from what is already documented in `03-product/vision.md` (roadmap, roles),
> `01-context/scope.md` and `03-product/product-backlog.md` (epics/features). No new user
> activity is invented here — this file only re-arranges existing, already-agreed scope into
> story-map form (backbone of activities per role + release horizons), per SPEC-008
> (2026-09-27).

---

## How to read this map

- **Backbone (rows):** the big steps each role walks through, left to right, in the order
  they'd naturally do them.
- **Release horizon (columns):** the same 4 horizons already defined in
  `03-product/vision.md` § "High-level roadmap" — H1 Private Beta, H2 Controlled Launch,
  H3 Growth, H4 Scale. A cell is filled only where that activity is already scoped for that
  horizon in `vision.md` or `product-backlog.md`.
- ✅ = built and Done today. 🔄 = in progress. ⛔ = scoped for a later horizon, not started.

---

## Backbone — CLIENT

| Activity | H1 Private Beta (now) | H2 Controlled Launch | H3 Growth | H4 Scale |
|----------|------------------------|----------------------|-----------|----------|
| Register / log in | ✅ HU-AUTH-001-A/B | — | — | — |
| Discover a barbershop and its services | ✅ (via app catalog, `barberservice`) | — | ⛔ Client-facing web app (QR-based, no install) | ⛔ Marketplace / discovery platform |
| Book an appointment (no double-booking) | ✅ HU-APPT-001-A | — | — | — |
| Get notified (booked/confirmed/cancelled, reminder) | ✅ HU-NOTIF-001-A/B (gap: completion) | — | — | — |
| Cancel an appointment within policy | ✅ HU-APPT-002-A | — | — | — |
| Earn and redeem loyalty stickers | ✅ HU-LOY-001, auto-apply via HU-APPT-001-B | — | — | — |
| Pay for a service in-app | — | ⛔ Trial-to-paid conversion flow (barbershop-side); client payment is Phase 3 | ⛔ Automated billing (PayU/Stripe/PSE) | — |
| Sync appointments with personal calendar | — | — | — | ⛔ Google/Apple Calendar integration |

## Backbone — BARBER

| Activity | H1 Private Beta (now) | H2 Controlled Launch | H3 Growth | H4 Scale |
|----------|------------------------|----------------------|-----------|----------|
| Register / log in | ✅ HU-AUTH-001-A/B | — | — | — |
| View my agenda / schedule | ✅ (`schedule`, HU-SHOP-001-B) | — | — | — |
| Track walk-in clients (not booked in-app) | 🔄 In progress (per `product-backlog.md` F-13) | — | — | — |
| Grant loyalty stickers to clients | ✅ HU-LOY-001 | — | — | — |
| View my stats / appointment history | ✅ (`dashboard`, mobile `(barber)/stats`) | — | — | — |

## Backbone — ADMIN (barbershop owner)

| Activity | H1 Private Beta (now) | H2 Controlled Launch | H3 Growth | H4 Scale |
|----------|------------------------|----------------------|-----------|----------|
| Self-register the barbershop, start 60-day trial | ✅ HU-AUTH-003 | — | — | — |
| Configure service catalog | ✅ HU-SHOP-001-A | — | — | — |
| Manage staff and weekly schedules (+ exceptions) | ✅ HU-SHOP-001-B | — | — | — |
| Manage appointments (cancel on behalf of client) | ✅ HU-APPT-002-A | — | — | — |
| Configure and track loyalty program | ✅ HU-LOY-001 | — | — | — |
| Track income and expenses | ✅ HU-FIN-001 | — | — | — |
| Track product inventory / low-stock alerts | ✅ HU-INV-001 | — | — | — |
| Convert from trial to paid subscription | — | ⛔ Trial-to-paid conversion flow (`vision.md` H2) | — | — |
| Generate advanced reports / exports | — | — | ⛔ Advanced analytics / PDF-Excel exports | — |
| Manage multiple barbershop locations | — | — | — | ⛔ Multi-location chain management |

## Backbone — SUPER_ADMIN (platform operator)

| Activity | H1 Private Beta (now) | H2 Controlled Launch | H3 Growth | H4 Scale |
|----------|------------------------|----------------------|-----------|----------|
| View all registered barbershops | ✅ HU-SADMIN-001-A | — | — | — |
| Manually transition trial → active (billing) | ✅ HU-SADMIN-001-A | — | — | — |
| Manage subscription plans (Starter/Profesional/Premium) | ✅ HU-SADMIN-001-A | — | — | — |
| Automatically expire an unconverted trial | 🔴 Not implemented (HU-SADMIN-001-B, F-14) | — | — | — |
| Operate billing without manual bank transfer/Nequi confirmation | — | — | ⛔ Automated billing (PayU/Stripe/PSE) | — |
| Curate a client-facing marketplace | — | — | — | ⛔ Marketplace / discovery platform |

---

## Release cut (walking skeleton already shipped)

The vertical slice already built and `✅ Done` across all 4 roles for **H1 Private Beta**
covers: authentication for all roles, barbershop self-registration with trial,
service/staff/schedule configuration, appointment booking with anti-double-booking,
cancellation, loyalty earn/redeem, finance tracking, inventory tracking, appointment
notifications (minus the completion-notification gap), and Super Admin manual billing
management (minus automatic trial expiration). This matches `03-product/product-backlog.md`
§ Epics almost entirely — the only two documented gaps inside H1 are HU-NOTIF-001-A's
completion-notification scenario and HU-SADMIN-001-B (automatic trial expiration).

---

## Correlations

- Roadmap horizons and dates → `03-product/vision.md` § "High-level roadmap"
- Sprint-ready HUs behind each ✅ cell → `04-requirements/user-stories.md`
- Everything in H3/H4 columns, organized as a real backlog → `03-product/mvp2-backlog.md`
- Epic-level source → `03-product/product-backlog.md`
