# Open Questions

> `guidelines.md` and `authentication.md` close the contract's baseline decisions
> (versioning, pagination, error format, JWT mechanism, roles, tenant isolation). This file
> declares what the contract layer still leaves open on purpose — each with the evidence
> that it's a real gap (not a guess), an owner, and the condition that closes it.

## OQ-01 — No rate limiting contract on `/api/auth/**`

**Status:** open.

**Evidence:** `00-governance/security-rules.md` (line 94) and `05-architecture/overview.md`
(risk `AT-002`, line 185) already flag brute-force exposure on login as a known risk,
"not yet implemented," targeted "before production." But nothing in `07-api/` reflects
this at the contract level: `guidelines.md`'s status-code table has no `429`, and
`auth-service.yaml` documents no rate-limit response for `POST /auth/login` or
`POST /auth/refresh`.

**Why it's still open:** the mechanism (Redis-backed, per overview.md's cache layer) isn't
built yet, so there is no real response shape to document — writing one now would be a
guess, not a contract.

**Responsible:** Daniel Cerquera — track alongside the Redis rate-limiting implementation;
once the throttling middleware exists, add the `429` response (with `Retry-After` header)
to `_shared.yaml#/components/responses/` and reference it from `auth-service.yaml`.

**Closing criterion:** a `429 TOO_MANY_REQUESTS` response, using the existing
`ErrorResponse` schema, is documented for every rate-limited endpoint, and
`guidelines.md`'s status-code table includes it.

---

## OQ-02 — `POST /api/admin/loyalty/grant` has no idempotency contract

**Status:** open, known non-drift gap.

**Evidence:** `02-domain/domain-events.md` (lines 113–116) documents this explicitly: the
manual grant endpoint "is not idempotent against an appointment that already triggered an
automatic grant. If staff also grant a sticker manually for the same completed appointment,
the client can be credited twice." It's flagged there as "a candidate for a short SPEC, not
fixed here" — but no contract file declares it, and no one owns closing it.

**Why it's still open:** fixing it means picking an idempotency strategy (an
`Idempotency-Key` header, or a server-side check against the appointment's existing grants)
and that decision hasn't been made — `02-domain/domain-events.md` only names the symptom.

**Responsible:** Daniel Cerquera — bring to the next SPEC round as a short, scoped SPEC
(candidate: SPEC-009) that picks one strategy and updates both `contracts/openapi/` (once
a `loyalty-service.yaml` or equivalent exists) and `02-domain/domain-events.md`.

**Closing criterion:** the endpoint's contract states its idempotency guarantee explicitly
(key-based or check-based), and `domain-events.md`'s "known gap" note is removed or
resolved to point at the closing SPEC.

---

## OQ-03 — `notification-service.yaml` has no extraction timeline or owner

**Status:** open, contract is a placeholder by design.

**Evidence:** the contract's own `info.description` says it is a "planned contract for
when notification is extracted from the modular monolith per ADR-003. Not yet an
independently deployed service" — today it's `com.barbersaas.notification` inside the
monolith, reached in-process, not through this OpenAPI file. ADR-003 (per
`05-architecture/decisions/records/`) justifies *why* it will be extracted, but neither the
ADR nor the contract says *when*, or who drives the extraction.

**Why it's still open:** the extraction is correctly sequenced after the monolith is stable
(per ADR-002/003's incremental-extraction rationale), so committing to a date now would be
speculative.

**Naming is open too, for the same reason.** The course convention names an extracted
component `<abbr>-<domain>-<piece>`, but assigning that name now would mean picking a side
of a question the team hasn't formally closed: ADR-002/003 (currently in effect) describe
one service extracted incrementally from the monolith, while a full-polyrepo split
(one repo per domain, with notification split further into separate api/app/db pieces) has
already started being explored outside this repo but isn't recorded as a decision here.
Naming the component today would silently pick the second model without the ADR to back
it — the name is a consequence of that decision, not a substitute for making it.

**Responsible:** Daniel Cerquera — flag for the team's next MVP-boundary planning session
(see `00-governance/branching-policy.md`'s MVP cadence) so a target MVP (2 or 3) gets
assigned, the monolith-extraction-vs-polyrepo question gets resolved with its own ADR, and
the component name follows from whichever wins.

**Closing criterion:** an ADR resolves whether notification extracts as one service or a
polyrepo split, `notification-service.yaml`'s `info.description` names the resulting
component per `<abbr>-<domain>-<piece>` and a target MVP milestone, and that milestone is
tracked in `15-project-control/`.
