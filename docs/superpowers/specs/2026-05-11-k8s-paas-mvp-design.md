# K8s PaaS MVP — v0.1 Design

**Date:** 2026-05-11
**Status:** Approved for implementation planning
**Scope:** v0.1 only — what ships in the first public release

---

## 1. Goal and positioning

Build an open-source, self-hostable PaaS that deploys apps to **Kubernetes** (rather than Docker Swarm, which is the failure mode that pushed the maintainer of Ptah-server to archive that project). Strategy is open-source first, with a future managed SaaS once the OSS validates demand.

**Target user for v0.1:** solo developers and hobbyists deploying side projects on their own VPS. Public archetype: r/selfhosted reader who today picks Coolify/Dokploy. We are not yet targeting agencies, multi-user teams, or enterprises — those features are deferred (see Section 8).

**Single-sentence pitch:** "Connect a Kubernetes cluster, point at a GitHub repo with a Dockerfile, get a deployed app with auto-SSL on a custom domain — without writing YAML."

---

## 2. Architecture

Four components total. Three live on the user's VPS (as Docker Compose services); one lives in the user's Kubernetes cluster.

```
┌─────────────────────────────────────────────────────────────────┐
│  User VPS (Docker Compose)                                       │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   Frontend   │    │   API (Go)   │    │  PostgreSQL  │      │
│  │  (React SPA) │◄──►│              │◄──►│   (state)    │      │
│  └──────────────┘    └──────┬───────┘    └──────────────┘      │
│                             │                                    │
│                             │ kubeconfig (BYO)                   │
└─────────────────────────────┼────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  User's K8s cluster (any provider)                               │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │  User apps   │    │ Kaniko Jobs  │    │ cert-manager │      │
│  │ (Deployments)│    │   (builds)   │    │   + Traefik  │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

**Components:**

1. **Go API** — the brain. Handles auth, receives GitHub webhooks, triggers builds, applies K8s manifests, exposes REST/JSON to the frontend.
2. **React SPA** — static UI served by nginx, talks to API via JSON.
3. **PostgreSQL** — stores users, apps, deploys, encrypted env vars, build logs.
4. **User's K8s cluster (BYO)** — where user apps run. The platform installs `cert-manager`, a Traefik Ingress controller, an internal Docker registry, and a `ptah-system` namespace for Kaniko build jobs.

**Key property:** the platform is stateless with respect to the cluster. Source of truth is Postgres plus the cluster's declared manifests. If the platform process is destroyed and reinstalled, it reads cluster state and reconciles.

---

## 3. Core user flow

The "happy path" from zero to deployed app, target: ~10 minutes for a new user.

### 3.1 Initial setup (one-time, ~3 min)

1. User runs `docker compose up -d` on their VPS following the README.
2. Opens the platform URL, creates account (email + password; first registration becomes admin).
3. "Connect cluster" screen: pastes `kubeconfig` text or uploads file.
4. Platform validates the connection, then auto-installs in the cluster:
   - `cert-manager` (for automatic SSL via Let's Encrypt)
   - `Traefik` as the Ingress controller
   - A `ptah-system` namespace containing an internal Docker registry (`registry:2`) and used for Kaniko build Jobs
   - If any of these are already present, detect and skip
5. UI shows "Cluster ready ✓".

### 3.2 Create an app (~2 min)

1. User clicks "New App".
2. Connects GitHub via OAuth (only git provider supported in v0.1).
3. Selects a repo and branch (default `main`).
4. Configures:
   - Dockerfile path (default `./Dockerfile`)
   - Internal port (default `3000`)
   - Domain (default: auto-generated subdomain; or custom)
   - Environment variables (key/value, with `is_secret` flag for UI masking)
5. Clicks "Deploy".

### 3.3 First deploy (~5 min, automatic)

UI shows a live timeline:

```
✓ Webhook received (commit abc123)
✓ Kaniko Job created in cluster
⟳ Building... (streaming logs)
✓ Image pushed → internal registry
✓ K8s Deployment applied
✓ Service + Ingress applied
✓ Let's Encrypt cert provisioned
✓ App live → https://app.example.com
```

### 3.4 Ongoing iteration

- `git push` → GitHub webhook → automatic build + deploy.
- UI shows deploy history, runtime logs (streamed via `kubectl logs`), one-click rollback to a previous successful deploy.

---

## 4. Technical components

### 4.1 Backend (Go API)

- **Router:** `chi` (minimal, idiomatic).
- **K8s client:** official `client-go` (same library used by `kubectl`).
- **DB layer:** `sqlc` (typed Go code generated from raw SQL queries; no ORM magic).
- **Migrations:** `golang-migrate`.
- **Auth:** cookie-based sessions (`gorilla/sessions`) with bcrypt for passwords.
- **Encryption at rest:** AES-GCM with a master key from `ENCRYPTION_KEY` env var.
- **Logging:** structured via stdlib `slog`.

Package structure:

```
cmd/api/main.go         ← entrypoint
internal/
  auth/                 ← login, sessions, middleware
  apps/                 ← CRUD apps, env vars
  deploys/              ← deploy lifecycle, history
  builder/              ← Kaniko Job creation, log streaming
  k8s/                  ← wrapper over client-go (apply, watch, install)
  github/               ← OAuth, webhooks, repo listing
  cluster/              ← kubeconfig connection, validation, install
  db/                   ← sqlc generated + queries
  web/                  ← HTTP handlers, routing
```

### 4.2 Frontend (React SPA)

- **Stack:** Vite + React + TypeScript + TanStack Router + TanStack Query.
- **UI:** Tailwind + `shadcn/ui` (copy-paste components, no lock-in).
- **Build artifact:** static files served by nginx in the same Docker Compose stack.
- **Live updates:** Server-Sent Events for streaming deploy logs (simpler than WebSockets for a one-way channel).

### 4.3 Builder pipeline

When a GitHub push fires a webhook:

1. `POST /webhooks/github` verifies the per-app HMAC signature.
2. API creates a `deploys` row with status `pending`.
3. API creates a K8s `Job` in `ptah-system` using `gcr.io/kaniko-project/executor`:
   - args: `--dockerfile=<path> --context=git://github.com/<owner>/<repo>.git#<sha> --destination=registry.ptah-system.svc.cluster.local:5000/<app>:<sha>`
   - GitHub token (for private repos) mounted as a `Secret`.
4. API watches the Job, streams logs (programmatic `kubectl logs -f` equivalent), appends to `deploys.build_logs`, and forwards to the frontend via SSE.
5. On successful build, API applies/updates `Deployment`, `Service`, `Ingress`, and env-var `Secret` in the user's app namespace (one namespace per app, e.g. `app-myblog`).

### 4.4 Internal registry

- `registry:2` as a `StatefulSet` in `ptah-system` with a 10Gi PVC default.
- `ClusterIP` only — not exposed outside the cluster.
- No internal auth (trust boundary is "has cluster access").

### 4.5 Per-app K8s resources

For each app, the platform maintains four resources in its dedicated namespace:

- `Deployment` (1 replica default, configurable later)
- `Service` (ClusterIP, exposes the internal port)
- `Ingress` (Traefik, annotation `cert-manager.io/cluster-issuer: letsencrypt-prod`)
- `Secret` (env vars from `env_vars` table, decrypted at apply time)

---

## 5. Data model

PostgreSQL, 7 tables.

```sql
users
  id              uuid pk
  email           text unique not null
  password_hash   text not null
  is_admin        bool default false
  created_at      timestamptz

github_integrations
  id              uuid pk
  user_id         uuid fk → users
  github_user     text
  access_token    bytea       -- AES-GCM encrypted
  refresh_token   bytea
  expires_at      timestamptz

clusters
  id              uuid pk
  user_id         uuid fk → users
  name            text
  kubeconfig      bytea       -- AES-GCM encrypted
  status          text        -- 'connecting' | 'ready' | 'error'
  last_check_at   timestamptz
  created_at      timestamptz

apps
  id              uuid pk
  user_id         uuid fk → users
  cluster_id      uuid fk → clusters
  name            text        -- slug, used as k8s namespace component
  github_repo     text        -- "owner/repo"
  github_branch   text        -- default 'main'
  dockerfile_path text        -- default './Dockerfile'
  internal_port   int         -- default 3000
  domain          text        -- "myapp.example.com"
  webhook_secret  text        -- HMAC secret for GH webhooks (per app)
  created_at      timestamptz
  UNIQUE(user_id, name)

env_vars
  id              uuid pk
  app_id          uuid fk → apps
  key             text
  value           bytea       -- AES-GCM encrypted (always)
  is_secret       bool        -- UI masking only
  UNIQUE(app_id, key)

deploys
  id              uuid pk
  app_id          uuid fk → apps
  commit_sha      text
  commit_message  text
  status          text        -- 'pending' | 'building' | 'deploying' | 'success' | 'failed'
  build_logs      text        -- appended during build
  error_message   text
  started_at      timestamptz
  finished_at     timestamptz

sessions
  id              uuid pk
  user_id         uuid fk → users
  token_hash      text unique
  expires_at      timestamptz
```

**Design notes:**

- UUIDs everywhere — eases future multi-tenant migration when SaaS phase begins.
- `bytea` for all encrypted fields — kubeconfigs, OAuth tokens, env-var values. Single master key in `ENCRYPTION_KEY` env var; rotation is a v0.3+ concern.
- `build_logs` as a `text` column is fine at v0.1 scale (typical < 1MB per deploy). If it grows, move to object storage in v0.2.
- No `teams` or `roles` tables — explicitly out of scope. When multi-user lands, add `org_id` foreign keys.
- `webhook_secret` is per-app so rotation is per-app.

---

## 6. Error handling and operation

### 6.1 Failure matrix

| Failure | Response |
|---|---|
| Invalid kubeconfig at paste time | Immediate validation (`/api` ping); clear error; block save. |
| Cluster becomes unreachable after connection | 60s health check; status → `error`; red banner in UI; keep retrying. |
| Build fails (broken Dockerfile, etc.) | Deploy marked `failed`; logs visible; current running app unchanged. |
| Build hangs > 30 min | Hard timeout; Job killed; deploy marked `failed`. |
| Invalid GitHub webhook signature | 401, ignored, security log entry. |
| GitHub token expires | API detects 401, marks integration expired, UI asks to reconnect. |
| Postgres down | API returns 503; frontend shows "platform unavailable, retrying"; Docker Compose restart policy brings it back. |
| cert-manager fails to issue SSL | Deploy marks `success` with visible "SSL pending" warning; retries every 5 min. |
| Custom domain DNS not pointing at cluster | Pre-check at save: DNS query; warn if mismatch. Non-blocking. |
| Push to internal registry fails (disk full) | Build fails with clear error; UI surfaces a "purge old images" action. |

### 6.2 Error philosophy

- **No silent failures.** Every error leaves a visible, actionable record in the UI with context.
- **Idempotency where it matters.** Re-applying the same K8s manifest is safe (K8s `apply` guarantee).
- **Light reconciliation loop:** every 5 minutes a worker re-applies the expected manifests for each app. If a user deleted something via `kubectl`, it gets restored. If the cluster lost state, recovery is automatic.

### 6.3 Observability

- Structured JSON logs via `slog`.
- Optional Sentry integration via env var, disabled by default in self-hosted builds.
- `/healthz` endpoint verifies DB + last successful cluster check.
- No Prometheus metrics in v0.1 (deferred to v0.3).

### 6.4 Backups (of the platform itself)

- Postgres volume is a named Docker volume. README documents `pg_dump` manual procedure.
- **No automatic backups in v0.1** — explicitly user's responsibility, called out in docs.

### 6.5 Upgrades

- `docker compose pull && docker compose up -d` upgrades.
- DB migrations applied automatically at API startup.
- Rollback procedure documented (pin image tag in compose file).

### 6.6 Known operational risks (called out, not solved in v0.1)

- **Loss of `ENCRYPTION_KEY`** — all encrypted secrets become unrecoverable. README warns prominently.
- **Manual `kubectl` edits** — reconciliation loop fights them; documented as expected behavior.

---

## 7. Testing strategy

### 7.1 Test levels

| Level | Coverage | Tools |
|---|---|---|
| Unit | Pure logic: encryption, slug generation, input validation, parsers (kubeconfig, GH webhook signature). | `testing` stdlib + `testify`. |
| Integration (DB) | sqlc queries, migrations, transactions. Real Postgres in ephemeral container. | `testcontainers-go` with Postgres. |
| Integration (K8s) | `apply` logic, Job watch, cert-manager/Traefik install. Real ephemeral cluster. | `kind` (Kubernetes-in-Docker) spun up in CI. |
| E2E | Golden path only: register user → connect kind cluster → connect GitHub (mocked) → create app → deploy from fixture repo → assert app responds HTTP. | Go test orchestrating kind + Postgres + API. |
| Frontend | Only components with non-trivial logic (forms, SSE log parsing). | Vitest + React Testing Library. |

### 7.2 Out of scope for v0.1 testing

- Visual / snapshot tests for the UI (manual check is enough).
- Performance / load tests.
- Disaster recovery / chaos testing (manual, documented).

### 7.3 CI

- GitHub Actions.
- Pipeline: `lint` (golangci-lint, eslint) → `unit + integration (DB)` → `integration (K8s)` via kind → `e2e` → `build images`.
- E2E runs only on PRs to `main` (cost optimization).

### 7.4 Manual QA before release

Self-imposed checklist before tagging v0.1:

- Fresh install from `docker-compose.yml` on a clean VPS.
- Deploy three example apps (different profiles): a Go static binary, a Node.js app, a longer build (e.g. Rails).
- Force two failure modes (broken Dockerfile, disconnected cluster) and verify UX.

---

## 8. Scope cuts — what is NOT in v0.1

### 8.1 Deferred to v0.2 (next cycle, ~1-2 months after v0.1)

- **Auto-install of k3s via SSH** (the other half of the BYO+auto-install decision; BYO ships in v0.1, auto-install follows once core flow is validated).
- **GitLab and Gitea** as git sources (only GitHub in v0.1).
- **Metrics and dashboards** (Prometheus + Grafana in-cluster; per-app CPU/RAM in UI).
- **Notifications** (Discord/Slack webhooks on deploy failure).

### 8.2 Deferred to v0.3+ (post-validation)

- **Managed databases** (one-click Postgres/MySQL/Redis). Highly requested but operationally heaviest.
- **Multi-user / teams** (invitations, roles).
- **Autoscaling** (HPA configurable from UI).
- **Persistent volumes for apps** (PVC management).
- **Aggregated logs** (Loki/Vector).
- **Templates / 1-click apps** (WordPress, Ghost, etc.).
- **Multi-cluster per account** (v0.1 is one user = one cluster).

### 8.3 Deferred to SaaS phase

- Billing (Stripe/Paddle).
- True multi-tenancy (isolated orgs).
- GitHub OAuth as platform login.
- Audit / activity log.
- Automated platform backups.

### 8.4 Likely never (or much later)

- Windows containers.
- Differentiated UX for managed K8s providers (GKE/EKS/AKS work as BYO; we don't integrate their APIs).
- App marketplace.
- CLI (possible later, but web UI is the priority).

### 8.5 Scope discipline rule

When a feature request appears during v0.1 development, the question is not "is this a good idea?" — it usually is. The question is "does this block v0.1 launch?". If no, it goes to v0.2+.

---

## 9. Open questions (to resolve during implementation planning)

These are intentionally deferred to the implementation plan, not the design:

- Project / product name (placeholder: `k8s-paas-mvp`).
- License choice (MIT vs Apache 2.0 vs Fair Source like Ptah). Likely MIT for maximum OSS adoption, but Fair Source is defensible if SaaS is on the roadmap.
- Default base domain for auto-generated subdomains (likely needs a cheap wildcard domain we own).
- Exact Kaniko version pin and how we keep it updated.

The reconciliation loop is implemented as a goroutine inside the API process for v0.1 (no separate worker process). If load justifies it later, it can be extracted.

---

## 10. Decision log

Decisions made during the brainstorming session, with the reasoning preserved for future contributors:

| # | Decision | Why |
|---|---|---|
| 1 | OSS-first, SaaS later | Validate demand with the open-source community before investing in billing/multi-tenancy infrastructure. |
| 2 | Target solo devs in v0.1 | Largest and most vocal audience; technically simplest (single-user). Coolify proved this audience works. |
| 3 | BYO cluster in v0.1, auto-install k3s in v0.2 | Ships faster; validates the core "K8s-based PaaS" idea before tackling installer complexity. |
| 4 | Git + Dockerfile build (not pre-built images, not Compose import) | Heroku-like UX is the differentiator; pre-built images alone are too bare. |
| 5 | No managed databases in v0.1 | Disproportionately complex (storage, backups, operators). Users connect external DB (Neon, Supabase). |
| 6 | Go for backend | Native K8s ecosystem (`client-go`); aligned with long-term direction even at cost of steeper learning curve vs Laravel/TypeScript. |
| 7 | Docker Compose deployment of the platform | Trivial install on any VPS; avoids the chicken-and-egg "platform runs inside cluster" problem. |

---

**Approved for next step:** writing an implementation plan via the `writing-plans` skill.
