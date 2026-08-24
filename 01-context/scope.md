# System Scope

---

## In Scope

What the system **DOES build and maintain**:

### MVP Features

| # | Feature | Description | Responsible service |
|---|---------|-------------|---------------------|
| 1 | Multi-tenant architecture | Tenant isolation by `barbershop_id`, resolved from the JWT and enforced in every service-layer call via `TenantContext` | shared / security |
| 2 | Authentication & roles | Registration, login, JWT sessions (24h expiry, 7-day refresh), password recovery via 6-digit email code, 4 roles (SUPER_ADMIN, ADMIN_BARBERSHOP, BARBER, CLIENT) | auth |
| 3 | Appointment booking | Real-time availability search, booking, cancellation, rescheduling, no-show marking, and a state machine (PENDING → CONFIRMED → IN_PROGRESS → COMPLETED) with anti-double-booking via pessimistic DB locking | appointment |
| 4 | Service catalog | Per-barbershop configuration of services, prices, and duration | barber-shop |
| 5 | Staff & schedule management | Add/edit/deactivate barbers, configurable weekly schedules, schedule exceptions (days off, modified hours) | barber-shop |
| 6 | Loyalty program | Configurable sticker-based loyalty card, automatic reward coupon generation on redemption, 100% discount applied on the client's next booking | loyalty |
| 7 | Financial tracking | Manual income/expense records, revenue auto-recorded from completed appointments, dashboard with revenue/expenses/net profit | finance |
| 8 | Inventory management | Product stock tracking with minimum-stock alerts | barber-shop |
| 9 | Notifications | Push notifications (FCM) for appointment events, persisted in-app history, email for password recovery only | notification |
| 10 | Super Admin dashboard | View/manage all barbershops, subscription plans, account status (activate/suspend/cancel), platform-wide metrics | super-admin |
| 11 | Self-registration & trial | 3-step self-registration wizard for barbershop owners; 60-day free trial (`TRIAL` status) | auth / barber-shop |
| 12 | Subscription plan management | Starter / Profesional / Premium plans with barber-count limits, manual trial-to-active transition by Super Admin | super-admin |
| 13 | Walk-in client tracking | Admins can add walk-in clients directly to a barber's agenda without requiring account registration (in progress) | appointment |
| 14 | Trial expiration automation | Automatic transition/handling of barbershop accounts as their 60-day free trial approaches or reaches expiration (in progress); notification mechanism for Super Admin still an open question | barber-shop / super-admin |

### Included integrations

| External system | Integration type | Purpose |
|----------------|-----------------|---------|
| Firebase Cloud Messaging (FCM) | SDK (Firebase Admin SDK) | Push notifications for appointment events (booked, confirmed, cancelled, completed) |
| Gmail SMTP | SDK (Spring Mail) | Transactional email for password recovery only |

### Environments being built

| Environment | Purpose |
|-------------|---------|
| Local | Development (currently on MySQL) |
| Staging / Pre-production | PostgreSQL migration validation, pre-production testing |
| Production | Hosted on Railway (planned); PostgreSQL, HTTPS/TLS 1.2+ enforced |

> Note: the PRD does not describe a separate CI/dev environment beyond local development; only local and the planned Railway production deployment are explicitly documented.

---

## Out of Scope

What the system **does NOT build** in this version and why:

| # | What is out of scope | Reason | Future version? |
|---|---------------------|--------|----------------|
| 1 | In-app payment processing (Stripe, PSE, Nequi) | Not MVP; billing is manual (Super Admin confirms payment) | Yes — Phase 3 (Q1 2027), automated billing via PayU/Stripe |
| 2 | Client-facing web app (QR-based booking, no app install) | Not MVP; mobile app is the only client interface | Yes — Phase 3 (Q1 2027) |
| 3 | Advanced analytics and reporting exports (PDF, Excel) | Not MVP | Yes — Phase 3 (Q1 2027), advanced reporting and PDF exports |
| 4 | Multi-location barbershop chains under a single account | Not MVP | Post-MVP add-on (no date committed) |
| 5 | Marketplace / discovery platform for clients to find new barbershops | Not MVP | Not currently planned |
| 6 | Integration with Google Calendar / Apple Calendar | Not MVP | Not currently planned |
| 7 | Automated billing and dunning management | Not MVP; MVP billing is manual | Yes — Phase 3 (Q1 2027) |

### What another system / team handles (and why not us)

| Feature | Who builds it | Why not us |
|---------|--------------|-----------|
| Automated payment processing (Stripe/PayU) | Future third-party payment provider integration | Not built in MVP; deferred to Phase 3 |
| Token revocation / rate limiting infrastructure (Redis-based) | Backend team, pre-production hardening | Listed as a "before production" recommendation, not yet implemented |

> The PRD does not mention a separate SSO/IAM team or a BI/Analytics team; these rows are not applicable to BarberSaaS as documented.

---

## Scope assumptions

> These assumptions are taken to be true. If they change, the scope must be renegotiated.

| # | Assumption | Consequence if false |
|---|-----------|---------------------|
| 1 | The current dev database (MySQL) can be migrated to PostgreSQL with only dialect/config changes via the JPA/Hibernate abstraction | Migration would require significantly more rework before Phase 2 launch |
| 2 | Target users (barbershop owners, barbers, clients) have smartphones and basic comfort with apps | Onboarding design and UX would need to change (e.g., heavier in-person support) |
| 3 | FCM push delivery may be unreliable on poor networks, but all notifications are persisted in the DB regardless of delivery status | If not true, users could miss critical appointment events entirely |
| 4 | Gmail SMTP volume stays under ~500 emails/day | Would need to migrate to Resend/SES sooner than planned |
| 5 | Billing can remain manual (Super Admin-confirmed) through the MVP and controlled launch phases | Would require prioritizing automated billing earlier, before Phase 3 |

---

## Constraints

| Type | Description |
|------|-------------|
| **Time** | Phase 1 (Private Beta) is in progress now; Phase 2 (Controlled Launch) targeted for Q4 2026; Phase 3 (Growth) targeted for Q1 2027 |
| **Technology** | Corporate stack: React Native + Expo (mobile), Spring Boot 3.3.4 + Java 21 (backend), PostgreSQL (target DB, currently MySQL in dev), Redis, Firebase (FCM), JWT auth |
| **Regulatory** | Open question: whether Colombian law requires client data to be stored within Colombia (owner: Legal, to be resolved before production) |
| **Team** | Product Owner / Lead Developer: Carlos Leal; product ownership shared across Carlos Leal, Daniel Cerquera, Juan Pablo Borrero, Carolay Arraut |

> Budget is not specified in the PRD.

---

## External dependencies

| Dependency | Team / Provider | Required date | Status |
|-----------|----------------|--------------|--------|
| PostgreSQL migration from MySQL | Engineering | Before Phase 2 (Q4 2026) launch | 🟡 In progress |
| Railway hosting (backend deployment) | DevOps / Railway | Phase 2 (Q4 2026) | 🟡 Planned |
| Production EAS Build (Play Store + App Store) | Expo / App stores | Phase 2 (Q4 2026) | 🟡 Planned |
| Terms & Conditions and Privacy Policy | Legal | Phase 2 (Q4 2026) | 🔴 Pending |
| JWT_SECRET migration to a secrets manager (AWS Secrets Manager / Railway secrets) | DevOps | Before production | 🔴 Pending |
| Token revocation list (Redis) for logout invalidation | Engineering | Before production | 🔴 Pending |
| Legal ruling on in-country data storage requirement | Legal | Before production | 🔴 Pending |
| Automated billing provider (PayU / Stripe) | Product / Engineering | Phase 3 (Q1 2027) | 🔴 Pending |

---

## How to update the scope

The scope can change, but the change has a process:

1. Document the proposed change in this file
2. Evaluate the impact on schedule and effort
3. Obtain approval from the Product Owner and Tech Lead
4. Update the roadmap in `03-product/vision.md`
5. Create or update HUs in `04-requirements/user-stories.md`

---

## Correlations

- Vision and roadmap → `03-product/vision.md`
- Term glossary → `01-context/glossary.md`
- System overview → `01-context/overview.md`
- Scope-related risks → `15-project-control/risks.md`

---

*Source: BarberSaaS PRD v1.0 (August 2026), sections 6 (Product Scope), 9 (Architecture), 12 (Security & Compliance), 14 (Release Roadmap), and 16 (Open Questions).*
