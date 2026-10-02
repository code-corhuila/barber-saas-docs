# ADR-013 — Interface: One Hybrid Mobile App (Ionic + Capacitor) with an Angular Shell, Four Angular and Four React Domain Apps

- **ID:** ADR-013
- **Date:** 2026-10-02
- **Status:** Accepted
- **Supersedes:** ADR-008 (React Native for the whole interface, *Proposed*)
- **Authors:** Carlos Mauricio Leal Medina, Daniel Felipe Cerquera Idrobo, Juan Pablo Borrero Morales, Carolay Arraut Heredia

---

## Context

BarberSaaS is a **mobile-only** product (C = 1): owners, barbers and clients all use the phone app,
and the teacher created eight `barber-saas-<domain>-app` repositories plus `barber-saas-front` for
it. There is no web channel and none is planned.

Annex J of the course norm (2026-10-01, J.1.2; J.9 corrects 4.2.2) requires one interface per
domain, packaged by the front, built with **React and Angular — at least two frameworks**. The annex
is written for every team and most of them build web portals; for us, the domain interface is the
`-app` repository. Annex H keeps its central rule for both frameworks: **the HTTP client and the
session live in the container, and only there** (norm 5.4.1).

ADR-008 proposed React Native (Expo) for every repository, reusing the prototype. React Native
draws native components, while Angular only reaches a phone through a web runtime, so the two
cannot be composed as micro-frontends inside one React Native host.

The prototype (`code-corhuila/barber-saas`, `barbersaas-mobile`) is advanced: Expo 54 with React 19,
about 43 screens grouped by role (`(admin)`, `(barber)`, `(client)`, `(super-admin)`, `(auth)`),
typed API modules, React Query and a Zustand session store. It is the base the product is finished
from.

**Known constraints:**
- The interface must keep running as **one** mobile app with **one** front that packages the
  domain apps.
- Two frameworks are mandatory; the team already works in React.
- The teacher's templates for the front are Angular (`front-shell-angular`, `portal-angular`, with
  Native Federation).
- The week-10 and week-15 cuts are close.

---

## Decision

> One hybrid mobile app built with Ionic and Capacitor: an Angular shell in `-front` that loads
> four Ionic Angular and four Ionic React domain apps.

**We decided:**

| Repository | Framework | Role |
|---|---|---|
| `barber-saas-front` | **Angular 21 + Ionic 8**, Native Federation, **Capacitor 7** | Shell and the native app: navigation, sign-in, the only HTTP client and session, error envelope, builds the Android/iOS app |
| `platform-admin-app`, `finance-inventory-app`, `loyalty-app`, `notifications-app` | **Ionic Angular** (Angular 21) | Remotes that expose `./routes` and run inside the shell's injector (teacher's `portal-angular` template) |
| `identity-auth-app`, `barbershop-app`, `appointment-app`, `schedule-app` | **Ionic React** (React 19) | Remotes that expose `./mount(element, context)`; the shell wraps each in a host component per route |

**One client, two frameworks (norm 5.4.1, annex H):**
- Angular remotes receive the shell's `HttpClient` and interceptor through the injector, and never
  call `provideHttpClient()`.
- The shell also **exposes `./apiClient`**, a framework-neutral client built on the same rules
  (gateway URL, token, `X-Correlation-Id`, 10 s timeout, `ApiError` with `userMessage`). React
  remotes import it as `shell/apiClient` and never call `fetch` or `axios` against the API, as
  annex H requires for React.
- The session (token, user, role) lives only in the shell; a remote reads it from the context it
  receives and never stores its own.

**Prototype as the base.** Each screen is moved, not redesigned:
- Screens are **regrouped by domain** instead of by role; each domain app shows what each role may
  see inside its own routes.
- The visual layer moves from React Native components to Ionic components: almost one-to-one for
  the four React domains, rewritten with the same design for the four Angular domains.
- `src/types/` and the request logic of `src/api/` are reused; `src/api/client.ts` and
  `authStore` move into the shell.
- Expo modules become Capacitor plugins: notifications → `@capacitor/push-notifications`, image
  picker → `@capacitor/camera`, location → `@capacitor/geolocation`, sharing →
  `@capacitor/share`; printing needs a plugin to be chosen.
- Endpoints move from the monolith's `/api/...` to `/api/v1/...` per domain through the gateway,
  as `07-api/contracts/openapi/` defines.

**Remotes in the packaged app.** In `develop`, `federation.manifest.json` points to each domain
app's dev server. For an installable build, each remote's bundle is copied into the shell's assets
and the manifest points to those local paths, so the app starts without a network round trip per
domain.

**Justification:** it is the only option that keeps **one** mobile app with **one** front that
packages the domain apps in **two** frameworks, and it builds on the teacher's Angular templates
and annex H instead of working around them.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Hybrid app: Angular shell + Ionic Angular and Ionic React remotes, Capacitor (chosen)** | One app, real micro-frontends, two frameworks, uses the teacher's templates and annex H; one client and session | The React Native screens are moved to Ionic; React remotes inside an Angular shell need a mount wrapper and the exposed client | — (chosen) |
| Two separate apps (React Native for some domains, Ionic Angular for the others) | Keeps the React Native prototype as it is | Two installed apps; no front packaging the domain apps; two sessions | Breaks "the front packages the domain interfaces" and norm 5.4.1 |
| React Native host that opens Angular domains in a WebView | Keeps React Native | Two runtimes that share no session or client; hard to test and maintain | Two clients and two sessions (norm 5.4.1) |
| Web portals in React and Angular next to the mobile app | Follows annex H literally | A channel the product does not need; doubles the interface work | Out of scope: BarberSaaS is mobile only |
| Keep React Native everywhere (ADR-008) | Full reuse of the prototype | One framework only | Contradicts Annex J J.1.2 |

---

## Consequences

**Positive:**
- The interface complies with Annex J (two frameworks) without adding a web channel.
- `-front` owns the only client and session, verifiable in both frameworks (no
  `provideHttpClient()` in an Angular remote, no `fetch`/`axios` to the API in a React remote).
- A domain app that fails to load shows its own notice; the shell and the other domains keep
  working (annex H).

**Negative / Trade-offs:**
- The React Native screens are not reused as code: the four React domains are ported to Ionic
  React and the four Angular domains are rewritten in Ionic Angular, keeping the design.
- A hybrid app renders through a WebView: animations and long lists can feel less native than
  React Native.
- React remotes in an Angular shell need an extra contract (`./mount` plus `shell/apiClient`)
  that the teacher's templates do not include.
- Federated remotes inside an installed app add a build step (copying each remote into the
  shell's assets).
- Some prototype screens mix domains (dashboards, statistics) and must be assigned to one domain
  app or split.

**Impact on the system:**
- Affected repositories: `barber-saas-front` and the eight `-app` repositories.
- Documents that must be updated: `05-architecture/overview.md`, `05-architecture/deployment.md`
  (§10), `08-diagrams/c4/c2-containers.md` and divergence D-7, `CLAUDE.md`, `12-ux-ui` (screen →
  domain map). Several of them are touched by open pull requests, so they are updated afterwards.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A React remote creates its own HTTP client | Medium | High | Review rule and a CI search for `fetch(`/`axios` against `/api` in React remotes |
| An Angular remote calls `provideHttpClient()` | Medium | High | Same check as the teacher's template; review against annex H |
| Shared dependency versions clash between shell and remotes | Medium | Medium | `shareAll` with `singleton` and `strictVersion` in the shell; React and Angular versions pinned |
| The port of 43 screens does not fit before the cut | High | High | Port by domain in order of the critical flow (identity-auth, barbershop, schedule, appointment first) |
| A Capacitor plugin does not cover a prototype feature (printing) | Low | Low | Choose the plugin before porting that screen |

---

## References

- Course norm 2026-B, **Annex J** (J.1.2, J.9 → 4.2.2), annex H, numerals 5.4 and 5.5
- Teacher's templates `front-shell-angular` and `portal-angular` (Native Federation)
- Issue #60 (question to the teacher about a mobile-only project)
- Related to: ADR-004, ADR-012
- Supersedes: ADR-008
