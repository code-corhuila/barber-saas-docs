# ADR-012 — Two Backend Languages: Java 21 for Eight Services, Python for notifications-api and worker

- **ID:** ADR-012
- **Date:** 2026-10-02
- **Status:** Accepted
- **Supersedes:** ADR-005 (Java 21 for all ten services)
- **Authors:** Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia

---

## Context

ADR-005 wrote all ten services — the eight `barber-saas-<domain>-api`, `barber-saas-worker` and
`barber-saas-workflow` — in Java 21 with Spring Boot 3.5. On 2026-10-01 the teacher issued
**Annex J** of the course norm, which prevails over the norm: the backend must use **at least two
different languages** among Go, Java, Python and C# (J.1.2; J.9 corrects numeral 4.2.2). ADR-005
no longer complies.

Annexes C (api), D (worker) and E (workflow) describe the expected structure in each of the four
languages, and the public contract is the same whatever the language: a consumer must not be able
to tell which language a service is written in (annex C, norm 5.3.12).

**Known constraints:**
- Four-person team; the week-10 and week-15 cuts are close.
- The prototype (`barber-saas`, category A) is Java 21 + Spring Boot, so every service moved out
  of Java loses the code that could be ported.
- Two team members work in Python: Carlos Mauricio Leal Medina and Daniel Felipe Cerquera Idrobo.
- The teacher's repository templates exist only for Java; a Python service is built from the
  structure of annexes C and D.

---

## Decision

> Java 21 stays for eight services; `notifications-api` and `worker` are written in Python.

**We decided:**

| Service | Language | Structure |
|---|---|---|
| `identity-auth-api`, `barbershop-api`, `appointment-api`, `schedule-api`, `loyalty-api`, `finance-inventory-api`, `platform-admin-api` | **Java 21 · Spring Boot 3.5** | Maven, three modules (`-core` without Spring, `-adapters`, `-app`), annex C |
| `workflow` | **Java 21 · Spring Boot 3.5** | Maven, three modules, annex E |
| `notifications-api` | **Python 3.12 · FastAPI** | `src/notifications/{domain,application,adapter}`, `apps/api/__main__.py`, `pyproject.toml`, annex C |
| `worker` | **Python 3.12**, standard library | `src/worker/{domain,application,adapter}`, `apps/worker/__main__.py`, `pyproject.toml`, annex D |

For the two Python services:
- **Dependency rule checked in CI:** Python cannot make a framework import fail to compile, so
  each repository adds an import contract (`import-linter`) that fails if `domain` or
  `application` imports FastAPI, a database driver or an adapter (annex C: "in Go and Python it is
  verified by reviewing the imports").
- **Tests:** `pytest`, with the same three levels as Java: domain without doubles, use cases with
  fake ports, adapters against a real engine.
- **Explicit limits** in the composition root (annex C): `pymongo` connection pool size and
  timeouts in `notifications-api`, per-request timeouts and bounded retries in `worker`
  (`urllib`, annex D), and uvicorn's keep-alive and graceful-shutdown values.
- **Same contract as every Java service:** RS256 JWT validated in the service, the common error
  envelope, `X-Correlation-Id`, `Idempotency-Key`, `{data, meta}` pagination.
- **Owners:** Carlos Mauricio Leal Medina and Daniel Felipe Cerquera Idrobo.

**Justification:** Python meets Annex J with the smallest change: two services that are small,
outside the booking and payment path, and that keep little prototype code.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Python for notifications-api and worker (chosen)** | Two team members know Python; notifications is MongoDB plus push and e-mail, with good libraries (`pymongo`, `firebase-admin`); annex D says the standard library is enough for the worker | Two toolchains and two CI recipes; dependency rule checked by tests instead of the compiler | — (chosen) |
| Python for a core domain (appointment, finance) | Shows the second language where it matters most | Concurrency, sagas and money lose the compile-time guarantee and the ported prototype rules | Risk on the critical path before the cuts |
| Go or C# for the second language | Go: small images; C#: compile-time layering like Java | Nobody on the team works in Go or C# | Team knowledge |
| Keep Java everywhere (ADR-005) | One stack | Contradicts Annex J J.1.2 | Forbidden by Annex J |

---

## Consequences

**Positive:**
- The backend complies with Annex J (two languages) and shows that the contract is language
  independent: the gateway, the front and the other services cannot tell the difference.
- The two Python services are the lightest of the platform, which also lowers memory use in
  development.

**Negative / Trade-offs:**
- Two CI recipes (`mvn -B verify` and `pytest` plus the import contract) and two ways of
  structuring a repository.
- The hexagonal rule in the Python services is enforced by a test, not by the compiler; if the
  import contract is removed or weakened, nothing stops a framework import in the domain.
- The prototype's Java notification code is not ported; `notifications-api` is written again
  from `06-data/models.md` §7 and `notification-service.yaml`.
- No teacher template exists for Python; the skeleton is built from annexes C and D.
- Only two people can review Python changes.

**Impact on the system:**
- Affected repositories: `barber-saas-notifications-api`, `barber-saas-worker`.
- Documents that must be updated: `05-architecture/decisions/README.md`,
  `05-architecture/hexagonal-architecture.md`, `05-architecture/overview.md`, `CLAUDE.md`, and —
  once their open pull requests are merged — `05-architecture/deployment.md` (resource targets)
  and `08-diagrams/c4/c2-containers.md` (divergence D-6).

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A framework or driver import reaches the domain of a Python service | Medium | High | `import-linter` contract in CI; review against annex C |
| The Python services drift from the common contract | Medium | Medium | Same OpenAPI contract and the same HTTP checks as the Java services (annex C) |
| Only two people can maintain the Python services | Medium | Medium | Both owners review each other; README answers how to run and test it |
| Unbounded MongoDB pool or HTTP calls without timeout | Low | Medium | Limits declared in the composition root, as annex C and D require |

---

## References

- Course norm 2026-B, **Annex J** (J.1.2, J.9 → 4.2.2), numerals 4.2.2, 4.2.3, 5.3.3, 5.3.12
- Annex C (Python — FastAPI structure and limits), annex D (Python worker), annex I (CI)
- Related to: ADR-004, ADR-006, ADR-011
- Supersedes: ADR-005
