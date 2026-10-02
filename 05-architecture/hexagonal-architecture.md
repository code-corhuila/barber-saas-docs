# Hexagonal Architecture (Ports & Adapters)

> Hexagonal architecture (Alistair Cockburn) keeps the **business domain independent** of the
> technology around it: the database, the web framework and the message transport are
> interchangeable details; the business rules live at the center.
>
> **In BarberSaaS this is a rule, not reference material.** Every service of the ADR-004
> topology — the eight `barber-saas-<domain>-api`, `-worker` and `-workflow` — is built this way
> (course norm 5.3.1–5.3.3, annex C). Eight services are Java 21 / Spring Boot 3.5 with three
> Maven modules; `notifications-api` and `worker` are Python 3.12
> ([ADR-012](decisions/records/ADR-012-two-backend-languages.md)), with the same layers under
> `src/<service>/` (see "Python services" below). The framework's generic
> [`_stacks/java-spring.md`](../_stacks/java-spring.md) shows a single-module layout; where it
> differs, the three-module layout of this document and annex C wins.

> **Prototype vs. target.** The first-cut prototype (`code-corhuila/barber-saas`, ADR-002 —
> superseded) did **not** follow this structure: per module it went `Controller` → `Service`
> (calling JPA repositories directly), with entities shared under `com.barbersaas.domain`. Its
> business rules are ported; its layering is not. Nothing in this document describes the
> prototype.

---

## The problem it solves

```
❌ Prototype layering                         ✓ Hexagonal (target)

  AppointmentController                        HTTP adapter ──▶ [port in] BookAppointment
        ↓                                                             │
  AppointmentService  ← business rules          use case ─────────────┤  (orchestrates)
        ↓               mixed with JPA,                               ▼
  AppointmentRepository (JPA)                   DOMAIN  Appointment, invariants INV-APPT-*
        ↓                                                             ▲
  MySQL                                         [port out] AppointmentRepository, Clock,
                                                           BarbershopCatalog, IdGenerator
To test "no double booking" you need                          ▲
Spring and a database.                          JDBC adapter · HTTP client adapter · outbox
```

---

## The layers (norm 5.3.2)

| Layer | What it holds | May depend on | Maven module |
|---|---|---|---|
| **Domain** | Entities, value objects, invariants, typed domain errors | Nothing of the project | `<domain>-core` |
| **Ports in** | What the service offers: use-case interfaces, commands, queries | Domain | `<domain>-core` |
| **Ports out** | What the service needs: repository, clock, id generator, other domains | Domain | `<domain>-core` |
| **Use cases** | One implementation per business operation | Ports and domain | `<domain>-core` |
| **HTTP adapter in** | RS256 token check, validation, error envelope, correlation, routes | Ports in | `<domain>-adapters` |
| **Adapters out** | JDBC persistence (with pool), outbox, HTTP clients to other domains | Ports out | `<domain>-adapters` |
| **Composition root** | The only place that knows every concrete type and every explicit limit | Everything | `<domain>-app` |

**Dependency rule:** arrows always point to the domain. `<domain>-core` does **not** declare
Spring, JPA or any driver in its `pom.xml`, so a `@Service`, `@Entity` or `@RestController` in
the domain **does not compile** (norm 5.3.3). The rule is enforced by the compiler, not by review.

---

## Folder structure — `barber-saas-appointment-api`

Package root `co.edu.corhuila.barbersaas.appointment` (annex C, Java):

```
barber-saas-appointment-api/
├── appointment-core/                      # no Spring, no JPA, no driver
│   └── src/main/java/…/appointment/
│       ├── domain/model/
│       │   ├── Appointment.java           # aggregate: state machine, INV-APPT-001…005
│       │   ├── AppointmentStatus.java
│       │   ├── Money.java                 # value object, long cents (ADR-010)
│       │   └── DomainException.java       # → 422 INVALID_STATUS_TRANSITION / BUSINESS_RULE_VIOLATION
│       └── application/
│           ├── port/in/
│           │   └── AppointmentUseCases.java    # book, confirm, start, complete, cancel, noShow, list
│           ├── port/out/
│           │   ├── AppointmentRepository.java  # save, findById(tenant, id), overlaps(...)
│           │   ├── IdempotencyStore.java       # key + resource in the same transaction
│           │   ├── Outbox.java                 # AppointmentCompleted, AppointmentCancelled …
│           │   ├── BarbershopCatalog.java      # service price and duration (barbershop-api)
│           │   ├── IdGenerator.java            # UUID
│           │   └── Clock.java
│           └── usecase/
│               └── AppointmentService.java     # implements AppointmentUseCases
├── appointment-adapters/
│   └── src/main/java/…/appointment/adapter/
│       ├── in/http/
│       │   ├── AppointmentController.java      # /api/v1/appointments — DTO ↔ use case
│       │   ├── AuthFilter.java · Rs256Verifier.java   # norm 5.3.7
│       │   ├── CorrelationFilter.java          # X-Correlation-Id, norm 5.3.9
│       │   ├── ErrorHandler.java · ApiError.java      # {error, message, details, traceId}
│       │   └── HealthController.java
│       └── out/
│           ├── persistence/JdbcAppointmentRepository.java · JdbcIdempotencyStore.java · JdbcOutbox.java
│           └── http/BarbershopCatalogClient.java      # service token, bounded retries
├── appointment-app/
│   └── src/main/
│       ├── java/…/appointment/app/AppointmentApplication.java · AppointmentConfiguration.java
│       └── resources/application.yml              # every explicit limit (norm 5.3.10)
├── deploy/ (compose.yml, Dockerfile)
└── pom.xml                                         # parent with the three modules
```

The schema is **not** here: it lives in `barber-saas-appointment-db` (norm 5.2.1).

---

## Ports and adapters in BarberSaaS

### Port in — what the service offers

```java
// appointment-core/.../application/port/in/AppointmentUseCases.java
public interface AppointmentUseCases {
    Appointment book(TenantId tenant, Actor actor, BookCommand cmd, IdempotencyKey key);
    Appointment confirm(TenantId tenant, Actor actor, AppointmentId id);
    Appointment complete(TenantId tenant, Actor actor, AppointmentId id);
    // cancel, start, noShow, list …
}
```

The tenant is an explicit argument, read from the token by the HTTP adapter — never from the
body (`07-api/authentication.md`).

### Port out — what the service needs

```java
// appointment-core/.../application/port/out/AppointmentRepository.java
public interface AppointmentRepository {
    Optional<Appointment> findById(TenantId tenant, AppointmentId id);   // other tenant → empty → 404
    boolean overlaps(BarberId barber, LocalDate date, LocalTime start, LocalTime end);
    void save(Appointment appointment);
}
```

### Use case — orchestrates, does not decide

```java
// appointment-core/.../application/usecase/AppointmentService.java
public Appointment complete(TenantId tenant, Actor actor, AppointmentId id) {
    Appointment appt = repository.findById(tenant, id).orElseThrow(NotFound::new);
    appt.complete(actor, clock.now());          // the aggregate enforces the state machine
    repository.save(appt);                      // same transaction as …
    outbox.add(AppointmentCompleted.of(appt));  // … the event (norm 5.3.11)
    return appt;
}
```

### Adapter in — HTTP to use case

```java
// appointment-adapters/.../adapter/in/http/AppointmentController.java
@PostMapping("/api/v1/appointments/{id}/complete")
AppointmentResponse complete(@PathVariable UUID id, AuthenticatedUser user) {
    return AppointmentResponse.from(                                    // never the entity (5.3.4)
        useCases.complete(user.tenant(), user.actor(), new AppointmentId(id)));
}
```

### Adapter out — JDBC implements the port

`JdbcAppointmentRepository` writes to `appointment.appointment` (`06-data/models.md` §5). The
exclusion constraint `ex_appointment_no_double_booking` is the final guarantee; the adapter
translates its violation into the domain's `BUSINESS_RULE_VIOLATION`.

---

## Python services — `notifications-api` and `worker`

Same layers, same contract, different folders (annex C, annex D):

| Layer | Java (`appointment-api`) | Python (`notifications-api`) |
|---|---|---|
| Domain | `appointment-core/…/domain/model/` | `src/notifications/domain/model/` |
| Ports in / out | `…/application/port/in/`, `…/port/out/` | `src/notifications/application/port/inbound/`, `…/outbound/` |
| Use cases | `…/application/usecase/` | `src/notifications/application/usecase/` |
| HTTP adapter | `appointment-adapters/…/adapter/in/http/` | `src/notifications/adapter/inbound/http/` (FastAPI) |
| Persistence | `…/adapter/out/persistence/` (JDBC) | `src/notifications/adapter/outbound/persistence/` (`pymongo`) |
| Composition root | `appointment-app/` | `apps/api/__main__.py` |

The worker follows annex D: `src/worker/` with a scheduler as inbound adapter and HTTP clients
(`urllib`, standard library) as outbound adapters, started from `apps/worker/__main__.py`.

**What replaces the compiler.** In Java a Spring annotation in `-core` does not compile. In
Python nothing stops it, so each Python repository runs an `import-linter` contract in CI:
`domain` imports nothing from the project, `application` imports only `domain`, and neither
imports FastAPI, `pymongo` or an adapter. A failing contract fails the pull request.

---

## Testing (annex C)

| Level | Tests | Needs |
|---|---|---|
| Core | Invariants (`INV-APPT-*`), transitions, idempotency by key, bounded page | Nothing: fakes of the ports, no Spring, no database |
| HTTP | Every `401` variant, error envelope and correlation, field validation, idempotent retry, page limit | The server with in-memory repositories |
| Integration | Round trip, idempotency key rollback, the double-booking constraint | PostgreSQL with the `-db` schema via `TEST_DATABASE_URL` (skipped if unset) |

```java
// appointment-core test — zero external dependencies
@Test void cannotConfirmACancelledAppointment() {
    var appt = AppointmentFixtures.cancelled();
    assertThrows(InvalidStatusTransition.class, () -> appt.confirm(staff, now));
}
```

TDD guide: `11-quality/tdd-guide.md`.

---

## Review checklist (every `-api`, `-worker`, `-workflow`)

- [ ] `<domain>-core/pom.xml` declares no Spring, JPA or driver dependency
- [ ] Domain and use cases import nothing from `adapter` or `app`
- [ ] Every repository and external call is a `port/out` interface; its implementation is in `adapter/out`
- [ ] Controllers call a `port/in` interface and map to response DTOs; the entity is never serialized (5.3.4)
- [ ] The tenant reaches every use case as an argument from the token; every tenant query filters by it
- [ ] No service reads another domain's database — it calls that domain through a `port/out` HTTP client (norm 7.3)
- [ ] Events are written to the outbox in the same transaction as the change (5.3.11)
- [ ] Every explicit limit is declared in `<domain>-app` (`application.yml`, `HikariConfig`) (5.3.10)
- [ ] One core test per invariant in `02-domain/entities-and-rules.md`, plus the HTTP checks of annex C

---

## Common mistakes

| Mistake | Why it is wrong | Fix |
|---|---|---|
| `@Entity` / `@Service` in `-core` | Couples the domain to Spring and JPA — and does not compile here | Plain Java in the domain; JPA/JDBC mapping in the adapter |
| Business rule in the controller | Changing the endpoint changes the business | Move it into the aggregate |
| Tenant read from the request body | Cross-tenant leak | Tenant only from the validated token |
| Joining another domain's table | Breaks database-per-service | Port out + HTTP client, or an event projection |
| `double` for money | Rounding errors | `long` cents (`Money`) |
| Validating the token only in the gateway | Internal calls enter unauthenticated | `Rs256Verifier` in every service |

---

## References and correlations

- Bounded contexts → `02-domain/domain-map.md`; invariants → `02-domain/entities-and-rules.md`
- Language and module layout → ADR-012 (supersedes ADR-005); data conventions → ADR-010
- Service catalog and topology → `05-architecture/overview.md`
- Complementary patterns (Saga, Outbox) → `05-architecture/pattern-guide.md`
- Course norm 5.3 and annex C → `Normas/C-api-hexagonal.md` (course material)
