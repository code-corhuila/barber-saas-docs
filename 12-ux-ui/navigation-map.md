# Navigation Map

> Defines the screen structure of the system, how screens connect to each other, and what routes
> exist. It is the reference when frontend and backend discuss what endpoints exist
> or how to reach a feature.

---

## Frontend route structure

Source: `barbersaas-mobile/app/` (Expo Router). Route groups in parentheses do not appear
in the URL and are not nested inside one another — each one is an independent navigator
that gets mounted depending on the authenticated user's role (see `app/_layout.tsx`). That
is why screens like `/agenda` or `/profile` exist in more than one group without colliding
at runtime: only one group is active per session.

```
/welcome                       → Landing / entry point (route group: auth)
├── /login                     → Login form
├── /register-choice           → "How do you want to use BarberSaaS?" — role picker
│   ├── client                 → /register (simple sign-up form)
│   ├── barbershop owner       → /register-owner/step1 → step2 → step3 (3-step wizard)
│   └── barber                 → info screen: "ask your shop's admin to add you from Team"
│                                 (barbers do not self-register)
├── /forgot-password           → Request password reset
└── /reset-password            → Set new password from reset link

/home                           (role: client)
├── /barbershop/:id             → Barbershop detail
├── /booking/:barbershopId      → Book an appointment
├── /appointments               → My appointments
├── /favorites                  → Favorite barbershops
├── /loyalty                    → Loyalty program
└── /profile                    → Profile

/agenda                         (role: barber)
├── /history                    → Appointment history
├── /stats                      → Personal performance stats
└── /profile                    → Profile

/dashboard                      (role: admin — barbershop owner)
├── /agenda                     → Shop-wide agenda
├── /employees                  → Team management
├── /schedule/:barberProfileId  → Set a barber's working schedule
├── /services                   → Service catalog
├── /inventory                  → Product inventory
├── /gallery                    → Shop photo gallery
├── /promotions                 → Promotions / discounts
├── /loyalty-config             → Configure the loyalty program
├── /loyalty-clients            → Clients enrolled in loyalty
├── /finance                    → Revenue / commissions
├── /invoices                   → Invoices
└── /profile                    → Profile

/dashboard                      (role: super-admin — SaaS operator)
├── /barbershops                → All barbershops (tenants)
│   └── /barbershops/create     → Onboard a new barbershop
├── /plans                      → Subscription plans
└── /profile                    → Profile

/notifications                  → Notification center (any authenticated role)
```

---

## Screen map

| Screen | Route | File | Role | Backend module (`com.barbersaas.*`) |
|--------|-------|------|------|--------------------------------------|
| Welcome | `/welcome` | `(auth)/welcome.tsx` | Public | — |
| Login | `/login` | `(auth)/login.tsx` | Public | `auth` |
| Role picker | `/register-choice` | `(auth)/register-choice.tsx` | Public | — |
| Client sign-up | `/register` | `(auth)/register.tsx` | Public | `auth`, `user` |
| Owner sign-up wizard | `/register-owner/step1..3` | `(auth)/register-owner/step{1,2,3}.tsx` | Public | `auth`, `barbershop` |
| Forgot password | `/forgot-password` | `(auth)/forgot-password.tsx` | Public | `auth` |
| Reset password | `/reset-password` | `(auth)/reset-password.tsx` | Public | `auth` |
| Client home | `/home` | `(client)/home.tsx` | client | `barbershop` |
| Barbershop detail | `/barbershop/:id` | `(client)/barbershop/[id].tsx` | client | `barbershop`, `barberservice` |
| Booking | `/booking/:barbershopId` | `(client)/booking/[barbershopId].tsx` | client | `appointment`, `schedule` |
| My appointments | `/appointments` | `(client)/appointments.tsx` | client | `appointment` |
| Favorites | `/favorites` | `(client)/favorites.tsx` | client | `favorite` |
| Loyalty (client view) | `/loyalty` | `(client)/loyalty.tsx` | client | `loyalty` |
| Barber agenda | `/agenda` | `(barber)/agenda.tsx` | barber | `appointment`, `schedule` |
| Barber history | `/history` | `(barber)/history.tsx` | barber | `appointment` |
| Barber stats | `/stats` | `(barber)/stats.tsx` | barber | `dashboard` |
| Admin dashboard | `/dashboard` | `(admin)/dashboard.tsx` | admin | `dashboard` |
| Shop agenda | `/agenda` | `(admin)/agenda.tsx` | admin | `appointment`, `schedule` |
| Team management | `/employees` | `(admin)/employees.tsx` | admin | `employee` |
| Barber schedule | `/schedule/:barberProfileId` | `(admin)/schedule/[barberProfileId].tsx` | admin | `schedule` |
| Service catalog | `/services` | `(admin)/services.tsx` | admin | `barberservice` |
| Inventory | `/inventory` | `(admin)/inventory.tsx` | admin | `inventory` |
| Gallery | `/gallery` | `(admin)/gallery.tsx` | admin | `gallery` |
| Promotions | `/promotions` | `(admin)/promotions.tsx` | admin | `promotion` |
| Loyalty config | `/loyalty-config` | `(admin)/loyalty-config.tsx` | admin | `loyalty` |
| Loyalty clients | `/loyalty-clients` | `(admin)/loyalty-clients.tsx` | admin | `loyalty` |
| Finance | `/finance` | `(admin)/finance.tsx` | admin | `finance` |
| Invoices | `/invoices` | `(admin)/invoices.tsx` | admin | `finance` |
| Super-admin dashboard | `/dashboard` | `(super-admin)/dashboard.tsx` | super-admin | `dashboard` |
| Barbershops (tenants) | `/barbershops` | `(super-admin)/barbershops.tsx` | super-admin | `barbershop` |
| Onboard barbershop | `/barbershops/create` | `(super-admin)/barbershops/create.tsx` | super-admin | `barbershop` |
| Plans | `/plans` | `(super-admin)/plans.tsx` | super-admin | `plan` |
| Profile (all roles) | `/profile` | `{group}/profile.tsx` | any | `user` |
| Notifications | `/notifications` | `notifications.tsx` | any | `notification` |

---

## Access matrix

| Screen group | client | barber | admin | super-admin |
|--------------|:---:|:---:|:---:|:---:|
| `/welcome`, `/login`, `/register*` | ✅ public | ✅ public | ✅ public | ✅ public |
| `/home`, `/barbershop/:id`, `/booking`, `/appointments`, `/favorites`, `/loyalty` | ✅ | ❌ | ❌ | ❌ |
| `/agenda`, `/history`, `/stats` (barber) | ❌ | ✅ | ❌ | ❌ |
| `/dashboard`, `/employees`, `/services`, `/inventory`, `/gallery`, `/promotions`, `/loyalty-config`, `/loyalty-clients`, `/finance`, `/invoices` | ❌ | ❌ | ✅ | ❌ |
| `/barbershops`, `/plans` | ❌ | ❌ | ❌ | ✅ |
| `/profile` | ✅ own | ✅ own | ✅ own | ✅ own |
| `/notifications` | ✅ | ✅ | ✅ | ✅ |

---

## Main user flows

### Flow 1 — Onboarding / sign-up
```
Welcome (/welcome)
    │
    ▼ "Crear cuenta"
Role picker (/register-choice)
    │
    ├── client ─────────────► Register (/register) ──► Home (/home)
    │
    ├── barbershop owner ───► Owner wizard (/register-owner/step1 → step2 → step3)
    │                          ──► Admin dashboard (/dashboard)
    │
    └── barber ─────────────► Info screen: "ask your shop's admin to add you from Team"
                               (no self-registration for this role)
```
**Visual reference:** `mockup-auth-onboarding.png` in this same folder.

### Flow 2 — Login
```
Welcome (/welcome)
    │
    ▼ "Iniciar sesión"
Login (/login)
    │
    ├── Valid credentials ──► role-specific home (/home | /agenda | /dashboard)
    │
    └── Invalid credentials ► Login with error message
```

### Flow 3 — Client books an appointment
```
Home (/home)
    │
    ▼ pick a barbershop
Barbershop detail (/barbershop/:id)
    │
    ▼ "Reservar"
Booking (/booking/:barbershopId)
    │
    ├── Confirmed ──► Appointments (/appointments)
    └── Slot unavailable ──► Booking with error, pick another slot
```

---

## Navigation rules

| Rule | Description |
|------|-------------|
| Authentication | Every route outside `(auth)` redirects to `/login` if there is no session |
| Authorization | A user can only see the route group matching their role — no cross-role navigation |
| Barber onboarding | Barbers cannot reach a private area until an admin adds them from `/employees` |
| Confirmation | Destructive actions (delete service, remove employee, cancel appointment) show a confirmation dialog before executing |

---

## Correlations
- Design system (visual components, real color tokens) → `12-ux-ui/design-system.md`
- Mockup (auth/onboarding flow) → `12-ux-ui/mockup-auth-onboarding.png`
- Real screen source → `barbersaas-mobile/app/` (route groups `(auth)`, `(client)`, `(barber)`, `(admin)`, `(super-admin)`)
- Backend modules consumed → `com.barbersaas.*` in `barbersaas-backend/src/main/java/com/barbersaas/`
