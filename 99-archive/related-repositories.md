# Related repositories — BarberSaaS ecosystem

> Reference index, not an archived decision. Kept in `99-archive` because it does not belong
> to any single lifecycle section (00–15): it just points out where the rest of the project
> lives, for anyone who only has this docs repo cloned.

The BarberSaaS project (Distributed Systems course, Group G2) is split across repositories
with different scopes. This repo (`barber-saas-docs`) is the documentation source of truth;
the other two hold the product code and the individual academic weekly deliverable.

| Repo | Scope | Remote |
|---|---|---|
| `barber-saas-docs` | Documentation source of truth (this repo) — SDD: architecture, domain, contracts, ADRs | https://github.com/code-corhuila/barber-saas-docs |
| `barber-saas` | Product code (backend + mobile) — branches `main` / `qa` / `develop` | https://github.com/code-corhuila/barber-saas |
| `sistemas-distribuidos-2026-b-g2-Juan-Pablo-Borrero` | Individual academic weekly-grading deliverable (per `00-governance/agile-conventions.md`, not governed by this repo's conventions) | https://github.com/JUANDAX233/sistemas-distribuidos-2026-b-g2-Juan-Pablo-Borrero |

## Why this matters

- `barber-saas` is where "to see the backend and mobile app code, switch to the `develop`
  branch" applies — `main` there has no promoted version yet.
- The weekly repo tracks each course week's `hu-status` and session evidence individually per
  team member; it is graded separately from the product backlog in `04-requirements/user-stories.md`.
- Neither repo is a submodule of this one — they are tracked here only as a pointer, so
  changes to either do not require a change here unless the pointer itself goes stale.
