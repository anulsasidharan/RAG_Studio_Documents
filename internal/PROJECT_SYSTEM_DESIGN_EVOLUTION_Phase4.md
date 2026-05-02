# Project system design evolution — Phase 4 (Designer mode backend)

> Part of the [master index](./PROJECT_SYSTEM_DESIGN_EVOLUTION.md).

---

## Phase 4 — Designer mode backend (API milestones, appended 2026-05-02)

The API is the **system of record** for workspaces (“projects”) and **Designer pipeline configurations**. CRUD is **user-scoped** via `X-User-ID` until JWT (Phase 12). Projects use **soft delete**; pipeline configs use **hard delete** with FK cascade where defined.

### After P4-1 · Projects API

```mermaid
flowchart TB
  subgraph clients [Clients]
    FE[Next.js]
  end
  subgraph apps_api [apps/api]
    PR[/api/projects REST/]
    PS[ProjectService]
    HC[Health / utilities / jobs…]
  end
  DB[(Postgres)]
  FE -->|Bearer-less; X-User-ID optional| PR
  PR --> PS
  PS --> DB
  HC --> DB
```

| Topic | Decision |
|--------|----------|
| Identity | Header `X-User-ID` + `default_user_id` in settings until P12 JWT |
| Delete | Soft delete via `deleted_at` |
| Detail payload | Summarized configs/builds to avoid huge JSON in list/detail |
| Persistence | Async SQLAlchemy; JSON columns portable across SQLite (tests) and Postgres |

### After P4-2 · Designer Config API

Designer persists **`PipelineConfigurationSchema`** in `pipeline_configs.config` (full JSON), with indexed scalar columns for listing. Access validates **project ownership** via join to `projects`.

```mermaid
flowchart TB
  subgraph clients [Clients]
    FE["Next.js"]
  end

  subgraph apps_api ["apps/api"]
    PR["/api/projects/"]
    DS["/api/designer/config"]
    PS["ProjectService"]
    DG["DesignerService"]
    HC["Health / utilities / jobs"]
  end

  DB[(Postgres)]

  FE --> PR
  FE --> DS
  PR --> PS
  DS --> DG
  PS --> DB
  DG --> DB
  HC --> DB
```

| Topic | Decision |
|--------|----------|
| Aggregate root | `PipelineConfig.project_id` → `projects.id` |
| Writes | `save_config` assigns server UUID; merges display `name` / `description` |
| Reads | DB timestamps injected into `metadata` on `SaveConfigResponse` |
| Delete | Hard row delete (evaluations/deployments cascade per schema) |

### Planned later in Phase 4

Cost API, export API, templates API — same tier; reuse projects + configs as aggregates.

```mermaid
flowchart LR
  FE[Designer UI]
  subgraph phase4 [Remaining Phase 4]
    P[/projects/]
    D[designer config done]
    C[/cost · export · templates/]
  end
  FE --> P
  FE --> D
  FE --> C
  P --> DB[(Postgres)]
  D --> DB
  C --> DB
```

---

## Looking ahead (compact)

Later phases add Designer UX depth (Phase 5), LangGraph autopilot agents + streaming APIs (Phases 6–7), evaluation/deployment endpoints (Phase 8), MLflow (Phase 9), automated testing gates (Phase 10), observability (Phase 11), and production hardening (Phase 12). Each increment extends this document with diagrams focused on new boundaries (auth, metrics, deployment planes).

---

## Later phases (preview)

- **Phase 5+:** Designer UI consumes Phase 4 endpoints end-to-end.
- **Phase 6–7:** Autopilot agents + web streaming — builds attach to `projects` and persist in existing tables.
- **Phase 12:** Replace header user with JWT and tighten row-level security.

*Append new sections at the end of this file when milestones land; preserve the full P0–P2 archive and later milestones above.*

---

## Phase 4 — P4-3 Cost Calculation API (Designer)

**Scope:** Stateless cost estimation for a validated `PipelineConfigurationSchema`. No Postgres read/write; pricing data is file-backed (`apps/api/catalogs/pricing.json` with fallback to repo `data/pricing.json`). This sub-phase adds the **estimation plane** next to existing **config persistence** (P4-2) and **projects** (P4-1).

### Architecture (level: Designer API + pricing catalog)

```mermaid
flowchart LR
  subgraph client [Client]
    FE[Designer UI / tools]
  end
  subgraph api [FastAPI]
    DC["POST /api/designer/cost"]
    CS[CostService]
    CE[CostEstimator]
  end
  subgraph data [Data]
    PJ[pricing.json]
  end
  FE -->|CostRequest JSON| DC
  DC --> CS
  CS --> CE
  CE --> PJ
  DC -->|CostEstimateSchema| FE
```

### Behaviour

| Input | Source |
|--------|--------|
| Pipeline stages | `CostRequest.config` — chunking, embedding model/dims, vector store provider, retrieval (strategy, `top_k`, multi-query config), reranking, generation |
| Volume assumptions | `queries_per_month`, `documents_count`, `avg_document_tokens` |
| Defaults | `pricing.json` → `assumptions` (avg input/output tokens per query) |

| Output | Meaning |
|--------|---------|
| `per_query` | Variable USD/query (embedding + generation + rerank + amortized retrieval reads) |
| `per_month` | Same components × scale + monthly vector storage |
| `breakdown` | Five rows: `embedding`, `vector_storage`, `retrieval_ops`, `reranking`, `generation` with percentages |

### Relation to utilities

`POST /api/utilities/cost` uses the same `CostEstimator` (via `app.core.utilities.cost` re-export). Designer-specific route is **`/api/designer/cost`** for product and OpenAPI grouping under `designer`.

### Next sub-phases in Phase 4

Export API, templates API — can call cost estimator for previews; no schema change required.

```mermaid
flowchart TB
  subgraph p4 [Phase 4 Designer backend]
    P1[Projects API]
    P2[Config API]
    P3[Cost API]
    P4[Export API]
    P5[Templates API]
  end
  P1 --> DB[(Postgres)]
  P2 --> DB
  P3 --> CAT[pricing.json]
  P4 --> DB
  P5 --> DB
  P5 --> TJ[templates.json]
```

---

## Phase 4 — P4-4 Export API (Designer)

**Scope:** Stateless code and manifest generation from a validated `PipelineConfigurationSchema`. Mirrors the P3-5 frontend generators so the Designer UI (P5-12) can call the API instead of generating only client-side. No Postgres; no pricing file.

### Architecture (export plane)

```mermaid
flowchart LR
  subgraph client [Client]
    FE[Designer UI / scripts]
  end
  subgraph api [FastAPI]
    EX["POST /api/designer/export"]
    ES[ExportService]
    G1[python_export]
    G2[yaml_export]
    G3[terraform_export]
    G4[docker_k8s_export]
  end
  FE -->|ExportRequest JSON| EX
  EX --> ES
  ES --> G1
  ES --> G2
  ES --> G3
  ES --> G4
  ES -->|ExportResponse| FE
```

### Phase 4 Designer backend (updated)

```mermaid
flowchart TB
  subgraph p4 [Phase 4 Designer backend]
    P1[Projects API]
    P2[Config API]
    P3[Cost API]
    P4[Export API]
    P5[Templates API]
  end
  P1 --> DB[(Postgres)]
  P2 --> DB
  P3 --> CAT[pricing.json]
  P4 -.->|stateless| GEN[Text generators]
  P5 --> DB
  P5 --> TJ[templates.json]
```

**Note:** The earlier diagram showed Export → DB; export is **stateless** — generators only. Templates API remains DB-backed.

### Next sub-phase in Phase 4

Templates API — list/apply `data/templates.json` and create `PipelineConfig` rows.

---

## Phase 4 · P4-5 Templates API (completed)

Designer backend can expose curated pipeline presets from the shared JSON catalog and materialize them as first-class saved configurations.

### Behaviour

- **Read paths** (`GET /api/templates`, `GET /api/templates/{id}`): load and validate `data/templates.json` (or `TEMPLATES_CATALOG_PATH` / optional `apps/api/catalogs/templates.json` mirror). No database reads for listing.
- **Apply** (`POST /api/templates/{id}/apply`): resolve template by id, build a `SaveConfigRequest` (optional `name` / `description` overrides), call `DesignerService.save_config` — same persistence rules as manual Designer create. Response extends `SaveConfigResponse` with `templateId` so clients know which preset was used.
- **Data fix:** `customer-support` template routing rules were aligned with `RoutingRuleSchema` (`condition` ∈ `keyword` | `query-length` | `semantic-complexity`, `targetModel`, optional `threshold` / `keywords`) so all templates validate as `PipelineConfigurationSchema`.

### Mermaid — Templates API (Phase 4)

```mermaid
flowchart LR
  subgraph read [Catalog read — stateless]
    FE1[Client / P5 gallery]
    TGET[GET /api/templates]
    TONE[GET /api/templates/id]
    TSVC[TemplateService.list / get]
    TJ[(templates.json)]
  end
  subgraph apply [Apply — persists]
    FE2[Client]
    TAP[POST .../apply]
    TAPL[TemplateService.apply]
    DS[DesignerService.save_config]
    DB[(Postgres PipelineConfig)]
  end
  FE1 --> TGET --> TSVC --> TJ
  FE1 --> TONE --> TSVC
  FE2 --> TAP --> TAPL --> DS --> DB
```

### Phase 4 Designer backend — revised summary

```mermaid
flowchart TB
  subgraph p4 [Phase 4 Designer backend]
    P1[Projects API]
    P2[Config API]
    P3[Cost API]
    P4[Export API]
    P5[Templates API]
  end
  P1 --> DB[(Postgres)]
  P2 --> DB
  P3 --> CAT[pricing.json]
  P4 -.->|stateless| GEN[Text generators]
  P5 -->|GET| TJ[templates.json]
  P5 -->|apply| DB
```

**Correction:** List/get template endpoints are **file-backed**, not DB-backed. Only **apply** writes through `DesignerService` to Postgres.

---

