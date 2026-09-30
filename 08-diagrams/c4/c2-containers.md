# C4-02 · Containers

> **Level:** C4 L2 · **Derived from:** `05-architecture/overview.md` §3–4,
> `05-architecture/deployment.md` · **Decisions:** ADR-004 (topology), ADR-006 (engine per
> domain), ADR-009 (saga store, proposed)

The deployable units of the 29 repositories: one front, one gateway, eight domain services with
eight databases, the workflow and the worker.

```mermaid
flowchart TB
  user(["Users<br/>4 roles"])
  front["barber-saas-front<br/>mobile host + domain -app remotes"]

  subgraph platform["Network: platform (internal)"]
    gw["barber-saas-api-gateway<br/>NGINX · only published port :8000"]

    subgraph domains["Domain services — Java 21 / Spring Boot, :8080 each"]
      auth["identity-auth-api"]
      shop["barbershop-api"]
      appt["appointment-api"]
      sched["schedule-api"]
      loy["loyalty-api"]
      notif["notifications-api"]
      fin["finance-inventory-api"]
      padm["platform-admin-api"]
    end

    wf["barber-saas-workflow<br/>cross-domain sagas"]
    wk["barber-saas-worker<br/>scheduled jobs · outbox relay"]

    authdb[("identity-auth-db<br/>PostgreSQL")]
    shopdb[("barbershop-db<br/>PostgreSQL")]
    apptdb[("appointment-db<br/>PostgreSQL")]
    scheddb[("schedule-db<br/>PostgreSQL")]
    loydb[("loyalty-db<br/>PostgreSQL")]
    notifdb[("notifications-db<br/>MongoDB rs0")]
    findb[("finance-inventory-db<br/>PostgreSQL")]
    padmdb[("platform-admin-db<br/>PostgreSQL")]
    wfdb[("saga state store<br/>PostgreSQL — proposed")]
  end

  fcm["Firebase Cloud Messaging"]
  mail["E-mail provider"]

  user --> front
  front -->|"HTTPS /api/v1/* · JWT RS256"| gw
  gw --> auth & shop & appt & sched & loy & notif & fin & padm
  gw -->|"/api/v1/sagas"| wf

  auth --> authdb
  shop --> shopdb
  appt --> apptdb
  sched --> scheddb
  loy --> loydb
  notif --> notifdb
  fin --> findb
  padm --> padmdb
  wf -.-> wfdb

  wf -->|"service token"| auth & shop & appt & loy & notif & padm
  wk -->|"service token"| notif & padm & loy
  appt -->|"service token<br/>price, duration, availability"| shop & sched
  notif --> fcm
  auth --> mail

  classDef svc fill:#1168bd,stroke:#0b4884,color:#fff
  classDef db fill:#2f6f4f,stroke:#1f4a35,color:#fff
  classDef ext fill:#999,stroke:#6b6b6b,color:#fff
  classDef pending fill:#fff,stroke:#999,color:#555,stroke-dasharray:4 3
  class front,gw,auth,shop,appt,sched,loy,notif,fin,padm,wf,wk svc
  class authdb,shopdb,apptdb,scheddb,loydb,notifdb,findb,padmdb db
  class fcm,mail ext
  class wfdb pending
```

## Rules the diagram encodes

| Rule | Where it shows | Source |
|---|---|---|
| One database per domain; nobody reads another domain's database | Each `-api` has exactly one arrow to a database | Norm 7.1–7.3, ADR-004 |
| Single entry point | Only the gateway is reached from outside the `platform` network | Norm 5.6.1, annex F |
| Internal calls carry a service token, never the user's token | Arrows from `workflow`, `worker` and `appointment-api` | `07-api/authentication.md`, norm 5.8.2 |
| No distributed transaction | Cross-domain effects go through outbox events or sagas | Norm 7.5, 5.3.11 |

## Not drawn

- **Event transport** between the outbox tables and their consumers is undecided (AT-004 in
  `05-architecture/overview.md`); the flows in `uml/seq-complete-appointment.md` show it as a
  relay.
- **Saga state store** is dashed because ADR-009 is still *Proposed*.
- Observability (OpenTelemetry collector, Prometheus, Grafana) lives in `barber-saas-infra`;
  see `05-architecture/deployment.md`.

Next level: [C4-03 · appointment-api components](c3-appointment-api.md).
