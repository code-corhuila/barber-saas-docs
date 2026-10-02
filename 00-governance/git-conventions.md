# Git Conventions

> **Read this document before making your first commit on the project.**
>
> This file records how **our team** works. The course rule is
> [`branching-policy.md`](./branching-policy.md) and the course standard is the
> *Norma de Repositorios 2026-B*. Where this file disagrees with either of them, they win
> (norma 1.2). The team may add stricter rules, never looser ones.

## Scope — repository categories

| Category | Repositories | Branch regime |
|---|---|---|
| **A — Prototype** | `code-corhuila/barber-saas` | No restrictions (norma 3.1). Not evaluated after the first cut (norma 3.2). Team exception, decided 2026-09-14: work happens directly on `develop`, no new branch per task. **This exception applies to this repository only.** |
| **B — Documentation** | `code-corhuila/barber-saas-docs` | One permanent branch: `main`. Every change goes through a child branch `docs/NNN-slug` → Pull Request → 1 approval from `ariel5253` → rebase and merge. **No direct commit or push to `main`, ever.** |
| **C — Code** | the 29 `code-corhuila/barber-saas-*` repositories | Three permanent branches, `develop`, `qa` and `main`, as described below. |

Branch strategy, promotion and review rules apply to category C. Branch naming, commit format and
the Pull Request policy apply to categories B and C.

---

## Branch strategy (category C)

```
develop  <── PR ──  feat/…  fix/…  chore/…
qa       <── PR ──  qa/…
main     <── PR ──  release/x.y.z  hotfix/…
```

**Rules:**
- No permanent branch accepts a direct commit. The organization enforces it; it is not a suggestion.
- Each parent branch is fed **only by its own children**: to enter `qa`, branch off `qa`; to enter
  `main`, branch off `main`.
- There is **no merge between permanent branches**: `merge develop → qa` and `merge qa → main` do
  not exist in this model.
- One branch = one task = one user story. A child branch lives at most **five business days**.
- Deleting a child branch after its merge is recommended.
- The permanent branches are named `develop`, `qa` and `main` in every repository. We do not use `dev`.

---

## Promotion and releases (category C)

**Promotion happens by re-application, never by merge** (see `branching-policy.md`):

```bash
git switch qa && git pull
git switch -c qa/hu-appt-003-walk-in-appointments
git cherry-pick -x <sha-of-the-commit-in-develop>
git push -u origin qa/hu-appt-003-walk-in-appointments
# open PR → qa
```

- `-x` is mandatory. The line `(cherry picked from commit <sha>)` is the only link between the two
  versions of a change. A commit in `qa` or `main` without it does not count as progress.
- The cited `<sha>` must exist in the source branch. A trail that points to a non-existent commit
  is falsified evidence.
- **Releases** are cut from `main`, start empty, and are filled with one commit per user story
  already validated in `qa`. The release Pull Request lists each story (id and title) with its
  `cherry picked from` trail, the declared scope (what is in and what is out), deployment
  instructions (migrations, new variables) and a rollback plan.
- **Cadence:** one release per course cut, in weeks 5, 10 and 15.
- After the release is merged, `main` is tagged with an annotated SemVer tag (see *Tags and versioning*).
- `hotfix/` is the only other child of `main`. Every hotfix is re-applied afterwards to `qa` and
  `develop`.

---

## Branch naming

Format: `<prefix>/<description-in-kebab-case>`, using lowercase letters, digits and hyphens only.

| Target branch | Allowed prefixes | Example |
|---|---|---|
| `develop` | `feat/`, `fix/`, `chore/` | `feat/015-appointment-hexagonal-skeleton` |
| `qa` | `qa/` | `qa/hu-appt-003-walk-in-appointments` |
| `main` (code) | `release/<major>.<minor>.<patch>`, `hotfix/` | `release/1.0.0`, `hotfix/null-token-expiration` |
| `main` (DOCS) | `docs/` | `docs/013-align-git-conventions` |

- When the work comes from a SPEC, its number goes in the slug (`docs/NNN-slug`, `feat/NNN-slug`).
- **No other prefix is allowed** (norma 6.3.3). This includes `spec/`, which the team retired on
  2026-09-28.

---

## Review rule (norma 9.4)

| Branch | Required before merging |
|---|---|
| `develop` | 1 approval from a teammate who is **not the author** · green CI · the PR references its user story |
| `qa` | 1 approval from a teammate who is neither the author nor the person who approved the change in `develop` · green CI · every commit carries its `(cherry picked from commit …)` trail · the PR comes from a `qa/…` branch |
| `main` | **1 approval from `ariel5253`** · CODEOWNERS · all conversations resolved · stale approvals dismissed (enforced by the organization) |
| `main` (DOCS) | **1 approval from `ariel5253`** |

- A reviewer answers within **24 business hours**.
- `.github/CODEOWNERS` is part of the protection rules: it is never modified or removed.

---

## Pull Request policy

- **User story:** every PR declares the story it serves: `code-corhuila/barber-saas-docs#NN`.
- **Target:** the branch that matches its prefix (see *Branch naming*). Any other target is a
  process error.
- **Size:** at most 400 changed lines, excluding tests and generated files. If larger, split it.
- **Template:** use `.github/pull_request_template.md`.
- **Green CI:** a PR with red CI is not merged.
- **Promotion trail:** PRs into `qa` or `main` list the re-applied commits with their
  `cherry picked from` lines.
- **Automatic review:** every finding of the automatic review gets an answer in the PR, either
  "applied (how)" or "not applied (technical reason)".
- **History:** never rewrite the history of a published branch (`push --force`, rebasing a shared
  branch).

---

## Merge policy

| Into | From | Method | Why |
|---|---|---|---|
| `develop` | `feat/` `fix/` `chore/` | **Rebase and merge** | every small commit of the story stays in the history (norm 8); each one is re-applied later with `-x` |
| `qa` | `qa/…` | **Rebase and merge** | keeps each re-applied commit with its `-x` trail and creates no merge commit |
| `main` (code) | `release/…`, `hotfix/…` | **Rebase and merge** | one commit per story in `main`, each with its trail |
| `main` (DOCS) | `docs/…` | **Rebase and merge** | every small commit of the documentation change stays in the history |

- **One method everywhere: rebase and merge.** The flow is the same in every repository — child
  branch → pull request → merge — and every small commit lands on the target branch, linear and
  without merge commits. Squash is not used: it collapses the steps the course evaluates.
- *Rebase and merge* on GitHub applies the PR commits on top of the target. It does not rewrite any
  published branch.
- **Never** merge one permanent branch into another.

---

## Commit format (Conventional Commits)

```
[type]([scope]): [lowercase description, imperative mood, no trailing period]

[optional body — explain WHY, not what]

[optional footer — user story reference, e.g. Refs code-corhuila/barber-saas-docs#NN; Spec: SPEC-NNN]
```

The subject must match:
`^(feat|fix|docs|style|refactor|test|chore|perf)(\([a-z0-9.-]+\))?: [a-z]`

**Types:**
| Type | When to use |
|------|-------------|
| `feat` | New functionality |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, whitespace (no logic change) |
| `refactor` | Code refactoring without behavior change |
| `test` | Add or modify tests |
| `chore` | Tooling, dependencies, CI |
| `perf` | Performance improvement |

**Examples:**
```
feat(iam): implement JWT login

fix(scheduling): correct schedule overlap validation
Closes #42

docs(api): update actor service OpenAPI contract

chore(deps): upgrade Spring Boot to 3.5.0
```

---

## Tags and versioning

Follow [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`. Tag `main` once the release PR is merged:

```bash
git switch main && git pull
git tag -a v1.2.0 -m "Release v1.2.0: short scope summary"
git push origin v1.2.0
```
