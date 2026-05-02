# RAG Studio - Complete Implementation Specification

> **The Complete RAG Platform: Design Manually or Build Automatically**

**Version:** 1.0.0 MVP  
**Project Type:** Unified RAG Development Platform  
**Architecture:** Monorepo with Next.js Frontend + FastAPI Backend + LangGraph Agents  
**Deployment:** Docker Compose (local) / Kubernetes (production)  

---

## 📖 Table of Contents

1. [Executive Summary](#executive-summary)
2. [Project Architecture](#project-architecture)
3. [Technical Stack](#technical-stack)
4. [Project Structure](#project-structure)
5. [Core Features](#core-features)
6. [Designer Mode Implementation](#designer-mode-implementation)
7. [Autopilot Mode Implementation](#autopilot-mode-implementation)
8. [Shared Core Services](#shared-core-services)
9. [Data Models & Types](#data-models--types)
10. [Database Schema](#database-schema)
11. [API Specification](#api-specification)
12. [UI/UX Components](#uiux-components)
13. [Agent System Architecture](#agent-system-architecture)
14. [Integration Layer](#integration-layer)
15. [Deployment Guide](#deployment-guide)
16. [Development Roadmap](#development-roadmap)
17. [Testing Strategy](#testing-strategy)
18. [Success Metrics](#success-metrics)

---

## 🎯 Executive Summary

### What is RAG Studio?

RAG Studio is a **unified platform** that provides two complementary modes for building production-ready Retrieval-Augmented Generation (RAG) systems:

1. **Designer Mode (RAGForge)** - Interactive visual builder for manual configuration
2. **Autopilot Mode (AutoRAG Architect)** - Autonomous AI agent that builds and optimizes automatically

### Core Value Proposition

**"Build RAG systems your way: Design step-by-step or let AI build automatically"**

- 🎨 **Learn by Doing** - Designer Mode teaches RAG concepts through guided building
- 🤖 **Deploy in Minutes** - Autopilot Mode automates the entire pipeline
- 🔄 **Best of Both Worlds** - Seamlessly switch between manual control and automation
- 📊 **Production-Ready** - Both modes generate deployable, optimized RAG systems

### Target Users

- **ML Engineers** - Learning RAG or designing custom architectures
- **AI Teams** - Building knowledge bases, chatbots, documentation assistants
- **Startups** - Need rapid prototyping and deployment
- **Enterprises** - Require compliance, control, and optimization

### Key Differentiator

Unlike LangChain/LlamaIndex (building blocks) or other RAG builders (manual only), RAG Studio is the **only platform** offering both guided manual design and autonomous optimization in one unified experience.

---

## 🏗️ Project Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER INTERFACE LAYER                        │
│                                                                     │
│  ┌──────────────────────────┐    ┌──────────────────────────────┐  │
│  │    DESIGNER MODE UI      │    │     AUTOPILOT MODE UI        │  │
│  │    (Next.js Pages)       │    │     (Next.js Pages)          │  │
│  │                          │    │                              │  │
│  │  • Step-by-step builder  │    │  • Document upload           │  │
│  │  • Visual pipeline       │    │  • Progress monitoring       │  │
│  │  • Cost calculator       │    │  • Results visualization     │  │
│  │  • Code export           │    │  • Decision explanation      │  │
│  └────────────┬─────────────┘    └──────────────┬───────────────┘  │
│               │                                  │                  │
│               └──────────────┬───────────────────┘                  │
└───────────────────────────────┼──────────────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │   Next.js API Routes  │
                    │   (API Gateway)       │
                    └───────────┬───────────┘
                                │
┌───────────────────────────────┼──────────────────────────────────────┐
│                       BACKEND API LAYER (FastAPI)                    │
│                                                                     │
│  ┌──────────────────────┐         ┌─────────────────────────────┐  │
│  │  Designer Endpoints  │         │  Autopilot Orchestrator     │  │
│  │                      │         │  (LangGraph Agents)         │  │
│  │  • /designer/config  │         │                             │  │
│  │  • /designer/export  │         │  • Document Analyst Agent   │  │
│  │  • /designer/cost    │         │  • Chunking Optimizer Agent │  │
│  │  • /designer/deploy  │         │  • Embedding Tester Agent   │  │
│  └──────────┬───────────┘         │  • Retrieval Optimizer      │  │
│             │                     │  • Evaluation Agent         │  │
│             │                     │  • Deployment Agent         │  │
│             │                     └──────────────┬──────────────┘  │
└─────────────┼────────────────────────────────────┼──────────────────┘
              │                                    │
              └────────────┬───────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────────┐
│                    SHARED CORE SERVICES                              │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  Ingestion   │  │  Chunking    │  │  Embedding   │              │
│  │  Service     │  │  Service     │  │  Service     │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  Vector DB   │  │  Retrieval   │  │  Generation  │              │
│  │  Service     │  │  Service     │  │  Service     │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  Evaluation  │  │  Deployment  │  │  Monitoring  │              │
│  │  Engine      │  │  Engine      │  │  Service     │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────────┐
│                       DATA PERSISTENCE LAYER                         │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  PostgreSQL  │  │  Redis       │  │  Vector DB   │              │
│  │  (Metadata)  │  │  (Cache)     │  │  (Qdrant)    │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  MLflow      │  │  S3/MinIO    │  │  Celery      │              │
│  │  (Tracking)  │  │  (Documents) │  │  (Jobs)      │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└──────────────────────────────────────────────────────────────────────┘
```

### Mode Interaction Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    MODE INTERACTIONS                        │
│                                                             │
│  Designer Mode                      Autopilot Mode          │
│  ─────────────                      ──────────────          │
│                                                             │
│  User builds       ──────────────>  "Optimize This"        │
│  manually                          Takes config as          │
│                                    starting point           │
│                                                             │
│  "Explain          <──────────────  AI builds              │
│  Decisions"                         automatically           │
│  Shows visual                                               │
│  pipeline                                                   │
│                                                             │
│  User tweaks       <──────────────> Shared config format   │
│  & re-deploys                       Both modes stay synced │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 💻 Technical Stack

### Frontend

```yaml
Framework: Next.js 14+ (App Router)
Language: TypeScript 5.0+
Styling: Tailwind CSS 3.4+
Components: shadcn/ui (Radix UI primitives)
State: Zustand 4.5+
Forms: React Hook Form + Zod
Diagrams: Mermaid.js 10+
Charts: Recharts
Code Highlighting: Prism.js
Icons: Lucide React
Animations: Framer Motion 11+
```

### Backend

```yaml
Framework: FastAPI 0.110+
Language: Python 3.11+
Package Manager: uv
Agent Framework: LangGraph 0.2+
LLM Integration: LangChain 0.1+
Async Runtime: Asyncio + Uvicorn
Task Queue: Celery 5.3+
Message Broker: Redis 7.2+
```

### Data & Storage

```yaml
Primary Database: PostgreSQL 16
Cache: Redis 7.2
Vector Database: Qdrant 1.8+ (primary), Pinecone (optional)
Object Storage: MinIO (S3-compatible)
Experiment Tracking: MLflow 2.11+
```

### RAG Components

```yaml
Embeddings:
  - sentence-transformers (open-source)
  - OpenAI API (text-embedding-3-*)
  - Cohere API (embed-*)
  - Google VertexAI

LLMs:
  - OpenAI (GPT-4o, GPT-4o-mini)
  - Anthropic (Claude Sonnet/Opus 4)
  - Google (Gemini 1.5 Pro/Flash)
  - Local (Ollama/LMStudio)

Evaluation:
  - RAGAS (RAG evaluation framework)
  - Custom metrics (faithfulness, relevance, etc.)

Vector Stores:
  - Qdrant (primary, embedded mode)
  - Pinecone (optional, managed)
  - Weaviate (optional, self-hosted)
  - FAISS (optional, local)
```

### DevOps & Infrastructure

```yaml
Containerization: Docker + Docker Compose
Orchestration: Kubernetes (production)
CI/CD: GitHub Actions
Monitoring: Prometheus + Grafana
Logging: ELK Stack (Elasticsearch, Logstash, Kibana)
API Gateway: Traefik
```

---

## 📁 Project Structure

```
rag-studio/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── cd.yml
│       └── tests.yml
├── apps/
│   ├── web/                              # Next.js Frontend
│   │   ├── public/
│   │   │   ├── logos/
│   │   │   │   ├── aws.svg
│   │   │   │   ├── gcp.svg
│   │   │   │   ├── azure.svg
│   │   │   │   └── multi-cloud.svg
│   │   │   ├── favicon.ico
│   │   │   └── og-image.png
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx              # Landing page
│   │   │   │   ├── globals.css
│   │   │   │   ├── designer/
│   │   │   │   │   ├── page.tsx          # Designer mode entry
│   │   │   │   │   ├── layout.tsx
│   │   │   │   │   └── [step]/
│   │   │   │   │       └── page.tsx      # Each pipeline step
│   │   │   │   ├── autopilot/
│   │   │   │   │   ├── page.tsx          # Autopilot mode entry
│   │   │   │   │   ├── build/
│   │   │   │   │   │   └── [id]/page.tsx # Build progress
│   │   │   │   │   └── results/
│   │   │   │   │       └── [id]/page.tsx # Build results
│   │   │   │   ├── projects/
│   │   │   │   │   ├── page.tsx          # Project list
│   │   │   │   │   └── [id]/page.tsx     # Project detail
│   │   │   │   ├── templates/
│   │   │   │   │   └── page.tsx          # Template gallery
│   │   │   │   └── api/
│   │   │   │       ├── designer/
│   │   │   │       │   ├── config/route.ts
│   │   │   │       │   ├── export/route.ts
│   │   │   │       │   └── cost/route.ts
│   │   │   │       └── autopilot/
│   │   │   │           ├── build/route.ts
│   │   │   │           └── status/route.ts
│   │   │   ├── components/
│   │   │   │   ├── ui/                   # shadcn/ui components
│   │   │   │   │   ├── button.tsx
│   │   │   │   │   ├── card.tsx
│   │   │   │   │   ├── select.tsx
│   │   │   │   │   ├── slider.tsx
│   │   │   │   │   ├── tabs.tsx
│   │   │   │   │   ├── badge.tsx
│   │   │   │   │   ├── dialog.tsx
│   │   │   │   │   ├── tooltip.tsx
│   │   │   │   │   ├── separator.tsx
│   │   │   │   │   ├── accordion.tsx
│   │   │   │   │   ├── alert.tsx
│   │   │   │   │   ├── progress.tsx
│   │   │   │   │   ├── table.tsx
│   │   │   │   │   └── textarea.tsx
│   │   │   │   ├── designer/
│   │   │   │   │   ├── CloudProviderSelector.tsx
│   │   │   │   │   ├── StageNavigator.tsx
│   │   │   │   │   ├── DataIngestionConfig.tsx
│   │   │   │   │   ├── ChunkingConfig.tsx
│   │   │   │   │   ├── EmbeddingSelector.tsx
│   │   │   │   │   ├── VectorStoreSelector.tsx
│   │   │   │   │   ├── RetrievalConfig.tsx
│   │   │   │   │   ├── RerankingSelector.tsx
│   │   │   │   │   ├── GenerationModelSelector.tsx
│   │   │   │   │   ├── RoutingLogicBuilder.tsx
│   │   │   │   │   ├── MemoryConfig.tsx
│   │   │   │   │   └── EvaluationConfig.tsx
│   │   │   │   ├── autopilot/
│   │   │   │   │   ├── DocumentUploader.tsx
│   │   │   │   │   ├── RequirementsForm.tsx
│   │   │   │   │   ├── BuildProgress.tsx
│   │   │   │   │   ├── AgentActivityFeed.tsx
│   │   │   │   │   ├── MetricsDashboard.tsx
│   │   │   │   │   ├── DecisionExplainer.tsx
│   │   │   │   │   └── ResultsSummary.tsx
│   │   │   │   ├── shared/
│   │   │   │   │   ├── PipelineVisualizer.tsx
│   │   │   │   │   ├── CostEstimator.tsx
│   │   │   │   │   ├── PerformanceMetrics.tsx
│   │   │   │   │   ├── CodeExporter.tsx
│   │   │   │   │   ├── DeploymentPanel.tsx
│   │   │   │   │   ├── ComparisonTable.tsx
│   │   │   │   │   ├── ModelCard.tsx
│   │   │   │   │   ├── InfoTooltip.tsx
│   │   │   │   │   └── LoadingSpinner.tsx
│   │   │   │   └── landing/
│   │   │   │       ├── Hero.tsx
│   │   │   │       ├── Features.tsx
│   │   │   │       ├── ModeComparison.tsx
│   │   │   │       ├── HowItWorks.tsx
│   │   │   │       ├── UseCases.tsx
│   │   │   │       ├── Pricing.tsx
│   │   │   │       └── CTA.tsx
│   │   │   ├── lib/
│   │   │   │   ├── utils.ts
│   │   │   │   ├── constants.ts
│   │   │   │   ├── validators.ts
│   │   │   │   ├── api-client.ts
│   │   │   │   └── generators/
│   │   │   │       ├── pythonCodeGenerator.ts
│   │   │   │       ├── yamlGenerator.ts
│   │   │   │       ├── terraformGenerator.ts
│   │   │   │       └── mermaidGenerator.ts
│   │   │   ├── types/
│   │   │   │   ├── pipeline.ts
│   │   │   │   ├── models.ts
│   │   │   │   ├── cloud.ts
│   │   │   │   ├── autopilot.ts
│   │   │   │   └── index.ts
│   │   │   └── store/
│   │   │       ├── designerStore.ts       # Designer mode state
│   │   │       ├── autopilotStore.ts      # Autopilot mode state
│   │   │       └── projectStore.ts        # Project management
│   │   ├── package.json
│   │   ├── next.config.js
│   │   ├── tailwind.config.ts
│   │   ├── tsconfig.json
│   │   └── components.json
│   │
│   └── api/                              # FastAPI Backend
│       ├── app/
│       │   ├── __init__.py
│       │   ├── main.py                   # FastAPI app entry
│       │   ├── config.py                 # Configuration
│       │   ├── dependencies.py           # DI dependencies
│       │   ├── routers/
│       │   │   ├── __init__.py
│       │   │   ├── designer.py           # Designer endpoints
│       │   │   ├── autopilot.py          # Autopilot endpoints
│       │   │   ├── projects.py           # Project CRUD
│       │   │   ├── templates.py          # Template management
│       │   │   ├── evaluation.py         # Evaluation endpoints
│       │   │   ├── deployment.py         # Deployment endpoints
│       │   │   └── health.py             # Health checks
│       │   ├── agents/                   # LangGraph Agents
│       │   │   ├── __init__.py
│       │   │   ├── orchestrator.py       # Main agent coordinator
│       │   │   ├── document_analyst.py   # Document analysis agent
│       │   │   ├── chunking_optimizer.py # Chunking strategy agent
│       │   │   ├── embedding_tester.py   # Embedding benchmark agent
│       │   │   ├── retrieval_optimizer.py# Retrieval optimization agent
│       │   │   ├── evaluation_agent.py   # Evaluation & metrics agent
│       │   │   ├── deployment_agent.py   # Deployment automation agent
│       │   │   └── utils/
│       │   │       ├── agent_state.py
│       │   │       ├── agent_tools.py
│       │   │       └── prompts.py
│       │   ├── core/                     # Core RAG Services
│       │   │   ├── __init__.py
│       │   │   ├── ingestion/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── loaders.py        # Document loaders
│       │   │   │   ├── preprocessors.py  # Text cleaning
│       │   │   │   └── extractors.py     # Metadata extraction
│       │   │   ├── chunking/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── strategies.py     # Chunking strategies
│       │   │   │   ├── semantic.py       # Semantic chunking
│       │   │   │   ├── recursive.py      # Recursive splitting
│       │   │   │   └── optimizers.py     # Chunk size optimization
│       │   │   ├── embedding/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── models.py         # Embedding model wrappers
│       │   │   │   ├── benchmarker.py    # Model benchmarking
│       │   │   │   └── cache.py          # Embedding cache
│       │   │   ├── vectorstore/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── qdrant_client.py  # Qdrant integration
│       │   │   │   ├── pinecone_client.py# Pinecone integration
│       │   │   │   ├── weaviate_client.py# Weaviate integration
│       │   │   │   └── factory.py        # VectorDB factory
│       │   │   ├── retrieval/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── strategies.py     # Retrieval strategies
│       │   │   │   ├── rerankers.py      # Reranking models
│       │   │   │   ├── hybrid.py         # Hybrid search
│       │   │   │   └── mmr.py            # MMR implementation
│       │   │   ├── generation/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── llm_providers.py  # LLM integrations
│       │   │   │   ├── prompts.py        # Prompt templates
│       │   │   │   └── chains.py         # RAG chains
│       │   │   ├── evaluation/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── ragas_eval.py     # RAGAS integration
│       │   │   │   ├── metrics.py        # Custom metrics
│       │   │   │   ├── test_sets.py      # Test set generation
│       │   │   │   └── analyzers.py      # Failure analysis
│       │   │   └── deployment/
│       │   │       ├── __init__.py
│       │   │       ├── packager.py       # Pipeline packaging
│       │   │       ├── docker_gen.py     # Dockerfile generation
│       │   │       ├── k8s_gen.py        # K8s manifests
│       │   │       └── deployers.py      # Cloud deployers
│       │   ├── models/                   # Database models
│       │   │   ├── __init__.py
│       │   │   ├── project.py
│       │   │   ├── pipeline_config.py
│       │   │   ├── build_history.py
│       │   │   ├── evaluation_run.py
│       │   │   └── deployment.py
│       │   ├── schemas/                  # Pydantic schemas
│       │   │   ├── __init__.py
│       │   │   ├── designer.py
│       │   │   ├── autopilot.py
│       │   │   ├── pipeline.py
│       │   │   ├── evaluation.py
│       │   │   └── deployment.py
│       │   ├── services/                 # Business logic
│       │   │   ├── __init__.py
│       │   │   ├── designer_service.py
│       │   │   ├── autopilot_service.py
│       │   │   ├── project_service.py
│       │   │   ├── cost_service.py
│       │   │   └── export_service.py
│       │   ├── utils/
│       │   │   ├── __init__.py
│       │   │   ├── logger.py
│       │   │   ├── validators.py
│       │   │   ├── cost_calculator.py
│       │   │   └── helpers.py
│       │   └── worker/                   # Celery workers
│       │       ├── __init__.py
│       │       ├── celery_app.py
│       │       ├── tasks.py
│       │       └── schedules.py
│       ├── tests/
│       │   ├── __init__.py
│       │   ├── conftest.py
│       │   ├── test_designer.py
│       │   ├── test_autopilot.py
│       │   ├── test_agents.py
│       │   ├── test_core/
│       │   │   ├── test_chunking.py
│       │   │   ├── test_embedding.py
│       │   │   ├── test_retrieval.py
│       │   │   └── test_evaluation.py
│       │   └── test_integration/
│       │       ├── test_designer_flow.py
│       │       └── test_autopilot_flow.py
│       ├── alembic/                      # Database migrations
│       │   ├── versions/
│       │   └── env.py
│       ├── requirements.txt
│       ├── requirements-dev.txt
│       ├── Dockerfile
│       └── pyproject.toml
│
├── data/                                 # Shared model catalogs
│   ├── models/
│   │   ├── embeddings.json
│   │   ├── generation.json
│   │   └── rerankers.json
│   ├── chunking-strategies.json
│   ├── vector-stores.json
│   ├── retrieval-strategies.json
│   ├── cloud-providers.json
│   ├── templates.json
│   └── pricing.json
│
├── docs/                                 # Documentation
│   ├── getting-started/
│   │   ├── installation.md
│   │   ├── quickstart-designer.md
│   │   └── quickstart-autopilot.md
│   ├── guides/
│   │   ├── designer-mode/
│   │   │   ├── pipeline-building.md
│   │   │   ├── cost-optimization.md
│   │   │   └── code-export.md
│   │   ├── autopilot-mode/
│   │   │   ├── build-configuration.md
│   │   │   ├── evaluation-metrics.md
│   │   │   └── deployment.md
│   │   └── integration/
│   │       ├── designer-to-autopilot.md
│   │       └── autopilot-to-designer.md
│   ├── api-reference/
│   │   ├── designer-api.md
│   │   ├── autopilot-api.md
│   │   └── core-services.md
│   └── tutorials/
│       ├── faq-chatbot.md
│       ├── documentation-qa.md
│       ├── code-assistant.md
│       └── multilingual-support.md
│
├── scripts/
│   ├── setup.sh
│   ├── seed-data.py
│   ├── migrate.sh
│   └── deploy.sh
│
├── docker/
│   ├── docker-compose.yml
│   ├── docker-compose.dev.yml
│   ├── docker-compose.prod.yml
│   └── nginx/
│       └── nginx.conf
│
├── k8s/                                  # Kubernetes manifests
│   ├── namespace.yaml
│   ├── web-deployment.yaml
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-deployment.yaml
│   ├── redis-deployment.yaml
│   ├── qdrant-deployment.yaml
│   ├── ingress.yaml
│   └── configmaps.yaml
│
├── .env.example
├── .gitignore
├── LICENSE
├── README.md
└── CLAUDE.md                             # This file
```

---

## 🎯 Core Features

### Designer Mode Features

**Pipeline Configuration:**
- ✅ Cloud provider selection (AWS, GCP, Azure, Multi-cloud)
- ✅ Step-by-step guided builder (10 stages)
- ✅ Real-time cost estimation
- ✅ Visual pipeline diagram (Mermaid)
- ✅ Model comparison tables
- ✅ Intelligent recommendations
- ✅ Template library
- ✅ Configuration validation

**Stages:**
1. Data Ingestion (source selection)
2. Chunking Strategy (7 strategies available)
3. Embedding Model (10+ models)
4. Vector Store (9 options)
5. Retrieval Strategy (6 methods)
6. Reranking (optional, 4 models)
7. Generation Model (9+ LLMs)
8. Routing Logic (query complexity-based)
9. Memory & Context (session management)
10. Evaluation Setup (metrics configuration)

**Export Capabilities:**
- ✅ Python code (LangChain/LlamaIndex)
- ✅ YAML configuration
- ✅ Terraform/CloudFormation
- ✅ Docker Compose
- ✅ Kubernetes manifests
- ✅ API documentation

**Integration:**
- ✅ "Optimize This" → sends to Autopilot
- ✅ Save/load configurations
- ✅ Share with team
- ✅ Deploy directly

---

### Autopilot Mode Features

**Autonomous Building:**
- ✅ Document upload & analysis
- ✅ Automatic format detection
- ✅ Intelligent strategy selection
- ✅ Multi-model benchmarking
- ✅ Iterative optimization loop
- ✅ One-command deployment

**Agent System:**
- ✅ Document Analyst Agent (analyzes corpus)
- ✅ Chunking Optimizer Agent (tests strategies)
- ✅ Embedding Tester Agent (benchmarks models)
- ✅ Retrieval Optimizer Agent (hybrid search tuning)
- ✅ Evaluation Agent (RAGAS metrics)
- ✅ Deployment Agent (cloud provisioning)

**Evaluation Framework:**
- ✅ RAGAS integration
- ✅ Metrics: faithfulness, answer_relevance, context_precision
- ✅ Synthetic test set generation
- ✅ Failure analysis & auto-fixing
- ✅ A/B testing
- ✅ Performance benchmarking

**Optimization:**
- ✅ Automatic hyperparameter tuning
- ✅ Cost optimization
- ✅ Latency optimization
- ✅ Quality vs speed tradeoffs
- ✅ Self-healing capabilities
- ✅ Continuous improvement

**Integration:**
- ✅ "Explain Decisions" → visualizes in Designer
- ✅ Manual override capabilities
- ✅ MLflow experiment tracking
- ✅ Production monitoring

---

### Shared Features

**Project Management:**
- ✅ Create/save/load projects
- ✅ Version control
- ✅ Collaboration (team sharing)
- ✅ Build history tracking
- ✅ Deployment management

**Evaluation:**
- ✅ RAGAS metrics
- ✅ Custom metrics
- ✅ A/B testing framework
- ✅ Performance analytics
- ✅ Cost tracking

**Deployment:**
- ✅ Docker packaging
- ✅ Kubernetes manifests
- ✅ AWS deployment (Bedrock, ECS)
- ✅ GCP deployment (Vertex AI, GKE)
- ✅ Azure deployment (OpenAI, AKS)
- ✅ Local deployment

**Monitoring:**
- ✅ Real-time metrics
- ✅ Cost dashboards
- ✅ Performance tracking
- ✅ Error alerting
- ✅ Usage analytics

---

## 🎨 Designer Mode Implementation

### User Flow

```
1. Landing Page
   ↓
2. Choose Mode → Designer
   ↓
3. Create New Project OR Load Template
   ↓
4. Select Cloud Provider
   ↓
5. Configure Each Stage:
   Step 1: Data Ingestion
   Step 2: Chunking Strategy
   Step 3: Embedding Model
   Step 4: Vector Store
   Step 5: Retrieval Strategy
   Step 6: Reranking (optional)
   Step 7: Generation Model
   Step 8: Routing Logic
   Step 9: Memory & Context
   Step 10: Evaluation Setup
   ↓
6. Review Pipeline
   - Visual diagram
   - Cost estimate
   - Performance estimate
   ↓
7. Actions:
   - Export Code
   - Save Configuration
   - Deploy
   - "Optimize This" → Autopilot
```

### Key Components

#### 1. Cloud Provider Selector

**File:** `apps/web/src/components/designer/CloudProviderSelector.tsx`

```typescript
'use client';

import { Card, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Cloud, Zap, Shield, Globe } from 'lucide-react';
import { useDesignerStore } from '@/store/designerStore';
import { CloudProvider } from '@/types/pipeline';
import cloudProviders from '@/data/cloud-providers.json';

const providerIcons = {
  aws: Cloud,
  gcp: Zap,
  azure: Shield,
  'multi-cloud': Globe,
};

export function CloudProviderSelector() {
  const { config, setCloudProvider } = useDesignerStore();

  return (
    <div className="space-y-6">
      <div>
        <h2 className="text-3xl font-bold">Select Cloud Provider</h2>
        <p className="text-neutral-600 mt-2">
          Choose your deployment platform for tailored recommendations
        </p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
        {cloudProviders.providers.map((provider) => {
          const Icon = providerIcons[provider.id];
          const isSelected = config.cloudProvider === provider.id;

          return (
            <Card
              key={provider.id}
              className={`cursor-pointer transition-all hover:shadow-lg ${
                isSelected
                  ? 'border-primary-500 ring-2 ring-primary-500 bg-primary-50'
                  : 'border-neutral-200 hover:border-primary-300'
              }`}
              onClick={() => setCloudProvider(provider.id as CloudProvider)}
            >
              <CardContent className="p-6">
                <div className="flex flex-col items-center text-center space-y-4">
                  <div
                    className={`p-4 rounded-xl ${
                      isSelected ? 'bg-primary-100' : 'bg-neutral-100'
                    }`}
                  >
                    <Icon
                      className={`h-10 w-10 ${
                        isSelected ? 'text-primary-600' : 'text-neutral-600'
                      }`}
                    />
                  </div>
                  <div>
                    <h3 className="font-semibold text-lg">{provider.name}</h3>
                    <p className="text-sm text-neutral-600 mt-1">
                      {provider.description}
                    </p>
                  </div>
                  <div className="flex flex-wrap gap-2 justify-center">
                    {provider.bestFor.slice(0, 2).map((tag) => (
                      <Badge key={tag} variant="secondary" className="text-xs">
                        {tag}
                      </Badge>
                    ))}
                  </div>
                  {provider.strengths && (
                    <ul className="text-xs text-left space-y-1 w-full">
                      {provider.strengths.slice(0, 3).map((strength, i) => (
                        <li key={i} className="text-neutral-600">
                          ✓ {strength}
                        </li>
                      ))}
                    </ul>
                  )}
                </div>
              </CardContent>
            </Card>
          );
        })}
      </div>

      {config.cloudProvider && (
        <div className="flex justify-end gap-4">
          <Button variant="outline" onClick={() => setCloudProvider(null)}>
            Change Provider
          </Button>
          <Button onClick={() => router.push('/designer/chunking')}>
            Continue to Chunking →
          </Button>
        </div>
      )}
    </div>
  );
}
```

#### 2. Stage Navigator

**File:** `apps/web/src/components/designer/StageNavigator.tsx`

```typescript
'use client';

import { Check, Circle } from 'lucide-react';
import { useDesignerStore } from '@/store/designerStore';

const stages = [
  { id: 'cloud', label: 'Cloud', path: '/designer' },
  { id: 'ingestion', label: 'Ingestion', path: '/designer/ingestion' },
  { id: 'chunking', label: 'Chunking', path: '/designer/chunking' },
  { id: 'embedding', label: 'Embedding', path: '/designer/embedding' },
  { id: 'vectorstore', label: 'Vector Store', path: '/designer/vectorstore' },
  { id: 'retrieval', label: 'Retrieval', path: '/designer/retrieval' },
  { id: 'reranking', label: 'Reranking', path: '/designer/reranking' },
  { id: 'generation', label: 'Generation', path: '/designer/generation' },
  { id: 'routing', label: 'Routing', path: '/designer/routing' },
  { id: 'memory', label: 'Memory', path: '/designer/memory' },
  { id: 'evaluation', label: 'Evaluation', path: '/designer/evaluation' },
  { id: 'review', label: 'Review', path: '/designer/review' },
];

export function StageNavigator({ currentStage }: { currentStage: string }) {
  const { config } = useDesignerStore();
  const router = useRouter();

  const isStageComplete = (stageId: string) => {
    switch (stageId) {
      case 'cloud':
        return !!config.cloudProvider;
      case 'chunking':
        return !!config.stages.chunking;
      case 'embedding':
        return !!config.stages.embedding;
      // ... other stages
      default:
        return false;
    }
  };

  const currentIndex = stages.findIndex((s) => s.id === currentStage);

  return (
    <div className="bg-white border-b sticky top-0 z-10">
      <div className="max-w-7xl mx-auto px-4 py-4">
        <div className="flex items-center justify-between overflow-x-auto">
          {stages.map((stage, index) => {
            const isComplete = isStageComplete(stage.id);
            const isCurrent = stage.id === currentStage;
            const isAccessible = index <= currentIndex + 1;

            return (
              <div key={stage.id} className="flex items-center">
                <button
                  onClick={() => isAccessible && router.push(stage.path)}
                  disabled={!isAccessible}
                  className={`flex flex-col items-center min-w-[100px] ${
                    !isAccessible ? 'opacity-50 cursor-not-allowed' : ''
                  }`}
                >
                  <div
                    className={`flex items-center justify-center w-10 h-10 rounded-full border-2 mb-2 ${
                      isComplete
                        ? 'bg-success-500 border-success-500'
                        : isCurrent
                        ? 'bg-primary-500 border-primary-500'
                        : 'bg-white border-neutral-300'
                    }`}
                  >
                    {isComplete ? (
                      <Check className="w-5 h-5 text-white" />
                    ) : (
                      <Circle
                        className={`w-5 h-5 ${
                          isCurrent ? 'text-white' : 'text-neutral-400'
                        }`}
                      />
                    )}
                  </div>
                  <span
                    className={`text-xs font-medium ${
                      isCurrent
                        ? 'text-primary-600'
                        : isComplete
                        ? 'text-success-600'
                        : 'text-neutral-600'
                    }`}
                  >
                    {stage.label}
                  </span>
                </button>
                {index < stages.length - 1 && (
                  <div
                    className={`h-0.5 w-12 mx-2 ${
                      isComplete ? 'bg-success-500' : 'bg-neutral-300'
                    }`}
                  />
                )}
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
}
```

#### 3. Chunking Configuration

**File:** `apps/web/src/components/designer/ChunkingConfig.tsx`

```typescript
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Label } from '@/components/ui/label';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { Slider } from '@/components/ui/slider';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { InfoTooltip } from '@/components/shared/InfoTooltip';
import { useDesignerStore } from '@/store/designerStore';
import chunkingStrategies from '@/data/chunking-strategies.json';
import { Lightbulb, TrendingUp, TrendingDown } from 'lucide-react';

export function ChunkingConfig() {
  const { config, updateStage } = useDesignerStore();
  const chunking = config.stages.chunking;

  const selectedStrategy = chunkingStrategies.strategies.find(
    (s) => s.id === chunking?.strategy
  );

  return (
    <div className="space-y-6">
      <div>
        <h2 className="text-3xl font-bold">Chunking Strategy</h2>
        <p className="text-neutral-600 mt-2">
          Configure how documents are split into smaller pieces for embedding
        </p>
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        {/* Strategy Selection */}
        <div className="lg:col-span-2 space-y-6">
          <Card>
            <CardHeader>
              <CardTitle className="flex items-center gap-2">
                Select Chunking Method
                <InfoTooltip content="Different strategies work better for different document types" />
              </CardTitle>
            </CardHeader>
            <CardContent className="space-y-4">
              <Select
                value={chunking?.strategy}
                onValueChange={(value) =>
                  updateStage('chunking', { strategy: value })
                }
              >
                <SelectTrigger>
                  <SelectValue placeholder="Choose a chunking strategy" />
                </SelectTrigger>
                <SelectContent>
                  {chunkingStrategies.strategies.map((strategy) => (
                    <SelectItem key={strategy.id} value={strategy.id}>
                      <div className="flex flex-col py-2">
                        <span className="font-medium">{strategy.name}</span>
                        <span className="text-xs text-neutral-600">
                          {strategy.description}
                        </span>
                        <div className="flex gap-2 mt-1">
                          <span className="text-xs text-neutral-500">
                            Complexity: {strategy.implementationComplexity}
                          </span>
                        </div>
                      </div>
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>

              {/* Chunk Size Slider */}
              <div className="space-y-3">
                <div className="flex justify-between items-center">
                  <Label className="text-base">Chunk Size (tokens)</Label>
                  <span className="text-sm font-medium px-3 py-1 bg-neutral-100 rounded">
                    {chunking?.chunkSize || 512}
                  </span>
                </div>
                <Slider
                  value={[chunking?.chunkSize || 512]}
                  onValueChange={([value]) =>
                    updateStage('chunking', { chunkSize: value })
                  }
                  min={128}
                  max={2048}
                  step={128}
                  className="w-full"
                />
                <div className="flex justify-between text-xs text-neutral-600">
                  <span>128</span>
                  <span>512</span>
                  <span>1024</span>
                  <span>2048</span>
                </div>
                <p className="text-sm text-neutral-600">
                  <TrendingDown className="inline h-4 w-4 mr-1" />
                  Smaller chunks = more precise retrieval
                  <br />
                  <TrendingUp className="inline h-4 w-4 mr-1" />
                  Larger chunks = more context for LLM
                </p>
              </div>

              {/* Chunk Overlap Slider */}
              <div className="space-y-3">
                <div className="flex justify-between items-center">
                  <Label className="text-base">
                    Chunk Overlap (tokens)
                    <InfoTooltip content="Overlap prevents information loss at chunk boundaries" />
                  </Label>
                  <span className="text-sm font-medium px-3 py-1 bg-neutral-100 rounded">
                    {chunking?.chunkOverlap || 50}
                  </span>
                </div>
                <Slider
                  value={[chunking?.chunkOverlap || 50]}
                  onValueChange={([value]) =>
                    updateStage('chunking', { chunkOverlap: value })
                  }
                  min={0}
                  max={200}
                  step={10}
                  className="w-full"
                />
                <p className="text-sm text-neutral-600">
                  Recommended: 10-20% of chunk size (
                  {Math.round((chunking?.chunkSize || 512) * 0.1)}-
                  {Math.round((chunking?.chunkSize || 512) * 0.2)} tokens)
                </p>
              </div>
            </CardContent>
          </Card>

          {/* Recommendation Alert */}
          {selectedStrategy && (
            <Alert>
              <Lightbulb className="h-5 w-5" />
              <AlertDescription>
                <strong className="font-semibold">
                  {selectedStrategy.name}
                </strong>{' '}
                is best for:{' '}
                <span className="text-primary-600">
                  {selectedStrategy.bestFor.join(', ')}
                </span>
              </AlertDescription>
            </Alert>
          )}
        </div>

        {/* Strategy Details Sidebar */}
        {selectedStrategy && (
          <Card>
            <CardHeader>
              <CardTitle className="text-lg">Strategy Details</CardTitle>
            </CardHeader>
            <CardContent>
              <Tabs defaultValue="pros">
                <TabsList className="grid w-full grid-cols-2">
                  <TabsTrigger value="pros">Pros</TabsTrigger>
                  <TabsTrigger value="cons">Cons</TabsTrigger>
                </TabsList>
                <TabsContent value="pros" className="space-y-2 mt-4">
                  {selectedStrategy.pros.map((pro, i) => (
                    <div key={i} className="flex items-start gap-2">
                      <Check className="h-4 w-4 text-success-600 mt-0.5 flex-shrink-0" />
                      <span className="text-sm">{pro}</span>
                    </div>
                  ))}
                </TabsContent>
                <TabsContent value="cons" className="space-y-2 mt-4">
                  {selectedStrategy.cons.map((con, i) => (
                    <div key={i} className="flex items-start gap-2">
                      <AlertCircle className="h-4 w-4 text-warning-600 mt-0.5 flex-shrink-0" />
                      <span className="text-sm">{con}</span>
                    </div>
                  ))}
                </TabsContent>
              </Tabs>

              <div className="mt-6 pt-6 border-t">
                <h4 className="font-semibold mb-3">Default Configuration</h4>
                <dl className="space-y-2 text-sm">
                  <div className="flex justify-between">
                    <dt className="text-neutral-600">Chunk Size:</dt>
                    <dd className="font-medium">
                      {selectedStrategy.defaultConfig.chunkSize} tokens
                    </dd>
                  </div>
                  {selectedStrategy.defaultConfig.chunkOverlap && (
                    <div className="flex justify-between">
                      <dt className="text-neutral-600">Overlap:</dt>
                      <dd className="font-medium">
                        {selectedStrategy.defaultConfig.chunkOverlap} tokens
                      </dd>
                    </div>
                  )}
                </dl>
              </div>
            </CardContent>
          </Card>
        )}
      </div>

      {/* Navigation */}
      <div className="flex justify-between pt-6 border-t">
        <Button variant="outline" onClick={() => router.push('/designer')}>
          ← Back to Cloud Selection
        </Button>
        <Button
          onClick={() => router.push('/designer/embedding')}
          disabled={!chunking?.strategy}
        >
          Continue to Embedding →
        </Button>
      </div>
    </div>
  );
}
```

#### 4. Pipeline Visualizer

**File:** `apps/web/src/components/shared/PipelineVisualizer.tsx`

```typescript
'use client';

import { useEffect, useRef } from 'react';
import mermaid from 'mermaid';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Download, Maximize2 } from 'lucide-react';
import { useDesignerStore } from '@/store/designerStore';
import { generateMermaidDiagram } from '@/lib/generators/mermaidGenerator';

mermaid.initialize({
  startOnLoad: true,
  theme: 'default',
  securityLevel: 'loose',
  flowchart: {
    useMaxWidth: true,
    htmlLabels: true,
    curve: 'basis',
  },
});

export function PipelineVisualizer() {
  const mermaidRef = useRef<HTMLDivElement>(null);
  const { config } = useDesignerStore();

  useEffect(() => {
    if (mermaidRef.current) {
      const diagram = generateMermaidDiagram(config.stages, config.cloudProvider);
      mermaidRef.current.innerHTML = diagram;
      mermaid.contentLoaded();
    }
  }, [config.stages, config.cloudProvider]);

  const handleDownload = () => {
    // Convert SVG to PNG and download
    const svg = mermaidRef.current?.querySelector('svg');
    if (svg) {
      const svgData = new XMLSerializer().serializeToString(svg);
      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d');
      const img = new Image();
      img.onload = () => {
        canvas.width = img.width;
        canvas.height = img.height;
        ctx?.drawImage(img, 0, 0);
        canvas.toBlob((blob) => {
          if (blob) {
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'pipeline-diagram.png';
            a.click();
            URL.revokeObjectURL(url);
          }
        });
      };
      img.src = 'data:image/svg+xml;base64,' + btoa(svgData);
    }
  };

  return (
    <Card>
      <CardHeader>
        <div className="flex items-center justify-between">
          <CardTitle>Pipeline Architecture</CardTitle>
          <div className="flex gap-2">
            <Button variant="outline" size="sm" onClick={handleDownload}>
              <Download className="h-4 w-4 mr-2" />
              Download
            </Button>
            <Button variant="outline" size="sm">
              <Maximize2 className="h-4 w-4 mr-2" />
              Fullscreen
            </Button>
          </div>
        </div>
      </CardHeader>
      <CardContent>
        <div
          ref={mermaidRef}
          className="mermaid bg-white p-6 rounded-lg border"
          style={{ textAlign: 'center', minHeight: '400px' }}
        />
        <div className="mt-4 p-4 bg-neutral-50 rounded-lg">
          <h4 className="font-semibold mb-2">Pipeline Summary</h4>
          <dl className="grid grid-cols-2 gap-3 text-sm">
            <div>
              <dt className="text-neutral-600">Cloud Provider:</dt>
              <dd className="font-medium">{config.cloudProvider || 'Not set'}</dd>
            </div>
            <div>
              <dt className="text-neutral-600">Chunking:</dt>
              <dd className="font-medium">
                {config.stages.chunking?.strategy || 'Not configured'}
              </dd>
            </div>
            <div>
              <dt className="text-neutral-600">Embedding:</dt>
              <dd className="font-medium">
                {config.stages.embedding?.model || 'Not configured'}
              </dd>
            </div>
            <div>
              <dt className="text-neutral-600">Vector Store:</dt>
              <dd className="font-medium">
                {config.stages.vectorStore?.provider || 'Not configured'}
              </dd>
            </div>
            <div>
              <dt className="text-neutral-600">Retrieval:</dt>
              <dd className="font-medium">
                {config.stages.retrieval?.strategy || 'Not configured'}
              </dd>
            </div>
            <div>
              <dt className="text-neutral-600">Generation:</dt>
              <dd className="font-medium">
                {config.stages.generation?.model || 'Not configured'}
              </dd>
            </div>
          </dl>
        </div>
      </CardContent>
    </Card>
  );
}
```

#### 5. Cost Estimator

**File:** `apps/web/src/components/shared/CostEstimator.tsx`

```typescript
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Progress } from '@/components/ui/progress';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { useDesignerStore } from '@/store/designerStore';
import { calculateCosts } from '@/lib/utils/costCalculator';
import {
  DollarSign,
  TrendingUp,
  TrendingDown,
  Info,
  Lightbulb,
} from 'lucide-react';

export function CostEstimator() {
  const { config } = useDesignerStore();
  const costs = calculateCosts(config.stages);

  const optimizationTips = [
    {
      condition: costs.perQuery > 0.1,
      type: 'warning',
      message: 'Consider using a smaller embedding model to reduce costs',
    },
    {
      condition: config.stages.reranking?.enabled,
      type: 'info',
      message: 'Reranking adds precision but increases cost by ~20-30%',
    },
    {
      condition: config.stages.generation?.model?.includes('gpt-4'),
      type: 'info',
      message: 'GPT-4 provides highest quality but consider GPT-4o-mini for simple queries',
    },
    {
      condition: !config.stages.routing?.enabled && costs.perQuery > 0.05,
      type: 'tip',
      message: 'Enable routing logic to use cheaper models for simple queries',
    },
  ];

  return (
    <Card>
      <CardHeader>
        <CardTitle className="flex items-center gap-2">
          <DollarSign className="h-5 w-5" />
          Cost Estimation
        </CardTitle>
      </CardHeader>
      <CardContent className="space-y-6">
        {/* Total Cost Summary */}
        <div className="grid grid-cols-2 gap-4">
          <div className="p-5 bg-primary-50 rounded-lg border border-primary-200">
            <p className="text-sm text-neutral-600 mb-1">Per 1K Queries</p>
            <p className="text-3xl font-bold text-primary-600">
              ${costs.perQuery.toFixed(3)}
            </p>
          </div>
          <div className="p-5 bg-neutral-50 rounded-lg border border-neutral-200">
            <p className="text-sm text-neutral-600 mb-1">
              Per Month (100K queries)
            </p>
            <p className="text-2xl font-bold text-neutral-900">
              ${costs.perMonth.toFixed(2)}
            </p>
          </div>
        </div>

        {/* Cost Breakdown */}
        <div className="space-y-4">
          <h4 className="font-semibold">Cost Breakdown</h4>
          {costs.breakdown.map((item) => (
            <div key={item.component} className="space-y-2">
              <div className="flex justify-between text-sm">
                <span className="capitalize font-medium">{item.component}</span>
                <span className="text-neutral-600">
                  ${item.totalCost.toFixed(4)} ({item.percentage.toFixed(1)}%)
                </span>
              </div>
              <Progress
                value={item.percentage}
                className="h-2"
                indicatorClassName={
                  item.percentage > 50
                    ? 'bg-danger-500'
                    : item.percentage > 30
                    ? 'bg-warning-500'
                    : 'bg-success-500'
                }
              />
            </div>
          ))}
        </div>

        {/* Detailed Breakdown Table */}
        <div className="border rounded-lg overflow-hidden">
          <table className="w-full text-sm">
            <thead className="bg-neutral-50">
              <tr>
                <th className="text-left p-3 font-semibold">Component</th>
                <th className="text-right p-3 font-semibold">Unit Cost</th>
                <th className="text-right p-3 font-semibold">Usage</th>
                <th className="text-right p-3 font-semibold">Total</th>
              </tr>
            </thead>
            <tbody className="divide-y">
              {costs.breakdown.map((item) => (
                <tr key={item.component}>
                  <td className="p-3 capitalize">{item.component}</td>
                  <td className="p-3 text-right text-neutral-600">
                    ${item.unitCost.toFixed(6)}
                  </td>
                  <td className="p-3 text-right text-neutral-600">
                    {item.estimatedUsage.toLocaleString()}
                  </td>
                  <td className="p-3 text-right font-medium">
                    ${item.totalCost.toFixed(4)}
                  </td>
                </tr>
              ))}
              <tr className="bg-neutral-50 font-semibold">
                <td className="p-3" colSpan={3}>
                  Total per 1K Queries
                </td>
                <td className="p-3 text-right">${costs.perQuery.toFixed(4)}</td>
              </tr>
            </tbody>
          </table>
        </div>

        {/* Optimization Tips */}
        {optimizationTips.some((tip) => tip.condition) && (
          <div className="space-y-3">
            <h4 className="font-semibold flex items-center gap-2">
              <Lightbulb className="h-4 w-4" />
              Optimization Suggestions
            </h4>
            {optimizationTips
              .filter((tip) => tip.condition)
              .map((tip, i) => (
                <Alert key={i} variant={tip.type === 'warning' ? 'destructive' : 'default'}>
                  {tip.type === 'warning' ? (
                    <TrendingUp className="h-4 w-4" />
                  ) : tip.type === 'tip' ? (
                    <Lightbulb className="h-4 w-4" />
                  ) : (
                    <Info className="h-4 w-4" />
                  )}
                  <AlertDescription>{tip.message}</AlertDescription>
                </Alert>
              ))}
          </div>
        )}

        {/* Cost Comparison */}
        <div className="p-4 bg-neutral-50 rounded-lg border">
          <h4 className="font-semibold mb-3">Industry Benchmark</h4>
          <div className="space-y-2 text-sm">
            <div className="flex justify-between">
              <span className="text-neutral-600">Your Configuration:</span>
              <span className="font-medium">${costs.perQuery.toFixed(3)}/1K</span>
            </div>
            <div className="flex justify-between">
              <span className="text-neutral-600">Industry Average:</span>
              <span className="font-medium">$0.050/1K</span>
            </div>
            <div className="flex justify-between">
              <span className="text-neutral-600">Budget-Optimized:</span>
              <span className="font-medium text-success-600">$0.015/1K</span>
            </div>
            <div className="flex justify-between">
              <span className="text-neutral-600">Premium (High-Quality):</span>
              <span className="font-medium text-warning-600">$0.120/1K</span>
            </div>
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

---

## 🤖 Autopilot Mode Implementation

### User Flow

```
1. Landing Page
   ↓
2. Choose Mode → Autopilot
   ↓
3. Create New Build
   ↓
4. Upload Documents
   - Drag & drop files
   - Connect data sources (S3, GCS, etc.)
   ↓
5. Set Requirements
   - Target metrics (faithfulness, relevance)
   - Budget constraints
   - Latency requirements
   - Cloud preference (optional)
   ↓
6. Click "Build RAG System"
   ↓
7. Monitor Progress
   - Real-time agent activity feed
   - Current task visualization
   - Metrics dashboard
   ↓
8. Review Results
   - Performance metrics
   - Cost analysis
   - Decision explanations
   ↓
9. Actions:
   - Deploy
   - "Explain Decisions" → Designer visualization
   - Refine & Rebuild
   - Export Configuration
```

### Agent System Architecture

```
┌────────────────────────────────────────────────────────────┐
│              AUTOPILOT ORCHESTRATOR (LangGraph)            │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Agent Coordination Flow                 │ │
│  │                                                      │ │
│  │  1. Document Analyst Agent                          │ │
│  │     ├─> Analyzes document corpus                    │ │
│  │     ├─> Detects format, language, structure         │ │
│  │     └─> Recommends initial strategy                 │ │
│  │              ↓                                       │ │
│  │  2. Chunking Optimizer Agent                        │ │
│  │     ├─> Tests multiple strategies                   │ │
│  │     ├─> Evaluates chunk quality                     │ │
│  │     └─> Selects optimal config                      │ │
│  │              ↓                                       │ │
│  │  3. Embedding Tester Agent                          │ │
│  │     ├─> Benchmarks 3-5 models                       │ │
│  │     ├─> Measures speed/quality/cost                 │ │
│  │     └─> Ranks by weighted score                     │ │
│  │              ↓                                       │ │
│  │  4. Retrieval Optimizer Agent                       │ │
│  │     ├─> Tests retrieval strategies                  │ │
│  │     ├─> Tunes hybrid search                         │ │
│  │     ├─> Configures reranking                        │ │
│  │     └─> Optimizes top-k                             │ │
│  │              ↓                                       │ │
│  │  5. Evaluation Agent                                │ │
│  │     ├─> Generates synthetic test set                │ │
│  │     ├─> Runs RAGAS evaluation                       │ │
│  │     ├─> Analyzes failures                           │ │
│  │     └─> Triggers iteration if needed                │ │
│  │              ↓                                       │ │
│  │  6. Deployment Agent                                │ │
│  │     ├─> Packages pipeline                           │ │
│  │     ├─> Generates deployment files                  │ │
│  │     └─> Provisions infrastructure                   │ │
│  │                                                      │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Shared Agent Tools                      │ │
│  │  • Document Loader                                   │ │
│  │  • Vector Index Manager                              │ │
│  │  • LLM Caller                                        │ │
│  │  • Metrics Calculator                                │ │
│  │  • Config Generator                                  │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. Build Orchestrator

**File:** `apps/api/app/agents/orchestrator.py`

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated, List
import operator
from app.agents.document_analyst import DocumentAnalystAgent
from app.agents.chunking_optimizer import ChunkingOptimizerAgent
from app.agents.embedding_tester import EmbeddingTesterAgent
from app.agents.retrieval_optimizer import RetrievalOptimizerAgent
from app.agents.evaluation_agent import EvaluationAgent
from app.agents.deployment_agent import DeploymentAgent

class AutopilotState(TypedDict):
    """State that flows through the agent graph"""
    # Input
    documents: List[str]
    requirements: dict
    
    # Intermediate state
    document_analysis: dict
    chunking_config: dict
    embedding_config: dict
    vectorstore_config: dict
    retrieval_config: dict
    generation_config: dict
    
    # Evaluation
    test_set: List[dict]
    metrics: dict
    failure_analysis: dict
    
    # Iteration control
    iteration: int
    max_iterations: int
    target_metrics: dict
    current_metrics: dict
    
    # Output
    final_config: dict
    deployment_info: dict
    
    # Agent communication
    messages: Annotated[List[str], operator.add]
    decisions: dict

class AutopilotOrchestrator:
    """Main orchestrator for Autopilot mode using LangGraph"""
    
    def __init__(self):
        self.document_analyst = DocumentAnalystAgent()
        self.chunking_optimizer = ChunkingOptimizerAgent()
        self.embedding_tester = EmbeddingTesterAgent()
        self.retrieval_optimizer = RetrievalOptimizerAgent()
        self.evaluation_agent = EvaluationAgent()
        self.deployment_agent = DeploymentAgent()
        
        self.graph = self._build_graph()
    
    def _build_graph(self) -> StateGraph:
        """Build the LangGraph workflow"""
        workflow = StateGraph(AutopilotState)
        
        # Add nodes
        workflow.add_node("analyze_documents", self.analyze_documents)
        workflow.add_node("optimize_chunking", self.optimize_chunking)
        workflow.add_node("test_embeddings", self.test_embeddings)
        workflow.add_node("optimize_retrieval", self.optimize_retrieval)
        workflow.add_node("evaluate_pipeline", self.evaluate_pipeline)
        workflow.add_node("decide_iteration", self.decide_iteration)
        workflow.add_node("deploy_system", self.deploy_system)
        
        # Define edges
        workflow.set_entry_point("analyze_documents")
        workflow.add_edge("analyze_documents", "optimize_chunking")
        workflow.add_edge("optimize_chunking", "test_embeddings")
        workflow.add_edge("test_embeddings", "optimize_retrieval")
        workflow.add_edge("optimize_retrieval", "evaluate_pipeline")
        workflow.add_edge("evaluate_pipeline", "decide_iteration")
        
        # Conditional edge: iterate or deploy
        workflow.add_conditional_edges(
            "decide_iteration",
            self.should_iterate,
            {
                "iterate": "optimize_chunking",  # Try again with improvements
                "deploy": "deploy_system",       # Good enough, deploy
            }
        )
        
        workflow.add_edge("deploy_system", END)
        
        return workflow.compile()
    
    def analyze_documents(self, state: AutopilotState) -> AutopilotState:
        """Step 1: Analyze document corpus"""
        analysis = self.document_analyst.analyze(state["documents"])
        
        state["document_analysis"] = analysis
        state["messages"].append(
            f"📄 Analyzed {len(state['documents'])} documents: "
            f"{analysis['total_chars']} chars, "
            f"detected {analysis['languages']}"
        )
        
        return state
    
    def optimize_chunking(self, state: AutopilotState) -> AutopilotState:
        """Step 2: Find optimal chunking strategy"""
        chunking_config = self.chunking_optimizer.optimize(
            documents=state["documents"],
            analysis=state["document_analysis"],
            requirements=state["requirements"]
        )
        
        state["chunking_config"] = chunking_config
        state["messages"].append(
            f"✂️  Selected {chunking_config['strategy']} chunking: "
            f"{chunking_config['chunk_size']} tokens, "
            f"{chunking_config['overlap']} overlap"
        )
        
        return state
    
    def test_embeddings(self, state: AutopilotState) -> AutopilotState:
        """Step 3: Benchmark embedding models"""
        embedding_config = self.embedding_tester.benchmark(
            chunks=state["chunking_config"]["sample_chunks"],
            requirements=state["requirements"]
        )
        
        state["embedding_config"] = embedding_config
        state["messages"].append(
            f"🎯 Selected embedding model: {embedding_config['model']} "
            f"(score: {embedding_config['score']:.2f})"
        )
        
        return state
    
    def optimize_retrieval(self, state: AutopilotState) -> AutopilotState:
        """Step 4: Optimize retrieval strategy"""
        retrieval_config = self.retrieval_optimizer.optimize(
            vectorstore=state["vectorstore_config"],
            requirements=state["requirements"]
        )
        
        state["retrieval_config"] = retrieval_config
        state["messages"].append(
            f"🔍 Configured retrieval: {retrieval_config['strategy']}, "
            f"top-k={retrieval_config['top_k']}, "
            f"reranking={'enabled' if retrieval_config.get('reranking') else 'disabled'}"
        )
        
        return state
    
    def evaluate_pipeline(self, state: AutopilotState) -> AutopilotState:
        """Step 5: Evaluate RAG pipeline with RAGAS"""
        # Build complete pipeline from config
        pipeline = self._build_pipeline_from_state(state)
        
        # Generate test set if not exists
        if not state.get("test_set"):
            state["test_set"] = self.evaluation_agent.generate_test_set(
                documents=state["documents"],
                num_questions=100
            )
        
        # Run evaluation
        metrics = self.evaluation_agent.evaluate(
            pipeline=pipeline,
            test_set=state["test_set"]
        )
        
        state["current_metrics"] = metrics
        state["messages"].append(
            f"📊 Evaluation: "
            f"faithfulness={metrics['faithfulness']:.2f}, "
            f"relevance={metrics['answer_relevance']:.2f}, "
            f"precision={metrics['context_precision']:.2f}"
        )
        
        return state
    
    def decide_iteration(self, state: AutopilotState) -> AutopilotState:
        """Decide whether to iterate or deploy"""
        state["iteration"] = state.get("iteration", 0) + 1
        
        # Analyze failures if below target
        target = state["target_metrics"]
        current = state["current_metrics"]
        
        if (current["faithfulness"] < target.get("faithfulness", 0.85) or
            current["answer_relevance"] < target.get("answer_relevance", 0.80)):
            
            # Perform failure analysis
            failures = self.evaluation_agent.analyze_failures(
                pipeline=self._build_pipeline_from_state(state),
                test_set=state["test_set"],
                metrics=current
            )
            
            state["failure_analysis"] = failures
            state["messages"].append(
                f"🔧 Detected issues: {', '.join(failures['categories'])}"
            )
        
        return state
    
    def should_iterate(self, state: AutopilotState) -> str:
        """Conditional: Should we iterate or deploy?"""
        # Check iteration limit
        if state["iteration"] >= state["max_iterations"]:
            state["messages"].append("⏸️  Max iterations reached, deploying current best")
            return "deploy"
        
        # Check if metrics meet targets
        target = state["target_metrics"]
        current = state["current_metrics"]
        
        meets_target = (
            current.get("faithfulness", 0) >= target.get("faithfulness", 0.85) and
            current.get("answer_relevance", 0) >= target.get("answer_relevance", 0.80)
        )
        
        if meets_target:
            state["messages"].append("✅ Target metrics achieved!")
            return "deploy"
        else:
            state["messages"].append("🔄 Iterating to improve metrics...")
            return "iterate"
    
    def deploy_system(self, state: AutopilotState) -> AutopilotState:
        """Step 6: Package and deploy RAG system"""
        final_config = {
            "chunking": state["chunking_config"],
            "embedding": state["embedding_config"],
            "vectorstore": state["vectorstore_config"],
            "retrieval": state["retrieval_config"],
            "generation": state["generation_config"],
        }
        
        deployment_info = self.deployment_agent.deploy(
            config=final_config,
            cloud_provider=state["requirements"].get("cloud_provider", "docker")
        )
        
        state["final_config"] = final_config
        state["deployment_info"] = deployment_info
        state["messages"].append(
            f"🚀 Deployed to {deployment_info['provider']}: "
            f"{deployment_info['endpoint']}"
        )
        
        return state
    
    def _build_pipeline_from_state(self, state: AutopilotState):
        """Helper to build RAG pipeline from current state"""
        # Implementation would create actual RAG pipeline
        # using LangChain or custom code
        pass
    
    async def run(self, documents: List[str], requirements: dict) -> dict:
        """Execute the autopilot workflow"""
        initial_state = {
            "documents": documents,
            "requirements": requirements,
            "iteration": 0,
            "max_iterations": requirements.get("max_iterations", 5),
            "target_metrics": requirements.get("target_metrics", {
                "faithfulness": 0.85,
                "answer_relevance": 0.80,
                "context_precision": 0.75,
            }),
            "messages": [],
            "decisions": {},
        }
        
        # Run the graph
        final_state = self.graph.invoke(initial_state)
        
        return {
            "config": final_state["final_config"],
            "metrics": final_state["current_metrics"],
            "deployment": final_state["deployment_info"],
            "messages": final_state["messages"],
            "iterations": final_state["iteration"],
        }
```

#### 2. Document Analyst Agent

**File:** `apps/api/app/agents/document_analyst.py`

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.llms import ChatOpenAI
from typing import List, Dict
import tiktoken

class DocumentAnalystAgent:
    """Analyzes document corpus to inform strategy selection"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
        self.tokenizer = tiktoken.get_encoding("cl100k_base")
    
    def analyze(self, documents: List[str]) -> Dict:
        """
        Comprehensive document analysis
        
        Returns:
            - total_docs: int
            - total_chars: int
            - total_tokens: int
            - avg_doc_length: int
            - languages: List[str]
            - format_types: Dict[str, int]
            - structure_analysis: Dict
            - recommended_chunk_size: int
        """
        analysis = {
            "total_docs": len(documents),
            "total_chars": sum(len(doc) for doc in documents),
            "avg_doc_length": sum(len(doc) for doc in documents) // len(documents) if documents else 0,
        }
        
        # Token analysis
        total_tokens = 0
        for doc in documents:
            total_tokens += len(self.tokenizer.encode(doc))
        analysis["total_tokens"] = total_tokens
        analysis["avg_tokens_per_doc"] = total_tokens // len(documents) if documents else 0
        
        # Language detection (simplified)
        analysis["languages"] = self._detect_languages(documents)
        
        # Structure analysis
        analysis["structure_analysis"] = self._analyze_structure(documents)
        
        # Recommended chunk size based on document characteristics
        analysis["recommended_chunk_size"] = self._recommend_chunk_size(analysis)
        
        return analysis
    
    def _detect_languages(self, documents: List[str]) -> List[str]:
        """Detect languages in documents"""
        # Simplified - would use langdetect or fastText in production
        return ["english"]
    
    def _analyze_structure(self, documents: List[str]) -> Dict:
        """Analyze document structure"""
        has_code = any("```" in doc or "def " in doc or "class " in doc for doc in documents)
        has_markdown_headers = any(doc.count("#") > 3 for doc in documents)
        has_lists = any("\n- " in doc or "\n* " in doc for doc in documents)
        
        return {
            "has_code_blocks": has_code,
            "has_markdown_headers": has_markdown_headers,
            "has_lists": has_lists,
            "recommended_strategy": "code-aware" if has_code else "markdown" if has_markdown_headers else "recursive"
        }
    
    def _recommend_chunk_size(self, analysis: Dict) -> int:
        """Recommend chunk size based on analysis"""
        avg_length = analysis["avg_tokens_per_doc"]
        
        if avg_length < 500:
            return 256  # Small documents, small chunks
        elif avg_length < 2000:
            return 512  # Medium documents
        else:
            return 1024  # Large documents, larger chunks for context
```

#### 3. Evaluation Agent

**File:** `apps/api/app/agents/evaluation_agent.py`

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from langchain.llms import ChatOpenAI
from langchain.embeddings import OpenAIEmbeddings
from typing import List, Dict
import pandas as pd

class EvaluationAgent:
    """Handles RAG pipeline evaluation using RAGAS"""
    
    def __init__(self):
        self.llm = ChatOpenAI(model="gpt-4o-mini")
        self.embeddings = OpenAIEmbeddings()
        
        self.metrics = [
            faithfulness,
            answer_relevancy,
            context_precision,
            context_recall,
        ]
    
    def generate_test_set(self, documents: List[str], num_questions: int = 100) -> List[Dict]:
        """Generate synthetic test set from documents"""
        # Use LLM to generate question-answer pairs
        # This is a simplified version - production would be more sophisticated
        
        test_set = []
        # Implementation of test set generation
        # Would use LLM to create diverse questions from documents
        
        return test_set
    
    def evaluate(self, pipeline, test_set: List[Dict]) -> Dict:
        """
        Evaluate RAG pipeline using RAGAS metrics
        
        Args:
            pipeline: RAG pipeline to evaluate
            test_set: List of {"question": str, "ground_truth": str}
        
        Returns:
            Dict of metric scores
        """
        # Prepare dataset for RAGAS
        questions = [item["question"] for item in test_set]
        ground_truths = [item["ground_truth"] for item in test_set]
        
        # Get pipeline responses
        answers = []
        contexts = []
        
        for question in questions:
            result = pipeline({"query": question})
            answers.append(result["answer"])
            contexts.append([doc.page_content for doc in result["source_documents"]])
        
        # Create evaluation dataset
        eval_dataset = {
            "question": questions,
            "answer": answers,
            "contexts": contexts,
            "ground_truth": ground_truths,
        }
        
        # Run RAGAS evaluation
        result = evaluate(
            dataset=eval_dataset,
            metrics=self.metrics,
            llm=self.llm,
            embeddings=self.embeddings,
        )
        
        return {
            "faithfulness": result["faithfulness"],
            "answer_relevance": result["answer_relevancy"],
            "context_precision": result["context_precision"],
            "context_recall": result["context_recall"],
        }
    
    def analyze_failures(self, pipeline, test_set: List[Dict], metrics: Dict) -> Dict:
        """
        Analyze failed cases to identify improvement opportunities
        
        Returns:
            Dict with failure categories and recommendations
        """
        failures = {
            "low_faithfulness": [],
            "low_relevance": [],
            "poor_retrieval": [],
            "categories": [],
            "recommendations": []
        }
        
        # Run detailed analysis on low-scoring examples
        # Categorize failures (retrieval, generation, both)
        # Provide actionable recommendations
        
        if metrics["faithfulness"] < 0.8:
            failures["categories"].append("hallucination")
            failures["recommendations"].append(
                "Reduce LLM temperature or add stronger grounding prompts"
            )
        
        if metrics["context_precision"] < 0.7:
            failures["categories"].append("retrieval_quality")
            failures["recommendations"].append(
                "Enable reranking or tune retrieval strategy"
            )
        
        return failures
```

#### 4. Build Progress Component

**File:** `apps/web/src/components/autopilot/BuildProgress.tsx`

```typescript
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Progress } from '@/components/ui/progress';
import { Badge } from '@/components/ui/badge';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { useAutopilotStore } from '@/store/autopilotStore';
import {
  CheckCircle2,
  Circle,
  Loader2,
  AlertCircle,
  FileText,
  Scissors,
  Zap,
  Database,
  Search,
  Brain,
  Rocket,
} from 'lucide-react';

const stageIcons = {
  analyze: FileText,
  chunking: Scissors,
  embedding: Zap,
  vectorstore: Database,
  retrieval: Search,
  evaluation: Brain,
  deployment: Rocket,
};

const stageLabels = {
  analyze: 'Analyzing Documents',
  chunking: 'Optimizing Chunking',
  embedding: 'Testing Embeddings',
  vectorstore: 'Creating Vector Index',
  retrieval: 'Optimizing Retrieval',
  evaluation: 'Evaluating Pipeline',
  deployment: 'Deploying System',
};

export function BuildProgress() {
  const { currentBuild, messages } = useAutopilotStore();

  if (!currentBuild) return null;

  const { status, currentStage, progress, stages, iteration } = currentBuild;

  const getStageStatus = (stageId: string) => {
    const stage = stages[stageId];
    if (!stage) return 'pending';
    return stage.status;
  };

  const getStageMessage = (stageId: string) => {
    const stage = stages[stageId];
    return stage?.message || '';
  };

  const overallProgress =
    (Object.values(stages).filter((s) => s.status === 'complete').length /
      Object.keys(stageLabels).length) *
    100;

  return (
    <div className="space-y-6">
      {/* Overall Progress */}
      <Card>
        <CardHeader>
          <div className="flex items-center justify-between">
            <CardTitle className="flex items-center gap-2">
              {status === 'running' && (
                <Loader2 className="h-5 w-5 animate-spin text-primary-600" />
              )}
              {status === 'complete' && (
                <CheckCircle2 className="h-5 w-5 text-success-600" />
              )}
              {status === 'failed' && (
                <AlertCircle className="h-5 w-5 text-danger-600" />
              )}
              Building Your RAG System
            </CardTitle>
            <Badge variant={status === 'complete' ? 'success' : 'default'}>
              {status === 'running' ? `${Math.round(overallProgress)}%` : status}
            </Badge>
          </div>
        </CardHeader>
        <CardContent className="space-y-4">
          <Progress value={overallProgress} className="h-3" />

          {iteration > 1 && (
            <Alert>
              <AlertDescription>
                Iteration {iteration}: Refining based on evaluation results
              </Alert>
            </Alert>
          )}
        </CardContent>
      </Card>

      {/* Stage Details */}
      <Card>
        <CardHeader>
          <CardTitle>Build Stages</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-4">
            {Object.entries(stageLabels).map(([stageId, label]) => {
              const status = getStageStatus(stageId);
              const message = getStageMessage(stageId);
              const Icon = stageIcons[stageId];
              const isCurrent = currentStage === stageId;

              return (
                <div
                  key={stageId}
                  className={`flex items-start gap-4 p-4 rounded-lg border ${
                    isCurrent ? 'bg-primary-50 border-primary-200' : 'bg-white'
                  }`}
                >
                  <div className="flex-shrink-0">
                    {status === 'complete' ? (
                      <CheckCircle2 className="h-6 w-6 text-success-600" />
                    ) : status === 'running' ? (
                      <Loader2 className="h-6 w-6 animate-spin text-primary-600" />
                    ) : status === 'failed' ? (
                      <AlertCircle className="h-6 w-6 text-danger-600" />
                    ) : (
                      <Circle className="h-6 w-6 text-neutral-300" />
                    )}
                  </div>

                  <div className="flex-1">
                    <div className="flex items-center gap-2 mb-1">
                      <Icon className="h-4 w-4 text-neutral-600" />
                      <h4 className="font-semibold">{label}</h4>
                    </div>

                    {message && (
                      <p className="text-sm text-neutral-600">{message}</p>
                    )}

                    {status === 'running' && (
                      <div className="mt-2">
                        <Progress value={50} className="h-1" />
                      </div>
                    )}
                  </div>

                  <Badge
                    variant={
                      status === 'complete'
                        ? 'success'
                        : status === 'failed'
                        ? 'destructive'
                        : 'secondary'
                    }
                  >
                    {status}
                  </Badge>
                </div>
              );
            })}
          </div>
        </CardContent>
      </Card>

      {/* Activity Feed */}
      <Card>
        <CardHeader>
          <CardTitle>Activity Log</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-2 max-h-96 overflow-y-auto">
            {messages.map((msg, i) => (
              <div
                key={i}
                className="flex items-start gap-3 p-3 bg-neutral-50 rounded text-sm"
              >
                <span className="text-neutral-400">{msg.timestamp}</span>
                <span>{msg.text}</span>
              </div>
            ))}
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

## 🔄 Integration Layer

### Designer → Autopilot Flow

**File:** `apps/web/src/components/shared/OptimizeButton.tsx`

```typescript
'use client';

import { Button } from '@/components/ui/button';
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog';
import { useDesignerStore } from '@/store/designerStore';
import { useAutopilotStore } from '@/store/autopilotStore';
import { useState } from 'react';
import { Zap } from 'lucide-react';

export function OptimizeButton() {
  const { config } = useDesignerStore();
  const { startBuild } = useAutopilotStore();
  const [showDialog, setShowDialog] = useState(false);
  const router = useRouter();

  const handleOptimize = async () => {
    // Convert Designer config to Autopilot requirements
    const autopilotConfig = {
      baseConfig: config,
      documents: [], // User will upload
      requirements: {
        target_metrics: {
          faithfulness: 0.85,
          answer_relevance: 0.80,
        },
        cloud_provider: config.cloudProvider,
        budget_constraint: null, // Will use current cost as baseline
        optimize_for: 'quality', // or 'cost' or 'latency'
      },
    };

    // Start Autopilot build with Designer config as starting point
    const buildId = await startBuild(autopilotConfig);
    
    // Redirect to Autopilot progress page
    router.push(`/autopilot/build/${buildId}`);
  };

  return (
    <>
      <Button
        onClick={() => setShowDialog(true)}
        className="flex items-center gap-2"
      >
        <Zap className="h-4 w-4" />
        Optimize This Configuration
      </Button>

      <Dialog open={showDialog} onOpenChange={setShowDialog}>
        <DialogContent>
          <DialogHeader>
            <DialogTitle>Optimize with Autopilot</DialogTitle>
          </DialogHeader>
          <div className="space-y-4">
            <p className="text-neutral-600">
              Autopilot will use your current configuration as a starting point and:
            </p>
            <ul className="list-disc list-inside space-y-2 text-sm">
              <li>Test alternative chunking strategies</li>
              <li>Benchmark different embedding models</li>
              <li>Optimize retrieval parameters</li>
              <li>Run A/B tests and evaluations</li>
              <li>Return the best-performing configuration</li>
            </ul>
            <Alert>
              <AlertDescription>
                Your current configuration will be saved. You can review and accept
                Autopilot's recommendations or keep your original design.
              </AlertDescription>
            </Alert>
            <div className="flex gap-3 justify-end">
              <Button variant="outline" onClick={() => setShowDialog(false)}>
                Cancel
              </Button>
              <Button onClick={handleOptimize}>
                Start Optimization
              </Button>
            </div>
          </div>
        </DialogContent>
      </Dialog>
    </>
  );
}
```

### Autopilot → Designer Flow

**File:** `apps/web/src/components/autopilot/DecisionExplainer.tsx`

```typescript
'use client';

import { Button } from '@/components/ui/button';
import { useRouter } from 'next/navigation';
import { useDesignerStore } from '@/store/designerStore';
import { Eye } from 'lucide-react';

export function DecisionExplainer({ buildResult }) {
  const { setConfig } = useDesignerStore();
  const router = useRouter();

  const handleExplainDecisions = () => {
    // Convert Autopilot config to Designer format
    const designerConfig = {
      cloudProvider: buildResult.config.cloud_provider,
      stages: {
        chunking: buildResult.config.chunking,
        embedding: buildResult.config.embedding,
        vectorStore: buildResult.config.vectorstore,
        retrieval: buildResult.config.retrieval,
        generation: buildResult.config.generation,
        // ... other stages
      },
      metadata: {
        source: 'autopilot',
        buildId: buildResult.id,
        decisions: buildResult.decisions,
      },
    };

    // Load into Designer
    setConfig(designerConfig);
    
    // Navigate to Designer review page
    router.push('/designer/review?source=autopilot');
  };

  return (
    <Button
      variant="outline"
      onClick={handleExplainDecisions}
      className="flex items-center gap-2"
    >
      <Eye className="h-4 w-4" />
      Explain Decisions in Designer
    </Button>
  );
}
```

---

## 📦 Data Models & Types

### Pipeline Configuration Types

**File:** `apps/web/src/types/pipeline.ts`

```typescript
export type CloudProvider = 'aws' | 'gcp' | 'azure' | 'multi-cloud';
export type ModelTier = 'fast' | 'balanced' | 'advanced';

export interface PipelineConfiguration {
  id: string;
  name: string;
  description?: string;
  cloudProvider: CloudProvider;
  
  stages: {
    dataIngestion?: DataIngestionConfig;
    chunking: ChunkingConfig;
    embedding: EmbeddingConfig;
    vectorStore: VectorStoreConfig;
    retrieval: RetrievalConfig;
    reranking?: RerankingConfig;
    generation: GenerationConfig;
    routing?: RoutingConfig;
    memory?: MemoryConfig;
    evaluation?: EvaluationConfig;
  };
  
  metadata: {
    createdAt: string;
    updatedAt: string;
    version: string;
    author?: string;
    source?: 'designer' | 'autopilot' | 'template';
    buildId?: string; // If from autopilot
  };
  
  estimatedCost: CostEstimate;
  estimatedPerformance: PerformanceEstimate;
}

export interface ChunkingConfig {
  strategy: ChunkingStrategy;
  chunkSize: number;
  chunkOverlap: number;
  separators?: string[];
  metadata?: {
    includeSource: boolean;
    includePageNumber: boolean;
    customMetadata?: Record<string, any>;
  };
}

export type ChunkingStrategy =
  | 'fixed-size'
  | 'semantic'
  | 'recursive-character'
  | 'document-based'
  | 'sentence-based'
  | 'paragraph-based'
  | 'markdown-header'
  | 'code-aware';

export interface EmbeddingConfig {
  model: string; // Model ID
  dimensions: number;
  batchSize?: number;
  maxTokens?: number;
  provider: 'openai' | 'cohere' | 'google' | 'huggingface' | 'custom';
}

export interface VectorStoreConfig {
  provider: VectorStoreProvider;
  indexName: string;
  configuration: {
    metric?: 'cosine' | 'euclidean' | 'dot-product';
    replicas?: number;
    shards?: number;
    namespace?: string;
    cloud?: {
      region: string;
      instanceType?: string;
    };
  };
}

export type VectorStoreProvider =
  | 'pinecone'
  | 'weaviate'
  | 'qdrant'
  | 'chroma'
  | 'faiss'
  | 'opensearch'
  | 'vertex-ai-vector-search'
  | 'azure-ai-search'
  | 'pgvector';

export interface RetrievalConfig {
  strategy: RetrievalStrategy;
  topK: number;
  scoreThreshold?: number;
  filters?: MetadataFilter[];
  hybridSearch?: HybridSearchConfig;
}

export type RetrievalStrategy =
  | 'similarity'
  | 'mmr'
  | 'hybrid'
  | 'parent-child'
  | 'multi-query'
  | 'ensemble';

export interface RerankingConfig {
  enabled: boolean;
  model: string;
  topN: number;
  provider: 'cohere' | 'custom';
}

export interface GenerationConfig {
  model: string;
  provider: 'openai' | 'anthropic' | 'google' | 'meta' | 'mistral';
  temperature: number;
  maxTokens: number;
  topP?: number;
  systemPrompt?: string;
  outputFormat?: 'text' | 'json' | 'markdown';
}

export interface CostEstimate {
  embedding: number;
  storage: number;
  retrieval: number;
  reranking: number;
  generation: number;
  total: number;
  perQuery: number;
  perMonth: number;
  currency: 'USD';
  breakdown: CostBreakdown[];
}

export interface CostBreakdown {
  component: string;
  unitCost: number;
  estimatedUsage: number;
  totalCost: number;
  percentage: number;
}
```

### Autopilot Types

**File:** `apps/web/src/types/autopilot.ts`

```typescript
export interface AutopilotBuild {
  id: string;
  status: 'pending' | 'running' | 'complete' | 'failed';
  progress: number;
  currentStage: string;
  iteration: number;
  
  input: {
    documents: string[];
    requirements: BuildRequirements;
  };
  
  stages: Record<string, StageStatus>;
  messages: BuildMessage[];
  
  result?: BuildResult;
  error?: string;
  
  createdAt: string;
  updatedAt: string;
  completedAt?: string;
}

export interface BuildRequirements {
  target_metrics: {
    faithfulness?: number;
    answer_relevance?: number;
    context_precision?: number;
  };
  cloud_provider?: CloudProvider;
  budget_constraint?: number; // Max cost per 1K queries
  latency_requirement?: number; // Max latency in ms
  optimize_for?: 'quality' | 'cost' | 'latency' | 'balanced';
  max_iterations?: number;
}

export interface StageStatus {
  status: 'pending' | 'running' | 'complete' | 'failed';
  startedAt?: string;
  completedAt?: string;
  message?: string;
  data?: any;
}

export interface BuildMessage {
  timestamp: string;
  text: string;
  type: 'info' | 'success' | 'warning' | 'error';
  agent?: string;
}

export interface BuildResult {
  config: PipelineConfiguration;
  metrics: {
    faithfulness: number;
    answer_relevance: number;
    context_precision: number;
    context_recall: number;
  };
  decisions: AgentDecisions;
  deployment: DeploymentInfo;
}

export interface AgentDecisions {
  chunking: {
    strategy: string;
    reasoning: string;
    alternatives_tested: string[];
  };
  embedding: {
    model: string;
    reasoning: string;
    benchmark_results: {
      model: string;
      score: number;
      cost: number;
      latency: number;
    }[];
  };
  retrieval: {
    strategy: string;
    reasoning: string;
    performance: Record<string, number>;
  };
  // ... other decisions
}

export interface DeploymentInfo {
  provider: string;
  endpoint: string;
  status: 'deployed' | 'failed';
  deployedAt?: string;
  healthCheckUrl?: string;
}
```

---

## 💾 Database Schema

### PostgreSQL Schema

**File:** `apps/api/alembic/versions/001_initial_schema.py`

```python
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import JSONB, UUID
import uuid

def upgrade():
    # Users table
    op.create_table(
        'users',
        sa.Column('id', UUID(as_uuid=True), primary_key=True, default=uuid.uuid4),
        sa.Column('email', sa.String(255), unique=True, nullable=False),
        sa.Column('name', sa.String(255)),
        sa.Column('created_at', sa.DateTime, server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime, onupdate=sa.func.now()),
    )
    
    # Projects table
    op.create_table(
        'projects',
        sa.Column('id', UUID(as_uuid=True), primary_key=True, default=uuid.uuid4),
        sa.Column('user_id', UUID(as_uuid=True), sa.ForeignKey('users.id'), nullable=False),
        sa.Column('name', sa.String(255), nullable=False),
        sa.Column('description', sa.Text),
        sa.Column('created_at', sa.DateTime, server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime, onupdate=sa.func.now()),
    )
    
    # Pipeline Configurations table
    op.create_table(
        'pipeline_configs',
        sa.Column('id', UUID(as_uuid=True), primary_key=True, default=uuid.uuid4),
        sa.Column('project_id', UUID(as_uuid=True), sa.ForeignKey('projects.id'), nullable=False),
        sa.Column('name', sa.String(255), nullable=False),
        sa.Column('version', sa.String(50)),
        sa.Column('cloud_provider', sa.String(50)),
        sa.Column('config', JSONB, nullable=False),  # Full pipeline config
        sa.Column('source', sa.String(50)),  # 'designer' | 'autopilot' | 'template'
        sa.Column('build_id', UUID(as_uuid=True)),  # Reference to autopilot_builds if from autopilot
        sa.Column('created_at', sa.DateTime, server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime, onupdate=sa.func.now()),
    )
    
    # Autopilot Builds table
    op.create_table(
        'autopilot_builds',
        sa.Column('id', UUID(as_uuid=True), primary_key=True, default=uuid.uuid4),
        sa.Column('project_id', UUID(as_uuid=True), sa.ForeignKey('projects.id'), nullable=False),
        sa.Column('status', sa.String(50), nullable=False),  # pending, running, complete, failed
        sa.Column('progress', sa.Integer, default=0),
        sa.Column('current_stage', sa.String(100)),
        sa.Column('iteration', sa.Integer, default=1),
        sa.Column('requirements', JSONB, nullable=False),
        sa.Column('stages', JSONB),  # Stage statuses
        sa.Column('messages', JSONB),  # Build messages log
        sa.Column('result', JSONB),  # Final result
        sa.Column('error', sa.Text),
        sa.Column('created_at', sa.DateTime, server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime, onupdate=sa.func.now()),
        sa.Column('completed_at', sa.DateTime),
    )
    
    # Evaluation Runs table
    op.create_table(
        'evaluation_runs',
        sa.Column('id', UUID(as_uuid=True), primary_key=True, default=uuid.uuid4),
        sa.Column('config_id', UUID(as_uuid=True), sa.ForeignKey('pipeline_configs.id')),
        sa.Column('build_id', UUID(as_uuid=True), sa.ForeignKey('autopilot_builds.id')),
        sa.Column('metrics', JSONB, nullable=False),  # RAGAS metrics
        sa.Column('test_set_size', sa.Integer),
        sa.Column('created_at', sa.DateTime, server_default=sa.func.now()),
    )
    
    # Deployments table
    op.create_table(
        'deployments',
        sa.Column('id', UUID(as_uuid=True), primary_key=True, default=uuid.uuid4),
        sa.Column('config_id', UUID(as_uuid=True), sa.ForeignKey('pipeline_configs.id'), nullable=False),
        sa.Column('provider', sa.String(50), nullable=False),  # docker, aws, gcp, azure
        sa.Column('status', sa.String(50), nullable=False),  # deploying, deployed, failed
        sa.Column('endpoint', sa.String(500)),
        sa.Column('deployment_info', JSONB),
        sa.Column('created_at', sa.DateTime, server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime, onupdate=sa.func.now()),
    )
    
    # Create indexes
    op.create_index('idx_projects_user_id', 'projects', ['user_id'])
    op.create_index('idx_pipeline_configs_project_id', 'pipeline_configs', ['project_id'])
    op.create_index('idx_autopilot_builds_project_id', 'autopilot_builds', ['project_id'])
    op.create_index('idx_autopilot_builds_status', 'autopilot_builds', ['status'])

def downgrade():
    op.drop_table('deployments')
    op.drop_table('evaluation_runs')
    op.drop_table('autopilot_builds')
    op.drop_table('pipeline_configs')
    op.drop_table('projects')
    op.drop_table('users')
```

---

## 🚀 API Specification

### Designer Endpoints

```yaml
# Designer API Endpoints

POST /api/designer/config
  Description: Save or update pipeline configuration
  Body:
    - config: PipelineConfiguration
  Response:
    - id: string
    - config: PipelineConfiguration

GET /api/designer/config/{id}
  Description: Load pipeline configuration
  Response:
    - config: PipelineConfiguration

POST /api/designer/export
  Description: Export configuration as code
  Body:
    - config: PipelineConfiguration
    - format: 'python' | 'yaml' | 'terraform'
  Response:
    - code: string
    - filename: string

POST /api/designer/cost
  Description: Calculate cost estimate
  Body:
    - config: PipelineConfiguration
  Response:
    - cost: CostEstimate

POST /api/designer/deploy
  Description: Deploy pipeline
  Body:
    - config: PipelineConfiguration
    - provider: string
  Response:
    - deployment_id: string
    - status: string
```

### Autopilot Endpoints

```yaml
# Autopilot API Endpoints

POST /api/autopilot/build
  Description: Start new autopilot build
  Body:
    - documents: File[]
    - requirements: BuildRequirements
    - base_config?: PipelineConfiguration  # Optional, from Designer
  Response:
    - build_id: string
    - status: string

GET /api/autopilot/build/{id}
  Description: Get build status and progress
  Response:
    - build: AutopilotBuild

GET /api/autopilot/build/{id}/stream
  Description: Server-Sent Events stream for real-time updates
  Response:
    - SSE stream with build updates

POST /api/autopilot/build/{id}/cancel
  Description: Cancel running build
  Response:
    - success: boolean

GET /api/autopilot/build/{id}/result
  Description: Get final build result
  Response:
    - result: BuildResult
```

### Shared Endpoints

```yaml
# Projects

POST /api/projects
  Description: Create new project
  Body:
    - name: string
    - description?: string
  Response:
    - project: Project

GET /api/projects
  Description: List user projects
  Response:
    - projects: Project[]

GET /api/projects/{id}
  Description: Get project details
  Response:
    - project: Project
    - configs: PipelineConfiguration[]
    - builds: AutopilotBuild[]

# Templates

GET /api/templates
  Description: List available templates
  Response:
    - templates: Template[]

POST /api/templates/{id}/apply
  Description: Apply template to project
  Response:
    - config: PipelineConfiguration

# Evaluation

POST /api/evaluation/run
  Description: Run RAGAS evaluation
  Body:
    - config_id: string
    - test_set?: dict[]
  Response:
    - run_id: string
    - metrics: dict

# Deployment

POST /api/deployment/deploy
  Description: Deploy pipeline
  Body:
    - config_id: string
    - provider: string
  Response:
    - deployment_id: string

GET /api/deployment/{id}/status
  Description: Check deployment status
  Response:
    - status: string
    - endpoint?: string
```

---

## 🧪 Development Roadmap

### Phase 1: Foundation (Weeks 1-3)

**Week 1: Project Setup**
- [ ] Initialize monorepo structure
- [ ] Setup Next.js frontend
- [ ] Setup FastAPI backend
- [ ] Configure PostgreSQL + Redis + Qdrant
- [ ] Create Docker Compose dev environment
- [ ] Setup CI/CD pipelines

**Week 2: Shared Core**
- [ ] Create data models (TypeScript + Python)
- [ ] Build model catalogs (JSON files)
- [ ] Implement cost calculator
- [ ] Create Mermaid diagram generator
- [ ] Setup database migrations
- [ ] Build authentication (basic)

**Week 3: Core Services**
- [ ] Document ingestion service
- [ ] Chunking service (7 strategies)
- [ ] Embedding service (model wrappers)
- [ ] Vector DB service (Qdrant integration)
- [ ] Retrieval service
- [ ] Generation service (LLM integrations)

---

### Phase 2: Designer Mode (Weeks 4-8)

**Week 4: UI Foundation**
- [ ] Landing page
- [ ] Designer mode layout
- [ ] Stage navigator
- [ ] Cloud provider selector
- [ ] Project management UI

**Week 5: Pipeline Stages (Part 1)**
- [ ] Chunking configuration UI
- [ ] Embedding selector UI
- [ ] Vector store selector UI
- [ ] Integration with backend APIs

**Week 6: Pipeline Stages (Part 2)**
- [ ] Retrieval configuration UI
- [ ] Reranking selector UI
- [ ] Generation model selector UI
- [ ] Routing logic builder UI

**Week 7: Visualization & Export**
- [ ] Pipeline visualizer (Mermaid)
- [ ] Cost estimator component
- [ ] Performance estimator
- [ ] Code exporters (Python, YAML, Terraform)

**Week 8: Designer Polish**
- [ ] Review page
- [ ] Template system
- [ ] Save/load configurations
- [ ] "Optimize This" button → Autopilot integration
- [ ] Testing & bug fixes

---

### Phase 3: Autopilot Mode (Weeks 9-14)

**Week 9: LangGraph Setup**
- [ ] LangGraph agent framework
- [ ] Agent state management
- [ ] Agent tools library
- [ ] Orchestrator skeleton

**Week 10: Core Agents (Part 1)**
- [ ] Document Analyst Agent
- [ ] Chunking Optimizer Agent
- [ ] Testing framework for agents
- [ ] Integration with core services

**Week 11: Core Agents (Part 2)**
- [ ] Embedding Tester Agent
- [ ] Retrieval Optimizer Agent
- [ ] RAGAS evaluation integration
- [ ] MLflow experiment tracking

**Week 12: Evaluation & Iteration**
- [ ] Evaluation Agent
- [ ] Failure analysis logic
- [ ] Iteration decision logic
- [ ] Test set generation

**Week 13: Deployment & UI**
- [ ] Deployment Agent
- [ ] Autopilot mode UI
- [ ] Document uploader
- [ ] Requirements form
- [ ] Build progress monitor

**Week 14: Autopilot Polish**
- [ ] Agent activity feed
- [ ] Metrics dashboard
- [ ] Decision explainer
- [ ] Results summary
- [ ] Testing & bug fixes

---

### Phase 4: Integration (Weeks 15-17)

**Week 15: Bidirectional Flow**
- [ ] Designer → Autopilot handoff
- [ ] Autopilot → Designer visualization
- [ ] Configuration format conversion
- [ ] Data synchronization

**Week 16: Deployment Engine**
- [ ] Docker packaging
- [ ] Kubernetes manifest generation
- [ ] AWS deployment (Bedrock, ECS)
- [ ] GCP deployment (Vertex AI, GKE)
- [ ] Azure deployment (OpenAI, AKS)

**Week 17: Monitoring & Analytics**
- [ ] Usage analytics
- [ ] Cost tracking
- [ ] Performance monitoring
- [ ] Error tracking
- [ ] Admin dashboard

---

### Phase 5: Polish & Launch (Weeks 18-20)

**Week 18: Testing**
- [ ] End-to-end tests (Designer flow)
- [ ] End-to-end tests (Autopilot flow)
- [ ] Integration tests
- [ ] Load testing
- [ ] Security audit

**Week 19: Documentation**
- [ ] User guides (Designer mode)
- [ ] User guides (Autopilot mode)
- [ ] API documentation
- [ ] Video tutorials
- [ ] FAQ

**Week 20: Launch Prep**
- [ ] Performance optimization
- [ ] Bug fixes
- [ ] Production deployment
- [ ] Launch marketing materials
- [ ] Community setup (Discord, GitHub)

---

## ✅ Success Metrics

### Technical Metrics

**Designer Mode:**
- [ ] Pipeline completion rate >70%
- [ ] Average time to complete: <15 minutes
- [ ] Cost calculator accuracy: ±10%
- [ ] Code export success rate: >95%

**Autopilot Mode:**
- [ ] Build success rate: >85%
- [ ] Average build time: <10 minutes
- [ ] Target metrics achieved: >80% of builds
- [ ] Deployment success rate: >90%

**Shared:**
- [ ] Page load time: <2 seconds
- [ ] API response time: <500ms (p95)
- [ ] Uptime: 99.5%
- [ ] Zero critical bugs

---

### Business Metrics

**Month 1 (Launch):**
- [ ] 500 signups
- [ ] 100 completed pipelines
- [ ] 50 autopilot builds
- [ ] 10 deployments

**Month 3:**
- [ ] 2,000 active users
- [ ] 500 completed pipelines
- [ ] 200 autopilot builds
- [ ] 50 paid users

**Month 6:**
- [ ] 5,000 active users
- [ ] 2,000 completed pipelines
- [ ] 500 autopilot builds
- [ ] 200 paid users
- [ ] $10K MRR

---

### User Satisfaction

- [ ] NPS Score: >50
- [ ] Designer mode rating: 4.5+ stars
- [ ] Autopilot mode rating: 4.8+ stars
- [ ] Template usage: >60% of new users
- [ ] Mode switching rate: >30% (indicates integration value)

---

## 🎉 Conclusion

This specification provides everything needed to build **RAG Studio** - the complete RAG platform that combines manual design with autonomous optimization.

**Key Deliverables:**
1. ✅ Designer Mode - Visual pipeline builder
2. ✅ Autopilot Mode - Autonomous RAG builder
3. ✅ Bidirectional Integration - Seamless flow between modes
4. ✅ Production Deployment - Docker + K8s ready
5. ✅ 20-week roadmap to MVP

**Unique Value:**
- **Only platform** offering both manual and autonomous RAG building
- **Educational** for beginners, **powerful** for experts
- **Production-ready** output from both modes
- **Seamless integration** unlocks hybrid workflows

**Next Steps:**
1. Review and approve specification
2. Assemble development team
3. Set up development environment
4. Begin Phase 1 (Foundation)
5. Ship MVP in 20 weeks

**Ready to build the future of RAG development? Let's go! 🚀**
