# Project system design evolution — Phase 1 (Shared contracts — P1-1 … P1-4)

> Part of the [master index](./PROJECT_SYSTEM_DESIGN_EVOLUTION.md).

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
| P2-4 | Vector store core | `apps/api/app/core/vectorstore/` — Qdrant / Pinecone / Weaviate clients, `VectorStoreFactory`, `VectorStoreService` |

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

