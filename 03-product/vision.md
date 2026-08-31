# Product Vision

> The vision is the team's north star. All sprints, design decisions,
> and trade-offs are evaluated against this vision.
> It must be ambitious yet achievable, inspiring but specific.

---

## Vision statement

**For** independent and small-team barbershop owners in mid-sized Colombian cities (Neiva, Ibagué, Pasto, Cúcuta, Manizales, and similar)
**who** currently run their scheduling, staff, loyalty, and finances through WhatsApp groups, paper notebooks, and verbal agreements
**the** BarberSaaS
**is a** cloud-based, multi-tenant SaaS mobile platform
**that** digitizes the full operation of a barbershop — appointment booking, staff scheduling, loyalty rewards, financial tracking, and inventory — from a single Spanish-language app, with a guaranteed anti-double-booking system and a 60-day free trial
**unlike** global tools such as Booksy or Fresha, which are priced and designed for Western markets and ignore local workflows (walk-in tracking, COP currency, offline-resilient scenarios), or the current status quo of WhatsApp, notebooks, and spreadsheets
**our product** is built natively for the Colombian independent-barbershop market, giving owners the same operational visibility and control that only large salon chains could previously afford, at a fraction of the cost.

---

## Team mission

We exist to bring affordable, locally-relevant digital operations tools to independent barbershops across Colombia, replacing manual and informal processes (WhatsApp, paper, verbal trust) with a single platform that gives owners real visibility into their business, gives barbers clarity over their work, and gives clients a modern, reliable booking experience.

---

## Strategic pillars

| Pillar | Description | Success metrics |
|--------|-------------|----------------|
| **Local relevance** | Native Spanish (Colombia) UX, COP currency, and workflows built around real Colombian barbershop behavior (walk-ins, informal scheduling, low connectivity) rather than adapted Western tooling | % of onboarded barbershops actively using the app vs. reverting to WhatsApp/notebooks; trial-to-paid conversion > 40% |
| **Operational reliability** | Guaranteed booking integrity and dependable notifications, since a failed booking or missed reminder directly costs the shop revenue | Zero double-bookings in first 90 days; API uptime > 99.5%; FCM delivery rate > 95% |
| **Business visibility for owners** | Give barbershop owners the per-barber revenue, appointment, and financial data they've never had access to, so they can manage the business instead of guessing | ≥ 60% of barbershops with active loyalty redemptions; adoption of financial and per-barber stats dashboards |
| **Accessible growth** | Low-friction onboarding and pricing that fits the reality of a small independent barbershop (60-day trial, manual bank transfer/Nequi billing, no credit card required) | 50 active paid barbershops and MRR of COP $5,000,000 within 12 months; churn < 5% |
| **Sustainable architecture** | A modular monolith with clear bounded contexts that can evolve into microservices only when production data justifies it, avoiding premature complexity for a small team | Documented, trigger-based extraction criteria met before any service extraction; single-developer bottleneck mitigated via documentation |

---

## High-level roadmap

```
Ago 2026 ──────── Q4 2026 ──────── Q1-Q2 2027 ──────── Q3-Q4 2027
     │                  │                  │                  │
 [Private Beta]  [Controlled Launch]    [Growth]           [Scale]
  Validate MVP    Production launch    Payments + web    Multi-location +
  with real shops  on Play/App Store    booking            marketplace
```

| Horizon | Period | Objective | Epics / Features | Uncertainty |
|---------|--------|----------|----------------|-------------|
| H1 (Now) — Phase 1: Private Beta | Now – Aug 2026 | Validate the full 4-role workflow end-to-end with real barbershops | Auth, barbershops, employees, services, schedules, appointments, loyalty, finance (8 backend phases — complete); walk-in tracking, trial expiration automation, T&Cs screen (in progress) | Low |
| H2 (Next) — Phase 2: Controlled Launch | Q4 2026 | Move from beta to a production-ready, store-published app with a paid conversion flow | MySQL → PostgreSQL migration; Railway production deployment; Play Store/App Store release; legal (Terms & Privacy Policy); trial-to-paid conversion flow; notification service extraction (if triggered) | Medium |
| H3 (Later) — Phase 3: Growth | Q1–Q2 2027 | Remove manual billing friction and extend reach beyond the mobile app | Client web app (QR-based booking, no install); automated billing (PayU/Stripe/PSE); advanced reporting/PDF export; appointment and auth service extraction (if triggered) | High |
| H3 (Later) — Phase 4: Scale | Q3–Q4 2027 | Support growth beyond single-location independent shops | Multi-location chain management; client discovery marketplace; Google/Apple Calendar integration; analytics service/data warehouse | High |

---

## Product principles

1. **Built for the Colombian independent barbershop, not adapted from abroad:** every feature must reflect how shops in Neiva, Ibagué, or Pasto actually operate today (walk-ins, informal scheduling, COP pricing) rather than porting a Western SaaS workflow.

2. **No double-bookings, ever:** appointment integrity is non-negotiable — the platform guarantees this at the database level (pessimistic locking), not just in the UI, because a broken promise here directly costs the barbershop money and trust.

3. **Low-friction adoption over feature completeness:** onboarding must work for owners with basic (not technical) comfort with apps — a 60-day trial with no credit card, a 3-step registration wizard, and features that are understandable without training.

4. **Resilient by design, not by luck:** notifications are persisted before delivery is attempted, and core flows are built to degrade gracefully (e.g., FCM failures never block an operation) — because target users have inconsistent connectivity and cannot tolerate silent failures.

5. **Grow the architecture only when the business proves it's needed:** stay a modular monolith with clear bounded contexts; extract services (Notifications, Appointments, Auth) only against documented, measurable triggers — not speculatively.

---

## Product Definition of Done

**Objective:** Prove that BarberSaaS can become the operating system for independent barbershops in mid-sized Colombian cities, starting with sustainable adoption and paid conversion in Huila and expanding regionally.

| Key Result | Baseline | Target | Date |
|------------|---------|--------|------|
| KR1: Active paid barbershop subscriptions | 0 | 50 barbershops | 12 months post-launch |
| KR2: Trial-to-paid conversion rate | N/A (pre-launch) | > 40% | 12 months post-launch |
| KR3: Monthly Recurring Revenue (MRR) | COP $0 | COP $5,000,000 | 12 months post-launch |
| KR4: Monthly churn rate | N/A (pre-launch) | < 5% | 12 months post-launch |
| KR5: Cities with ≥1 active barbershop | 1 (Neiva) | 5+ cities in Colombia | 12 months post-launch |
| KR6: % of appointments booked through the app vs. walk-in | N/A (pre-launch) | > 70% within 30 days of onboarding, per barbershop | Ongoing, from Phase 2 |

---

## Correlations

- Problem framing (the why) → `03-product/problem-framing.md`
- Backlog that implements the vision → `04-requirements/user-stories.md`
- KPIs in operations → `13-operations/README.md`
