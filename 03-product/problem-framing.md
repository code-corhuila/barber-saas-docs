# Problem Framing — Problem Definition

> **Why this document exists:** Before designing solutions, the team must be
> aligned on the problem it solves. This document captures that alignment.
> A well-defined problem is already halfway to a solution.

---

## 1. The problem in one sentence

**Independent barbershop owners in mid-sized Colombian cities** who **run shops with 2–4 barbers using WhatsApp, paper notebooks, and verbal agreements** struggle with **unpredictable scheduling, no visibility into per-barber or overall business performance, and no formal way to retain clients** because **no digital tool exists that is priced for and adapted to the Colombian independent-barbershop workflow (Spanish UX, COP pricing, walk-in tracking, low-connectivity resilience)**, resulting in **unpredictable client queues, income/expenses tracked manually with no consolidated view, and client retention that depends entirely on personal relationships rather than a formal loyalty program**.

---

## 2. Affected users

| Segment | Description | Estimated size | Priority |
|---------|-------------|---------------|---------|
| Barbershop owners (ADMIN_BARBERSHOP) | Run a shop with 2–4 barbers; need visibility into revenue, staff performance, and client retention, but have no technical background | ~15,000 barbershops in TAM (100k–500k population cities); ~2,000 in SAM (Huila) | High |
| Barbers (BARBER) | Employees who need clarity on their schedule and their own performance, and currently rely on informal coordination with the owner | Proportional to owner segment (avg. 2–4 per shop) | Medium |
| Clients (CLIENT) | Regular barbershop clients who book via WhatsApp or in person, forget appointments, and have no formal loyalty tracking | End consumers across the TAM/SAM barbershops | Medium |
| Walk-in clients | Clients who arrive without booking; entirely invisible to any existing tracking tool | Undetermined — currently untracked by definition | High (product-differentiating gap) |

### Jobs-to-be-done (JTBD)

**When** I own or work in a barbershop and need to manage appointments, staff, and client retention day to day,
**I want** a single tool that shows me my schedule, my numbers, and my clients' loyalty status without manual bookkeeping or relying on WhatsApp,
**so that** I can reduce no-shows and idle time, hold my team accountable, and keep clients coming back — without needing technical skills or Western-priced software.

---

## 3. Evidence of the problem

| Evidence type | Source | Date | Key finding |
|--------------|--------|------|------------|
| Market data | DANE, 2023 | 2023 | Colombia has approximately 80,000+ registered barbershops and hair salons; the mid-sized city segment (100k–500k population) is largely unserved by digital tools |
| Market sizing | Internal TAM/SAM/SOM analysis (PRD §5.1) | — | TAM ~15,000 barbershops (100k–500k population cities); SAM ~2,000 in Huila department; average monthly spend on operational tools in this segment is < COP $50,000 |
| Competitive benchmarking | Booksy, Fresha, Treatwell, WhatsApp/notebooks, Excel (PRD §5.2) | — | Existing global tools are English-first, USD-priced, and not adapted to Colombian workflows (no walk-in tracking, no offline resilience); the de-facto alternative is manual, error-prone tooling (WhatsApp, notebooks, spreadsheets) |
| Persona-level qualitative signals | User personas documented in PRD §4 (owner, barber, client) | — | Owner: *"I don't know how much each barber brings in. I just trust them."* Barber: *"Clients show up whenever, and I never know if it's going to be a busy day or empty."* Client: *"I have to send a message and wait for a reply to know if there's even availability."* |

> **Note:** the personas' quotes reflect the team's documented understanding of the target user rather than transcribed interviews, and no formal user-interview log or support-ticket dataset has been captured yet. Structured interview and support-data validation is a pending activity — see Section 7 (risks).

---

## 4. Current user solution (and its problems)

| Current solution | Limitations | Cost/Friction |
|-----------------|------------|--------------|
| WhatsApp + notebooks | No scheduling logic, no financial tracking, no loyalty program, completely manual, human error-prone | Unpredictable client queues; retention depends entirely on personal relationships |
| Custom Excel sheets | No real-time capability, no mobile access, no automation, no reminders | Manual, unconsolidated income/expense tracking; no visibility into which services generate the most revenue |
| Booksy / Fresha (global SaaS) | English-first UX, USD pricing, no Colombian Spanish localization, requires stable internet, no walk-in tracking | Priced out of reach for small independent shops; poor fit for local workflows |
| Verbal agreements (no tool) | No data, no accountability, no client retention tools | Owners cannot track individual barber performance; walk-in services remain invisible to business metrics |

---

## 5. Solution hypothesis

**We believe that** a mobile, Spanish-language, multi-tenant SaaS platform covering appointment booking (with guaranteed anti-double-booking), staff scheduling, a loyalty program with automatic reward coupons, and consolidated financial/inventory tracking
**for** independent barbershop owners, their barbers, and their clients in mid-sized Colombian cities,
**will achieve** a shift from manual/WhatsApp-based operations to digital, accountable, and retention-driving operations, at a price and onboarding effort that fits a small independent shop.
**We will know we succeeded when** more than 70% of appointments are booked through the app (vs. walk-in) within 30 days of onboarding, at least 60% of barbershops record a loyalty redemption monthly, and trial-to-paid conversion exceeds 40%.

---

## 6. Success metrics (North Star)

| Metric | Current baseline | 6-month target | How to measure it |
|--------|----------------|---------------|-------------------|
| Active paid barbershop subscriptions | 0 (pre-launch) | Progress toward 50 barbershops (12-month target) | Super Admin platform dashboard (ACTIVE/TRIAL/SUSPENDED counts) |
| Trial-to-paid conversion rate | N/A (pre-launch) | Progress toward > 40% (12-month target) | `trial_ends_at` tracking + Super Admin activation records |
| % of appointments booked through the app vs. walk-in | N/A (pre-launch) | > 70% within 30 days of onboarding, per barbershop | Appointment vs. WalkIn record ratio |
| Loyalty engagement | N/A (pre-launch) | > 60% of barbershops with ≥1 loyalty redemption/month | RewardCoupon generation events |

**North Star Metric:** % of appointments booked through the app (vs. informal walk-in/WhatsApp booking) per barbershop — it directly captures whether a shop has genuinely replaced its manual process with BarberSaaS.

---

## 7. Hypothesis risks

| Risk | Probability | Impact | Experiment to validate |
|------|------------|--------|----------------------|
| Low adoption due to unfamiliarity with digital tools among shop owners | High | High | < 5-minute onboarding flow; in-person setup support for the first barbershops in Neiva |
| The problem is not as costly to owners as assumed (they tolerate manual tracking) | Medium | High | Monitor early-cohort engagement (Leading Indicators, PRD §3.3); structured interviews and support-ticket analysis, currently pending (see Section 3) |
| Trial abuse (re-registering to reset the 60-day trial) | Medium | Medium | Email uniqueness constraint; Super Admin review of suspicious registration patterns |
| Double-booking under high concurrency undermines trust in the core promise | Low | High | Pessimistic `WRITE` DB lock — tested and guaranteed at the database level |
| Poor connectivity in target cities blocks reminders/notifications, reducing perceived reliability | Medium | Medium | All notifications persisted to DB before FCM delivery attempt; in-app notification history always available |

---

## 8. Out of scope (we do not solve)

- In-app/automated payment processing (Stripe, PSE, Nequi) in the MVP — billing is manual (bank transfer/Nequi, confirmed by Super Admin) until Phase 3.
- A client-facing web app (QR-based booking without installing the app) — planned for Phase 3, not part of the current problem scope.
- Advanced analytics and PDF/Excel report exports — planned as a Phase 3/4 add-on, not required to validate the core problem.
- Multi-location chain management — out of scope until independent single-location adoption is validated (Phase 4).
- Client discovery marketplace / map-based barbershop discovery — a growth-stage feature (Phase 4), not part of solving the initial operational problem.
- Google/Apple Calendar integration — Phase 4, not required for the core scheduling problem.

---

## Correlations

- Product vision → `03-product/vision.md`
- HUs that implement this solution → `04-requirements/user-stories.md`
- Detailed KPIs → `13-operations/README.md`
