# Definition of Ready (DoR)

> A user story is **Ready** when the team can start it in the next sprint without having to
> resolve a product, contract or data question halfway through. A story that fails any item
> goes back to refinement (`agile-conventions.md`, "Backlog Refinement").
>
> In BarberSaaS "starting" a story means writing its SPEC → PLAN → HANDOFF
> (`_ecosistema/SPEC-PLAN-PROMPT.md`, gates G1–G2). The DoR is what that SPEC needs as input.

---

## DoR checklist

### 1. The story

- [ ] It has an id `HU-<DOMAIN>-NNN` (e.g. `HU-APPT-001`, `HU-SHOP-001`) and lives in
      `04-requirements/user-stories.md`, written from `04-requirements/_template-hu.md`
- [ ] Format **As [role], I want [action], so that [benefit]**, where the role is one of
      `SUPER_ADMIN`, `ADMIN_BARBERSHOP`, `BARBER`, `CLIENT` (never "a user")
- [ ] It traces to at least one `FR-NNN` of `04-requirements/functional.md`, and the
      `traceability-matrix.md` row exists

### 2. Acceptance criteria

- [ ] At least 2 criteria in **Given / When / Then**, covering the happy path and the main error
- [ ] Every error criterion names the expected status and error code from the closed list
      (`VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`,
      `INVALID_STATUS_TRANSITION`, `BUSINESS_RULE_VIOLATION` — norm 5.3.5)
- [ ] If the story touches tenant data, one criterion covers another barbershop's token
      (expected `404`, `07-api/authentication.md`)
- [ ] No criterion is unmeasurable ("fast", "user-friendly")

### 3. Where it lives

- [ ] The owning domain and repositories are named (e.g. `barber-saas-appointment-{db,api,app}`),
      from the catalog in `05-architecture/overview.md` §4
- [ ] A story that crosses domains says how: call to a published API, event through the outbox,
      or saga in `barber-saas-workflow` — never a query to another domain's database (norm 7.3)
- [ ] Every business rule it relies on has its invariant (`INV-*`) in
      `02-domain/entities-and-rules.md`

### 4. Contract and data (before any code)

- [ ] Each endpoint it adds or changes is already in its contract under
      `07-api/contracts/openapi/`, merged in `main` (or in the same DOCS PR as the story)
- [ ] Each table or column it needs is already in `06-data/models.md` under its domain, with
      ADR-010 types (UUID, `_cents`, `CHECK`)
- [ ] Creating operations declare `Idempotency-Key`; lists are paginated
- [ ] If it forces a technical decision, the ADR is written first (norm 4.2.3)

### 5. Estimation and planning

- [ ] Estimated with the Fibonacci scale of `agile-conventions.md`; 8 or 13 → split first
- [ ] It fits the sprint's capacity together with the rest of the commitment
- [ ] The stories it depends on are Done or planned earlier in the same sprint
- [ ] The issue exists on the board in the **Ready** column, with its **Environment** field empty
      until work starts (norm 16.7)

### 6. Non-functional requirements

- [ ] Security: roles allowed per operation are stated; nothing takes the tenant from the body
- [ ] Observability: which events or log lines prove it works (`X-Correlation-Id` end to end)
- [ ] The test it needs is named (HTTP contract test, cross-tenant test, concurrency test for
      booking) — `04-requirements/traceability-matrix.md` today records zero tests, so every
      new story states its own

---

## Documentation-only stories

A story that only changes this repository (DOCS) needs sections 1, 2 and 5; its "contract" is the
list of files it changes and the tracker finding or FR it closes. It enters through a
`docs/NNN-slug` branch and a Pull Request to `main` (`branching-policy.md`).

---

## Common reasons a story is NOT ready

| Problem | What to do |
|---------|-----------|
| The endpoint is not in any contract | Write or extend the OpenAPI contract first (07), as its own DOCS PR |
| The table is not in `06-data/models.md` | Model it under its domain first (06), following ADR-010 |
| It needs data from another domain | Decide API call, event or saga; record it (OQ in `07-api/open-questions.md` or ADR) |
| Acceptance criteria without error codes | Add status + code from the closed list |
| Estimated 8 or more | Split by operation or by role |
| Blocked by a teacher decision (e.g. ADR-008, ADR-009) | Keep it in Backlog; do not start on an assumption |

---

## DoR vs DoD

| | Definition of Ready (DoR) | Definition of Done (DoD) |
|-|--------------------------|--------------------------|
| **When** | Before writing the SPEC | After the PR is merged and promoted |
| **Who verifies** | Team in refinement; Daniel approves the SPEC (gate G1) | Team in review; `review-gate` rubric on the HANDOFF report (gate G4) |
| **Purpose** | The team can start without guessing | The increment is in the repository, traced and shippable |

---

## Correlations

- Definition of Done → `00-governance/definition-of-done.md`
- Workflow SPEC → PLAN → HANDOFF → review → `_ecosistema/SPEC-PLAN-PROMPT.md` (workspace root)
- Story template and backlog → `04-requirements/_template-hu.md`, `04-requirements/user-stories.md`
- Estimation scale and board → `00-governance/agile-conventions.md`
- Contracts and data → `07-api/`, `06-data/models.md`
