# Task-based interview — Unified RAG Studio (internal)

> Supplement for candidates reviewing implementation tasks. This file keeps **cumulative** Q&A: detailed **P0-* … P2-*** blocks are preserved from historical work; **Phase 3+** uses **`Phase N · …`** headings and is appended after Phase 2. Answers reflect design intent and current codebase choices unless noted.

---

## P0-1 · Monorepo Skeleton

### Monorepo & Workspace Concepts

**Q1: What is a monorepo and why did you choose it for RAG Studio?**

A monorepo stores multiple related projects (frontend, backend, shared data) in a single git repository. For RAG Studio we chose it because:
- The frontend (`apps/web`) and backend (`apps/api`) share type definitions and JSON catalogs from the `data/` folder — a monorepo makes cross-package imports straightforward.
- A single PR can atomically update both the API contract and the frontend component that consumes it, eliminating version drift.
- CI/CD runs once per commit, giving a unified view of build health.

The trade-off is that the repository grows large and CI must be configured to run only the affected workspaces (e.g., via `turbo` or `nx` affected commands).

---

**Q2: How do npm workspaces work, and what does the root `package.json` do?**

`package.json` at the root declares `"workspaces": ["apps/web", "apps/api"]`. When you run `npm install` at the root, npm:
1. Installs all dependencies for every workspace into a single `node_modules/` at the root (hoisting).
2. Creates symlinks in `node_modules/` so workspace packages can import each other by name.

The root `package.json` also exposes convenience scripts (`npm run dev`, `npm run build`) that delegate to the individual workspace via `--workspace=apps/web`, keeping developer ergonomics simple.

---

**Q3: Why are placeholder directories created with `.gitkeep` files?**

Git tracks files, not directories. An empty directory is invisible to git and will not appear after a fresh clone. Adding a `.gitkeep` (a convention, not a git feature) forces git to track the directory, so the full project structure is present for every developer immediately after `git clone` — before any code is written.

---

### `.gitignore` Design

**Q4: Walk me through the sections of the `.gitignore` and the reasoning behind each.**

| Section | Key entries | Reason |
|---------|-------------|--------|
| Internal docs | `docs/internal/`, `prompt_history/` | Keep architecture specs and session logs out of the public repo |
| Node/Next.js | `node_modules/`, `.next/`, `dist/` | Build artefacts are reproducible from source; committing them causes merge conflicts |
| Python | `__pycache__/`, `*.pyc`, `.venv/`, `*.egg-info/` | Compiled bytecode and virtual environments are environment-specific |
| Environment | `.env`, `.env.*`, `!.env.example` | Secrets must never be committed; the negation `!.env.example` keeps the template file tracked |
| Docker | `.dockerignore` | Generated per project; not shared |
| Databases/data volumes | `postgres-data/`, `qdrant-data/`, etc. | Docker-managed volumes; large and should not be committed |

---

**Q5: Why do we commit `.env.example` but ignore `.env`?**

`.env.example` is a *template* — it lists every variable the application needs with placeholder values, so new developers know exactly what to configure. `.env` contains real secrets (API keys, database passwords) and must never be committed. The `.gitignore` rule `!.env.example` is a negation that exempts the template from the `*.env*` blanket ignore.

---

### Environment Variables

**Q6: What is the purpose of each environment variable group in `.env.example`?**

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | PostgreSQL connection string for SQLAlchemy; uses `asyncpg` driver for async queries |
| `REDIS_URL` | Redis connection for Celery broker + result backend + embedding cache |
| `QDRANT_URL` | Vector database HTTP endpoint |
| `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `COHERE_API_KEY`, `GOOGLE_API_KEY` | LLM and embedding provider credentials |
| `MLFLOW_TRACKING_URI` | MLflow server URL for logging Autopilot experiment runs |
| `MINIO_ENDPOINT`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY` | S3-compatible object storage for uploaded documents |
| `NEXT_PUBLIC_API_URL` | The backend URL that the Next.js frontend calls; `NEXT_PUBLIC_` prefix makes it available in browser bundles |
| `SECRET_KEY` | Used to sign JWTs and session tokens |

---

**Q7: Why is `NEXT_PUBLIC_API_URL` prefixed with `NEXT_PUBLIC_`?**

Next.js only exposes environment variables prefixed with `NEXT_PUBLIC_` to the browser bundle. Variables without this prefix are available only on the server (Node.js runtime). Since the frontend needs to make API calls from the browser, the backend URL must be browser-accessible, hence the prefix. Variables that should stay server-side only (e.g., `SECRET_KEY`) deliberately omit the prefix.

---

### Project Structure

**Q8: Describe the top-level directory layout and the responsibility of each folder.**

```
rag-studio/
├── apps/web/     Next.js frontend — all UI components, pages, state stores
├── apps/api/     FastAPI backend — routers, agents, core RAG services, DB models
├── data/         Shared JSON catalogs (model lists, pricing, chunking strategies)
├── docs/         User-facing documentation (getting-started, guides, API reference)
├── docker/       Docker Compose files for dev/prod; Nginx reverse-proxy config
├── k8s/          Kubernetes manifests for production cluster deployment
├── scripts/      Shell/Python scripts for setup, migrations, seeding data
```

The separation of `apps/web` and `apps/api` enforces a clear frontend/backend boundary. The `data/` folder acts as a shared source of truth for model catalogs that both the TypeScript frontend and Python backend read — avoiding duplication of model metadata.

---

**Q9: Why is the `data/` folder at the monorepo root rather than inside `apps/api/`?**

Placing `data/` at the root allows **both** apps to read from the same catalog files:
- `apps/web` imports `data/chunking-strategies.json` directly in React components to populate dropdowns.
- `apps/api` reads the same files to validate incoming configurations and calculate costs.

If the catalogs lived inside `apps/api/`, the frontend would have to fetch them over HTTP at runtime (adding latency and a network dependency) or duplicate the JSON files (creating a maintenance burden).

---

**Q10: What is the branch strategy described in TASKS.md and why does P0-1 land on `feature/p0-monorepo-skeleton`?**

The strategy follows **git flow**:
- `main` — production-ready releases only, protected
- `develop` — integration branch; all feature branches merge here first via PR
- `feature/<phase-id>-<slug>` — one focused branch per task

P0-1 lands on `feature/p0-monorepo-skeleton` because it is the very first task — it creates the directory skeleton that all subsequent tasks depend on. Once merged to `develop`, P0-2 through P0-5 can begin in parallel because they all only depend on P0-1.

This structure ensures that every task is reviewable in isolation, CI can gate on a per-branch basis, and rollbacks are surgical (revert one branch's changes, not the entire codebase).

---

**Q11: What is the difference between `docker-compose.yml`, `docker-compose.dev.yml`, and `docker-compose.prod.yml`? (Preview for P0-2)**

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Base configuration — service definitions, ports, networks |
| `docker-compose.dev.yml` | Override for development — bind-mounts source code for hot-reload, exposes debug ports |
| `docker-compose.prod.yml` | Override for production — removes dev mounts, adds resource limits, uses built images |

Docker Compose merges files when you pass multiple `-f` flags: `docker compose -f docker-compose.yml -f docker-compose.dev.yml up`. The base file defines the *what*, overrides define *how* per environment.

---

**Q12: If you had to add a new service (e.g., an observability tool) to this monorepo structure, where would each piece go?**

| Artefact | Location | Reason |
|----------|----------|--------|
| Docker service definition | `docker/docker-compose.yml` | Centralised service config |
| Kubernetes deployment | `k8s/<service>-deployment.yaml` | Namespace-scoped manifest |
| Environment variables | `.env.example` | Documents required config |
| Python client wrapper | `apps/api/app/utils/<service>_client.py` | Utility layer, not business logic |
| Frontend dashboard page | `apps/web/src/app/<service>/page.tsx` | App Router page |

The structure scales without friction because each concern has a designated home — no one has to guess where new files belong.

---

## P0-2 · Docker Compose Development Environment

### Docker Compose Architecture

**Q13: Why does RAG Studio use Docker Compose instead of running services directly on the host?**

Docker Compose gives every developer an identical, reproducible environment regardless of their OS. The eight services (web, api, worker, db, redis, vector-db, mlflow, minio) each have specific version requirements and inter-service dependencies. Running them natively would require each developer to install and configure all eight tools, manage port conflicts, and handle OS-specific quirks. With Compose, a single `docker compose up -d` command starts everything in an isolated network with the correct versions and wiring.

---

**Q14: Explain the three-file Compose strategy (`docker-compose.yml`, `.dev.yml`, `.prod.yml`).**

The three-file pattern follows the **Compose override** mechanism: Compose merges multiple files top-to-bottom, with later files overriding earlier ones.

| File | Role | What it controls |
|------|------|-----------------|
| `docker-compose.yml` | Base | Service names, images, ports, volumes, networks, health checks — everything that is environment-agnostic |
| `docker-compose.dev.yml` | Dev override | Bind-mounts source code for hot-reload, exposes debug ports, sets `NODE_ENV=development` |
| `docker-compose.prod.yml` | Prod override | Switches to pre-built GHCR images, adds CPU/memory limits, removes dev mounts, locks down internal ports |

Usage:
```bash
# Development
docker compose -f docker/docker-compose.yml -f docker/docker-compose.dev.yml up -d

# Production
docker compose -f docker/docker-compose.yml -f docker/docker-compose.prod.yml up -d
```

This avoids maintaining three separate full-length files while keeping each environment's concerns isolated.

---

**Q15: What does `depends_on` with `condition: service_healthy` do, and why is it important here?**

`depends_on` without a condition only waits for the dependent container to *start*, not to be *ready*. Databases and caches take several seconds after their container starts before they accept connections. Using `condition: service_healthy` makes Compose wait until the target service's `healthcheck` command returns exit code 0 before starting the dependent service.

For RAG Studio this matters because:
- `api` depends on `db` (healthy) — otherwise FastAPI will fail to connect to PostgreSQL on startup.
- `worker` depends on `redis` (healthy) — Celery can't connect to its broker until Redis is ready.
- `web` depends on `api` (healthy) — the frontend startup sequence hits the backend to verify connectivity.

Without health-gated dependencies, services start in parallel and race conditions cause cryptic startup errors.

---

**Q16: Walk through each health check and explain what it tests.**

| Service | Health check command | What it verifies |
|---------|---------------------|-----------------|
| `web` | `wget --spider http://localhost:3000` | Next.js HTTP server is accepting connections |
| `api` | `curl -f http://localhost:8000/health` | FastAPI app is up and the `/health` route returns 2xx |
| `worker` | `celery inspect ping` | Celery worker is registered with the broker and responding to control commands |
| `db` | `pg_isready -U raguser -d ragstudio` | PostgreSQL is accepting connections on the correct database |
| `redis` | `redis-cli ping` | Redis server responds with PONG |
| `vector-db` | `wget --spider http://localhost:6333/healthz` | Qdrant's built-in health endpoint returns 200 |
| `mlflow` | `curl -f http://localhost:5000/health` | MLflow tracking server is serving |
| `minio` | `mc ready local` | MinIO storage is available (uses MinIO Client built into the image) |
| `nginx` | `wget --spider http://localhost/health` | Nginx is running and the stub health route returns 200 |

Each check has `start_period` (grace time before failures count), `interval` (how often to check), `timeout` (per-check deadline), and `retries` (how many failures before marking unhealthy).

---

**Q17: What are named volumes and why are they used instead of bind-mounts for stateful services?**

A **named volume** (`postgres-data`, `redis-data`, etc.) is managed by Docker — Docker creates and owns the directory. A **bind-mount** maps a specific host path into the container.

For stateful services (databases, vector stores, object storage) named volumes are preferred because:
- Docker manages the storage location, so paths work cross-platform (Windows, macOS, Linux).
- Named volumes survive container restarts and recreations without data loss.
- They are explicitly declared and can be inspected with `docker volume ls`.
- Docker can back them up with `docker run --volumes-from`.

Bind-mounts are used for source code in dev (hot-reload) where the developer needs direct filesystem access.

---

**Q18: Why does the dev override use an anonymous volume for `node_modules`?**

```yaml
volumes:
  - ../apps/web:/app
  - /app/node_modules   # anonymous volume
```

When `../apps/web` is bind-mounted into `/app`, it overlays the entire directory including `node_modules`. On macOS and Windows, host `node_modules` contains platform-specific binaries compiled for the host OS, which are incompatible with the Linux container. The anonymous volume `/app/node_modules` shadows the bind-mounted `node_modules` with the container's own (correct) version, while still getting the bind-mount for all other source files.

---

**Q19: Explain the Nginx reverse proxy routing rules.**

Nginx acts as the single entry point on port 80, routing to two upstreams:

| Request pattern | Upstream | Special handling |
|-----------------|----------|-----------------|
| `/api/*` | `api:8000` (FastAPI) | Rate-limited to 30 req/s; 300s read timeout for long-running requests |
| `/api/autopilot/build/*/stream` | `api:8000` | `proxy_buffering off` + `proxy_cache off` for Server-Sent Events; 3600s timeout |
| `/_next/static/*` | `web:3000` | Long-lived cache headers (`immutable`, 1 year) |
| `/*` (catch-all) | `web:3000` (Next.js) | WebSocket `Upgrade` header forwarded for HMR in dev |

The SSE route deserves special attention: without `proxy_buffering off`, Nginx would buffer the entire SSE response before forwarding it to the client, breaking the real-time agent activity feed.

---

**Q20: What is the `x-logging` YAML anchor and how does it work in Compose?**

```yaml
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

`x-logging` is a Compose **extension field** (keys prefixed `x-` are ignored by Compose itself but can be referenced via YAML anchors). The `&default-logging` defines a YAML anchor; each service then uses `logging: *default-logging` to merge in the same block — a **YAML alias**.

This avoids repeating the same logging configuration across all nine services. The `json-file` driver with `max-size: 10m` and `max-file: 3` caps each service's logs at 30 MB total, preventing disk exhaustion on long-running containers.

---

**Q21: How does the production override switch from locally-built images to GHCR images?**

```yaml
# docker-compose.prod.yml
web:
  image: ghcr.io/${GITHUB_REPOSITORY}/rag-studio-web:${IMAGE_TAG:-latest}
  build: !reset null
```

- `image:` points to the GitHub Container Registry URL, which CI/CD (P0-3) will build and push.
- `build: !reset null` uses the Compose v2 `!reset` YAML tag to explicitly *remove* the `build` key inherited from the base file. Without this, Compose would still try to build locally even when `image:` is set.
- `${IMAGE_TAG:-latest}` allows pinning a specific version tag (e.g., `v1.0.0`) via environment variable, defaulting to `latest`.

---

**Q22: The Celery worker shares the same Docker image as the API. Why is this a good design?**

The Celery worker runs tasks defined in `apps/api/app/worker/tasks.py`, which import from the same `app` package as the FastAPI server. Using the same image means:
- **No code divergence** — the worker always runs the same version of the agent and service code as the API.
- **Simpler CI/CD** — one image build covers both services.
- **Easier debugging** — the worker behaves identically to the API's environment.

The only difference is the entrypoint command: the API runs `uvicorn`, the worker runs `celery worker`. Both are passed as Docker `command:` overrides.

---

**Q23: What networking strategy does the Compose setup use, and why is the subnet explicitly defined?**

All services share a single bridge network `rag-network`. The subnet `172.20.0.0/16` is explicitly defined to:
1. Avoid conflicts with common default Docker subnets (`172.17.0.0/16`).
2. Make the internal addresses predictable if needed for firewall rules or tooling.

Services communicate by Docker DNS name (e.g., `api` resolves to the API container's IP). Only ports explicitly listed under `ports:` are exposed to the host — all inter-service traffic stays internal to `rag-network`, following the principle of least privilege.

---

## P0-3 · CI/CD Pipelines

### GitHub Actions Architecture

**Q24: What is the purpose of having three separate workflow files (`ci.yml`, `cd.yml`, `tests.yml`) rather than one?**

Separating them by concern gives three benefits:

| Concern | File | Trigger |
|---------|------|---------|
| Fast feedback on every commit | `ci.yml` | Push to `develop`, PRs |
| Ship to production | `cd.yml` | Push to `main` only |
| Expensive long-running tests | `tests.yml` | Manual + PRs |

A monolithic workflow runs everything on every event, wasting minutes on Docker builds for a doc-only change. Separation means developers get lint/type/unit feedback in ~3 minutes without waiting for a 10-minute Docker build + E2E suite.

---

**Q25: Explain the `concurrency` block used in `ci.yml` and why `cancel-in-progress: true` is appropriate there but `cancel-in-progress: false` is used in `cd.yml`.**

```yaml
# ci.yml — cancel older run for same branch
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

# cd.yml — queue, never cancel
concurrency:
  group: cd-production
  cancel-in-progress: false
```

For **CI**: if you push three commits rapidly, only the latest one matters. Cancelling the older runs saves runner minutes and gives faster feedback on the most recent code.

For **CD**: cancelling a deployment mid-run could leave production in a partially-updated state (e.g., new API image deployed but old web image still running). Queuing ensures each deployment completes fully before the next starts.

---

**Q26: How does the CI workflow achieve parallel execution, and what is the `ci-success` job for?**

The four jobs `frontend-lint`, `frontend-tests`, `backend-lint`, `backend-tests` run **in parallel** — GitHub Actions starts all jobs with no `needs:` dependency simultaneously. `frontend-tests` depends on `frontend-lint` (to avoid running tests against unlinted code), and `backend-tests` depends on `backend-lint`, but frontend and backend pipelines are independent.

The `ci-success` job is a **required status check gate**. Branch protection rules can only require a *single* job name. Rather than listing four jobs in the protection rule (and updating it whenever a job is added/removed), we add one synthetic job that fails if any upstream job failed. GitHub branch protection is configured to require only `"CI — All checks passed"`.

---

**Q27: Why is the `backend-tests` job given a Redis service container but not a PostgreSQL or Qdrant container?**

Unit tests use **SQLite with aiosqlite** (`sqlite+aiosqlite:///./test.db`) to avoid requiring a real PostgreSQL server. SQLite runs in-process with no Docker overhead. Redis is still provided as a service because Celery and the embedding cache interact directly with Redis and cannot be easily mocked without changing the production code paths.

Qdrant is not provided for unit tests — any code that touches Qdrant in unit tests is mocked with `pytest-mock` or `unittest.mock`. Only integration tests (in `tests.yml`) run with a real Qdrant service container.

---

**Q28: Walk through the CD workflow's image tagging strategy.**

```yaml
tags: |
  type=ref,event=branch         # main
  type=semver,pattern={{version}} # v1.2.3
  type=semver,pattern={{major}}.{{minor}} # v1.2
  type=sha,prefix=sha-,format=short  # sha-abc1234
  type=raw,value=latest,enable={{is_default_branch}}  # latest (main only)
```

This uses `docker/metadata-action` to generate multiple tags from a single push:

| Tag | Example | Use case |
|-----|---------|---------|
| Branch name | `main` | Rollback to any branch tip |
| Full semver | `v1.2.3` | Pin exact version in prod |
| Major.minor | `v1.2` | Auto-get patch updates |
| Short SHA | `sha-abc1234` | Immutable, traceable to exact commit |
| `latest` | `latest` | Convenience for `docker pull` |

The SHA tag is the most important for production safety — it uniquely identifies the exact commit deployed, enabling precise rollbacks.

---

**Q29: What is Docker layer caching (`cache-from: type=gha`) and why does it matter for CI build times?**

```yaml
cache-from: type=gha,scope=web
cache-to: type=gha,mode=max,scope=web
```

GitHub Actions (GHA) has a built-in cache store. `type=gha` tells Buildx to read previously-cached Docker layers from the GHA cache before rebuilding. `mode=max` caches all intermediate layers (not just the final image), maximising cache hits.

Without caching, a Python `pip install` step that hasn't changed runs every time — potentially taking 2–4 minutes. With layer caching, if `requirements.txt` hasn't changed, that layer is restored from cache in seconds. For a project with 40+ Python dependencies, this can cut build time from 8 minutes to under 2 minutes.

The `scope` parameter (`scope=web`, `scope=api`) keeps caches separate for the two images.

---

**Q30: How does the CD workflow gate production deployments with a manual approval?**

```yaml
deploy:
  environment: production
```

Referencing a GitHub **Environment** named `production` enables environment-level protection rules. In GitHub Settings → Environments → production you can configure:
- **Required reviewers** — specific users/teams must approve before the deploy job runs
- **Wait timer** — delay deployment by N minutes (useful for canary smoke tests)
- **Deployment branches** — only `main` can deploy to `production`

The job is queued and paused until an approver clicks "Approve" in the GitHub UI. This prevents accidental deploys and creates an audit trail of who approved each production release.

---

**Q31: The `tests.yml` workflow uses `workflow_dispatch` with a `test-suite` input. What does this enable?**

`workflow_dispatch` adds a "Run workflow" button in the GitHub Actions UI. The `test-suite` input (`all` / `integration` / `e2e`) lets developers selectively run expensive test suites:

- A backend developer fixing a retrieval bug only needs `integration` — no need to spin up Playwright browsers.
- A frontend developer iterating on the build progress UI only needs `e2e`.
- Before merging to `main` the full `all` suite is run.

Without manual dispatch, integration and E2E tests would either run on every commit (slow, expensive) or never run automatically (risky). The dispatch trigger balances both concerns.

---

**Q32: Explain how branch protection rules connect to the CI workflows.**

GitHub branch protection rules have a "Required status checks" field that accepts job names from Actions workflows. For RAG Studio:

1. `ci.yml` runs on every PR to `develop` or `main` and produces a job called `"CI — All checks passed"`.
2. Branch protection for `develop` requires this job to pass.
3. The PR merge button is greyed out until CI passes — **you literally cannot merge without green CI**.

This creates a hard gate: no human can bypass the lint/test suite (not even admins, if "Do not allow bypassing the above settings" is checked). The `ci-success` summary job is what branch protection checks, so adding new CI jobs doesn't require updating protection rules.

---

**Q33: Why does the CI workflow set `OPENAI_API_KEY: "sk-test-placeholder"` rather than using a real key?**

Unit tests must not make real API calls because:
1. **Cost** — embedding 10,000 test chunks would cost real money per CI run.
2. **Determinism** — real LLM responses are non-deterministic; tests would be flaky.
3. **Speed** — network latency adds seconds per test.
4. **Safety** — real keys in CI risk leaking via logs or forked-repo PRs.

Unit tests mock the LLM/embedding clients with `unittest.mock.patch` or `pytest-mock`. The placeholder key satisfies Pydantic validation (the `Settings` class checks that the key is set) without triggering any real API calls. Integration tests in `tests.yml` use the same placeholder because the mock infrastructure is preserved.

---

**Q34: What is the difference between a GitHub Actions `secret` and a `var` (variable), and when should each be used?**

| | `secrets.*` | `vars.*` |
|--|-------------|----------|
| Stored encrypted | Yes | No |
| Visible in logs | Never masked, but redacted | Visible |
| Accessible in fork PRs | No (security) | Yes |
| Use for | API keys, passwords, tokens | Non-sensitive config (URLs, flags) |

In `cd.yml`:
```yaml
build-args: |
  NEXT_PUBLIC_API_URL=${{ vars.NEXT_PUBLIC_API_URL }}
```

`NEXT_PUBLIC_API_URL` is a public URL — not a secret — so it uses `vars`. `GITHUB_TOKEN` (used for GHCR login) is automatically injected by Actions and doesn't need to be manually created. `OPENAI_API_KEY` would be a `secret` in a real deployment.

---

**Q35: How does the `tests.yml` E2E job coordinate running both the API and frontend servers alongside Playwright?**

The job:
1. Starts stateful services (db, redis, vector-db) via Docker Compose as background containers.
2. Installs Python dependencies and runs `uvicorn` in the background (`&`), saving the PID.
3. Builds Next.js with `npm run build` then starts `npm start` in the background.
4. Uses `npx wait-on` to poll each server's health endpoint before proceeding.
5. Runs Playwright tests against `http://localhost:3000`.
6. In the `always()` cleanup step, kills both server PIDs and tears down Docker containers.

`wait-on` is critical — without it Playwright would start before the server is ready and all tests would fail with connection refused errors.

---

## P0-4 · Backend Project Scaffold

### FastAPI Application Design

**Q36: Why is `create_app()` used as a factory function instead of creating the `FastAPI` instance at module level?**

```python
def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(...)
    ...
    return app

app = create_app()
```

The factory pattern provides three benefits:

1. **Testability** — tests can call `create_app()` with different settings (e.g., `APP_ENV=test`) without relying on global module state.
2. **Lifespan isolation** — each call creates a fresh app instance with its own lifespan context, so test suites can spin up/down cleanly.
3. **Configuration injection** — settings are read inside the factory, so environment overrides set before `create_app()` are respected.

Module-level instantiation (`app = FastAPI()`) runs at import time, before any test can set environment variables.

---

**Q37: Explain the `lifespan` context manager pattern in FastAPI. Why is it preferred over `on_event("startup")`?**

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup code
    yield
    # shutdown code
```

`on_event("startup")` and `on_event("shutdown")` are **deprecated** in FastAPI 0.93+. The lifespan pattern:
- Uses standard Python `asynccontextmanager` — no FastAPI-specific decorator needed.
- Pairs startup and shutdown in one function, making the resource lifecycle explicit.
- Works with Python's `with` / `async with` semantics that developers already know.
- Allows initialising resources (DB connections, caches) before the first request and guaranteeing cleanup on shutdown.

For RAG Studio, the lifespan sets up structured logging and will eventually initialise DB connection pools and close them cleanly on SIGTERM.

---

**Q38: How does `pydantic-settings` work, and why use it over `os.getenv()`?**

`pydantic-settings` provides a `BaseSettings` class that:
1. Reads values from environment variables (or a `.env` file).
2. Validates and coerces types automatically (`str → int`, `str → list[str]`, etc.).
3. Documents all required settings in one place with type annotations.
4. Raises a clear `ValidationError` with field names if required settings are missing — far better than a cryptic `AttributeError` at runtime.

```python
class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://..."
    cors_origins: list[str] = ["http://localhost:3000"]
```

`os.getenv("CORS_ORIGINS")` returns a raw string — you'd need to manually parse the JSON/CSV, handle `None`, and validate types yourself. `pydantic-settings` does all this declaratively.

---

**Q39: What does `@lru_cache` on `get_settings()` do, and how does it interact with testing?**

```python
@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`lru_cache` with no arguments memoises the function forever — the first call constructs a `Settings()` instance, subsequent calls return the cached object. This means:
- Settings are loaded from the environment only once per process (efficient).
- All parts of the codebase share the same `Settings` object (consistent).

In tests, environment variables are set before the first call, so the cached settings reflect the test environment. If a test changes an env var mid-run and needs fresh settings, it calls `get_settings.cache_clear()` to force a reload. The `conftest.py` does this at session start and end.

---

**Q40: Why does `dependencies.py` use module-level singletons (`_redis_client`, `_qdrant_client`) instead of creating a new connection per request?**

Creating a new Redis or Qdrant connection per request would be expensive:
- **Redis**: connection handshake + AUTH on every request adds ~1–5ms latency and exhausts file descriptors under load.
- **Qdrant**: HTTP client setup has overhead; connection pooling is far more efficient.

The module-level singleton is created once on first use and reused. The `async with factory() as session:` pattern for SQLAlchemy is different — sessions are lightweight wrappers around a shared connection pool; a new session per request is the correct SQLAlchemy pattern.

---

**Q41: Walk through the database session dependency chain in `dependencies.py`.**

```
get_settings()           → Settings object (cached)
  ↓
get_session_factory()    → async_sessionmaker (module-level singleton)
  ↓
get_db_session()         → AsyncSession (new per request)
  ↓
route handler            → receives typed AsyncSession
```

The chain:
1. `get_session_factory` depends on `get_settings` to get `database_url`. It creates the engine + session factory once.
2. `get_db_session` calls `factory()` to open a new session, yields it to the handler, then commits or rolls back.
3. The `try/except` around `yield` guarantees rollback on any exception — the session is never left in an uncommitted state.
4. `expire_on_commit=False` means ORM objects remain accessible after the session is committed (important for async where the session closes before the response is serialised).

---

**Q42: What does `Annotated[AsyncSession, Depends(get_db_session)]` do, and why define type aliases for it?**

```python
DbSession = Annotated[AsyncSession, Depends(get_db_session)]

# Route usage:
async def my_route(db: DbSession): ...
```

`Annotated[T, metadata]` attaches metadata to a type without changing it. FastAPI reads the `Depends(...)` metadata and resolves the dependency when the route is called. The result is typed as `AsyncSession` for IDE autocompletion.

The type alias `DbSession` means:
- Route handlers write `db: DbSession` instead of `db: Annotated[AsyncSession, Depends(get_db_session)]` — much cleaner.
- If the dependency changes (e.g., adding a middleware layer), only the alias needs updating, not every route handler.

---

**Q43: Why does the Dockerfile use three stages (`builder`, `development`, `runtime`) rather than two?**

| Stage | Purpose | Key contents |
|-------|---------|-------------|
| `builder` | Install deps with build tools | `build-essential`, `libpq-dev`, all `requirements.txt` packages |
| `development` | Hot-reload dev server | Copies from builder + adds `requirements-dev.txt`; source bind-mounted |
| `runtime` | Minimal production image | Copies only installed packages from builder; no build tools; non-root user |

The `builder` stage requires C compiler tools (`build-essential`) for packages like `asyncpg`. Those tools add ~300MB to the image. The `runtime` stage copies only the compiled Python packages (not the tools), resulting in a significantly smaller production image (~150MB vs ~450MB).

The `development` stage adds dev tools (`ruff`, `mypy`, `pytest`) that are never in the production image. This is built only locally — CI/CD uses the `runtime` target.

---

**Q44: Why does the runtime Dockerfile stage create a non-root user (`appuser`)?**

Running containers as root is a security risk: if the application is compromised, the attacker has root access to the container filesystem. On misconfigured hosts, container root can map to host root.

```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
...
USER appuser
```

The `appuser` has no shell, no home directory (`-r` = system account), and no password. It can read the application files (via `COPY --chown=appuser:appuser`) but cannot write to system directories, install software, or escalate privileges. This follows the principle of least privilege for container security.

---

**Q45: What is `pool_pre_ping=True` in the SQLAlchemy engine, and why is it important in a containerised environment?**

```python
create_async_engine(database_url, pool_pre_ping=True, ...)
```

Connection pools keep TCP connections open. In Docker/Kubernetes, the database container may restart, causing existing pool connections to become "stale" (the TCP socket exists on the client side but the server has closed it). Without `pool_pre_ping`, the next query on a stale connection raises a cryptic `OperationalError`.

With `pool_pre_ping=True`, SQLAlchemy sends a lightweight `SELECT 1` before each borrowed connection. If the ping fails, the connection is discarded and a fresh one is created transparently. The cost is one extra round trip per connection checkout — negligible compared to preventing connection errors.

---

**Q46: Why is docs_url set to `None` in production?**

```python
docs_url="/docs" if not settings.is_production else None,
```

FastAPI's `/docs` (Swagger UI) and `/redoc` endpoints expose:
- All API routes, parameters, and response schemas.
- Authentication mechanisms and token formats.
- Internal service details useful for reconnaissance.

In production, this information should only be accessible to authenticated developers, not the public internet. Setting `docs_url=None` disables the routes entirely. API documentation for internal use is served separately (e.g., behind an auth gateway or generated statically and published to an internal wiki).

---

**Q47: Explain the request logging middleware pattern used in `main.py`.**

```python
@app.middleware("http")
async def request_logging_middleware(request: Request, call_next):
    start = time.perf_counter()
    request_id = request.headers.get("X-Request-ID", "")
    structlog.contextvars.bind_contextvars(request_id=request_id)
    response = await call_next(request)
    duration_ms = round((time.perf_counter() - start) * 1000, 2)
    logger.info("request", method=..., path=..., status=..., duration_ms=...)
    structlog.contextvars.clear_contextvars()
    return response
```

Key design choices:
- `structlog.contextvars.bind_contextvars(request_id=...)` attaches the request ID to every log line emitted during that request — enabling log correlation across services.
- `time.perf_counter()` is used (not `time.time()`) because it measures elapsed time without being affected by system clock adjustments.
- `clear_contextvars()` in the final step prevents request ID leaking into the next request handled by the same coroutine (important with async).
- `call_next(request)` passes control to the route handler; timing wraps the entire handler duration including DB queries.

---

## P0-5 · Frontend Project Scaffold

### Next.js 14 App Router Architecture

**Q48: Why did you choose Next.js 14 App Router over Pages Router for RAG Studio?**

The App Router (introduced in Next.js 13, stabilised in 14) offers several advantages for RAG Studio's use case:

| Feature | App Router | Pages Router |
|---------|-----------|-------------|
| Server Components | ✅ Default | ❌ Not supported |
| Streaming / Suspense | ✅ Built-in | ❌ Manual |
| Nested layouts | ✅ Per-segment `layout.tsx` | ❌ Single `_app.tsx` |
| Route groups | ✅ `(group)` directories | ❌ Not available |
| Colocation | ✅ Components alongside routes | ❌ Must be in `/components` |

For RAG Studio specifically:
- **Designer Mode** benefits from nested layouts — the stage navigator is rendered by `apps/web/src/app/designer/layout.tsx` and shared across all step sub-routes without re-rendering.
- **Autopilot build progress** uses React Suspense streaming to show partial UI as agent data arrives.
- Server Components reduce the client-side bundle by keeping data-fetching logic on the server.

---

**Q49: What is the `src/` directory convention in Next.js and why use it?**

Next.js projects can be structured with or without a `src/` directory. With `src/`:

```
apps/web/
├── src/
│   ├── app/         # Route segments (App Router)
│   ├── components/  # UI components
│   ├── lib/         # Utilities, API client
│   ├── store/       # Zustand stores
│   └── types/       # TypeScript types
├── public/          # Static assets (outside src/)
├── package.json
└── next.config.js
```

Benefits:
- Clearly separates application code from configuration files (`next.config.js`, `tailwind.config.ts`, `tsconfig.json`).
- Prevents accidental name collisions between top-level config files and route segments.
- Mirrors the structure of established monorepo conventions (e.g., `apps/api/app/`).
- The `@/*` TypeScript path alias maps to `./src/*` in `tsconfig.json`, keeping imports clean.

---

**Q50: Explain the shadcn/ui component model. How is it different from a traditional component library?**

Traditional component libraries (e.g., MUI, Ant Design) are NPM packages — you install them and import pre-built components as black boxes. Customisation requires overriding CSS or using theme tokens.

shadcn/ui takes a fundamentally different approach:

1. **Code ownership** — `npx shadcn@latest add button` copies the component source into your project under `src/components/ui/`. You own the code.
2. **Radix UI primitives** — Each component wraps a Radix UI primitive (accessible, unstyled, keyboard-navigable) with Tailwind CSS classes.
3. **`cn()` utility** — `class-variance-authority` (CVA) handles variant logic; `tailwind-merge` resolves conflicting Tailwind classes. The `cn()` helper combines both.
4. **`components.json`** — Configures the registry (style, Tailwind config path, CSS file, path aliases) so `npx shadcn@latest add` places files in the right location.

**Trade-off**: More boilerplate files per component, but complete control over styling, behaviour, and accessibility.

---

**Q51: What does `components.json` configure for shadcn/ui and why is it needed?**

```json
{
  "style": "default",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "src/app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils"
  }
}
```

- **`style`**: `"default"` uses the standard shadcn design tokens. `"new-york"` uses a more compact variant.
- **`rsc: true`**: Marks components as React Server Component-compatible (no `"use client"` unless interactive).
- **`cssVariables: true`**: Uses CSS custom properties (`--primary`, `--background`, etc.) instead of hard-coded Tailwind colours — enables runtime theme switching.
- **`aliases`**: Tells the CLI where to place new component files and which import alias to use (`@/components/ui/button.tsx` instead of a relative path).
- **`baseColor: "neutral"`**: The neutral grey scale is used for backgrounds, borders, and muted text.

Without `components.json`, every `npx shadcn@latest add` command would prompt for these settings interactively.

---

### State Management

**Q52: Why Zustand over Redux or React Context for RAG Studio's state management?**

| Criterion | Zustand | Redux Toolkit | React Context |
|-----------|---------|---------------|---------------|
| Bundle size | ~1KB | ~10KB | 0 (built-in) |
| Boilerplate | Minimal | Moderate | Minimal |
| DevTools | ✅ Redux DevTools compat | ✅ Native | ❌ None |
| Server Component compat | ✅ | ✅ | ❌ (Client only) |
| Selectors / memoisation | ✅ Automatic | Requires reselect | Manual useMemo |
| Persistence middleware | ✅ Built-in | Via redux-persist | Manual |

For RAG Studio:
- **`designerStore`** holds `PipelineConfiguration` — a deeply nested object that changes frequently as users configure each stage. Zustand's `immer`-style updates handle this cleanly.
- **`autopilotStore`** needs real-time updates from SSE events. Zustand's `set()` is called directly from the event handler — no action dispatching overhead.
- **`persist` middleware** (via `zustand/middleware/persist`) serialises the Designer config to `localStorage` so users can close and reopen the browser without losing their work.

Redux would be overkill for a single-app frontend with no complex action history requirements.

---

**Q53: How does Zustand's `persist` middleware work and what are its caveats?**

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useDesignerStore = create(
  persist(
    (set) => ({ config: DEFAULT_CONFIG, setConfig: (c) => set({ config: c }) }),
    {
      name: 'designer-config',  // localStorage key
      partialize: (state) => ({ config: state.config }),  // only persist config, not actions
    }
  )
);
```

On first render, `persist` rehydrates state from `localStorage` by parsing the stored JSON. On every `set()` call, it serialises the updated state back to storage.

**Caveats**:
1. **SSR hydration mismatch** — The server renders with the default state; the client rehydrates from localStorage, causing a flash. Fix: use `skipHydration: true` and manually call `useDesignerStore.persist.rehydrate()` in a `useEffect`.
2. **Stale state** — If the schema changes between deployments, old persisted data may fail to parse. Fix: add a `version` field and a `migrate` function.
3. **Storage quota** — `localStorage` is limited to ~5MB. Large pipeline configs could hit this. Fix: use `sessionStorage` or compress the JSON.

---

### Typed Fetch & API Client

**Q54: Walk through the `api-client.ts` design. Why a custom wrapper instead of using `fetch` directly or a library like `axios`?**

```typescript
async function request<TResponse, TBody = unknown>(
  path: string,
  options: RequestOptions<TBody> = {}
): Promise<TResponse>
```

The wrapper provides:
1. **Base URL injection** — All routes are relative (`/api/health`) with `API_BASE` prepended automatically. Changing the API URL only requires updating one env variable.
2. **Type safety** — `TResponse` generic lets callers specify the expected response shape: `apiClient.get<HealthResponse>('/health')` gives fully typed autocomplete.
3. **Error normalisation** — A non-2xx response throws an `ApiError` with `status`, `statusText`, and parsed body. Callers catch a single error type instead of checking `response.ok` everywhere.
4. **AbortSignal support** — Every method accepts an optional `signal` for request cancellation (used with React Query's `queryFn` to cancel in-flight requests on component unmount).
5. **No circular imports** — The client is a plain module with no React dependencies, making it safe to use in Server Components, route handlers, and Zustand actions.

`axios` would work but adds ~14KB to the bundle. The custom wrapper achieves the same goals with ~50 lines of code.

---

**Q55: What is the purpose of the `cn()` utility and how does it prevent Tailwind class conflicts?**

```typescript
import { clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**`clsx`** conditionally joins class names:
```typescript
clsx('p-4', isActive && 'bg-primary-500', { 'opacity-50': isDisabled })
// → "p-4 bg-primary-500" (if active) or "p-4 opacity-50" (if disabled)
```

**`tailwind-merge`** resolves conflicts between Tailwind utility classes. Without it:
```typescript
// Base: "p-4 bg-white"  Override: "bg-primary-500"
clsx('p-4 bg-white', 'bg-primary-500')
// → "p-4 bg-white bg-primary-500"  ← BOTH backgrounds applied; last wins but unpredictably
```

With `twMerge`:
```typescript
twMerge('p-4 bg-white', 'bg-primary-500')
// → "p-4 bg-primary-500"  ← bg-white is removed; correct override
```

For shadcn/ui, every component accepts a `className` prop. `cn()` merges the default styles with user overrides cleanly. This is the standard pattern across the React/Tailwind ecosystem.

---

### Multi-Stage Docker Build

**Q56: Describe the three stages of the Next.js Dockerfile and what each optimises.**

```dockerfile
FROM node:20-alpine AS deps       # Stage 1
FROM node:20-alpine AS builder    # Stage 2
FROM node:20-alpine AS runtime    # Stage 3
```

| Stage | Purpose | Key operations |
|-------|---------|----------------|
| `deps` | Reproducible dependency install | `npm ci --frozen-lockfile` (exact lock file) |
| `builder` | Next.js production build | `npm run build` → produces `.next/standalone/` |
| `runtime` | Minimal serving image | Copies only standalone output; `nextjs` non-root user; HEALTHCHECK |

**Why `output: 'standalone'` in `next.config.js`?**

Next.js `standalone` mode traces and bundles only the Node.js files needed to run the server — it excludes `node_modules/` (the ~300MB dev toolchain). The resulting `.next/standalone/` is a self-contained directory that runs with `node server.js`, making the final Docker image ~80MB instead of ~400MB.

**Size comparison:**
- Naïve `FROM node:20` with all `node_modules`: ~900MB
- Alpine + standalone output: ~120MB
- Difference: ~87% reduction

---

**Q57: What does the `next.config.js` rewrite rule accomplish and why is it needed in production?**

```javascript
async rewrites() {
  return [{
    source: '/api/:path*',
    destination: `${process.env.NEXT_PUBLIC_API_URL}/api/:path*`,
  }];
}
```

In development, the browser and Next.js dev server run on `localhost:3000`. The FastAPI backend runs on `localhost:8000`. A direct cross-origin request from the browser to `localhost:8000` violates CORS unless FastAPI allows `localhost:3000` explicitly.

The rewrite rule proxies `/api/*` requests through the Next.js server to the backend. From the browser's perspective, the API is on the same origin — no CORS issue. In production, Nginx handles this proxying at the infrastructure level, but the rewrite provides an identical developer experience without requiring Nginx locally.

`NEXT_PUBLIC_API_URL` makes the backend URL configurable per environment (local dev, staging, production) without code changes.

---

**Q58: What is `tailwindcss-animate` and how does it integrate with Radix UI components like `accordion`?**

`tailwindcss-animate` is a Tailwind plugin that provides animation utility classes (`animate-accordion-down`, `animate-fade-in`, etc.) generated from `@keyframes` definitions.

In `tailwind.config.ts`:
```typescript
keyframes: {
  'accordion-down': {
    from: { height: '0' },
    to: { height: 'var(--radix-accordion-content-height)' },
  },
},
animation: {
  'accordion-down': 'accordion-down 0.2s ease-out',
},
```

Radix UI's `Accordion.Content` exposes `--radix-accordion-content-height` as a CSS custom property. Tailwind's `animate-accordion-down` class uses this variable to animate from `height: 0` to the content's natural height — a smooth expand/collapse with no JavaScript measurement required.

This pattern (Radix exposes CSS variables → Tailwind animates using them) works for dialogs, dropdowns, and tooltips too, keeping all animation logic in CSS rather than `framer-motion` for simple transitions.

---

## P1-1 · JSON Model Catalogs

### Catalog Architecture

**Q59: Why are the model catalogs stored as JSON files in `data/` rather than in a database or hardcoded in the application?**

The `data/` folder acts as a **shared source of truth** that both sides of the monorepo can read without any network calls:

- `apps/web` imports `data/models/embeddings.json` directly at build time — populating dropdowns and comparison tables with zero API latency.
- `apps/api` reads the same files at startup to validate configuration payloads and compute costs.

If the catalogs lived in a database, the frontend would need an HTTP round-trip on every page load. If they were hardcoded, any model update would require a code change in two places (TypeScript and Python). JSON files give us:
1. **Single source of truth** — update once, both apps see the change.
2. **Zero runtime dependencies** — no DB connection needed to render a model selector.
3. **Versionable** — Git diffs make it obvious when a model's pricing or specs change.
4. **Human-readable** — non-engineers can update model metadata without touching code.

---

**Q60: Walk through the structure of `embeddings.json`. What fields does each model entry contain and why?**

Each embedding model entry has:

| Field | Type | Purpose |
|-------|------|---------|
| `id` | string | Machine-readable key used in `PipelineConfiguration` |
| `name` | string | Display name shown in the UI |
| `provider` | string | Groups models by vendor (openai, cohere, huggingface, etc.) |
| `dimensions` | number | Vector size — used in storage cost calculation |
| `maxTokens` | number | Chunk size upper bound — chunking stage must stay below this |
| `costPer1MTokens` | number | Ingestion cost input for the cost calculator |
| `speed` | enum | Qualitative latency rating for UX filtering |
| `quality` | enum | MTEB-derived qualitative quality rating |
| `tier` | enum | `fast / balanced / advanced` — maps to UI filter tabs |
| `openSource` | boolean | Drives "open-source only" toggle in the embedding selector |
| `mtebScore` | number | Quantitative quality benchmark (MTEB leaderboard) |
| `languageSupport` | array | Used to filter models when multilingual support is needed |
| `selfHosted` | boolean | Marks models that run locally (cost = 0 but infra applies) |

The `mtebScore` deserves attention: it's the industry-standard benchmark for embedding quality, allowing the frontend to render actual comparison bars instead of arbitrary "good/excellent" labels.

---

**Q61: How does the `tier` field in `embeddings.json` map to the UI experience?**

The `tier` field (`fast`, `balanced`, `advanced`) drives the filter tab in `EmbeddingSelector.tsx`:

```
Fast     → all-MiniLM-L6-v2, google-textembedding-gecko
Balanced → text-embedding-3-small, cohere-embed-v3, e5-large-v2, nomic-embed-text
Advanced → text-embedding-3-large, bge-large-en, cohere-embed-multilingual-v3
```

This lets users quickly narrow their selection based on their primary constraint:
- "Fast" tier: prototyping, CPU-only, or latency-critical
- "Advanced" tier: highest accuracy for production or compliance use cases
- "Balanced": the most common production choice

The `costPer1MTokens: 0.0` for HuggingFace models is intentional — they are genuinely free to use, though the UI should clarify that infrastructure costs apply for self-hosting.

---

**Q62: Why does `generation.json` separate `costInput` and `costOutput` instead of a single cost field?**

LLM providers charge differently for input (prompt) tokens vs output (completion) tokens because:
- **Output tokens are compute-bound** — each token requires one autoregressive forward pass.
- **Input tokens can be cached** — KV-cache means re-processing the same context is much cheaper (especially for Anthropic's prompt caching feature).

For RAG specifically, this distinction matters a lot:
- Input tokens = user query + retrieved context (potentially 2000–4000 tokens)
- Output tokens = generated answer (typically 300–500 tokens)

The cost calculator uses both fields separately:
```
costPerQuery = (contextTokens * costInput / 1M) + (outputTokens * costOutput / 1M)
```

GPT-4o at `$2.50 input / $10.00 output` vs `$15.00/$75.00` for Claude Opus means a 100K-query month costs dramatically different amounts — and the split is why.

---

**Q63: Explain the `defaultConfig` structure in `chunking-strategies.json` and how it's used.**

Each strategy's `defaultConfig` contains the recommended starting parameters:

```json
{
  "id": "recursive-character",
  "defaultConfig": {
    "chunkSize": 512,
    "chunkOverlap": 50,
    "separators": ["\n\n", "\n", ". ", " ", ""]
  }
}
```

When a user selects a strategy in `ChunkingConfig.tsx`, the component reads `strategy.defaultConfig` and auto-populates the sliders. This prevents users from starting with invalid configurations (e.g., overlap > chunk size, or a chunk size larger than the embedding model's `maxTokens`).

The `separators` array in `recursive-character` encodes the fallback hierarchy: try splitting on `\n\n` first (paragraphs), fall back to `\n` (lines), then `. ` (sentences), etc. This is exactly LangChain's `RecursiveCharacterTextSplitter` parameter.

---

**Q64: What is the `implementationComplexity` field in `chunking-strategies.json` used for?**

`implementationComplexity` (`low`, `medium`, `high`) serves two purposes:

1. **UI hint** — The `ChunkingConfig` component shows a complexity badge. Beginners are visually guided toward `fixed-size` or `recursive-character` (both `low`), while advanced users can opt into `semantic` or `code-aware` (both `high`).

2. **Autopilot guard** — The `ChunkingOptimizerAgent` uses this field to decide which strategies to test first. It always starts with `low`-complexity strategies (faster to evaluate) and only tries `high`-complexity strategies (like `semantic`, which requires an extra embedding pass) if the initial options don't meet the target metrics.

The complexity levels roughly map to implementation effort:
- `low` = pure string operations, no ML
- `medium` = NLP library or custom parsing required
- `high` = ML model call or AST parsing required

---

**Q65: What is the purpose of the `bestFor` and `notRecommendedFor` arrays in chunking strategies?**

These fields power the **recommendation engine** in the Designer:

```json
"recommendedFor": {
  "documentTypes": ["md", "mdx"],
  "useCases": ["documentation-qa"],
  "notRecommendedFor": ["pdfs", "plain-text"]
}
```

When a user configures their data ingestion (document types selected) before reaching the chunking stage, the `ChunkingConfig` component can:
1. **Pre-filter** strategies — mark `code-aware` as "Not recommended" if no code files are selected.
2. **Highlight** top recommendations — e.g., `markdown-header` surfaces as "Recommended" when `.md` files are in the ingestion config.
3. **Show warning alerts** — "This strategy is not recommended for PDF documents."

This creates a guided, intelligent experience without hardcoding rules in the component — the rules live in the JSON and can be updated without a code change.

---

**Q66: Why does `vector-stores.json` include both `pros`/`cons` AND `features` as separate objects?**

They serve different UI components:

- **`pros`/`cons`** are human-readable strings for the info sidebar shown when a user hovers or clicks a vector store card — educational content explaining the trade-offs in plain language.
- **`features`** is a machine-readable boolean map used for filtering:
  ```json
  "features": { "hybridSearch": true, "sparseVectors": true }
  ```
  If a user selects "Hybrid" retrieval in the retrieval stage, the `VectorStoreSelector` automatically filters out vector stores where `features.hybridSearch === false`, preventing invalid configurations.

Keeping them separate avoids parsing human-readable text for feature detection and avoids generating verbose prose from boolean flags.

---

**Q67: How does `cloud-providers.json` drive the recommendation system for other stages?**

The `ragStudioDefaults` object in each provider entry cascades to subsequent stages:

```json
{
  "id": "aws",
  "ragStudioDefaults": {
    "vectorStore": "opensearch",
    "objectStorage": "s3",
    "deployment": "ecs"
  }
}
```

When a user selects `aws` in the Cloud Provider step:
1. The `VectorStoreSelector` pre-selects `opensearch` and promotes it as "AWS Native".
2. The export generator uses `deployment: "ecs"` to produce AWS ECS-flavored Terraform and Kubernetes manifests.
3. The cost calculator uses AWS-specific pricing from `pricing.json`.

This means the cloud provider selection has a cascading effect through the entire pipeline — the JSON encoding avoids hardcoding these defaults in multiple components.

---

**Q68: Walk through the structure of a template entry in `templates.json` and explain why templates include a full `PipelineConfiguration`.**

Each template contains:

```json
{
  "id": "documentation-qa",
  "name": "Documentation Q&A",
  "complexity": "intermediate",
  "estimatedMonthlyCost": "$30-100",
  "tags": ["technical-docs", "hybrid-search"],
  "config": { /* full PipelineConfiguration */ }
}
```

The `config` field is a **complete, ready-to-use** `PipelineConfiguration` — not a partial diff or a reference. This is intentional:

- **Atomic** — `POST /api/templates/{id}/apply` can create a new pipeline config in one operation without merging or resolving defaults.
- **Testable** — Each template config can be validated against the `PipelineConfigurationSchema` in CI.
- **Designer-compatible** — `useDesignerStore().setConfig(template.config)` immediately loads the template into the Designer without any transformation.
- **Self-documenting** — Reading a template JSON shows the exact recommended configuration, which is valuable for learning.

The trade-off is that updating a field shared across templates (e.g., Qdrant becomes the vector store default) requires updating all template files. In practice, templates change rarely, so this is acceptable.

---

**Q69: Explain the `costCalculatorFormulas` object in `pricing.json`. How does the cost calculator use it?**

The `costCalculatorFormulas` object documents the mathematical formulas used by `apps/api/app/utils/cost_calculator.py` and the frontend's `CostEstimator` component:

```json
"totalCostPerQuery": {
  "formula": "embeddingCostPerQuery + generationCostPerQuery + rerankingCostPerQuery"
}
```

The key formula for `generationCostPerQuery`:
```
contextTokens = avgChunksRetrievedPerQuery × avgChunkSizeTokens + avgInputTokensPerQuery
cost = (contextTokens × inputCostPer1MTokens / 1M) + (avgOutputTokens × outputCostPer1MTokens / 1M)
```

This means for `gpt-4o` with 5 chunks of 512 tokens each and a 300-token user query:
- Input: (5 × 512 + 300) = 2860 tokens → 2860 × $2.50/1M = **$0.00715**
- Output: 300 tokens → 300 × $10.00/1M = **$0.003**
- Total: **$0.01015 per query** = **$1.015 per 1K queries**

The `pricing.json` file serves as both the data source AND the documentation — future developers can trace exactly how a cost figure was derived.

---

**Q70: What is the `benchmarks` object in `pricing.json` for?**

The `benchmarks` block provides four reference points for the `CostEstimator` component's comparison widget:

```json
"budgetOptimized": { "costPer1KQueries": 0.015 },
"balanced":        { "costPer1KQueries": 0.050 },
"highQuality":     { "costPer1KQueries": 0.150 },
"enterprisePremium": { "costPer1KQueries": 0.500 }
```

These are shown as a benchmark bar below the user's estimated cost:
```
Your config: $0.032/1K  ←  sits between balanced and high-quality
Budget: $0.015  |  Average: $0.050  |  Premium: $0.150  |  Enterprise: $0.500
```

Having these values in JSON (not hardcoded in the component) means they can be updated as model pricing changes without a frontend code change. The component that reads them gets the latest numbers automatically.

---

**Q71: How does the data layer (P1-1) act as a contract between the frontend and backend?**

The JSON files in `data/` establish a shared contract that both apps must respect:

- **Model IDs as foreign keys**: `config.stages.embedding.model = "text-embedding-3-small"` is validated against `data/models/embeddings.json` on both sides:
  - Frontend: dropdown only shows IDs from the JSON, so invalid values can't be selected.
  - Backend: `EmbeddingConfigSchema` validates that the model ID exists in the catalog before processing.

- **Pricing accuracy**: The cost calculator in Python (`cost_calculator.py`) reads `data/pricing.json` — the same file the frontend uses. This guarantees the estimate shown in the UI matches what the backend actually charges.

- **Single update surface**: When a new model is released (e.g., `text-embedding-4-mini`), one JSON entry is added. The frontend selector, backend validator, cost calculator, and interview Q&A generation all automatically include it.

This shared-data pattern is one of the key architectural advantages of the monorepo structure established in P0-1.

---

## P1-2 · TypeScript Shared Types

### Type System Architecture

**Q72: Why create a dedicated `types/` directory with multiple files instead of keeping types inline in components or the store?**

Centralising types in `apps/web/src/types/` provides three benefits:

1. **Single source of truth** — `PipelineConfiguration` is defined once. Every component, store, and API client that references a pipeline shape imports from the same declaration. If the shape changes, one edit propagates everywhere and the TypeScript compiler flags every mismatch.
2. **Barrel export via `index.ts`** — consumers write `import type { PipelineConfiguration } from '@/types'` rather than tracking which file holds each type. Refactoring the internal file layout is transparent to importers.
3. **Documentation surface** — a file named `pipeline.ts` containing only types is self-documenting. A new team member can read it end-to-end in five minutes and understand the entire data model without reading component code.

Inline types (defined next to the component that first needed them) drift apart over time: two developers define `CloudProvider` in two components, they diverge, and the compiler can't catch the mismatch.

---

**Q73: What is the difference between `type` aliases and `interface` in TypeScript, and which did you use where in this project?**

| Feature | `type` alias | `interface` |
|---------|-------------|-------------|
| Object shape | ✅ | ✅ |
| Union / intersection types | ✅ | ❌ (can approximate with `extends`) |
| Declaration merging | ❌ | ✅ |
| Extends other types | Limited | ✅ `extends` keyword |
| Tuple / primitive aliases | ✅ | ❌ |

In RAG Studio:
- **`type` is used for union types**: `CloudProvider`, `ChunkingStrategy`, `VectorStoreProvider`, `RetrievalStrategy`, `ModelTier`. These are finite sets of string literals — `interface` cannot express that.
- **`interface` is used for object shapes**: `PipelineConfiguration`, `EmbeddingConfig`, `AutopilotBuild`, etc. Interfaces are preferred for object shapes because they produce cleaner error messages and support `extends` for composition.

The rule of thumb: if it's an enumeration of values, use `type`; if it's an object with properties, use `interface`.

---

**Q74: Explain the `PipelineConfiguration` type and why the `stages` field uses optional properties for some sub-configs.**

```typescript
export interface PipelineStages {
  dataIngestion?: DataIngestionConfig;  // optional
  chunking: ChunkingConfig;             // required
  embedding: EmbeddingConfig;           // required
  vectorStore: VectorStoreConfig;       // required
  retrieval: RetrievalConfig;           // required
  reranking?: RerankingConfig;          // optional
  generation: GenerationConfig;         // required
  routing?: RoutingConfig;              // optional
  memory?: MemoryConfig;                // optional
  evaluation?: EvaluationConfig;        // optional
}
```

The optionality mirrors business rules:
- **Required stages** (`chunking`, `embedding`, `vectorStore`, `retrieval`, `generation`) are always present — a pipeline cannot function without them.
- **Optional stages** (`reranking`, `routing`, `memory`, `evaluation`) are features users can opt into. `reranking` adds precision but costs more; `routing` only makes sense when queries vary in complexity; `evaluation` is useful for monitoring but not always configured upfront.
- `dataIngestion` is optional because templates sometimes don't pre-populate it (the user fills it in at runtime).

If all stages were required, the `DEFAULT_CONFIG` in the Zustand store would need dummy values for every field — which would mislead the UI into showing a "configured" pipeline when it's actually empty.

---

**Q75: Why split the types into four files (`pipeline.ts`, `autopilot.ts`, `models.ts`, `cloud.ts`) rather than one file?**

Each file groups types by domain concern:

| File | Domain | Depends on |
|------|---------|-----------|
| `pipeline.ts` | The RAG pipeline data model — the core DSL | Nothing (no imports) |
| `autopilot.ts` | Build lifecycle and agent state | `pipeline.ts` (reuses `PipelineConfiguration`, `CloudProvider`) |
| `models.ts` | Catalog metadata matching JSON schemas | `pipeline.ts` (reuses `ModelTier`) |
| `cloud.ts` | Cloud provider config | `pipeline.ts` (re-exports `CloudProvider`) |

This separation keeps each file focused and avoids circular dependencies. `pipeline.ts` is deliberately dependency-free — it's the "leaf" in the import graph. If all types were in one file, every import of a single type would force the bundler to parse the entire file.

---

**Q76: How does `CloudProvider` avoid being defined twice across `pipeline.ts` and `cloud.ts`?**

`CloudProvider` is defined **once** in `pipeline.ts` because it is primarily a pipeline-level concept (every `PipelineConfiguration` has a `cloudProvider: CloudProvider`).

`cloud.ts` re-exports it:
```typescript
export type { CloudProvider } from './pipeline';
```

This gives developers importing from the cloud module convenient access without a second definition. The barrel `index.ts` exports `CloudProvider` only from `pipeline.ts` (via `export * from './pipeline'`) and explicitly only re-exports the cloud-specific types from `cloud.ts`:

```typescript
export type { CloudNativeService, CloudProviderConfig, CloudProviderDefaults, CloudPricingTier } from './cloud';
```

This prevents a "duplicate export" error while still making `CloudProvider` importable from `@/types/cloud` for consumers who naturally associate it with cloud configuration.

---

**Q77: What is `export type` and why does `index.ts` use it for cloud types?**

`export type` is a TypeScript 3.8+ feature that instructs the compiler (and transpilers like `esbuild`/`swc`) that this export is a **type-only** export — it does not exist at runtime. This matters for:

1. **Tree-shaking** — bundlers can safely eliminate type-only imports from production bundles with no side effects.
2. **Isolated module compilation** — tools that process files individually (e.g., `ts-node`, Babel) need to know which exports are types so they can safely erase them without runtime errors.

In `index.ts`:
```typescript
export type { CloudNativeService, CloudProviderConfig, ... } from './cloud';
```

Since these are all `interface` declarations (which have no runtime representation), `export type` is the correct and explicit form. The `export * from './pipeline'` form for the main pipeline exports works too because TypeScript detects them as type-only automatically, but being explicit with `export type` for the selective re-exports makes the intent clear.

---

**Q78: Walk through the `AutopilotBuild` type and explain the `stages` field design.**

```typescript
export interface AutopilotBuild {
  id: string;
  status: BuildStatus;
  progress: number;          // 0–100
  currentStage: AutopilotStageId | string;
  stages: Record<string, StageStatus>;
  messages: BuildMessage[];
  result?: BuildResult;
  // ...
}
```

Key design choices:

- **`stages: Record<string, StageStatus>`** — uses a dictionary rather than a tuple or array. The frontend can look up the status of any stage in O(1) by ID (`stages['embedding'].status`) without iterating an array. The key type is `string` (not `AutopilotStageId`) because future agents may introduce new stage IDs without a type change.

- **`currentStage: AutopilotStageId | string`** — allows the union of known stage IDs and arbitrary strings. This prevents the UI from crashing if the backend introduces a new stage before the frontend type is updated (graceful degradation).

- **`result?: BuildResult`** — optional because it only exists when `status === 'complete'`. Using `undefined` (via `?`) is cleaner than `result: BuildResult | null` because `undefined` is the natural TypeScript "not yet set" value and avoids the `null` vs `undefined` ambiguity in serialised JSON.

---

**Q79: How does the `MetadataFilter` type support the visual filter builder in the Designer's retrieval configuration?**

```typescript
export interface MetadataFilter {
  key: string;
  operator: FilterOperator;
  value: string | number | boolean | string[];
}

export type FilterOperator =
  | 'eq' | 'ne' | 'gt' | 'gte' | 'lt' | 'lte' | 'in' | 'nin' | 'contains';
```

The `operator` union type drives the UI directly:
- The filter builder renders an operator dropdown that lists exactly the values in `FilterOperator`.
- When `operator === 'in'` or `operator === 'nin'`, the `value` field must be `string[]` (a multi-value input), so the UI switches to a tag input. When it's `eq` or `ne`, the `value` can be a single scalar.

The `value: string | number | boolean | string[]` union intentionally mirrors what vector databases accept for metadata filters (Qdrant supports all these types). Using `unknown` would lose IDE autocompletion. Using `string` only would require string-encoding numbers ("5") and booleans ("true"), forcing backend parsing.

---

**Q80: What is `EmbeddingBenchmarkResult` and how does it enable the "explain decisions" feature in Autopilot?**

```typescript
export interface EmbeddingBenchmarkResult {
  model: string;
  score: number;
  costPer1MTokens: number;
  latencyMs: number;
}
```

This type is nested inside `AgentDecisions.embedding.benchmarkResults: EmbeddingBenchmarkResult[]`. Each entry represents one candidate model that the Embedding Tester Agent evaluated. When the build completes, the full `AgentDecisions` object is stored in `BuildResult.decisions`.

The Decision Explainer component renders this array as a comparison table:
```
Model                   Score   Cost/1M   Latency
text-embedding-3-large  0.89    $0.13     320ms   ← Selected
text-embedding-3-small  0.82    $0.02     95ms
cohere-embed-v3         0.84    $0.10     140ms
```

Without the typed `AgentDecisions` structure, the "explain in Designer" flow would have to display raw JSON — unreadable and unmaintainable. The typed structure allows the React component to destructure named fields and render them in meaningful sections, fulfilling the "explain decisions" value proposition.

---

**Q81: Why does `models.ts` mirror the JSON catalog structure exactly, and what is the trade-off of this approach?**

Each interface in `models.ts` (`EmbeddingModel`, `GenerationModel`, etc.) maps field-for-field to the corresponding JSON catalog entry:

```typescript
// TypeScript
export interface EmbeddingModel {
  id: string;
  dimensions: number;
  costPer1MTokens: number;
  mtebScore: number;
  // ...
}

// JSON (data/models/embeddings.json)
{
  "id": "text-embedding-3-small",
  "dimensions": 1536,
  "costPer1MTokens": 0.02,
  "mtebScore": 62.3
}
```

**Benefit**: When a component does `import embeddingsJson from '@/../data/models/embeddings.json'`, TypeScript can cast the result to `{ models: EmbeddingModel[] }` and get full autocomplete and type safety. The JSON is the source of truth; the TypeScript type is a typed lens over it.

**Trade-off**: If the JSON schema changes (e.g., a new field is added), the TypeScript type must be updated manually — there's no automatic synchronisation. The correct fix is to add the field as optional (`newField?: string`) until it appears in all catalog entries. A stricter approach (used by teams with more automation) would be to use `zod` to define both the schema AND the TypeScript type in one place, then validate at runtime:
```typescript
const EmbeddingModelSchema = z.object({ id: z.string(), dimensions: z.number(), ... });
type EmbeddingModel = z.infer<typeof EmbeddingModelSchema>;
```

For RAG Studio's current scope, manual mirroring is sufficient and avoids adding a Zod dependency to the type layer.

---

**Q82: What is a discriminated union and where could it be applied in the RAG Studio types?**

A discriminated union is a union of types where each member has a shared **discriminant field** (a literal type) that uniquely identifies it. TypeScript can narrow the type based on a check on the discriminant.

In RAG Studio, `MemoryConfig` is a candidate:

```typescript
// Current approach (simple union field)
export type MemoryType = 'none' | 'conversation-buffer' | 'summary-buffer' | 'vector-memory';
export interface MemoryConfig {
  type: MemoryType;
  windowSize?: number;    // only for conversation-buffer
  maxTokens?: number;     // only for summary-buffer
  sessionPersistence?: boolean;
}

// Discriminated union alternative
export type MemoryConfig =
  | { type: 'none' }
  | { type: 'conversation-buffer'; windowSize: number; sessionPersistence: boolean }
  | { type: 'summary-buffer'; maxTokens: number; sessionPersistence: boolean }
  | { type: 'vector-memory'; sessionPersistence: boolean };
```

The discriminated union makes field co-presence explicit — `windowSize` can only appear when `type === 'conversation-buffer'`. The simple approach (all fields on one interface with `?`) allows invalid states like `{ type: 'none', windowSize: 100 }`.

For MVP scope, the simple approach is used because the UI only reads/writes the fields relevant to the selected `type`, and the added complexity of discriminated unions is not yet necessary. This is documented as a future improvement.

---

**Q83: What is the purpose of the `PipelineMetadata` interface and why is `source` a union of string literals?**

```typescript
export interface PipelineMetadata {
  createdAt: string;
  updatedAt?: string;
  version: string;
  author?: string;
  source?: 'designer' | 'autopilot' | 'template';
  buildId?: string;
}
```

`PipelineMetadata` captures provenance — where a configuration came from and when it was last changed. The `source` field specifically enables the bidirectional integration:

- `'designer'` — user manually built the pipeline stage by stage.
- `'autopilot'` — the Autopilot orchestrator produced this configuration after optimization. When `source === 'autopilot'`, `buildId` is also set, linking back to the `AutopilotBuild` record.
- `'template'` — applied from the template gallery.

The Designer review page reads `metadata.source` from the URL query string and shows a contextual banner:
```
"Imported from Autopilot (build abc123) — review the decisions below"
```

Using a string literal union (instead of `string`) means the compiler will error if a component tries to set `source = 'manual'` — a misspelling that would silently pass with a plain `string` type.

---

## P1-3 · Python Pydantic Schemas

### Pydantic & Schema Design

**Q84: What is Pydantic v2 and why is it the right choice for FastAPI request/response validation?**

Pydantic v2 is a data validation library that uses Python type annotations to define models. It is the right choice for FastAPI because:

1. **FastAPI is built on Pydantic** — route parameter parsing, request body validation, and response serialisation all use Pydantic models natively.
2. **Compiled Rust core** — Pydantic v2 rewrote its validation engine in Rust, making it 5–50× faster than v1.
3. **Declarative constraints** — `Field(ge=0, le=100)` expresses business rules inline with the model definition, not in separate validation functions.
4. **JSON Schema generation** — FastAPI auto-generates OpenAPI documentation from Pydantic models, giving developers a self-documenting API at `/docs`.
5. **Serialisation control** — `model_config = ConfigDict(use_enum_values=True)` ensures enums are serialised as their string values in JSON responses, not as Python enum instances.

The alternative — manual `request.json()` parsing with ad-hoc validation — produces verbose, error-prone code where validation logic is scattered across route handlers.

---

**Q85: What is `StrEnum` and why is it used instead of plain `Enum` for `CloudProvider`, `ChunkingStrategy`, etc.?**

`StrEnum` (Python 3.11+) is an enum where every member's value is a string AND the enum itself is a subclass of `str`. This means:

```python
class CloudProvider(StrEnum):
    AWS = "aws"

# StrEnum: member IS a string
isinstance(CloudProvider.AWS, str)  # True
CloudProvider.AWS == "aws"          # True

# Plain Enum: member is NOT a string
class Provider(Enum):
    AWS = "aws"
isinstance(Provider.AWS, str)       # False
Provider.AWS == "aws"               # False
```

In RAG Studio this matters in three places:
1. **JSON serialisation** — `StrEnum` values serialise directly to `"aws"` without a `.value` call.
2. **Database storage** — When written to a PostgreSQL `VARCHAR` column (via SQLAlchemy in P1-4), the string value passes through without conversion.
3. **Comparison** — Route handlers can compare `config.cloud_provider == "aws"` without calling `.value`, reducing boilerplate.

The constraint is that `StrEnum` is only available from Python 3.11. The project targets 3.11+, so this is fine.

---

**Q86: Explain the `RAGBaseModel` pattern. Why have a shared base class instead of configuring each model individually?**

```python
class RAGBaseModel(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
        use_enum_values=True,
    )
```

**`alias_generator=to_camel`** — Automatically generates a camelCase JSON alias for every snake_case field. FastAPI uses the alias in the request/response JSON while Python code uses the snake_case name:
```python
class ChunkingConfigSchema(RAGBaseModel):
    chunk_size: int  # JSON key is "chunkSize", Python attr is "chunk_size"
```

**`populate_by_name=True`** — Without this, only the alias (`chunkSize`) would be accepted in model construction. With it, both `{"chunkSize": 512}` (JSON) and `{"chunk_size": 512}` (Python code) work. This matters in tests where you want to use Python keyword arguments directly.

**`use_enum_values=True`** — Stores the enum's `.value` (a string) instead of the enum instance. This ensures serialisation produces `"aws"` rather than `<CloudProvider.AWS: 'aws'>`.

Without a shared base, every schema would repeat this `model_config` block, and a future change (e.g., adding `from_attributes=True` for ORM mapping) would require touching every class.

---

**Q87: How does `alias_generator=to_camel` interact with the TypeScript frontend?**

The TypeScript frontend uses camelCase (`chunkSize`, `topK`, `cloudProvider`) everywhere — both in the type definitions (P1-2) and in the JSON payloads it sends and receives.

Python's convention is snake_case (`chunk_size`, `top_k`, `cloud_provider`). Without `alias_generator=to_camel`, FastAPI would expect JSON with snake_case keys, causing a mismatch:

```
Frontend sends: { "chunkSize": 512, "cloudProvider": "aws" }
FastAPI without alias: ValidationError — field "chunk_size" is required
FastAPI with alias:  ✅ maps "chunkSize" → chunk_size automatically
```

The `to_camel` function from `pydantic.alias_generators` applies `snake_case → camelCase` to every field name. The `populate_by_name=True` setting ensures the backend's own tests and internal service calls can still use Python snake_case without wrapping everything in the JSON alias.

---

**Q88: Why are the schema files split into five files (`pipeline.py`, `designer.py`, `autopilot.py`, `evaluation.py`, `deployment.py`) instead of one?**

Each file groups schemas by **API domain concern**, mirroring the router structure that will be built in P4–P8:

| File | Domain | Consumed by |
|------|---------|------------|
| `pipeline.py` | Core data model — the RAG config DSL | All other schema files; both modes |
| `designer.py` | Designer API request/response shapes | `routers/designer.py` (P4-2, P4-3, P4-4) |
| `autopilot.py` | Build lifecycle + agent decisions | `routers/autopilot.py` (P6-9) |
| `evaluation.py` | RAGAS metrics + failure analysis | `routers/evaluation.py` (P8-3) |
| `deployment.py` | Deployment lifecycle | `routers/deployment.py` (P8-4) |

`pipeline.py` is deliberately dependency-free (no imports from other schema files). All other files import from it. This prevents circular imports — if `autopilot.py` needed to import from `evaluation.py` and vice versa, the import graph would cycle. The current structure is a strict DAG: `pipeline → {designer, autopilot, evaluation, deployment}`.

---

**Q89: Explain how `PipelineConfigurationSchema` acts as the shared contract between Designer mode and Autopilot mode.**

`PipelineConfigurationSchema` is accepted or produced by endpoints in both modes:

- **Designer mode** — `POST /api/designer/config` accepts it as `SaveConfigRequest.config`. The user builds it stage by stage in the UI.
- **Autopilot mode** — `GET /api/autopilot/build/{id}/result` returns it as `BuildResultSchema.config`. The agent orchestrator produces it after optimisation.
- **Bidirectional handoff** — `StartBuildRequest.base_config` is an optional `PipelineConfigurationSchema` from the Designer "Optimize This" flow; `DecisionExplainer` sends the Autopilot's result back to the Designer as a `PipelineConfigurationSchema`.

This single type is the Rosetta Stone of the platform — it is the common language that both modes speak. Defining it once in `pipeline.py` (and mirroring it in TypeScript as `PipelineConfiguration`) ensures both sides always agree on the shape.

---

**Q90: What does `Field(ge=128, le=4096)` do in `ChunkingConfigSchema.chunk_size`, and what happens if the constraint is violated?**

`Field(ge=128, le=4096)` attaches a `GreaterThanEqual=128` and `LessThanOrEqual=4096` constraint to the field. Pydantic v2 validates this during model instantiation.

If a request body sends `{ "chunkSize": 64 }`:
```python
ChunkingConfigSchema(chunk_size=64)
# Raises: pydantic.ValidationError
# 1 validation error for ChunkingConfigSchema
# chunk_size
#   Input should be greater than or equal to 128 [type=greater_than_equal, input_value=64]
```

FastAPI catches this `ValidationError` and returns a `422 Unprocessable Entity` response with a structured error body:
```json
{
  "detail": [{
    "loc": ["body", "stages", "chunking", "chunkSize"],
    "msg": "Input should be greater than or equal to 128",
    "type": "greater_than_equal"
  }]
}
```

The 128-token lower bound prevents chunks too small to carry semantic meaning. The 4096 upper bound prevents chunks that exceed most embedding models' `maxTokens` limit (the largest in the catalog is 8191, but 4096 is a safe production ceiling that keeps retrieval sets manageable).

---

**Q91: Why is `DeploymentInfoSchema` defined in both `autopilot.py` and the data it shares with `deployment.py`? How is duplication avoided?**

`DeploymentInfoSchema` in `autopilot.py` represents deployment info embedded inside `BuildResultSchema` — it is a summary that the Autopilot includes in its final result.

`DeploymentStatusResponse` in `deployment.py` represents the full deployment record returned by `GET /api/deployment/{id}/status`.

They share the same fields but serve different contexts. Rather than duplicating:
- `autopilot.py`'s `DeploymentInfoSchema` is a **nested embedded type** within `BuildResultSchema` (lightweight: endpoint + status + deployed_at).
- `deployment.py`'s schemas are **standalone API responses** for the full deployment management API (includes pagination, environment, image_tag, error).

There is intentional overlap — both have `endpoint`, `status`, `deployed_at`. This is acceptable because they are used in different response shapes. The alternative (sharing one class) would create a coupling between the autopilot and deployment schema modules, which are on separate roadmap paths (P6 vs P8).

---

**Q92: What is the `__init__.py` barrel export pattern and what problem does it solve?**

```python
# schemas/__init__.py
from app.schemas.pipeline import PipelineConfigurationSchema, CloudProvider
from app.schemas.designer import SaveConfigRequest
# ...
```

Without the barrel:
```python
# Every router must know which sub-module holds which class
from app.schemas.pipeline import PipelineConfigurationSchema
from app.schemas.designer import SaveConfigRequest
from app.schemas.autopilot import BuildStatusResponse
```

With the barrel:
```python
# Every router uses one clean import
from app.schemas import PipelineConfigurationSchema, SaveConfigRequest, BuildStatusResponse
```

Benefits:
1. **Discoverability** — `from app.schemas import <Tab>` in an IDE shows all available schemas.
2. **Refactoring isolation** — If `PipelineConfigurationSchema` moves from `pipeline.py` to a new `core.py`, only `__init__.py` needs updating, not every router that imports it.
3. **`__all__` documentation** — The explicit `__all__` list in `__init__.py` serves as the authoritative list of public API types, useful for documentation generation.

---

**Q93: How do the `EvaluationMetrics` and `FinalMetricsSchema` in the evaluation module relate, and why have two separate metric types?**

```python
# evaluation.py
class EvaluationMetrics(RAGBaseModel):
    faithfulness: float
    answer_relevance: float
    context_precision: float
    context_recall: float
    avg_latency_ms: float | None = None
    cost_per_query: float | None = None

# autopilot.py
class FinalMetricsSchema(RAGBaseModel):
    # Identical fields
    faithfulness: float
    # ...
```

They are intentionally separate because they appear in different response types with different ownership:
- **`EvaluationMetrics`** is owned by the Evaluation module (`EvaluationRunResponse`). It is produced by `POST /api/evaluation/run` — a standalone, on-demand evaluation.
- **`FinalMetricsSchema`** is embedded in `BuildResultSchema` returned by `GET /api/autopilot/build/{id}/result`. It is produced by the Autopilot's internal evaluation step.

In a future phase, they may diverge: `EvaluationMetrics` might add per-question breakdowns, while `FinalMetricsSchema` might add iteration-over-iteration trend data. Keeping them separate now avoids premature coupling. A shared base class could be introduced when the fields genuinely converge.

---

**Q94: What is `FailureAnalysisResult` and how does it drive the Autopilot's iteration decision?**

```python
class FailureAnalysisResult(RAGBaseModel):
    total_failures: int
    failure_rate: float
    categories: list[FailureCategory]
    summary: str
```

Where `FailureCategory.category` is one of:
- `"hallucination"` — LLM generated a factual claim not supported by the retrieved context
- `"retrieval_quality"` — correct answers exist in the corpus but were not retrieved
- `"context_gap"` — the corpus genuinely does not contain the answer
- `"format_error"` — the LLM response structure was incorrect (e.g., not valid JSON when requested)

The Autopilot orchestrator's `decide_iteration` node reads the `categories` list to choose which agent to re-run:
- `"hallucination"` → lower LLM temperature or strengthen the grounding prompt
- `"retrieval_quality"` → tune `top_k`, enable reranking, or switch retrieval strategy
- `"context_gap"` → flag for the user (no pipeline fix can help if the data isn't there)
- `"format_error"` → adjust the system prompt's output format instruction

Without structured failure categories, the iteration loop would have to re-run all agents blindly on every failure, which is costly. The typed `FailureAnalysisResult` enables **targeted iteration** — only the relevant agent stage is re-run.

---

**Q95: Why does `BuildStatusResponse.stages` use `dict[str, StageStatusSchema]` rather than a `list[StageStatusSchema]` with a `stage_id` field?**

```python
stages: dict[str, StageStatusSchema]
# e.g. { "analyze": StageStatus, "chunking": StageStatus, ... }
```

A dict provides **O(1) lookup** by stage ID on both the backend and frontend:

**Backend** (inside the orchestrator):
```python
state["stages"]["embedding"].status = "complete"  # Direct update, no iteration
```

**Frontend** (React component):
```typescript
const embeddingStatus = build.stages["embedding"]?.status;  // O(1), no .find()
```

With a list, every update or lookup would require iterating to find the matching entry:
```python
stage = next(s for s in state["stages"] if s.stage_id == "embedding")  # O(n)
```

The trade-off: a dict does not preserve insertion order (in older Python), but Python 3.7+ dicts are ordered by insertion. And the UI renders stages in a fixed order from a constant array, not from the dict's iteration order — so order correctness is irrelevant here.

---

**Q96: What is `from_attributes=True` in Pydantic's ConfigDict, and when will it be needed for these schemas?**

`from_attributes=True` (formerly `orm_mode=True` in Pydantic v1) allows Pydantic to construct a model from an ORM object's attributes rather than from a dict:

```python
# Without from_attributes=True — must convert ORM object to dict first
pipeline_orm = await session.get(PipelineConfigDB, config_id)
response = PipelineConfigurationSchema(**pipeline_orm.__dict__)  # fragile

# With from_attributes=True — can pass ORM object directly
response = PipelineConfigurationSchema.model_validate(pipeline_orm)
```

It is not set on `RAGBaseModel` yet because:
- P1-3 only defines schemas — no ORM models exist yet (they come in P1-4).
- Setting it prematurely has no benefit and slightly increases model validation overhead.

In **P1-4** (database schema and migrations), SQLAlchemy ORM models will be created. The service layer (P4) will use `model_validate(orm_object)` to convert ORM rows to response schemas. At that point, `from_attributes=True` will be added to `RAGBaseModel` so all schemas can accept ORM objects without boilerplate conversion.

---

## P1-4 · Database Schema & Migrations

**Q97: Why use SQLAlchemy 2.0's `Mapped` and `mapped_column` instead of the older Column-based syntax?**

SQLAlchemy 2.0 introduced a new `Mapped[T]` annotation style that brings full type-safety to ORM models:

```python
# Old style (SQLAlchemy 1.x) — no type checking
class Project(Base):
    __tablename__ = "projects"
    id = Column(UUID(as_uuid=True), primary_key=True)
    name = Column(String(255), nullable=False)

# New style (SQLAlchemy 2.0) — fully typed
class Project(Base):
    __tablename__ = "projects"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
```

The `Mapped[T]` annotation is read by mypy/pyright to validate attribute access, catch type mismatches, and enable IDE autocomplete. The Python type annotation is the source of truth; `mapped_column()` provides the SQL-specific options.

---

**Q98: What is `DeclarativeBase` and why does RAG Studio define a custom `Base` class rather than using `declarative_base()`?**

`DeclarativeBase` is SQLAlchemy 2.0's class-based replacement for the older `declarative_base()` function:

```python
# Old (SQLAlchemy 1.x)
from sqlalchemy.orm import declarative_base
Base = declarative_base()

# New (SQLAlchemy 2.0) — preferred
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

Using a class allows shared class-level configuration (type annotation maps, registry overrides) to propagate to all ORM models. `TimestampMixin` is kept as a separate mixin so models that do not need timestamps can still inherit `Base` cleanly.

---

**Q99: Explain the `TimestampMixin` pattern. Why is it a mixin rather than a base class?**

`TimestampMixin` adds `created_at` and `updated_at` columns to any model that opts in:

```python
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )
```

As a mixin, models use `class Project(Base, TimestampMixin)` — multiple inheritance without coupling `Base` to timestamps. `server_default=func.now()` lets PostgreSQL set the value accurately regardless of application time drift. `onupdate=func.now()` is a SQLAlchemy client-side trigger fired when the ORM detects a changed row.

---

**Q100: Why are `config`, `requirements`, `stages`, and `messages` stored as JSONB rather than normalised columns?**

JSONB (PostgreSQL binary JSON) is appropriate here because:

1. **Schema volatility** — pipeline configurations evolve; a new provider or strategy option does not require a new column.
2. **Nested objects** — `PipelineConfigurationSchema` has deeply nested sub-schemas. Normalising them would require 20+ join tables.
3. **Indexed queries** — JSONB supports GIN indexes for key-existence queries (`WHERE config @> '{"cloudProvider":"aws"}'`).
4. **Clean boundary** — the service layer serialises Pydantic schemas to dicts (`.model_dump()`) before writing and deserialises on read (`.model_validate()`).

Stable, frequently-queried attributes like `cloud_provider`, `status`, and `source` remain as typed columns for direct SQL filtering.

---

**Q101: How does RAG Studio configure `alembic/env.py` for async PostgreSQL?**

```python
async def run_migrations_online() -> None:
    engine = create_async_engine(get_settings().database_url)
    async with engine.connect() as conn:
        await conn.run_sync(do_run_migrations)
    await engine.dispose()

if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

Key decisions: `create_async_engine` uses the `asyncpg` driver matching the app. `conn.run_sync(do_run_migrations)` bridges the async connection to Alembic's sync `context.run_migrations()` API. `get_settings()` reads `DATABASE_URL` from env, so no URL is hard-coded in `alembic.ini`.

---

**Q102: Why is the initial migration handwritten rather than autogenerated?**

Autogenerate requires a live database to diff against. A handwritten migration:

1. Is self-contained — reviewable in a PR without running a database.
2. Is deterministic — not dependent on an existing database state.
3. Documents intent explicitly — `server_default`, `ondelete`, index names are all visible.

Future migrations can be autogenerated (`alembic revision --autogenerate`) once a baseline schema exists to diff against.

---

**Q103: Explain the foreign key `ondelete` strategies across the 5 tables.**

| Relationship | `ondelete` | Reasoning |
|---|---|---|
| `pipeline_configs.project_id` | `CASCADE` | Config records are meaningless without their project. |
| `autopilot_builds.project_id` | `CASCADE` | Build history belongs to the project. |
| `evaluation_runs.config_id` | `CASCADE` | An evaluation without its config is orphaned. |
| `evaluation_runs.build_id` | `SET NULL` | Metrics remain valid even if the build is deleted; `build_id` becomes NULL. |
| `deployments.config_id` | `CASCADE` | A deployment record without its config is unusable. |

---

**Q104: How does `from_attributes=True` enable ORM-to-Pydantic conversion?**

It allows Pydantic to read attributes from an ORM object using `getattr()` instead of requiring a plain dict:

```python
# Without from_attributes=True — manual dict conversion required
pipeline_orm = await session.get(PipelineConfig, config_id)
schema = PipelineConfigurationSchema(**pipeline_orm.__dict__)   # fragile

# With from_attributes=True (added in P1-4)
schema = PipelineConfigurationSchema.model_validate(pipeline_orm)  # clean
```

It was omitted from P1-3 because no ORM models existed yet, and was added to `RAGBaseModel` in P1-4 once the model layer was introduced.

---

**Q105: What is the purpose of `app/models/__init__.py` importing all model classes?**

Alembic inspects `Base.metadata` to discover tables for autogenerate. `Base.metadata` only knows about tables whose model modules have been imported. The `__init__.py` imports all model classes as a side-effect:

```python
from app.models.project import Project
from app.models.pipeline_config import PipelineConfig
# ... etc.
```

Then `env.py` does `from app.models import Base`, which triggers all model imports and registers all five tables with `Base.metadata`. Without this, autogenerate produces empty migrations.

---

**Q106: Why does `EvaluationRun` have a nullable `build_id` FK with `SET NULL`?**

Evaluation runs can be triggered in two contexts:
- **Designer-mode** (`POST /api/evaluation/run`) — user manually evaluates a saved config; no build involved → `build_id = NULL`.
- **Autopilot-triggered** — the Evaluation Agent runs RAGAS during an automated build → `build_id` references the parent build.

`SET NULL` on delete means: if the parent build is deleted, the evaluation record (and its metric scores) are preserved with `build_id = NULL`, rather than being cascade-deleted.

---

**Q107: What does `server_default="{}"` on a JSONB column achieve vs Python-side `default=dict`?**

`server_default="{}"` — the database supplies `{}` for `INSERT` statements that omit the column, including raw SQL inserts and migrations. `default=dict` — SQLAlchemy calls `dict()` in Python when constructing the ORM object, which means it only applies to Python-originated inserts.

`server_default` is preferred because it works even when rows are inserted by tools that bypass the ORM (migrations, seed scripts). Using a mutable `default={}` (a dict literal) would be a bug — all instances would share the same dict object; `default=dict` (callable) is safe but `server_default` provides stronger guarantees.

---

**Q108: What is `alembic.ini`s `file_template` setting?**

`file_template = %%(rev)s_%%(slug)s` controls migration filenames. Instead of the default timestamp-prefixed `2026042900001234_initial_schema.py`, it produces `001_initial_schema.py`. Short revision-ID-prefixed names are easier to reference in `alembic downgrade 001` and more readable in `git log`. The `%%` double-escaping is required because `alembic.ini` uses Python `configparser`, which treats `%` as an interpolation character.

---

**Q109: Walk through the full lifecycle of saving a Designer config to the database.**

```python
# 1. FastAPI receives POST /api/designer/config — Pydantic validates
async def save_config(body: SaveConfigRequest, session: AsyncSession):

    # 2. Serialise Pydantic schema to dict for JSONB
    config_dict = body.config.model_dump(by_alias=True)  # camelCase

    # 3. Create ORM object
    record = PipelineConfig(
        project_id=uuid.UUID(body.project_id),
        name=body.name,
        cloud_provider=body.config.cloud_provider,
        config=config_dict,
        source=body.config.metadata.source,
    )

    # 4. Persist
    session.add(record)
    await session.commit()
    await session.refresh(record)

    # 5. Deserialise ORM → Pydantic (from_attributes=True)
    config_schema = PipelineConfigurationSchema.model_validate(record.config)
    return SaveConfigResponse(id=str(record.id), config=config_schema, ...)
```

Flow: **JSON → Pydantic validate → dict → JSONB → ORM → dict → Pydantic → JSON**.

---

**Q110: What indexes does the initial migration create and why?**

| Index | Table | Column | Why |
|---|---|---|---|
| `ix_projects_user_id` | `projects` | `user_id` | List all projects for a user — most common query. |
| `ix_pipeline_configs_project_id` | `pipeline_configs` | `project_id` | List configs for a project. |
| `ix_autopilot_builds_project_id` | `autopilot_builds` | `project_id` | List builds for a project. |
| `ix_evaluation_runs_config_id` | `evaluation_runs` | `config_id` | List eval history for a config. |
| `ix_evaluation_runs_build_id` | `evaluation_runs` | `build_id` | Fetch evals linked to a specific build. |
| `ix_deployments_config_id` | `deployments` | `config_id` | List deployments for a config. |

All FK columns that are used as filter predicates are indexed. Primary keys are automatically indexed by PostgreSQL. `status` on `autopilot_builds` is not indexed initially — once volume grows, a partial index on `status = 'running'` is more efficient than a full index.

---

## P2-1 · Document Ingestion Service

### Document Loading & Format Support

**Q111: Why use LangChain's `Document` type as the universal output rather than defining a custom dataclass?**

`langchain_core.documents.Document` is already a transitive dependency (required by LangChain), provides a standard `page_content` + `metadata` contract, and every downstream service (chunking, embedding, retrieval) in the RAG pipeline expects this type. Defining a custom equivalent would require adapters at every boundary — pure boilerplate with no added value.

---

**Q112: The `DocumentLoader` base class uses an abstract `load` method that accepts `str | Path | bytes`. Why not separate loaders for files vs bytes?**

A single `load` signature allows `LoaderFactory` to dispatch without knowing the source type at the factory level — the caller decides what to pass. It also means a `PDFLoader` can be unit-tested entirely in-memory (passing bytes) without touching the filesystem, which keeps tests fast and hermetic. The trade-off is that each concrete loader must handle the union type with an `isinstance` branch.

---

**Q113: Why does `LoaderFactory.from_extension` normalise the extension with `.lower().lstrip(".")`?**

Users or OS file systems may produce extensions in any casing (`PDF`, `.PDF`, `pdf`) or include the leading dot (from `Path.suffix`). Normalising at the factory boundary means all eight concrete loaders receive a clean key and never need defensive casing logic internally. This is a classic "parse at the boundary" principle — validate/normalise once, trust inside.

---

**Q114: The `PDFLoader` produces one `Document` per page. What are the implications for downstream chunking?**

Each page-level document carries `page_number` and `total_pages` in its metadata, which the chunking service can propagate into every chunk. This preserves provenance so the final retrieved chunk can cite the exact page. The downside is that very short pages (table of contents, blank pages) produce tiny documents — the ingestion service handles this by filtering out documents whose text is empty after preprocessing.

---

**Q115: How does `URLLoader` differ from the file-based loaders in its error handling?**

File loaders raise exceptions on I/O errors (which propagate to the caller). `URLLoader` returns an empty list for both fetch failures and empty extraction results — a network failure is expected and transient, so swallowing it and returning nothing is safer than crashing the ingestion pipeline. The failure is still logged at `WARNING` level so operators can observe it. This is an intentional asymmetry: disk I/O failures are programming errors; network failures are operational events.

---

**Q116: What is `trafilatura` and why was it chosen for URL extraction over `requests` + `BeautifulSoup`?**

`trafilatura` is a focused library for web content extraction. It implements heuristics to identify the main body text of a page (article, documentation, blog post) and discard navigation, ads, sidebars, and boilerplate — tasks that would require substantial custom logic with raw BeautifulSoup. It also handles encoding detection and structured text output. `BeautifulSoup` is still used for `HTMLLoader` where the full raw HTML is already available and needs simple tag stripping.

---

### Text Preprocessing

**Q117: Why are preprocessing transforms implemented as pure module-level functions AND wrapped in a `TextPreprocessor` class?**

Pure functions are independently testable, composable, and reusable outside the class (e.g., calling `fix_encoding` directly in a one-off script). The `TextPreprocessor` class provides a configurable pipeline — it holds user preferences as constructor arguments and calls the pure functions in the correct order. This is the "strategy pattern light": no inheritance or ABCs needed, just a thin orchestration wrapper over pure transforms.

---

**Q118: Why does `TextPreprocessor` always apply `fix_encoding` regardless of the config?**

Encoding issues (null bytes from PDFs, denormalised Unicode from mixed-language docs) can silently corrupt downstream tokenisation and vector embeddings. There is no legitimate use case for keeping null bytes or denormalised Unicode in document text — unlike HTML stripping or header removal, which might be intentionally skipped for certain source types. Making `fix_encoding` non-optional removes an entire class of subtle bugs.

---

**Q119: Explain the `remove_headers_footers` heuristic. What are its limitations?**

The function splits text on form-feed (`\f`) characters — the standard page separator emitted by pypdf — then counts how often each short line appears at the top or bottom of pages. Lines appearing ≥ `min_occurrences` (default 3) times are classified as headers/footers and stripped.

**Limitations:**
- Requires `\f` separators; DOCX or HTML documents don't have these, so it's a no-op.
- Threshold-based: a footer that only appears on 2 of 3 pages (e.g., first page has a different footer) won't be caught.
- May accidentally remove a legitimate repeated phrase (e.g., a chapter title that happens to appear at the bottom of each section page).
- String equality matching — slight variations (e.g., "Page 1 of 10" vs "Page 2 of 10") are not matched.

---

### Metadata Extraction

**Q120: What is the separation of concerns between `loaders.py` and `extractors.py`?**

`loaders.py` is responsible for parsing the source format and extracting raw text — it sets basic provenance metadata (`source`, `file_type`, `page_number`). `extractors.py` performs a second enrichment pass on the already-parsed content: reading PDF document info dictionaries, DOCX core properties, HTML `<head>` tags, and section heuristics. This separation means the loading step can run even if metadata extraction fails (it logs a warning and returns `{}`), and each concern can evolve independently.

---

**Q121: Why does `extract_section_headers` cap results at 10 in `IngestionService._extract_metadata`?**

Section headers are a lightweight structure signal stored in document metadata, not a primary retrieval index. Storing 100 headers per document wastes metadata space and adds noise. Ten headers capture the document's top-level structure without becoming a content dump. The cap is applied by the service (`headers[:10]`), not the extractor, so callers with different needs can still get the full list from the extractor directly.

---

**Q122: How does `_safe_str` in `extractors.py` protect against PDF/DOCX metadata quirks?**

PDF metadata values can be `None` (field not set), an empty string `""`, a `pypdf` special object, or a real string. `_safe_str` converts any value to `str`, strips whitespace, and returns `None` if the result is empty. This allows the caller to use a compact dict comprehension (`{k: v for k, v in ... if v is not None}`) to drop all missing fields without handling each type separately.

---

### IngestionService Design

**Q123: `IngestionService.load` filters out empty documents after preprocessing. Why not filter before preprocessing?**

Some documents are non-empty raw but become empty after cleaning (e.g., a PDF page containing only a watermark image — pypdf produces an empty string for it; another page may contain only whitespace or HTML boilerplate). Filtering before preprocessing would miss those cases. Filtering after ensures the final `Document` list contains only documents that carry real content to the downstream chunker.

---

**Q124: `IngestionConfig` mirrors fields from `DataIngestionPreprocessingSchema`. Why not reuse the schema directly?**

`DataIngestionPreprocessingSchema` is a Pydantic model living in `app/schemas/` — part of the API boundary layer. `IngestionConfig` is a plain `dataclass` in the core service layer. Using the Pydantic schema in core services would couple the business logic to the API contract, making it harder to change one without affecting the other. The service accepts a simple dataclass; the router layer is responsible for converting the Pydantic schema to `IngestionConfig`.

---

**Q125: Walk through what happens when `IngestionService.load` is called with a bytes source for a PDF.**

1. `_load_raw` detects `source_type="bytes"`, infers extension from `filename` (or uses explicit `file_type`).
2. `LoaderFactory.from_extension("pdf")` returns a `PDFLoader` instance.
3. `PDFLoader.load(content, filename=...)` creates a `PdfReader(BytesIO(content))`, iterates pages, returns one `Document` per page.
4. Back in `load`, `TextPreprocessor.preprocess` cleans each page's text (fix encoding, normalize whitespace).
5. Empty pages are discarded.
6. `_extract_metadata` is called: since `file_type` in doc metadata is `"pdf"` and source is bytes, `extract_pdf_metadata(content)` reads title/author/page_count from the PDF info dict.
7. `extract_file_metadata` is skipped (source is not `"file"` type).
8. `extract_section_headers` scans the cleaned text and attaches up to 10 headers.
9. `source.custom_metadata` is merged last (highest precedence).
10. The resulting `list[Document]` is logged and returned.

---

## P2-2 · Chunking Service

### Core Architecture & Design

**Q126: Why is `Chunk` defined as a type alias (`Chunk = Document`) rather than a separate subclass?**

LangChain's `Document` already carries both `page_content` and `metadata`. A subclass would require either overriding nothing (pure marker type, no value) or adding fields that break interoperability with every LangChain component expecting `Document`. A type alias makes intent clear in signatures (`list[Chunk]`) while keeping zero friction: every utility, splitter, and vector-store wrapper in the LangChain ecosystem accepts `Chunk` directly. A new class would produce silent `isinstance` failures in downstream code.

---

**Q127: `ChunkingConfig` is a plain `dataclass`, not a Pydantic model. Why?**

`ChunkingConfig` is internal service configuration, not an API boundary. Pydantic's validation overhead, JSON serialization, and field aliases add no value when the config is constructed in Python code by the service layer. A `dataclass` gives typed fields and a readable `__repr__` with zero import weight. At the API boundary, `ChunkingConfigSchema` (Pydantic, in `app/schemas/`) converts to `ChunkingConfig` — each layer uses the right tool for its concerns.

---

**Q128: Why does `TextChunker` use an ABC with a single abstract method instead of a protocol?**

A `Protocol` would be correct for structural typing but doesn't enforce implementation at class-definition time — a missing `chunk` method only fails at the call site. `ABC` raises `TypeError` when you try to instantiate an incomplete subclass, giving an immediate developer-friendly error: `"Can't instantiate abstract class X with abstract method chunk"`. Since we own all concrete classes (no third-party implementations), the stricter ABC check is the right choice. If the codebase ever needed to accept third-party chunkers by duck-typing, migrating to Protocol would be straightforward.

---

**Q129: Why is `_make_chunk` a method on `TextChunker` rather than a standalone module function?**

`_make_chunk` is implementation-shared behavior that every concrete chunker uses identically. Placing it on the base class ensures:
1. Every subclass gets it for free without imports.
2. If the metadata schema changes (e.g., adding `chunk_hash`), one edit propagates everywhere.
3. The leading `_` signals "internal helper, not public API" — subclasses use it but callers don't.

A module-level function would work, but it's looser coupling — a subclass could accidentally forget to call it or import a different version.

---

**Q130: What is the `_STRATEGY_MAP` dispatch pattern, and why is it better than a large `if/elif` chain?**

`_STRATEGY_MAP` is a `dict[str, type[TextChunker]]` mapping strategy name strings to chunker classes. It is defined once in `__init__.py` and looked up in `ChunkerFactory.from_strategy`. Benefits:
- **O(1) lookup** — no sequential string comparisons.
- **Extensibility without modification** — adding a new strategy requires only a new entry in the dict, not editing `if/elif` branches in two places.
- **Single source of truth** — `supported_strategies()` returns `sorted(_STRATEGY_MAP.keys())` without any duplication.
- **Testability** — you can patch a single dict entry rather than mocking branch logic.

The `if/elif` approach would require touching the factory AND the supported-list method on every addition, violating the Open-Closed Principle.

---

### Strategy-Specific Design

**Q131: `FixedSizeChunker` implements its own character-count sliding window instead of using `RecursiveCharacterTextSplitter(separators=[""])`. Why?**

`RecursiveCharacterTextSplitter` applies a hierarchy of separators before falling back to character splitting. Even with `separators=[""]`, it still tokenizes and does boundary logic. `FixedSizeChunker` is the "dumb baseline" — it must guarantee exact `chunk_size` character windows regardless of any boundary. A manual `while start < len(text)` loop is:
- Deterministic: no separator inference.
- Zero external dependencies: no LangChain import needed.
- Faster for benchmarking: the simplest possible baseline to compare other strategies against.

---

**Q132: What are the default separators in `RecursiveCharacterChunker` and why in that specific order?**

Default separators: `["\n\n", "\n", ". ", " ", ""]`.

The order implements a priority hierarchy of semantic boundaries:
1. `"\n\n"` — paragraph break (highest semantic boundary, split here first).
2. `"\n"` — single newline (sentence or list-item boundary).
3. `". "` — sentence end with trailing space (avoids splitting on decimal points like `1.5`).
4. `" "` — word boundary (last resort before character split).
5. `""` — character-level (only if no word boundary fits within `chunk_size`).

LangChain's splitter tries the first separator; if the resulting pieces are still too large, it recurses with the next separator. This ensures the smallest semantically-meaningful unit is never split unless necessary.

---

**Q133: `SemanticChunker` builds buffered context windows around sentences. What problem does buffering solve, and what is the cost?**

**Problem:** Two adjacent sentences may share a concept but individually appear semantically unrelated. For example, `"The capital of France."` and `"It is famous for the Eiffel Tower."` — the pronoun "It" makes them related but their isolated embeddings would have low cosine similarity.

**Solution:** `buffer_size=1` means each sentence's "context" is `[prev_sentence, current, next_sentence]` concatenated. The embedding represents the semantic context, not just the sentence.

**Cost:** Buffer size multiplies the embedding call count. With `buffer_size=1` on N sentences, you compute N embeddings of ~3× the length — roughly 3× the embedding tokens. For large documents, this makes semantic chunking significantly more expensive than the fixed-size baseline.

---

**Q134: `SemanticChunker` caches the embedding model in `self._model_cache`. Why not a module-level singleton?**

A module-level singleton would:
1. Load the model at import time, adding several seconds to startup even if semantic chunking is never used.
2. Make it impossible to run multiple `SemanticChunker` instances with different models in the same process (e.g., during benchmarking).
3. Complicate testing — the module-level object persists across tests and requires explicit teardown.

A per-instance dict (`self._model_cache`) loads lazily (only when `chunk()` is first called), supports per-instance isolation, and is trivially mocked in tests by passing a pre-populated `_model_cache` dict or patching `SentenceTransformer`.

---

**Q135: `MarkdownHeaderChunker` merges two metadata dicts: `{**doc.metadata, **split_doc.metadata}`. Why not just use `split_doc.metadata`?**

LangChain's `MarkdownHeaderTextSplitter` returns new `Document` objects whose metadata contains only the header hierarchy (e.g., `{"Header 1": "Introduction", "Header 2": "Background"}`). The original document's provenance metadata — `source`, `file_type`, `page_number`, `author` — is discarded. The merge `{**doc.metadata, **split_doc.metadata}` preserves all parent provenance fields while letting the splitter's header metadata win on key conflicts. Without this merge, every downstream retrieval result would lack the source attribution needed to cite documents to the user.

---

**Q136: How does `HTMLSectionChunker` handle arbitrary nested HTML without a fixed template?**

It uses BeautifulSoup to walk the DOM looking for heading tags (`h1`–`h4`). When a heading is found, it starts a new "section" with that heading as the title. It then collects text from sibling/descendant elements that carry content: `p`, `li`, `td`, `th`, `blockquote`, `pre`, `code`. This element whitelist avoids pulling navigation links, scripts, style tags, and decorative elements. The approach handles arbitrary nesting because BeautifulSoup's tree traversal is recursive — it doesn't depend on the HTML conforming to any specific template. Trade-off: documents without heading tags produce a single large chunk, which the caller should then pass through another splitter or filter.

---

**Q137: `SentenceChunker` uses a regex `(?<=[.!?])\s+` instead of NLTK or spaCy. What are the accuracy and dependency trade-offs?**

| Aspect | Regex | NLTK / spaCy |
|--------|-------|-------------|
| Accuracy | ~90% for English prose; fails on abbreviations (`Dr.`, `U.S.`), decimal numbers, ellipsis | >97% accuracy; handles abbreviations, multilingual |
| Dependencies | None (Python stdlib `re`) | NLTK needs punkt corpus download; spaCy needs language model (50–500 MB) |
| Startup | Instant | 1–10 s model load |
| Language support | English only | spaCy: 60+ languages |

For RAG Studio's use case (English-first, startup latency matters, no additional pip dependencies), the regex is the right trade-off. A future upgrade path exists if multilingual support is required.

---

**Q138: `ParagraphChunker` applies a fallback splitter for oversized paragraphs. What triggers the fallback and why `chunk_size * 2`?**

The threshold `len(para) > config.chunk_size * 2` is deliberately loose. A paragraph just over `chunk_size` (e.g., 540 chars when `chunk_size=512`) should stay intact — splitting a tight single paragraph breaks its semantic unity. The `2×` factor accepts moderate oversize before invoking the recursive splitter. The recursive splitter then applies the standard `chunk_size` / `chunk_overlap` settings, so the resulting pieces fit the embedding model's token window. Using exactly `chunk_size` as the threshold would over-split naturally long technical paragraphs.

---

**Q139: `CodeAwareChunker` resolves language through a three-level priority chain. What is the chain and why that order?**

Priority (highest to lowest):
1. **`config.language`** (explicit config, e.g., `"python"`) — developer intent overrides all inference.
2. **`file_extension` metadata** (e.g., `.py`, `.ts`) — set by the ingestion layer from the source filename; reliable for single-language files.
3. **`source` metadata** (URL or path) — fallback if `file_extension` wasn't set explicitly.
4. **`"python"` default** — safe default; Python's language separators (`\nclass `, `\ndef `) are common in many technical documents.

The order ensures explicit config always wins, file-level signals are used when available, and a deterministic fallback prevents `KeyError` on unknown extensions.

---

### Chunk Quality Scoring

**Q140: What do `content_density`, `completeness`, and `size_score` measure, and why are these three dimensions chosen?**

- **`content_density`** (proportion of non-whitespace chars): Detects "whitespace chunks" — artifacts from blank-line splits or header-only fragments. A chunk of 512 characters that is 70% spaces carries little semantic content for embedding.
- **`completeness`** (ends with `.`, `!`, or `?`): Detects mid-sentence cuts. An embedding trained on truncated sentences produces less accurate vectors because transformer attention relies on sentence-final context.
- **`size_score`** (linear proximity to `target_size`): Controls embedding efficiency — very small chunks waste the model's sequence capacity; very large chunks exceed token limits or dilute the semantic signal.

These three cover the primary failure modes of chunking: content-free, truncated, and wrongly-sized chunks. Semantic coherence (a fourth dimension) would require an LLM call, making it too expensive for routine filtering.

---

**Q141: Why does `ChunkQualityScorer.__init__` raise `ValueError` if weights don't sum to 1.0 rather than normalizing them silently?**

Silent normalization would mask misconfiguration. If a caller passes `density_weight=0.5, completeness_weight=0.5, size_weight=0.5`, normalizing gives each weight `~0.33` — the user's intent was to weight density and completeness equally at 50% each, but the size dimension was accidentally included. The error forces explicit intent: users must think about the trade-offs and write weights that sum to 1.0. The `1e-6` tolerance handles floating-point representation errors (e.g., `0.4 + 0.3 + 0.3 == 0.9999999...`).

---

**Q142: The `size_score` formula is `max(0.0, 1.0 - abs(len(text) - target_size) / target_size)`. What does this produce for a chunk at 2× the target size?**

At `len(text) = 2 * target_size`:
```
deviation = abs(2*target - target) / target = 1.0
size_score = max(0.0, 1.0 - 1.0) = 0.0
```
A chunk twice the target size scores zero on the size dimension. At 1.5× target: `deviation=0.5`, `size_score=0.5`. At 0.5× target: `deviation=0.5`, `size_score=0.5`. The formula is symmetric around `target_size` and clamps at 0 for chunks larger than `2×target`. This means the overall quality score for an over-large chunk is pulled down but not necessarily below the `min_score` threshold — the density and completeness dimensions can compensate, which is intentional (a long but complete and dense paragraph is still usable).

---

### Service & Factory Design

**Q143: `ChunkingService.chunk` accepts `config: ChunkingConfig | None` with a default of `None`. What is the rationale?**

Making `config` optional with `None → ChunkingConfig()` allows callers to get sensible defaults without boilerplate:
```python
chunks = ChunkingService().chunk(docs)  # Uses recursive-character, 512, overlap 50
```
This is the Autopilot agent's common pattern — it calls chunking with defaults first, then with tuned configs after optimization. Making `config` required would force every caller to construct a config object, adding ceremony with no safety benefit, since `ChunkingConfig()` itself has validated defaults.

---

**Q144: How would you add a new chunking strategy, say `"sliding-window"`, to this codebase?**

Four steps:
1. Create `apps/api/app/core/chunking/sliding_window.py` — implement `SlidingWindowChunker(TextChunker)` with a `chunk()` method.
2. Add `from .sliding_window import SlidingWindowChunker` to `__init__.py`.
3. Add `"sliding-window": SlidingWindowChunker` to `_STRATEGY_MAP` in `__init__.py`.
4. Add `"SlidingWindowChunker"` to the `__all__` list in `__init__.py`.

No changes to `ChunkerFactory`, `ChunkingService`, or any existing strategy file are needed. `ChunkerFactory.supported_strategies()` will automatically include `"sliding-window"` on the next call. This demonstrates the Open-Closed Principle: open for extension, closed for modification.

---

**Q145: `SemanticChunker` calls `RecursiveCharacterTextSplitter` as a fallback for oversized chunks. Why not just accept oversized chunks?**

Embedding models have hard token limits (e.g., `all-MiniLM-L6-v2` is 512 tokens). A chunk exceeding this limit is silently truncated by the tokenizer — the embedding represents only the first 512 tokens, discarding everything after. This produces an incorrect semantic embedding that will mislead retrieval. The fallback splitter ensures every output chunk is safe for the configured model by splitting at `config.chunk_size` characters (a conservative proxy for token count). The slight semantic discontinuity from the fallback split is preferable to silent data loss.

---

**Q146: When would you recommend each of the 8 chunking strategies in production?**

| Strategy | Ideal Use Case |
|----------|---------------|
| `fixed-size` | Homogeneous, unstructured text; baseline for benchmarking |
| `recursive-character` | General-purpose default; mixed-format documents |
| `semantic` | High-quality FAQ/Q&A where semantic coherence matters most; budget allows embeddings |
| `markdown-header` | Documentation sites, wikis, technical READMEs |
| `html-section` | Web-scraped content, HTML reports |
| `sentence-based` | News articles, prose narratives; preserving sentence integrity |
| `paragraph-based` | Academic papers, legal documents with natural paragraph structure |
| `code-aware` | Source code repositories; preserves function/class boundaries |

---

**Q147: Multiple chunkers use lazy imports inside `chunk()`. What are the benefits and trade-offs?**

**Benefits:**
- Chunkers are instantiated at startup for all strategies registered in `_STRATEGY_MAP`, but the heavy imports only occur when `chunk()` is actually called. Strategies never used by a deployment don't add to startup time.
- If a dependency is missing (e.g., `sentence-transformers` not installed), the error surfaces only when `SemanticChunker.chunk()` is called, not at module import. This allows the service to start even with a partial environment.

**Trade-offs:**
- Repeated calls re-execute the `from ... import ...` statement, though Python caches module imports in `sys.modules` so there's negligible overhead after the first call.
- Tooling (IDEs, mypy) may not trace lazy imports, making static analysis slightly weaker.

---

**Q148: Every chunk carries `chunk_index`, `total_chunks`, and `chunk_strategy` metadata fields. Who uses these and why?**

- **`chunk_index` / `total_chunks`**: The Autopilot's `ChunkingOptimizerAgent` uses these to reconstruct document structure, verify coverage (no lost chunks), and detect off-by-one errors in sliding windows. Frontend components can display "Chunk 3 of 12" provenance to users.
- **`chunk_strategy`**: `ChunkQualityScorer.filter_low_quality` logs the strategy name in the `chunk_filtered_low_quality` event, enabling per-strategy quality analysis. The evaluation agent correlates strategy names with RAGAS scores to identify which strategy produced the worst-quality chunks.

All parent metadata is also propagated (`source`, `page_number`, `author`, etc.) so that every retrieved chunk can be cited back to its original document location.

---

**Q149: When should a caller use `chunk_many` vs. multiple calls to `chunk`?**

`chunk_many` is a convenience wrapper that prevents accumulating partial results in the caller. It's correct to use when documents are logically grouped (e.g., by project or by file batch) and the caller wants a single flat `list[Chunk]` without managing intermediate lists. The current implementation is sequential. In a future async context, `chunk_many` would be the right place to add `asyncio.gather` without changing callers — the single-entry-point design makes that refactor non-breaking.

Multiple calls to `chunk` are better when the caller needs per-group metrics or wants to filter/transform chunks between groups.

---

**Q150: `ChunkingService` is a stateless class with only two methods. Why not just expose module-level functions?**

Stateless classes vs. functions is a style trade-off. In this codebase:
1. **Consistency**: `IngestionService` and all upcoming `EmbeddingService`, `RetrievalService` classes use the same `XyzService().method()` pattern — callers have a uniform mental model.
2. **Dependency Injection**: FastAPI's `Depends()` accepts callables and classes. `ChunkingService` can be injected into routers without changing the interface.
3. **Future state**: `ChunkingService` may need to hold a logger or config — a class makes that non-breaking. Adding state to a module function requires changing the function signature at all call sites.
4. **Mocking**: `patch("app.core.chunking.ChunkingService")` is cleaner than patching individual module functions during tests.

---

## P2-3 · Embedding Service

### Architecture & Design

**Q151: Walk me through the architecture of the Embedding Service. How does it fit into the wider RAG pipeline?**

The Embedding Service is the third core service in the RAG backend pipeline: `IngestionService → ChunkingService → EmbeddingService → VectorStoreService`.

Architecture layers:
- **`strategies.py`** — `Embedding = list[float]` type alias, `EmbeddingConfig` dataclass (mirrors `EmbeddingConfigSchema`), and `TextEmbedder` ABC with `embed_documents` and `embed_query` abstract methods.
- **Provider modules** (`openai.py`, `cohere.py`, `google.py`, `huggingface.py`, `nomic.py`) — one concrete `TextEmbedder` subclass per provider, each with lazy imports and its own model-ID mapping.
- **`benchmarker.py`** — `EmbeddingBenchmarker` runs multiple `EmbeddingConfig` options against the same corpus and ranks by throughput. Used by the Autopilot Embedding Tester Agent.
- **`cache.py`** — `EmbeddingCache` is a transparent cache-aside wrapper backed by Redis (binary-packed vectors) with an in-process dict fallback.
- **`__init__.py`** — `EmbedderFactory` dispatches from provider string to concrete class; `EmbeddingService` orchestrates the full pipeline and enriches output Document metadata.

Input: `list[Document]` (output of `ChunkingService.chunk()`).
Output: `list[tuple[Document, Embedding]]` — each chunk paired with its vector, ready for vector store upsert.

---

**Q152: Why does `EmbeddingService.embed()` return `list[tuple[Document, Embedding]]` instead of just `list[Embedding]`?**

Keeping Document and vector co-located prevents the caller from managing two parallel lists that must stay in sync across filtering, deduplication, or error recovery steps. The downstream `VectorStoreService.upsert()` needs both the text metadata (for filtering/attribution) and the vector — returning a tuple pair means the caller can never accidentally de-sync them.

The same design reason applies in the vector store: Qdrant and Pinecone upsert APIs accept `(id, vector, payload)` — the payload IS the Document metadata. Passing pairs from the embedding layer maps directly to that API shape.

---

**Q153: The `TextEmbedder` ABC has two abstract methods: `embed_documents` and `embed_query`. Why separate them?**

Some embedding models are trained to produce different vector representations for documents vs. queries to improve retrieval:

- **Cohere Embed v3**: accepts an `input_type` parameter (`"search_document"` vs. `"search_query"`). Documents and queries are encoded with different projection heads for asymmetric retrieval.
- **E5 models** (HuggingFace): require a `"passage: "` prefix for document text and `"query: "` prefix for query text to achieve best performance.
- **BGE models**: similarly benefit from prepending `"Represent this sentence: "` for queries.

Having a separate `embed_query` method means each concrete embedder can apply the correct prefix/input_type without the caller knowing. Callers that don't need this distinction simply call `embed_documents` for both.

---

**Q154: Why are all heavy imports (langchain_openai, sentence_transformers, etc.) inside method bodies rather than at the top of each module?**

This is the lazy-import pattern applied consistently across all core services. Reasons:
1. **Startup speed**: FastAPI starts in milliseconds even if sentence-transformers (which loads PyTorch) is not installed, because the import only fires when `HuggingFaceEmbedder.embed_documents()` is actually called.
2. **Optional dependencies**: A deployment that only uses OpenAI doesn't need `sentence_transformers` installed. Lazy imports let the app start and serve requests without every optional dependency being present.
3. **Test isolation**: Tests that mock `embed_documents` never trigger the real import, so the test suite runs without API keys or model downloads.

The trade-off is that `ImportError` surfaces at call time rather than at startup. This is acceptable because provider selection is runtime-configurable; the alternative (failing at startup for a provider not in use) is worse.

---

**Q155: The `EmbeddingConfig` is a plain Python `@dataclass`, not a Pydantic model. Why?**

All `core/` service configs are dataclasses by convention (matching `ChunkingConfig`, `IngestionConfig`). Reasons:
1. **No validation overhead**: Core services are internal; they receive already-validated data from the Pydantic schema layer (`EmbeddingConfigSchema`). Re-validating at every internal call is redundant.
2. **No Pydantic dependency inside `core/`**: The core services are self-contained and testable without importing Pydantic, keeping the layer boundary clean.
3. **Sensible defaults**: `@dataclass` with `field()` defaults allows `EmbeddingConfig()` with no arguments for prototyping, which Pydantic `BaseModel` also supports but with more ceremony.
4. **Faster instantiation**: Dataclass `__init__` is generated at class definition time; Pydantic v2 builds a Rust-backed validator, which is overkill for a private config object.

---

**Q156: How does the `EmbedderFactory` differ from the `ChunkerFactory`? Are they the same pattern?**

They are the same factory pattern — a module-level dispatch dict (`_PROVIDER_MAP` / `_STRATEGY_MAP`) and a static-method class with `from_X()` and `supported_X()` methods. The only structural difference is the key:

- `ChunkerFactory.from_strategy("recursive-character")` — key is a strategy name string matching `ChunkingStrategy` enum values.
- `EmbedderFactory.from_provider("openai")` — key is a provider name string matching `EmbeddingProvider` enum values.

Both return instances of the ABC (`TextChunker` / `TextEmbedder`). Both raise `ValueError` for unknown keys. Both expose a `supported_*()` method for introspection. The deliberate consistency means any developer familiar with one factory immediately understands the other.

---

### Provider Wrappers

**Q157: How does the `OpenAIEmbedder` handle the Matryoshka dimensions parameter?**

OpenAI's `text-embedding-3-small` and `text-embedding-3-large` support a `dimensions` parameter that compresses the output vector using Matryoshka Representation Learning (MRL). The legacy `text-embedding-ada-002` does not support this parameter and will error if it's passed.

`OpenAIEmbedder._build_client()` checks `if config.model in _DIMENSIONS_SUPPORTED` before adding `dimensions` to the kwargs — only v3 model IDs are in that set. This means:
- `EmbeddingConfig(model="text-embedding-3-small", dimensions=512)` → 512-dim vector (cheaper storage)
- `EmbeddingConfig(model="text-embedding-ada-002", dimensions=512)` → parameter silently omitted, 1536-dim vector returned

---

**Q158: The `CohereEmbedder` maps catalog model IDs like `"cohere-embed-v3"` to Cohere API names like `"embed-english-v3.0"`. Why maintain this mapping instead of using the API name directly in the config?**

The catalog in `data/models/embeddings.json` uses stable, human-readable IDs that the frontend `EmbeddingSelector` component renders. These IDs are also used in `EmbeddingProvider` enum validation and stored in `PipelineConfiguration`. If we stored Cohere's raw API names (`embed-english-v3.0`) in the user-facing config, three problems arise:

1. **Vendor lock-in leak**: internal API naming conventions bleed into the user-facing data model.
2. **Breaking changes**: if Cohere renames `embed-english-v3.0` to `embed-english-v3` (as they have done historically), every stored pipeline config breaks. The mapping localises the change to one dict in `cohere.py`.
3. **Catalog coherence**: the frontend's `embeddings.json` catalog can be updated independently of Cohere's API versioning.

---

**Q159: The HuggingFace and Nomic embedders use `normalize_embeddings=True`. What effect does this have and why is it correct here?**

`normalize_embeddings=True` L2-normalises each output vector to unit length (‖v‖ = 1). With unit-norm vectors:

- **Cosine similarity ≡ dot product**: `cos(u, v) = u·v / (‖u‖‖v‖) = u·v` when both are unit vectors. This is a significant speedup in vector stores that implement dot-product ANNS (approximate nearest neighbour search).
- **Qdrant and other stores default to cosine**: sending pre-normalised vectors avoids double-normalisation inside the vector store.
- **Catalog alignment**: `normalizable: true` is set for all HuggingFace and Nomic models in `embeddings.json`, confirming that normalisation is appropriate for these model families.

For models like `text-embedding-ada-002` (not normalizable in the catalog), normalisation would distort the magnitude information that the model encodes — which is why only the local-model embedders apply it.

---

### Caching

**Q160: Describe the `EmbeddingCache` design. How does it avoid re-embedding duplicate texts?**

`EmbeddingCache` implements the **cache-aside** pattern:

1. `embed_with_cache(embedder, texts, config)` iterates through all input texts.
2. For each text, it computes a cache key: `SHA-256("provider:model:dimensions:text")`.
3. **Cache hit** → return stored vector, no embedder call.
4. **Cache miss** → collect missed texts into a separate list.
5. After the full scan, call `embedder.embed_documents(miss_texts, config)` **once** for all misses (batch efficiency preserved).
6. Store each fresh vector in the cache, then assemble the full result list in original input order.

Storage: Redis primary (binary-packed, 4 bytes/float, TTL-controlled) with a dict fallback for tests/local dev without Redis.

Key design: the cache key includes `dimensions` so that `text-embedding-3-small` with `dimensions=512` and `dimensions=1536` produce different cache entries.

---

**Q161: Why are embeddings stored as binary-packed bytes in Redis rather than JSON?**

Embedding vectors for `text-embedding-3-large` are 3072 floats. Comparison:

| Format | Size per vector |
|--------|----------------|
| JSON string (`[0.123, ...]`) | ~25 KB |
| `struct.pack("3072f", ...)` | **12 KB** |

Binary packing (`struct.pack/unpack` with IEEE-754 single-precision) is:
- **2× smaller** → less Redis memory and faster network transfer
- **Faster** to serialise/deserialise than `json.dumps/loads`
- **Lossless** for the precision we need (single-precision ≈ 7 decimal digits, more than sufficient for cosine similarity ranking)

The trade-off is that Redis keys are not human-readable, but embedding vectors are never intended to be read directly.

---

**Q162: What happens if Redis is unavailable? Does the embedding service fail?**

No. `EmbeddingCache._get_redis()` calls `redis.from_url(...).ping()` on first access. If `ping()` raises (connection refused, timeout, wrong URL), the exception is caught silently, `self._redis` is set to `None`, and the method returns `None`. All subsequent calls skip the Redis branch and use the in-process `_memory` dict.

This makes the service resilient to missing infrastructure. The consequence is that in-memory cache data is lost on process restart and is not shared across multiple API worker processes — acceptable for development, but Redis should be healthy in production.

---

### Benchmarking

**Q163: The `EmbeddingBenchmarker` sorts results by `texts_per_second`. Is throughput the right primary metric for selecting an embedding model?**

Throughput is a good proxy for cost and latency but is not the only signal. The Autopilot agent that calls `EmbeddingBenchmarker` should combine throughput with:

1. **Retrieval quality** (MTEB score or domain-specific recall): the `embedding_sample` field in `BenchmarkResult` can be used to run a small similarity test on domain-relevant query/document pairs.
2. **Cost per million tokens**: `embeddings.json` provides `costPer1MTokens` — an open-source HuggingFace model with `costPer1MTokens: 0.0` but lower throughput may be cheaper overall than a fast paid API.
3. **Dimension/storage trade-off**: higher-dimensional vectors cost more to store and search.

Sorting by throughput alone produces a sensible default ranking (the fastest model is usually the cheapest API model), but the `BenchmarkResult` struct carries all the raw data needed for a weighted composite score.

---

**Q164: `BenchmarkResult.embedding_sample` stores the first embedding vector from the benchmark run. What is this for?**

`embedding_sample` lets the caller verify:
1. **Correct dimensionality**: `len(result.embedding_sample)` equals the actual output dimensions — catches cases where a provider ignores the `dimensions` parameter.
2. **Vector type**: ensures floats, not integers or strings, were returned.
3. **Similarity smoke test**: the calling agent can take `embedding_sample` and a known-good reference vector and compute cosine similarity to confirm the model produces meaningful embeddings for the domain corpus before committing to it.

This is especially useful in the Autopilot flow where the agent automatically selects a model — `embedding_sample` provides evidence for the decision log.

---

**Q165: How would you extend the Embedding Service to support a new provider, say Mistral or Jina?**

Four steps:
1. **Add a provider module** (`mistral.py`): create `MistralEmbedder(TextEmbedder)` with `embed_documents` and `embed_query` methods, using lazy imports.
2. **Register in `_PROVIDER_MAP`** (`__init__.py`): add `"mistral": MistralEmbedder`.
3. **Add to `EmbeddingProvider` enum** (`schemas/pipeline.py`): add `MISTRAL = "mistral"`.
4. **Add to the catalog** (`data/models/embeddings.json`): add the model entry with provider `"mistral"`.

No changes needed to `EmbeddingService`, `EmbedderFactory.supported_providers()`, or any tests that only test the factory dispatch — the strategy map is the single registration point. The design's open-closed property (open for extension, closed for modification) means adding a provider never risks breaking existing ones.

---

### Testing & Operations

**Q166: How do you unit-test `EmbeddingService` without calling OpenAI, Cohere, or loading sentence-transformers weights?**

Use a small concrete `TextEmbedder` subclass in the test module (or a `MagicMock`) that implements `embed_documents` / `embed_query` with deterministic vectors, then patch `app.core.embedding.EmbedderFactory.from_provider` to return that fake. Assertions focus on:

- Correct `list[tuple[Document, Embedding]]` length and ordering
- Metadata enrichment (`embedding_model`, `embedding_provider`, `embedding_dimensions`) on each output `Document`
- `embed_many` concatenating groups the same as sequential `embed` calls

This mirrors the chunking tests: mock heavy boundaries, assert orchestration and contracts.

---

**Q167: Why does the first `embed()` call with cache still send duplicate `page_content` strings to `embed_documents` in one batch?**

`EmbeddingCache.embed_with_cache` scans the input list in order; on the first pass nothing is stored yet, so every index is a miss and each string is appended to `miss_texts` — including repeated texts. After `embed_documents` returns, each miss is written to the cache. A **second** `embed()` with the same texts becomes all hits and does not call the provider again.

Deduplicating within a single batch would require an extra pass (text → first index map) and reordering logic; the current design optimises for simplicity and still saves cost across requests and sequential batches.

---

**Q168: What should integration tests cover that these unit tests deliberately skip?**

- Real Redis: verify `SETEX` / `GET` binary round-trip against a live `redis://` instance (e.g. Docker Compose service).
- One smoke test per provider behind env flags (`RUN_OPENAI_EMBEDDING_SMOKE=1`) with a tiny corpus and real API keys — catches SDK breakage and model deprecations.
- Dimensionality: assert `len(vector) == expected_dims` from the catalog for each model ID.

---

**Q169: How would you observe embedding latency and cache hit rate in production?**

- `structlog` already emits `embedding_complete` (provider, model, chunk count) and `embedding_cache_result` (hits, misses, total). Ship those fields to your log aggregator and build dashboards (p50/p95 latency from request middleware + embedding events).
- Add Prometheus counters/histograms in a later phase (`P11-*`) for `embedding_requests_total`, `embedding_cache_hits_total`, `embedding_duration_seconds`.

---

**Q170: If `BenchmarkResult.texts_per_second` is identical for two models, how do you break ties for Autopilot selection?**

Use secondary keys: (1) `avg_latency_ms` variance, (2) catalog `costPer1MTokens`, (3) `dimensions` (lower storage cost), (4) MTEB / domain recall from `embeddings.json`. The benchmarker only sorts by throughput; the agent should implement a stable composite sort for tie-breaking and log the rationale.

---

## P2-4 · Vector Store Service

### Architecture & API

**Q171: Where does `VectorStoreService` sit in the RAG pipeline relative to `EmbeddingService`?**

`EmbeddingService` outputs `list[tuple[Document, Embedding]]`. `VectorStoreService.index_pairs` consumes that shape directly (or `index` takes parallel lists). The vector store persists each dense vector with a JSON-serialisable payload derived from the `Document` (`page_content` + `metadata`). At query time, `EmbeddingService.embed_query` produces the query vector; `VectorStoreService.search` returns `ScoredDoc` objects for `RetrievalService` (P2-5) to fuse, rerank, or format into LLM context.

---

**Q172: Why is `VectorStoreService` async while `EmbeddingService` is synchronous?**

`AsyncQdrantClient` and remote HTTP (Weaviate REST) are naturally async-bound. Blocking a FastAPI worker thread on vector DB I/O would reduce concurrency under load. Embeddings are often batched CPU/GPU or HTTP calls wrapped by LangChain sync clients — that layer was kept synchronous for simpler use inside Celery tasks or scripts. Routers can `await` the vector store and run `embed_query` in `asyncio.to_thread` if both are needed in one handler without blocking the event loop.

---

**Q173: What is `VectorStoreRuntimeConfig` and why is it not a Pydantic model?**

Same pattern as `EmbeddingConfig` / `ChunkingConfig`: core services use plain `@dataclass` objects for internal speed and to avoid importing the entire schema package inside `app/core/`. `VectorStoreRuntimeConfig` carries `collection_name` (maps from `VectorStoreConfigSchema.index_name`), `vector_size`, `metric`, and optional Pinecone/Weaviate hints. Routers validate `VectorStoreConfigSchema` then map fields into `VectorStoreRuntimeConfig` before calling the service.

---

**Q174: How does `VectorStoreFactory.create` choose the implementation, and what must callers inject?**

`create(provider, config, **kwargs)` dispatches on `provider` string (`qdrant`, `pinecone`, `weaviate`). Qdrant **requires** `qdrant_client=AsyncQdrantClient` (no implicit global client inside `core/` — keeps tests hermetic and avoids hidden settings coupling). Pinecone requires `pinecone_api_key` (or `VectorStoreService` reads `PINECONE_API_KEY`). Weaviate requires `weaviate_url` (or `WEAVIATE_URL`). Unsupported providers raise `VectorStoreConfigurationError` with an explicit list of supported ids.

---

### Qdrant

**Q175: How are LangChain `Document` objects serialised into Qdrant payloads?**

Each `PointStruct` stores `payload = {"page_content": str, "metadata": dict}`. Non-JSON-native metadata values are stringified in `_json_safe_metadata` so Qdrant payload indexing never fails. Round-trip search reconstructs `Document(page_content=..., metadata=...)`. This mirrors how LangChain's own Qdrant integration names fields, making future interoperability easier.

---

**Q176: How are metadata filters (`VectorSearchFilter`) translated to Qdrant `Filter` objects?**

A minimal mapping covers `eq` → `MatchValue`, `in` (string lists) → `MatchAny`, and `contains` → `MatchText` for keyword-style substring search. Keys without a dot prefix are assumed to live under `metadata.<key>` in the payload. Complex boolean trees (`must_not`, nested `should`) are intentionally not fully implemented in P2-4 — the Retrieval layer can post-filter or we extend `_build_qdrant_filter` when product requirements demand it.

---

### Pinecone & Weaviate

**Q177: Why does `PineconeVectorStore` wrap SDK calls in `asyncio.to_thread`?**

The official Pinecone Python SDK exposes synchronous `Index.upsert` / `Index.query`. Running those directly inside an `async def` route would block the event loop. `asyncio.to_thread` offloads blocking work to the default thread pool, preserving async semantics at the `VectorStoreClient` boundary.

---

**Q178: Why is Weaviate implemented with raw `httpx` instead of `weaviate-client`?**

P2-4 aims for an optional second provider without adding another heavy dependency to every Docker image and CI install. The v1 REST schema API plus GraphQL `nearVector` covers create-class, batch upsert with external vectors, and similarity search for integration tests behind mocks. Teams that prefer the official client can wrap `WeaviateVectorStore` later or swap in LangChain's `WeaviateVectorStore` at the orchestration layer.

---

**Q179: What is the security story for `WeaviateVectorStore` class names in GraphQL?**

The class name is interpolated into a GraphQL query string. `__init__` rejects names that are not alphanumeric plus underscore, preventing injection of additional query clauses. For production, prefer allow-listed class names from `vector-stores.json` catalog metadata.

---

### Operations & testing

**Q180: How do you run automated tests without a Docker Qdrant container?**

Tests construct `AsyncQdrantClient(location=":memory:")` — Qdrant's embedded in-process engine. Each test gets an isolated client fixture that `await client.close()` on teardown. This satisfies the TASKS.md requirement for “Qdrant embedded mode” while keeping CI fast.

---

**Q181: When should you call `index(..., recreate_collection=True)`?**

When rebuilding an index from scratch after a breaking change to embedding model or dimensionality. It calls `delete_collection` first (Qdrant drops the collection; Pinecone logs a warning because account-level index deletion is destructive and intentionally not automated). For incremental updates, omit the flag and rely on `upsert` with new point IDs.

---

**Q182: What does `ScoredDoc.score` mean for each provider?**

- **Qdrant**: the engine's native score from `search` (cosine similarity interpretation depends on `Distance` mode — callers should treat it as a relative ranking key unless they calibrate thresholds per model).
- **Pinecone**: SDK `match.score` (cosine / dot per index metric configuration).
- **Weaviate**: derived as `1 / (1 + distance)` from `_additional { distance }` so higher is better, aligning loosely with cosine-style UX for UI display.

Retrieval and evaluation layers should normalise scores if they combine multi-source results.

---

## P2-5 · Retrieval Service

### Role in the pipeline

**Q183: What problem does `RetrievalService` solve that `VectorStoreService.search` alone does not?**

Dense `search` returns a single ranked list by vector similarity. Real pipelines need **diversity** (MMR), **lexical recall** (BM25 hybrid), **multiple query embeddings** (multi-query RRF), **strategy ensembles**, **parent/child chunk uplift**, and **cross-encoder reranking**. `RetrievalService` orchestrates those steps, normalises fusion scores where needed, and keeps the vector store client focused on I/O.

---

**Q184: Why is BM25 implemented in-process instead of using Qdrant sparse vectors in P2-5?**

P2-4 indexed only dense vectors. Building a production sparse index inside Qdrant (or OpenSearch) is a separate migration. For P2-5 we still need a **correct hybrid story** for Designer/Autopilot configs: an in-memory `BM25Index` over the same chunk texts the caller used at index time gives deterministic fusion tests and works for moderate corpora. A later phase can swap the sparse leg for native store sparse search without changing the `retrieve(..., sparse_corpus=...)` contract.

---

**Q185: How does hybrid fusion work (`rrf` vs `weighted`)?**

- **RRF (`reciprocal_rank_fusion_keys`)**: builds two orderings — dense hits and BM25 top‑`k` — keyed by normalised `page_content`, then sums `1/(k+rank)` across lists so documents strong in both channels rise without fragile score-scale alignment.
- **Weighted (`weighted_dense_sparse`)**: min-max normalises dense and BM25 scores per corpus index, then blends with `hybrid_search_alpha` as dense weight (mirrors `HybridSearchConfig.alpha` in the pipeline schema).

---

**Q186: Why does MMR require `embedding_service` on `RetrievalService`?**

MMR needs a vector per candidate chunk to measure redundancy against already selected chunks. Qdrant search results in P2-4 do not return vectors by default. The service **re-embeds** the text of each fetched candidate (same `EmbeddingService` as ingestion) so MMR uses the same model space as the index. If no embedder is configured, the service logs a warning and falls back to top‑`k` dense order.

---

**Q187: How is parent-child retrieval represented without a second index?**

Callers index **child** chunks (small, precise) with metadata such as `parent_id` and optional `parent_page_content`. `retrieve` with `strategy="parent-child"` runs dense search on children, then `_parent_child_uplift` collapses hits by `parent_id`, keeping the best score and swapping in `parent_page_content` when present so the LLM sees expanded context.

---

**Q188: What is the contract for multi-query retrieval?**

Either pass `multi_query_vectors` (aligned with extra variants) or set `RetrievalRuntimeConfig.multi_query_variants` **and** inject `embedding_service` so the service can `embed_query` each variant. Each vector triggers a dense search; rankings are fused with RRF over `page_content` keys. If variants are missing, the service logs and falls back to plain similarity.

---

**Q189: How does ensemble mode avoid infinite recursion?**

`_ensemble` iterates `cfg.ensemble_strategies` and builds a fresh `RetrievalRuntimeConfig` per sub-strategy with `strategy=name`. The dispatcher never calls `ensemble` from within those sub-calls, so the graph is acyclic. Unknown or unsupported members (e.g. `hybrid` without `sparse_corpus`) are skipped with a structured log.

---

**Q190: How is Cohere reranking wired, and what happens if the API fails?**

`RerankingRuntimeConfig` with `enabled=True` and provider/model implying Cohere instantiates `CohereReranker`, which POSTs to `https://api.cohere.ai/v1/rerank` via async `httpx`. Catalog IDs like `cohere-rerank-v3` map to API model names. On any exception, the service logs `cohere_rerank_failed` and falls back to the original order slice so retrieval never hard-fails for reranker outages.

---

**Q191: What does `retrieval_runtime_from_pipeline` do?**

Routers validate `RetrievalConfigSchema` (Pydantic). Core services prefer dataclasses. The bridge converts strategy enum to string, copies `top_k` / `score_threshold`, maps `MetadataFilter` rows to `VectorSearchFilter`, and copies hybrid `alpha`. It keeps `app/core/retrieval` importable without circular schema dependencies in the hot path beyond this one helper.

---

**Q192: How do you unit-test retrieval without paying for embeddings or Cohere?**

Use Qdrant `:memory:` plus `VectorStoreService`, inject a `MagicMock` for `EmbeddingService.embed` / `embed_query` for MMR, and `monkeypatch` `httpx.AsyncClient` for reranker tests — the suite in `tests/test_core/test_retrieval.py` follows this pattern.

---

**Q193: Why did `.gitignore` switch from `docs/internal/` to `docs/internal/*` with negated paths?**

Ignoring a directory wholesale (`docs/internal/`) prevents git from tracking *any* file inside it, including specs you want versioned. The pattern `docs/internal/*` ignores unknown internal artefacts by default while `!docs/internal/TASK_BASED_INTERVIEW_Q&A.md` (and similar) re-includes selected files. The same pattern applies to `prompt_history/*` with `!prompt_history/prompt_history.md` so session logs can be committed without whitelisting arbitrary scratch files.

---

## P2-6 · Generation Service

### Role in the pipeline

**Q194: What does `GenerationService` add that calling a raw LLM API does not?**

It standardises **RAG prompt assembly** (numbered context blocks, optional source lines from metadata, default faithfulness-oriented system prompt), accepts **`Document` or `ScoredDoc`** so retrieval output plugs in without manual unwrapping, maps **`GenerationConfigSchema` → `GenerationRuntimeConfig`** for parity with P1-3, and returns a **`GenerationResult`** with normalised text plus `finish_reason` / token usage when the chat model exposes them. Routers and agents get one orchestration surface instead of duplicating provider-specific message formats.

---

**Q195: How does multi-provider support work without exploding `if/elif` in the service?**

`create_chat_model` in `factory.py` centralises provider wiring: LangChain `ChatOpenAI`, `ChatAnthropic`, `ChatGoogleGenerativeAI`, `ChatCohere` (langchain-community), Mistral via **OpenAI-compatible** client + fixed Mistral base URL, and **meta/custom** via configurable OpenAI-compatible base URL + API key (Together, vLLM, local Llama). `GenerationService` only calls `ainvoke` / `astream` on the returned `BaseChatModel`, so new providers are factory changes, not service rewrites.

---

**Q196: Why are API keys read from `Settings` instead of the pipeline JSON?**

Pipeline configurations are stored and exported (Designer, templates) — they must stay **secret-free**. Credentials belong in environment-backed `Settings` (`OPENAI_API_KEY`, `MISTRAL_API_KEY`, `OPENAI_COMPATIBLE_*`, etc.) so the same saved pipeline can run in dev, staging, and production with different keys.

---

**Q197: How is JSON output mode implemented across providers?**

For **OpenAI**, `output_format == "json"` sets `model_kwargs={"response_format": {"type": "json_object"}}` on `ChatOpenAI` where the model supports it. For **Anthropic, Google, Cohere**, there is no single portable JSON mode in the factory; `build_rag_user_message` appends an instruction to emit **valid JSON only** (no fences). That keeps behaviour consistent enough for evaluation and UI while remaining honest about provider capabilities.

---

**Q198: What is the difference between `generate()` and `stream()`?**

`generate()` awaits a full `AIMessage` and returns `GenerationResult`. `stream()` wraps the model’s `astream` and yields text chunks for future **SSE** endpoints or progressive UI rendering, using the same prompts and context assembly.

---

**Q199: How does the service handle `ScoredDoc` from retrieval?**

`_normalize_context` detects `ScoredDoc` instances and uses `.document` only; similarity scores are intentionally **not** injected into the LLM prompt by default (they are not calibrated across providers). If product requirements change, a later iteration could add optional score-prefixed lines in `format_context_block`.

---

**Q200: Why does Google use `max_output_tokens` while OpenAI uses `max_tokens`?**

LangChain’s `ChatGoogleGenerativeAI` follows the Gemini API naming (`max_output_tokens`). The factory maps `GenerationRuntimeConfig.max_tokens` to that parameter explicitly instead of reusing the shared `_common_kwargs` helper used for OpenAI/Mistral.

---

**Q201: How would you unit-test generation without calling real LLMs?**

Patch `create_chat_model` to return an `AsyncMock` whose `ainvoke` returns a fixed `AIMessage`. The tests in `tests/test_core/test_generation.py` follow this pattern and assert prompt assembly / bridge logic separately from network I/O.

---

**Q202: Where does `generation_runtime_from_pipeline` sit in the architecture?**

Same role as `retrieval_runtime_from_pipeline`: at the **router boundary**, validate with Pydantic (`GenerationConfigSchema`), then convert enums and optional fields to plain strings and dataclasses the core layer can import without tight coupling to FastAPI or JSON schema evolution.

---

**Q203: What operational risk remains after P2-6 for production chat?**

**Context length**: retrieved chunks are concatenated without a tokenizer-based budget; long corpora can exceed model context. Mitigations (truncate by score, summarise, or map-reduce) belong in a later phase or router policy. **Streaming back-pressure** and **rate limits** are not handled inside the service — they belong in middleware or API gateways.

---

**Q204: Why expose Mistral via OpenAI-compatible `ChatOpenAI` instead of a dedicated SDK?**

The repo already depends on `langchain-openai`. Mistral’s REST API is OpenAI-compatible for chat completions; a dedicated package adds install surface without changing the `BaseChatModel` contract. If Mistral-specific features are required later, swapping the factory branch is localized.

---

## P2-7 · Evaluation Engine

### Role and data contract

**Q205: What does the Evaluation Engine consume, and what does it produce?**

It consumes a batch of ``EvaluationExample`` rows: ``question``, ``answer`` (model output), ``contexts`` (list of retrieved strings), and ``ground_truth`` (reference). It returns ``EvaluationEngineResult`` with aggregate ``EvaluationMetrics`` (faithfulness, answer relevance, context precision/recall, average wall-clock latency per query), optional ``FailureAnalysisResult`` from per-row scores, and ``per_row_scores`` for UI drill-down. It does **not** persist runs to Postgres — that remains the responsibility of the future evaluation API router (P4/P8).

---

**Q206: Why use RAGAS instead of hand-written metric code?**

RAGAS implements research-backed LLM-as-judge and embedding-based metrics with consistent scoring semantics, batching, and async execution. Reimplementing faithfulness or context recall would duplicate maintenance and diverge from what Autopilot and industry tooling expect. The engine wraps ``ragas.evaluate`` and maps RAGAS column names (e.g. ``answer_relevancy``) to API schema names (``answer_relevance``).

---

**Q207: How is the HuggingFace ``Dataset`` built, and why pad empty contexts?**

``build_dataset`` aligns columns with RAGAS defaults: ``question``, ``answer``, ``contexts`` (list of strings per row), ``ground_truth``. Empty context lists are replaced with a placeholder string so metrics that assume at least one context do not fail validation or throw during scoring.

---

**Q208: How does `metric_names_from_pipeline` interact with `EvaluationConfigSchema`?**

If evaluation is disabled or the schema is absent, it returns ``None`` and the engine runs the **default** four RAGAS metrics. If enabled with an explicit ``metrics`` list, pipeline names (including ``latency``) are filtered: ``latency`` is measured as wall-clock time around ``ragas.evaluate`` and is not passed to RAGAS. Unknown metric names raise ``ValueError`` at resolve time.

---

**Q209: How does failure analysis work without a second LLM call?**

``analyze_failures`` applies deterministic thresholds to per-row scores (faithfulness, context precision/recall, answer relevancy) and buckets examples into ``hallucination``, ``retrieval_quality``, ``context_gap``, or ``format_error`` (empty answer). It produces counts, up to five example questions per category, and short recommendations. This is a **heuristic** triage layer; deeper root-cause LLM analysis can be added in the Autopilot Evaluation Agent later.

---

**Q210: What does `compare_metrics` optimize for?**

It compares two ``EvaluationMetrics`` instances field-by-field (higher is better for the four RAGAS aggregates; **lower** ``avg_latency_ms`` wins when both sides present latencies). It emits ``MetricDelta`` rows and an overall winner by simple vote count across compared fields — suitable for A/B dashboards, not statistical significance testing.

---

**Q211: What is the purpose of `examples_from_documents` if answers are empty?**

It bootstraps **skeleton** evaluation rows from chunk text (``ground_truth`` excerpt + generic question). Callers must fill ``answer`` and ``contexts`` after running retrieval + generation. It exists for tests and for agents that will wire the full loop; it is not a substitute for LLM-based question generation.

---

**Q212: Why lazy-import RAGAS inside `EvaluationEngine.evaluate`?**

Importing ``ragas.evaluate`` inside the method keeps optional heavy imports off the critical path for unrelated tests and allows unit tests to ``patch("ragas.evaluate", ...)`` without importing the full metric stack at module import time.

---

**Q213: How are OpenAI models for RAGAS configured?**

``Settings.evaluation_llm_model`` and ``evaluation_embedding_model`` default to ``gpt-4o-mini`` and ``text-embedding-3-small``; they use ``OPENAI_API_KEY`` like the rest of the stack. Override via environment (Pydantic field names uppercased with underscores).

---

**Q214: What happens if `result.to_pandas()` fails during failure analysis?**

The engine logs ``failure_analysis_skipped`` and still returns aggregate metrics. Partial degradation avoids losing an entire evaluation run because of a pandas merge or column mismatch after a RAGAS upgrade.

---

**Q215: How would you production-harden this beyond P2-7?**

Add token-budgeted context truncation before building the dataset, persist runs via ``EvaluationRun`` ORM, move long jobs to Celery (P2-8), add idempotency keys for compare endpoints, wire cost estimation into ``cost_per_query``, and optionally run RAGAS with ``in_ci=True`` for more reproducible scores when needed.

---

## P2-8 · Celery Worker & Task Queue

### Architecture & broker

**Q216: Why introduce Celery when FastAPI already runs async endpoints?**

``async`` handlers keep I/O‑bound waits off the worker thread pool, but **CPU-heavy** work (embedding batches), **minute-scale** orchestration loops (future LangGraph Autopilot), and calls that **cannot be cancelled mid-flight easily** still block an API process once they start. Celery executes those units in dedicated worker processes sourced from Redis, so the FastAPI/Uvicorn event loop stays shallow: enqueue a task, return ``taskId``, and poll or subscribe separately.

---

**Q217: Why is Redis both broker and result backend in this codebase?**

The stack already depended on Redis for caching and Compose wiring. Celery happily uses Redis as a broker (FIFO lists) **and** a result store (serialized task return values keyed by Celery IDs). Dedicated brokers (e.g., RabbitMQ) add operational surface; Redis is adequate for MVP throughput until message durability / priority semantics demand an AMQP topology.

---

**Q218: Why does the Celery bootstrap import path use ``celery -A app.worker:celery_app worker`` rather than importing ``celery_app.py`` alone?**

The ``tasks`` module must register ``@celery_app.task`` handlers on the singleton application object. Loading only ``celery_app`` without importing ``tasks`` would start a worker **with zero registered tasks**. Importing ``app.worker`` executes ``package __init__.py``, which pulls ``celery_app`` then imports ``tasks`` as a deliberate side-effect, guaranteeing registration before worker boot.

---

**Q219: Where does synchronous SQLAlchemy enter the picture if the API uses asyncpg?**

FastAPI resolves DB access through ``asyncpg`` and ``AsyncSession``. Celery task bodies are synchronous coroutine-free Python; bridging them through ``nest_asyncio`` or ad-hoc event loops inside workers is fragile. The Settings therefore expose ``database_url_sync``, rewriting ``postgresql+asyncpg`` → ``postgresql+psycopg`` (and ``sqlite+aiosqlite`` → ``sqlite``) so workers open a conventional sync engine/session with ``sync_session_scope()``.

---

**Q220: Enumerate each implemented task name and payload contract.**

| Task | Purpose | Key arguments |
|------|---------|---------------|
| ``jobs.run_pipeline_build`` | Advance ``AutopilotBuild`` rows via **stub stages** pending LangGraph Phase 6 | ``build_id: str`` UUID |
| ``jobs.run_evaluation`` | Run ``EvaluationEngine.evaluate`` offline and persist aggregates on ``EvaluationRun`` | ``evaluation_run_id``, ``examples`` (list of dicts), optional ``metric_names`` |
| ``jobs.run_deployment`` | Simulate deployment completion for ``Deployment`` rows | ``deployment_id`` UUID |

---

**Q221: Why pass evaluation **examples inline** rather than storing them purely in Postgres beforehand?**

The ``EvaluationRun`` ORM persists status and outputs, not necessarily the curated test corpus (that arrives via future evaluation APIs). The worker task therefore accepts serialized rows in the enqueue payload—the same eventual shape Autopilot emits after retrieval + generation—so routers can hydrate runs without premature schema churn. When persistence hardens (Phase 8), examples can migrate server-side exclusively.

---

**Q222: Why do job responses serialize ``task_id`` JSON field as camelCase (`taskId`)?**

``RAGBaseModel`` configures ``alias_generator=to_camel``. Client surfaces (Designer/Autopilot) expect camelCase payloads; job endpoints deliberately reuse those schema bases for consistency—even though Celery internals still talk about ``task_id`` strings.

---

**Q223: Which HTTP endpoints glue the API surface to Celery today?**

- ``POST /api/jobs/build/{build_id}`` → ``run_pipeline_build.delay``
- ``POST /api/jobs/evaluation`` body → ``run_evaluation.delay``
- ``POST /api/jobs/deployment/{deployment_id}`` → ``run_deployment.delay``
- ``GET /api/jobs/tasks/{task_id}`` unwraps ``AsyncResult`` state/result/metadata for polling UIs ahead of SSE (Phase 7).

Full domain routers defer to Phase 6/8; these routes are provisional worker plumbing.

---

**Q224: Explain ``task_always_eager``.**

``Settings.celery_task_always_eager`` maps to Celery ``task_always_eager`` so tests or smoke scripts execute tasks synchronously inside the publisher process **without Redis**. Useful for deterministic CI subsets; unacceptable for production parallelism.

---

### Reliability / ops interview angles

**Q225: Why ``task_acks_late=True`` paired with ``worker_prefetch_multiplier=1``?**

Late ack acknowledges a message **after** completion, preventing loss if the worker crashes mid-task (the broker redelivers). Prefetch 1 avoids one greedy worker hoarding queued build jobs while others idle—a common Celery starvation pattern during uneven job durations.

---

**Q226: How would you observe stuck Autopilot builds after this phase?**

Check Redis queue depth, Celery Flower (optional addon), structured logs keyed by ``build_id``, and Postgres ``autopilot_builds.status/current_stage``. The stub task emits ``structlog`` events like ``build_complete_stub`` once messages append and stages mutate.

---

**Q227: How should secrets differ between ``api`` and ``worker`` containers?**

They share the Compose environment block intentionally: both need LLM credentials for evaluations, Postgres for ORM mutations, Redis for broker/back-end parity, and eventual object storage URLs. Separation via IAM roles enters Phase 12.

---

### Trade-offs vs alternatives

**Q228: When would you pick RQ / Dramatiq / Arq instead of Celery?**

RQ/Dramatiq reduce boilerplate for Redis-only fleets; Celery trades minimalism for **ecosystem depth**—retries, chains, canvases—needed before Autopilot’s multi-stage orchestrations land. Migrating remains feasible because task bodies already isolate sync DB boundaries.

---

### ``.gitignore`` / hygiene

**Q229: Did P2-8 introduce new artefacts to ignore?**

Optional local ``celerybeat-schedule`` files if beat is enabled later; current diff only documents env vars in ``.env.example``. No extra ignore entries were mandatory because workers run inside Docker without writing beat schedules into the repo root by default.

---

## P2-9 · Health & Utility Endpoints

### Probes and middleware

**Q230: Why split ``/health``, ``/health/live``, and ``/health/ready``?**

Legacy callers and Docker ``HEALTHCHECK`` continue to hit ``GET /health`` for a trivial liveness JSON. ``/live`` is an explicit **liveness** shim (process up). ``/ready`` runs **readiness** probes: today it checks PostgreSQL, Redis, and Qdrant with short timeouts and returns **503** if any check fails so Kubernetes or load balancers can stop routing traffic before the app is wired to its dependencies.

---

**Q231: Why does ``/health/ready`` skip Redis and Qdrant when ``APP_ENV=test``?**

Pytest sets ``APP_ENV=test`` and must not block on daemons that CI may not start. If readiness used shared ``Depends(get_redis)`` providers, FastAPI would **resolve dependencies before the route body**, potentially opening TCP connections or hanging. The handler short-circuits in test mode after the DB session resolves (SQLite in tests), returning ``200`` with a ``skipped`` explanation.

---

**Q232: Why open ephemeral Redis/Qdrant clients inside ``/ready`` instead of reusing ``get_redis``?**

Module-level singletons are convenient for request paths that always need cache/vector access, but readiness must avoid side effects during test collection and must not poison a shared pool when a probe fails mid-startup. Fresh, small clients with ``aclose()`` / ``close()`` in ``finally`` isolate probe traffic.

---

**Q233: What behaviour do you expect for ``X-Request-ID``?**

If the inbound request already sets ``X-Request-ID``, middleware preserves it, binds it into ``structlog`` context for the request log line, and echoes it on the response. If absent, middleware generates a UUID v4, binds it, and sets the response header so downstream gateways and support tickets can correlate logs.

---

### Utilities surface

**Q234: Which utility routes exist and what are they for?**

| Route | Role |
|-------|------|
| ``GET /api/utilities/info`` | Service name, semver, ``APP_ENV``, Python version for dashboards and support |
| ``POST /api/utilities/validate-pipeline`` | Run ``PipelineConfigurationSchema.model_validate`` without persisting — returns ``valid`` + Pydantic ``errors`` list (HTTP **200** even when invalid so UIs can render field messages) |
| ``POST /api/utilities/cost`` | Accepts ``CostRequest`` (same shape as future ``POST /api/designer/cost``), returns ``CostEstimateSchema`` |

---

**Q235: Why return HTTP 200 on validation failure instead of 422?**

Designer flows often want a **single response shape** (`valid` + structured errors) across paste/import UX without treating validation as an exceptional HTTP layer. APIs that prefer strict FastAPI behaviour can still POST the same body to a future strict endpoint; this utility is explicitly tolerant.

---

### Cost estimation

**Q236: Where does pricing data load from?**

``load_pricing`` tries, in order: ``Settings.pricing_catalog_path`` (``PRICING_CATALOG_PATH``) if set, then ``apps/api/catalogs/pricing.json`` (bundled for Docker build context), then repo-root ``data/pricing.json``. Missing files raise ``PricingLoadError`` mapped to **503** on ``/cost``.

---

**Q237: How does ``CostEstimator`` relate to ``costCalculatorFormulas`` in ``pricing.json``?**

Per-query **embedding** uses ``(topK * chunkSize / 1e6) * embeddingCostPer1M`` (pipeline ``retrieval.top_k`` and ``chunking.chunk_size``). **Generation** mirrors ``contextTokens`` = retrieved context tokens + avg input tokens from assumptions plus output token pricing. **Reranking** adds ``costPer1KQueries / 1000`` when reranking is enabled. **Monthly** aggregates per-query totals times ``queriesPerMonth`` plus **storage** from vector dimensions and token/chunk estimates, priced via managed-cloud or serverless rows when present, and optional Pinecone **read-unit** style charges surfaced under ``retrieval_ops``.

---

**Q238: Does the estimator amortise one-off corpus indexing separately?**

The shipped formulas in ``pricing.json`` express monthly variable traffic plus storage without a discrete “bulk re-embed cadence”; the implementation follows that split so Phase 4 designer polish can refine amortisation assumptions without breaking the catalogue contract.

---

**Q239: Where is ``API_SEMVER`` defined and why extract it from ``main.py``?**

``app/metadata.py`` exports ``API_SEMVER`` consumed by FastAPI ``version=``, ``/health``, ``/health/ready``, and ``/api/utilities/info`` so probes and docs never drift.

---

### Operations

**Q240: Should production readiness require Redis **and** Qdrant strictly?**

Current policy: **all** probes must succeed before ``200``. That matches “full stack warming” deployments. SaaS tenants that treat Redis/Qdrant as optional could split readiness tiers later (database-only readiness vs extended checks).

---

**Q241: Did P2-9 require `.gitignore` updates?**

No new generated artefacts landed in the repo root; ``catalogs/pricing.json`` is committed intentionally so the slim ``apps/api`` Docker context stays self-contained.

---

## Phase 3 onward (milestones P3-3+)

> Starting at **P3-3**, new sections follow the Phase 3 milestone naming used in `TASKS.md`. Everything above remains the archived **P0–P2** interview material.

---

## Phase 3 · P3-3 — App layout & navigation

### Why split `Providers`, `AppShell`, and `layout.tsx`?

**Answer:** Next.js **server** `layout.tsx` should stay thin: fonts, metadata, and HTML shell. **React Query** must live in a **client** component (`providers.tsx`) because hooks cannot run in server components. **`AppShell`** is client-side because it uses pathname-aware UI (sidebar visibility, mobile drawer). **`StoreHydration`** stays separate so Zustand `persist` rehydration runs once at startup.

### Why is React Query added before API-heavy screens exist?

**Answer:** Phase 4+ will mount hooks against Projects and Designer APIs. Providing `QueryClientProvider` early avoids refactors and establishes defaults (`staleTime`, no aggressive refetch on focus).

### How does mode switching (Designer vs Autopilot) work?

**Answer:** **`ModeToggle`** uses `usePathname()` and `<Link>` to `/designer` and `/autopilot`. Active state is derived from the URL prefix, not duplicated global state—single source of truth for navigation.

### Why does the sidebar disappear on the home page?

**Answer:** Marketing/landing typically stays **full-bleed** without project chrome. All other primary routes show **Navbar + Sidebar** so project context stays visible during configuration workflows.

### How is the sidebar collapsible and persisted?

**Answer:** Desktop collapse toggles width (`md:w-14` vs wider panel). State is stored under **`rag-studio-sidebar-collapsed`** in `localStorage` so preference survives refreshes. Mobile uses **overlay + backdrop** and closes on route change.

### How does the project dropdown relate to the sidebar?

**Answer:** Both read **`useProjectStore`** (Zustand + persist). The navbar dropdown selects **`activeProjectId`**; the sidebar lists projects and supports **“New project”** via `addProject`. Until **P4-1 Projects API**, data is **local-only**.

### Why Radix `DropdownMenu` without shadcn wrappers?

**Answer:** This milestone prioritized shell behaviour over regenerating shadcn primitives. Radix is already a dependency; dropdown triggers remain keyboard-accessible and portal-rendered.

### What is the avatar placeholder?

**Answer:** **No auth yet** (Phase 12). “RS” initials mark reserved space for future user menu.

### What do `not-found.tsx` and `error.tsx` accomplish?

**Answer:** **`not-found`** renders for unknown routes with recovery links. **`error.tsx`** is a **client** boundary (required by Next.js) that logs errors and offers **`reset()`** plus navigation home.

### Font strategy: task mentions Geist—what shipped?

**Answer:** **`next/font/google`** loads **Inter** and **JetBrains Mono** with CSS variables **`--font-geist-sans`** and **`--font-geist-mono`** so Tailwind `font-sans` / `font-mono` align with the design token names. Teams may swap to the **`geist`** npm package later without changing Tailwind wiring.

### How would you test navigation in CI?

**Answer:** Component tests for **`ModeToggle`** active classes (mock `usePathname`), and Playwright smoke for `/` → `/designer` link (Phase 10). Until Vitest is wired, manual QA paths suffice for this branch.

---

## Cross-cutting (layout-adjacent)

### Where should shared loading UI live?

**Answer:** Route-level **`loading.tsx`** or shared **`LoadingSpinner`** under `components/shared/` (P3-1). Global skeleton policy is product-specific.

### How does hydration interact with the shell?

**Answer:** **Zustand persist** skips hydration on SSR; **`StoreHydration`** calls `persist.rehydrate()` on mount so navbar/sidebar see projects after client loads. **`suppressHydrationWarning`** on `<html>` avoids noise if theme toggles appear later.

---

## Behavioural / scenario questions

### “A user reports the sidebar won’t open on mobile.”

**Answer:** Verify **`showSidebarTrigger`** is true (not on `/`), confirm **`openSidebar`** wires to state toggling **`mobileSidebarOpen`**, and check z-index conflicts with other overlays.

### “Should Designer store drive which tab is active in ModeToggle?”

**Answer:** Prefer **URL as source of truth** for mode to avoid desync when opening links in new tabs or sharing URLs.

---

*Extend this file after each phase with new Q&A blocks keyed by task ID.*

---

## Phase 3 · P3-4 — Landing Page

### Why are the landing sections split into individual components instead of one big `page.tsx`?

**Answer:** Each section (`Hero`, `ModeComparison`, `HowItWorks`, `Features`, `UseCases`, `Pricing`, `CTA`) has distinct responsibilities and different data shapes. Splitting them keeps each file under ~100 lines, makes A/B testing individual sections trivial, and lets future phases swap or reorder sections without touching the assembly file.

### Why are all landing components server components (no `'use client'`)?

**Answer:** None of the landing sections need browser APIs or React state — they are purely presentational. Keeping them as RSC means zero client-side JS for these sections, faster initial page load, and better Lighthouse scores. The animated gradient orbs are CSS `@keyframes`, not JavaScript animations.

### How does the Hero animated gradient work without framer-motion?

**Answer:** Two absolutely-positioned `div`s with `blur-3xl` and `rounded-full` act as "orbs". They animate via custom `@keyframes float` added to `globals.css`, referenced by Tailwind utility classes `animate-float` and `animate-float-delayed` (added in the `@layer utilities` block). No JavaScript is involved — the animation is pure CSS.

### Why use `pointer-events-none` on the hero orbs?

**Answer:** The orbs are decorative overlays. Without `pointer-events-none` they would intercept mouse events on the CTA buttons below them (even though they are visually behind content due to `z-index`). The class ensures clicks always reach the intended interactive elements.

### How is the `gradient-text` utility reused across Hero and other components?

**Answer:** `gradient-text` is defined once in `globals.css` as a `@layer utilities` rule that applies `bg-clip-text`, `text-transparent`, and the linear-gradient. All landing sections that need the brand gradient on text simply add `className="gradient-text"` — no duplication.

### What design decisions were made for the Pricing tier cards?

**Answer:** Three tiers — Free, Pro, Enterprise — follow a standard SaaS pattern. The `popular` flag drives the visual highlight (primary border, gradient background, "Most Popular" pill). Feature rows use a discriminated `included: boolean | 'partial'` field: `false` renders a muted `Minus` icon, `true` renders a green `Check`, and `'partial'` renders `Check` with an inline note (e.g., "3 / month"). This avoids a separate "partial" icon and keeps the data model simple.

### Why is `Minus` from lucide-react used instead of an X icon for missing features in Pricing?

**Answer:** A `Minus` (em-dash style) is a softer visual signal than a red `X`. Red crosses create anxiety on marketing pages. The neutral grey `Minus` clearly communicates unavailability without penalizing the plan visually, which is a common SaaS conversion best practice.

### How does the `ModeComparison` section differ from the `HowItWorks` section?

**Answer:** `ModeComparison` answers "what does each mode do?" — it shows features, tone, and a CTA. `HowItWorks` answers "how do I use each mode?" — it shows numbered sequential steps. Separating the two prevents information overload in a single section and lets users who already understand the modes skip directly to the how-to.

### What persona archetypes are in `UseCases` and why those three?

**Answer:** **Learning Engineer** (uses Designer to understand RAG internals), **Time-Strapped Startup** (uses Autopilot for speed), and **Enterprise Architect** (uses both for compliance + validation). These map to the three primary segments in CLAUDE.md's target users: ML Engineers, AI Teams/Startups, and Enterprises. Each persona has a quote, a recommended mode tag, and a benefit list to make the value proposition concrete.

### How does the CTA section's color scheme reinforce the brand?

**Answer:** The CTA uses `bg-gradient-to-br from-primary-600 via-primary-700 to-purple-700`, matching the brand gradient direction established in `gradient-brand` and `gradient-text`. This creates visual consistency: the first and last sections both carry the brand gradient, creating a "bookend" effect that frames the entire page.

### What is the `page.tsx` assembly pattern and why not import everything inline?

**Answer:** `page.tsx` is a pure orchestration file — it imports and renders the 7 section components in order. No markup lives there. This mirrors Next.js App Router idioms where `layout.tsx` orchestrates shell and `page.tsx` orchestrates content sections. It also makes it trivial to reorder, remove, or feature-flag sections (e.g., hide `Pricing` for a beta period) without touching the section code.

### Why does the footer live in `page.tsx` rather than a dedicated `Footer.tsx` component?

**Answer:** The footer is intentionally minimal (3 lines of text). At this stage it does not justify its own component — the TASKS.md spec does not list a `Footer.tsx`, and adding premature abstractions conflicts with the project principle of not designing for hypothetical future requirements. If the footer grows (nav links, social icons, legal pages), extracting it then is straightforward.

### How would you test the landing page in CI?

**Answer:** Playwright `e2e/landing.spec.ts` (Phase 10) will: (1) verify the page loads at `/`, (2) assert the "Start Designing" CTA links to `/designer`, (3) assert "Launch Autopilot" links to `/autopilot`, and (4) check that no sidebar renders on `pathname === '/'`. Until Playwright is wired, `tsc --noEmit` and Next.js build serve as correctness gates.

---

## Phase 3 · P3-5 — Lib Utilities & Validators

### Why create Zod validators separately from TypeScript types?

**Answer:** TypeScript types are erased at compile time — they only verify static correctness. **Zod schemas** are runtime constructs that validate unknown data at system boundaries (form submissions, API responses, localStorage deserialization). Keeping them separate from types allows the type system and validation logic to evolve independently. The `z.infer<>` utility then derives TypeScript types from the Zod schema when needed, eliminating drift.

### How does `ChunkingConfigSchema` enforce the overlap-less-than-size invariant?

**Answer:** The schema adds a `.refine()` predicate after the base `z.object()` definition: `(data) => data.chunkOverlap < data.chunkSize`. Zod's `.refine()` runs post-field validation, receives the parsed object, and can produce a targeted error message on the `chunkOverlap` path. This cross-field constraint cannot be expressed in TypeScript's type system alone.

### Why does `RetrievalConfigSchema` require `hybridSearch` only when strategy is `hybrid`?

**Answer:** This is a **discriminated validation** — the field is contextually required. Using `.refine()` with a path on `hybridSearch`, the schema rejects configs where `strategy === 'hybrid'` but `hybridSearch` is absent. This enforces invariants that the TypeScript union types cannot express, keeping the backend from receiving an underspecified retrieval config.

### How does `generateMermaidDiagram` avoid injecting user content into the diagram syntax?

**Answer:** All user-supplied strings (model names, index names, source types) are passed through a `q()` sanitiser that strips `"`, `[`, and `]` characters before embedding them in node labels. Mermaid flowchart node labels that contain unescaped brackets or quotes break the diagram parser. The sanitiser is intentionally minimal — it strips the specific characters that Mermaid treats as syntax.

### Why does the Mermaid generator use two sub-graphs (indexing path and query path)?

**Answer:** RAG pipelines have two distinct flows: the **offline indexing path** (document → chunking → embedding → vector store) and the **online query path** (query → retrieval → optional reranking → generation → answer). Separating them into sub-graphs makes the architecture immediately legible and mirrors how engineers mentally model RAG systems. Using a single flat graph would create a confusing tangle of arrows.

### What is the LCEL pattern in the Python code generator output?

**Answer:** **LangChain Expression Language (LCEL)** composes runnables with the `|` pipe operator: `retriever | format_docs | prompt | llm | parser`. The generated code uses `RunnableParallel` to run context retrieval and the passthrough question in parallel, then feeds both into the prompt template. LCEL enables streaming, batching, and tracing out of the box without custom orchestration code.

### How does the Python code generator handle different vector store backends?

**Answer:** An internal `VECTORSTORE_IMPORTS` dictionary maps provider IDs (`qdrant`, `pinecone`, `chroma`, etc.) to provider-specific import strings and initialization code. The `buildVectorStore()` function uses a switch/case on `stages.vectorStore.provider` to emit the correct connection setup (e.g., `QdrantClient` + URL for Qdrant, `Pinecone()` + `pc.Index()` for Pinecone). Unrecognised providers emit a `# TODO` stub so the file is still syntactically valid.

### Why does the YAML generator avoid a third-party serialisation library?

**Answer:** Third-party YAML libraries (like `js-yaml`) would add a dependency and produce output we can't control (e.g., quoting strategy, key ordering). Since the pipeline config is a known, bounded shape, hand-building the YAML with helper functions (`yamlString`, `yamlBool`, `yamlArray`) gives us full control over formatting, comment placement, and indentation. It also removes a runtime dependency from the browser bundle.

### How does `yamlString` decide when to quote a value?

**Answer:** The helper tests for characters that YAML treats as special syntax (`:#[]{},|>&*!,`) and also checks for embedded newlines. If any are present, the value is wrapped in double quotes with internal double quotes escaped as `\"`. Plain scalar values (most model names and IDs) are emitted without quotes, producing cleaner YAML.

### What does the Terraform generator produce for multi-cloud configurations?

**Answer:** Multi-cloud falls back to **AWS** since Terraform providers are cloud-specific — there is no single "multi-cloud" provider. The raw cloud value `"multi-cloud"` is preserved in the header comment so the engineer knows which logical target it represents, but the generated HCL uses the AWS provider block, ECS, and Secrets Manager. Engineers building true multi-cloud deployments would extend the output to combine multiple provider blocks.

### How are the generator tests structured to balance specificity with maintainability?

**Answer:** Tests use **two complementary techniques**: (1) targeted string assertions (`expect(result).toContain(...)`) for stable structural properties — import paths, constant names, YAML keys — and (2) `toMatchSnapshot()` for the complete output. Targeted assertions catch regressions in specific features without over-constraining the full output. Snapshots catch unexpected global changes. If a snapshot fails after an intentional change, running `vitest --update-snapshots` regenerates it.

### Why are test fixtures (`fixtures.ts`) shared across all generator tests?

**Answer:** A `minimalConfig` and a `fullConfig` fixture defined once in `__tests__/fixtures.ts` eliminate duplication and ensure every generator is tested against the same known inputs. If the `PipelineConfiguration` shape changes, only the fixtures need updating — not every test file. The fixtures also double as documentation of the valid config surface area.

### How does snapshot testing complement unit assertions for the generators?

**Answer:** Unit assertions verify **individual properties** (e.g., "the output contains the model name"), but snapshots verify **the entire serialised output** at once. This is particularly valuable for YAML and Terraform where whitespace, ordering, and overall structure matter. The first run writes the snapshot to disk; subsequent runs diff against it. Intentional changes are accepted with `--update-snapshots`; accidental regressions fail CI.

### Why are generator functions pure (no side effects)?

**Answer:** Pure functions — those that take inputs and return a string without reading files, environment variables, or global state — are trivially testable (no mocking needed), cacheable, and safe to call from both server components and client-side export buttons. The generators are called from the frontend code export UI and from the backend export API; purity lets them work identically in both environments.

### Why install Vitest in P3-5 rather than waiting for P10-3 (Frontend Unit Tests)?

**Answer:** P3-5 explicitly includes "unit tests for all generators". Running those tests requires a test runner. Installing **Vitest** now is a one-line devDependency addition that does not conflict with P10-3 (which adds React Testing Library and full component coverage on top of the same Vitest runner). Waiting would leave the generator tests in the repo as untestable stubs, which defeats the purpose of writing them.

---

## Document maintenance (append-only policy)

> **2026-05-02:** These internal study documents keep **cumulative history**. New tasks add **`## Phase …`** sections at the **end**. Do not replace earlier phases wholesale unless fixing a factual error. Create a **new** file only when explicitly requested.

---

## Phase 4 · P4-1 · Projects API

### What endpoints exist for projects and what do they do?

- **`POST /api/projects/`** — Creates a project with `name` (required, trimmed) and optional `description`. Returns `201` with `ProjectSummary` (camelCase JSON). Rows are scoped with `user_id` from `X-User-ID` or `DEFAULT_USER_ID` / settings default.
- **`GET /api/projects/`** — Paginated list (`page`, `page_size`; capped by `max_page_size`). Returns non–soft-deleted projects for the resolved user only.
- **`GET /api/projects/{project_id}`** — Detail including summarized `pipelineConfigs` and `autopilotBuilds` (no full JSON blobs).
- **`PUT /api/projects/{project_id}`** — Partial update; requires at least one of `name` / `description`.
- **`DELETE /api/projects/{project_id}`** — Soft delete (`deleted_at` set to UTC “now”); returns `204` on success.

### How is “soft delete” implemented?

A nullable `projects.deleted_at` (`DateTime(timezone=True)`) column (migration `002_project_soft_delete.py`). All read/update paths filter `deleted_at IS NULL`. No hard `DELETE` in the service for this task.

### How do you scope data to a user before real authentication (P12)?

`get_request_user_id` in `app/dependencies.py` reads the **`X-User-ID`** header; if missing, it uses `Settings.default_user_id` (configurable; default is a fixed UUID). Invalid header values return `400`.

### Why did you use SQLAlchemy `JSON` instead of PostgreSQL `JSONB` in ORM models?

PostgreSQL-only `JSONB` cannot compile `CREATE TABLE` on SQLite, which the API test suite uses (`DATABASE_URL=sqlite+aiosqlite`). `sqlalchemy.JSON` maps to JSON/JSONB appropriately per dialect and enables `Base.metadata.create_all()` in pytest.

### Why `Uuid` instead of `postgresql.UUID`?

Cross-dialect UUID storage: PostgreSQL’s UUID type caused SQLite to return incorrect raw types on `refresh()`, breaking UUID coercion. `sqlalchemy.Uuid(as_uuid=True)` behaves consistently for SQLite and Postgres.

### How does `ProjectService` load detail with related entities?

`selectinload(Project.pipeline_configs)` and `selectinload(Project.autopilot_builds)` on the detail query; summaries map ORM rows to `PipelineConfigSummary` / `AutopilotBuildSummary` Pydantic models.

### What validates pagination limits?

`page_size` is validated against `Settings.max_page_size`; exceeding it yields `400`.

### Why camelCase in HTTP responses?

`RAGBaseModel` uses Pydantic `alias_generator=to_camel` for API JSON compatibility with typical frontend conventions.

### What integration tests cover projects?

`apps/api/tests/test_projects.py`: full CRUD + soft delete, user isolation (distinct UUIDs), invalid `X-User-ID`, page-size cap, empty PUT body. Tests assume camelCase response keys (`userId`, `pipelineConfigs`, etc.).

---

## Phase 4 · P4-2 · Designer Config API

### What endpoints manage Designer pipeline configurations?

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/designer/config` | Create a `pipeline_configs` row for an owned project (`SaveConfigRequest`: `name`, `projectId`, `config`, optional `description`). Returns `201` + `SaveConfigResponse`. |
| `GET` | `/api/designer/config/{config_id}` | Load full pipeline JSON (`PipelineConfigurationSchema`) merged with DB timestamps. |
| `PUT` | `/api/designer/config/{config_id}` | Partial update via `UpdateDesignerConfigRequest` (`name`, `description`, and/or full `config`). At least one field required. |
| `GET` | `/api/designer/configs` | Query **`project_id`** (required), optional `page`, `page_size` — lists summaries (`ConfigListResponse`). |
| `DELETE` | `/api/designer/config/{config_id}` | Hard-delete row (cascade to dependent evaluation/deploy rows per FK). `204` on success. |

### How does `DesignerService` enforce access control?

Every operation joins or resolves **`PipelineConfig` → `Project`** and requires `Project.user_id == X-User-ID` (resolved user) and `projects.deleted_at IS NULL`. Missing ownership yields **`404`** (no enumeration leak).

### Where is the canonical pipeline JSON stored?

Column **`pipeline_configs.config`** (`JSON`). Indexed columns **`name`, `version`, `cloud_provider`, `source`, `build_id`** mirror commonly queried fields from `PipelineConfigurationSchema` / metadata for SQL filtering and list rows without parsing full JSON.

### Why does `SaveConfigResponse` re-validate stored JSON?

Responses call `PipelineConfigurationSchema.model_validate` after injecting **`metadata.created_at` / `metadata.updated_at`** from row timestamps so clients always see authoritative persistence times even if the blob was stale.

### What changed on `SaveConfigRequest.project_id`?

Typed as **`uuid.UUID`** (JSON property `projectId`) instead of `str` for strict validation at the schema layer.

### What do integration tests cover?

`tests/test_designer.py`: happy-path CRUD + list, unknown project (`404` on create), user isolation, `page_size` cap vs `max_page_size`, empty `PUT` body (`400`).

---

## General — FastAPI + async SQLAlchemy

### How does the DB session commit?

`get_db_session` yields a session; on success it `commit()`s, on exception `rollback()`s.

### Where is the projects router registered?

`app/main.py` `create_app()` includes `projects_router` with prefix `/api/projects` (trailing-slash routes `GET/POST /` on the router).

### Where is the designer router registered?

`create_app()` includes `designer_router` from `app/routers/designer.py` with prefix `/api/designer` (`POST /config`, `GET /configs`, etc.).

---

## Phase 4 · P4-3 — Cost Calculation API

### What endpoint estimates Designer pipeline cost?

`POST /api/designer/cost` with body `CostRequest`: `config` (`PipelineConfigurationSchema`), optional `queries_per_month` (default 100_000), `documents_count`, `avg_document_tokens`. Response: `CostEstimateSchema` (`embedding`, `storage`, `retrieval`, `reranking`, `generation`, `total`, `per_query`, `per_month`, `currency`, `breakdown`).

### How does that differ from `POST /api/utilities/cost`?

Same request shape and pricing math; **utilities** is a cross-cutting preview, **designer** is the product endpoint for Designer mode (P5 cost UI will call `/api/designer/cost`). Both read the same `pricing.json`.

### Where is the calculator implemented?

`apps/api/app/utils/cost_calculator.py`: `CostEstimator.estimate(CostRequest)`, `calculate_cost`, `load_pricing`, `estimate_pipeline_cost`. Legacy import path `app.core.utilities.cost` re-exports the same symbols for P2-9 callers.

### What does `CostService` do?

Thin wrapper in `apps/api/app/services/cost_service.py`: loads pricing via settings, runs `CostEstimator`. Used by the designer router; no database access.

### How is embedding cost estimated?

Approximate tokens for the retrieval path: `top_k × chunk_size` (tokens) × embedding `$ / 1M tokens` from `pricing.json` → per query; multiplied by `queries_per_month` for monthly embedding line. **Multi-query** strategy multiplies this (and retrieval ops) by `num_variants`; **ensemble** / **hybrid** apply small multipliers.

### How is vector storage cost estimated?

Corpus size: `documents_count × avg_document_tokens / chunk_size` ≈ chunk count; bytes ≈ `chunks × dimensions × 4` (float32); GB × provider `costPerGBPerMonth` (managed tiers may enforce a monthly minimum).

### How is retrieval (provider ops) cost estimated?

For Pinecone serverless: `queries_per_month × costPerReadUnit`. For Vertex AI Vector Search managed: `(queries_per_month / 1e6) × queryUnitCostPer1M`. Self-hosted Qdrant default storage tier may be $0 for listed `costPerGBPerMonth` in self-hosted block.

### How is reranking cost estimated?

From `reranking.models[model_id].costPer1KQueries`: **per-query** = `costPer1KQueries / 1000` (USD). Matches hosted APIs that bill per search call.

### How is generation cost estimated?

Input tokens ≈ retrieved context (`top_k × chunk_size`) + assumed prompt tokens from `assumptions.avgInputTokensPerQuery`. Output tokens = `min(assumptions.avgOutputTokensPerQuery, generation.max_tokens)`. Prices: `inputCostPer1MTokens` and `outputCostPer1MTokens` per model in `pricing.json`.

### What HTTP status if pricing file is missing?

`503` with detail from `PricingLoadError` (same as utilities cost).

### What tests exist?

`tests/test_utils/test_cost_calculator.py`: deterministic mini `pricing` dict asserts `per_query` / `breakdown` labels; Cohere rerank monthly total. `tests/test_designer.py::test_designer_cost_endpoint` integration test on live catalog.

---

## Phase 4 · P4-4 — Export API

### What endpoint exports a pipeline to code or infra artefacts?

`POST /api/designer/export` with body `ExportRequest`: `config` (`PipelineConfigurationSchema`), `format` one of `python` | `yaml` | `terraform` | `docker-compose` | `k8s`. Response: `ExportResponse` with `code` (full file text), `filename`, `format`, `contentType` (camelCase in JSON).

### Is export stateless?

Yes. No database session or `X-User-ID`; same pattern as `POST /api/designer/cost`. Validation is entirely on `PipelineConfigurationSchema`.

### What does `ExportService` do?

`apps/api/app/services/export_service.py` dispatches by format, returns `ExportResponse` with MIME `content_type` and a slugified filename (`docker-compose.yml` for compose; `*-main.tf` for Terraform; `*-k8s-manifests.yaml` for Kubernetes).

### Where is LangChain Python code generated?

`app/services/export_generators/python_export.py` — LCEL RAG script (embeddings, LLM, vector store, retriever, optional rerank/memory), aligned with `apps/web/src/lib/generators/pythonCodeGenerator.ts`.

### Where are YAML, Terraform, Docker, and K8s generated?

- YAML: `yaml_export.py` — human-readable pipeline YAML (mirrors `yamlGenerator.ts`).
- Terraform: `terraform_export.py` — AWS/GCP/Azure scaffolds (mirrors `terraformGenerator.ts`); `multi-cloud` maps to AWS.
- Docker Compose: `docker_k8s_export.py` — API + Postgres + Redis + Qdrant with env wiring.
- Kubernetes: `docker_k8s_export.py` — Namespace, ConfigMap, Qdrant + API Deployments/Services.

### Why is there an `_compat.ev()` helper?

`RAGBaseModel` uses `use_enum_values=True`; validated models may hold plain strings instead of StrEnum instances. `ev()` normalises to string for comparisons and string templates.

### What tests cover export?

`tests/test_export.py`: parametrized check for all five formats; assertions on Python LCEL/Terraform provider/Docker services/K8s multi-doc YAML.

---

## Phase 4 · P4-5 · Templates API

### What does the Templates API provide?

Three endpoints under `/api/templates`: **list** the full catalog (`GET /api/templates`), **get one** template by stable id (`GET /api/templates/{id}`), and **apply** a template to a project (`POST /api/templates/{id}/apply`). The catalog is `data/templates.json` at the repo root (validated as `TemplatesCatalogResponse` / `PipelineTemplate` with embedded `PipelineConfigurationSchema`).

### Where is the templates file loaded from?

Resolution order matches pricing: optional `TEMPLATES_CATALOG_PATH` env (`templates_catalog_path` in `Settings`), then `apps/api/catalogs/templates.json` if present, then `../../data/templates.json` relative to `apps/api`.

### Does listing templates hit the database?

No. Only JSON validation and response serialization. Same idea as stateless export/cost for read-only catalog data.

### What is the apply request body?

`ApplyTemplateRequest`: `projectId` (required UUID), optional `name` and `description` overrides. If `name` is omitted, the template’s display name is used; description defaults from the template’s top-level description when not overridden.

### How does apply persist configuration?

`TemplateService.apply` constructs a `SaveConfigRequest` from the template’s `config` field and calls `DesignerService.save_config`. The project must belong to the user (`X-User-ID`); otherwise the service returns `None` and the route responds **404** (“Template or project not found”).

### What does the apply response include?

`ApplyTemplateResponse`: same fields as `SaveConfigResponse` (`id`, `name`, `projectId`, `description`, `config`, `createdAt`, `updatedAt`) plus **`templateId`** (the catalog id, e.g. `faq-chatbot`) for UI analytics and redirects.

### How is `metadata.source` set for applied configs?

Templates already use `source: "template"` in JSON; `DesignerService._prepare_new_config` preserves `source` when it is `"template"` (it only defaults to `"designer"` when source is missing).

### Why was `data/templates.json` updated for customer-support?

The **customer-support** template had routing `rules` with free-form `condition` strings and `model` keys. The backend schema requires `RoutingRuleSchema`: `condition` must be `keyword` | `query-length` | `semantic-complexity`, and the target LLM must be `targetModel`. The file was updated to schema-valid rules (e.g. `query-length` + `semantic-complexity` with thresholds) so the catalog loads and validates end-to-end.

### What tests cover templates?

`tests/test_templates.py`: list + get by id, unknown template 404, apply creates a row loadable via `GET /api/designer/config/{id}`, unknown template apply 404, invalid project 404.

### How does this connect to the future Template Gallery (P5-14)?

The gallery page will call `GET /api/templates` for cards and `POST /api/templates/{id}/apply` with the active project id, then navigate to Designer review with the new `config` id returned in the response.

---

*Append new `## Phase … · …` sections at the end for future tasks; keep all prior sections intact.*

---

## Phase 4.5 · P4.5-1 Guardrails Core Infrastructure

### What problem does the guardrails core solve?

It provides a **single, extensible pattern** for safety and policy checks at three RAG touchpoints (user input, retrieved context, model output) without duplicating ad hoc validation in every router or service.

### Where does the code live?

Under `apps/api/app/core/guardrails/`: `types.py` (enums and result dataclasses), `base.py` (abstract `Guardrail`), `manager.py` (`GuardrailManager`), `orchestrator.py` (`GuardrailOrchestrator`, `RetrievalGuardPayload`), `stubs.py` (reference implementations). Schemas: `apps/api/app/schemas/guardrails.py`. Tests: `apps/api/tests/test_core/test_guardrails.py`.

### What are the three pipeline stages?

`GuardrailStage.INPUT`, `RETRIEVAL`, and `OUTPUT` — matching where you typically filter or transform the user query, retrieved documents, and generated answer respectively.

### What actions can a single guardrail return?

`GuardrailAction`: `ALLOW`, `WARN`, `BLOCK`, `MODIFY`. `WARN` records a non-fatal notice; `BLOCK` stops the stage; `MODIFY` may set `payload_override` to pass a transformed value to the next guardrail and as the stage’s final payload if the run completes.

### How does `GuardrailManager.run_stage` process multiple guardrails?

It runs registered guardrails **in registration order**. On `BLOCK`, it returns immediately with `allowed=False` and `blocked_by` set to the guardrail name. On `MODIFY` with a non-null `payload_override`, it replaces the working payload before the next check. `ALLOW` and `WARN` leave the payload unchanged.

### What does `GuardrailOrchestrator` add over the manager?

Typed entry points: `check_input(text)`, `check_retrieval(RetrievalGuardPayload)`, `check_output(text)` — each delegates to `run_stage` with the correct `GuardrailStage`. Retrieval payloads carry `query` and `documents` (as an immutable tuple of LangChain `Document`).

### Is the guardrails layer async?

No — checks are synchronous for P4.5-1. Future guardrails that call remote APIs can wrap async work internally or extend the contract in a later task.

### Are there HTTP endpoints for guardrails in P4.5-1?

No. This phase is **library infrastructure only**. APIs and generation wiring are planned for P4.5-5 (RAG Pipeline Integration).

### What is logged when a guardrail runs?

The manager emits a structured log event `guardrail_check` with `guardrail`, `stage`, `action`, and optionally `request_id` when `GuardrailContext.request_id` is set.

### What is `GuardrailContext` used for?

Optional correlation and tenancy metadata: `request_id`, `user_id`, `pipeline_config_id`, `project_id`, plus an `extra` dict for arbitrary labels (e.g. experiment id).

### What Pydantic types were added and why?

`GuardrailStageSettingsSchema` (per-stage `enabled` flag) and `GuardrailsConfigSchema` (input / retrieval / output sections) so future pipeline JSON can toggle stages without ad hoc dicts.

### What are the stub guardrails for?

`AlwaysAllowGuardrail` is a no-op placeholder. `BlockIfSubstringGuardrail` demonstrates **blocking** on a forbidden substring in string payloads (demos/tests only — not a security control).

### How do you unit test a custom guardrail?

Subclass `Guardrail`, implement `name`, `stage`, and `check`, register on a `GuardrailManager`, call `run_stage` or use `GuardrailOrchestrator`. The test suite includes examples: prefix modification chain, warn-only, retrieval empty-query block, and context capture.

### How does this connect to P4.5-2 and later?

P4.5-2–4 add real detectors (PII, toxicity, hallucination heuristics, etc.) as additional `Guardrail` classes registered on the appropriate stages. P4.5-5 calls the orchestrator from the generation path and APIs.

### What should you say in a code review about ordering?

Order matters: put **cheap** checks first (e.g. length, regex) and **expensive** checks (LLM judges) later so `BLOCK` exits early; use `register(..., first=True)` when a guardrail must run before others.

### Does P4.5-1 change `PipelineConfigurationSchema`?

No — guardrails config schemas are standalone so integration can add an optional field in P4.5-5 without breaking existing saved configs.

---

## Phase 4.5 · P4.5-2 Input Guardrails

### What ships in P4.5-2?

Three concrete `Guardrail` classes for `GuardrailStage.INPUT`: `PiiRedactionGuardrail`, `PromptInjectionGuardrail`, and `ToxicityFilterGuardrail`, plus `register_default_input_guardrails()` and `clear_input_guardrails()`.

### Where is the code?

`apps/api/app/core/guardrails/input/` (`pii.py`, `injection.py`, `toxicity.py`, `__init__.py`). Re-exported from `app.core.guardrails`.

### In what order should input guardrails run?

Default registration order: **PII redaction first** (so later checks see redacted text and PANs are not mistaken for phones), then **prompt-injection block**, then **toxicity block**. PII is registered with `first=True` on the manager.

### How does PII redaction behave?

`MODIFY` with `payload_override` when something was redacted; `ALLOW` if nothing matched. Placeholders: `[REDACTED_EMAIL]`, `[REDACTED_SSN]`, `[REDACTED_PHONE]`, `[REDACTED_CARD]`. Cards require **Luhn-valid** digit runs.

### Why is credit-card redaction ordered before phone?

The phone regex can match substrings inside long digit sequences. Running **card (Luhn) before phone** reduces false phone redaction on PAN-like strings.

### How does prompt injection detection work?

Case-insensitive **regex** patterns for phrases such as “ignore previous instructions”, “developer message:”, “jailbreak”, “DAN mode”, etc. A match returns `BLOCK`. The list is not exhaustive; extend via `PromptInjectionGuardrail(patterns=...)`.

### How does toxicity filtering work without an ML model?

`ToxicityFilterGuardrail` applies **optional** `blocked_terms` (word-boundary regex per term) and **`extra_patterns`**. Defaults include only a pattern that matches `___RAG_STUDIO_TOXICITY_SELF_TEST___` so normal users are not blocked until operators add terms or patterns (production lists can be loaded in P4.5-7).

### What Pydantic changes apply to guardrails config?

`GuardrailsConfigSchema.input` is now `InputStageGuardrailsSchema`, which extends `enabled` with `pii_redaction_enabled`, `prompt_injection_block_enabled`, and `toxicity_block_enabled` (all default `True`). The retrieval stage still uses `GuardrailStageSettingsSchema`; the output stage gained per-check flags in **P4.5-3** (`OutputStageGuardrailsSchema`).

### Does `register_default_input_guardrails` read the Pydantic schema?

Not automatically — wiring `InputStageGuardrailsSchema` flags to registration is for **P4.5-5** integration. The helper takes boolean kwargs `pii`, `prompt_injection`, `toxicity` and optional toxicity term/pattern overrides.

### How do you test toxicity without embedding slurs in the repo?

Tests pass `toxicity_blocked_terms=frozenset({"badword"})` and `toxicity_extra_patterns=()` or use the self-test marker string for the default pattern.

### Are HTTP routes added in P4.5-2?

No — same as P4.5-1; API integration is P4.5-5.

### What tests exist?

`apps/api/tests/test_core/test_input_guardrails.py` covers email/SSN/card redaction, injection block, toxicity custom term, registration order, orchestration after PII, and schema defaults.

---

## Phase 4.5 · P4.5-3 Output Guardrails

### What ships in P4.5-3?

Three concrete `Guardrail` classes for `GuardrailStage.OUTPUT`: `HallucinationHeuristicGuardrail`, `FactualityCheckGuardrail`, and `CitationVerificationGuardrail`, plus `register_default_output_guardrails()` and `clear_output_guardrails()`.

### Where is the code?

`apps/api/app/core/guardrails/output/` (`hallucination.py`, `factuality.py`, `citation.py`, `context_refs.py`, `__init__.py`). Re-exported from `app.core.guardrails`.

### In what order do output guardrails run?

Default registration: **hallucination heuristic** → **factuality** → **citation verification**. Citation runs last so invalid `[n]` citations can **BLOCK** after softer **WARN** checks.

### Why does output grounding use `GuardrailContext.extra`?

`GuardrailOrchestrator.check_output` still takes a plain `text: str`; retrieval chunks are passed out-of-band as `GuardrailContext.extra["reference_texts"]` until P4.5-5 attaches them from the RAG pipeline. Optional `extra["citation_source_count"]` overrides how many numbered sources exist for citation validation.

### What does `HallucinationHeuristicGuardrail` do?

When `reference_texts` are present, extracts substantive alphanumeric tokens from the answer, filters a small stop-word set, requires at least eight tokens, and compares **word-boundary** matches against joined references. Below a configurable grounding ratio threshold it returns **WARN** with metadata (`grounding_ratio`, token counts). With no references it **ALLOW**s and records `skipped: no_reference_texts`.

### Why WARN instead of BLOCK for hallucination heuristics?

Lexical overlap is a **weak** signal; false positives would harm UX. Production systems often log WARNs, show disclaimers, or trigger human review. BLOCK is reserved for clear policy violations (here: invalid citations).

### What does `FactualityCheckGuardrail` check?

**Dates** (ISO-like and slash forms) and **integers ≥ 100** (configurable) that appear in the answer must appear as substrings in the joined reference text; otherwise **WARN** with `missing_in_references`. Small integers are ignored to reduce noise from common prose. Decimals are checked if absent from references.

### How does `CitationVerificationGuardrail` work?

Regex finds bracket citations `[1]`, `[2]`, … Valid range is `1` through `citation_source_count` if set in `extra`, else `len(reference_texts)`. Out-of-range indices → **BLOCK**. Citations present with zero allowed sources → **WARN**. No citations → **ALLOW**.

### What Pydantic changes apply to guardrails config?

`GuardrailsConfigSchema.output` is now `OutputStageGuardrailsSchema`, adding `hallucination_heuristic_enabled`, `factuality_check_enabled`, and `citation_verification_enabled` (all default `True`) in addition to `enabled`.

### Does registration read the Pydantic schema automatically?

No — same as input; P4.5-5 will map flags to `register_default_output_guardrails(...)` kwargs.

### What tests cover P4.5-3?

`apps/api/tests/test_core/test_output_guardrails.py` exercises skip paths, WARN paths, BLOCK on bad citations, registration order, orchestrator chain with `blocked_by`, and schema defaults.

### How would you improve these checks in production?

Use NLI or LLM judges for claim verification, structured citation formats (doc IDs), cross-encoder scores against chunks, and calibrated thresholds from offline eval — while keeping heuristics as fast first-line filters.

---

## Phase 4.5 · P4.5-4 Retrieval Guardrails

### What ships in P4.5-4?

Three concrete `Guardrail` classes for `GuardrailStage.RETRIEVAL`: `RetrievedContentFilterGuardrail`, `SourceProvenanceGuardrail`, and `RetrievalBiasHeuristicGuardrail`, plus `register_default_retrieval_guardrails()` and `clear_retrieval_guardrails()`.

### Where is the code?

`apps/api/app/core/guardrails/retrieval/` (`content_filter.py`, `source_validation.py`, `bias.py`, `__init__.py`). Re-exported from `app.core.guardrails`.

### What payload shape does the RETRIEVAL stage use?

`RetrievalGuardPayload` (`query: str`, `documents: tuple[Document, ...]`) — the same type `GuardrailOrchestrator.check_retrieval` already passes to `GuardrailManager.run_stage`.

### In what order do retrieval guardrails run?

Default registration: **retrieved content filter** (drop bad chunks) → **source provenance** (optional; only registered when requirements are configured) → **bias heuristic** (WARN). Content filter runs first so provenance and bias see the sanitized chunk list.

### Why is `SourceProvenanceGuardrail` sometimes not registered?

If `source_validation=True` but `source_required_keys` is empty and `source_require_https_url` is false, there is nothing to enforce, so the provenance guardrail is omitted to avoid no-op runs. Pass `source_required_keys=frozenset({"doc_id"})` (or enable HTTPS URL checks) to activate it.

### What does `RetrievedContentFilterGuardrail` do?

Same composition as INPUT toxicity: optional `blocked_terms` (word-boundary) and `extra_patterns`. Default patterns match only `___RAG_STUDIO_RETRIEVAL_FILTER_SELF_TEST___`. Matching chunks are removed; **MODIFY** with a shorter document tuple. If **all** chunks match policy, the stage **BLOCK**s.

### What does `SourceProvenanceGuardrail` validate?

Each `Document.metadata` must contain non-empty string values for every key in `required_metadata_keys`. If `require_https_source_url` is true, when `source_url` is present and non-empty it must start with `https://`. Failing chunks are dropped (**MODIFY**); if none remain, **BLOCK**.

### What does `RetrievalBiasHeuristicGuardrail` do?

Scans the query string and each chunk’s `page_content` for regex **patterns** (defaults: self-test marker only). First match returns **WARN** with `where` (`query` or `document[i]`). Does not remove chunks (policy can evolve in P4.5-5 / P4.5-7).

### What Pydantic changes apply?

`GuardrailsConfigSchema.retrieval` is now `RetrievalStageGuardrailsSchema` with `content_filter_enabled`, `source_validation_enabled`, and `bias_detection_enabled` (default `True`), plus inherited `enabled`.

### Are HTTP routes added?

No — library-only; pipeline integration is P4.5-5.

### What tests exist?

`apps/api/tests/test_core/test_retrieval_guardrails.py` covers filter MODIFY/BLOCK, provenance drop/https, bias WARN, registration order, orchestrator chain, and schema defaults.

### How does this relate to the core `app.core.retrieval` service?

Core retrieval returns ranked `Document`s; guardrails are a **policy layer** on that result set before generation. They do not replace vector search — they post-filter and flag.

---

## Phase 4.5 · P4.5-5 RAG Pipeline Integration

### What problem does P4.5-5 solve?

Guardrails were implemented per stage (P4.5-2 … P4.5-4) but not **wired** into a single request path. P4.5-5 connects INPUT → RETRIEVAL (post-retrieval payload) → `GenerationService` → OUTPUT, applies saved **per-stage toggles** from configuration, and exposes **HTTP preview** endpoints for Designer and shared utilities callers.

### Where is the integration code?

- **Policy → manager** — `apps/api/app/core/guardrails/configure_manager.py` (`build_guardrail_manager`).
- **Orchestrated run** — `apps/api/app/core/rag/guarded_runner.py` (`run_guarded_rag_query`).
- **API** — `POST /api/designer/rag-preview` and `POST /api/utilities/rag-preview` (same body), implemented via `app/services/rag_preview_service.py`.

### How does `PipelineConfigurationSchema` carry guardrails?

Optional field `guardrails: GuardrailsConfigSchema | None` (JSON camelCase `guardrails`). When omitted, **defaults** match `GuardrailsConfigSchema()` (all sub-stage flags on). Resolved in the runner as `policy = explicit_override or pipeline.guardrails`; `build_guardrail_manager(None)` still applies full defaults.

### What is the execution order?

1. **INPUT** — `GuardrailOrchestrator.check_input(query)`; may MODIFY (e.g. PII redaction) or BLOCK.
2. **RETRIEVAL** — `check_retrieval(RetrievalGuardPayload(query_after_input, documents))`; may MODIFY (drop chunks) or BLOCK.
3. **Generation** — `await GenerationService.generate(...)` with the post-guard query and documents.
4. **OUTPUT** — `check_output(answer, context)` with `GuardrailContext.extra` populated with `reference_texts` and `citation_source_count` from the final chunk list. If OUTPUT **BLOCK**s, the API **does not** return model text (unsafe answer omitted).

### Why pass client-supplied `context_documents` instead of calling `RetrievalService` in the preview?

Phase 4 Designer does not yet bind every deployment to a live vector index in-process. The preview contract matches **“here is what retrieval returned”** so the same guardrail chain applies when Autopilot or a worker later wires real retrieval. Vector lookup can be swapped in without changing guard semantics.

### How do stage `enabled: false` and per-check flags work?

`build_guardrail_manager` clears each stage and registers nothing when that stage’s `enabled` is false. When enabled, individual checks map to `register_default_*_guardrails(..., pii=..., hallucination=..., content_filter=...)` exactly as documented in P4.5-2 … P4.5-4.

### What does the preview response include?

`RagPreviewResponse`: `allowed`, optional `blocked_stage` / `blocked_by`, `query_used`, optional `answer` / `model` / `provider`, three lists of `GuardrailCheckSummary` (input / retrieval / output), and `had_warnings`.

### What tests cover P4.5-5?

`apps/api/tests/test_core/test_guarded_rag_integration.py` — factory respects disabled INPUT stage, injection **BLOCK** on `run_guarded_rag_query`, happy path with mocked LLM, utilities **rag-preview** route integration, orchestrator smoke with configured manager.

### How would you extend this for production RAG?

Add an internal path that obtains `context_documents` from `RetrievalService` + vector store using `pipeline.stages`, reuse the same `run_guarded_rag_query`, stream tokens only after OUTPUT passes (or buffer then validate), and emit metrics (P4.5-6).

---

## Phase 4.5 · P4.5-6 Monitoring & Metrics

### What does P4.5-6 deliver?

**Prometheus-style observability** for guardrails: per-check counters and latency histograms, per-stage allow/block totals, end-to-end guarded RAG outcome counts, a **`GET /metrics`** scrape endpoint (OpenMetrics text), a **`GET /monitoring/guardrails`** JSON snapshot for quick dashboards, and a **`prometheus_metrics_enabled`** setting to disable those routes when needed. Structured logs from P4.5-1 (`guardrail_check` events) remain the narrative complement to numeric metrics.

### Which metrics exist and what are their labels?

| Metric | Type | Labels | Meaning |
|--------|------|--------|---------|
| `rag_guardrail_checks_total` | Counter | `stage`, `guardrail`, `action` | One increment per `Guardrail.check()` completion (`allow`, `warn`, `block`, `modify`, …). |
| `rag_guardrail_stage_results_total` | Counter | `stage`, `outcome` | One increment per `run_stage` completion: `allowed` or `blocked`. |
| `rag_guardrail_check_duration_seconds` | Histogram | `stage`, `guardrail` | Wall time inside each `check()` call (buckets tuned for sub-second guards). |
| `rag_guardrail_rag_runs_total` | Counter | `outcome` | One increment per `run_guarded_rag_query` terminal path: `success`, `blocked_input`, `blocked_retrieval`, `blocked_output`. |

### Where is the instrumentation wired?

- **Per check + stage** — `GuardrailManager.run_stage` in `apps/api/app/core/guardrails/manager.py` records check metrics after each result and stage outcome on exit (blocked or full pass).
- **Pipeline** — `run_guarded_rag_query` in `apps/api/app/core/rag/guarded_runner.py` records the RAG-level outcome on every return path.
- **Definitions** — `apps/api/app/core/guardrails/metrics.py`.

### Why both stage-level and RAG-level counters?

Stage metrics show **which layer** fires (input vs retrieval vs output) and **which guard** dominated (`action`, `guardrail` label). RAG-level outcomes answer product questions like “what fraction of preview requests were blocked before generation?” without joining every per-check series.

### How do you scrape Prometheus in Kubernetes?

Add a `ServiceMonitor` (Prometheus Operator) or annotate the API service for scraping, target **`/metrics`** on the API port, and set `PROMETHEUS_METRICS_ENABLED=true` (default). Restrict network access to `/metrics` if the endpoint should not be public.

### When would you set `prometheus_metrics_enabled` to false?

Edge deployments where the scrape surface must be absent (404 on `/metrics` and `/monitoring/guardrails`), or when a sidecar exports metrics instead. Tests can override via `PROMETHEUS_METRICS_ENABLED=false` and `get_settings.cache_clear()`.

### What does `GET /monitoring/guardrails` return?

A JSON object `{ "metrics": [ { "name", "labels", "value" }, ... ] }` listing samples whose names start with `rag_guardrail`, derived from the process registry — useful for a minimal admin UI before Grafana is available.

### How are label values sanitized?

Guardrail names and free-text labels are passed through `_label()` in `metrics.py` (non-alphanumeric → `_`, max length) to avoid runaway cardinality from user content while keeping stable series for built-in guard names.

### Does this replace Phase 11 Prometheus work?

No — P4.5-6 is **guardrail-focused** metrics and routes. Phase **P11-2** can add service-wide HTTP metrics, business KPIs, and Grafana dashboards that **include** these series alongside request latency and DB health.

### What tests cover P4.5-6?

`apps/api/tests/test_guardrail_metrics.py` — delta assertions on registry samples, `/metrics` body contains `rag_guardrail_*`, JSON snapshot route, disabled-metrics 404.

### How would you alert on this in production?

Example PromQL: high rate of `rag_guardrail_rag_runs_total{outcome="blocked_output"}` vs `success`, or spikes in `rag_guardrail_checks_total{action="block"}`. Pair with logs via `request_id` in `GuardrailContext` for drill-down.

---

## Phase 4.5 · P4.5-7 Configuration & Testing

### What does P4.5-7 add on top of earlier guardrail phases?

**File-based operator policies** for three areas: **INPUT toxicity** (blocked terms + regex), **RETRIEVAL content filter** (same shape), and **RETRIEVAL bias heuristic** (regex only). Paths are optional **environment variables** mapped through `Settings`; empty paths mean the same **code defaults** as before (self-test markers so CI and idle traffic behave predictably). **Tests** cover parsing, invalid regex failure, missing-file warning, and end-to-end **blocked INPUT** / **WARN retrieval** when policies are set.

### Why separate JSON files from the saved pipeline `guardrails` object?

Pipeline JSON encodes **which stages and checks are on** for a design. Operator word lists and regex policies change on **compliance** or **locale** cadence without editing every saved config. Mounting JSON from ConfigMaps, secrets, or `config/guardrails/local/` keeps **policy data** out of application code while **semantics** stay in the guardrail classes.

### What is the JSON schema for toxicity and content-filter files?

Each file is a JSON object with **`blocked_terms`**: array of strings (word-boundary matching after trim), and **`regex_patterns`**: array of **Python `re` patterns**. Loaded regexes are compiled in order; a bad pattern raises **`ValueError`** at load time. After load, patterns are **appended** to the built-in default extra patterns (self-test markers) so tests and safe defaults remain unless you replace behaviour in code.

### What is the JSON schema for bias patterns?

A single array field **`regex_patterns`**. Patterns are compiled and **merged** with the default bias self-test pattern tuple. Matches still produce **WARN**, not **BLOCK**, consistent with P4.5-4.

### Which environment variables map to `Settings`?

`GUARDRAILS_TOXICITY_POLICY_PATH`, `GUARDRAILS_CONTENT_FILTER_POLICY_PATH`, and `GUARDRAILS_BIAS_PATTERNS_POLICY_PATH` correspond to `guardrails_toxicity_policy_path`, `guardrails_content_filter_policy_path`, and `guardrails_bias_patterns_policy_path` in `app/config.py` (Pydantic settings, typical env naming).

### What happens if a path is set but the file is missing?

`policy_loader` logs a **warning** and returns **`None`** for that policy; `build_guardrail_manager` then behaves as if no file were configured for that dimension (no crash on startup).

### Where should operators put secrets or sensitive blocklists?

Prefer **mounted files** or a gitignored directory such as `apps/api/config/guardrails/local/` (listed in `.gitignore`) so terms never land in the repo. The committed **`examples/`** folder carries only empty or illustrative placeholders.

### How does this interact with `build_guardrail_manager` and saved pipeline guardrails?

`GuardrailsConfigSchema` still controls **stage on/off** and per-check toggles. File policies only **parameterize** the registered `ToxicityFilterGuardrail`, `RetrievedContentFilterGuardrail`, and `RetrievalBiasHeuristicGuardrail` instances when paths resolve—orthogonal to disabling a whole stage via `enabled: false`.

### What tests cover P4.5-7?

`apps/api/tests/test_core/test_guardrails_policy_loader.py` — loader merge and invalid regex, missing file, env-driven manager blocking on a blocked term, `run_guarded_rag_query` INPUT block, and retrieval **WARN** when a bias regex from file matches the query.

### How would you validate policies before rollout?

Unit-test the JSON in CI, run **`pytest`** with fixture files, dry-run in staging with `LOG_LEVEL=DEBUG` and guardrail metrics from P4.5-6, and canary a small percentage of traffic while watching `rag_guardrail_*` series.

---

## Phase 5 · P5-2 Cloud Provider Selector

### What does P5-2 deliver?

A **first-stage configuration UI** in Designer mode that lets users choose **AWS**, **GCP**, **Azure**, or **Multi-Cloud** using metadata from the committed **`data/cloud-providers.json`** catalog (not hard-coded strings in the component). The selection updates **`draft.cloudProvider`** in **`useDesignerStore`** via **`patchDraft`**, so it persists with the rest of the pipeline draft (Zustand **`persist`** → localStorage).

### Where is the catalog loaded and why import JSON instead of fetching?

**`apps/web/src/lib/cloud-providers-catalog.ts`** imports the JSON **at build time**. That keeps the catalog **versioned with the repo**, avoids an extra runtime fetch, and guarantees types align with **`CloudProviderMeta`** in **`apps/web/src/types/models.ts`**. Vitest tests assert provider **`id`** values match the **`CloudProvider`** union in **`pipeline.ts`**.

### How does the UI surface catalog fields?

**`CloudProviderSelector`** renders a **radiogroup** of cards (logo, short name, pricing tier badge, description excerpt). The selected provider shows a **detail panel**: **bestFor**, **strengths**, grouped **nativeServices**, **ragStudioDefaults** (hints for later stages), and **compliance** chips — all straight from the JSON.

### Why are there files under `apps/web/public/logos/`?

The catalog references **`/logos/aws.svg`** (and similar). Those paths resolve from **`public/`** in Next.js. **P5-2** adds lightweight SVG marks so **`next/image`** does not 404; **`onError`** falls back to **shortName** initials if an asset is missing.

### How does this connect to the backend?

**`PipelineConfiguration.cloudProvider`** on the frontend mirrors **`CloudProvider`** in **`apps/api/app/schemas/pipeline.py`**. Saving or exporting the designer draft sends the same string the API expects (`aws`, `gcp`, `azure`, `multi-cloud`). Cost and export code already branch on **`cloud_provider`**.

### Should selecting a cloud auto-change vector store defaults?

**P5-2** intentionally **does not** overwrite **`stages.vectorStore`** automatically — users may have already tuned later stages; upcoming selectors (e.g. P5-6) can apply **`ragStudioDefaults`** explicitly. The UI **surfaces** defaults as **hints** only.

### What accessibility choices were made?

Cards use **`role="radiogroup"`** / **`role="radio"`** with **`aria-checked`**. Keyboard focus uses **`focus-visible`** ring styles on each card button.

### What tests cover P5-2?

**`apps/web/src/lib/__tests__/cloud-providers-catalog.test.ts`** — catalog shape, four providers, id narrowing, **`ragStudioDefaults`** presence.

### How would you add a new cloud (e.g. Oracle) end-to-end?

1. Extend **`CloudProvider`** in TS and Python enums if the product adds a fifth provider.  
2. Append an entry to **`data/cloud-providers.json`** with the same **`id`**.  
3. Add a logo under **`public/logos/`** if using **`logo`** paths.  
4. Run **`npm run test`** and regression-test designer save/export.

### What is the main limitation of P5-2 alone?

Only the **cloud** stage is fully interactive; other stages still use **`DesignerStagePlaceholder`** until their P5 tasks ship. The **navigator** copy reflects that rollout.

---

## Phase 5 · P5-3 Data Ingestion Configuration

### What does P5-3 deliver?

An interactive **Data Ingestion** stage at **`/designer/ingestion`** that edits **`draft.stages.dataIngestion`** (`DataIngestionConfig`): **source type** (file upload, S3, GCS, Azure Blob, URL, database, API), **allowed file extensions**, **preprocessing** toggles (strip HTML, normalize whitespace, extract metadata) plus optional **custom rules** (one per line), **metadata** toggles (include source path, include page numbers) with optional **custom key/value metadata**, and **connection hints** stored in **`connectionConfig`** (non-secret placeholders only). Changes persist via **`useDesignerStore`** → **`updateStages`** with the same **Zustand `persist`** behavior as other Designer stages.

### Where is the UI implemented?

**`apps/web/src/components/designer/data-ingestion-configurator.tsx`** — exported as **`DataIngestionConfigurator`**. **`DesignerStagePlaceholder`** renders it when **`stageId === 'ingestion'`**, parallel to **`CloudProviderSelector`** on the cloud stage.

### How does state merge without wiping nested fields?

The component uses a **`mergeIngestion`** helper that starts from **`createDefaultPipelineConfiguration().stages.dataIngestion`** when missing and deep-merges **`preprocessing`** and **`metadata`**. **`patchIngestion`** composes partial updates so toggling one switch does not drop sibling keys.

### Which schema validates the shape?

Frontend **`DataIngestionConfigSchema`** in **`apps/web/src/lib/validators.ts`** (Zod) mirrors **`DataIngestionConfig`** in **`apps/web/src/types/pipeline.ts`** and aligns with **`DataIngestionConfigSchema`** in **`apps/api/app/schemas/pipeline.py`**. The UI runs **`safeParse`** and shows a compact issue list if the draft becomes invalid (for example after manual localStorage edits).

### Why separate “connection hints” from secrets?

**`connectionConfig`** is typed as **`Record<string, unknown>`** for flexibility; the UI labels it **reference only** and collects bucket names, regions, prefixes, hosts, etc. **Credentials belong in a secret manager at deploy time**, not in the committed pipeline JSON—consistent with export and Terraform generators treating config as **non-sensitive parameters**.

### What happens when the user changes source type?

**`connectionConfig`** is **reset to `{}`** so incompatible keys from the previous source do not leak into export payloads. The user re-enters hints for the new source.

### How are custom metadata rows edited?

Rows render from **`metadata.customMetadata`** (`Record<string, string>`). Users add/remove rows; empty keys are stripped on save so optional **`customMetadata`** can be omitted entirely.

### Does P5-3 upload files or connect to live buckets?

**No.** This task is **configuration only**. Actual ingestion execution remains in **Phase 2 `IngestionService`** and future worker/API flows; the Designer captures **intent** for YAML/Python/export.

### What appears in the stage navigator for ingestion?

**`StageNavigator`** shows a short **source hint** under **Data Ingestion** (for example **S3**, **GCS**, **Upload**) derived from **`draft.stages.dataIngestion.sourceType`**, similar to the cloud abbreviation for the first stage.

### What accessibility patterns are used?

Source selection uses **`role="radiogroup"`** / **`role="radio"`** with **`aria-checked`**. Switches use **`role="switch"`** / **`aria-checked`**. Sections have **`aria-labelledby`** where headings anchor the region.

### What tests should be added or extended for P5-3?

There is **no new Vitest file** in this change set; regression relies on **`tsc --noEmit`** and existing **`validators`** tests. A follow-up can add component tests for **`mergeIngestion`** or smoke-render **`DataIngestionConfigurator`**.

### What remains placeholder after P5-3?

Stages **chunking** through **review** still use the dashed placeholder until **P5-4+**. Only **Cloud Provider** and **Data Ingestion** are fully interactive.

### How does ingestion config flow to exports?

 **`draft.stages.dataIngestion`** is part of **`PipelineConfiguration`** consumed by **`yamlGenerator`**, **`pythonCodeGenerator`**, **`mermaidGenerator`**, and backend export services—same field names as today (**`dataIngestion`** in TS JSON).

---

## Phase 5 · P5-4 · Chunking Configuration

### What problem does the Chunking stage solve in Designer mode?

Users must choose **how documents are split** before embedding: strategy (fixed, recursive, semantic, etc.), **token budget** per chunk, **overlap** for boundary context, and optionally **recursive separators** and **chunk metadata**. **P5-4** surfaces those controls with catalog-backed guidance so the draft matches **`ChunkingConfig`** in **`pipeline.ts`** and validates with **`ChunkingConfigSchema`** (Zod).

### Where does strategy metadata come from?

**`data/chunking-strategies.json`** is imported at build time through **`apps/web/src/lib/chunking-strategies-catalog.ts`** (same pattern as **`cloud-providers-catalog.ts`**). The UI lists **name**, **description**, **complexity** badge, and **pros / cons / best for** from the catalog.

### How does changing strategy update numbers?

**`chunkingDefaultsFromCatalog(strategy)`** reads **`defaultConfig`** from the catalog entry, **clamps** **`chunkSize`** to **128–4096** and **`chunkOverlap`** to **0–1024** with **`overlap < chunkSize`**. Catalog entries with symbolic sizes (e.g. sentence counts) are lifted to at least **128** tokens so **`ChunkingConfigSchema`** stays valid. **Recursive character** also copies **`separators`** when present.

### Which fields are persisted on the draft?

**`updateStages({ chunking })`** merges into **`draft.stages.chunking`**: **`strategy`**, **`chunkSize`**, **`chunkOverlap`**, optional **`separators`**, optional **`metadata`** (**`includeSource`**, **`includePageNumber`**, **`customMetadata`**). Zustand **`persist`** saves to **localStorage** with the rest of the pipeline.

### How are separators edited for recursive-character?

A **textarea** shows **one separator string per line**. Inline control characters use escapes: **`\n`**, **`\t`**, **`\r`**, and **`\\`** for a literal backslash. This mirrors LangChain’s ordered separator list (large structures first).

### Does the UI run chunking on documents?

**No.** Configuration only. Actual splitting runs in backend **`ChunkingService`** (**P2-2**) when a build or export executes.

### How does the sidebar summarize chunking?

**`StageNavigator`** prints **`chunkingHint`**: short strategy label plus **`chunkSize/chunkOverlap`** (e.g. **Recursive · 512/50**).

### How does this align with the API schema?

Backend **`ChunkingConfigSchema`** in **`apps/api/app/schemas/pipeline.py`** includes **`strategy`**, **`chunk_size`**, **`chunk_overlap`**, **`separators`**. Frontend **`metadata`** on chunking is **designer/export enrichment**; confirm serializer paths when posting full pipeline JSON to the API.

### What validation rules catch bad drafts?

**`chunkOverlap < chunkSize`**, **chunk size** in **[128, 4096]**, **overlap** in **[0, 1024]**. The component surfaces **Zod** issues in an alert region.

### What accessibility patterns are used?

Strategy grid uses **`role="radiogroup"`** / **`role="radio"`** / **`aria-checked`**. Metadata toggles use **`role="switch"`**. Section headings use **`aria-labelledby`**.

### What tests cover this task?

**`apps/web/src/lib/__tests__/chunking-strategies-catalog.test.ts`** asserts catalog size, meta lookup, **clamping** behavior, and **recursive** separators. **`ChunkingConfigurator`** relies on existing **`ChunkingConfigSchema`** tests in **`validators`** tests.

### What remains placeholder after P5-4?

Stages **embedding** through **review** still ship as dashed placeholders until **P5-5+**. **Cloud**, **ingestion**, and **chunking** are interactive.

---

## Phase 5 · P5-5 · Embedding Model Selector

### What problem does the Embedding stage solve in Designer mode?

Users must pick **which embedding model** drives vectorization: **provider**, **output dimensions**, **context length**, and cost/quality tradeoffs. **P5-5** surfaces **`data/models/embeddings.json`** with **search** and **filters** so the draft’s **`EmbeddingConfig`** matches **`pipeline.ts`** and **`EmbeddingConfigSchema`** (Zod).

### Where does embedding catalog metadata come from?

**`data/models/embeddings.json`** is imported at build time through **`apps/web/src/lib/embeddings-catalog.ts`** (same pattern as **`chunking-strategies-catalog.ts`**). The UI lists **name**, **description**, **tier**, **quality**, **dimensions**, **speed**, and flags like **deprecated** and **open source**.

### How does selecting a model update the draft?

**`embeddingConfigFromCatalogEntry(modelId, { batchSize })`** returns **`model`**, **`provider`**, **`dimensions`**, **`maxTokens`**, and optional **`batchSize`**. **`EmbeddingConfigurator`** calls **`updateStages({ embedding: merge(...) })`** so nested **`EmbeddingConfig`** replaces fields consistently with the catalog row.

### Which fields can users tune in the UI?

**Model choice** (from catalog cards), **`batchSize`** (slider **1–2048**). **Dimensions**, **provider**, and **max tokens** follow the catalog entry for the selected **`model` id — they are shown read-only in the summary panel to avoid schema drift.

### How does filtering work?

Client-side **`useMemo`** over **`listEmbeddingModels()``**: optional **text search** (id, name, description, **bestFor**, languages), dropdowns for **provider**, **tier**, **quality**, **speed**, **open source** (all / yes / no), and a **hide deprecated** toggle (default on).

### Does the UI call the embedding API?

**No.** Configuration only. Actual embedding runs in backend **`EmbeddingService`** (**P2-3**) during builds or workers.

### How does the sidebar summarize embedding?

**`StageNavigator`** prints **`embeddingHint`**: short display **name** (from catalog) plus **`dimensions`** (e.g. **text-embedding-3-small · 1536d**).

### Why extend **`ModelSpeed`** with **very-fast**?

The catalog includes **`all-MiniLM-L6-v2`** with **`speed: "very-fast"`**. **`EmbeddingModel`** in **`types/models.ts`** aligns with catalog values so the filter and badges stay type-safe.

### What validation rules apply?

**`EmbeddingConfigSchema`**: **model** non-empty string, **provider** enum, **dimensions** **1–8192**, **batchSize** optional **1–2048**, **maxTokens** optional **≥ 1**. Invalid drafts show the same alert pattern as **Chunking**.

### What accessibility patterns are used?

Model grid uses **`role="radiogroup"`** / **`role="radio"`** / **`aria-checked`**. Search has a visible label via **`sr-only`**. Filter sections use **`aria-labelledby`**.

### What tests cover this task?

**`apps/web/src/lib/__tests__/embeddings-catalog.test.ts`** asserts model **count**, **lookup**, **`isEmbeddingProvider`**, **`embeddingConfigFromCatalogEntry`** mapping, and unknown ids. **`EmbeddingConfigurator`** relies on existing **`EmbeddingConfigSchema`** tests in **`validators`** tests.

### What remains placeholder after P5-5?

Stages **vector store** through **review** still use dashed placeholders until **P5-6+**. **Cloud**, **ingestion**, **chunking**, and **embedding** are interactive.

### What if filters hide the model that is already selected?

**`EmbeddingConfigurator`** computes **`displayModels`**: if **`getEmbeddingModelMeta(cfg.model)`** is missing from **`filteredModels`**, that catalog row is **prepended** so the radiogroup always lists the active choice. **`pinnedSelectionId`** drives a **“Current · outside filters”** badge and helper copy under the filter summary so users know why an extra card appears and how it relates to **`draft.stages.embedding`**.

### Why not allow arbitrary model strings in the UI?

Free-text **`model`** ids would drift from **`embeddings.json`** and break **`embeddingConfigFromCatalogEntry`** mapping (**provider** / **dimensions** / **maxTokens**). Catalog-backed cards keep the draft aligned with **Phase 2 `EmbeddingService`** expectations and with **vector index dimensionality** (provision indexes with **`cfg.dimensions`**).

### How does the Embedding stage relate to the Python export generator?

**`pythonCodeGenerator`** picks **`langchain_*`** imports from **`draft.stages.embedding.provider`** (see **`EMBEDDING_IMPORTS`** / **`EMBEDDING_CLASS`** in **`apps/web/src/lib/generators/pythonCodeGenerator.ts`**). Selecting a catalog model therefore shapes generated code; **`batchSize`** is the main extra tunable in the UI beyond the catalog row.

### How do you add a new embedding model to the product?

1. Add a row to **`data/models/embeddings.json`** with **id**, **provider** (must be a known **`EmbeddingProvider`**), **dimensions**, **maxTokens**, and metadata used by filters. 2. Rebuild the web app so the JSON import is bundled. 3. Optional: extend **`apps/web/src/lib/__tests__/embeddings-catalog.test.ts`** expected model count. Backend **P2-3** must support the **provider** if you expect live embedding.

---

## Phase 5 · P5-6 · Vector Store Selector

### What problem does the Vector Store stage solve in Designer mode?

Users must pick **where vectors are stored and queried**: **provider** (Qdrant, Pinecone, Weaviate, etc.), an **index/collection name**, **similarity metric**, scaling hints (**replicas**, **shards**), optional **namespace**, and optional **non-secret cloud placement**. **P5-6** surfaces **`data/vector-stores.json`** with search and filters so **`draft.stages.vectorStore`** validates against **`VectorStoreConfigSchema`** (Zod) and matches **`VectorStoreConfig`** in **`pipeline.ts`**.

### Where does vector store catalog metadata come from?

**`data/vector-stores.json`** is imported at build time via **`apps/web/src/lib/vector-stores-catalog.ts`**. Entries describe **type** (managed / self-hosted / embedded), **cloudNative** flags (AWS, GCP, Azure, self-managed), **features** (e.g. **hybridSearch**), **bestFor**, **pros/cons**, and **supportedMetrics** strings.

### How does selecting a store update the draft?

**`vectorStorePatchFromCatalog(storeId, current)`** sets **`provider`** and aligns **`configuration.metric`** with metrics that are valid for **both** the catalog row **and** **`SimilarityMetric`** (**cosine**, **euclidean**, **dot**). Other **`configuration`** fields (**indexName**, **replicas**, **shards**, **namespace**, **cloud**) are merged by **`mergeVectorStore`** so users do not lose edits when switching providers.

### Why map catalog metrics (`l2`, `ip`, …) to pipeline metrics?

**`VectorStoreConfigSchema`** only allows **cosine**, **euclidean**, **dot** (see **`SimilarityMetricSchema`**). Catalog rows use vendor names (**l2** ≈ **euclidean**, **ip** ≈ **dot**). **`catalogMetricToSchemaMetric`** and **`schemaMetricsForStore`** restrict the metric dropdown to **export-safe** values; metrics with no mapping (e.g. **hamming**) are omitted from the selector.

### Which fields are editable in **`VectorStoreConfigurator`**?

**Provider** (cards), **indexName** (validated pattern), **metric** (intersection of catalog + schema), **replicas**, **shards**, optional **namespace**, optional **cloud.region** and **cloud.instanceType** (hints only — no secrets). Clearing both region and instance **removes** **`configuration.cloud`** from state.

### How does filtering work?

Client-side filters: **text search** (id, name, description, bestFor), **deployment type**, **cloud native** (AWS/GCP/Azure must be true on **`cloudNative`**), and **hybrid search capable** (**features.hybridSearch**). If filters exclude the **currently selected** provider, that row is **pinned** at the top (same pattern as **Embedding**).

### Does the UI create or connect to a live index?

**No.** Configuration only. Provisioning and queries run through backend **`VectorStoreService` (P2-4)** and runtime credentials — not in the Designer draft.

### How does **`StageNavigator`** summarize vector store?

**`vectorStoreHint`** shows catalog **display name** · **indexName** (truncated), using **`getVectorStoreMeta`**.

### How does this align with code export?

**`pythonCodeGenerator`** switches on **`stages.vectorStore.provider`** for **`QdrantVectorStore`**, **`PineconeVectorStore`**, etc. (**`VECTORSTORE_IMPORTS`** / **`buildVectorStore`**). Terraform and YAML generators also read **`vectorStore`**.

### What validation rules apply?

**`VectorStoreConfigSchema`**: **provider** enum; **indexName** non-empty, **≤ 100** chars, **`/^[a-z0-9-]+$/`**; **configuration** with optional **metric**, **replicas** ≥ 1, **shards** ≥ 1, optional **namespace**, optional **cloud** with **region** required when the object is present (Zod shape).

### What accessibility patterns are used?

Provider grid uses **`role="radiogroup"`** / **`role="radio"`** / **`aria-checked`**. Filter controls use **`aria-labelledby`** / **`aria-live`** where appropriate.

### What tests cover this task?

**`apps/web/src/lib/__tests__/vector-stores-catalog.test.ts`** covers store **count**, **lookup**, **`isVectorStoreProvider`**, metric **mapping**, **`schemaMetricsForStore`**, and **`vectorStorePatchFromCatalog`**. **`VectorStoreConfigurator`** relies on existing **`VectorStoreConfigSchema`** tests in **`validators.test.ts`**.

### What remains placeholder after P5-6?

Stages **retrieval** through **review** still use dashed placeholders until **P5-7+**. **Cloud** through **vector store** are interactive.

### How do you add a new vector store to the catalog?

1. Add an entry to **`data/vector-stores.json`** with **`id`** equal to a **`VectorStoreProvider`** in **`validators.ts`**. 2. Ensure **`supportedMetrics`** map to pipeline metrics where possible. 3. Extend **`pythonCodeGenerator`** **`VECTORSTORE_IMPORTS`** / **`buildVectorStore`** if codegen should support the provider. 4. Bump **`vector-stores-catalog.test.ts`** expected count.

---

## Phase 5 · P5-7 · Retrieval Configuration

### What problem does the Retrieval stage solve in Designer mode?

Users must define **how chunks are retrieved** from the vector index: **strategy** (similarity, MMR, hybrid, parent–child, multi-query, ensemble), **top-k**, optional **score threshold**, optional **metadata filters**, and optional **reranking** (provider, model, top-n). **P5-7** surfaces **`data/retrieval-strategies.json`** and **`data/models/rerankers.json`**, writing **`updateStages({ retrieval, reranking })`** so the draft matches **`RetrievalConfig`** / **`RerankingConfig`** and **`RetrievalConfigSchema`** / **`RerankingConfigSchema`**.

### Where do retrieval strategy descriptions come from?

**`data/retrieval-strategies.json`** is bundled via **`apps/web/src/lib/retrieval-strategies-catalog.ts`**. Each row has **bestFor**, **pros/cons**, **implementationComplexity**, and **defaultConfig** used to seed **`retrievalDefaultsFromCatalog(strategy)`**.

### How does choosing a strategy update **`draft.stages.retrieval`**?

**`applyStrategy(id)`** merges **`retrievalDefaultsFromCatalog`** (including clearing **hybridSearch** / **parentChildConfig** / **multiQueryConfig** when switching away) while preserving **filters** unless the user edits them. **Hybrid** requires **`hybridSearch.alpha`** (slider). **Parent–child** sets **`parentChildConfig`**. **Multi-query** sets **`multiQueryConfig.numVariants`** and **`llmModel`** (defaults from **`draft.stages.generation.model`** when available).

### Why are MMR / ensemble extra knobs only described in copy, not extra fields?

**`RetrievalConfig`** in **`pipeline.ts`** only persists fields that **exports and validators** understand (top-k, threshold, hybrid blend, parent–child sizes, multi-query variants). Catalog text explains **fetch-k**, **λ**, **RRF** for interviews and docs; **Python/YAML generators** already branch on **strategy** with sensible literals.

### How do metadata filters work?

**`filters`** is **`MetadataFilter[]`**: **key**, **operator** (**eq**, **ne**, **gt**, …, **in**, **nin**, **contains**), **value** (string / number / boolean / string[]). **in** / **nin** values are edited as comma-separated lists and parsed in the client.

### How does reranking integrate?

**`RerankingConfig`**: **enabled**, optional **provider** (**cohere** | **huggingface** | **custom**), **model** (catalog id or free text for custom), **topN**. Catalog **`rerankers.json`** is loaded through **`rerankers-catalog.ts`**. **`pythonCodeGenerator`** emits **Cohere** or **CrossEncoder** rerank when **enabled**.

### What is the **`rerank-focus`** variant?

The **`Reranking`** route (**`/designer/reranking`**) renders **`RetrievalConfigurator variant="rerank-focus"`**: compact **current retrieval** summary + **full reranking panel** + link back to **`/designer/retrieval`** for strategy/filters. **`/designer/retrieval`** uses **`variant="full"`** (strategies, parameters, filters, reranking).

### How does this relate to **P2-5 Retrieval Service**?

The Designer stores **configuration only**. Runtime retrieval, hybrid BM25 wiring, and rerank execution live in backend **`RetrievalService`** and workers — the draft is the contract for codegen and APIs.

### What validation applies?

**`RetrievalConfigSchema`**: **topK** 1–100; **hybrid** requires **hybridSearch**; **multiQueryConfig** variants 2–10 and non-empty **llmModel**. **Filters** must satisfy **`MetadataFilterSchema`**. **Reranking** validated by **`RerankingConfigSchema`**.

### What tests cover this task?

**`apps/web/src/lib/__tests__/retrieval-strategies-catalog.test.ts`** exercises strategy **count**, **lookup**, **`isRetrievalStrategyId`**, and **`retrievalDefaultsFromCatalog`**. **`validators.test.ts`** already covers retrieval/reranking shapes.

### What remains placeholder after P5-7?

Stages **generation** through **review** stay dashed until **P5-8+**. **Cloud** through **retrieval/reranking** (configured on Retrieval + Reranking routes) are interactive.

### How do you add a new retrieval strategy to the product?

1. Extend **`RetrievalStrategy`** and **`RetrievalStrategySchema`** if the id is new. 2. Add a strategy block to **`data/retrieval-strategies.json`**. 3. Teach **`retrievalDefaultsFromCatalog`** / UI sections if new persisted fields are required. 4. Update **`pythonCodeGenerator`** / **`yamlGenerator`** / **`mermaidGenerator`** if export behavior should change.

---

## Phase 5 · P5-8 · Generation Model Selector

### What problem does the Generation stage solve in Designer mode?

Users must pick the **LLM** that will **answer** from retrieved context: **model id**, **provider**, and **inference** settings (**temperature**, **max output tokens**, optional **top-p**, optional **system prompt**, optional **output format**). **P5-8** surfaces **`data/models/generation.json`** through **`generation-catalog.ts`**, writing **`updateStages({ generation })`** so the draft matches **`GenerationConfig`** and **`GenerationConfigSchema`**.

### Where do generation model metadata and pricing come from?

**`data/models/generation.json`** is bundled like embeddings: **`apps/web/src/lib/generation-catalog.ts`** imports the JSON, exposes **`listGenerationModels`**, **`getGenerationModelMeta`**, and **`isGenerationProvider`**. Each entry includes **context window**, **max output tokens**, **cost per 1M in/out**, **tier**, **latency**, and capability flags (**supportsStreaming**, **supportsToolUse**, **supportsJsonMode**).

### How does selecting a model update **`draft.stages.generation`**?

**`generationConfigFromCatalogEntry(modelId, current)`** returns **`model`**, **`provider`**, and a **max token** value: **preserves** the user’s current **`maxTokens`** when it still fits the new model; otherwise **caps** to **`maxOutputTokens`** from the catalog (and floors at **64** to satisfy Zod). **Temperature**, **topP**, **systemPrompt**, and **outputFormat** are **not** cleared on model change — they are merged in the component with **`mergeGeneration`**.

### Why is the current model “pinned” when filters hide it?

Same pattern as **P5-5**: if **search/filters** exclude the **already selected** model, the card grid **prepends** that row and labels it **“Current · outside filters”** so the selection never disappears.

### Which inference controls does the UI expose?

- **Temperature**: slider **0–2** (step **0.05**).  
- **Max output tokens**: range **64** to **`min(32768, catalog maxOutputTokens)`** for the active model.  
- **Top-p**: optional; checkbox **“Use nucleus sampling”** toggles **`topP`** between **undefined** (omit from payload / provider default) and a **0–1** slider (default **0.95** when enabled).  
- **Output format**: **text** | **json** | **markdown** (**`OutputFormat`**).  
- **System prompt**: optional textarea; empty string clears **`systemPrompt`** (**`undefined`**).

### How does **`StageNavigator`** summarize the Generation stage?

**`generationHint`** shows **catalog display name** (or id), **temperature** (**`T0.1`**), and **max tokens** (**`1024 tok`**), using **`getGenerationModelMeta`**.

### How does this relate to **P2-6 Generation Service**?

The Designer stores **configuration only**. Actual provider calls, streaming, and tool execution live in backend **`GenerationService`** and related workers; this stage is the **contract** for codegen, cost APIs, and exports.

### What validation applies?

**`GenerationConfigSchema`**: **model** non-empty; **provider** enum; **temperature** **0–2**; **maxTokens** **64–32768**; **topP** **0–1** optional; **systemPrompt** max **10,000** chars; **outputFormat** optional enum.

### What tests cover this task?

**`apps/web/src/lib/__tests__/generation-catalog.test.ts`** covers model **count**, **lookup**, **`isGenerationProvider`**, and **`generationConfigFromCatalogEntry`** (including **capping** max tokens to a model’s **maxOutputTokens**).

### What remains placeholder after P5-8?

Stages **routing** through **review** stay dashed until **P5-9+**. **Cloud** through **generation** are interactive in the Designer.

### How do you add a new generation model to the product?

1. Add a row to **`data/models/generation.json`** with **`provider`** in **`GenerationProviderSchema`**. 2. If the **id** is new, ensure **exports** and **codegen** (e.g. **`pythonCodeGenerator`**) can map it or treat it as a string model id. 3. Bump **`generation-catalog.test.ts`** expected **count** if the test asserts it.

---

## Phase 5 · P5-9 · Routing, Memory & Evaluation Config

### What problem do the Routing, Memory, and Evaluation stages solve in Designer mode?

**Routing** lets operators **send different user queries to different LLMs** (e.g. cheap model for simple FAQs, larger model for complex reasoning) using **ordered rules** and a **fallback** model. **Memory** configures **multi-turn context** (none vs buffer vs summary vs vector-backed memory) for conversational RAG. **Evaluation** defines **quality metrics**, **test set size**, and **schedule** intent for the **Evaluation Engine (P2-7)** and CI. **P5-9** surfaces all three on separate routes and persists them via **`updateStages({ routing | memory | evaluation })`**.

### How does **`RoutingConfigurator`** align with the shared types?

It edits **`RoutingConfig`**: **`enabled`**, optional **`defaultModel`**, optional **`rules`** (**`RoutingRule`**: **`condition`** — keyword | query-length | semantic-complexity; optional **`keywords`**, **`threshold`**; **`targetModel`**). **`RoutingConfigSchema`** in **`validators.ts`** validates the shape. **Target** and **fallback** models are chosen from **`listGenerationModels()`** so ids stay consistent with **`Generation`** and **`data/models/generation.json`**.

### Why reuse the generation catalog for routing models?

Routing **`targetModel`** / **`defaultModel`** must be **provider-executable model ids** already used elsewhere in the draft. Reusing **`generation-catalog.ts`** avoids free-text drift and keeps exports (**`pythonCodeGenerator`**) aligned with **LangChain** / provider constructors.

### How are routing rules ordered?

The UI displays rules **top to bottom**; the narrative matches typical engines (**first match wins**). The backend orchestrator interprets the saved JSON; the Designer captures **declarative** intent for YAML/Python export.

### What does each **memory type** mean?

- **`none`**: Stateless RAG (no chat memory).  
- **`conversation-buffer`**: Sliding window of recent messages (**`windowSize`**).  
- **`summary-buffer`**: Summarize history; **`windowSize`** and optional **`maxTokens`** cap cost.  
- **`vector-memory`**: Embed/store past turns for retrieval-style memory; optional **`maxTokens`**.

**`MemoryConfigSchema`** constrains **`windowSize`** (1–100) and **`MemoryType`** enum.

### How does **Python codegen** use memory?

**`pythonCodeGenerator.ts`** emits **`ConversationBufferWindowMemory`** (and related wiring) when **`stages.memory.type !== 'none'`**, using **`windowSize`** as **`k`**. The Designer only sets **configuration**; runtime libraries execute memory.

### Which evaluation metrics can users select?

**`EvaluationMetricName`**: **faithfulness**, **answer_relevance**, **context_precision**, **context_recall**, **latency**. The UI toggles checkboxes; **`metrics`** is stored as an array. **`testSetSize`** is clamped **10–1000** by schema; **`schedule`** is **on-demand** or **continuous** (continuous assumes worker/cron support later).

### What validation applies?

- **`RoutingConfigSchema`**: **`enabled`** boolean; each rule has **`condition`** enum, **`targetModel`** non-empty; optional **`threshold`**, **`keywords`**.  
- **`MemoryConfigSchema`**: **`type`** enum; optional **`windowSize`**, **`maxTokens`**, **`sessionPersistence`**.  
- **`EvaluationConfigSchema`**: **`enabled`**; optional **`metrics`** array of allowed metric names; **`testSetSize`** optional int range; **`schedule`** optional enum.

### How does **`StageNavigator`** summarize these stages?

- **`routingHint`**: **Off** / **On · N rules** (or **no rules** when enabled but empty).  
- **`memoryHint`**: Short label (**None**, **Buffer**, **Summary**, **Vector**).  
- **`evaluationHint`**: **Off** / **On · N metrics**.

### How do these stages relate to **P2-6 / P2-7** backend services?

**Generation** execution remains **`GenerationService`**. **Evaluation** configuration informs **`Evaluation Engine`** jobs and RAGAS-style runs but does not execute tests inside the Designer. **Routing** may be realized in application code or LangGraph; the draft is the **contract** for export.

### What remains placeholder after P5-9?

Only **Review** and later **P5-10+** tasks (**pipeline visualizer**, **cost**, **export**, **template gallery**) remain as separate milestones; **routing** through **evaluation** are interactive.

### How would you add a new routing condition type?

1. Extend **`RoutingRule.condition`** in **`pipeline.ts`** and **`RoutingRuleSchema`** in **`validators.ts`**. 2. Add UI controls in **`RoutingConfigurator`** for the new branch. 3. Update **`pythonCodeGenerator`** / **`yamlGenerator`** / **`mermaidGenerator`** if exports should emit logic for that condition.

### How would you add a new evaluation metric?

1. Add to **`EvaluationMetricName`** and **`EvaluationMetricNameSchema`**. 2. Add a checkbox row in **`EvaluationConfigurator`**. 3. Teach **`Evaluation Engine`** (backend) to compute it.

---

## Phase 5 · P5-10 · Pipeline Visualizer

### What does **P5-10** deliver?

A **live pipeline preview** in Designer mode: a **text summary** (one-line chain plus bullets) and a **Mermaid flowchart** that updates whenever **`draft`** changes in **`useDesignerStore`**. It appears **below the stage list** on **`lg+`** and as a **collapsible “Pipeline preview”** above the main panel on **narrow** screens.

### Where is it mounted?

**`DesignerShell`** renders **`PipelineVisualizer`** with **`placement="sidebar"`** next to **`StageNavigator`** and **`placement="main"`** above **`children`**. Only one placement is visible per breakpoint so **Mermaid** runs once.

### How is the diagram produced?

**`generateMermaidDiagram(stages, cloudProvider)`** in **`apps/web/src/lib/generators/mermaidGenerator.ts`** builds a **`flowchart LR`** string with **`subgraph`** blocks for **Indexing** (source → chunk → embed → vector store) and **Query** (user query → optional memory → retrieve → optional rerank → optional router → generate → answer → optional evaluate). **`VS --> RET`** connects the index to retrieval across subgraphs.

### Why fix edges for memory and routing?

Earlier drafts duplicated **`QUERY --> RET`** when memory was on and attached **`ROUTER`** directly from **`QUERY`**, which misrepresented runtime order. The generator now chains **memory between query and retrieval** and places **routing between the post-retrieval tail and generation**.

### What is **`generatePipelineHighlights`**?

A companion helper returning **short bullet strings** (cloud, ingestion, chunking, embedding, vector store, retrieval, reranking, generation, routing, memory, evaluation) for the **summary list** next to the SVG.

### How does client rendering work?

**`mermaid.render(id, definition)`** runs in **`useEffect`** after **`mermaid.initialize({ theme, securityLevel: 'loose' })`**. Theme follows **`.dark`** on **`documentElement`** and **`prefers-color-scheme`**. A **loader** shows while **SVG** is generated; errors surface inline.

### Does the visualizer persist anything?

No. It **reads** **`draft`** only; persistence stays **`persist`** middleware on **`useDesignerStore`** (same as other configurators).

### How does this relate to **exports**?

**`generateMermaidDiagram`** is shared with **codegen / export** flows from **P3** utilities; the Designer preview uses the **same canonical graph** shape as exported diagrams where applicable.

### Trade-offs of **Mermaid in the browser**

**Pros**: Diagram stays aligned with **shared generator** logic; no extra graph library. **Cons**: Bundle includes **mermaid**; **SSR** does not render SVG (client-only). Mitigation: diagram mounts only after breakpoint selection and shows a **loading** state.

### Accessibility notes

The SVG container uses **`role="img"`** with **`aria-label="Pipeline flow diagram"`**; the collapsible mobile panel uses a **`summary`** control with **focus-visible** ring.

### How would you add **guardrails** to the picture?

Extend **`generateMermaidDiagram`** (or pass **`guardrails`** into a wrapper) to draw optional nodes for **input / retrieval / output** guard stages when **`draft.guardrails`** is non-null, matching **`GuardrailsConfig`** in **`pipeline.ts`**.

### Interview trap: why **`placement`** props instead of CSS-only hide?

Hiding with **`hidden`** still mounts children; **two** **Mermaid** instances would **double-render** and fight for **`innerHTML`**. **`placement`** plus **`matchMedia`** ensures **one** active diagram instance.

### What remains after **P5-10** for Designer polish?

**Code export UI** (**P5-12**), **review page** (**P5-13**), and **template gallery** (**P5-14**) remain separate milestones. **P5-11** (cost estimator) is implemented — see the **Phase 5 · P5-11** section below.

---

## Phase 5 · P5-11 — Cost Estimator Component

### What problem does the cost estimator solve?

Designers need **approximate USD** visibility while tuning models, chunk sizes, **`top_k`**, vector store, and reranking so they can compare trade-offs before exporting or deploying. The UI shows **per-query** and **monthly** totals plus a **line-item breakdown** aligned with backend heuristics.

### Which API does the frontend call — is `/api/designer/cost` different?

**`POST /api/designer/cost`** and **`POST /api/utilities/cost`** both accept **`CostRequest`** and return **`CostEstimateSchema`** via the same **`CostEstimator`** / **`CostService`** logic. The **P5-11** UI calls **`/api/utilities/cost`** so Autopilot or other clients can share the same URL family for **validate / cost / rag-preview** without implying a persisted Designer project; either endpoint would be correct for a pure estimate.

### What is in the request body?

- **`config`**: full **`PipelineConfiguration`** from **`useDesignerStore`** (camelCase JSON).
- **`queriesPerMonth`**, **`documentsCount`**, **`avgDocumentTokens`**: defaults **100_000**, **1_000**, **500** match **`CostRequest`** in **`apps/api/app/schemas/designer.py`** when omitted on the server; the UI exposes them so users can stress-test scale assumptions.

### How does the backend compute costs?

**`CostEstimator`** in **`apps/api/app/utils/cost_calculator.py`** loads **`pricing.json`**, then combines **query-path embedding** (chunk tokens × embedding $/1M, scaled by retrieval strategy variants), **generation** (context + output tokens × model input/output $/1M), optional **reranking** (**`costPer1KQueries`**), **monthly storage** (estimated vector GB × provider $/GB/mo), and **retrieval read/query** units where priced. It returns **`CostEstimateSchema`** with **`breakdown`** rows **`embedding`**, **`vector_storage`**, **`retrieval_ops`**, **`reranking`**, **`generation`**.

### Why debounce the client requests?

Every slider or text change updates **`draft`** frequently. A **~450 ms** debounce limits load on the API and avoids flickering numbers while the user is still dragging controls.

### Why keep workload assumptions out of Zustand?

**`persist`** middleware writes **localStorage** on each **`draft`** change. Bumping **`queriesPerMonth`** on every keystroke would **re-serialize** the whole pipeline object unnecessarily. Local **`useState`** in **`CostEstimator`** keeps assumptions ephemeral unless we later add an explicit “save assumptions” feature.

### How are API errors surfaced?

**`apiClient.post`** throws **`ApiError`** on non-2xx responses. The component reads **`detail`** from FastAPI JSON when present (e.g. **503** if **`pricing.json`** is missing) and renders an **`role="alert"`** banner.

### How does the UI map cryptic breakdown **`component`** strings?

**`componentLabel`** maps **`vector_storage`**, **`retrieval_ops`**, etc., to human-readable labels for the table; percentages come from the API (**`percentage`** field on each **`CostBreakdown`**).

### Interview trap: **`per_month`** vs sum of category fields

**`perMonth`** / **`total`** should match the sum of **monthly** category amounts (**embedding + storage + retrieval + reranking + generation**); **`breakdown`** rows use the same underlying totals with **percentage** split. If they diverge, that would be a backend bug — the UI displays API values as-is.

### How would you cache or dedupe cost calls across tabs?

Use **SWR** / **TanStack Query** with a **`queryKey`** of **`[ 'cost', digest ]`**, or a small server-side cache keyed by **hash(config)+assumptions** with short TTL — trade freshness vs load.

### What remains after **P5-11**?

**P5-12** code export UI, **P5-13** review page, **P5-14** template gallery.

---

## Phase 5 · P5-12 — Code Export Component

### What does **P5-12** deliver?

A **Designer-mode export viewer** mounted in **`DesignerShell`** between **`CostEstimator`** and **`PipelineVisualizer`**. Users pick one of five **formats** aligned with **`ExportFormat`** in **`apps/api/app/schemas/designer.py`**: **python**, **yaml**, **terraform**, **docker-compose**, **k8s**. The UI shows generated **source text**, **copy** and **download** actions, and optional **deploy hints** (starter shell commands, copyable separately).

### Which endpoint does the UI call?

**`POST /api/designer/export`** on the FastAPI **`designer`** router (**prefix** **`/api/designer`**). The Next.js app uses **same-origin** **`apiClient.post('/api/designer/export', …)`** so **`next.config.js`** rewrites proxy to the backend without CORS issues.

### What is the JSON contract?

**Request**: **`{ config, format }`** where **`config`** is the full **`PipelineConfiguration`** from **`useDesignerStore`** (camelCase keys, **`RAGBaseModel`** symmetry with Pydantic). **Response**: **`DesignerExportResponse`** — **`code`**, **`filename`**, **`format`**, **`contentType`** — matching **`ExportResponse`** with **camelCase** aliases.

### How does debouncing interact with format changes?

**`draftDigest`** is **`JSON.stringify(draft)`**. An effect schedules **`fetchExport`** with **~450 ms** debounce when the digest changes (user editing stages). When **`format`** alone changes, **`prevFormat`** detects a **tab switch** and uses **0 ms** delay so the new artefact appears immediately.

### Why keep export output out of Zustand?

Export blobs can be **large** and change on every keystroke if stored globally. Keeping **`result`** in **local React state** avoids **`localStorage`** **`persist`** bloat and keeps the store focused on the **editable** pipeline contract.

### How do **Copy** and **Download** work?

**Copy** uses **`navigator.clipboard.writeText(result.code)`** with a short **“Copied”** affordance. **Download** builds a **`Blob`** from **`result.code`** with **`result.contentType`**, then triggers an **`<a download={filename}>`** programmatic click and revokes the object URL.

### What does **Deploy hints** mean if we do not auto-deploy?

**No cloud APIs** run from the browser. **`deploy-hints.ts`** returns **educational** commands (e.g. **`docker compose -f … up`**, **`kubectl apply -f …`**, **`terraform init`**) so operators have a **starting checklist**. Users must still supply **secrets**, **kube context**, and **TF backend** configuration.

### Why sanitize filenames in deploy hints?

User-controlled **`draft.name`** flows into suggested **`filename`** values from **`ExportService._filename_for`**. **`deployHintCommand`** replaces characters outside a safe set when interpolating into shell snippets to reduce **copy-paste injection** surprises (defence in depth; not a substitute for reviewing commands).

### How does this relate to **P3** code generators?

**P4-4** server-side **`ExportService`** is the **source of truth** for downloadable artefacts. **P3** TypeScript generators (**`pythonCodeGenerator`**, **`yamlGenerator`**, etc.) support **client-side previews** and **consistency tests**; the **Designer export UI** always displays **API-generated** output so **deployable bytes** match what operators save from the backend.

### Interview trap: **`/api/utilities/export`**?

There is **no** utilities export route in this codebase — exports live under **`/api/designer/export`** because they are **Designer pipeline** artefacts. **Cost** optionally uses **`/api/utilities/cost`** for shared access, but export is **Designer-scoped**.

### How would you add **zip multi-file export**?

Extend **`ExportService`** with a **`format: "bundle"`** that streams a **zip** (Python + compose + k8s), return **`contentType: application/zip`**, and teach the frontend to **download** binary **`ArrayBuffer`** instead of UTF-8 text only.

### How would you add **syntax highlighting**?

Integrate **Prism** (already a dependency) or **Shiki** in a **`useEffect`** that highlights **`result.code`** when **`format`** changes; guard **SSR** (highlight runs client-only).

### What automated tests cover the new behaviour?

**`deploy-hints.test.ts`** (**Vitest**) asserts command strings for **python**, **docker-compose**, **k8s**, and partial checks for **terraform**. The **React** layer can be covered later with **RTL** + **MSW** if you want to mock **`/api/designer/export`**.

### What remains after **P5-12**?

**P5-14** template gallery (and later phases).

---

## Phase 5 · P5-13 · Designer Review Page

### What problem does the Review stage solve?

After eleven configuration stages, operators need a **single checkpoint** that surfaces the **whole draft** (names, key parameters, flow bullets) without hunting through each step again, while keeping **cost**, **export**, and **live Mermaid** one scroll away. Review also wires **deep links** into the existing footer strips so we do not duplicate API calls or diagram rendering logic.

### Where does the Review UI live in the component tree?

**`/designer/review`** resolves through **`apps/web/src/app/designer/[[...step]]/page.tsx`** → **`DesignerStepView`** → **`DesignerStagePlaceholder`**, which **early-returns** **`DesignerReviewPage`** when **`stageId === 'review'`**. **`DesignerShell`** still wraps all designer routes and continues to mount **`CostEstimator`**, **`CodeExporter`**, and **`PipelineVisualizer`** below the scrollable main pane.

### Why add **`designer-section-anchors.ts`**?

Three stable DOM **`id`** values (**`designer-section-cost`**, **`designer-section-export`**, **`designer-section-pipeline`**) are shared between **`DesignerShell`** (passed into each footer **`section`**) and **`DesignerReviewPage`** (scroll targets). A tiny module avoids string drift and keeps Review buttons aligned with the visual stack order (**cost → export → graph**).

### How does smooth scrolling behave with a sticky layout?

Each footer **`section`** accepts an optional **`id`** and adds **`scroll-mt-4`** so **`scrollIntoView({ behavior: 'smooth', block: 'start' })`** leaves a small offset under fixed chrome / borders and the target is not flush-hidden at the viewport edge.

### How is the textual summary produced?

**`generatePipelineSummary`** and **`generatePipelineHighlights`** from **`mermaidGenerator.ts`** are called with **`maxVisitedStageIndex = DESIGNER_STAGES.length - 1`** so Review always reflects the **full draft**, even if the progressive diagram still reflects a lower **`diagramMaxVisitedStageIndex`** until the user has navigated through every stage.

### What actions are offered besides jumping to footer strips?

- **Copy text summary** — pipeline name, one-line flow, and bullet list (clipboard).  
- **Copy draft JSON** — pretty-printed **`PipelineConfiguration`** for tickets or API replays.  
- **Reset draft** — calls **`resetDraft()`** after **`window.confirm`** (clears persisted Zustand state in the browser).  
- **Stage checklist** — links back to each non-review **`DESIGNER_STAGES`** path for quick edits.  
- Footer links: **home**, **templates**, **projects**.

### Why not embed a second Mermaid instance inside Review?

**`PipelineVisualizer`** already owns Mermaid init, zoom, PiP, and invalidation. Duplicating it would double CPU work and risk theme drift. Review **navigates** to the existing section instead.

### Interview trap: does Review save to PostgreSQL?

Not in this task. The draft is still **`persist`**’d **`useDesignerStore`** to **`localStorage`**; **Projects** / **Designer Config API** persistence is a separate workflow. The Review page links to **`ROUTES.projects`** for discoverability only.

### How would you add “Save to project” on Review?

Add a button that **`POST`s** **`/api/designer/config`** (or projects sub-resource) with **`draft`**, handle loading/errors, show toast, and optionally redirect to **`/projects/{id}`** — keeping Review as an orchestration surface, not a second source of truth.

### What automated tests would you add next?

**RTL** test that **`DesignerReviewPage`** renders summary cards from a **hydrated** store fixture; **Playwright** journey that lands on **`/designer/review`** and asserts **`#designer-section-cost`** exists after clicking **“Cost estimate”**.

### What remains after **P5-13**?

**P5-14** template gallery.

---

## Phase 5 · P5-14 · Template Gallery Page

### What user problem does the Template Gallery solve?

Operators often want a **known-good baseline** (chunk sizes, models, vector store) instead of configuring eleven stages from scratch. The gallery surfaces **catalog metadata** (use case, complexity, cost band, tags, providers) and offers **one-click persistence** via the **Templates** and **Projects** APIs, then hands off to **Designer** with the saved **`PipelineConfiguration`**.

### Where does the UI live?

**`/templates`** is implemented by **`apps/web/src/app/templates/page.tsx`**, which renders **`TemplateGallery`** (**`apps/web/src/components/templates/template-gallery.tsx`**). The route is already linked from the **navbar**, **Designer** placeholders, and **Review** cross-links.

### Which backend endpoints does the gallery call?

- **`GET /api/templates`** — returns **`TemplatesCatalogResponse`** (version, description, **`templates[]`** with full **`config`** per entry, camelCase JSON).  
- **`GET /api/projects?page=1&page_size=50`** — populates the **“Existing project”** dropdown when applying a template.  
- **`POST /api/projects/`** — optional when the user chooses **“New project”** before apply.  
- **`POST /api/templates/{templateId}/apply`** — body **`{ projectId, name? }`**; responds **201** with **`ApplyTemplateResponse`** (saved config row plus **`templateId`**).

### Why both “Use template” and “Preview locally”?

**Preview locally** only calls **`loadPipeline(template.config)`** and navigates to **`/designer/review`** — no database row, useful for demos when the API is down. **Use template** persists a **Designer config** against a **real project UUID** so the pipeline can be reloaded, shared, and extended in later phases.

### How is client state kept consistent with the server?

After a successful apply, **`loadPipeline(applied.config)`** hydrates **`useDesignerStore`**. **`useProjectStore`** is updated: **`addProject`** when the server project is new to this browser, or **`updateProject`** with **`linkedPipelineId`** when the UUID already exists locally, then **`setActiveProject`**.

### Why default “New project” in the apply dialog?

The **sidebar project list** is still primarily **local-first**; not every browser session has already fetched server projects. Defaulting to **create** avoids an empty **Existing** selection and matches the common “start a workspace from this preset” mental model.

### How do search and complexity filters work?

Filtering is **client-side** over the catalog payload: a case-insensitive substring match across **name**, **description**, **useCase**, and **tags**, plus an optional **complexity** chip (**all / beginner / intermediate / advanced**).

### What UX stack is used for the apply modal?

**Radix Dialog** (**`@radix-ui/react-dialog`**) provides focus management and overlay semantics, consistent with other Radix usage in the app (**dropdown menu** in the navbar).

### Interview trap: does **`GET /api/templates`** return only summaries?

No — each catalog entry includes the **full** embedded **`config`** (same shape as **`PipelineConfiguration`**). The UI only *displays* metadata on cards to keep the layout light; **Preview** / **Apply** reuse the embedded object without a second **`GET /api/templates/{id}`** round trip.

### What error states should you expect?

- **Catalog load failure** — misconfigured **`TEMPLATES_CATALOG_PATH`**, missing **`data/templates.json`**, or API offline; the page shows a destructive-styled alert with the **proxy hint** from **`apiClient`**.  
- **Apply failure** — invalid **`projectId`**, unknown template id, or DB constraint; **`ApiError`** **`detail`** is surfaced in-dialog.

### What automated tests already guard this flow?

**`apps/api/tests/test_templates.py`** covers list/get/apply happy paths and 404 cases. Frontend tests could add **MSW** + **RTL** for **`TemplateGallery`** fetch/apply if you want CI coverage without a live API.

### What remains after **P5-14**?

**Phase 6** Autopilot (**LangGraph** agents and orchestration).

---

## Phase 6 · P6-1 · LangGraph Agent Infrastructure

### Why introduce LangGraph for Autopilot instead of a single Celery loop?

Autopilot coordinates **multiple decision steps** (corpus analysis, chunking search, embedding benchmarks, retrieval tuning, evaluation). LangGraph gives a **first-class state machine**: explicit nodes, conditional edges, **checkpointing** for resume/cancel, and **message history** compatible with LangChain chat models. A flat Celery task is fine for linear stubs, but it does not model **branching, retries, or human-in-the-loop** without reinventing a graph runtime.

### What lives under `app/core/agents/`?

| Module | Role |
|--------|------|
| `state.py` | **`AutopilotGraphState`** TypedDict with reducers: **`add_messages`**, **`operator.add`** for `agent_trace`, **`merge_stage_outputs`** for `stage_outputs`. |
| `prompts.py` | Shared **system** and **stage** prompt strings plus **`format_stage_delegation`** for consistent logging / LLM context. |
| `tools.py` | **`@tool`** definitions and **`get_autopilot_bootstrap_tools()`** registry (stubs until P6-2+ wire real services). |
| `graph.py` | **`compile_autopilot_bootstrap_graph`** — minimal **prepare → finalize** graph proving imports, reducers, and optional **`MemorySaver`** checkpointer. |

### How does `AutopilotGraphState` map to the database?

`AutopilotBuild` already stores **`requirements`**, **`stages`**, **`messages`**, **`result`**. The graph state is the **in-memory execution view**: `agent_trace` entries can be appended to **`messages`** JSON on the ORM row; `stage_outputs` keys align with **`stages`** and future **`AgentDecisionSchema`**. `pipeline_config` mirrors optional **`PipelineConfiguration`** from Designer handoff.

### What is `AUTOPILOT_STAGE_ORDER` and why import it from the worker?

It is the **canonical ordered tuple** of specialist stages (`analyze` … `evaluation`). **`app.worker.tasks`** previously duplicated the tuple as `_AUTOPILOT_STAGE_KEYS`; P6-1 **imports the same tuple** so stub progress and LangGraph subgraph ordering **cannot drift**.

### Interview trap: why `Annotated[list[AnyMessage], add_messages]`?

LangGraph merges node outputs per key. Without a reducer, the last write would **replace** the entire message list. **`add_messages`** concatenates and dedupes by message id so **LLM and tool messages** accumulate correctly across turns.

### How do you test a graph without an LLM?

The bootstrap and document-analyst nodes return **`AIMessage`** and dict updates only — **no model call**. Tests invoke **`invoke_autopilot_bootstrap`** with and without **`MemorySaver`** to validate compilation, reducers, and terminal **`current_stage == "analyze_complete"`** (P6-2 onward) plus populated **`stage_outputs["analyze"]`**.

### What is the next task after P6-1?

**P6-2 · Document Analyst Agent** is implemented: the compiled graph runs **bootstrap → document analyst**, and **`stage_outputs["analyze"]`** carries corpus summary plus chunking recommendations. The next milestone is **P6-3 · Chunking Optimizer Agent**.

---

## Phase 6 · P6-2 · Document Analyst Agent

### What problem does the Document Analyst solve?

Autopilot must choose **chunking** before embeddings. The analyst turns **corpus-level signals** (file types, rough size, markdown/code/tabular hints from optional **`corpus_profiles`**) into a **catalog-aligned strategy id** (`fixed-size`, `recursive-character`, `semantic`, `markdown-header`, `code-aware`, `paragraph-based`) plus **suggested chunk size / overlap**, so downstream agents (P6-3) start from an evidence-based default instead of a random preset.

### Where does `corpus_profiles` come from?

**Not invented in the graph.** Ingestion / upload flows (future P7 / worker wiring) should attach a JSON-serializable list under **`requirements["corpus_profiles"]`** after documents are parsed. If it is missing, the analyst **synthesizes** one profile per **`document_id`** with **`file_type: "unknown"`**, which still completes the stage but produces a **conservative** recommendation (`recursive-character`) and a trace note.

### Why keep heuristics in pure Python instead of only an LLM?

- **Deterministic CI** — same inputs yield the same **`primary_strategy`**, so unit tests guard regressions.  
- **No API key** required for the default Autopilot path.  
- An LLM can still be added later as a **refinement node** that consumes `stage_outputs["analyze"]`.

### How does `recommend_chunking` choose a strategy?

Priority-ordered rules on **`corpus_summary["signals"]`** and **`optimize_for`**: **code-heavy** → `code-aware`; **markdown structure** → `markdown-header`; **tabular / CSV** → `fixed-size`; **quality** + multiple non-trivial docs → `semantic`; **latency** → `fixed-size`; else **`recursive-character`** as the general default. Alternates are returned for the Chunking Optimizer to search around.

### What does the LangGraph node emit?

**`document_analyst`** returns updates: **`current_stage: "analyze_complete"`**, **`stage_outputs["analyze"]`** (`status`, `corpus_summary`, `chunking_recommendation`, `profile_count`), **`agent_trace`** with `event: "document_analyst"`, and an **`AIMessage`** summarising counts, dominant types, and the primary strategy (Markdown bold on strategy name for UI parsers).

### Which LangChain tools exist for P6-2?

| Tool | Role |
|------|------|
| `document_corpus_analyze` | Full analyst JSON from `document_ids_json` + `requirements_json`. |
| `summarize_corpus_profiles_json` | Returns **`corpus_summary`** only from a profiles array. |
| `recommend_chunking_from_summary_json` | Returns **`chunking_recommendation`** from a summary + requirements. |

They mirror the same logic as the graph node so a future **tool-calling LLM** can replan without duplicating business rules.

### Interview trap: does the graph still end at `bootstrap_complete`?

**No after P6-2.** The terminal **`current_stage`** is **`analyze_complete`**. Bootstrap remains **`stage_outputs["bootstrap"]`** with `next: "document_analyst"`.

### How do tests cover the analyst without real PDFs?

**`tests/test_core/test_document_analyst.py`** builds synthetic **`corpus_profiles`** and asserts strategy ids. **`test_agents_langgraph.py`** asserts **`stage_outputs["analyze"]["status"] == "complete"`** and extended **`agent_trace`** length after **`invoke_autopilot_bootstrap`**.

### What is the next task after P6-2?

**P6-3 · Chunking Optimizer Agent** — search / evaluate alternative chunking parameters around the analyst’s recommendation.

---

## Phase 6 · P6-3 · Chunking Optimizer Agent

### What problem does the Chunking Optimizer solve?

The analyst proposes **one primary** strategy plus **alternates**, but real chunk quality depends on **tokenizer boundaries**, **overlap**, and **strategy-specific behaviour**. The optimizer **materialises candidate `ChunkingConfig` rows**, runs the production **`ChunkingService`** on a **small benchmark corpus**, scores outputs with **`ChunkQualityScorer`**, and picks the **highest composite score** so Autopilot exports a defensible `chunk_size` / `chunk_overlap` / `strategy` triple—not only a rule-based guess.

### Why benchmark on synthetic text instead of full uploads?

The LangGraph state carries **`document_ids` and optional `corpus_profiles`**, not full file bytes. Until ingestion streams text into the worker, the optimizer builds (or accepts) **short representative documents**: default **signal-aware synthetic prose/markdown/code/tabular snippets**, or optional **`requirements["chunking_sample_documents"]`** for tests and future wiring that passes real snippets from storage.

### How is the composite score defined?

Per candidate, chunks are scored with **`ChunkQualityScorer`** (density, sentence completeness, proximity to target `chunk_size`). The mean **overall** metric is adjusted by **`optimize_for`**: **latency** penalises high chunk counts; **cost** penalises many chunks (embedding calls); **quality** gives a small bonus for reasonable fragmentation; **balanced** uses the mean alone.

### What happens if a strategy fails at runtime (e.g. semantic embeddings)?

Each candidate is wrapped in **try/except**; failures become **`candidates_tried[].error`** strings and **do not crash** the graph. If **every** candidate fails, **`stage_outputs["chunking"].status`** is **`failed`** with **`reason: all_chunking_benchmarks_failed`**.

### How does the graph topology change after P6-3?

**Linear:** `bootstrap_prepare` → `bootstrap_finalize` → `document_analyst` → **`chunking_optimizer`** → END. The terminal **`current_stage`** is **`chunking_complete`** (success path) even when chunking payload records an internal failure, so orchestrators can rely on a single stage pointer.

### What keys does `stage_outputs["chunking"]` contain on success?

| Key | Role |
|-----|------|
| `status` | `complete` or `failed` |
| `selected` | `strategy`, `chunk_size`, `chunk_overlap`, `composite_score`, `rationale` |
| `candidates_tried` | List of per-candidate metrics (`chunk_count`, `composite_score`, optional `error`) |
| `alternatives_tested` | Strategy ids of non-winning successful runs (aligns with API “alternatives tested” wording) |
| `benchmark_document_count` | How many LangChain documents were chunked |

### Which LangChain tool was added for P6-3?

**`chunking_optimizer_run(analyze_json, requirements_json)`** — accepts the **same JSON shape** as `stage_outputs["analyze"]` plus requirements; returns **optimizer JSON** for future tool-calling LLMs.

### Interview trap: is `current_stage` still `analyze_complete` after the full bootstrap graph?

**No after P6-3.** After a successful end-to-end invoke, **`current_stage` is `chunking_complete`**. The analyst node still sets **`analyze_complete` internally before merge**, but the **last write wins** on the scalar field.

### How do tests cover the optimizer without large models?

**`tests/test_core/test_chunking_optimizer.py`** exercises candidate deduplication, JSON tool errors, and deterministic benchmarks with **`chunking_sample_documents`**. **`test_agents_langgraph.py`** asserts **`stage_outputs["chunking"]["status"] == "complete"`** and extended **`agent_trace`** after **`invoke_autopilot_bootstrap`**.

### What is the next task after P6-3?

**P6-4 · Embedding Tester Agent** — benchmark embedding models against cost/latency/quality goals.

---

## Phase 6 · P6-4 · Embedding Tester Agent

### What problem does the Embedding Tester solve?

Chunking output fixes **text boundaries**, but **retrieval quality, bill, and tail latency** still depend on the **embedding model**. The tester **materialises catalog-backed `EmbeddingConfig` rows**, embeds a **small text sample** aligned to the winning chunker, measures **throughput and per-text latency** via **`EmbeddingBenchmarker`**, and fuses those signals with **catalog MTEB scores and `costPer1MTokens`** so Autopilot picks a **single provider/model** (plus rationale) before retrieval optimisation.

### Where do benchmark texts come from?

By default the service **re-chunks** the same **signal-aware synthetic corpus** used in P6-3 using the **selected** `strategy` / `chunk_size` / `chunk_overlap`. Tests and future workers can inject **`requirements["embedding_sample_texts"]`** (plain string list) to avoid provider calls or stabilise CI.

### How is the winner chosen if live benchmarks only return some providers?

**`EmbeddingBenchmarker`** drops failing configs (missing API keys, unsupported models). The agent builds **`candidates_tried`** including **error rows** for configs that never returned vectors. If **every** config fails, **`stage_outputs["embedding"].status`** is **`failed`** with **`all_embedding_benchmarks_failed`**. Otherwise the **highest composite score** among successful runs wins; weights depend on **`optimize_for`** (`quality`, `latency`, `cost`, `balanced`).

### How does the LangGraph topology change after P6-4?

**Linear extension:** `chunking_optimizer` → **`embedding_tester`** → END. Terminal **`current_stage`** is **`embedding_complete`**.

### What keys does `stage_outputs["embedding"]` expose on success?

| Key | Role |
|-----|------|
| `status` | `complete` or `failed` |
| `selected` | `provider`, `model`, `dimensions`, throughput/latency, `composite_score`, `rationale` |
| `candidates_tried` | Per-run metrics or `error` markers for skipped providers |
| `benchmark_text_count` | Number of strings embedded in the timed run |

### Which LangChain tool was added for P6-4?

**`embedding_tester_run(chunking_json, analyze_json, requirements_json, pipeline_config_json?)`** — mirrors the graph inputs so a future **tool-calling LLM** can rerun or explain the stage without invoking the full graph.

### Interview trap: does CI call OpenAI for every pytest graph invoke?

**No.** **`tests/test_core/test_agents_langgraph.py`** patches **`EmbeddingBenchmarker`** inside **`embedding_tester`** with stub **`BenchmarkResult`** rows so the graph and tool smoke tests stay **offline and deterministic**, while production code paths still call real embedders when keys are present.

### What is the next task after P6-4?

**P6-5 · Retrieval Optimizer Agent** — tune retrieval mode, top-k, hybrid weights, and reranking.

---

*Append new `## Phase … · …` sections at the end for future tasks; keep all prior sections intact.*

---

## Phase 6 · P6-5 · Retrieval Optimizer Agent

### What does the Retrieval Optimizer add to the Autopilot graph?

**P6-5** extends the compiled bootstrap graph with a **`retrieval_optimizer`** node after **`embedding_tester`**. The terminal **`current_stage`** becomes **`retrieval_complete`**, and **`stage_outputs["retrieval"]`** holds the winning **strategy**, **top_k**, optional **hybrid α / fusion**, **MMR λ**, and **reranking** flags plus **`candidates_tried`** and a short **rationale**.

### Why benchmark retrieval without calling `RetrievalService` or a vector DB?

Autopilot runs **before** a project may have a populated index. The optimizer therefore reuses the **same chunk texts** as P6-4 (synthetic corpus + winning chunker, or **`requirements["retrieval_sample_texts"]`**), scores **hashing-vector “dense”** similarity and **BM25** lexically, and applies the same **fusion patterns** as production (**RRF**, **weighted** hybrid, **MMR**, **multi-query** RRF, **ensemble** RRF). That yields a **deterministic, CI-safe** trade-off surface; live Qdrant/PG paths remain in **P2-5** for runtime.

### How is a “good” retrieval setting measured offline?

For each evaluation query, the code treats the **BM25 argmax chunk** as a lexical oracle and computes **mean reciprocal rank (MRR)** of that chunk in the candidate ranking. That score is normalised across candidates and fused with a **latency proxy** (heavier strategies and higher **top_k** cost more) using **`optimize_for`** (`quality`, `latency`, `cost`, `balanced`) — analogous to the embedding tester’s composite.

### What happens if chunking yields only one chunk?

The optimizer **duplicates** the single chunk with a small suffix so BM25 and ranking logic always have **≥2** documents, avoiding degenerate benchmarks while still reflecting the real chunk content.

### Which LangChain tool exposes P6-5?

**`retrieval_optimizer_run(embedding_json, chunking_json, analyze_json, requirements_json, pipeline_config_json?)`** returns JSON aligned with **`stage_outputs["retrieval"]`**, so a future tool-calling LLM can re-run or explain the stage.

### Interview trap: does the graph still end at `embedding_complete` after P6-5?

**No.** Successful runs end at **`retrieval_complete`**. The embedding node still sets **`embedding_complete`** on that hop, but the graph’s **final** `current_stage` after **`retrieval_optimizer`** is **`retrieval_complete`**.

### What is the next task after P6-5?

**P6-6 · Evaluation Agent** — synthetic or curated test sets, pipeline evaluation, and failure diagnosis.

---

## Phase 6 · P6-6 · Evaluation Agent

### What does the Evaluation Agent add to the Autopilot graph?

**P6-6** appends an **`evaluation_agent`** node after **`retrieval_optimizer`**. The terminal **`current_stage`** is **`evaluation_complete`**, and **`stage_outputs["evaluation"]`** holds **offline lexical proxies** for **faithfulness**, **answer_relevance**, **context_precision**, and **context_recall**, plus **`failure_analysis`** (from **`analyze_failures`**), **`meets_targets`**, optional **`target_gaps`** vs **`requirements["target_metrics"]`**, **`per_row_scores`**, and a short **`rationale`**.

### Why use proxies instead of calling the P2-7 RAGAS engine inside the graph?

Autopilot’s LangGraph bootstrap must stay **deterministic and CI-safe** without API keys. **RAGAS** (real LLM judges) runs asynchronously via **Celery** **`run_evaluation`** when persisted **`EvaluationExample`** rows exist. The agent reuses **P6-5 chunk texts** and the **selected retrieval ranking** to fabricate **extractive answers** and score **token-overlap** stand-ins that correlate with “did retrieval surface the oracle chunk?” — enough to drive **`failure_analysis`** and **target gating** before deployment agents.

### How are test rows constructed?

For each query from **`_build_eval_queries`** (or **`requirements["retrieval_eval_queries"]`**), the code sets **oracle** = **BM25 argmax** chunk, **contexts** = top-**k** chunks from **`_rank_indices_for_candidate`** using the **winning retrieval profile**, and **answer** = a trimmed excerpt from the **rank-1** chunk. That yields one **per-row** dict with scores the failure analyzer already understands.

### Which LangChain tool exposes P6-6?

**`evaluation_agent_run(retrieval_json, chunking_json, analyze_json, requirements_json, pipeline_config_json?)`** — mirrors graph inputs so a **tool-calling LLM** can rerun or explain evaluation without the full graph.

### Interview trap: does `meets_targets` default to true when no targets are set?

**Yes.** If **`requirements["target_metrics"]`** omits thresholds, nothing fails the gate; **`target_gaps`** stays empty. Tight **`faithfulness`** / **`answer_relevance`** / **`context_precision`** / **`context_recall`** values make **`meets_targets`** false when any proxy falls short.

### What is the next task after P6-6?

**P6-7 · Deployment Agent** — packaging/deployment artefacts and cloud deployer stubs.

---

## Phase 6 · P6-7 · Deployment Agent

### What does the Deployment Agent add to the Autopilot graph?

**P6-7** appends a **`deployment_agent`** node after **`evaluation_agent`**. The terminal **`current_stage`** is **`deployment_complete`**, and **`stage_outputs["deployment"]`** holds **`artefacts`** (`docker_compose`, `kubernetes_manifest`, `terraform_stub` strings), **`cloud_deployers`** (AWS/GCP/Azure **stub** records with **`apply_gated: true`**), **`synthesized_from`** (`pipeline_config` vs **`stage_outputs_fallback`**), optional **`warnings`**, **`rationale`**, and **`operator_notes`**.

### When does the agent use real P4 export generators vs fallback sketches?

If **`pipeline_config`** in graph state **`model_validate`s** as **`PipelineConfigurationSchema`**, the agent calls **`generate_docker_compose`**, **`generate_kubernetes`**, and **`generate_terraform`** from **`app.services.export_generators`**. If validation fails or config is absent, it emits **compact fallback** YAML/HCL derived from **`stage_outputs`** retrieval/embedding/chunking **`selected`** fields so CI still gets deterministic text.

### Why are cloud deployers “stubs”?

**Terraform apply**, **`kubectl apply`**, and cloud control-plane calls require **credentials, approvals, and network** access that must not run inside an unreviewed Autopilot graph tick. The payload therefore lists **`would_run`** commands and documentation links while **`apply_gated`** stays **true** until an operator promotes artefacts through your org’s pipeline (**P8-4** / **P12**).

### Does deployment require evaluation to succeed?

**No.** The graph always runs **after** the evaluation node, but **`run_deployment_agent`** only needs **complete** retrieval, embedding, and chunking stages. If **`evaluation.status` ≠ `complete`**, the payload adds a **warning** and still produces artefacts using the upstream selections.

### Which LangChain tool exposes P6-7?

**`deployment_agent_run(evaluation_json, retrieval_json, chunking_json, embedding_json, requirements_json, pipeline_config_json, build_id, project_id)`** — JSON in/out aligned with **`stage_outputs["deployment"]`**.

### What changed in `AUTOPILOT_STAGE_ORDER` for worker stub builds?

**`deployment`** is appended after **`evaluation`**, so **`jobs.run_pipeline_build`**’s Celery stub iterates **one more** stage key for progress messages until the real orchestrator replaces the stub.

### Interview trap: will Autopilot accidentally provision my AWS account?

**Not from this agent.** There are **no boto3 / google / azure SDK calls** here—only text generation and structured **“would run”** hints. Real provisioning belongs behind **explicit** deployment APIs and credentials (**Phase 8 / 12**).

### What is the next task after P6-7?

**P6-8 · Autopilot Orchestrator** — end-to-end LangGraph orchestration with iteration logic and progress events.

---

## Phase 6 · P6-8 · Autopilot Orchestrator

### What problem does P6-8 solve that P6-7 left open?

**P6-7** ended in a **strictly linear** LangGraph: **analyze → … → evaluation → deployment** with **no feedback loop** if **`meets_targets`** was false, and **no machine-readable progress** beyond ad-hoc trace dicts. **P6-8** adds an **`orchestration_gate`** after **evaluation**, **optional retries** back to **chunking** (bounded by **`requirements.max_iterations`**), **`evaluation_pass_index`** in state, **`autopilot_progress`** fields on trace rows, and wires **`run_pipeline_build`** to **`invoke_autopilot_orchestrator`** so workers run the **same** graph the API will stream (**P6-9**).

### How does the orchestration gate decide “retry” vs “deploy”?

The gate calls **`_orchestration_decision`**: **deploy** if evaluation **`status` ≠ `complete`**, if **`meets_targets`** is true, or if **`evaluation_pass_index` ≥ `max_iterations` − 1** (so **`max_iterations`** counts full optimisation attempts including the first). Otherwise it chooses **retry**, increments **`evaluation_pass_index`**, and LangGraph routes **`retry_chunking` → `chunking_optimizer`**. **Document analyst** is **not** re-run on retry (the prior **`stage_outputs["analyze"]`** still feeds chunking).

### Why loop back to chunking instead of retrieval or embedding only?

Chunking is the **first tunable knob** that changes downstream **embedding benchmarks**, **retrieval** rankings, and **evaluation** proxies in this codebase; looping there maximises **signal per retry** without multiplying **corpus profile** work. A future subgraph could branch to **embedding-only** retries if product requirements demand it.

### What is `evaluation_pass_index` and how does it differ from `iteration`?

**`evaluation_pass_index`** (P6-8) counts **post-evaluation retry scheduling**—the gate increments it when it chooses **retry**. **`iteration`** is still bumped once in **`bootstrap_finalize`** for backward compatibility with earlier tests and any UI that probed **`iteration == 1`** after a single forward pass.

### What shape do “progress events” take for SSE?

Every major **`agent_trace`** row merges **`progress_trace_fields`** so entries include **`kind: "autopilot_progress"`**, **`progress`** (0–100 using a **stage ladder** plus a **small bump per pass**), **`stage`**, **`detail`**, **`build_id`**, and **`evaluation_pass_index`**. Consumers can filter **`kind == "autopilot_progress"`** or surface **`event == "orchestration_gate"`** for orchestrator-specific lines.

### How does `jobs.run_pipeline_build` read document IDs and base pipeline config?

It copies **`row.requirements`** from the DB: **`document_ids`** must appear as **`requirements["document_ids"]`** (list of strings), and optional **`requirements["base_config"]`** (object) becomes **`pipeline_config`** in **`initial_autopilot_graph_state`**. Until **P6-9** persists these fields on enqueue, builds may run with **empty** `document_ids`—the document analyst still returns **minimal profiles**.

### Why does the worker mark the `generation` stage complete without a generation agent node?

**`AUTOPILOT_STAGE_ORDER`** still lists **`generation`** for **API contract parity** with **`BuildStatusResponse.stages`**. The P6-8 graph does **not** run a separate generation optimiser; the worker injects a **short message** so polling clients do not see a **perpetually pending** stage.

### What public entry points alias the orchestrator?

**`compile_autopilot_orchestrator_graph`** and **`invoke_autopilot_orchestrator`** are thin aliases over the **same** compiled graph as **`compile_autopilot_bootstrap_graph`** / **`invoke_autopilot_bootstrap`** (names kept for **import stability**).

### Interview trap: can retries run forever?

**No.** **`max_iterations`** is clamped to **1…10**; the gate **always** routes to **deployment** once **`evaluation_pass_index` ≥ `max_iterations` − 1** or evaluation is incomplete / targets met.

### What is the next task after P6-8?

**P6-9 · Autopilot API Endpoints** — HTTP APIs to start, poll, **SSE stream**, cancel, and fetch build results against the orchestrator.

---

## Phase 6 · P6-9 · Autopilot API Endpoints

### Which HTTP routes implement the Autopilot build lifecycle?

| Method | Path | Role |
|--------|------|------|
| `POST` | `/api/autopilot/build` | Create an `autopilot_builds` row (pending), persist **`requirements`** + **`document_ids`** + optional **`base_config`**, enqueue **`jobs.run_pipeline_build`**, store **`_celery_task_id`** — returns **`202 Accepted`**. |
| `GET` | `/api/autopilot/build/{id}` | Poll **`BuildStatusResponse`** (stages, messages, progress, optional typed **`result`**). |
| `GET` | `/api/autopilot/build/{id}/stream` | **Server-Sent Events**: repeated **`data:`** lines with the same JSON shape as the poll endpoint until status is terminal (**`complete`**, **`failed`**, **`cancelled`**). |
| `POST` | `/api/autopilot/build/{id}/cancel` | **`celery_app.control.revoke`** (best-effort, broker optional in dev), then mark row **`cancelled`**. |
| `GET` | `/api/autopilot/build/{id}/result` | Returns the **opaque** JSON blob from **`AutopilotBuild.result`** (orchestrator / worker output), **`404`** if nothing persisted yet. |

### How is multi-tenant ownership enforced on every call?

Handlers load the build through **`AutopilotBuildService.get_owned`**: an SQLAlchemy **`select`** joins **`autopilot_builds` → `projects`**, filters **`projects.user_id`** to the resolved **`X-User-ID`** (or configured default) and **`projects.deleted_at IS NULL`**. Missing rows surface as **`404`**, not **`403`**, to avoid leaking build existence across users.

### Where exactly are `document_ids` and `base_config` stored for the worker?

They are merged into the same **`requirements`** JSON column written at **`POST /build`**: **`document_ids`** as a string list and optional **`base_config`** as a dict. **`run_pipeline_build`** already read **`requirements["document_ids"]`** and **`requirements["base_config"]`** for **`initial_autopilot_graph_state`** — P6-9 makes that path live for real enqueue traffic.

### Why stash the Celery AsyncResult id under `_celery_task_id` inside JSON?

Avoids a DB migration while still giving **cancel** a concrete task handle. The worker ignores keys it does not read; the API treats **`_celery_task_id`** as internal metadata.

### Why wrap `celery_app.control.revoke` in a try/except instead of letting it fail?

**Revoke** talks to the **broker** (Redis). In **CI** or **offline dev** the broker may be down; the product decision here is to **still mark the build cancelled in PostgreSQL** and log a warning so operators see intent even if the worker cannot be signalled immediately.

### How does the Celery task cooperate with a cancelled build?

**Before** flipping **`pending → running`**, **`run_pipeline_build`** returns immediately if the row is already **`cancelled`**. **Before** writing the success **`complete`** state, it re-reads the row and aborts if **`cancelled`** (so a late cancel during graph execution is not overwritten by **`complete`**). The exception path also avoids clobbering **`cancelled`** with **`failed`**.

### What is the difference between `GET …/build/{id}` `result` and `GET …/build/{id}/result`?

**`BuildStatusResponse.result`** is **`BuildResultSchema | null`** — only populated when the stored JSON fully validates as the **normalised** “final pipeline + metrics + decisions” contract. Until then it stays **`null`**. The **`/result`** route returns **`BuildArtifactResultResponse`** (arbitrary dict) so Phase 7 UIs and operators can inspect **`stage_outputs`**, flags, and other orchestrator dumps without pretending the shape is final-user-facing.

### Why does the SSE handler open a fresh async session each second instead of reusing `Depends(get_db_session)`?

A stream can run for **minutes**. Holding one request-scoped session across **`asyncio.sleep`** ties up a pool slot and keeps an implicit transaction boundary ambiguous. Short **`async with session_factory()`** blocks per tick are easier to reason about for read-only polling.

### What HTTP code does a successful enqueue return, and why not `200`?

**`202 Accepted`** — the server accepted responsibility to **process later**; the heavy LangGraph work is not done inside the request/response cycle.

### What happens if you cancel the same build twice?

The first **`POST /cancel`** succeeds (**`200`**) with **`status: "cancelled"`**. The second returns **`409 Conflict`** because the build is already in a terminal cancelled state (same for **`complete`**).

### Does the Autopilot API implement document upload (`POST /api/autopilot/upload`)?

**Not in P6-9.** The **`StartBuildRequest`** schema still documents MinIO object IDs for when **P7-1** adds upload; clients may pass **placeholder** IDs until storage is wired.

### Interview trap: will SSE events be camelCase like the REST of the API?

Yes — SSE payloads call **`model_dump(mode="json", by_alias=True)`** so keys match the OpenAPI / frontend **`camelCase`** convention.

### What is the next task after P6-9?

**Phase 7 · P7-1 · Document Uploader** (and subsequent P7 UI) — consume these endpoints from the browser with real uploads, live progress, and dashboards.

---

## Phase 7 · P7-1 · Document Uploader

### Which route stores corpus bytes, and what does it return?

**`POST /api/autopilot/upload`** — **multipart/form-data** with **`projectId`** (camelCase form field) and one or more **`files`** parts. On success it returns **`201 Created`** with **`{ "documents": [ … ] }`**. Each element includes **`objectId`** (the MinIO object key), **`originalFilename`**, **`sizeBytes`**, and optional **`contentType`**. The client passes **`objectId`** strings verbatim as **`documentIds`** when calling **`POST /api/autopilot/build`**.

### How is tenant isolation enforced on upload?

The handler parses **`projectId`** as a UUID, loads **`projects`** by primary key, and requires **`project.user_id`** to equal the resolved **`X-User-ID`** (or configured default) and **`deleted_at IS NULL`**. Otherwise it responds **`404 Project not found`** — same “existence hiding” posture as **`AutopilotBuildService.get_owned`**.

### Why multipart instead of JSON with base64 payloads?

**Efficiency and ergonomics**: browsers already expose **`File`** / **`FormData`** with streaming-friendly boundaries; base64 would inflate size (~33%) and force the API to decode large strings inside JSON. Multipart matches nginx **`client_max_body_size`** tuning and standard object-storage tooling.

### Where are objects written in MinIO, and why include `user_id` and `project_id` in the key?

Keys look like **`autopilot/{user_id}/{project_id}/{uuid}_{sanitized_original}`**. The prefix prevents accidental cross-tenant reads when future download or ingestion jobs list by prefix, makes lifecycle policies easier, and keeps human operators able to correlate objects with owning projects without a separate index.

### How are unsafe filenames and unsupported types handled?

The original basename is stripped of directories, reduced to **`[a-zA-Z0-9._-]`**, and truncated. Extensions must belong to an allow-list (**`.pdf`**, **`.txt`**, **`.md`**, **`.markdown`**, **`.csv`**, **`.json`**, **`.html`**, **`.htm`**, **`.docx`**). Violations raise **`400`** with a clear message before any bytes hit object storage.

### What happens if MinIO is down?

The synchronous MinIO client runs inside **`asyncio.to_thread`** so the event loop stays responsive. Connection or **`S3Error`** failures bubble as **`AutopilotStorageUnavailableError`**, mapped to **`503 Service Unavailable`** with a plain-text **`detail`** string for operators.

### Why lazy-import the MinIO client inside helper functions?

Some developer machines (e.g. **CPython 3.13** without wheels for every transitive dependency) still need **`import app.main`** for unit tests. Deferring **`import minio`** until an upload actually executes lets **`pytest`** import the FastAPI app and patch **`upload_blobs_sync`** without requiring a successful **`pip install minio`** in that environment. Production containers still install **`minio`** from **`requirements.txt`**.

### How does the Next.js client reach `/api/autopilot/upload` without CORS pain?

The browser uses **same-origin** `fetch('/api/...')`; **`next.config.js`** rewrites **`/api/*`** to the FastAPI origin, mirroring the existing **`apiClient`** pattern. **`postFormData`** deliberately omits a manual **`Content-Type`** header so the runtime sets **`multipart/form-data; boundary=…`**.

### What state does the Zustand autopilot store persist for uploads?

**`selectedBackendProjectId`**, **`uploadedDocuments`** (each with **`objectId`**, **`originalName`**, **`sizeBytes`**), plus existing wizard fields. The storage key bumped to **`rag-studio-autopilot-v2-p7`** to avoid deserialising obsolete **`documents: string[]`**-only shapes.

### How does the UI obtain selectable `projectId` values?

It **`GET /api/projects/?page=1&page_size=50`** (trailing slash avoids **`307`** redirect issues behind the rewrite proxy, same lesson as the template gallery). If the list is empty, copy links users to **Templates** or **Projects** to create a server-backed row first.

### What automated tests cover upload?

**`tests/test_autopilot_router.py`** patches **`upload_blobs_sync`** to avoid a real MinIO daemon, asserts **`201`** plus JSON shape for happy path, and asserts **`404`** when **`X-User-ID`** does not own the target project.

### Interview trap: are `documentIds` validated to exist in MinIO before enqueue?

**Not in P7-1.** **`StartBuildRequest`** still only checks **non-empty** IDs; verifying object existence (or re-hydrating corpus profiles) is a natural follow-up when ingestion agents read bytes from storage rather than placeholders.

### What is the next task after P7-1?

**P7-2 · Requirements Form** — capture **`BuildRequirementsSchema`** fields in the browser before wiring **`POST /api/autopilot/build`**.

---

## Phase 7 · P7-2 · Requirements Form

### What TypeScript shape represents Autopilot requirements in the browser?

**`BuildRequirements`** in **`apps/web/src/types/autopilot.ts`**: **`targetMetrics`** (optional **faithfulness**, **answerRelevance**, **contextPrecision**, **contextRecall** each **0–1**), optional **`cloudProvider`**, optional **`budgetConstraint`** (USD per **1K** queries), optional **`latencyRequirement`** (ms), **`optimizeFor`** (**quality** | **cost** | **latency** | **balanced**), and **`maxIterations`**. The Zustand store persists **`requirements`** beside uploads so a later **Start build** button can read one coherent payload.

### How does the UI stay aligned with backend validation?

The form uses **`BuildRequirementsSchema`** (**Zod** in **`apps/web/src/lib/validators.ts`**) on **Validate**, mirroring the intent of the FastAPI **`BuildRequirementsSchema`** in **`apps/api/app/schemas/autopilot.py`**. **`maxIterations`** is capped at **10** on both sides (orchestrator graph also clamps iteration caps). Optional numeric fields parse on blur; non-finite input is dropped so the store does not retain **`NaN`**.

### Why camelCase in TS types when Pydantic uses snake_case?

The API layer serialises with **aliases** so JSON over the wire is **camelCase** (**`targetMetrics`**, **`optimizeFor`**, …). The Python schema uses **snake_case** field names as the internal representation. The frontend types match the **wire** JSON contract consumers see in **`fetch`**.

### How does `optimize_for` influence agents?

Chunking and related optimisers read **`requirements["optimize_for"]`** (after normalisation) when scoring candidates—for example preferring cheaper chunking configs when **`cost`** is selected. The requirements form writes **`optimizeFor`** into Zustand so the eventual **`POST /api/autopilot/build`** body matches what **`graph.py`** and optimizer modules already expect in **`state["requirements"]`**.

### Why offer an optional cloud provider on the requirements form?

**`cloudProvider`** hints packaging and defaults for the deployment stage and keeps Autopilot aligned with Designer’s **cloud catalog** (**`data/cloud-providers.json`** via **`listCloudProviders()`**). It is optional: **`No preference`** omits the field so the orchestrator is not over-constrained.

### What interview trap exists around “budget” units?

**`budgetConstraint`** is documented as **maximum cost per 1K queries in USD**—not per token and not monthly cap. Interviewees should state assumptions clearly if product later adds different billing dimensions.

### How do you reset corrupted local state?

**Reset defaults** calls **`createDefaultRequirements()`** (exported from **`autopilot-store.ts`**) and **`setRequirements`**, re-syncing controlled text fields for budget and latency. Users can also clear **localStorage** key **`rag-studio-autopilot-v2-p7`** if a future schema bump requires it.

### What is the next task after P7-2?

**P7-3 · Build Progress Monitor** — subscribe to **`GET /api/autopilot/build/{id}/stream`** (or poll **`GET …/build/{id}`**) and render live progress from **`BuildStatusResponse`**.

---

## Phase 7 · P7-3 · Build Progress Monitor

### How does the frontend subscribe to Autopilot build progress?

**`useAutopilotBuildSubscription(activeBuildId)`** opens a browser **`EventSource`** against same-origin **`GET /api/autopilot/build/{id}/stream`**. The FastAPI handler emits **`data: { …BuildStatusResponse… }\n\n`** about once per second until the row reaches a terminal **`status`** (**`complete`**, **`failed`**, **`cancelled`**). The hook parses JSON, merges into **`useAutopilotStore.builds`** via **`upsertBuild`**, and preserves **`input`** (documents, requirements, optional **`baseConfig`**) from the optimistic row created when **`POST /api/autopilot/build`** returns.

### Why add a polling fallback if SSE already exists?

**`EventSource`** has no custom headers (fine here because **`X-User-ID`** defaults server-side), but **corporate proxies**, **HTTP/2 intermediaries**, and **dev tooling** sometimes drop or buffer SSE. On **`onerror`** (or JSON parse failure from a truncated chunk), the hook **closes** the stream and starts **`setInterval`** polling **`GET /api/autopilot/build/{id}`** every **2s** until terminal, so the UI still converges without a manual refresh.

### How do you avoid resetting user-owned fields when each SSE tick arrives?

**`parseBuildStatusPayload`** returns **`Omit<AutopilotBuild,'input'>`**. **`mergeBuildFromServer(prev, server)`** spreads server fields but keeps **`prev.input`** when present. **`BuildProgressMonitor`** seeds **`input`** immediately after **`202 Accepted`** from **`POST /api/autopilot/build`** so the first SSE snapshot cannot wipe document or requirement context.

### What does `BuildStatusResponse` contain for the progress bar and stage list?

**`progress`** (0–100), **`currentStage`** (string, e.g. **`queued`**, **`orchestrate`**, agent keys), **`iteration`**, per-stage **`stages`** map (**`pending` \| `running` \| `complete` \| `failed`** with optional **`message`**), **`messages`** (timestamped agent lines), optional typed **`result`**, optional **`error`**, and timestamps. The UI orders stages using **`AUTOPILOT_STAGE_ORDER`** aligned with **`app.core.agents.state.AUTOPILOT_STAGE_ORDER`**.

### Why can `EventSource` use a relative `/api/...` URL in Next.js?

The browser targets the **Next.js origin**; **`next.config.js`** **`rewrites`** proxy **`/api/*`** to FastAPI, avoiding **CORS** and mismatched hosts (**`127.0.0.1`** vs **`localhost`**). Server components are not involved—the hook is **`'use client'`**.

### How does cancel interact with the worker?

**`POST /api/autopilot/build/{id}/cancel`** revokes the stored **Celery** task id (when present) and marks the DB row **`cancelled`**. The UI re-fetches status once so Zustand reflects the terminal state even if the client had fallen back to polling.

### Interview trap: does every SSE payload include the full `messages` array?

**Yes** in the current API—the handler dumps the whole **`build_status_response(row)`** each tick. The Zustand persist layer already **truncates** long **`messages`** arrays (**`MAX_PERSISTED_MESSAGES`**) on **`upsertBuild`** to keep **localStorage** bounded; **P7-4** surfaces the retained slice in **`AgentActivityFeed`** with filters and export.

### How is `Start build` validated before POST?

The monitor requires a **selected project**, at least one **uploaded object id**, and **`BuildRequirementsSchema.safeParse`** (same intent as the Requirements panel’s **Validate**). Payload keys are **camelCase** (**`projectId`**, **`documentIds`**, **`requirements`**, optional **`baseConfig`**) matching **`RAGBaseModel`** aliases.

### What is the next task after P7-3?

**P7-4 · Agent Activity Feed** (implemented): full log stream, filters, and export on top of the **`messages`** array. Then **P7-5 · Metrics Dashboard**.

---

## Phase 7 · P7-4 · Agent Activity Feed

### Where does the agent activity feed get its data?

**`useAutopilotStore`**: the feed reads **`builds[activeBuildId].messages`**, the same **`BuildMessage[]`** merged from **`parseBuildStatusPayload`** during SSE or polling (**`useAutopilotBuildSubscription`** in **`BuildProgressMonitor`**). No second network subscription is required as long as the Autopilot page mounts the progress monitor (or any hook that keeps the store updated).

### Why implement filtering on the client instead of a new API?

**`BuildStatusResponse`** already ships the **`messages`** array on each tick; duplicating query parameters on the server would add latency and contract surface for little gain while message counts remain bounded (**`MAX_PERSISTED_MESSAGES`** in **`autopilot-store.ts`**). Client filters (**text**, **agent**, **type** toggles) operate on the in-memory slice instantly and match export semantics (**filtered** rows only).

### How do JSON and text export differ?

**JSON** export from **`AgentActivityFeed`** writes a structured object: **`buildId`**, **`buildStatus`**, **`exportedAt`**, **`filters`** (active query / agent / included types), **`messageCount`**, and **`messages`** (the visible rows). **Text** export flattens each line to **`[timestamp] Agent · TYPE — text`** with a short header—suitable for pasting into tickets or **`grep`**.

### What happens to filters when the user switches the active build?

An **`useEffect`** keyed on **`activeBuildId`** resets **search**, **agent** (**All agents**), and **all type toggles on**, so a filter from build A does not hide messages on build B.

### Why auto-scroll only when the user is near the bottom?

The feed keeps **`stickBottomRef`** in sync with **`scrollHeight - scrollTop - clientHeight`**. New SSE appends only auto-scroll if the user was already near the end (terminal-style behaviour). If they scrolled up to read history, the viewport does not jump.

### How are message types styled?

Rows use a **left border** and light background tint by **`BuildMessage.type`** (**`info`**, **`success`**, **`warning`**, **`error`**) so operators can scan failures without reading full text.

### Interview trap: does export include messages truncated on the server?

**Exports reflect whatever is in the store** after **`trimBuildForPersistence`**. If the backend ever sends more than the cap, the UI would still only export the truncated tail until persist rules change—interviewees should mention this consistency trade-off.

### What is the next task after P7-4?

**P7-5 · Metrics Dashboard** (implemented): live charts and SLO cards fed by **`dashboardMetrics`** on **`BuildStatusResponse`**, plus client-sampled progress trends. Next: **P7-6 · Decision Explainer & Results**.

---

## Phase 7 · P7-5 · Metrics Dashboard

### Why add `dashboard_metrics` to the build status API instead of only charting in the browser?

The worker persists the LangGraph orchestrator blob on **`AutopilotBuild.result`**, but that JSON is **not** a valid **`BuildResultSchema`** (it wraps **`stage_outputs`**, flags, and metadata), so **`result`** stays **`null`** on **`GET /api/autopilot/build/{id}`**. **`extract_dashboard_metrics`** reads **`result.stage_outputs`** and returns a **small typed projection** (**`AutopilotDashboardMetricsSchema`**) that is safe to chart: evaluation proxies, embedding bench rows, retrieval summary. The UI avoids re-implementing orchestrator parsing rules and stays stable when the opaque artifact evolves.

### How does the metrics dashboard get live updates?

The same **SSE / polling** path as P7-3: **`useAutopilotBuildSubscription`** merges each **`BuildStatusResponse`** into Zustand. **`parseBuildStatusPayload`** maps **`dashboardMetrics`** (camelCase) onto **`AutopilotBuild.dashboardMetrics`**. The **`MetricsDashboard`** component reads **`builds[activeBuildId]`**; no extra HTTP calls.

### What is client-sampled “progress trend”?

Between API ticks the dashboard appends a **local** point when **`progress`**, **`iteration`**, or **`updatedAt`** changes, capped (~240 samples). That yields a **line chart** even before evaluation metrics exist, and mirrors operator perception of motion during long orchestrations.

### How are quality bars interpreted?

**`dashboardMetrics.quality`** carries **faithfulness**, **answer_relevance**, **context_precision**, **context_recall**, optional **avg_latency_ms**, and **meets_targets** from the evaluation agent payload. The UI builds horizontal bars **observed vs target** using **`build.input.requirements.targetMetrics`** (merged earlier from the seed row). Missing targets render as **`null`** bars so Recharts skips the grey “target” segment for that metric.

### How does the latency section relate to requirements?

Embedding candidates expose **`latencyMs`** per tried model. The chart draws an optional **`ReferenceLine`** at **`latencyRequirement`** when the user set an SLO. A separate card compares **eval proxy latency** (**`avg_latency_ms`**) to the same SLO.

### What about “cost” if there is no dollar metric in `dashboard_metrics`?

**`budget_constraint`** from requirements is shown as a **budget cap card** (USD / 1K queries). There is no per-query dollar estimate in the v1 snapshot; interviewees should distinguish **SLO capture** from **full cost analytics** (later phases such as P11-3).

### Interview trap: will `dashboard_metrics` always be non-null after `complete`?

Only when **`stage_outputs`** contains extractable sections. If the orchestrator shape changes and validation fails, **`_optional_dashboard_metrics`** returns **`null`**—the UI must tolerate **empty chart placeholders**.

### What is the next task after P7-5?

**P7-6 · Decision Explainer & Results** — explainability and final result visualization screens (implemented: see Phase 7 · P7-6 section below).

---

## Phase 7 · P7-6 · Decision Explainer & Results

### Why was `GET /api/autopilot/build/{id}` returning `result: null` even after a successful build?

The Celery worker persisted an **opaque orchestrator blob** (`autopilot_graph`, `stage_outputs`, …) that was **not** a `BuildResultSchema`. FastAPI’s `_optional_typed_result` only accepts payloads that validate as **`BuildResultSchema`**, so **`result` stayed `null`**. **P7-6** adds **`compose_build_result_payload`** (`app/services/autopilot_build_result.py`), invoked at the end of **`run_pipeline_build`**, which **merges** a typed **`config`**, **`metrics`**, **`decisions`**, optional **`deployment`**, and **`total_iterations`** onto the same JSON row. Existing diagnostic keys remain for debugging.

### How is the synthesised `PipelineConfiguration` built?

The compositor starts from a **minimal default shell** (valid catalog IDs) or from **`requirements.base_config`** when it validates as **`PipelineConfigurationSchema`**. It then **patches** `stages.chunking`, `stages.embedding`, `stages.retrieval`, and optional **`reranking`** from **`stage_outputs`** “selected” payloads, and stamps **`metadata.source = autopilot`** plus **`metadata.build_id`**.

### What appears in `decisions` for the Decision Explainer UI?

**Chunking** — strategy, chunk size, analyst/optimiser **rationale**, and **alternatives_tested**. **Embedding** — winning model id, rationale, and a **benchmark_results** table (score clamped 0–1, latency ms, cost per 1M tokens when present). **Retrieval** — strategy, `top_k`, reranking flag, rationale, and a small **performance** map (e.g. MRR, composite score). **Generation** — carried from the merged pipeline (the LangGraph does not mutate generation in v1).

### Where do final evaluation numbers on the Results Summary come from?

**`metrics`** maps **`stage_outputs.evaluation.metrics`** (faithfulness, answer relevance, context precision/recall, optional avg latency). If evaluation is incomplete, the compositor falls back to **neutral 0.5 proxies** so the schema still validates — operators should treat those as **placeholders**, not measured RAGAS.

### What does the “Open in Designer” button do?

It calls **`useDesignerStore.loadPipeline(result.config)`**, patches **`metadata.source`** / **`buildId`**, and navigates to **`/designer/review?source=autopilot`** so the user can inspect the same pipeline JSON in the Designer review shell.

### Why show an amber banner when status is `complete` but `result` is still missing?

**Older builds** (pre-P7-6 worker) never received a normalised **`BuildResult`**. The **`ResultsSummary`** component explains that and asks the user to **run a new build**; charts from **P7-5** may still work via **`dashboard_metrics`** + **`stage_outputs`**.

### How does JSON download differ from `GET …/build/{id}/result`?

**Download** serialises the **already-fetched** typed **`build.result`** object in the browser (camelCase). **`GET /api/autopilot/build/{id}/result`** returns the **raw persisted orchestrator artifact** (opaque). Interviewees should distinguish **client convenience export** from **server artifact introspection**.

### Interview trap: will `deployment` always reflect a live cloud URL?

**No.** When **`stage_outputs.deployment.status == "complete"`**, the compositor attaches a **stub** **`DeploymentInfoSchema`** (local stub host, `docker_image_tag`, `status: deployed`). Real cloud endpoints belong in later phases (e.g. P12).

### What tests protect the compositor?

**`tests/test_services/test_autopilot_build_result.py`** exercises a **happy-path** `BuildResultSchema` round-trip and a **chunking-failed** scenario (chunking decision omitted, defaults still produce a valid config).

### What is the next task after P7-6?

**P7-7 · Autopilot Entry & History Pages** (implemented; see Phase 7 · P7-7 section below). After Phase 7, **Phase 8** covers Designer ↔ Autopilot handoff APIs and flows.

---

## Phase 7 · P7-7 · Autopilot Entry & History Pages

### Why split Autopilot across `/autopilot`, `/autopilot/new`, `/autopilot/history`, and `/autopilot/projects`?

A **single long page** mixed marketing entry, the upload wizard, and observability widgets. **P7-7** introduces an **`AutopilotShell`** layout (sub-nav) so users land on an **overview** with clear CTAs, run the **wizard** on **`/autopilot/new`**, inspect **server-backed history** on **`/autopilot/history`**, and manage **backend project selection** on **`/autopilot/projects`** without scrolling past unrelated panels.

### What does `GET /api/autopilot/builds` provide that the Zustand store does not?

The store persists **this browser’s** merged poll/SSE snapshots. **`GET /api/autopilot/builds`** returns **authoritative rows** from **`autopilot_builds`** joined to **`projects`** (name + ownership), with **pagination** and optional **`project_id`** filter. History UI uses it so a fresh session or another device still sees past runs.

### How is user isolation enforced on the list endpoint?

`AutopilotBuildService.list_for_user` joins **`AutopilotBuild` → `Project`** and filters **`Project.user_id == request user`** and **`deleted_at IS NULL`**. Optional **`project_id`** first calls **`_require_owned_project`** so users cannot probe other tenants’ UUIDs.

### What is the deep-link pattern from History back into the wizard?

Each row links to **`/autopilot/new?build={id}&project={projectId}`**. **`AutopilotNewPageBody`** sets **`activeBuildId`** and **`selectedBackendProjectId`**, then **GETs `/api/autopilot/build/{id}`** once if the build is missing locally, **`parseBuildStatusPayload` + `mergeBuildFromServer`**, so **`useAutopilotBuildSubscription`** can attach SSE/poll immediately.

### Why keep `/projects` and add `/autopilot/projects`?

**`/projects`** is the **local-first** project store UI from earlier phases. **`/autopilot/projects`** lists **`GET /api/projects`** rows and toggles **`selectedBackendProjectId`** in **`useAutopilotStore`** — the same id **multipart upload** and **start build** require — so Autopilot operators have a focused path without redesigning the legacy page.

### Interview trap: does `GET /api/autopilot/builds` return full `messages` / `stages`?

**No.** List items are **`AutopilotBuildListItemSchema`** (status, progress, stage label, timestamps, error). Full stage maps and logs remain on **`GET /api/autopilot/build/{id}`** to keep history tables fast.

### What tests cover the new API?

**`tests/test_autopilot_router.py`** adds **`test_autopilot_list_builds`** (filtered by **`project_id`**, asserts the created **`buildId`** row) and **`test_autopilot_list_builds_filter_unknown_project`** (404). **`_create_project`** uses **`POST /api/projects`** without a trailing slash to match FastAPI’s registered path.

### What is the next task after P7-7?

**Phase 8 · P8-1 · Designer → Autopilot Handoff** — wire “Optimize this” style flows so Designer configs seed Autopilot requirements without manual re-entry. **(Done — see Phase 8 · P8-1 section below.)**

---

## Phase 8 · P8-1 · Designer → Autopilot Handoff

### What user problem does P8-1 solve?

Users who finish a **Designer** draft should not **re-type** chunk sizes, embedding IDs, retrieval strategy, and cloud preference when switching to **Autopilot**. P8-1 carries the draft into Autopilot as an optional **`baseConfig`** on **`POST /api/autopilot/build`** and surfaces that intent in the UI.

### Where does the user start the handoff?

On **`/designer/review`**, the **`DesignerToAutopilotHandoff`** card explains the flow and navigates to **`/autopilot/new?from=designer`** after confirmation.

### What happens in client state when the user confirms?

**`useAutopilotStore.startFromDesigner(draft)`** stores **`baseConfig: PipelineConfiguration`**, resets **`uploadedDocuments`** and **`selectedBackendProjectId`**, reapplies default **`requirements`**, and sets **`requirements.cloudProvider`** from **`draft.cloudProvider`** so the requirements form matches the Designer cloud choice.

### How does the build request include the Designer baseline?

**`BuildProgressMonitor`** already merges **`baseConfig`** into the JSON body as **`baseConfig`** (camelCase). FastAPI’s **`RAGBaseModel`** aliases **`base_config`** ↔ **`baseConfig`**. **`StartBuildRequest.base_config`** validates as **`PipelineConfigurationSchema`** and is persisted under **`requirements["base_config"]`** on the build row for workers.

### Does Autopilot run without new uploads?

**No.** **`document_ids`** remains **`min_length=1`**; the user must **upload corpus files** and pick a **backend project** before **Start build**. The Designer draft describes pipeline hyperparameters, not document binaries.

### What does the banner on `/autopilot/new` communicate?

**`AutopilotDesignerBaselineBanner`** appears when **`baseConfig`** is set: pipeline name, reminder that **`baseConfig`** is attached on start, optional **Clear baseline** calling **`clearHandoff()`** (drops **`baseConfig`** only).

### Interview trap: does handoff replace Autopilot optimization?

**No.** **`base_config`** is a **starting point** for the orchestrator and result compositor (see **`autopilot_build_result`**); agents may still change chunking, embeddings, and retrieval based on corpus analysis and targets.

### What is the next task after P8-1?

**P8-2 · Autopilot → Designer Visualization** — (done; see Phase 8 · P8-2 below). **P8-3** and **P8-4** are complete (see Phase 8 · P8-3 and P8-4). Next in the roadmap: **P9-1 · MLflow Integration**.

---

## Phase 8 · P8-2 · Autopilot → Designer Visualization

### What user problem does P8-2 solve?

After an Autopilot build completes, users need to **see the optimised pipeline in the same Designer visualisation** (Mermaid graph, cost strip, export) they would get from a manual design — without re-entering stage fields. P8-2 turns **`BuildResult.config`** into the Designer **`draft`**, keeps **evaluation metrics** visible on review, and links back to the Autopilot run.

### How does the user open Autopilot output in Designer?

On the completed build screen, **`DecisionExplainer`** exposes **Open in Designer**. That calls **`applyAutopilotBuildResult(buildId, result)`** on **`useDesignerStore`**, then navigates to **`/designer/review?source=autopilot&build=<id>`**.

### What does `applyAutopilotBuildResult` do?

It **loads** **`result.config`** into **`draft`** (via **`deepStripNulls`**), sets **`metadata.source: 'autopilot'`** and **`metadata.buildId`**, unlocks the full stage diagram (**`diagramMaxVisitedStageIndex`**), and stores **`autopilotImportSnapshot`** with **`metrics`** and **`totalIterations`** for the review banner.

### What appears on `/designer/review` after import?

**`AutopilotDesignerImportBanner`** renders when the snapshot’s **`buildId`** matches **`draft.metadata.buildId`**: RAGAS-style metric tiles (faithfulness, relevance, precision, recall), latency and iteration count, plus **Open Autopilot build** → **`/autopilot/new?build=…`**. The existing **DesignerShell** footer still renders **PipelineVisualizer**, **CostEstimator**, and **CodeExporter** bound to the same **`draft`**.

### Why is `syncAutopilotSnapshotFromStores` needed?

If **`metadata`** already marks **`source: autopilot`** but the metric snapshot was cleared (e.g. **`loadPipeline`** from a template wipes **`autopilotImportSnapshot`**), the sync tries **`useAutopilotStore.builds[buildId].result`** to **rehydrate** metrics without another navigation.

### What clears the Autopilot import snapshot?

**`loadPipeline`**, **`resetDraft`**, and any path that sets **`autopilotImportSnapshot: null`** — template apply and full reset should not keep stale Autopilot metrics.

### Does P8-2 add a new backend endpoint?

**No.** It is a **frontend integration** on top of **`BuildResult`** already returned by **`GET /api/autopilot/build/{id}`** and the orchestrator’s composed result.

### Interview trap: does “Open in Designer” mutate Autopilot state?

**No.** It only reads **`build.result`** and updates **`useDesignerStore`**; existing **`useAutopilotStore`** build records are not modified by this action.

### What is the next task after P8-2?

**P8-3 · Evaluation API Endpoints** is implemented (see Phase 8 · P8-3). **P8-4 · Deployment API Endpoints** is also complete (see Phase 8 · P8-4). Next: **P9-1 · MLflow Integration**.

---

## Phase 8 · P8-3 · Evaluation API Endpoints

### What user problem does P8-3 solve?

Designer and Autopilot need a **shared backend** to **run RAGAS-style evaluations** on a saved **pipeline configuration**, **review past runs**, and **compare two configs** (A/B) without re-implementing RAGAS in the client or ad-hoc scripts.

### What HTTP surface does the API expose?

| Method | Path | Role |
|--------|------|------|
| `POST` | `/api/evaluation/run` | Create an `evaluation_runs` row, execute evaluation, return `EvaluationRunResponse` (status `complete` or `failed`). |
| `GET` | `/api/evaluation/run/{run_id}` | Return one run (metrics, failure analysis, error) for an **owned** config. |
| `GET` | `/api/evaluation/runs?config_id=<uuid>` | List runs for a config, **newest first** (with `total` count). |
| `POST` | `/api/evaluation/compare` | Compare two configs by **reusing two completed run IDs** or by **running both** on a **shared synthetic** test set of `test_set_size`. |

### How is the acting user determined?

Same pre-auth pattern as other APIs: **`X-User-ID`** (or `default_user_id` from settings). `EvaluationService` joins `evaluation_runs` → `pipeline_configs` → `projects` and requires `projects.user_id` to match.

### How does a run build the test set?

- If the client sends **`test_set`**: each row is a **`TestSetEntry`** (`question`, `ground_truth`, optional `context` list). Empty **`test_set`** is rejected (`400`).
- If **`test_set` is omitted**: the service generates **`test_set_size`** rows (**10–500**) from an internal synthetic corpus (deterministic segments), matching the schema comment that absent rows imply generated data.

### How are model answers and contexts produced?

For each row, the service calls **`run_guarded_rag_query`**: context documents are **`entry.context`** if provided, otherwise **`ground_truth`** as a single **`Document`** (simulates grounded retrieval). The **generation** answer feeds RAGAS alongside **`contexts`** derived from **`documents_used`** (fallback to **`ground_truth`**).

### Which metrics run in RAGAS?

The **`EvaluationEngine`** resolves metric names via **`resolve_ragas_metric_names`**. Request **`metrics`** overrides pipeline defaults; otherwise **`metric_names_from_pipeline`** reads **`stages.evaluation`** when evaluation is enabled; **`latency`** in the pipeline list is excluded from RAGAS (wall-clock is tracked separately as **`avg_latency_ms`** on the result).

### What gets persisted on `evaluation_runs`?

**`status`**, serialised **`EvaluationMetrics`**, optional **`FailureAnalysisResult`**, **`test_set_size`**, **`error`**, **`completed_at`**. **`build_id`** remains `NULL` for API-triggered runs (Autopilot can link later).

### How does `POST /api/evaluation/compare` work?

- If **both** `run_id_a` and `run_id_b` are set: load **complete** runs, verify each **run’s `config_id`** matches the given config id, reuse stored metrics.
- If **neither** is set: build **one** shared synthetic test set, run **`run_evaluation` twice** (config A and B), then **`compare_metrics`** (per-metric winner + overall tally).
- If only one run id is set: **`400`** — must supply both or neither.

### What does the compare response include?

**`CompareConfigsResponse`**: **`metrics_a` / `metrics_b`**, per-metric **`deltas`** (with **`winner`** `a` / `b` / `tie`, including **`avg_latency_ms`** when present), **`overall_winner`**, and a short human **`summary`**.

### Interview trap: does evaluation require a deployed vector index?

**Not for this API path.** Context is either user-supplied, or **ground-truth text** used as the chunk passed into the guarded RAG path — so the endpoint is **self-contained** for integration and lab use. A future enhancement can wire **real retrieval** from Qdrant when indexed corpora exist.

### What is the next task after P8-3?

**P8-4 · Deployment API Endpoints** is implemented (see Phase 8 · P8-4). Next in the roadmap: **P9-1 · MLflow Integration**.

---

## Phase 8 · P8-4 · Deployment API Endpoints

### What user problem does P8-4 solve?

Saved **pipeline configurations** need a **first-class HTTP API** to **start** a deployment (Docker / K8s / cloud targets), **poll status**, **list** deployment history under a **project**, and **tear down** a deployment record — without bypassing the DB or calling Celery manually from scripts.

### What HTTP surface does the API expose?

| Method | Path | Role |
|--------|------|------|
| `POST` | `/api/deployment/deploy` | Create a `deployments` row (`status=deploying`), commit, enqueue Celery `run_deployment`, return `DeployResponse` (`201`). |
| `GET` | `/api/deployment/{deployment_id}/status` | Return endpoint, health URL, image tag, timestamps, `error` from `deployment_info` when `failed`. |
| `GET` | `/api/deployment/deployments?project_id=<uuid>` | Paginated list (`page`, `page_size` ≤ 100) for all configs in that **owned** project. |
| `DELETE` | `/api/deployment/{deployment_id}` | Logical teardown: `status=teardown`, clear `endpoint` / `health_check_url`, set `deployment_info.teardown_at`. |

### How is the acting user scoped?

Same as evaluation/projects: **`X-User-ID`** (or **`default_user_id`**). Every query joins **`deployments` → `pipeline_configs` → `projects`** and requires **`projects.user_id`** to match and **`deleted_at` IS NULL**.

### Why commit before enqueueing Celery?

The **`run_deployment`** worker opens a **new synchronous session** and loads the row by ID. If the FastAPI request transaction had not yet committed, the worker could **race** and return “deployment not found”. **`DeploymentService.trigger_deploy`** **`commit()`**s immediately after flush, then calls **`tasks.run_deployment.delay`**.

### What does the existing Celery task do?

**`jobs.run_deployment`** (see **`app/worker/tasks.py`**) is a **stub**: it sets **`deployed`**, a **stub HTTPS endpoint** under **`stub.rag-studio.local`**, **`health_check_url`**, **`docker_image_tag`**, **`deployed_at`**, and merges **`deployment_info.stub: true`**. Real Terraform / `kubectl` / cloud apply remains **operator-gated** (see deployment agent and Phase 12).

### How do optional deploy body fields persist?

**`region`** and **`image_tag`** map to **`deployment_info.region`** (JSON) and **`docker_image_tag`** on the row, respectively.

### What is idempotent about teardown?

Calling **`DELETE`** on an already **teardown** deployment still requires **ownership**; the handler updates the row again (safe) and returns **`DeploymentStatusResponse`** with **`status: teardown`**.

### Interview trap: does list filter by configuration?

**No** — it filters by **`project_id`** only, so you see **all deployments** for every **saved config** in that project (newest first). Use **`config_id`** from each **`DeploymentListItem`** to correlate.

### What is the next task after P8-4?

**P9-1 · MLflow Integration** — experiment tracking for Autopilot runs (see `docs/internal/project_status.md`).

---

