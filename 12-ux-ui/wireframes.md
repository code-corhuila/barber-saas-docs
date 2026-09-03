# Wireframes

> Low-fidelity / mockup designs of the main screens, organized by role. This file tracks
> what already has a visual reference and what is still missing, so it is clear at a
> glance where to focus next — instead of leaving the gap silent.

---

## Coverage status

| Area | Status | Reference |
|------|--------|-----------|
| Auth / onboarding (welcome, login, role picker, register, owner wizard) | ✅ Covered | `mockup-auth-onboarding.png` (this folder) |
| Client area (home, barbershop detail, booking, appointments, favorites, loyalty) | ⬜ Pending | — |
| Barber area (agenda, history, stats) | ⬜ Pending | — |
| Admin area (dashboard, team, services, inventory, gallery, promotions, finance) | ⬜ Pending | — |
| Super-admin area (tenants, plans) | ⬜ Pending | — |

---

## Auth / onboarding — covered by `mockup-auth-onboarding.png`

Seven screens, matching the real routes in `barbersaas-mobile/app/(auth)/`:

1. **Welcome** (`welcome.tsx`) — logo, three value-proposition bullets, "Iniciar sesión" /
   "Crear cuenta" buttons.
2. **Login** (`login.tsx`) — email + password (with visibility toggle), "Iniciar sesión"
   button, links to register and password recovery.
3. **Role picker** (`register-choice.tsx`) — "¿Cómo quieres usar BarberSaaS?" with three
   options: client, barbershop owner, barber.
4. **Client sign-up** (`register.tsx`) — simple registration form, "Registrarme" button.
5. **Owner sign-up wizard** (`register-owner/step1.tsx` → `step2.tsx` → `step3.tsx`) —
   3-step form ("Tu cuenta") with a step indicator, "Continuar" button.
6. **Barber info screen** (part of `register-choice.tsx` flow) — explains that barbers
   don't self-register: they must ask the shop's admin to add them from "Team".

See `navigation-map.md` → "Flow 1 — Onboarding / sign-up" for how these connect.

---

## Pending areas

The following areas exist as real code (see the route list in `navigation-map.md`) but do
not have a visual mockup yet. Whoever picks this up should either:
- Add mockup images here (same convention: `mockup-<area>.png` in this folder), or
- Build ASCII/low-fidelity wireframes directly in this file, one section per area.

- `client/` — home (browse barbershops), barbershop detail, booking flow, appointments,
  favorites, loyalty.
- `barber/` — agenda, history, stats.
- `admin/` — dashboard, team management, services, inventory, gallery, promotions, loyalty
  config, finance, invoices.
- `super-admin/` — tenant list, tenant onboarding, subscription plans.

---

## Correlations
- Navigation map → `12-ux-ui/navigation-map.md`
- Design system (colors, components) → `12-ux-ui/design-system.md`
