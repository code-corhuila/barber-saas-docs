# C4-01 · System Context

> **Level:** C4 L1 · **Derived from:** `05-architecture/overview.md` §2, `07-api/authentication.md`
> (roles) · **Decision:** ADR-004

BarberSaaS as a single box: who uses it and which external systems it depends on.

```mermaid
flowchart TB
  owner(["Barbershop owner<br/>ADMIN_BARBERSHOP"])
  barber(["Barber<br/>BARBER"])
  client(["Client<br/>CLIENT"])
  sadmin(["Platform administrator<br/>SUPER_ADMIN"])

  subgraph system["BarberSaaS"]
    core["Multi-tenant platform for barbershops<br/>booking · schedules · loyalty · notifications ·<br/>finance &amp; inventory · subscription plans"]
  end

  fcm["Firebase Cloud Messaging<br/>(external)"]
  mail["E-mail provider (SMTP)<br/>(external)"]

  owner -->|"Manages catalog, staff, schedules, finance"| core
  barber -->|"Manages own appointments and schedule"| core
  client -->|"Books appointments, collects stickers"| core
  sadmin -->|"Onboards barbershops, manages plans"| core
  core -->|"Push notifications<br/>Firebase Admin SDK"| fcm
  core -->|"Password-reset codes<br/>SMTP"| mail

  classDef person fill:#08427b,stroke:#052e56,color:#fff
  classDef sys fill:#1168bd,stroke:#0b4884,color:#fff
  classDef ext fill:#999,stroke:#6b6b6b,color:#fff
  class owner,barber,client,sadmin person
  class core sys
  class fcm,mail ext
```

| Element | Notes |
|---|---|
| Users | Exactly four roles, one per user (`identity_auth.app_user.role`). Every role uses the mobile app |
| Firebase Cloud Messaging | Called only by `notifications-api`; a delivery failure never blocks the operation that produced the notification |
| E-mail provider | Called only by `identity-auth-api` for the 6-digit password-reset code (15 min, single use) |

Next level: [C4-02 · Containers](c2-containers.md).
