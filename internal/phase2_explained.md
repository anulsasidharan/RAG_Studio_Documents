# Phase 2 Explained — For Recruiters & Non‑Technical Interview Boards

This document describes **Phase 2 (Designer Mode)** of **Unified RAG Studio** in everyday language. You can read it aloud or hand it out as-is. Technical names appear where they match the real product and codebase; each is briefly explained.

---

## What Problem Does Phase 2 Solve?

Organizations want **AI assistants that answer questions using their own documents** — policies, manuals, product specs — not guesses from the public internet.

**RAG** (Retrieval‑Augmented Generation) means:

1. **Put knowledge in a searchable form** (indexing).
2. **When someone asks a question, find the most relevant pieces** (retrieval).
3. **Have a large language model write an answer that stays faithful to those pieces** (generation).

Phase 2 builds the **engine and configuration layer** so teams can **design** that pipeline: choose how documents are loaded, split, turned into search-friendly math, stored, searched, and finally used to produce answers — and **estimate cost** and **check quality** where the product supports it.

---

## The Big Picture: An Assembly Line

Think of the system as a **factory line** for knowledge:

**Raw inputs** (PDFs, web pages, Word files, uploaded files)  
→ **Cleaning and labeling**  
→ **Cutting into bite-sized pieces**  
→ **Turning text into “fingerprints” computers can compare** (embeddings)  
→ **Filing those fingerprints in a searchable index** (vector database)  
→ **When a user asks something, finding the best matching pieces**  
→ **Writing a careful answer using only what was found**

Each **folder** in the backend roughly matches **one station** on that line. The **same design** is meant to support both a visual “Designer” experience and future automated “Autopilot” tuning.

---

## Simple Vocabulary (How We Name Things in Code)

| Term in the codebase | Plain meaning |
|----------------------|----------------|
| **Service** | The **main worker** at a station. It knows the steps end-to-end for that station (for example: “load documents and clean them”). |
| **Strategy** | A **chosen approach** inside that station (for example: split text by fixed size vs. by headings vs. by meaning). |
| **Factory** | A **switchboard** that picks the right concrete implementation when the user selects a vendor or algorithm name from a catalog. |
| **Pipeline bridge** | A thin **translator** that turns **saved pipeline settings** (JSON from the Designer) into the **runtime knobs** our services actually use. |
| **Schema** | The **official checklist** for what configuration is allowed — same “shape” on the web app and the API so mistakes are caught early. |

You do **not** need to remember filenames. Do remember: **services do the work; strategies describe *how*; bridges connect settings to execution.**

---

## Station by Station (What Each Major Folder Does)

### 1. Ingestion (`ingestion`)

**Job:** Bring content in from the real world — files, URLs, or raw uploads.

**What happens:** Detect the format, extract readable text, normalize it, optionally pull **metadata** (title, author, page numbers), and attach **traceability** (where this text came from).

**Supporting files:** **Loaders** open files; **preprocessors** clean text; **extractors** read extra fields from formats like PDF or HTML.

---

### 2. Chunking (`chunking`)

**Job:** Chop long documents into **smaller passages** so search stays precise.

**Why it matters:** A whole 100‑page PDF is unwieldy. Smaller chunks match specific questions better.

**Strategy examples:** Fixed-size pieces, splits that respect headings, semantic “natural break” chunking, or code-aware splits for repositories.

Optional **quality helpers** score chunks so obviously weak snippets can be filtered.

---

### 3. Embedding (`embedding`)

**Job:** Convert each chunk of text into a **numerical fingerprint** (a vector) that captures meaning for similarity search.

**Strategy angle:** Different **providers** (OpenAI, Cohere, Google, Hugging Face, etc.) expose different models; the factory picks the right client. **Caching** avoids paying twice to fingerprint duplicate text.

---

### 4. Vector store (`vectorstore`)

**Job:** Store and query those fingerprints in a **vector database**.

**Strategy angle:** The product supports multiple vendors (for example **Qdrant**, **Pinecone**, **Weaviate**) through a consistent interface — index documents, search by similarity, return **ranked hits with scores**, optional filters.

---

### 5. Retrieval (`retrieval`)

**Job:** Given a **user question**, find the **best supporting passages** from the index.

**Why it’s rich:** Beyond “pure similarity,” the layer can combine **keyword-style** search with **vector** search, diversify results so they are not all nearly identical, merge several search strategies, handle **parent/child** document relationships, and **rerank** candidates with a dedicated reranker when configured.

**Pipeline bridge:** Reads the Designer’s retrieval settings (strategy, top‑K, thresholds, hybrid balance, filters) and maps them into the runtime configuration the retrieval service expects.

---

### 6. Generation (`generation`)

**Job:** Turn **retrieved passages + user question** into a **natural-language answer** using a chosen **LLM provider** (OpenAI, Anthropic, Google, etc.).

**What it controls:** Model choice, tone (temperature), length limits, system instructions, optional **chat history**, and output shape (for example plain text vs. structured JSON).

**Pipeline bridge:** Maps saved generation settings from the pipeline JSON into the parameters the generation service uses.

---

### 7. Evaluation (`evaluation`)

**Job:** **Measure** how good the RAG behavior is — not just “it sounds nice,” but metrics aligned with **faithfulness to sources**, **answer relevance**, and **how well the retrieved context supports the answer**.

The engine can use industry-standard **RAGAS** metrics, optional **failure analysis** to group weak spots, **synthetic** test question helpers, and **comparison** utilities for A/B style reviews.

**Pipeline bridge:** Decides which metrics to run when evaluation is enabled in the saved configuration.

---

### 8. Utilities (`utilities` API and `core/utilities`)

**Job:** Cross-cutting helpers for the Designer experience.

Examples:

- **Validate pipeline** — check a full pipeline JSON **before** saving or running it.
- **Cost estimation** — rough **monthly or per-query** cost from shared pricing data, so stakeholders see trade-offs early.
- **Service info** — version and environment details for dashboards and support.

---

### 9. Background work (`worker`)

**Job:** Long-running tasks (builds, evaluations, deployment-related jobs) run **outside** the main API process so the user interface stays responsive. A **task queue** (Celery with Redis) holds work; workers pick it up and update **database** records as stages complete.

---

### 10. Data & contracts (schemas, models, database)

**Job:** **Persist** pipeline designs (JSON in the database linked to projects), evaluation runs, deployments, and related history. **Schemas** define the **allowed pipeline shape** so the frontend, API, and documentation stay aligned.

---

## How Phase 2 Fits the Product Roadmap

Public roadmap language calls Phase 2 **“Designer Mode”**: visual or guided configuration of many pipeline stages, **cost awareness**, and paths toward **export** and templates. The code you have invested in implements the **core backend capabilities** — the **stations** above — so those product features have a **real engine** underneath, not just a diagram.

Later phases add more **automation** (agents, iterative optimization, deployment wiring). Phase 2’s modular layout means those features can **swap strategies** and **call the same services** instead of rewriting the pipeline from scratch.

---

## One-Sentence Summary for Closing

**Phase 2 turns “we want our own document-grounded AI” into a configurable, testable assembly line:** ingest → chunk → embed → index → retrieve → generate → evaluate — with validation, costing, persistence, and background jobs supporting real product workflows.
