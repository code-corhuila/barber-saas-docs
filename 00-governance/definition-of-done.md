# Definition of Done (DoD)

> A User Story is **DONE** when it meets ALL criteria on this checklist.
> If even one is missing, the story is NOT done — it goes back to In Progress.

## Mandatory checklist

### Code
- [ ] Code implements all acceptance criteria of the user story
- [ ] Code was reviewed and approved according to the review rule in [git-conventions.md](./git-conventions.md#review-rule-norma-94)
- [ ] Code follows project standards (linting and formatting pass in CI)
- [ ] No technical debt introduced without registering it in `15-project-control/technical-backlog.md`

### Tests
- [ ] Unit tests written for new business logic
- [ ] Test coverage does not decrease from the project baseline
- [ ] All tests pass locally and in CI
- [ ] Acceptance criteria verified (manual or automated)

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
