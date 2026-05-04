# Project system design evolution — Phase 7 (Autopilot frontend)

> **Append-only companion.** Phase 7 adds the **Autopilot browser experience**: real **uploads**, **requirements** editing, **live build** observation (SSE + poll fallback), **observability** (feed, charts), **typed results** and **Designer handoff**, then a **multi-route shell** and **server history list**. This file moves from **single-page wizard** beginnings to **multi-page IA** at the end of the phase.
>
> Canonical cumulative log (all phases): [`PROJECT_SYSTEM_DESIGN_EVOLUTION.md`](./PROJECT_SYSTEM_DESIGN_EVOLUTION.md).

---

## Design level 1 — Real corpus path (after P7-1 · Document Uploader)

**Goal:** Replace opaque **`documentIds`** with **multipart upload** to **object storage** and **Zustand** metadata the build request can reference.

**`POST /api/autopilot/upload`** validates **project ownership**, enforces caps from **`Settings`**, writes **MinIO** keys under **`autopilot/{user}/{project}/…`**. **`DocumentUploader`** lists **`GET /api/projects`**, calls **`postFormData`**, stores **`objectId`** rows in **`useAutopilotStore`**.

```mermaid
flowchart LR
  subgraph Web["Next.js Autopilot UI"]
    DU[DocumentUploader]
    Z[(Zustand persist)]
  end
  subgraph API["FastAPI /api/autopilot"]
    UP[POST /upload]
    OWN[Project ownership]
  end
  subgraph Obj["MinIO / S3"]
    M[(object bucket)]
  end
  DU -->|list| PG[(PostgreSQL projects)]
  DU -->|multipart| UP
  UP --> OWN
  OWN --> PG
  UP -->|put_object| M
  UP -->|201 documents| DU
  DU --> Z
```

```mermaid
sequenceDiagram
  participant U as UI
  participant API as FastAPI
  participant S3 as MinIO
  participant JOB as Celery

  U->>API: POST /upload
  API->>S3: put_object
  API-->>U: objectIds
  Note over U: P7-2+ requirements
  U->>API: POST /build documentIds
  API->>JOB: run_pipeline_build
```

**Evolution:** Phase 6 APIs could enqueue builds; Phase 7-1 supplies **real bytes** and stable **object keys** for those **`documentIds`**.

---

## Design level 2 — Authoritative constraints (after P7-2 · Requirements Form)

**Goal:** **`StartBuildRequest.requirements`** is edited in the UI with **Zod** validation before any **Start** action.

**`RequirementsForm`** binds sliders and cards to **`useAutopilotStore.requirements`** (persisted): **target metrics**, **`optimizeFor`**, budget/latency, **cloud provider** from catalog, **`maxIterations`**.

```mermaid
flowchart TB
  subgraph Web["Autopilot wizard"]
    RF[RequirementsForm]
    Zod[BuildRequirementsSchema]
    Z[(useAutopilotStore.requirements)]
  end
  RF -->|patchRequirements| Z
  RF -.->|safeParse| Zod
```

```mermaid
sequenceDiagram
  participant U as UI
  participant Z as Zustand
  participant API as POST /build

  U->>Z: persist requirements
  Note over U,Z: P7-3 wires Start
  U->>API: projectId documentIds requirements baseConfig
```

**Evolution:** **Defaults** become **operator-authored** constraints aligned with the backend schema.

---

## Design level 3 — Closed-loop run (after P7-3 · Build Progress Monitor)

**Goal:** **Start**, **Cancel**, and **observe** a build from the product UI with **resilient transport**.

**`BuildProgressMonitor`** calls **`POST /api/autopilot/build`**, **`useAutopilotBuildSubscription`** uses **SSE** **`…/stream`** then **poll** **`GET …/build/{id}`**. **`autopilot-build-status.ts`** merges **`BuildStatusResponse`** into **`AutopilotBuild`** while preserving **`input`**.

```mermaid
flowchart TB
  subgraph UI["Wizard"]
    BPM[BuildProgressMonitor]
    H[useAutopilotBuildSubscription]
    Z[(useAutopilotStore.builds)]
  end
  subgraph API["FastAPI P6-9"]
    S[GET …/stream]
    G[GET …/build]
    B[POST …/build]
    C[POST …/cancel]
  end
  BPM --> B
  BPM --> C
  BPM --> H
  H -->|EventSource| S
  H -->|interval| G
  H -->|upsertBuild| Z
```

```mermaid
sequenceDiagram
  participant M as BuildProgressMonitor
  participant API as FastAPI
  participant ES as SSE / poll
  participant Z as Zustand

  M->>API: POST /build
  API-->>M: 202 buildId
  M->>Z: upsertBuild seed
  M->>ES: subscribe
  loop Until terminal
    API-->>ES: BuildStatusResponse
    ES->>Z: merge stages messages progress
  end
```

**Evolution:** The wizard is **operationally complete** for enqueue + watch; further levels add **depth** (logs, charts, reports, navigation).

---

## Design level 4 — Agent log UX (after P7-4 · Agent Activity Feed)

**Goal:** **`messages`** are **searchable**, **filterable**, and **exportable** without devtools.

**`AgentActivityFeed`** reads **`builds[activeBuildId].messages`**, supports agent/type filters, **JSON / text export**, **smart scroll-to-bottom**.

```mermaid
flowchart TB
  subgraph Page["Autopilot wizard"]
    BPM[BuildProgressMonitor]
    AAF[AgentActivityFeed]
  end
  subgraph Client["Browser"]
    H[useAutopilotBuildSubscription]
    Z[(builds + activeBuildId)]
  end
  subgraph API["FastAPI"]
    S[SSE]
    G[poll GET]
  end
  BPM --> H
  H --> S
  H --> G
  H -->|upsertBuild| Z
  AAF -->|read filter export| Z
```

**Evolution:** Observability moves from **progress bars only** to **first-class operator logs**.

---

## Design level 5 — Structured metrics slice (after P7-5 · Metrics Dashboard)

**Goal:** Chart **quality / embedding / retrieval** signals while **`result`** may still be **opaque** orchestrator JSON.

**`extract_dashboard_metrics`** projects **`result.stage_outputs`** → **`BuildStatusResponse.dashboard_metrics`**. **`MetricsDashboard`** + **Recharts** render on each poll/SSE tick.

```mermaid
flowchart LR
  subgraph Worker["Celery"]
    G[LangGraph]
    R[(result JSON)]
  end
  subgraph API["build_status_response"]
    E[extract_dashboard_metrics]
    B[BuildStatusResponse]
  end
  subgraph Web["MetricsDashboard"]
    RC[Recharts]
  end
  G --> R
  R --> E
  E --> B
  B -->|SSE or poll| RC
```

```mermaid
sequenceDiagram
  participant W as Worker
  participant DB as PostgreSQL
  participant API as GET build / SSE
  participant Z as Zustand
  participant D as MetricsDashboard

  W->>DB: persist stage_outputs
  API->>DB: read row
  API->>API: extract_dashboard_metrics
  API-->>Z: dashboardMetrics
  Z->>D: charts + SLO cards
```

**Evolution:** Operators see **SLO-aligned charts** without waiting for a normalised **`BuildResult`**.

---

## Design level 6 — Typed build report + Designer (after P7-6 · Decision Explainer & Results)

**Goal:** **`BuildStatusResponse.result`** validates as **`BuildResultSchema`** for **metric cards**, **JSON download**, **DecisionExplainer**, and **Open in Designer**.

**`compose_build_result_payload`** merges typed **`config`**, **`metrics`**, **`decisions`**, **`deployment`**, **`total_iterations`** onto **`autopilot_builds.result`**. **`ResultsSummary`** + **`DecisionExplainer`** consume **`build.result`**.

```mermaid
flowchart TB
  subgraph Worker["Celery run_pipeline_build"]
    G[LangGraph]
    C[compose_build_result_payload]
    R[(autopilot_builds.result)]
  end
  subgraph API["GET /api/autopilot/build/{id}"]
    V[_optional_typed_result]
    B[BuildStatusResponse.result]
  end
  subgraph Web["apps/web"]
    RS[ResultsSummary]
    DE[DecisionExplainer]
    Z[(useAutopilotStore)]
  end
  G --> C
  C --> R
  R --> V
  V --> B
  B -->|SSE / poll| Z
  Z --> RS
  Z --> DE
```

```mermaid
sequenceDiagram
  participant U as User
  participant DE as DecisionExplainer
  participant ZD as designerStore
  participant R as Next.js router

  U->>DE: Open in Designer
  DE->>ZD: loadPipeline result.config
  DE->>R: /designer/review?source=autopilot
```

**Evolution:** Autopilot output becomes **interchangeable** with Designer **`PipelineConfiguration`** for review and iteration.

---

## Design level 7 — Information architecture + history API (after P7-7 · Autopilot Entry & History Pages)

**Goal:** **Discoverable routes**, **server-backed build lists**, and **deep links** back into the wizard—without changing the **rich** **`GET /build/{id}`** contract.

**`AutopilotShell`** wraps **`/autopilot`** (overview), **`/autopilot/new`** (full wizard), **`/autopilot/history`** (paginated **`GET /api/autopilot/builds`**), **`/autopilot/projects`** (backend project picker). **`?build=`** + **`?project=`** on **`/autopilot/new`** hydrates store via one-shot **GET** when needed.

```mermaid
flowchart LR
  subgraph Shell["AutopilotShell"]
    O["/autopilot"]
    N["/autopilot/new"]
    H["/autopilot/history"]
    P["/autopilot/projects"]
  end
  O --> N
  O --> H
  O --> P
  H -->|"Open ?build=&project="| N
```

```mermaid
sequenceDiagram
  participant UI as History page
  participant API as GET /api/autopilot/builds
  participant DB as PostgreSQL

  UI->>API: page optional project_id
  API->>DB: JOIN builds projects user scoped
  DB-->>API: rows + total
  API-->>UI: AutopilotBuildListResponse
```

```mermaid
flowchart TB
  subgraph Simple["Earlier Phase 7"]
    One["Single /autopilot URL"]
    W[All widgets stacked]
  end
  subgraph Advanced["After P7-7"]
    M[Multi-route App Router]
    L[AutopilotShell nav]
    API[GET /builds list]
  end
  One --> M
  W --> L
  M --> API
```

**Evolution:** **Everything on one scroll** becomes a **product surface** with **overview**, **wizard**, **history**, and **project** contexts; list API supports **fresh sessions** and **ops dashboards**.

---

*Append new “Design level” sections here for any Phase 7 follow-on (e.g. auth-gated history, build comparison UI).*
