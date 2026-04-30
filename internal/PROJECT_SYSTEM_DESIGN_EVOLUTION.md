# RAG Studio — System Design Evolution

> This document captures the evolving architecture of RAG Studio across each implementation phase.
> Designs start simple and advance progressively — each phase adds a new architectural layer.

---

## Phase P0-1 · Monorepo Skeleton

**What changed:** Established the repository structure. A single git repo containing two apps (`web`, `api`) and shared `data/` catalogs. No services are running yet.

### Design Level 1 — Repository Layout

```mermaid
graph TD
    subgraph REPO["rag-studio/ (monorepo)"]
        PKG["package.json (workspaces)"]
        ENV[".env.example"]
        GIT[".gitignore"]

        subgraph APPS["apps/"]
            WEB["apps/web/  (placeholder)"]
            API["apps/api/  (placeholder)"]
        end

        subgraph DATA["data/"]
            MODELS["models/ (placeholder)"]
        end

        subgraph INFRA["docker/ · k8s/ · scripts/"]
            DOCKER["docker/  (placeholder)"]
            K8S["k8s/    (placeholder)"]
        end

        subgraph DOCS["docs/"]
            INTERNAL["docs/internal/ (specs)"]
        end
    end

    PKG --> WEB
    PKG --> API
```

**Key decisions:**
- npm workspaces for dependency hoisting and cross-package scripts
- `data/` as neutral territory — neither frontend nor backend owns it
- `.env.example` as the single source of truth for required env vars

---

## Phase P0-2 · Docker Compose Development Environment

**What changed:** All infrastructure services are now defined and can run locally with a single `docker compose up -d`. The architecture grows from a file layout to a live service mesh.

### Design Level 2 — Local Service Topology

```mermaid
graph TB
    subgraph CLIENT["Client (Browser)"]
        BROWSER["Browser :80"]
    end

    subgraph NGINX_LAYER["Reverse Proxy Layer"]
        NGINX["nginx :80\n/api/* → api:8000\n/* → web:3000"]
    end

    subgraph APP_LAYER["Application Layer"]
        WEB["web (Next.js)\n:3000"]
        API["api (FastAPI)\n:8000"]
        WORKER["worker (Celery)\n— no port —"]
    end

    subgraph DATA_LAYER["Data & Cache Layer"]
        DB["db (PostgreSQL 16)\n:5432"]
        REDIS["redis (Redis 7)\n:6379"]
        QDRANT["vector-db (Qdrant)\n:6333"]
    end

    subgraph INFRA_LAYER["Infrastructure Layer"]
        MLFLOW["mlflow\n:5000"]
        MINIO["minio (S3)\n:9000 / :9001"]
    end

    BROWSER --> NGINX
    NGINX --> WEB
    NGINX --> API
    API --> DB
    API --> REDIS
    API --> QDRANT
    API --> MLFLOW
    API --> MINIO
    WORKER --> DB
    WORKER --> REDIS
    WORKER --> QDRANT
```

**Key decisions:**
- Nginx as single entry point — clients never speak directly to `web` or `api`
- All services health-checked so `depends_on: condition: service_healthy` works
- Dev override (bind mounts) vs prod override (pre-built images, resource limits) via compose file layering

---

## Phase P0-3 · CI/CD Pipelines

**What changed:** Code changes now flow through automated quality gates before reaching any environment. The "manual push to server" anti-pattern is replaced by a structured pipeline.

### Design Level 3 — CI/CD Flow

```mermaid
flowchart LR
    DEV["Developer\npushes code"]

    subgraph PR["Pull Request → develop"]
        CI["ci.yml\n(GitHub Actions)"]
        subgraph JOBS["Parallel Jobs"]
            FE_LINT["frontend-lint\nESLint + tsc"]
            FE_TEST["frontend-tests\nJest + coverage"]
            BE_LINT["backend-lint\nRuff + mypy"]
            BE_TEST["backend-tests\npytest (unit)"]
        end
        GATE["ci-success\n(branch protection gate)"]
    end

    subgraph MERGE["Merge to main"]
        CD["cd.yml\n(GitHub Actions)"]
        subgraph BUILD["Build & Push"]
            IMG_WEB["web image → GHCR"]
            IMG_API["api image → GHCR"]
        end
        DEPLOY["deploy job\n(environment: production\nrequires approval)"]
    end

    subgraph TESTS["Manual / PR dispatch"]
        TESTS_WF["tests.yml"]
        INT["integration tests\n(real Postgres/Redis/Qdrant)"]
        E2E["e2e tests\n(Playwright)"]
    end

    DEV --> PR
    CI --> FE_LINT & FE_TEST & BE_LINT & BE_TEST
    FE_LINT & FE_TEST & BE_LINT & BE_TEST --> GATE
    GATE -->|all pass| MERGE
    CD --> IMG_WEB & IMG_API --> DEPLOY
    DEV --> TESTS
    TESTS_WF --> INT & E2E
```

**Key decisions:**
- `ci-success` synthetic gate means branch protection needs only one check regardless of how many jobs are added
- CD triggers only on `main` — `develop` never auto-deploys
- Integration + E2E tests separated into `tests.yml` to keep CI fast

---

## Phase P0-4 · Backend Project Scaffold

**What changed:** The `apps/api/` placeholder is now a real FastAPI application with configuration, dependency injection, a health endpoint, and a production-grade Dockerfile.

### Design Level 4 — Backend Internal Architecture

```mermaid
graph TB
    subgraph CLIENT["Client / Nginx"]
        REQ["HTTP Request"]
    end

    subgraph FASTAPI["FastAPI Application (apps/api/)"]
        MAIN["main.py\ncreate_app() factory\nlifespan handler\nCORS + logging middleware"]

        subgraph LAYERS["Request Lifecycle"]
            CORS_MW["CORSMiddleware"]
            LOG_MW["Request Logging Middleware\n(X-Request-ID, duration_ms)"]
            ROUTER["Router → /health"]
        end

        subgraph DI["Dependency Injection (dependencies.py)"]
            DB_DEP["get_db_session()\nAsyncSession"]
            REDIS_DEP["get_redis()\naioredis.Redis"]
            QDRANT_DEP["get_qdrant()\nAsyncQdrantClient"]
        end

        subgraph CONFIG["Configuration (config.py)"]
            SETTINGS["Settings (pydantic-settings)\nreads from .env"]
            CACHE["@lru_cache\nget_settings()"]
        end
    end

    subgraph INFRA["Infrastructure"]
        PG["PostgreSQL\nasyncpg driver"]
        RDS["Redis\naioredis"]
        QD["Qdrant\nqdrant-client async"]
    end

    REQ --> MAIN
    MAIN --> CORS_MW --> LOG_MW --> ROUTER
    ROUTER --> DB_DEP & REDIS_DEP & QDRANT_DEP
    DB_DEP --> PG
    REDIS_DEP --> RDS
    QDRANT_DEP --> QD
    SETTINGS --> DB_DEP & REDIS_DEP & QDRANT_DEP
```

**Key decisions:**
- `create_app()` factory (not module-level instance) enables isolated test clients
- `@lru_cache` on `get_settings()` means `.env` is parsed exactly once per process
- Type aliases (`DbSession = Annotated[AsyncSession, Depends(...)]`) keep route signatures clean
- 3-stage Dockerfile: builder (C tools) → development (hot-reload) → runtime (non-root, minimal)

---

## Phase P0-5 · Frontend Project Scaffold

**What changed:** The `apps/web/` placeholder is now a real Next.js 14 App Router application with TypeScript, Tailwind CSS, shadcn/ui configuration, Zustand state management setup, a typed API client, and a production-grade Dockerfile. The frontend can now serve a landing page and is ready for feature development.

### Design Level 5 — Frontend Internal Architecture

```mermaid
graph TB
    subgraph BROWSER["Browser"]
        USER["User"]
    end

    subgraph NEXTJS["Next.js 14 App Router (apps/web/src/)"]
        subgraph APP_DIR["app/"]
            LAYOUT["layout.tsx\nRootLayout\nfont vars, metadata"]
            PAGE["page.tsx\nLanding Page\nHero + Mode Comparison"]
            DESIGNER_RT["designer/\n[step]/ routes"]
            AUTOPILOT_RT["autopilot/\nbuild/[id]/ routes"]
        end

        subgraph COMPONENTS["components/"]
            UI_COMP["ui/\nshadcn/ui primitives\n(Button, Card, Slider…)"]
            DESIGNER_COMP["designer/\nCloudProviderSelector\nChunkingConfig…"]
            AUTOPILOT_COMP["autopilot/\nDocumentUploader\nBuildProgress…"]
            SHARED_COMP["shared/\nPipelineVisualizer\nCostEstimator\nCodeExporter…"]
        end

        subgraph LIB["lib/"]
            API_CLIENT["api-client.ts\nTyped fetch wrapper\napiClient.get/post/put"]
            UTILS["utils.ts\ncn() helper\nclsx + tailwind-merge"]
            CONSTANTS["constants.ts\nDESIGNER_STAGES\nSTAGE_ROUTE_MAP\nAPI_BASE_URL"]
        end

        subgraph STORE["store/ (Zustand)"]
            DS["designerStore.ts\nPipelineConfiguration\npersist → localStorage"]
            AS["autopilotStore.ts\nAutopilotBuild\nBuildMessages"]
            PS["projectStore.ts\nProject[]"]
        end

        subgraph TYPES["types/"]
            PIPELINE_T["pipeline.ts"]
            AUTOPILOT_T["autopilot.ts"]
            MODELS_T["models.ts"]
        end
    end

    subgraph EXTERNAL["External"]
        API["FastAPI Backend\n:8000"]
        LS["localStorage\n(Zustand persist)"]
    end

    USER --> LAYOUT
    LAYOUT --> PAGE
    PAGE --> DESIGNER_RT & AUTOPILOT_RT
    DESIGNER_RT --> DESIGNER_COMP
    AUTOPILOT_RT --> AUTOPILOT_COMP
    DESIGNER_COMP & AUTOPILOT_COMP --> SHARED_COMP
    SHARED_COMP --> API_CLIENT --> API
    DESIGNER_COMP --> DS
    AUTOPILOT_COMP --> AS
    DS --> LS
    DESIGNER_COMP & AUTOPILOT_COMP & SHARED_COMP --> UI_COMP
```

### Design Level 5b — Full Stack Integration View (P0-1 through P0-5)

```mermaid
graph TB
    subgraph USER["User"]
        BROWSER["Browser"]
    end

    subgraph INFRA["Infrastructure (docker-compose)"]
        NGINX_BOX["Nginx :80"]
    end

    subgraph FRONTEND["apps/web (Next.js 14)"]
        NEXT_APP["Next.js App Router\nSSR + Client Components"]
        ZUSTAND["Zustand Stores\ndesigner · autopilot · project"]
        API_CL["api-client.ts\nTyped fetch → /api/*"]
        TW["Tailwind CSS\n+ shadcn/ui"]
    end

    subgraph BACKEND["apps/api (FastAPI)"]
        FASTAPI_APP["FastAPI App\nlifespan · CORS · logging"]
        DI_LAYER["Dependency Injection\nDbSession · Redis · Qdrant"]
        SETTINGS_BOX["pydantic-settings\n.env → Settings"]
    end

    subgraph DATA_STORES["Data Layer"]
        PG_BOX["PostgreSQL 16"]
        REDIS_BOX["Redis 7"]
        QDRANT_BOX["Qdrant"]
        MINIO_BOX["MinIO (S3)"]
        MLFLOW_BOX["MLflow"]
    end

    subgraph CI_CD["CI/CD (GitHub Actions)"]
        CI_BOX["ci.yml: lint + tests"]
        CD_BOX["cd.yml: build + push GHCR"]
        TESTS_BOX["tests.yml: integration + e2e"]
    end

    BROWSER --> NGINX_BOX
    NGINX_BOX --> NEXT_APP
    NGINX_BOX --> FASTAPI_APP
    NEXT_APP --> ZUSTAND
    NEXT_APP --> API_CL
    NEXT_APP --> TW
    API_CL --> FASTAPI_APP
    FASTAPI_APP --> DI_LAYER
    DI_LAYER --> PG_BOX & REDIS_BOX & QDRANT_BOX
    FASTAPI_APP --> MINIO_BOX & MLFLOW_BOX
    SETTINGS_BOX --> FASTAPI_APP

    CI_CD -.->|"validates on every PR"| FRONTEND
    CI_CD -.->|"validates on every PR"| BACKEND
```

**Key decisions:**
- `output: 'standalone'` in `next.config.js` reduces the production Docker image from ~400MB to ~120MB
- Zustand `persist` middleware serialises `PipelineConfiguration` to `localStorage` — users never lose in-progress pipeline designs on page refresh
- `api-client.ts` typed wrapper normalises errors into `ApiError` instances — all callers handle one error type
- CSS custom properties (`--primary`, `--background`) in `globals.css` enable runtime theme switching without rebuilding Tailwind
- `components.json` with `cssVariables: true` ensures shadcn/ui components inherit the same CSS tokens

---

---

## Phase P1-1 · JSON Model Catalogs

**What changed:** The `data/` directory is now the shared source of truth for all model metadata, strategies, pricing, and templates. Both frontend and backend read from these files — eliminating duplication and establishing a shared data contract.

### Design Level 6 — Shared Data Layer

```mermaid
graph TB
    subgraph DATA["data/ (Shared Catalog Layer)"]
        subgraph MODELS["data/models/"]
            EMB["embeddings.json\n10 models (OpenAI, Cohere,\nHuggingFace, Nomic)\ndimensions, cost, quality, tier"]
            GEN["generation.json\n9 LLMs (GPT-4o, Claude,\nGemini, Llama, Mistral)\ncontextWindow, costInput, costOutput"]
            RNK["rerankers.json\n4 models (Cohere, BGE,\nMS-MARCO, FlashRank)\ncost, quality, latency"]
        end

        subgraph STRATEGIES["data/"]
            CHK["chunking-strategies.json\n7 strategies\nbestFor, pros, cons, defaultConfig"]
            VEC["vector-stores.json\n9 stores (Qdrant, Pinecone,\nWeaviate, FAISS, pgvector...)\ntype, features, pricing"]
            RET["retrieval-strategies.json\n6 strategies (similarity, MMR,\nhybrid, parent-child, multi-query, ensemble)\nparameters, complexity"]
            CLD["cloud-providers.json\n4 providers (AWS, GCP,\nAzure, Multi-cloud)\nnativeServices, ragStudioDefaults"]
        end

        subgraph CATALOG["data/"]
            TPL["templates.json\n6 templates\nfull PipelineConfiguration per template"]
            PRC["pricing.json\nper-model costs\ncost calculator formulas\nbenchmark reference points"]
        end
    end

    subgraph FRONTEND["apps/web (Next.js)"]
        SEL["EmbeddingSelector\nVectorStoreSelector\nChunkingConfig\nCloudProviderSelector"]
        COST_FE["CostEstimator\n(reads pricing.json)"]
        TPL_FE["Template Gallery\n(reads templates.json)"]
    end

    subgraph BACKEND["apps/api (FastAPI)"]
        SCHEMA["Pydantic Schemas\n(validates model IDs against catalogs)"]
        COST_BE["cost_calculator.py\n(reads pricing.json)"]
        TPL_BE["TemplateService\n(reads templates.json)"]
    end

    EMB & GEN & RNK --> SEL
    PRC --> COST_FE & COST_BE
    TPL --> TPL_FE & TPL_BE
    CHK & VEC & RET & CLD --> SEL
    EMB & GEN --> SCHEMA
```

**Key decisions:**
- JSON files in `data/` serve as a **shared contract** — model IDs used in `PipelineConfiguration` are validated against these files on both frontend (TypeScript) and backend (Pydantic)
- `pricing.json` contains both raw prices AND documented formulas — cost calculator logic is traceable without reading code
- Templates store complete `PipelineConfiguration` objects — enabling atomic `POST /api/templates/{id}/apply` with no server-side merging
- `cloudProvider.ragStudioDefaults` cascades to downstream stages — selecting AWS pre-suggests OpenSearch, S3, and ECS Terraform templates
- Open-source models have `costPer1MTokens: 0.0` — the UI shows "free" but notes that self-hosting infrastructure costs apply

### Design Level 6b — Data Flow: JSON Catalog → UI → API

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Next.js Frontend
    participant JSON as data/ (JSON Catalogs)
    participant API as FastAPI Backend

    Note over FE,JSON: At build time (Next.js)
    FE->>JSON: import embeddings.json
    FE->>JSON: import chunking-strategies.json
    FE->>JSON: import vector-stores.json
    JSON-->>FE: Catalog data bundled into JS

    Note over U,FE: At runtime (Designer Mode)
    U->>FE: Selects "text-embedding-3-large"
    FE->>FE: Reads dimensions=3072, costPer1MTokens=0.13
    FE->>FE: Updates CostEstimator (reads pricing.json)
    FE->>FE: Filters VectorStores by hybridSearch feature

    Note over FE,API: On save/submit
    U->>FE: Saves pipeline configuration
    FE->>API: POST /api/designer/config { embedding.model: "text-embedding-3-large" }
    API->>JSON: Validate model ID exists in embeddings.json
    API->>JSON: Read costPer1MTokens from pricing.json
    API-->>FE: { id, cost_estimate }
```

---

## Phase P1-2 · TypeScript Shared Types

**What changed:** The `apps/web/src/types/` directory now holds the complete TypeScript type system for the frontend. Every shape — from `PipelineConfiguration` to `AutopilotBuild` — is declared once, exported from a barrel, and shared across components, stores, and the API client. The JSON catalogs from P1-1 now have matching TypeScript interfaces, enabling type-safe imports of catalog data.

### Design Level 7 — TypeScript Type Layer

```mermaid
graph TB
    subgraph TYPES["apps/web/src/types/ (TypeScript Type Layer)"]
        INDEX["index.ts\n(barrel export)"]

        subgraph PIPELINE["pipeline.ts"]
            CP["CloudProvider\n'aws'|'gcp'|'azure'|'multi-cloud'"]
            MT["ModelTier\n'fast'|'balanced'|'advanced'"]
            CS["ChunkingStrategy\n7 variants"]
            VSP["VectorStoreProvider\n9 variants"]
            RS["RetrievalStrategy\n6 variants"]
            PC["PipelineConfiguration\n(root config type)"]
            STAGES["PipelineStages\nall 10 stage configs"]
            COST["CostEstimate\n+ CostBreakdown"]
            PERF["PerformanceEstimate"]
        end

        subgraph AUTOPILOT["autopilot.ts"]
            AB["AutopilotBuild\nfull build lifecycle"]
            BR["BuildRequirements\ntarget metrics, budget"]
            SS["StageStatus\npending|running|complete|failed"]
            BM["BuildMessage\ntyped log entries"]
            RESULT["BuildResult\nfinal config + metrics"]
            AD["AgentDecisions\nper-agent explanations"]
            DI["DeploymentInfo\nendpoint + status"]
        end

        subgraph MODELS["models.ts"]
            EM["EmbeddingModel\nmatches embeddings.json"]
            GM["GenerationModel\nmatches generation.json"]
            RM["RerankerModel\nmatches rerankers.json"]
            CSM["ChunkingStrategyMeta\nmatches chunking-strategies.json"]
            VSM["VectorStoreMeta\nmatches vector-stores.json"]
            RSM["RetrievalStrategyMeta\nmatches retrieval-strategies.json"]
            CPM["CloudProviderMeta\nmatches cloud-providers.json"]
            TPL["Template\nmatches templates.json"]
        end

        subgraph CLOUD["cloud.ts"]
            CPC["CloudProviderConfig\nfull provider shape"]
            CNS["CloudNativeService\nnative service lists"]
            CPD["CloudProviderDefaults\nvectorStore/storage/deploy"]
        end

        INDEX --> PIPELINE
        INDEX --> AUTOPILOT
        INDEX --> MODELS
        INDEX --> CLOUD
    end

    subgraph CONSUMERS["Consumers"]
        STORE["Zustand Stores\ndesignerStore · autopilotStore"]
        COMP["React Components\nDesigner · Autopilot · Shared"]
        API_CL["api-client.ts\ntyped request/response"]
        JSON_IMPORT["JSON Catalog Imports\ndata/*.json"]
    end

    INDEX --> STORE
    INDEX --> COMP
    INDEX --> API_CL
    MODELS --> JSON_IMPORT
```

### Design Level 7b — Type Dependency Graph

```mermaid
graph LR
    PIPELINE["pipeline.ts\n(no imports)"]
    CLOUD["cloud.ts\nre-exports CloudProvider"]
    AUTOPILOT["autopilot.ts\nimports pipeline.ts"]
    MODELS["models.ts\nimports pipeline.ts"]
    INDEX["index.ts\nimports all 4 files"]

    PIPELINE --> CLOUD
    PIPELINE --> AUTOPILOT
    PIPELINE --> MODELS
    CLOUD --> INDEX
    AUTOPILOT --> INDEX
    MODELS --> INDEX
    PIPELINE --> INDEX
```

**Key decisions:**
- `pipeline.ts` has zero imports — it is the leaf of the dependency graph; all other type files depend on it, never the reverse
- `CloudProvider` defined once in `pipeline.ts`; `cloud.ts` re-exports it for cloud-module consumers; `index.ts` exposes it only from `pipeline.ts` to prevent duplicate-export errors
- Object shapes use `interface` (better error messages, supports `extends`); union enumerations use `type` (required for string-literal unions)
- Optional stage fields (`reranking?`, `routing?`, `memory?`, `evaluation?`) model business rules directly — stages that must always be present are required, opt-in features are optional
- `stages: Record<string, StageStatus>` in `AutopilotBuild` uses a dictionary (not array) for O(1) lookup by stage ID in UI rendering
- `AgentDecisions` typed structure enables the "Explain in Designer" feature — the Decision Explainer renders named fields rather than raw JSON

---

## Design Progression Summary

| Phase | Layer Added | Key Artefact |
|-------|------------|-------------|
| P0-1 | Repository structure | `package.json` workspaces, `.gitignore`, `.env.example` |
| P0-2 | Live service mesh | `docker/docker-compose.yml` — 9 services wired |
| P0-3 | Quality gates | `.github/workflows/ci.yml`, `cd.yml`, `tests.yml` |
| P0-4 | Backend application | `apps/api/app/main.py`, `config.py`, `dependencies.py`, `Dockerfile` |
| P0-5 | Frontend application | `apps/web/src/` — Next.js 14 App Router, Zustand, API client, Tailwind, Dockerfile |
| P1-1 | Shared data layer | `data/` — 9 JSON catalogs covering models, strategies, pricing, and templates |
| P1-2 | TypeScript type system | `apps/web/src/types/` — 4 type files + barrel export; all catalog shapes typed |

> **Next phases** will add Python Pydantic schemas (`P1-3`) and the database schema and migrations (`P1-4`) — completing the shared data contract before backend core services begin.
