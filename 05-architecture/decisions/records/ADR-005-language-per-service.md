# ADR-005 — Language per Service: Java 21 and Spring Boot 3.5 for All Ten Services

- **ID:** ADR-005
- **Date:** 2026-09-28
- **Status:** Accepted
- **Authors:** Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia

---

## Context

The course norm (4.2.2) requires recording, in an ADR, the language of each service: Go, Java,
Python or C#. ADR-004 adopted ten services: eight `barber-saas-<domain>-api`, `barber-saas-worker`
and `barber-saas-workflow`. Until this is decided, it is not known which structure of annexes C, D
and E applies, and no CI pipeline can be written.

**Known constraints:** four-person team; seven weeks left in the course; a prototype
(`barber-saas`, category A) already written in Java 21 + Spring Boot 3.3; the teacher checks every
`-api` over HTTP without looking at its language (norm 5.3.12).

---

## Decision

**We decided:** every one of the ten services is written in **Java 21 with Spring Boot 3.5**, as a
Maven build with three modules (`<svc>-core`, `<svc>-adapters`, `<svc>-app`) following annexes C
(api), D (worker) and E (workflow).

---

## Evaluated alternatives (options)

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| **Java 21 + Spring Boot 3.5 everywhere (chosen)** | known by the whole team; domain rules and entities reusable from the prototype; one CI recipe (`mvn -B verify`) | heavier images and slower start-up; more files per repository | — (chosen) |
| Java for the `-api`, Go or Python for worker and workflow | smaller worker/workflow; shows language independence | two toolchains, two CI recipes; dependency rule checked by hand in Go/Python (5.3.3) | capacity: a second stack for two small services |
| A different language per domain | broad learning | four structures and pipelines to maintain with four people | risk before the week-10 and week-15 cuts |

---

## Dominant criterion

**Team knowledge and reuse of the prototype**, plus a structural guarantee: with the core in its own
Maven module that does not declare Spring, a framework annotation in the domain **does not compile**
(norm 5.3.3). The hexagonal rule is enforced by the compiler instead of by review.

## Accepted cost

Ten JVM services use more memory and start more slowly than Go equivalents; the repositories are
more verbose (three modules and a parent POM each); the project does not show language diversity;
the prototype code must be moved from Spring Boot 3.3 to 3.5.

---

## Consequences

- CI (annex I): `mvn -B verify` with Java 21 in every service repository.
- The composition root declares all explicit limits (norm 5.3.10) in `application.yml` and `HikariConfig`.
- JWT validation with RS256 and a closed list of algorithms in each service (norm 5.3.7).
- To watch: memory usage of the full platform in development (ten JVMs plus eight databases).

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| The local machine cannot run the whole platform | Medium | Medium | limit JVM heap per container; bring up only the domains under work |
| Porting from the prototype drags framework code into the core | Medium | High | the core module has no Spring dependency; review against annex C |

## References

- Course norm 4.2.2, 4.2.3, 5.3.3, annexes C, D, E, I
- Related to: ADR-004 (full microservice decomposition)
