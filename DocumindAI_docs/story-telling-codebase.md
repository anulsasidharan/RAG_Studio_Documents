# DocuMind AI — Storytelling Walkthrough (Interview Script)

**How to use this file:** Read it aloud a few times, then rely on memory, not verbatim reading. Aim for roughly **fifteen to twenty minutes** with natural pauses, one live demo beat, and room for interruptions. Think of it as a river: source, bends, tributaries, and the sea—not a filing cabinet.

Below is prose meant for speaking; what follows labelled “anchors” exists only so you rehearse—not so you chant acronyms in the panel.

### Timing cheat sheet (approximately)

| Stretch | Rough minutes |
|--------|----------------|
| Opening + Acts one–three | 8–11 |
| Acts four–five + one-breath summary | 3–5 |
| Live demo bead | 3–5 |
| One or two pivots from “if the river bends” | 1–3 |

If you have **fifteen minutes**, skip one pivot and keep the demo tight. If you have **twenty**, linger on Act two (retrieval judgment) and Act three (trust boundaries)—that is where senior interviewers lean in.

---

## Before you speak (twenty seconds)

You are not describing files. You are describing a journey: from messy documentation to trustworthy answers that stay anchored in what actually exists.

---

## The opening: why this exists

Picture a developer drowning in PDFs, internal wikis, and sprawling Markdown. Large language models *feel* smart, but if you feed them guesses, they will happily invent plausible nonsense. What I wanted was something calmer—a system that says, **“I will answer from your documents—or tell you plainly when they do not contain the answer.”**

That constraint is DocuMind AI’s personality. Everything in the codebase exists to honour it.

From here on, if you catch yourself listing paths, gently steer back to *forces*: trust, scope, and operability. File names are landmarks for you; the interviewer should feel *currents*.

---

## Act one — Giving the corpus a backbone

Imagine you have poured your documentation into one place. Nothing useful happens until three quiet things occur.

First, **the ears**: the system opens many kinds of documents—Markdown, plain text, PDF, HTML, Word, even CSV or JSON—not because format variety is flashy, but because real organisations never publish in one shape alone. Those readers live alongside the splitter in the ingestion side of the project.

Second, **the breathing room**. Long pages are sliced into overlapping windows. Humans read whole sections; retrieval works best when each chunk carries one coherent idea, with overlap so phrases are not orphaned at boundaries. Chunk size is not mystical—it’s a dial you tune once you observe how brittle answers become.

Third, **the map**. Each fragment is turned into a vector with a hosted embedding model, then stored under a stable identifier in **Qdrant**. At that moment, the corpus stops being scattered files on disk and becomes something you can navigate by *meaning*: “things close to what the user meant,” not merely matching keywords.

The heart orchestrating ingestion and question time is **`rag_pipeline`**. If you traced one path with your finger, you’d see: bring documents in, split them, embed them, write them once—then reuse that store every time someone asks a question.

There is also a thin legacy entry file that only whispers “call the pipeline”—a fossil of early sketches. I keep mentioning it briefly so nobody thinks the repository began as seventeen micro-services; it matured into structure when the behaviours stabilised.

**Memory anchor:** *Ingest • Chunk • Embed • Store.*

---

## Act two — The two-step retrieval dance

Pure vector search is fast and surprisingly good. It also has blind spots—it can retrieve chunks that merely *sound* related. So I layered a quiet second judgement on top.

When a question arrives, **Qdrant** returns more candidates than the model should ever swallow—think of it as casting a sensible net—and then another model, heavier but focused, reranks those candidates pairwise: “Given this precise question, which passages truly deserve attention?” Under the hood I use **LangChain’s compression retriever** with a Hugging Face **cross‑encoder**. The model weighs question against passage—that is the hallmark of reranking—not just vector proximity to the raw question embedding.

Whatever survives becomes the tight **context bundle** stitched into the system prompt before the conversational model speaks.

Why does this matter in an interview? Because you’re showing you understand the difference between *recall* (did we find something nearby?) and *precision* (did we hand the LLM the *right* paragraphs?). Reranking is where many toy RAG demos stop trying; I kept going.

Once passages are chosen, generation is disciplined by a narrator voice in the system prompt: cite where you learned it, refuse to spin fiction when passages are mute, wrap answers in Markdown so humans can skim. None of this replaces retrieval—but it reinforces the pact with the reader so that every token still feels tethered beneath the eloquence.

**Memory anchor:** *Cast a wide net; pass a narrow basket.*

---

## Act three — Memory that stays under control

Documentation Q&A is rarely one shot. People refine. They say “what about the previous step?” or “only in the deployment section.”

So the chain is wrapped in **conversation history** keyed by a simple **session id**. You get continuity without pretending the model remembers anything on its own—because it does not. The programme remembers transcripts; the retrieval layer still anchors every answer to stored chunks.

Independently, chunks carry a **document id** stamped at index time. When the caller passes one or several ids—say, “only answer from last week’s upload”—similarity runs through a filtered lens. Suddenly the same codebase serves both exploratory search across everything and disciplined answers inside a perimeter.

Clearing sessions when switching scope avoids ghost context from haunting the wrong document. Tiny detail, enormous trust.

**Memory anchor:** *Sessions for dialogue; ids for fences.*

---

## Act four — Opening the gates without losing the spine

Internals are worthless if nobody can reach them politely. FastAPI fronts the orchestration—not as bureaucracy, but as a contract anyone can reuse.

Someone can **inspect health**, **query** with optional document scope, **list what is indexed** so the corpus is inspectable rather than mythical, **upload** new material—or **upload and ask immediately** scoped to what just landed. There is **clear-session** for when intent shifts mid-demo.

Behind each route hides the same story: ingest when needed; otherwise retrieve, rerank, generate.

Treat **upload pathways** as a slower heartbeat than **question pathways**. Organisations ingest in batches yet interrupt you with hourly questions—the architecture echoes that asymmetry rather than cramming everything into anonymous CRUD choreography.

Streamlit is the friendly shoreline. It does not reinvent business logic—it calls the API via HTTP—so demo day never depends on accidental duplication. You can sketch the UX in one phrase: sidebar for documents and uploads, main canvas for grounded chat, heartbeat to prove the spine is awake.

**Memory anchor:** *API owns truth; UI only knocks.*

---

## Act five — Running it like infrastructure

At home the stack settles into Docker Compose—**vector database**, **API**, **UI** tied together with sensible overrides so inside the containers Qdrant and the API URLs resolve cleanly. Volumes persist the index across restarts so you are not babysitting brittle state.

Uploads optionally mirror to Google Cloud Storage when credentials permit; if not configured, the path shrinks gracefully to local indexing—the point is readiness for production habits without blocking local iteration.

Indexing also exists outside the GUI: scripts walk directories—batch loads for when you ingest a handbook, not single files.

Later, richer batch orchestration could move into a pipeline module; the story already names the seat even if the film is still in post.

Across this act, if someone asks whether you optimised for GCP day one, translate it into intent: repeatable units, declarative environments, artefacts that deserve object storage—all of that is already implied in how upload and batch paths are separated from question-time paths.

**Memory anchor:** *Compose for shape; GCS when the cloud is ready.*

---

## The river from source to sea (one breath)

If they ask you to tie it together in one sentence, you might say:

**Documents land, become vectors in Qdrant; each question pulls candidates, reranks them, injects history and filters if needed; the prompt demands grounded answers; FastAPI exposes that contract; Streamlit demonstrates it—all runnable as services when you want to behave like grown-up infra.**

Say it slower in the room. Let the comma breathe.

---

## Optional live demo bead (three to five minutes)

If time allows:

1. Show **health**, then indexed documents—or index a small Markdown file.
2. Ask one broad question, then tighten scope if your UI exposes document selection.
3. Mention rerun without naming every variable: retrieval width, rerank depth, chunk size—all live where configuration is centralised so swapping models or collections is declarative rather than surgical.

Pause after the first answer so they absorb that the answer cites structure, not vibes.

---

## If the river bends (short pivots—no cramming)

- **Why Qdrant?** Efficient filtered vector queries, pragmatic self-host story, aligns with GCP-friendly deployment narratives.
- **Why not lexical search too?** Honest variant: dense plus reranking first; lexical blend is an obvious sibling when synonym drift hurts.
- **Cost and latency:** Embeddings batched offline; rerank cost is narrower than blindly stuffing everything into GPT; cross-encoder is local CPU which trades hosting complexity for predictable API billing on that step.
- **Evaluation:** Offline you watch retrieval sanity; live you observe citation behaviour and hallucination refusal when chunks do not justify the claim—your prompt insists on admitting absence.

Deliver these almost as whispered annotations, then return upstream to the storyteller rhythm.

---

## Five beats to memorize by heart

1. **Grounded answers** beat clever ones—we index, retrieve, constrain.
2. **Chunk and overlap** marry human structure with retrieval physics.
3. **Wide retrieve, narrow rerank** protects both recall and precision.
4. **Session memory plus document fences** tame multi-turn ambiguity.
5. **API-first, Compose-ready** separates product story from bedtime prototype.

Practice once while walking once while drawing the river on paper without labels—you should recognise each bend.

---

## Closing line (leave them downstream)

“I built DocuMind so that expertise stored in scattered documentation could behave like one calm expert in the room: quick to cite, reluctant to pretend, structured enough that another engineer could integrate it tomorrow—not because files say so, but because the narrative does.”

End with eye contact—not with a file path.
