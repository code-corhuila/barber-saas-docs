# Documentation Rules

> These rules determine how documentation is written, organized, and maintained in this project.
> Documentation that does not follow these rules may be rejected in code review.

---

## Core principle

> **"Documentation is code. If it's not up to date, it's broken."**

Every HU that modifies system behavior MUST include updating the affected documents.
The DoD requires it.

---

## Language

| Artifact | Language |
|----------|----------|
| Source code (variables, functions, classes) | English |
| Code comments | English |
| Commits | English (Conventional Commits) |
| Branch names | English |
| Markdown documentation | English |
| OpenAPI contracts (descriptions) | English |
| Pull Requests, issues, board items | English |
| Error `message` returned by an API | English — as in every example of `_shared.yaml` |
| Text shown to end users (app screens, push notifications, e-mails) | Spanish (Colombia) — localized at the edge (ADR-001, `05-architecture/overview.md` P6) |
| Internal system logs | English |

> **Rule:** the languages above are fixed by ADR-001 and bind the entire project. Mixing languages
> in the same category is grounds for PR rejection. Conversation inside the team may be in Spanish;
> nothing that lands in a repository or on GitHub is.

---

## File structure

The repository is organized in numbered sections, in the order of the framework blocks (see
[README](./README.md#governance-framework-pillars)):

| Block | Sections |
|---|---|
| Governance (wraps all) | `00-governance` |
| Discovery | `01-context`, `02-domain`, `03-product`, `04-requirements` |
| Design | `05-architecture`, `06-data`, `07-api` |
| Detail | `08-diagrams`, `09-microservices`, `12-ux-ui`; `16-bpmn` is expected and does not exist yet |
| Implementation & operations | `10-devops`, `11-quality`, `13-operations`, `14-training`, `15-project-control` |
| Retired content | `99-archive` — kept for history, never cited as current |

- Each section has a `README.md` that explains its purpose.
- Content documents use `kebab-case.md` (e.g. `domain-map.md`, `data-dictionary.md`).
- Templates are prefixed with `_` to appear first (e.g. `_template-hu.md`, `_template-adr.md`,
  `_template-service.yaml`).
- ADRs are numbered sequentially and never renumbered: `ADR-NNN-short-title.md`, registered in
  `05-architecture/decisions/README.md`.
- Per-domain material uses the domain names of the repositories — `identity-auth`, `barbershop`,
  `appointment`, `schedule`, `loyalty`, `notifications`, `finance-inventory`, `platform-admin` —
  never the prototype's module names.

---

## What to document and what NOT to

### DO document

| What | Where |
|------|-------|
| Non-obvious architectural decisions | `05-architecture/decisions/records/ADR-NNN.md` |
| Business rules and domain invariants | `02-domain/entities-and-rules.md` |
| API contracts for each domain | One file per domain in `07-api/contracts/openapi/` (e.g. `appointment-service.yaml`; identity-auth is `auth-service.yaml`), reusing `_shared.yaml` — index in `07-api/README.md` |
| Open questions a contract or model cannot settle yet | `07-api/open-questions.md` (`OQ-NN`) |
| Data model changes | `06-data/models.md` (tables per `-db`) and `06-data/data-dictionary.md` (meaning of fields) |
| Deployment and environments | `05-architecture/deployment.md`, `10-devops/environments.md` |
| Architectural debt | Debt table of `05-architecture/overview.md` (`AT-NNN`) |
| Operational procedures | `13-operations/` |
| Identified risks | `15-project-control/risks.md` |

### DO NOT document

- What the code already says clearly (do not repeat in comments what can be read in the code)
- Temporary decisions or experiments that will be reverted
- Implementation details of external libraries (those have their own documentation)
- Change history (that's what git log is for)

---

## Framework rules (pillars)

The documentation framework is organized in pillars (see [README](./README.md#governance-framework-pillars)).
Three rules follow from it and apply to every section:

| Rule | Statement | Example of a violation |
|---|---|---|
| **P1 — Upstream traceability** | Every artifact points to its origin in the previous block: a commit or PR → a user story in `04-requirements`; a service → its ADR (`05`), its contract (`07`) and its data model (`06`); a saga → its process flow (`16-bpmn`); a pipeline → its environment (`10`) | a repository without an ADR for its language or database engine |
| **P2 — Doc–code coherence** | What `06-data`, `07-api`, `08-diagrams`, `09-microservices` and `12-ux-ui` say matches the published repositories. A divergence is fixed or recorded as a risk in `15-project-control` | a service catalog that describes services that do not exist |
| **P3 — Governance wraps everything** | Every team convention that the course norm asks to record lives in `00-governance` | a review rule agreed in chat but not written in `git-conventions.md` |

---

## ADR format (course norm 4.2.3)

Every ADR required by the course norm has these sections, with these titles:

| Section | Content |
|---|---|
| Context | The problem and the constraints that force a decision |
| Options | At least two real alternatives, not one option and its caricature |
| Dominant criterion | The factor that decided: team knowledge, required consistency, operating cost, latency… |
| Accepted cost | What the chosen option sacrifices, said plainly |
| Consequences | What changes in the system and what must be watched |

An ADR without a dominant criterion and an accepted cost is a statement, not a decision.

ADRs the norm requires: the language of each service, the database engine and the migration tool of
each domain, the interface framework (norm 4.2.2), where saga state is persisted (norm 5.8.4), and
every repository added beyond the mandatory ones — before it is created (norm 4.2.1).

---

## Owners per section

| Section | Owner (role) | Who today | Review frequency |
|---------|-------|-----------|-----------------|
| `00-governance/` | Tech Lead | Carlos Leal | Start of each sprint |
| `02-domain/` | Tech Lead + PO | Carlos Leal | When the domain changes |
| `04-requirements/` | Product Owner | Carlos Leal (the whole team acts as PO, `01-context/overview.md`) | Each sprint |
| `05-architecture/` | Tech Lead | Carlos Leal; ADRs are signed by the whole team | Each design decision |
| `06-data/`, `07-api/contracts/` | Developer who owns the domain | Not assigned per domain yet — assign on the board with the first story of each domain | Each schema or API change |
| `09-microservices/` | Developer who owns the domain | Same as above | Each release |
| `13-operations/` | DevOps / On-call | Not assigned — no environment runs yet | After each incident |
| `15-project-control/` | Tech Lead | Carlos Leal | Weekly review |

Team: Carlos Leal (Tech Lead / Lead Developer / PO), Daniel Cerquera, Juan Pablo Borrero,
Carolay Arraut (`00-governance/agile-conventions.md`). Any member may change any section through a
PR; the owner is who answers for its coherence (rule P2).

---

## Document format

### Headings
- `# H1` — only one per file; it is the title
- `## H2` — main sections
- `### H3` — subsections
- Do not use H4 or deeper; if you need it, the document has too much hierarchy

### Tables
Use tables for comparisons, registers, and matrices. Do not use tables for simple lists.

### Code
Always use code blocks with the language specified:
````
```typescript
const x = 1;
```
````

### Template instructions
Blocks marked `> [!NOTE] INSTRUCTIONS` indicate the document is an unfilled template.
Remove them when the document is complete.

---

## Update process

1. The developer identifies which documents their change affects
2. Updates the documents together with the code (same PR)
3. The reviewer verifies the documentation is up to date
4. If the PR closes a HU that had API impact → the OpenAPI contract must be updated

---

## Correlations

- Course rule on branches and approvals → `00-governance/branching-policy.md`
- Git conventions → `00-governance/git-conventions.md`
- Per-microservice documentation standard → `00-governance/microservices-documentation.md`
- Definition of Done (docs as part of DoD) → `00-governance/definition-of-done.md`
