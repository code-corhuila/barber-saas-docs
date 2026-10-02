# Definition of Done (DoD)

> A User Story is **DONE** when it meets ALL criteria on this checklist.
> If even one is missing, the story is NOT done — it goes back to In Progress.
> Counterpart before work starts: [definition-of-ready.md](./definition-of-ready.md).

## How a story reaches Done in BarberSaaS

Every story goes through the team's gated cycle. **This section is its versioned definition.**
Team members also keep a local working copy of the cycle (`_ecosistema/SPEC-PLAN-PROMPT.md` in
their workspace), but that folder is not tracked in any repository. Where the two differ, this
document is the one that counts.

```
G0 intent → G1 SPEC → G2 PLAN → G3 execution → G4 review → G5 validation → merge
```

| Gate | Input | Output | Who signs |
|---|---|---|---|
| G0 — Intent | A need in plain language | A prioritized intent | Story owner |
| G1 — SPEC | The intent + a story that meets the [DoR](./definition-of-ready.md) | `SPEC-NNN`: problem, expected result, verifiable acceptance criteria, out of scope, risks, traceability | Story owner |
| G2 — PLAN | The approved SPEC | `PLAN-NNN`: atomic tasks, order, points of no return | Story owner |
| G3 — Execution | The HANDOFF (role, repository and branch, minimal context, tasks, restrictions, mandatory verification) run in Claude Code | Branch + commits + an execution report with sections *Executed*, *Evidence (commands + output)*, *Files touched*, *Deviations*, *Out-of-scope findings*, *Acceptance criteria one by one* | — |
| G4 — Review | The execution report | A `review-gate` verdict (rubric below) and, if needed, a correction HANDOFF | Reviewer |
| G5 — Validation | The verdict + the owner's own review | A Pull Request approved under the [review rule](./git-conventions.md#review-rule-norma-94) and merged, referencing its story (`code-corhuila/barber-saas-docs#NN`) | Story owner + approver |

No gate is skipped. The DoD is checked at G3–G5.

**`review-gate` rubric (G4).** Six axes, each *met* (2) / *partial* (1) / *not met* (0):

| Axis | What is checked |
|---|---|
| E1 Contract | Every acceptance criterion of the SPEC, one by one, with real evidence and not a claim |
| E2 Scope | Nothing less, nothing more; out-of-scope findings recorded, not executed |
| E3 Architecture | Module and layer boundaries respected; no inverted dependency; consistent with the accepted ADRs |
| E4 Multi-tenant | Every new query or endpoint filters by tenant; no data crosses barbershops |
| E5 Verifiability | The verification commands exist, were run and their output is in the report |
| E6 Traceability | Commits carry `Spec: SPEC-NNN`; docs and ADRs updated if a decision changed |

Verdict: `ACCEPTED` (12/12, or 11/12 with no zero); `ACCEPTED WITH CORRECTIONS` (no zero in E1,
E3, E4 — a correction HANDOFF is issued); `REJECTED` (any zero in E1, E3 or E4 — redone from the
gate that failed). E3 and E4 admit no zero because their damage is silent: a tenant leak or an
inverted dependency breaks nothing today and everything three weeks later.

"It says verified" is not evidence: a criterion counts only with the command or inspection that
proves it pasted in the report or the PR.

## Mandatory checklist

### Code
- [ ] Code implements all acceptance criteria of the user story, each one checked in the execution report
- [ ] The HANDOFF report passed the `review-gate` rubric (G4) — verdict `ACCEPTED`, or its corrections applied
- [ ] Code was reviewed and approved according to the review rule in [git-conventions.md](./git-conventions.md#review-rule-norma-94)
- [ ] Code follows project standards (linting and formatting pass in CI) — **pending: no repository has a `ci.yml` yet** (see "Criteria not yet enforceable")
- [ ] Out-of-scope findings from the report are recorded as their own backlog items, not fixed silently
- [ ] No technical debt introduced without registering it in `15-project-control/` (or in the debt table of `05-architecture/overview.md` when it is architectural)

### Tests
- [ ] Unit tests written for new business logic — the domain core (`<domain>-core`) is testable without Spring (`05-architecture/hexagonal-architecture.md`)
- [ ] Test coverage does not decrease from the project baseline — **pending: there is no baseline yet**
- [ ] All tests pass locally and in CI — **pending: no CI yet**
- [ ] Acceptance criteria verified (manual or automated), with the evidence in the report

### Integration
- [ ] Changes do not break other services (integration tests pass)
- [ ] If API changes: OpenAPI contract updated in `07-api/contracts/`
- [ ] If data model changes: service `data-model.md` updated
- [ ] If new/modified events: `event-catalog.md` updated

### Deployment
- [ ] Merged into `develop` through its Pull Request, with green CI
- [ ] To reach QA: re-applied to `qa` with `git cherry-pick -x` through a `qa/…` Pull Request, with green CI
- [ ] Smoke test passing in the target environment, through the API gateway

### Traceability (course norm)
- [ ] The Pull Request declares its user story: `code-corhuila/barber-saas-docs#NN` (norm 9.1)
- [ ] Every commit subject matches Conventional Commits (norm 8, check 15.2)
- [ ] The branch prefix matches its target branch (norm 6.3) and the PR has at most 400 changed lines, excluding tests and generated files (norm 9.2)
- [ ] Every commit promoted to `qa` or `main` carries its `(cherry picked from commit <sha>)` line (norm 10.3)
- [ ] The story's **Environment** field on the board (Dev / QA / Main) matches where the change actually is (norm 16.7)
- [ ] The self-check of the touched repository reports no new failures (see [README](./README.md#checking-compliance-before-each-cut))

### Documentation
- [ ] Service `README.md` updated if the public interface changed
- [ ] If a significant technical decision was made: ADR created or updated, with Dominant criterion and Accepted cost (norm 4.2.3)
- [ ] Documents of the affected section are consistent with the code (rule P2 in [documentation-rules.md](./documentation-rules.md))

---

## Documentation stories (this repository)

`barber-saas-docs` has only `main` (category B, [git-conventions.md](./git-conventions.md)). A
documentation change is Done when:

- [ ] It went through a `docs/…` branch and a Pull Request, never a direct commit to `main`
- [ ] The PR is approved by `ariel5253` and merged with rebase and merge
- [ ] Every changed OpenAPI contract passes `npx @redocly/cli lint` and its `$ref`s still resolve
- [ ] The documents it touches agree with the sections they cite (rule P2); a contradiction that
      cannot be resolved with evidence is written in the PR, not guessed
- [ ] At most 400 changed lines (norm 9.2); a bigger change is split into several PRs

The Deployment and code-test criteria above do not apply to documentation stories.

---

## Criteria not yet enforceable (state on 2026-09-30)

The 29 code repositories contain only `README.md` and `.github/CODEOWNERS`
(`05-architecture/overview.md`, AT-007). Until that changes, these criteria **cannot be met and
are never ticked** — a story that needs them is not Done, it waits:

| Criterion | Why it cannot be met today | Unblocked by |
|---|---|---|
| Lint, formatting and tests "pass in CI" | No `ci.yml` in any repository (norm annex I) | The first `-api` scaffold with its pipeline |
| Coverage does not decrease | No test suite, so no baseline | First service with tests; record the baseline in `11-quality/testing-strategy.md` |
| Integration tests between services | No service runs | `barber-saas-infra` compose (`05-architecture/deployment.md` §10) |
| Smoke test through the gateway | No gateway | `barber-saas-api-gateway` scaffold |

Reporting any of them as met before its pipeline exists is a false report, not a shortcut.

---

## Allowed exceptions

The following exceptions must be explicitly agreed to by the Tech Lead:
- E2E tests omitted due to environment limitations (document the risk)
- Documentation deferred for urgent delivery (create a tech-debt ticket)
- Exceptions never cover rules of the course norm (branch regime, promotion trail, grave faults)

---

## What is NOT a Done criterion

- "The code is on my machine" — it must be in the repository
- "It works on my local environment" — it must work in the `qa` environment
- "The PM/PO approved it" — that is the product Definition of Done, not the code's
