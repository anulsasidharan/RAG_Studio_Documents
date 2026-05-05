# Phase 9 explained (simple interview version)

## What Phase 9 does in one line

Phase 9 gives our Autopilot system a "memory book" (MLflow) so every build can be compared, explained, and audited later.

## Simple analogy

Think of Autopilot as a chef trying recipes.

- Before Phase 9, we only kept the final dish.
- After Phase 9, we also keep a recipe card: ingredients used, settings, taste score, and notes.

That recipe card is what MLflow gives us.

## Why this phase matters

Without Phase 9, teams can still run builds, but it is hard to answer:

- Which run was best and why?
- What exactly changed between two runs?
- Can we prove how a production setup was decided?

Phase 9 solves this by capturing each run in a standard experiment tracker.

## How it works (non-technical flow)

1. User starts an Autopilot build.
2. Background worker completes the build.
3. Worker logs run details to MLflow (settings, quality metrics, and artifacts).
4. Main product database remains the source of truth.
5. If MLflow is unavailable, the build still succeeds; tracking is "best effort," not a blocker.

## What gets stored in MLflow

- **Run identity:** build ID, project ID, and status.
- **Inputs:** goals like quality/cost/speed targets.
- **Key decisions:** chunking, embedding, retrieval choices.
- **Outcomes:** evaluation metrics and iteration count.
- **Artifact:** compact JSON snapshot for review/debugging.

## Business value recruiters understand

- Better accountability: "show your work" for AI decisions.
- Faster team learning: compare many runs quickly.
- Better governance: stronger audit and compliance story.
- Lower risk: tracking failures do not break production builds.

## 30-second interview script

"Phase 9 introduced MLflow-based experiment tracking for Autopilot runs. Every build now has a structured record of inputs, decisions, and outcomes, so we can compare runs and explain why a configuration was chosen. Importantly, tracking is non-blocking, so customer builds continue even if MLflow has a temporary issue."
