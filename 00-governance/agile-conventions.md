# Agile Team Conventions

> Defines how the team works through its development cycles. Agree on and sign off
> with the entire team before the first sprint. Update when the team decides to change something.

---

## Sprint structure

| Field | Value |
|-------|-------|
| Duration | 1 week |
| Sprint start | Monday |
| Sprint end | Sunday |
| Current sprint | Track in `15-project-control/` (sprint number/dates not tracked here to avoid duplicating a moving value — see that section for the live status) |
| Estimated capacity | 3–5 story points/week. Effective team size for hands-on implementation is 1 developer (Daniel) most weeks — the single-developer bottleneck is an explicit constraint documented in `ADR-002-modular-monolith.md` |

---

## Ceremonies

### Sprint Planning
- **When:** First day of the sprint — day/time by team agreement (not fixed here)
- **Duration:** Maximum 1h
- **Who:** Entire team — Carlos Leal (Tech Lead / Lead Developer), Daniel Cerquera, Juan Pablo Borrero, Carolay Arraut
- **Goal:** Select and commit to the sprint's user stories, break down into technical tasks (HANDOFFs for Claude Code, per the gated cycle in [definition-of-done.md](./definition-of-done.md#how-a-story-reaches-done-in-barbersaas))
- **Output artifact:** Sprint Backlog reflected in `04-requirements/user-stories.md`

### Daily Stand-up
- **When:** Async, written — the team is not consistently co-located, so there is no fixed live daily meeting; each member posts an update in the team's agreed channel
- **Duration:** N/A (async)
- **Format:**
  1. What did I do yesterday?
  2. What will I do today?
  3. Is anything blocking me?
- **Rule:** Technical discussions happen in a separate thread, not inline with the daily update

### Sprint Review
- **When:** Last day of the sprint — day/time by team agreement
- **Duration:** Maximum 30 min
- **Who:** Team + Product Owner
- **Goal:** Show what was built during the sprint and collect feedback

### Sprint Retrospective
- **When:** Last day of the sprint, immediately after the review
- **Duration:** Maximum 20 min
- **Format:** What went well / What to improve / Action commitments
- **Rule:** Each retro produces at least 1 improvement action with an owner and due date

### Backlog Refinement
- **When:** Ad hoc mid-sprint, whenever a HANDOFF returns findings out of scope (see `review-gate` skill) or a new HU is drafted via `spec-forge`
- **Duration:** Maximum 1h
- **Goal:** Detail and estimate user stories for the next sprint
- **Exit criterion:** The user story meets the Definition of Ready

---

## Estimation

### Scale
| Points | Meaning |
|--------|---------|
| 1 | Trivial — done in hours |
| 2 | Small — done in one day |
| 3 | Medium — takes 2–3 days |
| 5 | Large — takes almost a full sprint |
| 8 | Very large — should be split |
| 13 | Epic — MUST be split before the sprint |

**Technique:** Planning Poker, informal — async agreement during Sprint Planning (the team is not always co-located, so estimates are proposed and confirmed in writing rather than with physical cards)
**Tool:** GitHub Issues, once the CODE repo is initialized (see `repo-hygiene` skill / SPEC-001); until then, estimates are tracked directly in `04-requirements/user-stories.md`. This is the product's own backlog — independent of the `sistemas-distribuidos-2026-b-g2-daniel-cerquera` repo, which is the separate academic weekly-grading deliverable and is not governed by this document.

### Estimation rule
- If there is disagreement of 2+ levels (e.g., someone says 3 and another says 8), discuss before voting again.
- If a story is estimated at 8 or 13, it must be split into smaller sub-tasks.

---

## Backlog tool

**Tool:** GitHub Projects, attached to the `barber-saas-docs` repo and to the CODE repo once it is initialized (backend + mobile — decision pending in SPEC-001: monorepo vs. two repos)
**Board URL:** Pending — no board has been created yet; create it as part of SPEC-001 (CODE repo initialization) and record the URL here

### Board columns
| Column | Meaning | WIP limit |
|--------|---------|-----------|
| Backlog | Pending refinement | None |
| Ready | Ready to enter the sprint (meets DoR) | None |
| In Progress | Someone is actively working on it | **1** |
| In Review | In Pull Request / code review | **2** |
| Done | Meets DoD and is closed | None |

**Why these limits:** effective team size for hands-on implementation is 1 developer
(Daniel) most weeks — the same single-developer bottleneck already documented in
`ADR-002-modular-monolith.md` and in "Estimated capacity" above. A WIP limit of 1 on
"In Progress" makes that bottleneck visible on the board instead of letting several stories
sit half-started; "In Review" allows up to 2 so a finished item can wait for review/CI while
the next one starts, without turning "In Review" into an uncontrolled backlog of its own.
Approved by Daniel, 2026-09-27.

---

## Team velocity

**Retroactive estimation methodology (SPEC-008, 2026-09-27):** the 17 sprint-ready HUs in
`04-requirements/user-stories.md` that were already `✅ Done` before any formal
sprint-planning cycle existed were sized after the fact using Fibonacci (1/2/3/5), based on:
1 point = 1 Gherkin scenario touching 1 module; 2 points = 2 scenarios, or 1 module with
non-trivial business logic; 3 points = 2–3 scenarios touching 2 modules; 5 points = more
than 3 scenarios, or 3+ modules touched (e.g. cross-cutting stories). These are **not**
planning-poker consensus — they estimate already-built work to make historical throughput
visible, and are labeled `(retroactive)` everywhere they appear. See
`04-requirements/user-stories.md` for the per-HU breakdown.

| Sprint | Story points completed | Notes |
|--------|----------------------|-------|
| Sprint 1 | Not attributable | No commit-level evidence ties specific HUs to a specific sprint number before this backlog existed — see aggregate row |
| Sprint 2 | Not attributable | Same as above |
| Sprint 3 | Not attributable | Same as above |
| **Sprints previos (agregado)** | **48 (retroactive)** | Sum of all 17 `✅ Done` sprint-ready HUs as of 2026-09-27 — see per-HU sizes in `04-requirements/user-stories.md` |
| Average | 48 ÷ number of real sprints elapsed — **not computed**, because the number of sprints those 48 points were spread across is unknown (see `15-project-control/sprint-status.md`) | Do not average 48 points into a single unverified sprint count; wait for real sprint boundaries going forward |

---

## Related documents

- Definition of Ready → `00-governance/definition-of-ready.md`
- Definition of Done → `00-governance/definition-of-done.md`
- Risk management → `15-project-control/risks.md`
- Technical debt backlog → `15-project-control/tech-backlog.md`
