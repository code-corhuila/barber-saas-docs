# MVP & Weekly — Index

> Consolidates, in one place, where the MVP and weekly-tracking information for this
> project actually lives. This file does not move that information — it points to it —
> since the weekly deliverable is tracked in a separate academic repository, not in DOCS.

## MVP cadence

Defined in [`00-governance/branching-policy.md`](../00-governance/branching-policy.md#release-branches):

| Milestone | Week |
|---|---|
| MVP1 | Week 5 |
| MVP2 | Week 10 |
| MVP3 | Week 15 |

Each MVP is cut from `main` as a `release/x.y.z` branch in the CODE repo (`barber-saas`),
filled gradually with commits validated in `qa`, and merged back to `main` via Pull Request.

## Weekly tracking

Per-week academic status (session notes, HU status, posters) is tracked in the separate
weekly-grading repository referenced in
[`00-governance/agile-conventions.md`](../00-governance/agile-conventions.md) —
**not** governed by this DOCS repo. That repository organizes each week as:

```
NN-week/
├── 01-session/
├── 02-session/
└── hu-status/
    └── README.md   ← weekly summary
```

## Related documents in this repo

- [`00-governance/branching-policy.md`](../00-governance/branching-policy.md) — MVP release cadence
- [`00-governance/agile-conventions.md`](../00-governance/agile-conventions.md) — backlog tool and weekly-repo boundary
- [`03-product/product-backlog.md`](../03-product/product-backlog.md) — current backlog
