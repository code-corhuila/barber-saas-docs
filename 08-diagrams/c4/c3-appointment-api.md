# C4-03 · appointment-api Components

> **Level:** C4 L3 · **Derived from:** `05-architecture/hexagonal-architecture.md` (folder
> structure of `barber-saas-appointment-api`) · **Rule:** norm 5.3.2 (seven layers), 5.3.3

The inside of one domain service. Appointment is drawn because it is the core domain; every
other `-api` has the same shape with its own ports.

```mermaid
flowchart LR
  gw["api-gateway"]

  subgraph app["appointment-app — composition root"]
    cfg["AppointmentApplication ·<br/>AppointmentConfiguration<br/>wires ports to adapters · explicit limits"]
  end

  subgraph adin["appointment-adapters — in/http"]
    filters["AuthFilter · Rs256Verifier<br/>CorrelationFilter"]
    ctrl["AppointmentController<br/>/api/v1/appointments"]
    err["ErrorHandler<br/>{error, message, details, traceId}"]
  end

  subgraph core["appointment-core — no Spring, no JPA, no driver"]
    pin["port/in<br/>AppointmentUseCases"]
    uc["usecase<br/>AppointmentService"]
    dom["domain/model<br/>Appointment · AppointmentStatus · Money"]
    pout["port/out<br/>AppointmentRepository · IdempotencyStore ·<br/>Outbox · BarbershopCatalog · IdGenerator · Clock"]
  end

  subgraph adout["appointment-adapters — out"]
    jdbc["persistence<br/>JdbcAppointmentRepository ·<br/>JdbcIdempotencyStore · JdbcOutbox"]
    http["http<br/>BarbershopCatalogClient<br/>service token · bounded retries"]
  end

  db[("appointment-db")]
  shop["barbershop-api"]

  gw -->|"HTTP + JWT"| filters --> ctrl
  ctrl -->|"calls"| pin
  ctrl -.->|"exceptions"| err
  uc -->|"implements"| pin
  uc --> dom
  uc -->|"uses"| pout
  jdbc -->|"implements"| pout
  http -->|"implements"| pout
  jdbc --> db
  http --> shop
  cfg -.->|"wires"| ctrl & uc & jdbc & http

  classDef coreC fill:#f6d365,stroke:#b8902a,color:#000
  classDef adapter fill:#1168bd,stroke:#0b4884,color:#fff
  classDef ext fill:#999,stroke:#6b6b6b,color:#fff
  class pin,uc,dom,pout coreC
  class filters,ctrl,err,jdbc,http,cfg adapter
  class gw,db,shop ext
```

## Reading it

| Layer (norm 5.3.2) | Component | Depends on |
|---|---|---|
| Domain | `Appointment` aggregate (state machine, INV-APPT-001…005), `Money` in cents | Nothing |
| Port in | `AppointmentUseCases` — book, confirm, start, complete, cancel, noShow, list | Domain |
| Port out | Repository, idempotency store, outbox, barbershop catalog, id generator, clock | Domain |
| Use case | `AppointmentService` orchestrates; the aggregate decides | Ports |
| Adapter in | Controller + filters; reads the tenant from the token, never from the body | Port in |
| Adapter out | JDBC adapters (one transaction for row + idempotency key + outbox event), HTTP client | Port out |
| Composition root | The only place that knows concrete classes | Everything |

Arrows only point inward: `appointment-core` compiles without Spring (norm 5.3.3). The
lifecycle the aggregate enforces is [ST-01](../uml/state-appointment.md).
