# Project system design evolution — Phase 5 (Designer UI)

> **Append-only.** Phase 5 adds the visual pipeline builder experience in **Designer mode**. This file starts simple and deepens as sub-phases land (P5-1 layout → P5-2 cloud selector → P5-3 ingestion → …).

---

## Design level 1 — Shell only (after P5-1)

The user navigates **12 stages** via **StageNavigator** + routes under `/designer/**`. **Pipeline configuration** is still mostly placeholders except routing and persisted draft shape.

```mermaid
flowchart TB
  subgraph Browser
    Shell[DesignerShell]
    Nav[StageNavigator]
    Placeholder[DesignerStagePlaceholder]
  end
  Shell --> Nav
  Shell --> Placeholder
```

---

## Design level 2 — Cloud catalog bound to draft (after P5-2)

The **Cloud Provider** stage renders **`CloudProviderSelector`**, which reads **`data/cloud-providers.json`** at build time via **`apps/web/src/lib/cloud-providers-catalog.ts`**. Selecting a provider calls **`patchDraft({ cloudProvider })`** so **`PipelineConfiguration.cloudProvider`** stays aligned with backend enums (`aws`, `gcp`, `azure`, `multi-cloud`). **Logo assets** ship under **`apps/web/public/logos/`** so catalog **`logo`** paths resolve. The sidebar shows the current provider abbreviation next to the Cloud stage link.

```mermaid
flowchart LR
  subgraph Repo["Monorepo"]
    JSON[(data/cloud-providers.json)]
  end
  subgraph Next["apps/web"]
    Lib[cloud-providers-catalog.ts]
    Cmp[CloudProviderSelector]
    Store[(useDesignerStore draft)]
  end
  JSON --> Lib
  Lib --> Cmp
  Cmp --> Store
```

---

## Design level 3 — Data ingestion bound to `draft.stages.dataIngestion` (after P5-3)

The **Data Ingestion** stage renders **`DataIngestionConfigurator`**, which edits **`DataIngestionConfig`** fields aligned with **`apps/web/src/types/pipeline.ts`** and **`apps/api/app/schemas/pipeline.py`**. Updates flow through **`updateStages`** (nested merge for preprocessing/metadata). **`connectionConfig`** holds non-secret hints only (bucket, prefix, host, etc.). **`StageNavigator`** surfaces a compact **source type** label next to the ingestion step.

```mermaid
flowchart LR
  subgraph UI["apps/web"]
    Cmp[DataIngestionConfigurator]
    Store[(draft.stages.dataIngestion)]
  end
  Cmp -->|"updateStages"| Store
```

---

## Design level 4 — Chunking bound to `draft.stages.chunking` (after P5-4)

The **Chunking** stage renders **`ChunkingConfigurator`**, which edits **`ChunkingConfig`** (**`strategy`**, **`chunkSize`**, **`chunkOverlap`**, optional **`separators`**, optional **`metadata`**) aligned with **`apps/web/src/types/pipeline.ts`** and **`ChunkingConfigSchema`** in **`validators.ts`**. Strategy cards are driven by **`data/chunking-strategies.json`**; switching strategy applies catalog defaults (clamped to **128–4096** tokens and overlap rules). **Recursive character** exposes a **separator ladder** textarea with `\n` / `\t` escapes. **`StageNavigator`** shows a compact **strategy · tokens/overlap** subtitle under **Chunking**.

```mermaid
flowchart LR
  subgraph UI["apps/web"]
    Cmp[ChunkingConfigurator]
    Store[(draft.stages.chunking)]
    Cat[data/chunking-strategies.json]
  end
  Cat --> Cmp
  Cmp -->|"updateStages"| Store
```

---

## Design level 5 — Embedding catalog bound to `draft.stages.embedding` (after P5-5)

The **Embedding** stage renders **`EmbeddingConfigurator`**, which reads **`data/models/embeddings.json`** via **`apps/web/src/lib/embeddings-catalog.ts`**. Users **search** and **filter** (provider, tier, quality, speed, open-source, hide deprecated), pick a **model card**, and tune **`batchSize`** within **`EmbeddingConfigSchema`**. Selection calls **`embeddingConfigFromCatalogEntry`** so **`model`**, **`provider`**, **`dimensions`**, and **`maxTokens`** stay aligned with the catalog record. **`StageNavigator`** shows **`embeddingHint`** (display name · dimensions).

```mermaid
flowchart LR
  subgraph Repo["Monorepo"]
    JSON[(data/models/embeddings.json)]
  end
  subgraph Next["apps/web"]
    Lib[embeddings-catalog.ts]
    Cmp[EmbeddingConfigurator]
    V[Zod EmbeddingConfigSchema]
    Z[(draft.stages.embedding)]
  end
  JSON --> Lib
  Lib --> Cmp
  Cmp -->|"updateStages"| Z
  Cmp -.->|safeParse| V
```

---

## Future levels (placeholder)

Remaining P5 tasks add panels for **vector store**, **retrieval**, **generation**, **routing/memory/evaluation**, **visualizer**, **cost**, **export**, **review**, and **templates** — each **`updateStages`** / **`patchDraft`** onto the same **`draft`** consumed by Phase 4 APIs.

```mermaid
flowchart TB
  Draft[PipelineConfiguration draft]
  Draft --> VS[P5-6 vector store …]
  Draft --> Later[Later P5 stages …]
```

---

*Append new “Design level” sections at the end as P5-6+ ship.*
