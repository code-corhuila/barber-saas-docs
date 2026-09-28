# ADR-008 — Interface Framework: React (React Native for the Mobile Channel)

- **ID:** ADR-008
- **Date:** 2026-09-28
- **Status:** Proposed — pending the teacher's answer (see "Open question")
- **Authors:** Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia

---

## Context

The course norm (4.2.2) requires one interface framework per project, React or Angular, on React 19
or Angular 21 (5.5.1). Our only channel is mobile (C = 1): the domain interfaces are
`barber-saas-<domain>-app` repositories composed by `barber-saas-front`. Annex H describes web
micro-frontends (Module Federation / Native Federation) and does not describe a mobile container.
The prototype is Expo / React Native on React 19.1.

## Open question (to the teacher)

"Our channel is mobile (C = 1, `<domain>-app` repositories). Should `-front` and the `-app`
repositories be React Native with mobile federation, or web micro-frontends in React/Angular as in
annex H? Is one framework per project required, or two micro-frontends in different frameworks?"

---

## Decision (proposed)

**We propose:** **React** for the project — React Native (Expo) with React 19 for `-front` and the
eight `-app` repositories, `-front` as the host that loads each `-app` as a remote and owns the only
HTTP client and the session (norm 5.4.1, 5.5).

---

## Evaluated alternatives (options)

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| **React Native (Expo) + React 19 with mobile federation (proposed)** | matches the real channel; reuses the prototype | federation on React Native is less documented; annex H rules apply by analogy | — (proposed) |
| React 19 web micro-frontends (Vite + Module Federation) | follows annex H literally | drops the mobile channel; a web channel would call for `-portal`, not `-app` (norm 4.4) | depends on the teacher's answer |
| Angular 21 (Native Federation) | full annex H | nobody on the team starts from Angular | cost and time |

---

## Dominant criterion

**The product's real channel (mobile) and reuse of the prototype**, within the "React" option the
norm allows.

## Accepted cost

Mobile federation is less documented than web federation; part of annex H is applied by analogy
and must be justified; the decision may change after the teacher's answer.

---

## Consequences

- No `-app` implements an HTTP client, a token or a session (norm 5.4.1): all of it comes from `-front`.
- Every creation sends `Idempotency-Key`; money is parsed from the typed text (norm 5.4.2, 5.4.3).
- On acceptance: status → Accepted; if the teacher asks for web, a new ADR supersedes this one.

## References

- Course norm 4.2.2, 4.4, 5.4, 5.5, annex H
- Related to: ADR-004
