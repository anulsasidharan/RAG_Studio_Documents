# Phase 3 Explained — For Non-Technical Audiences

**How to use this document:** You can read sections aloud in order, or pick the “One-minute summary” first and then go deeper only if people ask questions.

---

## One-minute summary

**RAG Studio** helps teams design and run AI systems that answer questions using their own documents—not just the open internet.

- **Phase 2** (already built on the server side) is the **engine room**: it loads documents, breaks them into searchable pieces, stores them smartly, finds the best snippets when someone asks a question, and writes answers with an AI model.
- **Phase 3** (what this document focuses on) is the **front door and workspace**: the website people actually see—consistent visuals, navigation, pages that introduce the product, and in-browser memory so work-in-progress is not lost when the user refreshes the page.

Together, Phase 2 answers “**Can the system do the work?**” Phase 3 answers “**Can a person understand and use the product comfortably?**”

---

## The big picture (simple analogy)

Think of a **library**:

- **Phase 2** decides how books are received, catalogued, split into useful sections, indexed, found when needed, and summarized for a reader.
- **Phase 3** is the **building**: the entrance, signs, desks, and shelves that help visitors know where to go and feel oriented—even before they ask the librarian a question.

---

## What Phase 3 added (in plain language)

### 1. A shared visual design system (components library)

We standardized buttons, forms, panels, and other UI building blocks so every screen looks and behaves like part of one product—not a collection of random pages.

**Why it matters:** Faster future features, fewer visual bugs, and a professional first impression for users and stakeholders.

---

### 2. App layout: top bar, sidebar, and responsive behavior

The product has a **top navigation bar** (brand, light/dark mode, links, project picker) and, on inner pages, a **sidebar** listing projects and shortcuts (for example Designer and Autopilot).

**Why it matters:** People always know where they are, how to switch areas of the app, and how to work on mobile versus desktop (including a mobile-friendly menu).

The **home page** is intentionally **full width** without the sidebar so marketing-style content can breathe; **inside the app**, the sidebar appears to support day-to-day work.

---

### 3. State stores: remembering the user’s work in the browser

We use lightweight in-browser “memory” for three areas:

| Area | What we remember (conceptually) |
|------|-----------------------------------|
| **Designer** | The pipeline configuration the user is editing, and which step they are on. |
| **Autopilot** | Requirements for an automated build, uploaded document references, optional handoff from Designer, and build history summaries. |
| **Projects** | A simple list of projects, which one is active, and basic metadata. |

**Why it matters:** Users can close the tab and come back without losing draft work. This is especially important for long forms and multi-step flows.

**Technical note for curious listeners:** Data is stored in the browser’s local storage with sensible limits (for example trimming very long chat-style histories so the app stays fast).

---

### 4. Landing page: explaining the product before login

The landing page walks a visitor through **what RAG Studio is**, **how Designer vs Autopilot differ**, **how the workflow works**, **key benefits**, **example use cases**, and **pricing-style messaging**, ending with a clear call to action.

**Why it matters:** Recruiters and buyers often decide whether to care in the first minute. This page is optimized for clarity, not jargon.

---

### 5. Shared utilities: validation and future exports

We added **validators**—rules that check whether a configuration is structurally valid (allowed providers, sensible numeric ranges, required fields, and so on).

We also added **generators** that can turn a pipeline configuration into human-readable artifacts—such as **diagrams**, **YAML**, **Python-style scaffolding**, or **infrastructure sketches**—when the product needs to show or download them.

**Why it matters:** The same “truth” about a pipeline can be checked for mistakes early and later translated into formats engineers expect—without rebuilding everything by hand each time.

---

## How Phase 2 fits in (backend, explained simply)

You do **not** need to remember product codes—only the **story**:

1. **Bring documents in** (ingestion).
2. **Split them into manageable chunks** (chunking).
3. **Turn text into mathematical fingerprints** (embeddings).
4. **Store and search those fingerprints efficiently** (vector store).
5. **Retrieve the best pieces for a question** (retrieval; sometimes mixing keyword + semantic search and reranking).
6. **Compose an answer with an AI model** (generation).
7. **Measure quality where possible** (evaluation).
8. **Run long jobs in the background** so the website stays responsive (task queue).
9. **Expose health and helper endpoints** so operators know the system is alive and consistent.

Phase 3 is the layer where users **discover, navigate, and prepare** their work; Phase 2 is where heavy lifting **runs on the server** when builds and queries happen.

---

## Interview-friendly closing lines

- “Phase 3 gave us a **credible, cohesive product shell**: layout, navigation, landing story, and reliable draft memory.”
- “Phase 2 is the **RAG engine**; Phase 3 is the **human-facing experience** built to grow into full Designer and Autopilot workflows in later phases.”
- “We invested early in **design consistency and validation** so we ship faster later—with fewer surprises for users and engineers.”

---

## Document info

- **Purpose:** Explain Phase 3 to mixed audiences (including non-engineers).
- **Companion:** High-level roadmap and phase matrix live in `docs/internal/project_status.md`.
