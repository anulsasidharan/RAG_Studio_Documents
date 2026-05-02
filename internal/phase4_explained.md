# RAG Studio — Phase 4 Explained in Plain Language

**What Phase 4 is:** the **backend services** for **Designer mode**—the part of the product where someone **designs** an AI “knowledge assistant” pipeline (how documents are handled, how search works, which models are used, and so on). Phase 4 makes that design **storable**, **understandable in money terms**, **exportable** to engineering, and **easier to start** with templates.

You can read this document **as-is** to a mixed interview panel, including people who are not engineers.

---

## The one-sentence pitch

**Phase 4 built the server-side “control room” for Designer mode:** users can keep work in **projects**, save the full **pipeline blueprint**, **see what it would cost to run**, **download that blueprint** in standard technical formats, and **begin from expert-written templates** instead of from scratch.

---

## Why this matters to the business

Designer mode is where a user **turns intent into a concrete plan** for a RAG (Retrieval-Augmented Generation) system. Phase 4 answers four questions leadership and partners always ask:

1. **Where is the plan kept?** → In the database, tied to a **project** and a **saved configuration**.
2. **Is the plan valid and consistent?** → The server checks it against shared rules and catalogs before storing.
3. **What does this plan cost at scale?** → A **cost** service turns the same blueprint into estimates.
4. **How do we hand this to IT or an ML team?** → **Export** produces files they can review and deploy.

Together, this is what makes the product feel **serious and governable**, not just a demo screen.

---

## A simple story you can tell out loud

Imagine someone scoping an internal “documentation Q&A” assistant:

1. They open a **project** called *Engineering Docs Pilot*—a container for that engagement.
2. They configure the **pipeline** step by step—cloud, chunking, embeddings, vector database, retrieval style, generation model, and related options.
3. They click **estimate cost**—the system uses **pricing catalogs** to show cost per query and at a monthly volume, with a breakdown by major parts of the stack.
4. Security or platform engineering asks for something they can put in version control—**export** gives them **Python**, **YAML**, **Terraform**, **Docker Compose**, or **Kubernetes** text, plus a suggested filename.
5. If they didn’t start from a blank canvas, they picked a **template** (for example “FAQ chatbot” or “documentation Q&A”)—the server loaded a **pre-filled, validated blueprint** and saved it like any other configuration.

**Phase 4 is everything on the server behind steps 1–5:** APIs, validation, storage, and reads from shared **JSON catalogs** (pricing, templates, and the same model/strategy metadata the rest of the app uses).

---

## The five building blocks of Phase 4

### 1. Projects API — “folders for real work”

**Idea:** Important work should not live as anonymous drafts. A **project** has a name and description and **owns** the configurations and history for that engagement.

**What to say to non-technical listeners:**

- Teams can **create**, **list**, **open**, **update**, and **soft-delete** projects.
- Listing is **paginated** (a few items per page), which keeps the system responsive as usage grows.
- This matches how **enterprise SaaS** normally organizes customer or internal programs.

---

### 2. Designer Config API — “the single source of truth for the blueprint”

**Idea:** The **pipeline configuration** is the full description of the RAG pipeline the user is designing.

**What to say:**

- Users can **create**, **load**, **update**, **list**, and **delete** configurations **inside a project**.
- The server **validates** the blueprint so invalid combinations are rejected early—fewer surprises later.
- Saved configs are what the **frontend**, **Autopilot**, **export**, and **cost** flows can all agree on.

---

### 3. Cost Calculation API — “from technical choices to dollars”

**Idea:** Finance and engineering should not maintain two different spreadsheets. The same structured blueprint sent for cost estimation is priced using **central pricing data** (embedding, storage, retrieval, optional reranking, generation, and so on).

**What to say:**

- You get **rough order-of-magnitude** cost: per query and at a chosen monthly query volume.
- You get a **breakdown**—where the money goes (models, storage, etc.).
- That supports **procurement conversations** and **architecture trade-offs** (quality vs. budget).

---

### 4. Export API — “make the blueprint portable”

**Idea:** Screenshots are not enough for production. **Export** turns the validated blueprint into **text artifacts** that other tools and teams already use.

**What to say:**

- Formats include **Python** (LangChain-style), **YAML**, **Terraform**, **Docker Compose**, and **Kubernetes** YAML.
- The response includes **content** and a **suggested filename**—easy to drop into a repo or ticket.
- This is the **bridge** between “we designed it here” and “we can build or review it in the standard toolchain.”

---

### 5. Templates API — “governed shortcuts”

**Idea:** Blank forms slow adoption. **Templates** are curated starter blueprints (FAQ bot, legal research, customer support, etc.) maintained in **catalog files**.

**What to say:**

- **Listing** templates does not require the database—it’s **read from validated catalog data**, so marketing and product can evolve presets without code changes.
- **Apply template** creates a **real saved pipeline configuration** using the **same save path** as manual design—nothing “second class.”
- Good for **standardization** and **faster time-to-first-value**.

---

## Where Phase 4 sits in the bigger picture (system design, simply)

**Upstream:** Shared **catalogs and schemas**—what models exist, what strategies are allowed, how much things cost, what templates are offered. Everyone reads the same rulebook.

**Phase 4:** The **Designer backend**—turn UI and API actions into **persisted**, **validated** blueprints, **prices** them, **exports** them, and **materializes** templates into saved configs.

**Downstream:** Later phases and other modes (like **Autopilot**) can use the **same blueprint format**, so manual design and assisted design don’t fork into incompatible worlds.

You can draw it mentally as: **Catalogs → Phase 4 APIs & services → Database + generated files → rest of the platform.**

---

## What Phase 4 is *not* (set expectations clearly)

This avoids over-selling in front of a panel:

- Phase 4 **does not** replace the **heavy RAG engines** built in Phase 2 (ingestion, chunking, embedding, vector stores, retrieval, generation, evaluation workers, etc.). Phase 4 **orchestrates and serves** the **design** side; those engines power **execution** and experiments elsewhere.
- **Export** outputs are **starting points**—real organizations still apply review, security review, and CI/CD.
- **User identity** today follows a **staging-style** pattern (for example a header standing in for a user id). Full **enterprise authentication** is explicitly a later hardening phase—not a hidden claim of Phase 4.

---

## Closing lines you can use verbatim

- “Phase 4 is the **backbone of Designer mode**: structured **projects**, authoritative **saved blueprints**, **cost transparency**, **export to engineering formats**, and **template-driven** onboarding.”
- “It’s how we show the product is not only a pretty configurator—it’s **data-backed**, **storable**, **priced**, and **handoff-ready**.”

---

*Aligned with roadmap tasks P4-1 through P4-5: Projects API, Designer Config API, Cost API, Export API, Templates API.*
