# Phase 6 Explained — For Recruiters and Mixed Interview Boards

**Audience:** People who do not need to read code. You can read this document aloud or share it as-is.  
**What Phase 6 is:** The **“Autopilot” brain** on the **server** for **RAG Studio** — the part that tries to **automatically design and tune** a retrieval-augmented generation (RAG) pipeline instead of the user configuring every knob by hand.

---

## First, two product ideas in plain English

**RAG (retrieval-augmented generation)** means: when someone asks a question, the system first **finds relevant documents or chunks**, then **generates an answer** using those sources. That is how many enterprise “chat your documents” products work.

**RAG Studio** offers two modes:

1. **Designer mode** — A person (or team) **manually chooses** settings: how documents are split, which embedding model is used, how search works, which LLM answers, and so on. Think of it as **building a custom recipe step by step**.

2. **Autopilot mode** — The product **runs a structured, automated process** that walks through the same kinds of decisions **for** the user, using **specialist software agents** and clear rules. Think of it as **hiring a small expert team** that proposes and refines a recipe, subject to the user’s goals and limits.

**Phase 6 is entirely about Autopilot on the backend.** The polished screens for Autopilot are planned in a later phase (Phase 7); Phase 6 delivers the **engine**, the **workflow**, and the **APIs** so that engine can be driven by a UI, tests, or integrations.

---

## The big picture: what Phase 6 delivers

Imagine a **project room** with a **whiteboard** and several **specialists** who work **in order**, each leaving **notes and scores** for the next person.

- The **whiteboard** is the **shared state** of the run: which project it is, which documents are in scope, what quality or cost goals the user asked for, what each stage decided, and a **log of progress** suitable for dashboards.
- Each **specialist** is a **named agent** with a clear job (analyze documents, tune chunking, compare embeddings, and so on).
- A **coordinator** (the **orchestrator**) can say: “We are not happy with quality yet — go back and try another round of tuning,” **up to a safe limit** the user configured. That is how the system **iterates** instead of only running forward once.
- When the heavy work runs, it does **not** block the main web server for minutes. Work is **queued** and processed by a **background worker**, like a kitchen taking orders at the front and cooking in the back.
- Finally, **HTTP APIs** are the **front door**: start a run, check status, **stream live updates**, cancel if needed, and fetch results. That is how the rest of the product (or partners) talks to Autopilot **without** knowing internal details.

That mental model is what Phase 6 implemented in code.

---

## How the pieces fit together (still non-technical)

**1. Foundation (P6-1)**  
We defined **one common “notebook”** every agent reads and writes: project and build identifiers, user requirements, optional starting configuration from Designer, a **timeline of decisions**, and **outputs per stage** (analyze, chunking, embedding, and so on). We also wired **shared prompts and tools** so behaviour stays consistent. This is the **plumbing** everything else sits on.

**2. Specialist agents (P6-2 through P6-7)**  
Each item below is a **focused role** in the pipeline story. In the product, they produce **structured recommendations** (and sometimes benchmarks or checklists) that the next stage can use.

| Piece | Simple explanation |
|--------|-------------------|
| **Document analyst** | Looks at the **document side** of the problem and informs **how to split and treat** content so search stays meaningful. |
| **Chunking optimizer** | Explores **how big or small** text pieces should be and **which splitting strategy** fits the corpus better. |
| **Embedding tester** | Compares **embedding options** against goals like quality, speed, and cost so the system is not guessing blindly. |
| **Retrieval optimizer** | Tunes **how we search** the vector store (for example how many results to pull, whether reranking is on) so answers lean on the right context. |
| **Evaluation agent** | Runs **quality-style checks** on the pipeline story (for example test-style questions and scores) so improvements can be **measured**, not only assumed. |
| **Deployment agent** | Prepares the **story of how this pipeline could go live** (packaging and deployment hints). In early phases this is intentionally **conservative** so nothing silently provisions real cloud accounts without explicit product steps later. |

**3. Orchestrator (P6-8)**  
This is the **“project lead”** that runs the specialists **in a defined order**, emits **progress signals** (so a future UI can show a progress bar or a feed), and implements **controlled retries**: if evaluation says targets are not met, the flow can **loop back** to earlier tuning (within a **maximum number of rounds** the user allowed). That makes Autopilot closer to **“optimize until good enough or limit reached”** than a single one-shot pass.

**4. Autopilot APIs (P6-9)**  
This is how **clients** (browser app, partner system, or internal tools) interact with Autopilot **safely and predictably**:

- **Start** a new automated build for a project, with documents and goals.
- **Check status** or **subscribe to a live stream** of updates (like progress notifications).
- **Cancel** a run when the user changes their mind.
- **Fetch results** in a form suitable for **explainability screens** and engineering review.

Behind the scenes, each run is **stored in the database**, tied to a **project and user**, and long work is **handed off to a background worker** so the API stays responsive.

---

## Why this matters to the business (what you can say in an interview)

- **Time to value:** Customers do not need to become RAG engineers on day one; Autopilot is a **guided path** from documents and goals toward a **defensible configuration**.
- **Transparency:** Progress and stage outputs are designed so the product can show **what was tried and why** — important for **trust**, audits, and internal review.
- **Governance and safety:** Heavy or risky actions (for example real cloud provisioning) are **staged**; Phase 6 focuses on **decisioning, measurement, and APIs**, not on silently taking over customer infrastructure.
- **Scalability of the architecture:** Separating **API** (fast responses) from **worker** (long jobs) matches how serious SaaS products run **reliable** background processing.

---

## A tiny glossary (optional to read aloud)

| Term | Plain meaning |
|------|----------------|
| **Agent** | A software module with a **narrow job** and clear inputs/outputs in the Autopilot story. |
| **Orchestrator** | The logic that **calls agents in order**, can **repeat** steps when needed, and records **progress**. |
| **Pipeline** | The **end-to-end path** from documents to answers: ingest, chunk, embed, store, retrieve, generate. |
| **API** | The **official way** other software starts and monitors an Autopilot run. |
| **Background worker** | A **separate process** that does the long Autopilot job so users are not left staring at a frozen page. |

---

## One closing sentence you can use

**Phase 6 built the server-side Autopilot “expert team”: shared context, specialist tuning steps, a coordinator that can iterate within limits, background execution for reliability, and public APIs so the product and partners can drive and observe the whole thing — all in support of helping customers get from documents to a quality RAG setup with less manual trial and error.**

---

*Document purpose: interview and stakeholder explanation. Technical deep dives live in internal design notes and code; this file stays intentionally simple.*
