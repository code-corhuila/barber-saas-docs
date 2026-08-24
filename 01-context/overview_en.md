# System Overview

---

## What is BarberSaaS?

BarberSaaS is a cloud-based, multi-tenant SaaS platform for digitizing barbershop operations in Colombia. It lets barbershop owners, barbers, and clients manage appointments, staff, a loyalty program, inventory, and finances — all from a single mobile app.

## Problem it solves

**Before the system:** Barbershops (especially in mid-sized cities like Neiva, Ibagué, Pasto, and Cúcuta) run on WhatsApp groups, paper notebooks, and verbal agreements. Owners have no idea how much each barber brings in or whether the business turned a profit at month's end; there's no loyalty program or inventory control; clients have to call or message to check availability and get no appointment reminders.

**With the system:** BarberSaaS centralizes scheduling with real-time availability and anti-double-booking, gives owners visibility into income/expenses and per-barber performance, automates a loyalty program (stickers and reward coupons), tracks inventory with stock alerts, and sends push notifications to clients and barbers.

## Main users

| Role | Description | What they do in the system |
|------|-------------|--------------------------|
| SUPER_ADMIN | Internal platform team | Creates barbershop accounts, manages subscription plans, activates/suspends/cancels accounts, monitors platform health |
| ADMIN_BARBERSHOP | Barbershop owner (2–4 barbers on average) | Manages employees and schedules, service catalog, finances (income/expenses), loyalty program, inventory |
| BARBER | Barber, barbershop employee | Checks daily agenda, sees upcoming appointments, grants loyalty stickers, registers walk-in clients |
| CLIENT | Barbershop client | Searches for barbershops, books/cancels/reschedules appointments, earns and redeems loyalty rewards, receives reminders |

## Technology stack

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Mobile frontend | React Native + Expo SDK 54 + Expo Router | Single codebase for Android/iOS; file-based routing; fast iteration |
| State management | Zustand (auth) + TanStack Query (server state) | Lightweight, no boilerplate, automatic cache invalidation |
| Backend | Spring Boot 3.3.4 + Java 21 | Production-grade, strong typing, solid JPA/Security ecosystem |
| Authentication | JWT (jjwt 0.12.6) + Spring Security | Stateless, scalable, no server-side session needed |
| Database | PostgreSQL (MySQL in current dev environment; migration planned before production) | ACID compliance, superior JSON/JSONB support, better concurrent performance than MySQL |
| Cache / Pub-Sub | Redis | Session invalidation, future rate limiting |
| Push notifications | Firebase Admin SDK (FCM) | Native push for Android and iOS via a single integration |
| Email | Gmail SMTP (Spring Mail) | Zero cost for MVP volume; migrate to Resend/SES at scale |
| Build & Deploy (mobile) | EAS Build (Expo Application Services) | Cloud-based native builds without a local toolchain |
| Hosting (backend) | Railway (planned) | Zero-config deployment for Spring Boot + PostgreSQL; automatic TLS |

**Architecture pattern:** Modular monolith — a single deployable unit, internally structured by bounded context (auth, barber-shop, appointment, loyalty, finance, notification), with multi-tenant isolation via `barbershop_id` resolved from the JWT on every request (`TenantContext`).

## Current status

- **Phase:** In development — Phase 1: Private Beta
- **Document (PRD) version:** 1.0
- **Last updated:** August 2026
- **Completed so far:**
  - ✅ Core backend (8 phases: auth, appointments, loyalty, notifications, finance, inventory)
  - ✅ Mobile app — all 4 roles functional end-to-end
  - ✅ Self-registration wizard for barbershop owners
  - ✅ FCM push notifications (development build)
  - ✅ Password recovery via email
  - 🔄 In progress: walk-in client tracking, trial expiration automation
- **Next milestone:** Phase 2 — Controlled Launch (Q4 2026): migration to PostgreSQL, deployment on Railway, production builds (Play Store + App Store), Terms & Conditions/Privacy Policy, trial-to-paid conversion flow

## Monetization model

- Monthly SaaS subscription with a **60-day free trial**.
- Plans: **Starter** (COP $39,900/month, up to 2 barbers), **Profesional** (COP $79,900/month, up to 5 barbers), **Premium** (COP $149,900/month, unlimited barbers).
- Manual billing in the MVP: the Super Admin transitions the account from `TRIAL` to `ACTIVE` upon payment confirmation; accounts that don't convert move to `SUSPENDED`.
- Automated billing (Stripe/PayU) planned post-MVP.

## Project contacts

| Role | Name | Contact |
|------|------|---------|
| Product Owner / Lead Developer | Carlos Leal | — |
| Product Owners (team) | Carlos Leal, Daniel Cerquera, Juan Pablo Borrero, Carolay Arraut | — |
| Platform transactional email | — | sasbarberias@gmail.com |
| Expo project | — | carlosleal / barbersaas-mobile |
| Firebase project | — | barbersaas (Spark plan) |

---

*Source: BarberSaaS PRD v1.0 (August 2026). Related documents: `docs/ADR-001-barbersaas.md`, `db/init.sql`, `application.yml`, `app.json`.*
