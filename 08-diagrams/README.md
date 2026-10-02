# 08 — Diagrams

> **What is this?** The visual view of BarberSaaS: **C4** for the architecture, **UML** for
> behavior (sequences and state machines) and **ER** for the data of each domain. Every diagram
> is drawn from a document of sections 02, 05, 06 or 07 and says which one; when a diagram and
> its source disagree, the source wins and the diagram is fixed in the same Pull Request.
>
> This section was called `08-uml` until 2026-09-30; the framework names it `08-diagrams`.

**Framework block:** 🟣 DETAIL — fed by `09-microservices`, feeds `16-bpmn` (not created yet).

## Contents

| Folder | What it holds | Diagrams |
|---|---|---|
| [`c4/`](c4/) | Architecture at three zoom levels | [C4-01 System context](c4/c1-system-context.md) · [C4-02 Containers](c4/c2-containers.md) · [C4-03 appointment-api components](c4/c3-appointment-api.md) |
| [`uml/`](uml/) | Behavior across services and inside one aggregate | [SEQ-01 Login](uml/seq-login.md) · [SEQ-02 Book an appointment](uml/seq-book-appointment.md) · [SEQ-03 Complete an appointment](uml/seq-complete-appointment.md) · [SEQ-04 Owner onboarding saga](uml/seq-owner-onboarding-saga.md) · [ST-01 Appointment](uml/state-appointment.md) · [ST-02 Barbershop](uml/state-barbershop.md) |
| [`er/`](er/) | One ER diagram per domain schema (one instance per engine, Annex J) | [ERD-01…09](er/erd-domain-databases.md) |

The full registry, with the source of each diagram, is [`diagram-index.md`](diagram-index.md).

## Conventions

- **Tool:** Mermaid inside Markdown (`.md`), so GitHub renders every diagram without a build
  step. No exported images are versioned: the source is the diagram.
- **One file per diagram** (the ER file groups the eight databases so they can be compared).
  Each file starts with a header: type, the documents it is derived from, and the ADRs or norm
  numerals it depicts.
- **File names:** `c1-`, `c2-`, `c3-<service>` for C4 levels; `seq-<flow>`; `state-<entity>`;
  `erd-<scope>`. English, lowercase, hyphens (ADR-001).
- **Colors (C4):** dark blue = person, blue = container or component, green = database,
  grey = external system or neighbor, dashed = proposed and not yet decided.
- **Honesty rule:** what is not decided is drawn dashed or labeled *proposed*, with the ADR or
  open question that holds it. A diagram never shows as decided what the documents leave open.

## When a diagram must change

| If this changes… | …update |
|---|---|
| `05-architecture/overview.md`, ADR-004/005/006/008/009, course norm annexes | C4-01, C4-02 |
| `05-architecture/hexagonal-architecture.md` | C4-03 |
| `06-data/models.md` or a `-db` changelog | ERD-01…09 |
| A contract in `07-api/contracts/openapi/` | The sequence diagrams that call it |
| A state machine in `02-domain/entities-and-rules.md` | ST-01, ST-02 |

## Correlations

- Topology and principles → `05-architecture/overview.md`
- Internal structure of every service → `05-architecture/hexagonal-architecture.md`
- Tables and constraints → `06-data/models.md`
- Contracts → `07-api/contracts/openapi/`
- Service catalog → `09-microservices/service-catalog.md`
- Business processes and sagas → `16-bpmn/` (not created yet)
