# RAG Studio — Implementation Task Board

> **Branch Strategy:**  
> `main` — production-ready releases only  
> `develop` — integration branch; all feature branches merge here first  
> `feature/<phase-id>-<slug>` — one branch per task below  
>  
> **Workflow:** `feature/*` → PR → `develop` → tested → PR → `main`  
> **Dependency rule:** Complete tasks in the order listed. Later phases depend on earlier ones being merged to `develop`.

---

## Phase 0 — Project Bootstrap & Infrastructure
> **Goal:** Get the monorepo running locally with all services wired up.  
> **No code dependencies.** Start here.

---

### P0-1 · Monorepo Skeleton ✅
**Branch:** `feature/p0-monorepo-skeleton`

- [x] Create root `rag-studio/` directory layout as specified in CLAUDE.md §Project Structure
- [x] Add root `package.json` with workspaces: `apps/web`, `apps/api`
- [x] Add `.gitignore` (Node, Python, Docker, env files, `.pyc`, `__pycache__`, `.next`, `dist`)
- [x] Add `.env.example` with all required variables:
  - `DATABASE_URL`, `REDIS_URL`, `QDRANT_URL`
  - `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `COHERE_API_KEY`
  - `MLFLOW_TRACKING_URI`, `MINIO_ENDPOINT`
  - `NEXT_PUBLIC_API_URL`
- [x] Add root `README.md` quick-start block
- [x] Create empty placeholder directories: `apps/web/`, `apps/api/`, `data/`, `docs/`, `scripts/`, `docker/`, `k8s/`

---

### P0-2 · Docker Compose Development Environment ✅
**Branch:** `feature/p0-docker-compose-dev`

> **Depends on:** P0-1 merged

- [x] Create `docker/docker-compose.yml` with services:
  - `web` (Next.js, port 3000)
  - `api` (FastAPI/Uvicorn, port 8000)
  - `db` (PostgreSQL 16, port 5432)
  - `redis` (Redis 7-alpine, port 6379)
  - `vector-db` (Qdrant latest, port 6333)
  - `mlflow` (port 5000)
  - `minio` (S3-compatible, ports 9000/9001)
  - `worker` (Celery worker, depends on redis + api)
- [x] Create `docker/docker-compose.dev.yml` override (hot-reload volumes, debug ports)
- [x] Create `docker/docker-compose.prod.yml` override (no dev mounts, resource limits)
- [x] Create `docker/nginx/nginx.conf` (reverse proxy: `/` → web:3000, `/api` → api:8000)
- [x] Add health-check directives to each service
- [x] Verify `docker compose up -d` starts all 8 services cleanly

---

### P0-3 · CI/CD Pipelines ✅
**Branch:** `feature/p0-cicd-pipelines`

> **Depends on:** P0-1 merged

- [x] Create `.github/workflows/ci.yml`:
  - Triggers: push to `develop`, PRs targeting `develop` or `main`
  - Jobs: lint (ESLint + Ruff), type-check (tsc), unit tests (pytest + jest)
- [x] Create `.github/workflows/cd.yml`:
  - Triggers: push to `main`
  - Jobs: build Docker images, push to GHCR, deploy (placeholder step)
- [x] Create `.github/workflows/tests.yml`:
  - Triggers: manual dispatch + PR
  - Jobs: integration tests, E2E tests
- [x] Add branch protection rules documentation in `CONTRIBUTING.md`

---

### P0-4 · Backend Project Scaffold ✅
**Branch:** `feature/p0-backend-scaffold`

> **Depends on:** P0-1 merged

- [x] Initialise `apps/api/` Python project:
  - `pyproject.toml` with tool configs (Ruff, mypy, pytest)
  - `requirements.txt` (FastAPI, uvicorn, sqlalchemy, alembic, celery, redis, qdrant-client, langchain, langgraph, ragas, mlflow, tiktoken, python-dotenv, pydantic-settings)
  - `requirements-dev.txt` (pytest, httpx, factory-boy, faker)
- [x] Create `apps/api/app/__init__.py`
- [x] Create `apps/api/app/main.py` — bare FastAPI app, CORS, lifespan handler
- [x] Create `apps/api/app/config.py` — `pydantic-settings` Settings class reading from env
- [x] Create `apps/api/app/dependencies.py` — DB session, Redis, Qdrant client DI
- [x] Create `apps/api/Dockerfile` (multi-stage: builder + runtime)
- [x] Verify `uvicorn app.main:app` starts and `/health` returns 200

---

### P0-5 · Frontend Project Scaffold ✅
**Branch:** `feature/p0-frontend-scaffold`

> **Depends on:** P0-1 merged

- [x] Initialise Next.js 14 App Router project inside `apps/web/`:
  - `npx create-next-app@latest --typescript --tailwind --eslint --app --src-dir`
- [x] Install core dependencies:
  - `shadcn/ui` (init with default theme)
  - `zustand`, `react-hook-form`, `zod`
  - `lucide-react`, `framer-motion`
  - `mermaid`, `recharts`, `prismjs`
  - `@tanstack/react-query`
- [x] Add `components.json` for shadcn/ui
- [x] Create `apps/web/src/lib/api-client.ts` — typed fetch wrapper pointing at `NEXT_PUBLIC_API_URL`
- [x] Create `apps/web/src/lib/utils.ts` — `cn()` helper
- [x] Create `apps/web/src/lib/constants.ts` — stage names, route map
- [x] Create `apps/web/Dockerfile` (multi-stage: builder + runtime)
- [x] Verify `npm run dev` starts and `localhost:3000` loads

---

## Phase 1 — Shared Data Layer
> **Goal:** Establish the single source of truth for all models, providers, and types.  
> **No backend or frontend logic yet.** Both sides import from here.

---

### P1-1 · JSON Model Catalogs ✅
**Branch:** `feature/p1-json-model-catalogs`

> **Depends on:** P0-1 merged

Create all catalog files under `data/`:

- [x] `data/models/embeddings.json`
  - Fields per model: `id`, `name`, `provider`, `dimensions`, `maxTokens`, `costPer1MTokens`, `speed`, `quality`, `tier`
  - Models: `text-embedding-3-small`, `text-embedding-3-large`, `text-embedding-ada-002`, `cohere-embed-v3`, `cohere-embed-multilingual`, `google-textembedding-gecko`, `bge-large-en`, `e5-large-v2`, `all-MiniLM-L6-v2`, `nomic-embed-text`
- [x] `data/models/generation.json`
  - Fields: `id`, `name`, `provider`, `contextWindow`, `costInput`, `costOutput`, `tier`, `strengths`
  - Models: `gpt-4o`, `gpt-4o-mini`, `claude-opus-4-7`, `claude-sonnet-4-6`, `gemini-1.5-pro`, `gemini-1.5-flash`, `llama-3-70b`, `mistral-large`, `command-r-plus`
- [x] `data/models/rerankers.json`
  - Models: `cohere-rerank-v3`, `bge-reranker-large`, `cross-encoder-ms-marco`, `flashrank`
- [x] `data/chunking-strategies.json`
  - 7 strategies: `fixed-size`, `recursive-character`, `semantic`, `markdown-header`, `sentence-based`, `paragraph-based`, `code-aware`
  - Fields per strategy: `id`, `name`, `description`, `bestFor`, `pros`, `cons`, `implementationComplexity`, `defaultConfig`
- [x] `data/vector-stores.json`
  - 9 stores: `qdrant`, `pinecone`, `weaviate`, `chroma`, `faiss`, `opensearch`, `vertex-ai-vector-search`, `azure-ai-search`, `pgvector`
  - Fields: `id`, `name`, `type` (managed/self-hosted/embedded), `bestFor`, `pricing`, `cloudNative`
- [x] `data/retrieval-strategies.json`
  - 6 strategies: `similarity`, `mmr`, `hybrid`, `parent-child`, `multi-query`, `ensemble`
- [x] `data/cloud-providers.json`
  - 4 providers: `aws`, `gcp`, `azure`, `multi-cloud`
  - Fields: `id`, `name`, `description`, `bestFor`, `strengths`, `nativeServices`
- [x] `data/templates.json`
  - Templates: `faq-chatbot`, `documentation-qa`, `code-assistant`, `multilingual-support`, `legal-research`, `customer-support`
  - Each with a pre-filled `PipelineConfiguration`
- [x] `data/pricing.json`
  - Pricing formulas for cost calculator: embedding cost/token, vector storage/GB, retrieval cost/query, generation cost/token per provider

---

### P1-2 · TypeScript Shared Types ✅
**Branch:** `feature/p1-typescript-types`

> **Depends on:** P0-5, P1-1 merged

Create all type files under `apps/web/src/types/`:

- [x] `apps/web/src/types/pipeline.ts`
  - `CloudProvider`, `ModelTier`, `ChunkingStrategy`, `VectorStoreProvider`, `RetrievalStrategy`
  - `PipelineConfiguration`, `ChunkingConfig`, `EmbeddingConfig`, `VectorStoreConfig`
  - `RetrievalConfig`, `RerankingConfig`, `GenerationConfig`, `RoutingConfig`, `MemoryConfig`, `EvaluationConfig`
  - `DataIngestionConfig`, `MetadataFilter`, `HybridSearchConfig`
  - `CostEstimate`, `CostBreakdown`, `PerformanceEstimate`
- [x] `apps/web/src/types/autopilot.ts`
  - `AutopilotBuild`, `BuildRequirements`, `StageStatus`, `BuildMessage`, `BuildResult`
  - `AgentDecisions`, `DeploymentInfo`
- [x] `apps/web/src/types/models.ts`
  - `EmbeddingModel`, `GenerationModel`, `RerankerModel`
  - `ChunkingStrategyMeta`, `VectorStoreMeta`, `CloudProviderMeta`, `Template`
- [x] `apps/web/src/types/cloud.ts`
  - `CloudProvider`, `CloudProviderConfig`, `CloudNativeService`
- [x] `apps/web/src/types/index.ts` — re-exports everything

---

### P1-3 · Python Pydantic Schemas ✅
**Branch:** `feature/p1-python-schemas`

> **Depends on:** P0-4, P1-1 merged

Create all schema files under `apps/api/app/schemas/`:

- [x] `apps/api/app/schemas/pipeline.py`
  - `CloudProvider` enum, `ChunkingStrategy` enum, `VectorStoreProvider` enum, `RetrievalStrategy` enum
  - `ChunkingConfigSchema`, `EmbeddingConfigSchema`, `VectorStoreConfigSchema`
  - `RetrievalConfigSchema`, `RerankingConfigSchema`, `GenerationConfigSchema`
  - `PipelineConfigurationSchema`, `CostEstimateSchema`
- [x] `apps/api/app/schemas/designer.py`
  - `SaveConfigRequest`, `SaveConfigResponse`, `ExportRequest`, `ExportResponse`, `CostRequest`
- [x] `apps/api/app/schemas/autopilot.py`
  - `BuildRequirementsSchema`, `StartBuildRequest`, `StartBuildResponse`
  - `BuildStatusResponse`, `BuildResultSchema`, `AgentDecisionSchema`
- [x] `apps/api/app/schemas/evaluation.py`
  - `EvaluationRunRequest`, `EvaluationMetrics`, `FailureAnalysisResult`
- [x] `apps/api/app/schemas/deployment.py`
  - `DeployRequest`, `DeployResponse`, `DeploymentStatusResponse`
- [x] `apps/api/app/schemas/__init__.py` — re-exports

---

### ✅ P1-4 · Database Schema & Migrations
**Branch:** `feature/p1-database-schema`

> **Depends on:** P0-4, P1-3, P0-2 merged

- [x] Set up Alembic: configured `alembic.ini` + `alembic/env.py` inside `apps/api/`
- [x] Configure `alembic/env.py` to read `DATABASE_URL` from settings
- [x] Create SQLAlchemy base models in `apps/api/app/models/`:
  - [x] `apps/api/app/models/project.py` — `Project` (id, user_id, name, description, timestamps)
  - [x] `apps/api/app/models/pipeline_config.py` — `PipelineConfig` (id, project_id, name, version, cloud_provider, config JSONB, source, build_id, timestamps)
  - [x] `apps/api/app/models/build_history.py` — `AutopilotBuild` (id, project_id, status, progress, current_stage, iteration, requirements JSONB, stages JSONB, messages JSONB, result JSONB, error, timestamps)
  - [x] `apps/api/app/models/evaluation_run.py` — `EvaluationRun` (id, config_id, build_id, metrics JSONB, test_set_size, timestamps)
  - [x] `apps/api/app/models/deployment.py` — `Deployment` (id, config_id, provider, status, endpoint, deployment_info JSONB, timestamps)
  - [x] `apps/api/app/models/__init__.py`
- [x] Handwritten initial migration: `alembic/versions/001_initial_schema.py` (5 tables + indexes + FK constraints)
- [x] Create `scripts/migrate.sh` to run migrations from project root
- [x] Added `from_attributes=True` to `RAGBaseModel` to enable ORM → Pydantic conversion

---

## Phase 2 — Backend Core Services
> **Goal:** Implement the shared RAG pipeline services that both Designer and Autopilot consume.

---

### ✅ P2-1 · Document Ingestion Service
**Branch:** `feature/p2-ingestion-service`

> **Depends on:** P1-4 merged

Create `apps/api/app/core/ingestion/`:

- ✅ `loaders.py` — document loaders for: PDF (pypdf), DOCX (python-docx), TXT, Markdown, HTML (beautifulsoup4), CSV, JSON, URL (requests + trafilatura)
- ✅ `preprocessors.py` — text cleaning: strip HTML tags, normalize whitespace, remove headers/footers, fix encoding
- ✅ `extractors.py` — metadata extraction: title, author, date, source URL, page numbers, section headers
- ✅ `__init__.py` — `IngestionService` class with `load(source) -> List[Document]` method
- ✅ Unit tests in `apps/api/tests/test_core/test_ingestion.py`

---

### ✅ P2-2 · Chunking Service
**Branch:** `feature/p2-chunking-service`

> **Depends on:** P2-1 merged

Create `apps/api/app/core/chunking/`:

- ✅ `strategies.py` — `Chunk` type alias, `ChunkingConfig` dataclass, `TextChunker` ABC with `_make_chunk` helper
- ✅ `recursive.py` — `RecursiveCharacterChunker` (LangChain `RecursiveCharacterTextSplitter`, lazy import)
- ✅ `semantic.py` — `SemanticChunker` (sentence-transformers cosine similarity, buffered windows, per-instance model cache)
- ✅ `document_based.py` — `MarkdownHeaderChunker` (LangChain splitter + metadata merge), `HTMLSectionChunker` (BeautifulSoup DOM walk)
- ✅ `code_aware.py` — `CodeAwareChunker` (Language enum dispatch, 3-level language detection)
- ✅ `sentence.py` — `SentenceChunker` (regex boundary, sliding window with overlap), `ParagraphChunker` (double-newline split + fallback)
- ✅ `fixed_size.py` — `FixedSizeChunker` (pure character sliding window, zero external deps)
- ✅ `optimizers.py` — `ChunkQualityMetrics` dataclass, `ChunkQualityScorer` (density, completeness, size scoring with weight validation)
- ✅ `__init__.py` — `_STRATEGY_MAP` (8 strategies), `ChunkerFactory`, `ChunkingService` with `chunk()` and `chunk_many()`
- ✅ Unit tests in `apps/api/tests/test_core/test_chunking.py` — 53 tests covering all 8 strategies, factory, service, scorer, and metadata propagation

---

### ✅ P2-3 · Embedding Service
**Branch:** `feature/p2-embedding-service`

> **Depends on:** P1-1 merged (for model catalog), P2-2 merged

Create `apps/api/app/core/embedding/`:

- ✅ `strategies.py` — `Embedding` type alias, `EmbeddingConfig` dataclass, `TextEmbedder` ABC (`embed_documents`, `embed_query`)
- ✅ `openai.py`, `cohere.py`, `google.py`, `huggingface.py`, `nomic.py` — one `TextEmbedder` implementation per catalog provider (lazy imports; Cohere catalog-ID → API name mapping; Google Gecko → `text-embedding-004`; local models L2-normalised)
- ✅ `benchmarker.py` — `EmbeddingBenchmarker` + `BenchmarkResult`; ranks configs by `texts_per_second`; skips providers that error
- ✅ `cache.py` — `EmbeddingCache` (SHA-256 key `provider:model:dimensions:text`, Redis binary pack + in-memory fallback, `embed_with_cache`)
- ✅ `__init__.py` — `EmbedderFactory`, `EmbeddingService` (`embed`, `embed_query`, `embed_many`) with metadata enrichment (`embedding_model`, `embedding_provider`, `embedding_dimensions`)
- ✅ Unit tests in `apps/api/tests/test_core/test_embedding.py`

---

### ✅ P2-4 · Vector Store Service
**Branch:** `feature/p2-vectorstore-service`

> **Depends on:** P2-3 merged, P0-2 merged (Qdrant running)

Create `apps/api/app/core/vectorstore/`:

- ✅ `strategies.py` — `VectorStoreClient` ABC, `VectorStoreRuntimeConfig`, `VectorSearchFilter`, `ScoredDoc`, `VectorStoreConfigurationError`
- ✅ `qdrant_client.py` — `QdrantVectorStore`: async create/delete/upsert/search via ``AsyncQdrantClient``; payload `page_content` + `metadata`; basic Qdrant ``Filter`` mapping
- ✅ `pinecone_client.py` — `PineconeVectorStore`: lazy ``pinecone`` SDK import; ``asyncio.to_thread`` for upsert/query; optional ``pinecone_index_host`` (serverless)
- ✅ `weaviate_client.py` — `WeaviateVectorStore`: v1 REST + GraphQL ``nearVector`` (``httpx`` only; no ``weaviate-client`` dependency)
- ✅ `factory.py` — `VectorStoreFactory.create(provider, config, **kwargs)` → `VectorStoreClient` (qdrant / pinecone / weaviate)
- ✅ `__init__.py` — `VectorStoreService` with `index`, `index_pairs`, `search` (async); env fallbacks `PINECONE_API_KEY`, `WEAVIATE_URL`, `WEAVIATE_API_KEY`
- ✅ Unit tests in `apps/api/tests/test_core/test_vectorstore.py` — Qdrant ``location=":memory:"`` + factory errors + mocked Weaviate REST

---

### P2-5 · Retrieval Service
**Branch:** `feature/p2-retrieval-service`

> **Depends on:** P2-4 merged

Create `apps/api/app/core/retrieval/`:

- [ ] `strategies.py` — `RetrievalStrategy` abstract class
- [ ] `similarity.py` — plain cosine similarity top-k
- [ ] `mmr.py` — Maximum Marginal Relevance implementation
- [ ] `hybrid.py` — hybrid dense + sparse (BM25) search with alpha weighting
- [ ] `parent_child.py` — parent-child retrieval (retrieve child chunks, return parent context)
- [ ] `multi_query.py` — LLM generates N query variants, union results
- [ ] `ensemble.py` — RRF (Reciprocal Rank Fusion) ensemble
- [ ] `rerankers.py` — `RerankerService`: Cohere Rerank v3, BGE-Reranker, FlashRank
- [ ] `__init__.py` — `RetrievalService` with `retrieve(query, strategy, top_k, reranking_config) -> List[Document]`
- [ ] Unit tests in `apps/api/tests/test_core/test_retrieval.py`

---

### P2-6 · Generation Service
**Branch:** `feature/p2-generation-service`

> **Depends on:** P2-5 merged

Create `apps/api/app/core/generation/`:

- [ ] `llm_providers.py` — `LLMWrapper` abstract class + implementations:
  - `OpenAILLM` (gpt-4o, gpt-4o-mini)
  - `AnthropicLLM` (claude-opus-4-7, claude-sonnet-4-6) using Anthropic SDK with prompt caching
  - `GoogleLLM` (gemini-1.5-pro, gemini-1.5-flash)
  - `OllamaLLM` (local models)
- [ ] `prompts.py` — system prompt templates for RAG: basic QA, conversational, chain-of-thought, JSON output
- [ ] `chains.py` — `RAGChain`: retrieval → context formatting → LLM call → response
- [ ] `__init__.py` — `GenerationService` with `generate(query, context_docs, model, config) -> GenerationResponse`
- [ ] Unit tests in `apps/api/tests/test_core/test_generation.py`

---

### P2-7 · Evaluation Engine
**Branch:** `feature/p2-evaluation-engine`

> **Depends on:** P2-6 merged

Create `apps/api/app/core/evaluation/`:

- [ ] `ragas_eval.py` — RAGAS integration: `faithfulness`, `answer_relevancy`, `context_precision`, `context_recall`
- [ ] `metrics.py` — custom metrics: latency (p50/p95/p99), cost per query, chunk hit rate
- [ ] `test_sets.py` — `TestSetGenerator`: uses LLM to generate synthetic QA pairs from documents
- [ ] `analyzers.py` — `FailureAnalyzer`: categorise failures into `hallucination`, `retrieval_quality`, `context_gap`, `format_error`; produce fix recommendations
- [ ] `__init__.py` — `EvaluationEngine` with `evaluate(pipeline, test_set) -> EvaluationMetrics`
- [ ] Unit tests in `apps/api/tests/test_core/test_evaluation.py`

---

### P2-8 · Celery Worker & Task Queue
**Branch:** `feature/p2-celery-worker`

> **Depends on:** P0-2 merged, P2-6 merged

Create `apps/api/app/worker/`:

- [ ] `celery_app.py` — Celery app configured with Redis broker + backend
- [ ] `tasks.py` — async tasks:
  - `run_autopilot_build(build_id, documents, requirements)` → calls orchestrator
  - `run_evaluation(config_id, test_set)` → calls evaluation engine
  - `run_deployment(config_id, provider)` → calls deployment engine
- [ ] `schedules.py` — periodic tasks (e.g., clean up stale builds > 24h)
- [ ] Wire tasks to FastAPI background via `celery_app.send_task()`
- [ ] Unit tests in `apps/api/tests/test_worker.py`

---

### P2-9 · Health & Utility Endpoints
**Branch:** `feature/p2-health-endpoints`

> **Depends on:** P0-4, P1-4 merged

Create `apps/api/app/routers/`:

- [ ] `health.py` — `GET /health` (200 OK), `GET /health/db` (DB ping), `GET /health/redis`, `GET /health/qdrant`
- [ ] `apps/api/app/utils/logger.py` — structured JSON logger (using `structlog`)
- [ ] `apps/api/app/utils/validators.py` — common validation helpers
- [ ] `apps/api/app/utils/cost_calculator.py` — `calculate_cost(pipeline_config) -> CostEstimate` using `data/pricing.json`
- [ ] `apps/api/app/utils/helpers.py` — pagination, response formatting
- [ ] Register routers in `main.py`
- [ ] Integration test: `GET /health` returns 200 in Docker environment

---

## Phase 3 — Frontend Foundation
> **Goal:** Shared UI infrastructure — layout, navigation, state stores, and landing page.

---

### P3-1 · shadcn/ui Component Library Setup
**Branch:** `feature/p3-shadcn-components`

> **Depends on:** P0-5 merged

Add all required shadcn/ui components:

- [ ] Run `npx shadcn@latest add` for: `button`, `card`, `select`, `slider`, `tabs`, `badge`, `dialog`, `tooltip`, `separator`, `accordion`, `alert`, `progress`, `table`, `textarea`, `label`, `input`, `sheet`, `dropdown-menu`, `avatar`, `skeleton`, `switch`, `form`
- [ ] Create `apps/web/src/components/ui/` — verify all components present
- [ ] Extend `tailwind.config.ts` with custom colour tokens:
  - `primary` (blue scale), `success` (green), `warning` (amber), `danger` (red), `neutral` (grey)
- [ ] Add global CSS variables in `globals.css` for light/dark mode tokens
- [ ] Create `apps/web/src/components/shared/LoadingSpinner.tsx`
- [ ] Create `apps/web/src/components/shared/InfoTooltip.tsx`

---

### P3-2 · Zustand State Stores
**Branch:** `feature/p3-zustand-stores`

> **Depends on:** P1-2, P3-1 merged

- [ ] `apps/web/src/store/designerStore.ts`
  - State: `config: PipelineConfiguration`, `currentStage: string`, `isDirty: boolean`
  - Actions: `setCloudProvider`, `updateStage`, `setConfig`, `resetConfig`, `setCurrentStage`
  - Persistence: `zustand/middleware/persist` (localStorage)
- [ ] `apps/web/src/store/autopilotStore.ts`
  - State: `currentBuild: AutopilotBuild | null`, `messages: BuildMessage[]`, `history: AutopilotBuild[]`
  - Actions: `startBuild`, `updateBuild`, `addMessage`, `clearBuild`
- [ ] `apps/web/src/store/projectStore.ts`
  - State: `projects: Project[]`, `activeProject: Project | null`
  - Actions: `loadProjects`, `setActiveProject`, `createProject`, `deleteProject`
- [ ] Unit tests for each store (zustand testing patterns)

---

### P3-3 · App Layout & Navigation
**Branch:** `feature/p3-app-layout`

> **Depends on:** P3-1, P3-2 merged

- [ ] `apps/web/src/app/layout.tsx` — root layout with `<html>`, `<body>`, QueryClientProvider, font (Geist)
- [ ] `apps/web/src/app/globals.css` — Tailwind base + CSS variable definitions
- [ ] `apps/web/src/components/shared/` nav components:
  - `Navbar.tsx` — logo, mode switcher (Designer / Autopilot), project dropdown, user avatar
  - `Sidebar.tsx` — collapsible sidebar for project management
  - `ModeToggle.tsx` — pill toggle between Designer and Autopilot
- [ ] `apps/web/src/app/not-found.tsx` — custom 404 page
- [ ] `apps/web/src/app/error.tsx` — global error boundary

---

### P3-4 · Landing Page
**Branch:** `feature/p3-landing-page`

> **Depends on:** P3-3 merged

Create `apps/web/src/app/page.tsx` and landing components in `apps/web/src/components/landing/`:

- [ ] `Hero.tsx` — headline, sub-headline, two CTA buttons (Designer / Autopilot), animated gradient background
- [ ] `ModeComparison.tsx` — side-by-side cards: Designer mode vs Autopilot mode features
- [ ] `HowItWorks.tsx` — numbered steps for each mode
- [ ] `Features.tsx` — feature grid (6 key features with icons)
- [ ] `UseCases.tsx` — persona cards: Learning Engineer, Time-Strapped Startup, Enterprise Architect
- [ ] `Pricing.tsx` — Free / Pro / Enterprise tier cards
- [ ] `CTA.tsx` — bottom call-to-action
- [ ] `apps/web/src/app/page.tsx` — assembles all landing sections

---

### P3-5 · Lib Utilities & Validators
**Branch:** `feature/p3-lib-utilities`

> **Depends on:** P1-1, P1-2 merged

- [ ] `apps/web/src/lib/validators.ts` — Zod schemas matching TypeScript types (PipelineConfiguration, BuildRequirements)
- [ ] `apps/web/src/lib/constants.ts` — stage route map, default configs, magic numbers
- [ ] `apps/web/src/lib/generators/mermaidGenerator.ts` — `generateMermaidDiagram(stages, cloudProvider) -> string`
- [ ] `apps/web/src/lib/generators/pythonCodeGenerator.ts` — `generatePythonCode(config) -> string` (LangChain LCEL)
- [ ] `apps/web/src/lib/generators/yamlGenerator.ts` — `generateYAML(config) -> string`
- [ ] `apps/web/src/lib/generators/terraformGenerator.ts` — `generateTerraform(config, cloudProvider) -> string`
- [ ] Unit tests for all generators with snapshot testing

---

## Phase 4 — Designer Mode Backend
> **Goal:** All API endpoints required by the Designer mode frontend.

---

### P4-1 · Projects API
**Branch:** `feature/p4-projects-api`

> **Depends on:** P1-4, P2-9 merged

Create `apps/api/app/routers/projects.py` and `apps/api/app/services/project_service.py`:

- [ ] `POST /api/projects` — create project
- [ ] `GET /api/projects` — list projects (paginated)
- [ ] `GET /api/projects/{id}` — project detail with configs + builds
- [ ] `PUT /api/projects/{id}` — update name/description
- [ ] `DELETE /api/projects/{id}` — soft delete
- [ ] `ProjectService` with full CRUD using SQLAlchemy async sessions
- [ ] Integration tests in `apps/api/tests/test_projects.py`

---

### P4-2 · Designer Config API
**Branch:** `feature/p4-designer-config-api`

> **Depends on:** P4-1, P2-9 merged

Create `apps/api/app/routers/designer.py` and `apps/api/app/services/designer_service.py`:

- [ ] `POST /api/designer/config` — save or create pipeline configuration
- [ ] `GET /api/designer/config/{id}` — load configuration by ID
- [ ] `PUT /api/designer/config/{id}` — update existing configuration
- [ ] `GET /api/designer/configs?project_id=` — list configs for a project
- [ ] `DELETE /api/designer/config/{id}` — delete configuration
- [ ] `DesignerService.save_config()`, `DesignerService.load_config()`
- [ ] Integration tests in `apps/api/tests/test_designer.py`

---

### P4-3 · Cost Calculation API
**Branch:** `feature/p4-cost-api`

> **Depends on:** P2-9, P1-1 merged

- [ ] `POST /api/designer/cost` — accepts `PipelineConfigurationSchema`, returns `CostEstimateSchema`
- [ ] `apps/api/app/utils/cost_calculator.py` — full implementation:
  - Embedding cost: `tokens * price_per_token[model]`
  - Vector storage cost: `vectors * dims * storage_price[provider]`
  - Retrieval cost: `queries * retrieval_price[provider]`
  - Reranking cost: `queries * top_k * rerank_price[model]` (if enabled)
  - Generation cost: `(input_tokens + output_tokens) * price[model]`
  - Returns per-query and per-month estimates + breakdown array
- [ ] `CostService` class wrapping the calculator
- [ ] Unit tests with known pricing fixtures

---

### P4-4 · Export API
**Branch:** `feature/p4-export-api`

> **Depends on:** P4-2, P3-5 merged (generators can be ported or called from frontend)

Create `apps/api/app/routers/designer.py` export endpoint and `apps/api/app/services/export_service.py`:

- [ ] `POST /api/designer/export` — accepts config + format (`python` | `yaml` | `terraform` | `docker-compose` | `k8s`)
- [ ] `ExportService`:
  - `to_python(config) -> str` — LangChain LCEL pipeline code
  - `to_yaml(config) -> str` — configuration YAML
  - `to_terraform(config) -> str` — cloud infra (AWS/GCP/Azure)
  - `to_docker_compose(config) -> str`
  - `to_kubernetes(config) -> str`
- [ ] Returns file content + suggested filename
- [ ] Integration tests for each format

---

### P4-5 · Templates API
**Branch:** `feature/p4-templates-api`

> **Depends on:** P1-1, P4-2 merged

Create `apps/api/app/routers/templates.py` and `apps/api/app/services/template_service.py`:

- [ ] `GET /api/templates` — list all templates from `data/templates.json`
- [ ] `GET /api/templates/{id}` — get single template
- [ ] `POST /api/templates/{id}/apply` — creates a new `PipelineConfig` from template for a project
- [ ] `TemplateService.list_templates()`, `TemplateService.apply(template_id, project_id) -> PipelineConfig`
- [ ] Integration tests

---

## Phase 5 — Designer Mode Frontend
> **Goal:** Complete step-by-step visual pipeline builder UI.

---

### P5-1 · Designer Layout & Stage Navigator
**Branch:** `feature/p5-designer-layout`

> **Depends on:** P3-3, P3-2 merged

- [ ] `apps/web/src/app/designer/layout.tsx` — two-column layout: left pipeline builder, right live preview panel
- [ ] `apps/web/src/app/designer/page.tsx` — designer entry (redirects to cloud provider step)
- [ ] `apps/web/src/components/designer/StageNavigator.tsx`
  - 12-step progress bar (Cloud → Ingestion → Chunking → Embedding → VectorStore → Retrieval → Reranking → Generation → Routing → Memory → Evaluation → Review)
  - Completed stages shown with green check
  - Current stage highlighted
  - Click to navigate to any completed or adjacent stage
- [ ] Dynamic routing via `apps/web/src/app/designer/[step]/page.tsx`

---

### P5-2 · Cloud Provider Selector
**Branch:** `feature/p5-cloud-provider-selector`

> **Depends on:** P5-1, P1-1 merged

- [ ] `apps/web/src/components/designer/CloudProviderSelector.tsx`
  - 4-column card grid (AWS, GCP, Azure, Multi-cloud)
  - Each card: icon, name, description, "Best For" badges, strength bullet points
  - Selected card: primary border + ring highlight
  - Reads from `data/cloud-providers.json`
- [ ] `apps/web/src/app/designer/[step]/page.tsx` — renders correct component per step

---

### P5-3 · Data Ingestion Configuration
**Branch:** `feature/p5-ingestion-config`

> **Depends on:** P5-1 merged

- [ ] `apps/web/src/components/designer/DataIngestionConfig.tsx`
  - Source type selector: File Upload, S3/GCS/Azure Blob, URL, Database, API
  - File type checklist: PDF, DOCX, TXT, Markdown, HTML, CSV, JSON
  - Preprocessing options: strip HTML, normalize whitespace, extract metadata
  - Metadata fields toggle: source, page number, section, custom key-value

---

### P5-4 · Chunking Configuration
**Branch:** `feature/p5-chunking-config`

> **Depends on:** P5-1, P1-1 merged

- [ ] `apps/web/src/components/designer/ChunkingConfig.tsx`
  - Strategy dropdown (reads `data/chunking-strategies.json`)
  - Chunk Size slider (128–2048, step 128)
  - Chunk Overlap slider (0–200, step 10)
  - Right sidebar: pros/cons tabs for selected strategy
  - Recommendation alert: "Best for: ..." based on selected strategy
  - Default config auto-fill when strategy changes

---

### P5-5 · Embedding Model Selector
**Branch:** `feature/p5-embedding-selector`

> **Depends on:** P5-1, P1-1 merged

- [ ] `apps/web/src/components/designer/EmbeddingSelector.tsx`
  - Model card grid (reads `data/models/embeddings.json`)
  - Each card: model name, provider logo, dimensions, max tokens, cost badge, speed/quality bars
  - Filter bar: by provider, tier (fast/balanced/advanced), open-source only toggle
  - Comparison table view toggle
  - Selected model: highlighted card

---

### P5-6 · Vector Store Selector
**Branch:** `feature/p5-vectorstore-selector`

> **Depends on:** P5-1, P1-1 merged

- [ ] `apps/web/src/components/designer/VectorStoreSelector.tsx`
  - Card grid (reads `data/vector-stores.json`)
  - Each card: name, type badge (managed/self-hosted/embedded), cloud-native indicator, best-for tags
  - Filter: by type, cloud provider compatibility
  - Configuration panel appears on selection: index name, metric (cosine/euclidean/dot), namespace

---

### P5-7 · Retrieval Configuration
**Branch:** `feature/p5-retrieval-config`

> **Depends on:** P5-1, P1-1 merged

- [ ] `apps/web/src/components/designer/RetrievalConfig.tsx`
  - Strategy cards (reads `data/retrieval-strategies.json`)
  - Top-K slider (1–20)
  - Score threshold slider (0–1.0)
  - Hybrid search alpha slider (appears when `hybrid` selected)
  - Metadata filter builder: key, operator, value rows
- [ ] `apps/web/src/components/designer/RerankingSelector.tsx`
  - Enable/disable toggle
  - Reranker model dropdown (reads `data/models/rerankers.json`)
  - Top-N slider

---

### P5-8 · Generation Model Selector
**Branch:** `feature/p5-generation-selector`

> **Depends on:** P5-1, P1-1 merged

- [ ] `apps/web/src/components/designer/GenerationModelSelector.tsx`
  - Model cards (reads `data/models/generation.json`)
  - Filter: provider, context window size, cost tier
  - Temperature slider (0–2.0)
  - Max tokens slider (256–8192)
  - System prompt textarea with character count
  - Output format selector: text, JSON, markdown

---

### P5-9 · Routing, Memory & Evaluation Config
**Branch:** `feature/p5-routing-memory-eval`

> **Depends on:** P5-1 merged

- [ ] `apps/web/src/components/designer/RoutingLogicBuilder.tsx`
  - Enable/disable routing toggle
  - Complexity detection: keyword list, query length threshold
  - Route map: simple query → cheap model, complex query → powerful model
- [ ] `apps/web/src/components/designer/MemoryConfig.tsx`
  - Memory type: none, conversation buffer, summary buffer, vector memory
  - Window size slider (for buffer memory)
  - Session persistence toggle
- [ ] `apps/web/src/components/designer/EvaluationConfig.tsx`
  - Metrics checklist: faithfulness, answer_relevance, context_precision, context_recall, latency
  - Test set size input
  - Evaluation schedule: on-demand / continuous

---

### P5-10 · Pipeline Visualizer
**Branch:** `feature/p5-pipeline-visualizer`

> **Depends on:** P3-5, P5-1 merged

- [ ] `apps/web/src/components/shared/PipelineVisualizer.tsx`
  - Renders Mermaid flowchart from `generateMermaidDiagram(config.stages, cloudProvider)`
  - Updates live on every stage change
  - Download as PNG button
  - Fullscreen/modal view
  - Pipeline summary table below diagram

---

### P5-11 · Cost Estimator Component
**Branch:** `feature/p5-cost-estimator`

> **Depends on:** P5-10, P4-3 merged

- [ ] `apps/web/src/components/shared/CostEstimator.tsx`
  - Fetches from `POST /api/designer/cost` on config change (debounced 500ms)
  - Shows: cost per 1K queries, cost per month (100K queries)
  - Stacked progress bars per component
  - Detailed breakdown table
  - Optimization tips alert section
  - Industry benchmark comparison

---

### P5-12 · Code Export Component
**Branch:** `feature/p5-code-exporter`

> **Depends on:** P4-4 merged, P3-1 merged

- [ ] `apps/web/src/components/shared/CodeExporter.tsx`
  - Tab strip: Python | YAML | Terraform | Docker Compose | Kubernetes
  - Calls `POST /api/designer/export` with selected format
  - Prism.js syntax-highlighted code block
  - Copy to clipboard button
  - Download file button
  - "Deploy" button (posts to deployment API)

---

### P5-13 · Designer Review Page
**Branch:** `feature/p5-designer-review`

> **Depends on:** P5-10, P5-11, P5-12 merged

- [ ] `apps/web/src/app/designer/review/page.tsx`
  - Full pipeline summary (all stages configured)
  - PipelineVisualizer (Mermaid diagram)
  - CostEstimator with final breakdown
  - CodeExporter (all formats)
  - Action buttons: Save, Deploy, "Optimize This Configuration" (→ Autopilot)
  - "Source: autopilot" banner if config came from Autopilot explain-decisions flow
- [ ] `apps/web/src/components/shared/DeploymentPanel.tsx` — deployment target selector + deploy button

---

### P5-14 · Template Gallery Page
**Branch:** `feature/p5-template-gallery`

> **Depends on:** P4-5, P3-3 merged

- [ ] `apps/web/src/app/templates/page.tsx`
  - Calls `GET /api/templates`
  - Card grid: template name, use case, provider logos, complexity badge
  - "Use This Template" button → applies template and redirects to Designer review
- [ ] `apps/web/src/components/shared/ModelCard.tsx` — reusable card for model/template info

---

## Phase 6 — Autopilot Mode Backend Agents
> **Goal:** LangGraph agent system that autonomously builds RAG pipelines.

---

### P6-1 · LangGraph Agent Infrastructure
**Branch:** `feature/p6-langgraph-infrastructure`

> **Depends on:** P2-1 through P2-7 merged

Create `apps/api/app/agents/`:

- [ ] `utils/agent_state.py` — `AutopilotState` TypedDict (all fields from CLAUDE.md §Orchestrator)
- [ ] `utils/agent_tools.py` — LangChain tool wrappers: `load_documents`, `create_vector_index`, `call_llm`, `calculate_metrics`, `generate_config`
- [ ] `utils/prompts.py` — all agent system prompts (analyst, chunking optimizer, embedding tester, retrieval optimizer, evaluator, deployer)
- [ ] `apps/api/app/agents/__init__.py`
- [ ] Verify LangGraph imports cleanly

---

### P6-2 · Document Analyst Agent
**Branch:** `feature/p6-document-analyst-agent`

> **Depends on:** P6-1, P2-1 merged

- [ ] `apps/api/app/agents/document_analyst.py` — `DocumentAnalystAgent`:
  - `analyze(documents) -> dict`:
    - Token count per document (tiktoken)
    - Language detection
    - Structure analysis (has_code, has_markdown_headers, has_lists)
    - Average document length
    - Recommended chunking strategy + chunk size
    - Detected file types + content domain
- [ ] Unit tests in `apps/api/tests/test_agents.py::test_document_analyst`

---

### P6-3 · Chunking Optimizer Agent
**Branch:** `feature/p6-chunking-optimizer-agent`

> **Depends on:** P6-2, P2-2 merged

- [ ] `apps/api/app/agents/chunking_optimizer.py` — `ChunkingOptimizerAgent`:
  - `optimize(documents, analysis, requirements) -> ChunkingConfig`:
    - Tests top-3 strategies recommended by document analysis
    - Scores chunks: coherence (LLM judge), completeness, size distribution
    - Selects strategy + parameters with highest aggregate score
    - Returns `sample_chunks` for embedding benchmarking

---

### P6-4 · Embedding Tester Agent
**Branch:** `feature/p6-embedding-tester-agent`

> **Depends on:** P6-3, P2-3 merged

- [ ] `apps/api/app/agents/embedding_tester.py` — `EmbeddingTesterAgent`:
  - `benchmark(chunks, requirements) -> EmbeddingConfig`:
    - Selects 3–5 candidate models based on requirements (budget, latency)
    - Embeds sample chunks with each model
    - Scores: embedding quality (semantic similarity on known pairs), speed (ms/chunk), cost
    - Weights scores based on `optimize_for` flag
    - Returns best model config + full benchmark results

---

### P6-5 · Retrieval Optimizer Agent
**Branch:** `feature/p6-retrieval-optimizer-agent`

> **Depends on:** P6-4, P2-4, P2-5 merged

- [ ] `apps/api/app/agents/retrieval_optimizer.py` — `RetrievalOptimizerAgent`:
  - `optimize(vectorstore, requirements) -> RetrievalConfig`:
    - Builds vector index with best embedding model
    - Tests: similarity, MMR, hybrid (α=0.5), parent-child
    - Evaluates on 20 sample queries using precision@k
    - Tunes hybrid alpha (grid search: 0.3, 0.5, 0.7)
    - Decides on reranking (cost/benefit analysis)
    - Returns optimal strategy + top_k + reranking config

---

### P6-6 · Evaluation Agent
**Branch:** `feature/p6-evaluation-agent`

> **Depends on:** P6-5, P2-6, P2-7 merged

- [ ] `apps/api/app/agents/evaluation_agent.py` — `EvaluationAgent`:
  - `generate_test_set(documents, num_questions) -> List[dict]` — LLM generates diverse QA pairs
  - `evaluate(pipeline, test_set) -> EvaluationMetrics` — runs RAGAS on full pipeline
  - `analyze_failures(pipeline, test_set, metrics) -> FailureAnalysis` — categorises bad cases
  - Returns metric dict + failure categories + actionable recommendations

---

### P6-7 · Deployment Agent
**Branch:** `feature/p6-deployment-agent`

> **Depends on:** P6-6 merged

Create `apps/api/app/core/deployment/` and `apps/api/app/agents/deployment_agent.py`:

- [ ] `apps/api/app/core/deployment/packager.py` — assembles final RAG pipeline code from config
- [ ] `apps/api/app/core/deployment/docker_gen.py` — generates `Dockerfile` + `docker-compose.yml` for deployed pipeline
- [ ] `apps/api/app/core/deployment/k8s_gen.py` — generates K8s `Deployment`, `Service`, `ConfigMap` manifests
- [ ] `apps/api/app/core/deployment/deployers.py` — cloud deployers (AWS ECS, GCP Cloud Run, Azure Container Apps) — stub implementations with clear TODO markers
- [ ] `DeploymentAgent.deploy(config, cloud_provider) -> DeploymentInfo`

---

### P6-8 · Autopilot Orchestrator
**Branch:** `feature/p6-autopilot-orchestrator`

> **Depends on:** P6-2 through P6-7 merged

- [ ] `apps/api/app/agents/orchestrator.py` — `AutopilotOrchestrator`:
  - Builds LangGraph `StateGraph` with 7 nodes (analyze → chunking → embedding → retrieval → evaluate → decide → deploy)
  - Conditional edge `should_iterate`: checks metrics vs targets, increments iteration counter, routes to `optimize_chunking` or `deploy_system`
  - Persists progress to `AutopilotBuild` DB record after each node (for real-time frontend polling)
  - Emits progress events to Redis pub/sub channel `build:{build_id}`
- [ ] Integration tests in `apps/api/tests/test_agents.py::test_orchestrator_full_flow`

---

### P6-9 · Autopilot API Endpoints
**Branch:** `feature/p6-autopilot-api`

> **Depends on:** P6-8, P2-8, P4-1 merged

Create `apps/api/app/routers/autopilot.py` and `apps/api/app/services/autopilot_service.py`:

- [ ] `POST /api/autopilot/build` — validates request, creates `AutopilotBuild` record, dispatches `run_autopilot_build` Celery task, returns `build_id`
- [ ] `GET /api/autopilot/build/{id}` — returns current build status + stage statuses + messages
- [ ] `GET /api/autopilot/build/{id}/stream` — Server-Sent Events: subscribes to Redis pub/sub `build:{id}`, streams events to client
- [ ] `POST /api/autopilot/build/{id}/cancel` — sends revoke signal to Celery task
- [ ] `GET /api/autopilot/build/{id}/result` — returns full `BuildResult` once complete
- [ ] `AutopilotService` wiring build lifecycle
- [ ] Integration tests in `apps/api/tests/test_autopilot.py`

---

## Phase 7 — Autopilot Mode Frontend
> **Goal:** Real-time build monitoring UI with agent activity feed and results.

---

### P7-1 · Document Uploader
**Branch:** `feature/p7-document-uploader`

> **Depends on:** P3-1, P3-3 merged

- [ ] `apps/web/src/components/autopilot/DocumentUploader.tsx`
  - Drag-and-drop dropzone (react-dropzone)
  - Multi-file: PDF, DOCX, TXT, MD, HTML, CSV, JSON
  - File list with remove button, file size, type icon
  - S3/GCS/Azure Blob URL input tab
  - Upload progress bar
  - Calls file upload endpoint (multipart form to `POST /api/autopilot/upload`)
- [ ] `apps/api/app/routers/autopilot.py` — `POST /api/autopilot/upload` saves files to MinIO, returns document IDs

---

### P7-2 · Requirements Form
**Branch:** `feature/p7-requirements-form`

> **Depends on:** P3-1, P1-2 merged

- [ ] `apps/web/src/components/autopilot/RequirementsForm.tsx`
  - Target metrics sliders: faithfulness (0–1), answer relevance (0–1), context precision (0–1)
  - Optimization goal radio: Quality / Cost / Latency / Balanced
  - Cloud provider selector (optional, same component as Designer)
  - Budget constraint input (max $/1K queries)
  - Latency requirement input (max ms)
  - Max iterations slider (1–10)
  - Zod form validation with React Hook Form

---

### P7-3 · Build Progress Monitor
**Branch:** `feature/p7-build-progress`

> **Depends on:** P3-2, P3-1 merged

- [ ] `apps/web/src/components/autopilot/BuildProgress.tsx`
  - Overall progress bar (computed from completed stages)
  - Stage list: 7 stages each with status icon (pending/running/complete/failed), message, sub-progress bar
  - Iteration counter alert (shows if > 1 iteration)
  - Uses SSE (`EventSource`) to connect to `GET /api/autopilot/build/{id}/stream`
  - Falls back to polling `GET /api/autopilot/build/{id}` every 3s if SSE unsupported
- [ ] `apps/web/src/app/autopilot/build/[id]/page.tsx` — build page

---

### P7-4 · Agent Activity Feed
**Branch:** `feature/p7-agent-activity-feed`

> **Depends on:** P7-3 merged

- [ ] `apps/web/src/components/autopilot/AgentActivityFeed.tsx`
  - Scrolling log of `BuildMessage[]` from store
  - Each message: timestamp, emoji icon, text, type badge (info/success/warning/error)
  - Auto-scroll to latest
  - Filter by message type
  - Export log as TXT button

---

### P7-5 · Metrics Dashboard
**Branch:** `feature/p7-metrics-dashboard`

> **Depends on:** P7-3, P3-1 merged

- [ ] `apps/web/src/components/autopilot/MetricsDashboard.tsx`
  - Real-time metric cards: faithfulness, answer relevance, context precision, context recall
  - Line chart (Recharts) showing metric trends across iterations
  - Latency distribution bar chart
  - Cost per query estimate
  - Target vs actual metric comparison

---

### P7-6 · Decision Explainer & Results
**Branch:** `feature/p7-results-summary`

> **Depends on:** P7-5, P5-10 merged

- [ ] `apps/web/src/components/autopilot/DecisionExplainer.tsx`
  - Accordion for each agent decision: chunking, embedding, retrieval, generation
  - Each section: chosen value, reasoning text, alternatives tested table
  - "Explain in Designer" button → converts autopilot config to Designer format, loads it, redirects to `/designer/review?source=autopilot`
- [ ] `apps/web/src/components/autopilot/ResultsSummary.tsx`
  - Final metric scores with pass/fail vs targets
  - Selected configuration summary table
  - Deployment info card (endpoint URL, status)
  - Action buttons: Deploy, Download Code, Explain Decisions, Rebuild
- [ ] `apps/web/src/app/autopilot/results/[id]/page.tsx`

---

### P7-7 · Autopilot Entry & History Pages
**Branch:** `feature/p7-autopilot-pages`

> **Depends on:** P7-1, P7-2 merged

- [ ] `apps/web/src/app/autopilot/page.tsx` — entry: upload docs + fill requirements → "Build" button
- [ ] `apps/web/src/app/autopilot/layout.tsx` — layout with back navigation
- [ ] `apps/web/src/app/projects/page.tsx` — project list page
- [ ] `apps/web/src/app/projects/[id]/page.tsx` — project detail: tabs for Designer configs and Autopilot builds

---

## Phase 8 — Integration Layer (Bidirectional Flow)
> **Goal:** Wire Designer ↔ Autopilot handoff; both modes stay in sync.

---

### P8-1 · Designer → Autopilot Handoff
**Branch:** `feature/p8-designer-to-autopilot`

> **Depends on:** P5-13, P7-1, P7-2 merged

- [ ] `apps/web/src/components/shared/OptimizeButton.tsx`
  - Dialog explaining what Autopilot will do
  - Converts `PipelineConfiguration` → `BuildRequirements` (preserves cloud provider, uses current config as starting point)
  - Calls `POST /api/autopilot/build` with `base_config` attached
  - Redirects to `/autopilot/build/{buildId}`
- [ ] Update `apps/api/app/agents/orchestrator.py` to accept and honour `base_config` (skip stages already configured)
- [ ] Integration test: end-to-end Designer config → Autopilot build start

---

### P8-2 · Autopilot → Designer Visualization
**Branch:** `feature/p8-autopilot-to-designer`

> **Depends on:** P7-6, P5-13 merged

- [ ] `apps/web/src/components/autopilot/DecisionExplainer.tsx` "Explain in Designer" button:
  - `convertAutopilotConfigToDesigner(buildResult) -> PipelineConfiguration`
  - Calls `useDesignerStore().setConfig(designerConfig)`
  - Adds metadata: `source: 'autopilot'`, `buildId`
  - Navigates to `/designer/review?source=autopilot`
- [ ] `apps/web/src/app/designer/review/page.tsx` — shows "Imported from Autopilot" banner if `source=autopilot` in query string
- [ ] Integration test: Autopilot result → Designer review page loads with correct config

---

### P8-3 · Evaluation API Endpoints
**Branch:** `feature/p8-evaluation-api`

> **Depends on:** P2-7, P4-1 merged

Create `apps/api/app/routers/evaluation.py`:

- [ ] `POST /api/evaluation/run` — triggers evaluation Celery task for a config_id + optional test_set
- [ ] `GET /api/evaluation/run/{id}` — evaluation run status + metrics
- [ ] `GET /api/evaluation/runs?config_id=` — evaluation history for a config
- [ ] `POST /api/evaluation/compare` — A/B test: compares two config_ids, returns side-by-side metrics
- [ ] Integration tests

---

### P8-4 · Deployment API Endpoints
**Branch:** `feature/p8-deployment-api`

> **Depends on:** P6-7, P4-1 merged

Create `apps/api/app/routers/deployment.py`:

- [ ] `POST /api/deployment/deploy` — deploys config_id to specified provider
- [ ] `GET /api/deployment/{id}/status` — deployment status + endpoint URL
- [ ] `GET /api/deployment/deployments?project_id=` — list deployments
- [ ] `DELETE /api/deployment/{id}` — teardown deployment
- [ ] Integration tests

---

## Phase 9 — MLflow & Experiment Tracking
> **Goal:** Log all Autopilot experiments to MLflow for reproducibility.

---

### P9-1 · MLflow Integration
**Branch:** `feature/p9-mlflow-integration`

> **Depends on:** P6-8, P0-2 merged

- [ ] Configure MLflow tracking server URI from settings
- [ ] In `AutopilotOrchestrator`:
  - Create MLflow experiment per build
  - Log params: chunking strategy, chunk size, embedding model, retrieval strategy, top_k
  - Log metrics: faithfulness, answer_relevance, context_precision, latency
  - Log artifacts: generated code, config YAML
  - Tag runs with: build_id, iteration, cloud_provider
- [ ] `apps/api/app/utils/mlflow_tracker.py` — `MLflowTracker` context manager
- [ ] Verify MLflow UI at `localhost:5000` shows runs

---

## Phase 10 — Testing
> **Goal:** Comprehensive test coverage before launch.

---

### P10-1 · Backend Unit Tests
**Branch:** `feature/p10-backend-unit-tests`

> **Depends on:** P2-9 merged

- [ ] `apps/api/tests/conftest.py` — fixtures: async test client, test DB (in-memory SQLite), mock Redis, mock Qdrant
- [ ] `apps/api/tests/test_core/test_chunking.py` — all 7 strategies, edge cases (empty input, single sentence)
- [ ] `apps/api/tests/test_core/test_embedding.py` — mock OpenAI/Cohere, verify cache hit/miss
- [ ] `apps/api/tests/test_core/test_retrieval.py` — all 6 strategies with fixture vectors
- [ ] `apps/api/tests/test_core/test_evaluation.py` — mock RAGAS, verify metric extraction
- [ ] `apps/api/tests/test_utils/test_cost_calculator.py` — pricing accuracy for all model/provider combinations
- [ ] Target: ≥80% backend code coverage

---

### P10-2 · Backend Integration Tests
**Branch:** `feature/p10-backend-integration-tests`

> **Depends on:** P4-5, P6-9 merged, P10-1 merged

- [ ] `apps/api/tests/test_integration/test_designer_flow.py`
  - Create project → save config → calculate cost → export Python → save to DB → retrieve
- [ ] `apps/api/tests/test_integration/test_autopilot_flow.py`
  - Upload docs → start build → mock agents → check progress → retrieve result
- [ ] `apps/api/tests/test_integration/test_evaluation_flow.py`
  - Config → run evaluation → compare two configs
- [ ] Use `docker-compose.dev.yml` test profile with real PostgreSQL + Qdrant

---

### P10-3 · Frontend Unit Tests
**Branch:** `feature/p10-frontend-unit-tests`

> **Depends on:** P5-13, P7-6 merged

- [ ] Setup Vitest + React Testing Library
- [ ] `ChunkingConfig.test.tsx` — slider interactions, strategy selection, defaults
- [ ] `CostEstimator.test.tsx` — mock API, verify breakdown renders
- [ ] `PipelineVisualizer.test.tsx` — Mermaid renders, download button
- [ ] `BuildProgress.test.tsx` — stage status rendering, SSE mock
- [ ] Zustand store tests: designer state transitions, autopilot build lifecycle
- [ ] Code generator snapshot tests (Python, YAML, Terraform)
- [ ] Target: ≥75% component coverage

---

### P10-4 · End-to-End Tests
**Branch:** `feature/p10-e2e-tests`

> **Depends on:** P10-2, P10-3 merged

- [ ] Setup Playwright
- [ ] `e2e/designer-flow.spec.ts` — full 12-step pipeline build → export code
- [ ] `e2e/autopilot-flow.spec.ts` — upload docs → requirements → monitor build → view results
- [ ] `e2e/integration-flow.spec.ts` — Designer "Optimize This" → Autopilot → "Explain in Designer"
- [ ] `e2e/template-flow.spec.ts` — select FAQ chatbot template → verify pre-filled config

---

## Phase 11 — Monitoring & Observability
> **Goal:** Production-grade logging, metrics, and alerting.

---

### P11-1 · Structured Logging
**Branch:** `feature/p11-structured-logging`

> **Depends on:** P2-9 merged

- [ ] Configure `structlog` in `apps/api/app/utils/logger.py` with JSON processor
- [ ] Add request/response logging middleware to FastAPI (`X-Request-ID` header)
- [ ] Log all agent state transitions with `build_id`, `stage`, `iteration`, `duration_ms`
- [ ] Frontend: error boundary reports to `/api/logs/client-error`

---

### P11-2 · Prometheus Metrics
**Branch:** `feature/p11-prometheus-metrics`

> **Depends on:** P11-1 merged, P0-2 merged

- [ ] Add `prometheus-fastapi-instrumentator` to `requirements.txt`
- [ ] Expose `/metrics` endpoint in FastAPI
- [ ] Custom metrics:
  - `rag_build_duration_seconds` (histogram, labels: provider, optimize_for)
  - `rag_evaluation_score` (gauge, labels: metric_name, build_id)
  - `rag_api_cost_usd` (counter, labels: provider, model)
- [ ] Add Prometheus + Grafana services to `docker-compose.yml`
- [ ] Create `docker/grafana/dashboards/rag-studio.json` — default dashboard

---

### P11-3 · Cost & Usage Analytics
**Branch:** `feature/p11-usage-analytics`

> **Depends on:** P4-3, P6-9 merged

- [ ] Track per-build cost in `AutopilotBuild.result`
- [ ] `GET /api/analytics/costs?project_id=&from=&to=` — cost breakdown by time range
- [ ] `GET /api/analytics/usage` — builds count, success rate, avg duration
- [ ] Frontend analytics page at `apps/web/src/app/analytics/page.tsx` — Recharts cost trend chart

---

## Phase 12 — Production Hardening & Launch
> **Goal:** Security, performance, and final deployment.

---

### P12-1 · Authentication & Authorization
**Branch:** `feature/p12-auth`

> **Depends on:** P4-1 merged

- [ ] Add JWT-based authentication (using `python-jose` + `passlib`)
- [ ] `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/refresh`
- [ ] Protect all non-health endpoints with `Depends(get_current_user)`
- [ ] Frontend: login page, JWT stored in `httpOnly` cookie
- [ ] Row-level security: users can only access their own projects/builds

---

### P12-2 · Security Hardening
**Branch:** `feature/p12-security-hardening`

> **Depends on:** P12-1 merged

- [ ] Input sanitisation on all user-supplied strings (prevent prompt injection in system prompts)
- [ ] File upload validation: MIME type check, max size (50MB), virus scan stub
- [ ] Rate limiting: `slowapi` on `/api/autopilot/build` (5 builds/min per user)
- [ ] CORS lockdown: only allow `NEXT_PUBLIC_API_URL` origin in production
- [ ] Secrets: rotate all hardcoded example keys, use Vault/AWS Secrets Manager stubs
- [ ] Run `bandit` security scan on Python codebase; fix all HIGH findings

---

### P12-3 · Performance Optimisation
**Branch:** `feature/p12-performance`

> **Depends on:** P10-4 merged

- [ ] Embedding cache hit rate target: >80% (Redis TTL tuning)
- [ ] Database query optimisation: add missing indexes, explain-analyze slow queries
- [ ] Next.js bundle analysis: code-split designer and autopilot chunks
- [ ] API response compression (gzip middleware)
- [ ] Qdrant: tune HNSW `ef_construct` and `m` parameters for recall/speed tradeoff
- [ ] Load test with k6: 100 concurrent users, verify p95 API latency < 500ms

---

### P12-4 · Kubernetes Production Manifests
**Branch:** `feature/p12-kubernetes-manifests`

> **Depends on:** P0-2 merged

Create all files in `k8s/`:

- [ ] `namespace.yaml`
- [ ] `web-deployment.yaml` (3 replicas, HPA)
- [ ] `api-deployment.yaml` (3 replicas, HPA)
- [ ] `worker-deployment.yaml` (2 replicas)
- [ ] `postgres-deployment.yaml` + PVC
- [ ] `redis-deployment.yaml`
- [ ] `qdrant-deployment.yaml` + PVC
- [ ] `ingress.yaml` (Traefik, TLS)
- [ ] `configmaps.yaml`
- [ ] `secrets.yaml` (sealed-secrets template, no real values)
- [ ] Verify `kubectl apply -f k8s/` deploys cleanly to local Kind cluster

---

### P12-5 · Final Documentation Pass
**Branch:** `feature/p12-documentation`

> **Depends on:** All Phase 1–11 features merged

- [ ] `docs/getting-started/installation.md` — prerequisites, Docker setup, first run
- [ ] `docs/getting-started/quickstart-designer.md` — build an FAQ chatbot in 15 minutes
- [ ] `docs/getting-started/quickstart-autopilot.md` — upload docs and deploy in 10 minutes
- [ ] `docs/guides/designer-mode/pipeline-building.md`
- [ ] `docs/guides/designer-mode/cost-optimization.md`
- [ ] `docs/guides/designer-mode/code-export.md`
- [ ] `docs/guides/autopilot-mode/build-configuration.md`
- [ ] `docs/guides/autopilot-mode/evaluation-metrics.md`
- [ ] `docs/guides/integration/designer-to-autopilot.md`
- [ ] `docs/api-reference/` — auto-generated from FastAPI OpenAPI schema
- [ ] Update `README.md` with badges, screenshots, demo GIF

---

### P12-6 · Production Deployment & Launch
**Branch:** `feature/p12-production-launch`

> **Depends on:** P12-1 through P12-5 all merged to `develop`, `develop` → `main` PR approved

- [ ] Tag release `v1.0.0-rc1` on `develop`
- [ ] Build and push Docker images to GHCR: `web:1.0.0`, `api:1.0.0`, `worker:1.0.0`
- [ ] Deploy to staging Kubernetes cluster; run E2E test suite against staging
- [ ] Fix any staging regressions
- [ ] Merge `develop` → `main`; tag `v1.0.0`
- [ ] Deploy to production Kubernetes cluster
- [ ] Verify all `/health` endpoints return 200 in production
- [ ] Publish GitHub release notes (generated from `CHANGELOG.md`)

---

## Quick Reference — Branch Checklist

```
develop
├── feature/p0-monorepo-skeleton
├── feature/p0-docker-compose-dev
├── feature/p0-cicd-pipelines
├── feature/p0-backend-scaffold
├── feature/p0-frontend-scaffold
│
├── feature/p1-json-model-catalogs
├── feature/p1-typescript-types
├── feature/p1-python-schemas
├── feature/p1-database-schema
│
├── feature/p2-ingestion-service
├── feature/p2-chunking-service
├── feature/p2-embedding-service
├── feature/p2-vectorstore-service
├── feature/p2-retrieval-service
├── feature/p2-generation-service
├── feature/p2-evaluation-engine
├── feature/p2-celery-worker
├── feature/p2-health-endpoints
│
├── feature/p3-shadcn-components
├── feature/p3-zustand-stores
├── feature/p3-app-layout
├── feature/p3-landing-page
├── feature/p3-lib-utilities
│
├── feature/p4-projects-api
├── feature/p4-designer-config-api
├── feature/p4-cost-api
├── feature/p4-export-api
├── feature/p4-templates-api
│
├── feature/p5-designer-layout
├── feature/p5-cloud-provider-selector
├── feature/p5-ingestion-config
├── feature/p5-chunking-config
├── feature/p5-embedding-selector
├── feature/p5-vectorstore-selector
├── feature/p5-retrieval-config
├── feature/p5-generation-selector
├── feature/p5-routing-memory-eval
├── feature/p5-pipeline-visualizer
├── feature/p5-cost-estimator
├── feature/p5-code-exporter
├── feature/p5-designer-review
├── feature/p5-template-gallery
│
├── feature/p6-langgraph-infrastructure
├── feature/p6-document-analyst-agent
├── feature/p6-chunking-optimizer-agent
├── feature/p6-embedding-tester-agent
├── feature/p6-retrieval-optimizer-agent
├── feature/p6-evaluation-agent
├── feature/p6-deployment-agent
├── feature/p6-autopilot-orchestrator
├── feature/p6-autopilot-api
│
├── feature/p7-document-uploader
├── feature/p7-requirements-form
├── feature/p7-build-progress
├── feature/p7-agent-activity-feed
├── feature/p7-metrics-dashboard
├── feature/p7-results-summary
├── feature/p7-autopilot-pages
│
├── feature/p8-designer-to-autopilot
├── feature/p8-autopilot-to-designer
├── feature/p8-evaluation-api
├── feature/p8-deployment-api
│
├── feature/p9-mlflow-integration
│
├── feature/p10-backend-unit-tests
├── feature/p10-backend-integration-tests
├── feature/p10-frontend-unit-tests
├── feature/p10-e2e-tests
│
├── feature/p11-structured-logging
├── feature/p11-prometheus-metrics
├── feature/p11-usage-analytics
│
├── feature/p12-auth
├── feature/p12-security-hardening
├── feature/p12-performance
├── feature/p12-kubernetes-manifests
├── feature/p12-documentation
└── feature/p12-production-launch
```

---

## Dependency Map (Critical Path)

```
P0-1 → P0-2, P0-3, P0-4, P0-5
P0-1 → P1-1
P1-1 → P1-2 (needs P0-5), P1-3 (needs P0-4)
P1-3 → P1-4 (needs P0-4)
P1-4 → P2-1 → P2-2 → P2-3 → P2-4 → P2-5 → P2-6 → P2-7
P2-7 → P2-8, P6-1
P0-5 → P3-1 → P3-2 (needs P1-2) → P3-3 → P3-4
P2-9 → P4-1 → P4-2 → P4-3, P4-4, P4-5
P3-3, P3-2 → P5-1 → P5-2..P5-14 (in parallel)
P6-1 → P6-2 → P6-3 → P6-4 → P6-5 → P6-6 → P6-7 → P6-8 → P6-9
P5-13 + P7-6 → P8-1, P8-2
Everything → P10-* → P11-* → P12-*
```

> **Tip for Claude:** When starting a new task, first read the `feature/` branch name, check all listed "Depends on" branches are merged to `develop`, then implement exactly the sub-tasks listed. Do not implement work from other tasks — keep each branch focused.
