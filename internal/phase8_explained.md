# Phase 8 explained — for recruiters and non-technical audiences

*Reading time: about 5 minutes. You can read this aloud in an interview as a structured story.*

---

## One sentence: what Phase 8 is

**Phase 8 connects our two ways of building an AI knowledge system**—the manual “designer” path and the automatic “autopilot” path—**so the same project can move back and forth**, and **so we can measure quality and prepare deployment using shared, reliable steps behind the scenes.**

If someone only remembers one idea, let it be that: **two modes, one bridge, one shared foundation.**

---

## Why this matters (the business picture)

Our product helps teams build **RAG** systems—tools that **find the right documents** and **answer questions** using those documents. We offer:

- **Designer mode:** a person configures the pipeline step by step (like choosing ingredients for a recipe).
- **Autopilot mode:** the system tries different settings automatically and proposes a strong configuration.

**Before Phase 8**, those two paths did not fully “talk” to each other. A team might design something manually and then **re-enter everything from scratch** to try autopilot—or finish autopilot but **not see the result in the same review screen** they use for manual work.

**After Phase 8**, handoffs are intentional: **drafts travel with the user**, results can **come back into the designer view**, and the platform has **standard ways to evaluate and deploy** tied to the same saved project.

---

## The four building blocks of Phase 8 (simple names)

Think of Phase 8 as **four upgrades** that stack on top of each other:

| # | Plain name | What it does in human terms |
|---|--------------|------------------------------|
| 1 | **Carry my draft into Autopilot** | “I designed something; use it as the starting point when I run Autopilot.” |
| 2 | **Bring Autopilot’s answer back to Designer** | “Show me the winning setup in the same review screen I already know.” |
| 3 | **Measure quality on demand** | “Run a structured quality check on a saved configuration—and compare two setups if needed.” |
| 4 | **Deploy and track deployments** | “Start a deployment from a saved configuration, check status, list what’s deployed, and tear down when appropriate.” |

You do **not** need to say “API” or “database” in a panel interview unless asked. You can say **“a standard service in the platform”** or **“the backend keeps a record.”**

---

## Block 1 — Carrying a Designer draft into Autopilot

**Story you can tell:**  
Imagine a customer spends time in **Designer** picking chunk sizes, embedding models, and retrieval behavior. When they click to **optimize with Autopilot**, we **don’t throw that work away**. We **attach their draft** to the new Autopilot run so the system **starts from their choices** instead of from zero.

**Important nuance (one line):**  
The draft describes **how** the pipeline should behave. The user still **uploads their own documents** for Autopilot to learn from—we don’t magically move files for them in this step.

**Why recruiters care:**  
It shows **respect for the user’s time** and **continuity** between product modes—not two disconnected products.

---

## Block 2 — Bringing Autopilot’s result back into Designer

**Story you can tell:**  
When Autopilot finishes, the customer gets metrics and explanations. With Phase 8, they can also **open the winning configuration in Designer**—the same **visual summary, cost view, and export** they would see if they had built it by hand.

**Why recruiters care:**  
It supports **trust and transparency**: “Show me the recipe, not just the headline.” It also fits **enterprise** storytelling—auditability and review before production.

---

## Block 3 — Quality measurement everyone can anchor on

**Story you can tell:**  
Teams need to answer: **“Is this configuration good enough?”** Phase 8 adds a **consistent way** to run those checks against a **saved pipeline configuration**: run an evaluation, look up past runs, and **compare two configurations** (for example, “before vs after” or “manual vs autopilot”).

**How to say it without jargon:**  
“We run structured quality checks and keep **repeatable records** so decisions aren’t based on a one-off demo.”

**Why recruiters care:**  
It’s the difference between a **demo tool** and something a team can **iterate on with evidence**.

---

## Block 4 — Deployment you can trigger and track

**Story you can tell:**  
When a configuration is ready, Phase 8 supports **starting a deployment from that saved configuration**, **checking progress and endpoints**, **seeing deployments grouped under a project**, and **recording a teardown** when something should be taken offline or decommissioned for that record.

**How to say it without jargon:**  
“The platform knows **what was deployed**, **for which project**, and **in what state**—instead of deployments living only in someone’s terminal history.”

**Honest product note (if asked):**  
Early stages may use **stub or simulated endpoints** for certain environments; the **workflow and records** are real so production hardening can follow without redesigning the user journey.

**Why recruiters care:**  
It shows a path from **experiment** to **operational handoff**—relevant for “MVP to scale” questions.

---

## The “system design” in plain words (for the whole room)

You can close with this **three-layer story**:

1. **People layer:** Users work in **Designer** or **Autopilot** in the browser. Phase 8 makes it **easy to switch** without losing context.
2. **Agreement layer:** Both modes rely on the **same idea of a pipeline configuration** saved under a **project**—so handoffs aren’t a copy-paste hack.
3. **Operational layer:** The platform can **run evaluations** and **manage deployment records** against those same saved configurations—so **quality and release** aren’t separate silos.

That is **system design** in everyday language: **one shared idea of the product**, **two experiences**, **connected workflows**.

---

## Optional analogy (30 seconds)

**Designer** is like **writing a meal plan** yourself. **Autopilot** is like **asking a nutrition coach** to improve it. Phase 8 is the moment you can **hand the coach your written plan** (not start from an empty page) and then **paste the improved plan back into your own notebook** so you can **adjust spices and budget** before you **cook for guests** (deploy).

---

## If they ask “what did we technically deliver?”

You can answer briefly:

- **User experience bridges** between Designer and Autopilot (handoff and return path).
- **Backend services** to **evaluate** and **deploy** pipeline configurations with **clear records** tied to projects.

You stay high-level unless they want details.

---

## Quick glossary (say only if someone asks)

| Term | Simple meaning |
|------|----------------|
| **Pipeline configuration** | The full recipe: how documents are split, embedded, searched, and how the AI answers. |
| **Project** | A container in the product for the customer’s work (configs, builds, deployments). |
| **Evaluation** | A structured quality check on how well the system answers using retrieved context. |
| **Deployment** | Packaging or placing the system where it can run for users (staging or production), with status tracked in the product. |

---

*End of document — Phase 8 in non-technical terms.*
