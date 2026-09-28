# 00-governance — Team Rules

> This section defines the agreements the team commits to follow throughout the project.
> Every team member must read all documents in this section before making their first commit.

---

## Documents in this section

| File | Purpose |
|------|---------|
| [branching-policy.md](./branching-policy.md) | **Course rule (teacher):** permanent branches, promotion by re-application, approvals — the team does not edit it |
| [git-conventions.md](./git-conventions.md) | Branch strategy, commit format, PR policy, merge rules |
| [agile-conventions.md](./agile-conventions.md) | Sprint structure, ceremonies, estimation, backlog tool |
| [definition-of-done.md](./definition-of-done.md) | Checklist that every completed user story must satisfy |
| [definition-of-ready.md](./definition-of-ready.md) | Checklist for a user story to enter a sprint |
| [documentation-rules.md](./documentation-rules.md) | How to write, update, and delete documentation |
| [microservices-documentation.md](./microservices-documentation.md) | Required documents per microservice |
| [security-policy.md](./security-policy.md) | How the team handles vulnerabilities and security incidents |
| [security-rules.md](./security-rules.md) | Code-level security rules: secrets, auth, input validation |

---

## How governance applies

Governance rules apply to **the entire project** — all sections, all repositories, all team members.

**Order of precedence** (highest first):

1. **Course norm** — *Norma de Repositorios, Control de Versiones y Evaluación 2026-B* (the
   teacher's document and its annexes A–I) and [`branching-policy.md`](./branching-policy.md).
2. **Team governance** — the documents in this section.
3. **Local conventions** of a section or repository.

A lower level may add stricter rules; it **never relaxes** a higher one (norm 1.2). An ADR can change
a team rule, never the course norm.

> Change a governance rule only through team agreement.
> Document the change and the reason. Announce it before the next sprint.

---

## Governance framework (pillars)

`00-governance` wraps the whole project and applies to every section. The other sections are grouped
in four blocks, in build order (see the diagram in the repository [README](../README.md)):

| Block | Sections |
|---|---|
| 🔵 DISCOVERY | `01-context` → `02-domain` → `03-product` → `04-requirements` |
| 🟢 DESIGN | `05-architecture` → `06-data`, `07-api` |
| 🟣 DETAIL | `09-microservices` → `08-uml` (→ `16-bpmn`, not created yet), `12-ux-ui` |
| 🟠 IMPL & OPS | `10-devops` → `11-quality`, `13-operations` → `14-training`, `15-project-control` |

Three rules follow from the framework — traceability (P1), doc–code coherence (P2) and governance as
the envelope (P3). They are defined in [`documentation-rules.md`](./documentation-rules.md).

---

## Checking compliance before each cut

The teacher grades each cut (weeks 5, 10 and 15) by running the norm's audit commands (numeral 15)
on the **published** repositories — work that exists only on someone's machine does not count (norm
1.4). Before each cut, the team runs the same checks on itself: the self-evaluation list of the norm
(numeral 17) and the "How it is verified" list of the annex for each repository type. The team keeps
a parametrized copy of the norm and a read-only self-check script in its local workspace
(`Normas/_parametros/`, outside this repository).
