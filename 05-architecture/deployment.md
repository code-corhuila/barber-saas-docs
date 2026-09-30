# Deployment — BarberSaaS

> How the platform described in [`overview.md`](overview.md) is assembled and run in each
> environment. Rules come from the course norm 2026-B (5.6, 5.9, 6.2, 7) and its annexes F
> (api-gateway) and G (infra).

> **Current state (re-checked 2026-09-30):** the 29 repositories hold only `README.md` and
> `.github/CODEOWNERS` on `develop`, `qa` and `main`. Nothing in this document is deployed yet; it
> is the target every repository must meet. What is missing, file by file, is listed in §10.

---

## 1. Deployment model

- **Unit of deployment:** one container per service and one per database instance.
- **Orchestration:** Docker Compose. Every runnable repository (`-db`, `-api`, `-worker`,
  `-workflow`, `-api-gateway`, `-front`) ships its own `deploy/compose.yml`;
  `barber-saas-infra` only **composes** them with `include` (norm 5.9.1, annex G).
- **No Kubernetes.** Not required by the norm and out of the team's capacity (ADR-004 risks).
- The repositories are cloned as siblings; `-infra` is the folder the platform is started from.

```
workspace/
  barber-saas-infra/            <- everything starts here
  barber-saas-api-gateway/
  barber-saas-workflow/
  barber-saas-worker/
  barber-saas-front/
  barber-saas-<domain>-db/      x 8
  barber-saas-<domain>-api/     x 8
```

---

## 2. Infrastructure diagram

```
                    host
  ─────────────────────────────────────────────────────────────────────────
     :8000 api-gateway (NGINX)                 :3000 Grafana (observability)
          │
  ════════╪═════════════════ network: platform (internal) ═════════════════
          │
          ├── /api/v1/auth ................ identity-auth-api:8080 ──► identity-auth-db:5432
          ├── /api/v1/barbershops|services|barbers barbershop-api:8080 ──► barbershop-db:5432
          ├── /api/v1/appointments ........ appointment-api:8080 ───► appointment-db:5432
          ├── /api/v1/barber-schedules|schedule-exceptions|availability
          │                                 schedule-api:8080 ──────► schedule-db:5432
          ├── /api/v1/loyalty ............. loyalty-api:8080 ───────► loyalty-db:5432
          ├── /api/v1/notifications|device-tokens
          │                                 notifications-api:8080 ─► notifications-db:27017 (rs0)
          ├── /api/v1/finance|inventory ... finance-inventory-api:8080 ► finance-inventory-db:5432
          ├── /api/v1/plans|platform ...... platform-admin-api:8080 ► platform-admin-db:5432
          └── /api/v1/sagas ............... workflow:8080 ──────────► saga store, PostgreSQL (ADR-009, proposed)

      worker (health only) ── service token ──► domain APIs
      otel-collector · prometheus  ◄── logs, metrics, traces from every container
```

---

## 3. Network and ports

| Rule | Value | Source |
|---|---|---|
| Shared network | `platform`, external, created by `-infra/scripts/up.sh` | Annex G |
| Published to the host | **Only** `api-gateway` (`8000`) and Grafana | Norm 5.6.1 |
| Domain services | `expose: 8080`, never `ports` | Annex G |
| Addresses inside the network | Service name, never `localhost` (`appointment-db:5432`) | Annex G |
| Gateway → service | `proxy_pass` through a variable, resolved on every request (`resolver 127.0.0.11`) | Norm 5.6.3, annex F |
| CORS | Only the front's origins; headers `Authorization`, `Content-Type`, `Idempotency-Key`, `X-Correlation-Id`; exposes `X-Correlation-Id`, `Location` | Norm 5.6.4 |

If a domain container is down, its routes answer `503 SERVICE_UNAVAILABLE` with the common
error envelope and every other route keeps working.

---

## 4. Environments

Each code repository has three permanent branches, one per environment (norm 6.2.1). Changes
move between them by cherry-pick with `-x`, never by merge (norm 10).

| Environment | Branch | Fed by | Keys and service tokens | Data |
|---|---|---|---|---|
| **develop** | `develop` | `feat/`, `fix/`, `chore/` PRs | Development keys from `scripts/dev-keys.sh` | Seeds from each `-db` |
| **qa** | `qa` | `qa/` PRs (cherry-picks from `develop`) | Issued by identity-auth, stored as environment secrets | Synthetic |
| **main** | `main` | `release/` and `hotfix/` PRs | Issued by identity-auth, stored as environment secrets | Real |

Each environment has its own `-infra/env/.env.<environment>.example` with variable names and
placeholders only. Where `qa` and `main` are hosted is **not decided** (see §9).

---

## 5. Databases and migrations

- Eight instances, one per domain, each with its own volume (norm 7.1): PostgreSQL for seven,
  MongoDB single-node replica set for `notifications` (ADR-006).
- Each `-db` ships, next to its instance, a **migration runner** `<domain>-db-migrate`
  (Liquibase with a pinned image version, ADR-007) with `profiles: [tooling]`: it does not
  start with `up`, it waits for its database to be healthy, and it mounts the `-db`
  repository read-only.
- Migrations run from `-infra`, once per `-db`:
  `docker compose --env-file .env run --rm appointment-db-migrate`. Running it again must
  apply zero changes.
- **Order in `qa` and `main`:** migrate the database **before** deploying the `-api` version
  that needs it. A breaking change (rename, drop or retype a column) ships in two releases,
  expand then contract (annex A).

---

## 6. Identity and secrets

| Item | develop | qa / main |
|---|---|---|
| `JWT_PUBLIC_KEY` (RS256, read by every service) | `dev-keys.sh` writes it to `.env` | Published by identity-auth, environment secret |
| JWT private key | `keys/` in `-infra`, ignored by git | Only inside identity-auth |
| `SERVICE_TOKEN` (worker, workflow) | `dev-keys.sh` | Issued by identity-auth |
| DB credentials (`<DOMAIN>_DB_USER`, `<DOMAIN>_DB_PASSWORD`) | `.env` | Environment secrets |
| Firebase and SMTP credentials | `.env` | Environment secrets |

`.env`, `keys/` and `*.pem` are ignored by git; a password that is missing makes the command
fail with a clear message (`${VAR:?…}`). Development keys never leave `develop` (norm 5.9.2).

---

## 7. Observability

| Signal | How |
|---|---|
| Logs | Structured JSON, one line per event, always with `X-Correlation-Id` |
| Traces | OpenTelemetry collector in `-infra` (`observability/otel-collector.yaml`) |
| Metrics | Prometheus scraping each service (`observability/prometheus.yml`) |
| Dashboards | Grafana, the only observability port published to the host |
| Health | `GET /health` on every service without token; the gateway has its own |

The same `X-Correlation-Id` must appear in the logs of every service a request touched.

---

## 8. Startup (develop)

```bash
cd barber-saas-infra
cp env/.env.develop.example .env           # fill in every *_DB_PASSWORD
./scripts/dev-keys.sh                      # RSA pair + JWT_PUBLIC_KEY + SERVICE_TOKEN
./scripts/up.sh                            # checks .env, creates 'platform', starts everything
docker compose --env-file .env run --rm identity-auth-db-migrate   # repeat for each -db
curl -H "Authorization: Bearer $(./scripts/dev-token.sh alice)" \
     http://localhost:8000/api/v1/appointments
```

Until a schema is migrated, that domain's API answers `500 INTERNAL_ERROR` with a `traceId`;
nothing waits forever and nothing fails silently.

### Resource targets

The full platform is ten JVM services plus eight database instances. Each `deploy/compose.yml`
declares memory limits so a laptop can run it; these are **targets, not measurements**:

| Container | Memory limit (target) |
|---|---|
| Each Java service (JVM heap capped inside) | 512 MB |
| Each PostgreSQL instance | 256 MB |
| MongoDB (notifications) | 512 MB |
| Gateway, collector, Prometheus, Grafana | 128–256 MB each |

When the machine cannot hold everything, start only the slice under work (for example
identity-auth, barbershop, schedule, appointment and the gateway).

---

## 9. Open points

| ID | Question | Blocks |
|---|---|---|
| DEP-01 | Hosting for `qa` and `main` (the prototype used Railway; nothing chosen for the polyrepo) | Release evidence |
| DEP-02 | Saga state instance owned by `-workflow` or a separate `workflow-db` repository | ADR-009 (awaiting teacher) |
| DEP-03 | Event transport between services: message broker or worker-polled outbox | AT-004 in `overview.md` |

---

## 10. Pending — not implemented yet

Every artifact this document relies on, and where it has to appear. None exists today in any
branch of the repositories (checked with `git ls-files` on `develop`, `qa` and `main`,
2026-09-30).

| Artifact | Repository | Needed by |
|---|---|---|
| `compose.yml` with `include` of every repository, `platform` network | `barber-saas-infra` | §1, §8 |
| `scripts/up.sh`, `scripts/dev-keys.sh`, `scripts/dev-token.sh` | `barber-saas-infra` | §6, §8 |
| `env/.env.develop.example`, `env/.env.qa.example`, `env/.env.main.example` | `barber-saas-infra` | §4, §6 |
| `observability/otel-collector.yaml`, `observability/prometheus.yml`, Grafana | `barber-saas-infra` | §7 |
| `deploy/compose.yml` + NGINX config with one routes file per domain | `barber-saas-api-gateway` | §2, §3 |
| `deploy/compose.yml` with the instance and its `<domain>-db-migrate` runner; Liquibase changelog | each `barber-saas-<domain>-db` (8) | §5 |
| `Dockerfile` + `deploy/compose.yml` (`expose: 8080`, memory limit, `GET /health`) | each `barber-saas-<domain>-api` (8), `-workflow`, `-worker` | §3, §7, §8 |
| `.gitignore` covering `.env`, `keys/`, `*.pem` | every runnable repository | §6 |
| Hosting for `qa` and `main` | — | DEP-01 |

The `-app` repositories (8) are not containers: they are remotes loaded by `barber-saas-front`, so
they do not appear in the compose file.

---

## 11. Verification checklist

- [ ] With the repositories cloned as siblings, `up.sh` starts the whole platform
- [ ] Only the gateway and Grafana publish ports to the host
- [ ] Each `-db` starts its own instance; its migration applies, and a second run applies nothing
- [ ] `-infra` contains no migration and no credential in any `compose.yml`
- [ ] A resource is created and listed through the gateway with a token from `dev-token.sh`
- [ ] worker and workflow authenticate with `SERVICE_TOKEN`
- [ ] One `X-Correlation-Id` appears in the logs of every service a request touched
- [ ] Stopping one domain container makes only its routes answer `503`

---

## Key correlations

- Topology and service catalog → `05-architecture/overview.md`
- Engines and migration tool → `ADR-006`, `ADR-007`
- Saga state store → `ADR-009` (proposed)
- CI/CD per repository → `10-devops/`
- Branching and promotion → `00-governance/branching-policy.md`, `00-governance/git-conventions.md`
