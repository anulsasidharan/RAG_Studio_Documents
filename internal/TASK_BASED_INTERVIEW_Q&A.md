# Task-Based Interview Q&A — RAG Studio

> **Purpose:** Interview preparation keyed to implementation tasks. Answers reflect the codebase as of Phase 4 / P4-1.

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

## General — FastAPI + async SQLAlchemy

### How does the DB session commit?

`get_db_session` yields a session; on success it `commit()`s, on exception `rollback()`s.

### Where is the projects router registered?

`app/main.py` `create_app()` includes `projects_router` with prefix `/api/projects` (trailing-slash routes `GET/POST /` on the router).

---

*Add new sections per phase/task as features land.*
