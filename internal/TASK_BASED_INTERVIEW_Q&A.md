# Task-based interview — Unified RAG Studio (internal)

> Supplement for candidates reviewing implementation tasks. Answers reflect design intent and current codebase choices unless noted.

---

## Phase 3 · P3-3 — App layout & navigation

### Why split `Providers`, `AppShell`, and `layout.tsx`?

**Answer:** Next.js **server** `layout.tsx` should stay thin: fonts, metadata, and HTML shell. **React Query** must live in a **client** component (`providers.tsx`) because hooks cannot run in server components. **`AppShell`** is client-side because it uses pathname-aware UI (sidebar visibility, mobile drawer). **`StoreHydration`** stays separate so Zustand `persist` rehydration runs once at startup.

### Why is React Query added before API-heavy screens exist?

**Answer:** Phase 4+ will mount hooks against Projects and Designer APIs. Providing `QueryClientProvider` early avoids refactors and establishes defaults (`staleTime`, no aggressive refetch on focus).

### How does mode switching (Designer vs Autopilot) work?

**Answer:** **`ModeToggle`** uses `usePathname()` and `<Link>` to `/designer` and `/autopilot`. Active state is derived from the URL prefix, not duplicated global state—single source of truth for navigation.

### Why does the sidebar disappear on the home page?

**Answer:** Marketing/landing typically stays **full-bleed** without project chrome. All other primary routes show **Navbar + Sidebar** so project context stays visible during configuration workflows.

### How is the sidebar collapsible and persisted?

**Answer:** Desktop collapse toggles width (`md:w-14` vs wider panel). State is stored under **`rag-studio-sidebar-collapsed`** in `localStorage` so preference survives refreshes. Mobile uses **overlay + backdrop** and closes on route change.

### How does the project dropdown relate to the sidebar?

**Answer:** Both read **`useProjectStore`** (Zustand + persist). The navbar dropdown selects **`activeProjectId`**; the sidebar lists projects and supports **“New project”** via `addProject`. Until **P4-1 Projects API**, data is **local-only**.

### Why Radix `DropdownMenu` without shadcn wrappers?

**Answer:** This milestone prioritized shell behaviour over regenerating shadcn primitives. Radix is already a dependency; dropdown triggers remain keyboard-accessible and portal-rendered.

### What is the avatar placeholder?

**Answer:** **No auth yet** (Phase 12). “RS” initials mark reserved space for future user menu.

### What do `not-found.tsx` and `error.tsx` accomplish?

**Answer:** **`not-found`** renders for unknown routes with recovery links. **`error.tsx`** is a **client** boundary (required by Next.js) that logs errors and offers **`reset()`** plus navigation home.

### Font strategy: task mentions Geist—what shipped?

**Answer:** **`next/font/google`** loads **Inter** and **JetBrains Mono** with CSS variables **`--font-geist-sans`** and **`--font-geist-mono`** so Tailwind `font-sans` / `font-mono` align with the design token names. Teams may swap to the **`geist`** npm package later without changing Tailwind wiring.

### How would you test navigation in CI?

**Answer:** Component tests for **`ModeToggle`** active classes (mock `usePathname`), and Playwright smoke for `/` → `/designer` link (Phase 10). Until Vitest is wired, manual QA paths suffice for this branch.

---

## Cross-cutting (layout-adjacent)

### Where should shared loading UI live?

**Answer:** Route-level **`loading.tsx`** or shared **`LoadingSpinner`** under `components/shared/` (P3-1). Global skeleton policy is product-specific.

### How does hydration interact with the shell?

**Answer:** **Zustand persist** skips hydration on SSR; **`StoreHydration`** calls `persist.rehydrate()` on mount so navbar/sidebar see projects after client loads. **`suppressHydrationWarning`** on `<html>` avoids noise if theme toggles appear later.

---

## Behavioural / scenario questions

### “A user reports the sidebar won’t open on mobile.”

**Answer:** Verify **`showSidebarTrigger`** is true (not on `/`), confirm **`openSidebar`** wires to state toggling **`mobileSidebarOpen`**, and check z-index conflicts with other overlays.

### “Should Designer store drive which tab is active in ModeToggle?”

**Answer:** Prefer **URL as source of truth** for mode to avoid desync when opening links in new tabs or sharing URLs.

---

*Extend this file after each phase with new Q&A blocks keyed by task ID.*

---

## Phase 3 · P3-4 — Landing Page

### Why are the landing sections split into individual components instead of one big `page.tsx`?

**Answer:** Each section (`Hero`, `ModeComparison`, `HowItWorks`, `Features`, `UseCases`, `Pricing`, `CTA`) has distinct responsibilities and different data shapes. Splitting them keeps each file under ~100 lines, makes A/B testing individual sections trivial, and lets future phases swap or reorder sections without touching the assembly file.

### Why are all landing components server components (no `'use client'`)?

**Answer:** None of the landing sections need browser APIs or React state — they are purely presentational. Keeping them as RSC means zero client-side JS for these sections, faster initial page load, and better Lighthouse scores. The animated gradient orbs are CSS `@keyframes`, not JavaScript animations.

### How does the Hero animated gradient work without framer-motion?

**Answer:** Two absolutely-positioned `div`s with `blur-3xl` and `rounded-full` act as "orbs". They animate via custom `@keyframes float` added to `globals.css`, referenced by Tailwind utility classes `animate-float` and `animate-float-delayed` (added in the `@layer utilities` block). No JavaScript is involved — the animation is pure CSS.

### Why use `pointer-events-none` on the hero orbs?

**Answer:** The orbs are decorative overlays. Without `pointer-events-none` they would intercept mouse events on the CTA buttons below them (even though they are visually behind content due to `z-index`). The class ensures clicks always reach the intended interactive elements.

### How is the `gradient-text` utility reused across Hero and other components?

**Answer:** `gradient-text` is defined once in `globals.css` as a `@layer utilities` rule that applies `bg-clip-text`, `text-transparent`, and the linear-gradient. All landing sections that need the brand gradient on text simply add `className="gradient-text"` — no duplication.

### What design decisions were made for the Pricing tier cards?

**Answer:** Three tiers — Free, Pro, Enterprise — follow a standard SaaS pattern. The `popular` flag drives the visual highlight (primary border, gradient background, "Most Popular" pill). Feature rows use a discriminated `included: boolean | 'partial'` field: `false` renders a muted `Minus` icon, `true` renders a green `Check`, and `'partial'` renders `Check` with an inline note (e.g., "3 / month"). This avoids a separate "partial" icon and keeps the data model simple.

### Why is `Minus` from lucide-react used instead of an X icon for missing features in Pricing?

**Answer:** A `Minus` (em-dash style) is a softer visual signal than a red `X`. Red crosses create anxiety on marketing pages. The neutral grey `Minus` clearly communicates unavailability without penalizing the plan visually, which is a common SaaS conversion best practice.

### How does the `ModeComparison` section differ from the `HowItWorks` section?

**Answer:** `ModeComparison` answers "what does each mode do?" — it shows features, tone, and a CTA. `HowItWorks` answers "how do I use each mode?" — it shows numbered sequential steps. Separating the two prevents information overload in a single section and lets users who already understand the modes skip directly to the how-to.

### What persona archetypes are in `UseCases` and why those three?

**Answer:** **Learning Engineer** (uses Designer to understand RAG internals), **Time-Strapped Startup** (uses Autopilot for speed), and **Enterprise Architect** (uses both for compliance + validation). These map to the three primary segments in CLAUDE.md's target users: ML Engineers, AI Teams/Startups, and Enterprises. Each persona has a quote, a recommended mode tag, and a benefit list to make the value proposition concrete.

### How does the CTA section's color scheme reinforce the brand?

**Answer:** The CTA uses `bg-gradient-to-br from-primary-600 via-primary-700 to-purple-700`, matching the brand gradient direction established in `gradient-brand` and `gradient-text`. This creates visual consistency: the first and last sections both carry the brand gradient, creating a "bookend" effect that frames the entire page.

### What is the `page.tsx` assembly pattern and why not import everything inline?

**Answer:** `page.tsx` is a pure orchestration file — it imports and renders the 7 section components in order. No markup lives there. This mirrors Next.js App Router idioms where `layout.tsx` orchestrates shell and `page.tsx` orchestrates content sections. It also makes it trivial to reorder, remove, or feature-flag sections (e.g., hide `Pricing` for a beta period) without touching the section code.

### Why does the footer live in `page.tsx` rather than a dedicated `Footer.tsx` component?

**Answer:** The footer is intentionally minimal (3 lines of text). At this stage it does not justify its own component — the TASKS.md spec does not list a `Footer.tsx`, and adding premature abstractions conflicts with the project principle of not designing for hypothetical future requirements. If the footer grows (nav links, social icons, legal pages), extracting it then is straightforward.

### How would you test the landing page in CI?

**Answer:** Playwright `e2e/landing.spec.ts` (Phase 10) will: (1) verify the page loads at `/`, (2) assert the "Start Designing" CTA links to `/designer`, (3) assert "Launch Autopilot" links to `/autopilot`, and (4) check that no sidebar renders on `pathname === '/'`. Until Playwright is wired, `tsc --noEmit` and Next.js build serve as correctness gates.

---

## Phase 3 · P3-5 — Lib Utilities & Validators

### Why create Zod validators separately from TypeScript types?

**Answer:** TypeScript types are erased at compile time — they only verify static correctness. **Zod schemas** are runtime constructs that validate unknown data at system boundaries (form submissions, API responses, localStorage deserialization). Keeping them separate from types allows the type system and validation logic to evolve independently. The `z.infer<>` utility then derives TypeScript types from the Zod schema when needed, eliminating drift.

### How does `ChunkingConfigSchema` enforce the overlap-less-than-size invariant?

**Answer:** The schema adds a `.refine()` predicate after the base `z.object()` definition: `(data) => data.chunkOverlap < data.chunkSize`. Zod's `.refine()` runs post-field validation, receives the parsed object, and can produce a targeted error message on the `chunkOverlap` path. This cross-field constraint cannot be expressed in TypeScript's type system alone.

### Why does `RetrievalConfigSchema` require `hybridSearch` only when strategy is `hybrid`?

**Answer:** This is a **discriminated validation** — the field is contextually required. Using `.refine()` with a path on `hybridSearch`, the schema rejects configs where `strategy === 'hybrid'` but `hybridSearch` is absent. This enforces invariants that the TypeScript union types cannot express, keeping the backend from receiving an underspecified retrieval config.

### How does `generateMermaidDiagram` avoid injecting user content into the diagram syntax?

**Answer:** All user-supplied strings (model names, index names, source types) are passed through a `q()` sanitiser that strips `"`, `[`, and `]` characters before embedding them in node labels. Mermaid flowchart node labels that contain unescaped brackets or quotes break the diagram parser. The sanitiser is intentionally minimal — it strips the specific characters that Mermaid treats as syntax.

### Why does the Mermaid generator use two sub-graphs (indexing path and query path)?

**Answer:** RAG pipelines have two distinct flows: the **offline indexing path** (document → chunking → embedding → vector store) and the **online query path** (query → retrieval → optional reranking → generation → answer). Separating them into sub-graphs makes the architecture immediately legible and mirrors how engineers mentally model RAG systems. Using a single flat graph would create a confusing tangle of arrows.

### What is the LCEL pattern in the Python code generator output?

**Answer:** **LangChain Expression Language (LCEL)** composes runnables with the `|` pipe operator: `retriever | format_docs | prompt | llm | parser`. The generated code uses `RunnableParallel` to run context retrieval and the passthrough question in parallel, then feeds both into the prompt template. LCEL enables streaming, batching, and tracing out of the box without custom orchestration code.

### How does the Python code generator handle different vector store backends?

**Answer:** An internal `VECTORSTORE_IMPORTS` dictionary maps provider IDs (`qdrant`, `pinecone`, `chroma`, etc.) to provider-specific import strings and initialization code. The `buildVectorStore()` function uses a switch/case on `stages.vectorStore.provider` to emit the correct connection setup (e.g., `QdrantClient` + URL for Qdrant, `Pinecone()` + `pc.Index()` for Pinecone). Unrecognised providers emit a `# TODO` stub so the file is still syntactically valid.

### Why does the YAML generator avoid a third-party serialisation library?

**Answer:** Third-party YAML libraries (like `js-yaml`) would add a dependency and produce output we can't control (e.g., quoting strategy, key ordering). Since the pipeline config is a known, bounded shape, hand-building the YAML with helper functions (`yamlString`, `yamlBool`, `yamlArray`) gives us full control over formatting, comment placement, and indentation. It also removes a runtime dependency from the browser bundle.

### How does `yamlString` decide when to quote a value?

**Answer:** The helper tests for characters that YAML treats as special syntax (`:#[]{},|>&*!,`) and also checks for embedded newlines. If any are present, the value is wrapped in double quotes with internal double quotes escaped as `\"`. Plain scalar values (most model names and IDs) are emitted without quotes, producing cleaner YAML.

### What does the Terraform generator produce for multi-cloud configurations?

**Answer:** Multi-cloud falls back to **AWS** since Terraform providers are cloud-specific — there is no single "multi-cloud" provider. The raw cloud value `"multi-cloud"` is preserved in the header comment so the engineer knows which logical target it represents, but the generated HCL uses the AWS provider block, ECS, and Secrets Manager. Engineers building true multi-cloud deployments would extend the output to combine multiple provider blocks.

### How are the generator tests structured to balance specificity with maintainability?

**Answer:** Tests use **two complementary techniques**: (1) targeted string assertions (`expect(result).toContain(...)`) for stable structural properties — import paths, constant names, YAML keys — and (2) `toMatchSnapshot()` for the complete output. Targeted assertions catch regressions in specific features without over-constraining the full output. Snapshots catch unexpected global changes. If a snapshot fails after an intentional change, running `vitest --update-snapshots` regenerates it.

### Why are test fixtures (`fixtures.ts`) shared across all generator tests?

**Answer:** A `minimalConfig` and a `fullConfig` fixture defined once in `__tests__/fixtures.ts` eliminate duplication and ensure every generator is tested against the same known inputs. If the `PipelineConfiguration` shape changes, only the fixtures need updating — not every test file. The fixtures also double as documentation of the valid config surface area.

### How does snapshot testing complement unit assertions for the generators?

**Answer:** Unit assertions verify **individual properties** (e.g., "the output contains the model name"), but snapshots verify **the entire serialised output** at once. This is particularly valuable for YAML and Terraform where whitespace, ordering, and overall structure matter. The first run writes the snapshot to disk; subsequent runs diff against it. Intentional changes are accepted with `--update-snapshots`; accidental regressions fail CI.

### Why are generator functions pure (no side effects)?

**Answer:** Pure functions — those that take inputs and return a string without reading files, environment variables, or global state — are trivially testable (no mocking needed), cacheable, and safe to call from both server components and client-side export buttons. The generators are called from the frontend code export UI and from the backend export API; purity lets them work identically in both environments.

### Why install Vitest in P3-5 rather than waiting for P10-3 (Frontend Unit Tests)?

**Answer:** P3-5 explicitly includes "unit tests for all generators". Running those tests requires a test runner. Installing **Vitest** now is a one-line devDependency addition that does not conflict with P10-3 (which adds React Testing Library and full component coverage on top of the same Vitest runner). Waiting would leave the generator tests in the repo as untestable stubs, which defeats the purpose of writing them.
