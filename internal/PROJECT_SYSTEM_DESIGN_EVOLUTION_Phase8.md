# Project system design evolution — Phase 8 (Designer ↔ Autopilot integration)

> **Scope.** Phase 8 connects **Designer mode** and **Autopilot mode** through a **shared pipeline config contract**, **HTTP APIs** for **evaluation** and **deployment**, and **Zustand** handoff state on the web app. This document evolves from a **siloed two-mode** picture (start of phase) to the **integrated platform** at phase completion (**P8-1 → P8-4**).

---

## Design level 0 — Siloed modes (before Phase 8)

**Designer** and **Autopilot** share JSON catalogs and types, but there is **no first-class handoff**: starting an Autopilot build does not carry a Designer draft as a structured baseline, and finished builds are not imported back into **`useDesignerStore`** as a reviewable draft. **Evaluation** and **deployment** are not yet exposed as cohesive REST surfaces under a single integration story.

```mermaid
flowchart TB
  subgraph Designer["Designer mode"]
    D[Manual draft in Zustand]
    DRev[Review / export only]
    D --> DRev
  end
  subgraph Autopilot["Autopilot mode"]
    U[Upload + requirements]
    B[Build + agents]
    R[Result JSON]
    U --> B --> R
  end
  Designer -.->|"no structured bridge"| Autopilot
```

---

## Design level 1 — Designer → Autopilot handoff (after **P8-1**)

The user can open **“Optimize with Autopilot”** from **`/designer/review`**. **`useAutopilotStore.startFromDesigner(draft)`** stores the Designer **`PipelineConfiguration`** as **`baseConfig`**. **`BuildProgressMonitor`** submits **`POST /api/autopilot/build`** with **`baseConfig`** (camelCase) so the backend persists **`requirements["base_config"]`** on the build row for the LangGraph orchestrator. **Document uploads remain mandatory** (`document_ids`); the handoff carries **hyperparameters**, not corpus binaries.

```mermaid
sequenceDiagram
  participant U as User
  participant Rev as Designer review page
  participant AS as useAutopilotStore
  participant API as POST /api/autopilot/build
  participant DB as autopilot_builds

  U->>Rev: Confirm Optimize with Autopilot
  Rev->>AS: startFromDesigner(draft)
  AS->>API: body includes baseConfig
  API->>DB: requirements.base_config persisted
  Note over DB: Orchestrator uses draft as starting point
```

```mermaid
flowchart LR
  subgraph Web["apps/web"]
    DS[(useDesignerStore draft)]
    AP[(useAutopilotStore)]
  end
  subgraph API["apps/api"]
    B["POST /api/autopilot/build"]
  end
  DS -->|"baseConfig on start"| AP
  AP --> B
```

---

## Design level 2 — Autopilot → Designer visualization (after **P8-2**)

Completed builds expose **“Open in Designer”**: **`applyAutopilotBuildResult`** copies **`BuildResult.config`** into **`draft`**, sets **`metadata.source: autopilot`**, **`metadata.buildId`**, and **`autopilotImportSnapshot`** (metrics, iterations). **`/designer/review?source=autopilot`** renders **`AutopilotDesignerImportBanner`** alongside the existing **PipelineVisualizer**, **CostEstimator**, and **CodeExporter** — same visualization stack as a manual design, with Autopilot provenance visible.

```mermaid
flowchart TB
  subgraph AutopilotDone["Build complete"]
    Res[BuildResult.config + metrics]
  end
  subgraph DesignerUI["Designer review"]
    Draft[(useDesignerStore draft)]
    Vis[PipelineVisualizer + cost + export]
    Ban[AutopilotDesignerImportBanner]
  end
  Res -->|"applyAutopilotBuildResult"| Draft
  Draft --> Vis
  Draft --> Ban
```

```mermaid
flowchart LR
  subgraph Modes["Phase 8 bidirectional UX"]
    D2A[Designer → Autopilot<br/>baseConfig]
    A2D[Autopilot → Designer<br/>import snapshot]
  end
  D2A <-->|Shared PipelineConfiguration| A2D
```

---

## Design level 3 — Evaluation API as shared backend (after **P8-3**)

**RAGAS-style evaluation** is available over HTTP for **saved pipeline configs**: **`POST /api/evaluation/run`**, **`GET /api/evaluation/run/{id}`**, **`GET /api/evaluation/runs?config_id=`**, **`POST /api/evaluation/compare`**. **`EvaluationService`** joins **`evaluation_runs` → `pipeline_configs` → `projects`** with **`X-User-ID`** scoping. Runs can feed **comparison** of two configs (paired run ids or fresh runs on a shared synthetic set). This binds **Designer** and **Autopilot** outputs to the **same evaluation surface** once a config id exists in the database.

```mermaid
flowchart TB
  subgraph Clients["Future / current callers"]
    FE[Next.js Designer & Autopilot UIs]
    Script[Scripts & CI]
  end
  subgraph EvalAPI["/api/evaluation/*"]
    Run[POST /run]
    One[GET /run/...]
    List[GET /runs]
    Cmp[POST /compare]
  end
  subgraph Data["PostgreSQL"]
    ER[(evaluation_runs)]
    PC[(pipeline_configs)]
  end
  FE --> EvalAPI
  Script --> EvalAPI
  Run --> ER
  One --> ER
  List --> ER
  Cmp --> PC
```

```mermaid
flowchart LR
  subgraph P8_3["P8-3 service layer"]
    ES[EvaluationService]
    GE[EvaluationEngine + RAGAS]
    GR[run_guarded_rag_query]
  end
  ES --> GR
  GR --> GE
```

---

## Design level 4 — Deployment API + async worker (after **P8-4**)

**Deployments** are **first-class resources**: **`POST /api/deployment/deploy`** creates a **`deployments`** row and enqueues **`jobs.run_deployment`** via Celery (commit **before** enqueue to avoid worker races). **`GET /api/deployment/{id}/status`**, **`GET /api/deployment/deployments?project_id=`**, and **`DELETE /api/deployment/{id}`** (logical teardown) complete the lifecycle. The worker **stub** promotes **`deployed`** with a placeholder endpoint; real **`terraform apply` / cloud apply** stays gated for later phases. **Project-scoped listing** ties deployments to the same **project** object that owns **pipeline configs**.

```mermaid
flowchart TB
  subgraph HTTP["FastAPI"]
    Dep[POST /api/deployment/deploy]
    St[GET .../status]
    Lst[GET .../deployments]
    TD[DELETE .../id]
  end
  subgraph DB["PostgreSQL"]
    DRow[(deployments)]
  end
  subgraph Queue["Celery + Redis"]
    T[run_deployment task]
  end
  Dep -->|flush + commit| DRow
  Dep -->|delay| T
  T -->|sync session update| DRow
  St --> DRow
  Lst --> DRow
  TD --> DRow
```

```mermaid
sequenceDiagram
  participant C as Client
  participant API as DeploymentService
  participant DB as deployments
  participant W as run_deployment worker

  C->>API: POST /deploy (config_id, provider)
  API->>DB: insert deploying
  API->>DB: commit
  API->>W: delay(deployment_id)
  API-->>C: 201 + deploymentId
  W->>DB: stub: status deployed, endpoint URL
```

---

## Design level 5 — Phase 8 complete (consolidated integration view)

At the end of Phase 8, **two UX bridges** (handoff + import) and **two backend pillars** (evaluation + deployment) align on **`PipelineConfiguration`** persisted under **projects**. The system is ready for **Phase 9 (MLflow)** to attach experiment metadata to the same build and config identifiers.

```mermaid
flowchart TB
  subgraph Phase8["Phase 8 — integration layer"]
    direction TB
    subgraph UX["apps/web"]
      D[Designer draft]
      A[Autopilot build + result]
      D -->|P8-1 baseConfig| A
      A -->|P8-2 import| D
    end
    subgraph API_P8["apps/api routers"]
      Evl["/api/evaluation/*<br/>P8-3"]
      Dep["/api/deployment/*<br/>P8-4"]
      AP["/api/autopilot/*<br/>existing"]
    end
    subgraph Data["Persistence"]
      PC[(pipeline_configs)]
      AB[(autopilot_builds)]
      ER[(evaluation_runs)]
      DP[(deployments)]
    end
    UX --> AP
    UX --> Evl
    UX --> Dep
    AP --> PC
    AP --> AB
    Evl --> ER
    Evl --> PC
    Dep --> DP
    Dep --> PC
  end
```

---

## Sub-phase → diagram map

| Sub-phase | Primary design levels | Focus |
|-----------|----------------------|--------|
| **P8-1** | 0 → 1 | `baseConfig` handoff; `POST /api/autopilot/build` payload |
| **P8-2** | 1 → 2 | `applyAutopilotBuildResult`; Designer review banners + same visualizers |
| **P8-3** | 2 → 3 | `EvaluationService`; `evaluation_runs`; compare endpoint |
| **P8-4** | 3 → 4 → 5 | `DeploymentService`; Celery `run_deployment`; project-scoped lists |

---

## References (code)

| Area | Location |
|------|----------|
| Autopilot build + `base_config` | `apps/api` autopilot service / build start; `apps/web` autopilot store & build monitor |
| Designer import | `useDesignerStore.applyAutopilotBuildResult*`, `AutopilotDesignerImportBanner`, `DecisionExplainer` |
| Evaluation API | `apps/api/app/routers/evaluation.py`, `services/evaluation_service.py` |
| Deployment API | `apps/api/app/routers/deployment.py`, `services/deployment_service.py`, `app/worker/tasks.py` → `run_deployment` |
