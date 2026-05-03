# Phase 5 Explained — For Recruiters and Non-Technical Interviewers

This note describes **Phase 5** of RAG Studio in everyday language. You can read it **as-is** to the interview board. No coding background is required.

---

## What Phase 5 is (one sentence)

**Phase 5 is the “Design your own RAG pipeline” experience:** a guided walkthrough where someone picks settings step by step—like choosing ingredients for a recipe—and the product shows **cost**, **exportable code**, and a **live picture of the pipeline** updating along the way.

---

## The big picture: what problem does it solve?

A **RAG** (Retrieval-Augmented Generation) system connects several ideas: where documents come from, how they are split for search, how they are turned into numbers a computer can compare, where those numbers are stored, how answers are retrieved, and which AI model writes the final answer.

That is a lot to decide at once. Phase 5 **breaks that into clear stages** so a user is never overwhelmed. They always see **where they are in the journey**, what they have already chosen, and what still needs attention.

---

## The main idea: a living “draft”

While the user moves through the steps, the app keeps a single **draft configuration**—think of it as a **working document** that gets filled in.

- Every screen **reads** from that draft and **writes** back to it when the user makes a choice.
- The draft is **remembered in the browser** for that session (so refreshing the page does not throw away work casually).
- The draft is the **one source of truth** for “what the user has designed so far.”

That is why different parts of the screen **stay in sync**: they are all looking at the same draft.

---

## How the screen is organized (three layers)

Imagine the Designer workspace as a **desk** with three layers:

1. **Left: the roadmap (stage navigator)**  
   A vertical list of stages: Cloud, Data ingestion, Chunking, Embedding, Vector store, Retrieval, Reranking, Generation, Routing, Memory, Evaluation, and finally Review.  
   The user can jump between stages; the roadmap shows **where they are** and **what comes next**.

2. **Center: the current step’s form**  
   This is the “work area” for the active stage—questions and controls specific to that topic (for example, which cloud provider, or how large text chunks should be).

3. **Bottom: three always-on panels** (same on every stage after the user has started moving forward)  
   - **Cost estimate** — a rough idea of what the chosen setup might cost to run.  
   - **Code export** — a way to copy or download starter code or configuration files the product generates from the draft.  
   - **Pipeline picture** — a simple **diagram** of the pipeline that grows as the user explores more stages.

So: **roadmap on the left**, **details in the middle**, **live summary tools at the bottom**.

---

## What each stage means (simple definitions)

You do not need to memorize technical terms. Here is what each stage is *about* in plain English:

| Stage | What the user is really deciding |
|--------|-----------------------------------|
| **Cloud provider** | Which major cloud environment this design is aimed at (or a general “multi-cloud” option). |
| **Data ingestion** | How documents get into the system—file types, cleaning options, and what metadata to keep. |
| **Chunking** | How long each “piece” of text should be for search, and how pieces overlap—like choosing paragraph size for an index. |
| **Embedding** | Which model turns text into searchable vectors (numbers that capture meaning). |
| **Vector store** | Where those vectors live for fast similarity search. |
| **Retrieval** | How many candidate chunks to pull back and how strict the match should be. |
| **Reranking** | Whether a second pass re-orders results for higher quality (often optional). |
| **Generation** | Which AI model answers the user’s question and how “creative” or strict it should be. |
| **Routing** | Whether different kinds of questions should follow different paths (optional in many setups). |
| **Memory** | Whether the assistant should remember earlier turns in a conversation. |
| **Evaluation** | Whether built-in quality checks run on answers (useful for teams that care about reliability). |
| **Review** | A **summary page**: key facts in one place, links back to earlier steps, and shortcuts to the cost, export, and diagram sections. |

Each of these stages is implemented as its own **screen component**—a focused form—so the codebase stays maintainable and each topic can evolve independently.

---

## How the roadmap and the diagram stay aligned

When the user visits a stage, the app **marks that stage as reached**. The pipeline diagram at the bottom can **reveal steps progressively**: early on it might show only part of the flow; as the user visits more stages, the picture **fills out** to match their progress.

That gives a sense of **momentum** and **completeness** without forcing a rigid wizard they cannot leave.

---

## The Review stage: the “executive summary”

The **Review** step is intentionally different from the others. It is less about new inputs and more about **confirmation and communication**:

- Short **summary cards** for the main choices.  
- A **readable bullet list** of what the pipeline does end-to-end.  
- **Links** that jump back to specific earlier stages if something needs a tweak.  
- **Copy** actions so someone can paste a text summary or the full technical JSON into email, a ticket, or a document.  
- A careful **“reset draft”** action so power users can start over without hunting for hidden controls.

The Review page **does not duplicate** the big diagram logic; it **scrolls the user** to the existing diagram section. That avoids confusion and duplicate maintenance.

---

## Template gallery: starting from a proven recipe

Separate from the step-by-step Designer, Phase 5 also includes a **Template gallery** page.

Think of templates as **pre-filled drafts** for common situations (for example, a cost-conscious FAQ bot or a documentation-heavy Q&A setup).

Two paths exist:

1. **Preview locally** — loads the template into the Designer as a draft and opens Review. **No server save**; good for demos or when someone just wants to explore.  
2. **Use template (save)** — creates or picks a **project** on the server, saves a real pipeline configuration, loads it into the Designer, and sends the user to Review so they can continue editing.

That connects **marketing-friendly presets** to **serious, saved work** when the product is hooked up to the backend.

---

## Where “the truth” lives (important for non-technical governance)

- **While designing:** the draft mainly lives in the **browser** as the user edits. That makes the experience fast and responsive.  
- **When applying a template with save:** the product talks to the **server** so the configuration becomes a **stored record** tied to a project—appropriate for teams that need auditability and reuse.

This distinction is useful when someone asks: “Is this already saved in our database?” The answer depends on whether they only **previewed** or used **save with template / future save flows**.

---

## How this helps recruiters talk about the product

You can describe Phase 5 to candidates or clients like this:

- **“It is a guided configurator, not a blank terminal.”**  
- **“It turns a complex AI architecture into a checklist of business decisions.”**  
- **“Cost and export are visible early, so buyers and engineers see feasibility, not surprises at the end.”**  
- **“Templates let us sell outcomes—FAQ bot, doc Q&A—while still allowing full customization.”**

---

## One-line summary for the interview board

**Phase 5 delivers the full Designer Mode: a stage-by-stage builder with a persistent draft, live cost and export, a growing pipeline diagram, a final Review hub, and a Template gallery that can either preview or save real configurations—so both business and technical stakeholders can see and steer the same system.**

---

*This document reflects the Phase 5 Designer and Template Gallery scope as implemented in the product codebase. It is written for verbal presentation; adjust examples to your audience if needed.*
