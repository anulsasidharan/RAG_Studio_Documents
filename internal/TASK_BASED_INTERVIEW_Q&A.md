# RAG Studio — Task-Based Interview Q&A

> Interview questions and answers generated after completing each implementation task.  
> Organised by phase. Questions cover both technical depth and system design reasoning.

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

