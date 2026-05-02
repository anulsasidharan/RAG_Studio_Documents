# Project system design evolution — Unified RAG Studio

> Narrative and diagrams showing how the architecture deepens by phase.  
> Last updated: Phase 3 · P3-5 (Lib Utilities & Validators).

---

## Phase 0 — Bootstrap & runnable skeleton

**Goal:** Monorepo layout, local infra (Docker), CI skeleton, API + web scaffolds.

At this stage the system is “boxes and arrows”: services exist mainly so engineers can run lint/tests and prove connectivity—not yet rich domain logic.

```mermaid
flowchart LR
  subgraph Dev["Developer workstation"]
    DEV[Next.js dev server]
    API[FastAPI scaffold]
  end
  subgraph Infra["Docker Compose (conceptual)"]
    PG[(Postgres)]
    RD[(Redis)]
    VS[(Vector DB stub)]
  end
  DEV -. HTTP .-> API
  API -. drivers .-> PG
  API -. cache queue .-> RD
```

**Characteristics:** Single repo, minimal coupling between UI and API; contracts informal.

---

## Phase 1 — Shared contracts as source of truth

**Goal:** JSON catalogs + mirrored TS/Python types + migrations establish **one vocabulary** for pipelines, models, and autopilot builds.

```mermaid
flowchart TB
  subgraph Contracts["Shared contracts"]
    CAT[JSON catalogs]
    TS[TypeScript types]
    PY[Pydantic schemas]
    DB[(SQL schema)]
  end
  CAT --> TS
  CAT --> PY
  PY --> DB
```

**Characteristics:** Frontend and backend compile against the same shapes; fewer translation bugs before complex UI arrives.

---

## Phase 2 — Reusable backend RAG platform services

**Goal:** Implement ingestion → chunking → embeddings → vector stores → retrieval → generation → evaluation, orchestrated asynchronously where needed.

```mermaid
flowchart LR
  subgraph Core["Backend core services"]
    ING[Ingestion]
    CH[Chunking]
    EMB[Embedding]
    VS[Vector store]
    RET[Retrieval]
    GEN[Generation]
    EV[Evaluation]
  end
  ING --> CH --> EMB --> VS --> RET --> GEN
  RET --> EV
  subgraph Async["Async execution"]
    Q[Celery workers]
  end
  GEN -. optional jobs .-> Q
  EV -. optional jobs .-> Q
```

**Characteristics:** Designer and Autopilot both consume the same engines via forthcoming APIs; complexity concentrates server-side.

---

## Phase 3 — Frontend foundation (evolving)

Phase 3 grows the experience layer in sub-phases.

### P3-1 — UI system

shadcn/ui + Tailwind tokens establish consistent interaction primitives (buttons, dialogs, forms).

### P3-2 — Client state stores (this milestone)

**Goal:** Persisted **Designer drafts**, **Autopilot sessions/build snapshots**, and **local project metadata** in the browser using Zustand + `localStorage`, coordinated with Next.js hydration.

```mermaid
flowchart TB
  subgraph Browser["Next.js client runtime"]
    UI[React UI]
    DS[Designer store]
    AS[Autopilot store]
    PS[Projects store]
    LH[(localStorage)]
  end
  subgraph FutureAPI["Phase 4+ APIs (planned)"]
    PAPI["Projects API"]
    CAPI["Designer config API"]
    AAPI["Autopilot API"]
  end
  UI --> DS & AS & PS
  DS & AS & PS -. persist .-> LH
  DS -. sync later .-> CAPI
  PS -. sync later .-> PAPI
  AS -. sync later .-> AAPI
```

**Characteristics:** UX continuity offline/between refreshes; explicit seam for server reconciliation once CRUD endpoints land.

### P3-3 — App shell & navigation (this milestone)

**Goal:** A consistent **app chrome** for every route: top navigation (mode switcher, project switcher, placeholders for templates/account), optional **project sidebar** on non-marketing routes, **React Query** at the root for upcoming API integration, and **404 / error** boundaries for resilient UX.

```mermaid
flowchart TB
  subgraph Shell["Next.js App Router"]
    RL["layout.tsx"]
    PV["Providers + QueryClient"]
    SH["StoreHydration"]
    AS["AppShell"]
    NB["Navbar"]
    SB["Sidebar — projects"]
    PG["page segments"]
  end
  RL --> PV --> SH --> AS
  AS --> NB
  AS --> SB
  AS --> PG
  subgraph Stores["Zustand persisted"]
    PS[(Projects metadata)]
  end
  NB --> PS
  SB --> PS
```

**Characteristics:** Sidebar hidden on `/` for a clean landing; Designer / Autopilot / Templates / Projects routes share chrome; collapse state persisted locally; server APIs remain Phase 4+.

### P3-4 — Landing page (this milestone)

**Goal:** A full-featured **marketing/entry experience** at `/` composed of seven focused section components assembled in `page.tsx`. All sections are **React Server Components** — zero client JS for the landing route — ensuring fast initial load and optimal Core Web Vitals.

```mermaid
flowchart TB
  subgraph Page["apps/web/src/app/page.tsx (RSC)"]
    HE["Hero\n badge · headline · gradient orbs · 2 CTAs"]
    MC["ModeComparison\n Designer card | Autopilot card"]
    HW["HowItWorks\n 4 numbered steps × 2 modes"]
    FT["Features\n 6-feature icon grid"]
    UC["UseCases\n 3 persona cards"]
    PR["Pricing\n Free / Pro / Enterprise tiers"]
    CT["CTA\n gradient banner · 2 CTAs"]
    FO["Footer\n minimal brand line"]
  end
  subgraph CSS["globals.css additions"]
    AN["@keyframes float\n .animate-float\n .animate-float-delayed"]
  end
  HE -->|uses| AN
  subgraph Routes["Navigation targets"]
    DR["/designer"]
    AR["/autopilot"]
  end
  HE --> DR & AR
  MC --> DR & AR
  CT --> DR & AR
```

**Component breakdown:**

| Component | Purpose | Key decisions |
|---|---|---|
| `Hero` | First impression | CSS-only floating orbs (`@keyframes float`), ping animation badge, two CTAs |
| `ModeComparison` | Feature contrast | Designer (primary) vs Autopilot (purple) visual split; feature lists with checkmarks |
| `HowItWorks` | Step-by-step guide | Two 4-step columns with connector lines; server component |
| `Features` | Capability grid | 6 cards with color-mapped lucide icons; hover lift transition |
| `UseCases` | Persona-driven | Learning Engineer / Startup / Enterprise; quote + benefit list pattern |
| `Pricing` | Conversion | `included: boolean | 'partial'` discriminated feature rows; Minus vs Check icons |
| `CTA` | Final conversion | Brand gradient banner; mirrors Hero's visual language |

**Characteristics:** Sidebar is suppressed on `/` (AppShell `isHome` check from P3-3). All sections are RSC — no `'use client'` required. Animated orbs use native CSS (`@keyframes`), not framer-motion. Routing from CTA buttons goes to `/designer` and `/autopilot`.

### P3-5 — Utilities & validators (this milestone)

**Goal:** Zod validation schemas for all pipeline types, four code/diagram generators (Mermaid, Python LCEL, YAML, Terraform), and a Vitest unit-test suite covering all generator outputs.

```mermaid
flowchart TB
  subgraph Lib["apps/web/src/lib/"]
    VAL["validators.ts\nZod schemas for\nPipelineConfiguration\nBuildRequirements"]
    CONST["constants.ts\nstage route map\ndefault values"]
  end

  subgraph Gen["apps/web/src/lib/generators/"]
    MG["mermaidGenerator.ts\ngenerateMermaidDiagram()\ngeneratePipelineSummary()"]
    PG["pythonCodeGenerator.ts\ngeneratePythonCode()\n— LangChain LCEL"]
    YG["yamlGenerator.ts\ngenerateYAML()"]
    TG["terraformGenerator.ts\ngenerateTerraform()\n— AWS / GCP / Azure"]
  end

  subgraph Tests["__tests__/  (Vitest)"]
    FX["fixtures.ts\nminimalConfig\nfullConfig\nazureConfig"]
    MT["mermaidGenerator.test.ts\n17 assertions + snapshot"]
    PT["pythonCodeGenerator.test.ts\n18 assertions + snapshot"]
    YT["yamlGenerator.test.ts\n19 assertions + 2 snapshots"]
    TT["terraformGenerator.test.ts\n21 assertions + 2 snapshots"]
    VT["validators.test.ts\n38 assertions"]
  end

  subgraph Types["apps/web/src/types/"]
    PC["PipelineConfiguration"]
    BR["BuildRequirements"]
  end

  Types --> VAL
  Types --> MG & PG & YG & TG
  FX --> MT & PT & YT & TT & VT
  VAL --> VT
```

**Key design decisions:**

| Module | Key design decision |
|---|---|
| `validators.ts` | Zod schemas mirror TS types; cross-field refinements (overlap < chunkSize, hybrid requires hybridSearch config) |
| `mermaidGenerator.ts` | Two sub-graphs (indexing vs query path); node labels sanitised to strip Mermaid syntax characters |
| `pythonCodeGenerator.ts` | Provider-to-import maps; LCEL `RunnableParallel` pattern; all optional stages (reranking, memory, multi-query) wired conditionally |
| `yamlGenerator.ts` | No third-party serialiser; hand-built helpers for quoting, bool, arrays; block scalars for system prompts |
| `terraformGenerator.ts` | Three concrete cloud targets (AWS/GCP/Azure); multi-cloud falls back to AWS; Pinecone handled as managed service with secrets-manager wiring |
| Tests | Targeted `.toContain()` assertions + `toMatchSnapshot()` per generator; shared fixtures eliminate duplication |

**Characteristics:** All generators are pure functions (no I/O, no global state), callable from both the browser export UI and the backend export API. The Vitest runner is installed now (`devDependencies`) so P10-3 adds React Testing Library on top rather than replacing this setup. 113 tests pass with 7 snapshots written on first run.

---

## Looking ahead (compact)

Later phases add Designer UX depth (Phase 5), LangGraph autopilot agents + streaming APIs (Phases 6–7), evaluation/deployment endpoints (Phase 8), MLflow (Phase 9), automated testing gates (Phase 10), observability (Phase 11), and production hardening (Phase 12). Each increment extends this document with diagrams focused on new boundaries (auth, metrics, deployment planes).
