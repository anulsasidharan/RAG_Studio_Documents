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

---

## Phase P1-3 · Python Pydantic Schemas

**What changed:** The `apps/api/app/schemas/` package now provides Pydantic v2 schemas for every API boundary. These are the Python mirror of the TypeScript types (P1-2) and use the same enum values as the JSON catalogs (P1-1). The shared contract is now complete on both sides.

### Design Level 8 — Python Schema Layer

```mermaid
graph TB
    subgraph SCHEMAS["apps/api/app/schemas/ (Pydantic Schema Layer)"]
        INIT["__init__.py\n(barrel re-export ~60 classes)"]

        subgraph PIPELINE["pipeline.py\n(no imports from other schema files)"]
            ENUMS["StrEnum types\nCloudProvider · ChunkingStrategy\nVectorStoreProvider · RetrievalStrategy\nEmbeddingProvider · GenerationProvider\nMemoryType · OutputFormat\nSimilarityMetric · FilterOperator"]
            BASE["RAGBaseModel\nalias_generator=to_camel\npopulate_by_name=True\nuse_enum_values=True"]
            STAGE_SCHEMAS["Stage Schemas\nDataIngestionConfigSchema\nChunkingConfigSchema\nEmbeddingConfigSchema\nVectorStoreConfigSchema\nRetrievalConfigSchema\nRerankingConfigSchema\nGenerationConfigSchema\nRoutingConfigSchema\nMemoryConfigSchema\nEvaluationConfigSchema"]
            TOP_LEVEL["PipelineConfigurationSchema\nCostEstimateSchema\nPerformanceEstimateSchema\nPipelineStagesSchema\nPipelineMetadataSchema"]
        end

        subgraph DESIGNER["designer.py"]
            D1["SaveConfigRequest\nSaveConfigResponse\nConfigListItem\nConfigListResponse"]
            D2["ExportRequest\nExportResponse\nCostRequest\nCostResponse"]
        end

        subgraph AUTOPILOT["autopilot.py"]
            A1["BuildRequirementsSchema\nTargetMetricsSchema\nStartBuildRequest\nStartBuildResponse"]
            A2["StageStatusSchema\nBuildMessageSchema\nAgentDecisionSchema\n(per-agent sub-schemas)"]
            A3["BuildResultSchema\nFinalMetricsSchema\nBuildStatusResponse\nDeploymentInfoSchema"]
        end

        subgraph EVALUATION["evaluation.py"]
            E1["EvaluationRunRequest\nTestSetEntry\nEvaluationMetrics"]
            E2["FailureCategory\nFailureAnalysisResult\nEvaluationRunResponse"]
            E3["CompareConfigsRequest\nMetricDelta\nCompareConfigsResponse"]
        end

        subgraph DEPLOYMENT["deployment.py"]
            DP1["DeployRequest\nDeployResponse"]
            DP2["DeploymentStatusResponse\nDeploymentListResponse"]
        end

        PIPELINE --> DESIGNER
        PIPELINE --> AUTOPILOT
        PIPELINE --> EVALUATION
        PIPELINE --> DEPLOYMENT
        DESIGNER --> INIT
        AUTOPILOT --> INIT
        EVALUATION --> INIT
        DEPLOYMENT --> INIT
        PIPELINE --> INIT
    end

    subgraph CONSUMERS["Future Consumers (P4–P8)"]
        ROUTER_D["routers/designer.py\n(P4-2, P4-3, P4-4)"]
        ROUTER_A["routers/autopilot.py\n(P6-9)"]
        ROUTER_E["routers/evaluation.py\n(P8-3)"]
        ROUTER_DP["routers/deployment.py\n(P8-4)"]
    end

    INIT --> ROUTER_D
    INIT --> ROUTER_A
    INIT --> ROUTER_E
    INIT --> ROUTER_DP
```

### Design Level 8b — Schema ↔ Catalog ↔ TypeScript Contract

```mermaid
graph LR
    subgraph JSON["data/ (JSON Catalogs — P1-1)"]
        EMB_J["embeddings.json\nmodel IDs"]
        GEN_J["generation.json\nmodel IDs"]
        CHK_J["chunking-strategies.json\nstrategy IDs"]
    end

    subgraph TS["apps/web/src/types/ (TypeScript — P1-2)"]
        TS_P["pipeline.ts\nCloudProvider · ChunkingStrategy\nPipelineConfiguration"]
        TS_A["autopilot.ts\nAutopilotBuild · BuildResult"]
    end

    subgraph PY["apps/api/app/schemas/ (Python — P1-3)"]
        PY_P["pipeline.py\nCloudProvider · ChunkingStrategy\nPipelineConfigurationSchema"]
        PY_A["autopilot.py\nBuildStatusResponse · BuildResultSchema"]
    end

    EMB_J -- "model IDs validated against" --> PY_P
    GEN_J -- "model IDs validated against" --> PY_P
    CHK_J -- "strategy IDs validated against" --> PY_P

    TS_P -- "camelCase ↔ snake_case\nalias_generator=to_camel" --> PY_P
    TS_A -- "camelCase ↔ snake_case" --> PY_A

    PY_P -- "same enum values" --> TS_P
    PY_A -- "same build lifecycle shape" --> TS_A
```

**Key decisions:**
- `RAGBaseModel` shared base with `alias_generator=to_camel` — one change propagates camelCase aliases to all ~60 schemas
- `StrEnum` for all enumerated values — members ARE strings, so they serialise and compare without `.value` calls
- `pipeline.py` has zero imports from other schema files — it is the leaf of the schema dependency graph, preventing circular imports
- `__init__.py` barrel with explicit `__all__` — single import point for all routers; refactoring a schema's file location only requires updating the barrel
- `from_attributes=True` deliberately omitted — will be added to `RAGBaseModel` in P1-4 when ORM models are introduced

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
| P1-3 | Python schema layer | `apps/api/app/schemas/` — 6 files, ~60 Pydantic v2 schemas; camelCase aliases; StrEnum types |
| P1-4 | Database + migrations | `apps/api/app/models/` — 5 ORM models; `alembic/` — initial migration, 5 tables, indexes, FK constraints |
| P2-1 | Document ingestion core | `apps/api/app/core/ingestion/` — loaders, preprocessors, extractors, `IngestionService` |
| P2-2 | Chunking core | `apps/api/app/core/chunking/` — 8 strategies, `ChunkerFactory`, `ChunkQualityScorer`, `ChunkingService` |
| P2-3 | Embedding core | `apps/api/app/core/embedding/` — 5 providers, `EmbeddingCache`, `EmbeddingBenchmarker`, `EmbeddingService` |

---

## Phase P1-4 — Database Schema & Migrations

### Design Level 9 — ORM Model Layer

The database layer sits between the Pydantic schema layer and the service/route layer. It introduces 5 SQLAlchemy 2.0 ORM models with typed `Mapped[T]` annotations, JSONB columns for nested structures, and Alembic migrations for async PostgreSQL.

```mermaid
erDiagram
    projects {
        UUID id PK
        UUID user_id
        varchar name
        text description
        timestamptz created_at
        timestamptz updated_at
    }

    pipeline_configs {
        UUID id PK
        UUID project_id FK
        varchar name
        varchar version
        varchar cloud_provider
        jsonb config
        varchar source
        text build_id
        timestamptz created_at
        timestamptz updated_at
    }

    autopilot_builds {
        UUID id PK
        UUID project_id FK
        varchar status
        int progress
        varchar current_stage
        int iteration
        jsonb requirements
        jsonb stages
        jsonb messages
        jsonb result
        text error
        timestamptz completed_at
        timestamptz created_at
        timestamptz updated_at
    }

    evaluation_runs {
        UUID id PK
        UUID config_id FK
        UUID build_id FK
        varchar status
        jsonb metrics
        jsonb failure_analysis
        int test_set_size
        text error
        timestamptz completed_at
        timestamptz created_at
        timestamptz updated_at
    }

    deployments {
        UUID id PK
        UUID config_id FK
        varchar provider
        varchar environment
        varchar status
        text endpoint
        text health_check_url
        varchar docker_image_tag
        jsonb deployment_info
        timestamptz deployed_at
        timestamptz created_at
        timestamptz updated_at
    }

    projects ||--o{ pipeline_configs : "has"
    projects ||--o{ autopilot_builds : "has"
    pipeline_configs ||--o{ evaluation_runs : "evaluated by"
    pipeline_configs ||--o{ deployments : "deployed via"
    autopilot_builds ||--o{ evaluation_runs : "triggers (nullable)"
```

---

### Design Level 9b — ORM to Schema Data Flow

```mermaid
sequenceDiagram
    participant Client as Next.js Client
    participant API as FastAPI Router
    participant Schema as Pydantic Schema
    participant ORM as SQLAlchemy ORM
    participant DB as PostgreSQL

    Client->>API: POST /api/designer/config (camelCase JSON)
    API->>Schema: SaveConfigRequest.model_validate(json)
    Schema->>Schema: PipelineConfigurationSchema validates nested stages
    Schema->>ORM: .model_dump(by_alias=True) produces camelCase dict
    ORM->>DB: INSERT INTO pipeline_configs (config JSONB ...)
    DB-->>ORM: row with server_default timestamps
    ORM-->>Schema: model_validate(orm_obj) via from_attributes=True
    Schema-->>API: SaveConfigResponse (camelCase JSON)
    API-->>Client: 201 Created
```

---

### Design Level 9c — Alembic Migration Architecture

```mermaid
graph TB
    subgraph "Migration Tooling"
        INI[alembic.ini]
        ENV[alembic/env.py - asyncio.run + create_async_engine]
        MAKO[alembic/script.py.mako]
        V001[alembic/versions/001_initial_schema.py]
    end

    subgraph "Python App Layer"
        SETTINGS[app/config.py - get_settings DATABASE_URL]
        BASE[app/models/base.py - Base + TimestampMixin]
        INIT[app/models/__init__.py - registers all metadata]
    end

    subgraph "PostgreSQL"
        PG[(ragstudio DB)]
        AM[alembic_version table]
    end

    INI --> ENV
    ENV --> SETTINGS
    ENV --> INIT
    INIT --> BASE
    ENV --> V001
    V001 --> PG
    PG --> AM
    SCRIPTS[scripts/migrate.sh] --> ENV
```

---

## Phase P2-1 · Document Ingestion Service

**What changed:** Implemented the first core service layer module — a format-agnostic document ingestion pipeline. The backend can now load text from PDFs, DOCX, TXT, Markdown, HTML, CSV, JSON files, and remote URLs, clean the text, extract rich metadata, and return a unified list of LangChain `Document` objects ready for the downstream chunking service.

### Design Level 10 — Document Ingestion Pipeline

```mermaid
flowchart TD
    subgraph INPUT["Input Sources"]
        FILE["File path\n.pdf / .docx / .txt / .md\n.html / .csv / .json"]
        BYTES["Raw bytes\n(uploaded file)"]
        URL["Remote URL\nhttps://..."]
    end

    subgraph INGESTION_SERVICE["IngestionService  (app/core/ingestion/__init__.py)"]
        IS_LOAD["load(source, config)"]
        IS_LOADRAW["_load_raw(source)"]
        IS_META["_extract_metadata(source, doc)"]
        IS_FILTER["filter empty docs"]
    end

    subgraph LOADER_FACTORY["LoaderFactory  (loaders.py)"]
        LF_EXT["from_extension(ext)"]
        LF_PATH["from_path(path)"]
        LF_URL["for_url()"]
    end

    subgraph LOADERS["Concrete Loaders  (loaders.py)"]
        PDF_L["PDFLoader\npypdf → 1 doc/page"]
        DOCX_L["DOCXLoader\npython-docx → 1 doc"]
        TXT_L["TextLoader\nplain text / Markdown"]
        HTML_L["HTMLLoader\nBeautifulSoup tag strip"]
        CSV_L["CSVLoader\n1 doc/row as key:val"]
        JSON_L["JSONLoader\nlist→1 doc/item\nobj→1 doc"]
        URL_L["URLLoader\ntrafilatura body extract"]
    end

    subgraph PREPROCESSORS["TextPreprocessor  (preprocessors.py)"]
        FE["fix_encoding()\n• NFC normalise\n• strip null bytes"]
        SH["strip_html_tags()\n(optional)"]
        RHF["remove_headers_footers()\n(optional, form-feed pages)"]
        NW["normalize_whitespace()\n• collapse spaces/tabs\n• cap newline runs"]
    end

    subgraph EXTRACTORS["Metadata Extractors  (extractors.py)"]
        EPDF["extract_pdf_metadata()\ntitle · author · page_count"]
        EDOCX["extract_docx_metadata()\ncore_properties"]
        EHTML["extract_html_metadata()\n<title> · meta tags · og:*"]
        EURL["extract_url_metadata()\nsource_url + html meta"]
        EFILE["extract_file_metadata()\nfilename · extension · size"]
        ESEC["extract_section_headers()\n# ATX + title-case heuristic"]
    end

    subgraph OUTPUT["Output"]
        DOCS["list[Document]\npage_content: cleaned text\nmetadata: source · file_type\npage_number · title · author\nsection_headers · custom_metadata"]
    end

    FILE --> IS_LOAD
    BYTES --> IS_LOAD
    URL --> IS_LOAD

    IS_LOAD --> IS_LOADRAW
    IS_LOADRAW --> LF_EXT
    IS_LOADRAW --> LF_PATH
    IS_LOADRAW --> LF_URL

    LF_EXT & LF_PATH --> PDF_L & DOCX_L & TXT_L & HTML_L & CSV_L & JSON_L
    LF_URL --> URL_L

    PDF_L & DOCX_L & TXT_L & HTML_L & CSV_L & JSON_L & URL_L --> IS_LOAD

    IS_LOAD --> FE --> SH --> RHF --> NW
    NW --> IS_FILTER
    IS_FILTER --> IS_META

    IS_META --> EPDF
    IS_META --> EDOCX
    IS_META --> EHTML
    IS_META --> EURL
    IS_META --> EFILE
    IS_META --> ESEC

    EPDF & EDOCX & EHTML & EURL & EFILE & ESEC --> OUTPUT
    IS_FILTER --> OUTPUT
```

### Design Level 10b — Ingestion Service in Full Stack Context

```mermaid
graph TB
    subgraph API["FastAPI Backend (apps/api)"]
        subgraph CORE["app/core/ (P2-x services — being built)"]
            subgraph INGESTION["app/core/ingestion/ ✅ P2-1"]
                ING_SVC["IngestionService\nload() / load_many()"]
                LOADERS_M["loaders.py\n7 format loaders"]
                PREPRO_M["preprocessors.py\nTextPreprocessor"]
                EXTRAC_M["extractors.py\n6 metadata extractors"]
            end
            CHUNKING["app/core/chunking/\n✅ P2-2"]
            EMBEDDING["app/core/embedding/\n✅ P2-3"]
            VECTORSTORE["app/core/vectorstore/\n⬜ P2-4"]
            RETRIEVAL["app/core/retrieval/\n⬜ P2-5"]
            GENERATION["app/core/generation/\n⬜ P2-6"]
        end

        ROUTERS["app/routers/\n(will call IngestionService\nvia Designer/Autopilot endpoints)"]
        MODELS_M["app/models/ ✅ P1-4"]
        SCHEMAS_M["app/schemas/ ✅ P1-3"]
    end

    DOCS_IN["Documents\n(files / URLs / bytes)"] --> ING_SVC
    ING_SVC --> LOADERS_M --> PREPRO_M --> EXTRAC_M
    ING_SVC --> CHUNKING
    CHUNKING --> EMBEDDING
    EMBEDDING --> VECTORSTORE
    VECTORSTORE --> RETRIEVAL
    RETRIEVAL --> GENERATION
    ROUTERS --> ING_SVC
    SCHEMAS_M -.->|IngestionConfig| ING_SVC
```

---

## Phase P2-2 · Chunking Service

**What changed:** Implemented the second core service — the Chunking layer. The service accepts `list[Document]` from the Ingestion Service and returns `list[Chunk]` (a `Document` type alias with enriched metadata). Eight chunking strategies are available, each isolated in its own module. A `ChunkerFactory` dispatches to the correct implementation via a strategy-name map. `ChunkQualityScorer` enables Autopilot agents to filter low-quality chunks before embedding.

### Design Level 11 — Chunking Service Architecture

```mermaid
graph TD
    subgraph CHUNKING_PKG["app/core/chunking/ ✅ P2-2"]
        INIT["__init__.py\nChunkingService · ChunkerFactory\n_STRATEGY_MAP\n8 strategies registered"]

        subgraph BASE["strategies.py (base layer)"]
            CHUNK_ALIAS["Chunk = Document\n(type alias)"]
            CFG["ChunkingConfig\n(dataclass)\n12 config fields"]
            ABC_CLS["TextChunker (ABC)\nchunk() abstract\n_make_chunk() helper"]
        end

        subgraph STRATEGIES["Concrete Chunkers"]
            FX["fixed_size.py\nFixedSizeChunker\nPure character sliding window\nZero external deps"]
            RC["recursive.py\nRecursiveCharacterChunker\nLangChain RCTS\nLazy import"]
            SEM["semantic.py\nSemanticChunker\nsentence-transformers\nCosine similarity\nPer-instance model cache"]
            DOC["document_based.py\nMarkdownHeaderChunker\nHTMLSectionChunker\nBeautifulSoup DOM walk"]
            CODE["code_aware.py\nCodeAwareChunker\nLanguage enum dispatch\n12 extension mappings"]
            SENT["sentence.py\nSentenceChunker\nParagraphChunker\nRegex boundary detection"]
        end

        subgraph QUALITY["optimizers.py (quality layer)"]
            METRICS["ChunkQualityMetrics\ncontent_density\ncompleteness\nsize_score\noverall"]
            SCORER["ChunkQualityScorer\nweighted scoring\nfilter_low_quality()\nscore_batch()"]
        end
    end

    ABC_CLS --> FX & RC & SEM & DOC & CODE & SENT
    CFG --> ABC_CLS
    CHUNK_ALIAS --> ABC_CLS
    FX & RC & SEM & DOC & CODE & SENT --> INIT
    METRICS --> SCORER
```

### Design Level 11b — Chunking Data Flow

```mermaid
sequenceDiagram
    participant CALLER as Caller<br/>(Agent or Router)
    participant SVC as ChunkingService
    participant FAC as ChunkerFactory
    participant STRAT as ConcreteChunker
    participant ABC as TextChunker._make_chunk
    participant SCORER as ChunkQualityScorer

    CALLER->>SVC: chunk(docs, ChunkingConfig(strategy="semantic"))
    SVC->>FAC: from_strategy("semantic")
    FAC-->>SVC: SemanticChunker()
    SVC->>STRAT: chunker.chunk(docs, config)
    loop For each Document
        STRAT->>STRAT: split text into raw pieces
        loop For each raw piece
            STRAT->>ABC: _make_chunk(text, parent_meta, i, total, "semantic")
            ABC-->>STRAT: Chunk (Document + enriched metadata)
        end
    end
    STRAT-->>SVC: list[Chunk]
    SVC-->>CALLER: list[Chunk]

    Note over CALLER,SCORER: Optional quality filtering step
    CALLER->>SCORER: filter_low_quality(chunks, min_score=0.5)
    loop For each Chunk
        SCORER->>SCORER: score(chunk) → ChunkQualityMetrics
    end
    SCORER-->>CALLER: filtered list[Chunk]
```

### Design Level 11c — Chunking Service in Full Stack Context

```mermaid
graph TB
    subgraph API["FastAPI Backend (apps/api)"]
        subgraph CORE["app/core/ (P2-x services)"]
            subgraph INGESTION_BOX["app/core/ingestion/ ✅ P2-1"]
                ING_SVC2["IngestionService\nload() → list[Document]"]
            end

            subgraph CHUNKING_BOX["app/core/chunking/ ✅ P2-2"]
                CHUNK_SVC["ChunkingService\nchunk() / chunk_many()"]
                CHUNK_FAC["ChunkerFactory\nfrom_strategy()"]
                EIGHT_STRAT["8 Chunkers\nfixed-size · recursive · semantic\nmarkdown · html · sentence\nparagraph · code-aware"]
                QUALITY_SVC["ChunkQualityScorer\nfilter_low_quality()"]
            end

            EMBEDDING_BOX["app/core/embedding/\n✅ P2-3"]
            VECTORSTORE_BOX["app/core/vectorstore/\n⬜ P2-4 next"]
            RETRIEVAL_BOX["app/core/retrieval/\n⬜ P2-5"]
            GENERATION_BOX["app/core/generation/\n⬜ P2-6"]
        end

        AGENTS["app/agents/\nAutopilot uses ChunkingService\n+ ChunkQualityScorer\nfor optimization loop"]
        ROUTERS2["app/routers/\nDesigner endpoints\ncall ChunkingService"]
    end

    DOCS["Documents\n(list[Document])"] --> ING_SVC2
    ING_SVC2 -->|"list[Document]"| CHUNK_SVC
    CHUNK_SVC --> CHUNK_FAC --> EIGHT_STRAT
    EIGHT_STRAT -->|"list[Chunk]"| QUALITY_SVC
    QUALITY_SVC -->|"filtered list[Chunk]"| EMBEDDING_BOX
    EMBEDDING_BOX --> VECTORSTORE_BOX --> RETRIEVAL_BOX --> GENERATION_BOX
    AGENTS --> CHUNK_SVC
    AGENTS --> QUALITY_SVC
    ROUTERS2 --> CHUNK_SVC
```

---

## Phase P2-3 · Embedding Service

**What changed:** Implemented the third core service — the Embedding layer. The service accepts `list[Document]` (output of ChunkingService) and returns `list[tuple[Document, Embedding]]` — each chunk paired with its float vector, ready for upsert into the vector store. Five provider wrappers cover every model in the catalog (OpenAI, Cohere, Google, HuggingFace, Nomic). An `EmbeddingBenchmarker` lets Autopilot agents compare providers on throughput. An `EmbeddingCache` backed by Redis (with in-process dict fallback) eliminates redundant API calls for duplicate texts.

### Design Level 12 — Embedding Service Architecture

```mermaid
graph TD
    subgraph EMBEDDING_PKG["app/core/embedding/ ✅ P2-3"]
        INIT["__init__.py\nEmbeddingService · EmbedderFactory\n_PROVIDER_MAP\n5 providers registered"]

        subgraph BASE["strategies.py (base layer)"]
            EMB_ALIAS["Embedding = list[float]\n(type alias)"]
            CFG["EmbeddingConfig\n(dataclass)\nmodel · provider · dimensions\nbatch_size · max_tokens"]
            ABC_CLS["TextEmbedder (ABC)\nembed_documents() abstract\nembed_query() abstract"]
        end

        subgraph PROVIDERS["Concrete Embedders"]
            OAI["openai.py\nOpenAIEmbedder\nlangchain-openai\nMatryoshka dimensions support\nBatch: 100 texts"]
            COH["cohere.py\nCohereEmbedder\nlangchain-community\nCatalog ID → API name map\nBatch: 96 texts (API limit)"]
            GOO["google.py\nGoogleEmbedder\nlangchain-google-genai\ntext-embedding-004 (Gecko)\nBatch: 100 texts"]
            HF["huggingface.py\nHuggingFaceEmbedder\nsentence-transformers\nPer-instance model cache\nL2-normalised · Batch: 32"]
            NOM["nomic.py\nNomicEmbedder\nsentence-transformers\ntrust_remote_code=True\nSingleton model cache · 8K ctx"]
        end

        subgraph UTILITIES["Support Modules"]
            BENCH["benchmarker.py\nBenchmarkResult (dataclass)\nEmbeddingBenchmarker\nbenchmark() → sorted by\ntexts_per_second"]
            CACHE["cache.py\nEmbeddingCache\nSHA-256 key derivation\nRedis (binary pack 4B/float)\nIn-memory dict fallback\nembed_with_cache()"]
        end
    end

    ABC_CLS --> OAI & COH & GOO & HF & NOM
    CFG --> ABC_CLS
    EMB_ALIAS --> ABC_CLS
    OAI & COH & GOO & HF & NOM --> INIT
    BENCH --> INIT
    CACHE --> INIT
```

### Design Level 12b — Embedding Data Flow

```mermaid
sequenceDiagram
    participant CALLER as Caller<br/>(Agent or Router)
    participant SVC as EmbeddingService
    participant CACHE as EmbeddingCache
    participant FAC as EmbedderFactory
    participant PROV as ConcreteEmbedder
    participant REDIS as Redis

    CALLER->>SVC: embed(chunks, EmbeddingConfig(provider="openai"))
    SVC->>FAC: from_provider("openai")
    FAC-->>SVC: OpenAIEmbedder()
    SVC->>SVC: extract page_content from each chunk

    alt Cache enabled
        SVC->>CACHE: embed_with_cache(embedder, texts, config)
        loop For each text
            CACHE->>REDIS: GET emb:<sha256>
            REDIS-->>CACHE: hit → packed bytes OR miss → nil
        end
        Note over CACHE: Collect miss_texts
        CACHE->>PROV: embed_documents(miss_texts, config)
        PROV-->>CACHE: list[Embedding]
        loop For each miss
            CACHE->>REDIS: SETEX emb:<sha256> TTL packed_bytes
        end
        CACHE-->>SVC: list[Embedding] (hits + fresh)
    else No cache
        SVC->>PROV: embed_documents(texts, config)
        PROV-->>SVC: list[Embedding]
    end

    loop For each (chunk, vector)
        SVC->>SVC: enrich metadata with embedding_model,\nembedding_provider, embedding_dimensions
    end
    SVC-->>CALLER: list[tuple[Document, Embedding]]
```

### Design Level 12c — Embedding Service in Full Stack Context

```mermaid
graph TB
    subgraph API["FastAPI Backend (apps/api)"]
        subgraph CORE["app/core/ (P2-x services)"]
            subgraph INGESTION_BOX3["app/core/ingestion/ P2-1"]
                ING_SVC3["IngestionService\nload() -> list Document"]
            end

            subgraph CHUNKING_BOX3["app/core/chunking/ P2-2"]
                CHUNK_SVC3["ChunkingService\nchunk() / chunk_many()"]
                QUALITY_SVC3["ChunkQualityScorer\nfilter_low_quality()"]
            end

            subgraph EMBEDDING_BOX3["app/core/embedding/ P2-3"]
                EMB_SVC["EmbeddingService\nembed() / embed_query()\nembed_many()"]
                EMB_FAC["EmbedderFactory\nfrom_provider()"]
                FIVE_PROV["5 Embedders\nopenai cohere google\nhuggingface nomic"]
                BENCH_SVC["EmbeddingBenchmarker\nbenchmark()"]
                CACHE_SVC["EmbeddingCache\nRedis + memory fallback"]
            end

            VECTORSTORE_BOX3["app/core/vectorstore/ P2-4 next"]
            RETRIEVAL_BOX3["app/core/retrieval/ P2-5"]
            GENERATION_BOX3["app/core/generation/ P2-6"]
        end

        AGENTS3["app/agents\nEmbedding Tester Agent"]
        ROUTERS3["app/routers\nDesigner endpoints"]
        REDIS3["Redis Cache"]
    end

    DOCS3["Documents"] --> ING_SVC3
    ING_SVC3 -->|list Document| CHUNK_SVC3
    CHUNK_SVC3 -->|list Chunk| QUALITY_SVC3
    QUALITY_SVC3 -->|filtered chunks| EMB_SVC
    EMB_SVC --> EMB_FAC --> FIVE_PROV
    EMB_SVC <--> CACHE_SVC <--> REDIS3
    FIVE_PROV -->|document embedding pairs| VECTORSTORE_BOX3
    VECTORSTORE_BOX3 --> RETRIEVAL_BOX3 --> GENERATION_BOX3
    AGENTS3 --> BENCH_SVC
    BENCH_SVC --> FIVE_PROV
    ROUTERS3 --> EMB_SVC
```
