# Git Conventions

> **Read this document before making your first commit on the project.**

## Scope and per-repo exceptions

This document is the general standard for the whole project. The `main ← dev ← feat/...`
strategy below assumes a repo with a real CI/CD environment to stage through. Not every
repo in the ecosystem has one — where that's the case, the exception below overrides the
general strategy for that repo only (per "How governance applies" in `00-governance/
README.md`: changed through team agreement, documented here with the reason).

| Repo | Branch strategy | Reason |
|---|---|---|
| **CODE** (`barber-saas`) | `main` ← `dev`/`qa` ← `feat/fix/chore/hotfix`, as described below | Real CI environments per stage |
| **DOCS** (this repo) | Only `main`. No `dev`/`qa`. One branch per change, named `docs/NNN-slug` (`NNN` = the SPEC number the change implements), one PR each, **squash merge** straight into `main` | Documentation-only repo, no build/deploy pipeline to stage through a `dev` environment. Decided 2026-09-14. |

> **CODE currently deviates further:** by explicit decision (2026-09-14), no new branch is
> created per task — work happens directly on `develop`. This is a temporary, explicit
> exception to "every task = one branch + one PR" below, not an oversight; revisit it if
> the team grows past a single contributor.

The rest of this document (naming format, commit format, PR policy, merge policy) applies
to every repo, including DOCS — only the branch *topology* differs.

## Branch strategy

```
main        ← Production. Merge from release only. Always stable.
  └── dev   ← Continuous integration. Merge from features.
        └── feat/[description]    ← One branch per feature/user story
        └── fix/[description]     ← One branch per bugfix
        └── chore/[description]   ← Infrastructure, docs, dependency changes
        └── hotfix/[description]  ← Urgent fixes directly to main
```

**Rules:**
- Nobody commits directly to `main` or `dev`
- Every task = one branch + one Pull Request
- One branch = one task (do not mix different features)
- Branches are deleted after merge

---

## Branch naming format

```
[type]/[description-in-kebab-case]

Examples:
feat/oauth2-login
fix/schedule-overlap-calculation
chore/update-spring-dependencies
hotfix/null-token-expiration
```

**DOCS exception:** branches are named `docs/NNN-slug`, where `NNN` is the SPEC number
(kept for traceability to `_ecosistema/specs/SPEC-NNN-*.md`). Example:
`docs/002-academic-microservice-extraction`.

---

## Commit format (Conventional Commits)

```
[type]([scope]): [lowercase description, imperative mood, no trailing period]

[optional body — explain WHY, not what]

[optional footer — issue/user story references]
```

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

chore(deps): upgrade Spring Boot to 3.2.0
```

---

## Pull Request policy

- **Size:** maximum 400 lines of code (excluding tests). If larger, split it.
- **Reviewers:** minimum 1 approval before merging
- **Review time:** reviewer has a maximum of 24 business hours
- **Template:** use the template at `.github/pull_request_template.md`
- **Green CI:** merge only proceeds if all pipeline checks pass

---

## Merge policy

- Use **Squash and Merge** for features (keeps `dev` history clean)
- Use **Merge Commit** for releases to `main` (preserves full history)
- **Do not** use Rebase & Merge (creates confusion in shared history)

---

## Tags and versioning

Follow [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`

```bash
# When releasing to production
git tag -a v1.2.0 -m "Release v1.2.0: add reports module"
git push origin v1.2.0
```
