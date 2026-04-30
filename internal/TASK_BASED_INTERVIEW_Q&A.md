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
