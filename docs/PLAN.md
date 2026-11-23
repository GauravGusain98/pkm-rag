# PKM-RAG — Build Plan

A local Personal Knowledge Management system: upload PDFs, DOCX, Markdown, TXT and code files, then ask questions and get answers **with citations and page numbers** from your own documents.

This plan is written for **learning**, not just shipping. Every phase says what to learn, what you write yourself, what AI may help with, and how you'll know you understood it.

> **For AI agents:** don't read this whole file. Read `docs/PROGRESS.md` to find the current phase, then read only that phase's section here. The `start-phase` skill does this for you.

---

## 0. How to use this plan

### 0.1 Help levels (attached to every task)

| Tag | Meaning | What the AI does | What you do |
|---|---|---|---|
| **[YOU]** | Core learning. | Explains concepts, asks questions, gives hints one level at a time (see `tutor` skill), reviews your code. **Never writes the solution unless you say "show me".** | Write all of it. |
| **[PAIR]** | Important, with some boring parts. | Writes scaffolding or the boring parts (types, wiring) and leaves `# TODO(you):` where the core logic goes. | Write the TODOs, then explain the scaffold back in your own words. |
| **[AI]** | Plumbing that isn't the point (configs, Dockerfiles, CSS). | May generate it. | Read every line. Add a one-line comment for anything you couldn't have predicted. |

Rule of thumb: **if it's an engineering challenge listed in the brief, it's [YOU].**

### 0.2 The loop for each phase

1. `/start-phase N`: concept primer, reading list, 3 warm-up questions, task breakdown.
2. Read the primary source (docs or paper) **before** coding. Spend 20–40 minutes on it.
3. Build the tasks. Use `/tutor` when stuck, and climb the hint ladder rather than asking for the answer.
4. `/review-my-code`: a mentor-style review that explains *why*, not a rewrite.
5. Meet the phase's **Done when** criteria (tests pass and the demo works).
6. Answer the **Checkpoint questions** out loud or in `docs/learning/journal.md`, without looking anything up.
7. `/checkpoint`: updates `PROGRESS.md`, the journal, the context files and ADRs, and asks a 2–3 question mini-quiz. **This is what keeps future sessions cheap.**
8. **Phase quiz (required before the phase counts as done):** `/quiz-me phase N`. It's a deep quiz that builds a concept map of the phase and climbs four levels for each concept. Pass mark: at least 70% overall, with at least one correct L3 answer.
   - **L1 Beginner:** what is it, and why does it exist?
   - **L2 Intermediate:** how does it work, and how do you use it here?
   - **L3 Advanced:** edge cases, failure modes, debugging.
   - **L4 Professional:** scale, trade-offs, defending the decision in a design review.
9. **Decision quizzes:** after writing an ADR, run `/quiz-me ADR-NNNN`. It asks you to defend the decision against the options you rejected.
10. **Concept quizzes, any time:** `/quiz-me pgvector`, `/quiz-me idempotency` and so on, for cross-phase topics.
11. **Review quiz** every 2–3 phases: `/quiz-me` with no topic gives you spaced repetition that re-asks your past misses in new wording.

**Quiz bank:** every quiz question, your answer, the verdict, the model answer and the source go into **one file**, `docs/learning/quiz-bank.md`. It's created once, every quiz appends to it, and it's **gitignored** (personal, never committed). Re-read your `wrong` and `partial` entries before interviews: `grep -n -A6 "Verdict: wrong" docs/learning/quiz-bank.md`.

### 0.3 Time estimate

About 10–14 weeks at 8–10 hours a week. Phases 1–8 (the core RAG pipeline) take about 60% of that. Don't skip the evaluation phase (Phase 9). It's what moves you past "basic tutorial" level.

---

## 1. Architecture overview

```
 ┌──────────── React + TS (Vite) ────────────┐
 │ Upload · Library · Search · Chat · Cards  │
 └───────────────┬───────────────────────────┘
                 │ REST + SSE (streaming answers)
 ┌───────────────▼──────────────┐       ┌──────────────┐
 │ FastAPI (API process)        │──────▶│ Redis        │ broker, result backend,
 │ auth · upload · search · RAG │◀──────│              │ cache, locks
 └──────┬───────────────┬───────┘       └──────▲───────┘
        │ SQL           │ enqueue              │ consume
 ┌──────▼──────────┐    │            ┌─────────┴─────────┐
 │ PostgreSQL      │◀───┼────────────│ Celery workers    │
 │ + pgvector      │    │            │ parse→chunk→embed │
 │ + FTS (tsvector)│    │            │ summaries, cards  │
 └─────────────────┘    │            └─────────┬─────────┘
        ▲               ▼                      │
        │        ./storage (files,             │
        │        content-addressed)            │
        │                                      ▼
        │                         Sentence-Transformers (in worker)
        └──────── Ollama (native on host, LLM generation) ◀── API (RAG) / worker (summaries)
```

**Key decisions (write an ADR for each in Phase 0):**
- **Ollama runs natively on macOS, not in Docker.** Docker on Mac can't use the Metal GPU. Containers reach it at `host.docker.internal:11434`.
- **pgvector instead of a separate vector database.** Vectors live next to the access-control data, so filtering and joins are a single SQL query. This is the main reason challenge 3 is tractable.
- **Celery for anything slow.** The API never parses or embeds a file inside a request.
- **Content-addressed storage.** Files are stored at `storage/<sha256[:2]>/<sha256>`, which makes deduplication natural.

### 1.1 Repository layout (target)

```
pkm-rag/
├── AGENTS.md  CLAUDE.md  README.md  Makefile  docker-compose.yml  .env.example
├── backend/
│   ├── pyproject.toml            # uv-managed
│   ├── alembic/                  # migrations
│   ├── app/
│   │   ├── main.py               # FastAPI app factory
│   │   ├── core/                 # config, db session, logging, security
│   │   ├── models/               # SQLAlchemy ORM models
│   │   ├── schemas/              # Pydantic request/response models
│   │   ├── api/routes/           # thin HTTP layer only
│   │   ├── services/             # business logic (no FastAPI imports)
│   │   │   ├── ingestion/        # upload, dedup, versioning
│   │   │   ├── parsing/          # one parser per format → common Document IR
│   │   │   ├── chunking/         # strategies behind one interface
│   │   │   ├── embedding/        # model wrapper, batching
│   │   │   ├── retrieval/        # semantic, keyword, hybrid, rerank
│   │   │   ├── rag/              # prompt building, citations, query rewrite
│   │   │   ├── llm/              # Ollama client behind an interface
│   │   │   └── study/            # summaries, flashcards
│   │   ├── repositories/         # DB access, always scoped by user/collection
│   │   └── workers/              # celery_app.py, tasks/*.py
│   └── tests/{unit,integration,fixtures}
├── frontend/src/{api,features/{library,search,chat,study},components,lib}
├── eval/{datasets,runs,run_eval.py}
├── docs/
│   ├── PLAN.md  PROGRESS.md  ARCHITECTURE.md
│   ├── adr/                      # architecture decision records
│   ├── context/                  # one short file per subsystem, for agents
│   └── learning/                 # journal.md, concepts.md
├── storage/                      # gitignored uploads
└── .claude/{settings.json,skills/}
```

### 1.2 Data model (target; it grows phase by phase)

```
users(id, email, password_hash, created_at)
collections(id, name, owner_id)                          -- a "knowledge base"
collection_members(collection_id, user_id, role)         -- owner | editor | viewer
documents(id, collection_id, title, source_type, active_version_id NULL, created_by, deleted_at)
document_versions(id, document_id, version_no, content_hash, storage_path, mime_type,
                  size_bytes, status, error, embedding_model, created_at, indexed_at)
     UNIQUE(document_id, version_no)
     status ∈ {uploaded, queued, parsing, chunking, embedding, ready, failed, superseded}
chunks(id, version_id, document_id, collection_id,       -- collection_id denormalized for filtering
       ordinal, text, token_count, page_start, page_end, heading_path, char_start, char_end,
       tsv tsvector GENERATED, embedding vector(384))
     INDEX hnsw(embedding vector_cosine_ops), GIN(tsv), btree(collection_id)
processing_jobs(id, version_id, kind, status, attempts, last_error, celery_task_id,
                idempotency_key UNIQUE, started_at, finished_at)
conversations(id, user_id, collection_id, title, created_at)
messages(id, conversation_id, role, content, citations JSONB, retrieval_debug JSONB, created_at)
summaries(id, version_id, level, content, model)
flashcards(id, version_id, question, answer, source_chunk_ids, ease, interval, due_at)
```

---

## Phase 0: Foundations and AI context setup (≈1 week)

**Goal:** A repo where `make up` starts Postgres with pgvector, Redis and the API, and where an AI agent gets oriented in under 2k tokens.

### Concepts to learn
- **uv** (Python packaging): lockfiles and virtual environments. Why lockfiles matter.
- **Docker Compose**: services, volumes, networks, healthchecks, `depends_on: condition: service_healthy`.
- **12-factor config**: settings come from the environment, not from code.
- **ADRs**: why you write down *decisions*, not just code. (Read Michael Nygard's "Documenting Architecture Decisions".)
- **How coding agents use context**: CLAUDE.md/AGENTS.md are loaded every session, so every line costs tokens on every turn. Skills load only when triggered (progressive disclosure).

### Tasks
1. **[AI]** `git init` and a `.gitignore` covering `storage/`, `.venv`, `node_modules`, `.env` and `eval/runs/`.
2. **[PAIR]** `docker-compose.yml` with `pgvector/pgvector:pg16` and `redis:7`, healthchecks and named volumes. *You* write the healthchecks and explain `pg_isready`.
3. **[AI]** Install Ollama natively, then `ollama pull llama3.1:8b` (or `qwen2.5:7b`). **With 8 GB of RAM, use `llama3.2:3b` or `qwen2.5:3b` instead.** An 8B model plus Docker plus an embedding model won't fit. Verify with `curl localhost:11434/api/tags`.
4. **[PAIR]** Set up `backend/` with uv. Dev tools: ruff, mypy (strict on `services/`), pytest, pytest-asyncio and pre-commit.
5. **[YOU]** Write ADR-0001 to ADR-0004 (pgvector vs. Qdrant, native Ollama, Celery vs. FastAPI BackgroundTasks, content-addressed storage). ADR-0005, the Redis topology, came out of the compose task. One page each: Context, Decision, Consequences.
6. **[YOU]** Review `AGENTS.md` (already scaffolded). Cut anything you don't actually believe, and add your own rules.
7. **[AI]** `Makefile` targets: `up`, `down`, `api`, `worker`, `test`, `lint`, `migrate`, `eval`.

### Agent-context files (already created; you maintain them)

| File | Loaded when | Purpose | Size budget |
|---|---|---|---|
| `CLAUDE.md` | every session | One line, `@AGENTS.md` | 1 line |
| `AGENTS.md` | every session | Stack, commands, rules, learning-mode policy, pointers | < 120 lines |
| `docs/PROGRESS.md` | start of each session | Current phase, done/next, open questions | < 60 lines |
| `docs/ARCHITECTURE.md` | on demand | Current real architecture (not the aspirational one) | < 150 lines |
| `docs/context/<subsystem>.md` | on demand | Contract, invariants, gotchas for one subsystem | < 60 lines each |
| `docs/adr/NNNN-*.md` | on demand | Why things are the way they are | 1 page |
| `docs/learning/journal.md` | by `checkpoint` and `quiz-me` | Your notes and checkpoint answers | unbounded (agents grep it) |
| `docs/learning/quiz-bank.md` | by `quiz-me` (grep only) | **All** quiz questions and answers in one file. Gitignored. | unbounded (agents grep it) |
| `.claude/skills/*` | when triggered | Workflows | < 100 lines each |
| `.claude/settings.json` | harness | Denies reads of `storage/`, `.venv`, `node_modules`, `eval/runs` | — |

**Token-saving rules (also in AGENTS.md):**
- Point to files instead of pasting content into AGENTS.md.
- Each subsystem's `docs/context/*.md` is the *contract*: inputs, outputs, invariants. An agent reads that instead of the whole module.
- After Phase 14, the generated OpenAPI schema (`frontend/src/api/schema.d.ts`) is the single source of truth for the API shape. Don't describe endpoints in prose.
- `PROGRESS.md` is rewritten (not appended to) at every checkpoint, so it stays short.

### Done when
- `make up && make api` serves `GET /health` → `{"db": "ok", "redis": "ok", "ollama": "ok"}`.
- 4 ADRs are committed.
- A fresh agent session asked "what phase am I on and what's next?" answers correctly after reading only `AGENTS.md` and `PROGRESS.md`.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Why can't the Dockerized API reach Ollama at `localhost:11434`?
2. What's the difference between `depends_on` with and without `condition: service_healthy`?
3. Why is a skill cheaper than putting the same text in CLAUDE.md?

---

## Phase 1: FastAPI skeleton, database and migrations (≈1 week)

**Goal:** A layered FastAPI app with async SQLAlchemy 2.0, Alembic migrations, settings and structured logging.

### Concepts to learn
- **ASGI and async in Python**: the event loop, `async def` vs. `def` endpoints in FastAPI (threadpool!), and why blocking calls inside `async def` hurt.
- **FastAPI dependency injection**: `Depends`, `yield` dependencies for DB sessions, overriding dependencies in tests.
- **SQLAlchemy 2.0**: `Mapped[]` typing, `AsyncSession`, the unit-of-work pattern, and lazy loading pitfalls with async.
- **Alembic**: autogenerate (and its limits), and keeping migrations reversible.
- **Layered architecture**: routes (HTTP), then services (logic), then repositories (data). Read about "Ports and Adapters" (hexagonal) at a high level.
- **Pydantic v2 and pydantic-settings**.

### Tasks
1. **[PAIR]** App factory `create_app()`, `core/config.py` (pydantic-settings) and `core/db.py` (async engine and session dependency).
2. **[YOU]** The `users`, `collections` and `collection_members` models, plus the first Alembic migration. Write the migration's `downgrade()` by hand.
3. **[YOU]** Structured JSON logging with a request-ID middleware (generate an ID, put it in a contextvar, include it in every log line).
4. **[PAIR]** A test setup using a real Postgres (testcontainers, or the compose DB with a separate test database). Use a transaction-per-test rollback fixture. *You* write the fixture.
5. **[YOU]** A global exception handler that maps domain errors (e.g. `NotFound`, `Forbidden`) to an HTTP problem-details JSON body.

### Done when
- `alembic upgrade head` followed by `downgrade base` works cleanly.
- `pytest` passes, and tests don't leak data between each other.
- Every log line carries a `request_id`.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. You call a sync `requests.get()` inside an `async def` route. What happens under load, and why?
2. Why shouldn't `services/` import anything from `fastapi`?
3. What does Alembic autogenerate miss? (Hint: think about some index types and server defaults.)

**Context file to create:** `docs/context/backend-core.md` (layers, how to add an endpoint, how DB sessions work).

---

## Phase 2: Authentication and access control (≈1 week) — *challenge 3 starts here*

**Goal:** Users log in, own collections and share them with roles. **Every data access is scoped by authorization from day one**, because bolting it on later is how data leaks happen.

### Concepts to learn
- Password hashing (argon2 or bcrypt). Why plain SHA-256 is wrong for passwords.
- JWT access tokens: claims, expiry, and why you don't put permissions *in* the token for this app.
- **Authorization vs. authentication.** RBAC with collection roles.
- **Insecure Direct Object Reference (IDOR)**, the #1 bug in multi-tenant apps.
- (Stretch) Postgres **Row-Level Security** as defense in depth.

### Tasks
1. **[PAIR]** Register and login endpoints, password hashing, JWT issuing, and a `current_user` dependency.
2. **[YOU]** `AccessPolicy` service: `can_read(user, collection)` and `can_write(...)`, plus a repository helper `accessible_collection_ids(user)`.
3. **[YOU]** Collections CRUD and member management (owner only).
4. **[YOU]** **IDOR test suite:** for every endpoint, user B must get 404 (not 403, so B can't even learn the resource exists) on user A's resources. Make it parametrized so new endpoints get added to it.
5. **[YOU, stretch]** Enable RLS on `chunks` and `documents` with `SET LOCAL app.user_id`, as a second safety net.

### Done when
- The IDOR suite passes for every endpoint that exists so far.
- ADR-0006, "Authorization model and where it's enforced", is written.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Why return 404 instead of 403 for other users' documents?
2. Where exactly is the authorization check enforced, and what stops a future endpoint from forgetting it?
3. What attack does RLS protect against that your service-layer check doesn't?

**Context file:** `docs/context/auth.md`.

---

## Phase 3: Upload, storage and deduplication (≈1 week) — *challenge 1*

**Goal:** Upload files safely, store them content-addressed, and never process the same bytes twice.

### Concepts to learn
- Streaming uploads (`UploadFile`, chunked reads) vs. loading the whole file into memory.
- **Content hashing** (SHA-256 computed while streaming) and content-addressable storage (how Git stores objects).
- MIME sniffing vs. trusting the file extension or `Content-Type` (python-magic or `filetype`).
- **Idempotency** and **DB unique constraints as the real guard** against duplicates. Check-then-insert is a race; the constraint isn't.
- **The dedup side channel:** if dedup works *across users*, user B can learn that user A uploaded a specific file. Decide the scope deliberately.

### Tasks
1. **[YOU]** A streaming upload service: hash while writing to a temp file, then atomically `os.replace` it into `storage/ab/abcdef…`. Enforce a size limit and an allowlist of formats.
2. **[YOU]** `documents` and `document_versions` models and migration, with `UNIQUE(collection_id, content_hash)` on an active-versions view or index (you design this).
3. **[YOU]** Dedup logic:
   - Same bytes in the same collection: return the existing document, `200` plus `"deduplicated": true`.
   - Same bytes in a different collection: store a new document row but **reuse the stored file**. Embedding reuse comes in Phase 7.
   - Handle a concurrent identical upload by catching `IntegrityError` and returning the winner's row.
4. **[YOU]** A test that fires 10 concurrent identical uploads and asserts that exactly 1 version is created.
5. **[AI]** Frontend slice (start the Vite app here): an upload page with drag-and-drop, a progress bar and a documents list.

### Done when
- The concurrency test passes reliably (run it 20 times).
- Uploading a `.exe` renamed to `.pdf` is rejected.
- ADR-0007, "Dedup scope and the privacy side channel", is written.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Why isn't "SELECT, and if not found then INSERT" safe?
2. Why hash while streaming rather than after the upload?
3. You chose per-collection dedup. What do you lose, and how could you still share *embeddings* across users safely? (Revisit this in Phase 7.)

**Context file:** `docs/context/ingestion.md`.

---

## Phase 4: Background processing with Celery (≈1 week) — *challenges 1 and 5*

**Goal:** Uploading enqueues a pipeline. Jobs are idempotent, retry with backoff and fail *visibly*.

### Concepts to learn
- Message brokers, at-least-once delivery, and why **every task must be idempotent**.
- Celery basics: tasks, `acks_late`, `task_reject_on_worker_lost`, `autoretry_for`, `retry_backoff`, `max_retries`, chains vs. a single orchestrator task, and worker prefetch (`worker_prefetch_multiplier=1` for long tasks).
- **State machines** for job and version status. Allowed transitions only.
- **Compare-and-set** state transitions: `UPDATE … SET status='parsing' WHERE id=:id AND status='queued' RETURNING id`. Zero rows returned means someone else has it.
- **The transactional outbox problem:** you commit the DB row, then enqueue, and the process crashes in between. What happens? (Read "Transactional Outbox" on microservices.io.)
- Poison messages, dead-letter handling and observability (Flower).

### Tasks
1. **[PAIR]** `celery_app.py` with a Redis broker. Wire it into compose as a `worker` service. *You* choose and justify each config flag in comments.
2. **[YOU]** `processing_jobs` table and a status state machine (a pure function `next_status(current, event)` that raises on an illegal transition, with unit tests).
3. **[YOU]** An `ingest_version(version_id)` orchestrator task. Steps are stubs for now; real ones come in Phases 5–7.
   - It claims the job with compare-and-set (safe if the task is delivered twice).
   - It's an idempotent step design: each step can re-run without duplicating chunks (delete-then-insert inside one transaction, or upsert on `(version_id, ordinal)`).
4. **[YOU]** Failure handling:
   - Transient errors (Ollama down, DB blip) are retried with exponential backoff and jitter.
   - Permanent errors (corrupt PDF) mark the version `failed` immediately with a user-readable `error`. No retry.
   - After max retries, mark it `failed` and record `last_error` and `attempts`.
5. **[YOU]** A **reaper** (Celery beat, every 5 minutes) that finds jobs stuck in a processing state longer than N minutes (the worker died) and re-enqueues them or fails them.
6. **[YOU]** A retry endpoint, `POST /versions/{id}/reprocess`, for failed versions. Add a "Retry" button in the UI.
7. **[YOU]** Solve the outbox gap: either enqueue in `after_commit`, or let the reaper also pick up `queued` rows that are older than X. Document which you chose.

### Done when
- Kill the worker mid-job (`docker kill`). After restart and the reaper running, the job completes and no duplicate chunks exist.
- Delivering the same task twice produces one result (test it by calling the task function twice).
- ADR-0008, "Job reliability: idempotency, retries, reaper", is written.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. What does `acks_late=True` buy you, and what new problem does it create?
2. How do you tell a transient error from a permanent one? Give two examples of each for this pipeline.
3. Walk through what happens if the API commits the version row and crashes before `delay()` is called.

**Context file:** `docs/context/workers.md` (task list, state machine diagram, retry policy).
**Skill exercise:** write your own `.claude/skills/debug-pipeline/SKILL.md`. It should tell an agent how to inspect a stuck document: which tables to query, Flower, logs by `version_id`. Use the `superpowers:writing-skills` guidance, but write it yourself.

---

## Phase 5: Parsing to a common document representation (≈1 week)

**Goal:** Every format becomes the same intermediate representation (IR), which preserves **page numbers and structure** for citations.

### Concepts to learn
- How PDFs actually store text (positioned glyphs, not paragraphs), so extraction is lossy. Reading order, headers and footers, ligatures, hyphenation.
- Libraries: **PyMuPDF** (`fitz`) for PDFs, `python-docx` for DOCX (DOCX has **no pages**, so decide what a "location" means: heading path and paragraph index), `markdown-it-py` for Markdown structure, and plain text plus language detection for code.
- The **Strategy / Registry pattern**: `Parser` protocol, registered by MIME type (open/closed principle).
- Scanned PDFs and OCR (just know they exist; stretch with `ocrmypdf`).

### IR (you design it; a suggested shape)
```python
@dataclass(frozen=True)
class Block:
    text: str
    kind: Literal["heading", "paragraph", "code", "list", "table"]
    page: int | None          # 1-based; None for formats without pages
    heading_path: tuple[str, ...]  # ("Ch 5", "Sagas")
    char_start: int; char_end: int # offsets into the full normalized text

@dataclass(frozen=True)
class ParsedDocument:
    title: str | None
    blocks: list[Block]
    metadata: dict[str, str]
```

### Tasks
1. **[YOU]** The `Parser` protocol and registry, plus a `parse(path, mime) -> ParsedDocument` entry point.
2. **[YOU]** PDF parser: per-page extraction, drop repeated headers and footers (lines that repeat on more than 50% of pages), fix hyphenation.
3. **[PAIR]** DOCX parser: headings via paragraph styles; tables become text.
4. **[YOU]** Markdown parser: heading hierarchy becomes `heading_path`, and fenced code becomes `kind="code"`.
5. **[YOU]** TXT and code-file parser (`.py`, `.ts` and so on): for code, put each top-level function or class in its own block, using Python's `ast` module for `.py` and plain text otherwise.
6. **[YOU]** Golden tests: put small fixture files in `tests/fixtures/` and snapshot the IR output.

### Done when
- The System Design Primer PDF parses, and `page` matches what a PDF viewer shows (spot-check 5 pages).
- A password-protected or corrupt PDF raises a `PermanentParseError` and the version is marked `failed` (Phase 4 wiring).

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Why parse into an IR instead of chunking straight from each library's output?
2. What does "page number" mean for a DOCX citation, and what did you decide?
3. How does adding EPUB support touch the codebase? (The answer should be "one new file".)

**Context file:** `docs/context/parsing.md`.

---

## Phase 6: Chunking (≈1 week)

**Goal:** Split the IR into retrievable chunks that keep their citation metadata. Build **several strategies** so Phase 9 can compare them.

### Concepts to learn
- Why chunk at all: embedding models have context limits, and smaller units retrieve more precisely. The trade-off between precision and context.
- Tokens vs. characters. Use the embedding model's tokenizer for counting.
- Strategies: fixed-size with overlap, recursive (paragraph, then sentence, then word), structure-aware (respect headings and code blocks), and (later) semantic chunking.
- **Contextual chunk headers:** prepend `Document title > Heading path` to the text you embed (but not to what you display). This is cheap and very effective.
- Parent/child ("small-to-big") retrieval: embed small chunks, return the larger parent.

### Tasks
1. **[YOU]** A `Chunker` protocol: `chunk(doc: ParsedDocument) -> list[Chunk]`. A chunk carries `text`, `embed_text`, `page_start`, `page_end`, `heading_path`, `char_start`, `char_end` and `token_count`.
2. **[YOU]** `FixedTokenChunker(size, overlap)`.
3. **[YOU]** `StructureAwareChunker`: never splits mid-code-block, starts a new chunk at a heading change, merges tiny blocks and splits huge ones recursively.
4. **[YOU]** Page-span tracking: a chunk that crosses pages 41–42 records both pages.
5. **[YOU]** Property-based tests with **Hypothesis**: there's no text loss (concatenated chunks minus overlap equals the source), `token_count <= max`, and pages are monotonic.
6. **[PAIR]** Wire chunking into the pipeline step, and store `chunks` rows (no embeddings yet).

### Done when
- Hypothesis tests pass.
- A debug endpoint or CLI prints a document's chunks with their page ranges, and they look sane.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. What fails if chunks are too small? Too large?
2. Why is `embed_text` different from `text`?
3. What invariant does your property test check that example-based tests would likely miss?

**Context file:** `docs/context/chunking.md`.

---

## Phase 7: Embeddings and pgvector (≈1 week)

**Goal:** Embed chunks in the worker, store them in pgvector, and index them with HNSW.

### Concepts to learn
- What an embedding is. Cosine similarity vs. dot product vs. L2, and why **normalized** vectors make cosine and dot product equivalent.
- **Model choice:** `BAAI/bge-small-en-v1.5` (384 dimensions, strong) or `all-MiniLM-L6-v2` (384 dimensions, fast). Read the model card. Some models want a query prefix (BGE: `"Represent this sentence for searching relevant passages: "`).
- The **MTEB leaderboard**: how to read it, and why your own evaluation (Phase 9) matters more.
- Vector indexes: **HNSW** vs. IVFFlat, and the parameters `m`, `ef_construction` and `ef_search`. Approximate search means recall below 100%.
- **pgvector filtering gotcha:** with an HNSW index, a `WHERE collection_id = ANY(...)` filter is applied *after* the approximate search, so you can get fewer than k results. Learn `hnsw.ef_search` and pgvector's iterative index scans (`hnsw.iterative_scan`, pgvector ≥ 0.8).
- Batching and throughput. Load the model **once per worker process** (`worker_process_init`), never per task.
- **Embedding versioning:** store `embedding_model` with every vector. Changing the model means re-indexing (Phase 12).

### Tasks
1. **[PAIR]** An `Embedder` interface plus a `SentenceTransformerEmbedder` (batching, normalization, separate query and document methods).
2. **[YOU]** Add `embedding vector(384)` to chunks, in a migration that also creates the HNSW index. Write the index DDL by hand.
3. **[YOU]** The embedding pipeline step: batch by 64, write in bulk, and make it idempotent.
4. **[YOU]** **Embedding cache by content:** key it by `(sha256(embed_text), model)` in a table or Redis. Identical chunks across documents or collections reuse vectors. This is the *safe* cross-user dedup you couldn't do at the file level in Phase 3, because the cache never reveals ownership.
5. **[YOU]** A benchmark script: embed 1,000 chunks and report chunks per second on CPU vs. MPS (Apple GPU, `device="mps"`).

### Done when
- Uploading the System Design Primer reaches `ready`, and the `chunks` rows all have embeddings.
- `EXPLAIN ANALYZE` shows the HNSW index being used for a k-NN query.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Why do BGE queries need a prefix and documents don't?
2. Your filtered vector query returns 3 results when you asked for 10. Why, and what are two fixes?
3. Why is the per-chunk embedding cache safe across users when the per-file dedup wasn't?

**Context file:** `docs/context/embeddings.md`.

---

## Phase 8: Search — semantic, keyword and hybrid (≈1 week) — *challenge 3 enforced in SQL*

**Goal:** A `/search` endpoint with three modes, always scoped to the documents the user can access.

### Concepts to learn
- **Postgres full-text search:** `tsvector`, `tsquery`, `websearch_to_tsquery`, `ts_rank_cd`, GIN indexes, language configs. A generated column for `tsv`.
- **BM25** in theory, and why Postgres `ts_rank` isn't BM25. (Stretch: the `pg_search` / ParadeDB extension, or doing the BM25 math yourself.)
- When keyword search beats semantic search: exact identifiers, error codes, function names, acronyms ("2PC").
- **Reciprocal Rank Fusion (RRF):** `score = Σ 1/(k + rank_i)` with `k = 60`. Why fuse ranks instead of raw scores (the score scales aren't comparable).
- **Rerankers / cross-encoders** (`BAAI/bge-reranker-base`): a bi-encoder retrieves, then a cross-encoder reorders the top 50.
- **Access filtering placement:** pre-filter (inside SQL) vs. post-filter (after retrieval). Post-filtering leaks through timing and counts, and returns fewer results.

### Tasks
1. **[YOU]** `SemanticRetriever`, `KeywordRetriever` and `HybridRetriever` (RRF) behind one `Retriever` protocol. Each takes `allowed_collection_ids` as a **required** argument, so there's no default and it can't be forgotten.
2. **[YOU]** Only `active_version_id` chunks are searchable: join on `documents.active_version_id = chunks.version_id`. This matters in Phase 12.
3. **[YOU]** Leakage tests: user B searches for a phrase that exists only in user A's document. Zero results, in all three modes.
4. **[PAIR]** An optional reranking stage behind a flag.
5. **[PAIR]** A debug response mode that returns each chunk's semantic rank, keyword rank and fused score. You'll use this constantly.
6. **[AI]** Frontend: a search page showing results, highlighted snippets, document and page, and a mode toggle.

### Done when
- Searching "2PC" finds Two-Phase Commit content (keyword helps). Searching "how do distributed transactions stay consistent without locking" finds Saga content (semantic helps). Hybrid gets both.
- The leakage tests pass.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Why RRF rather than `0.5 * cosine + 0.5 * ts_rank`?
2. Name a query where semantic search fails and keyword search wins, from your own corpus.
3. Why is `allowed_collection_ids` a required parameter and not read from a global?

**Context file:** `docs/context/retrieval.md`.

---

## Phase 9: Retrieval evaluation (≈1 week) — *challenge 4. Don't skip this.*

**Goal:** Measure retrieval quality with numbers, so every later change (chunk size, hybrid, reranker, prompt) is a measured experiment, not a vibe.

### Concepts to learn
- Ground-truth datasets: (query → relevant chunk IDs or relevant document plus page).
- Metrics: **Recall@k**, **Hit rate@k**, **MRR**, **nDCG@k**, and precision@k. Know what each rewards.
- **Synthetic query generation** with the LLM: given a chunk, generate a question it answers. Know the bias this introduces (lexical overlap) and how to counter it (ask for paraphrases, and add hand-written questions).
- Answer-level evaluation (used in Phase 10): **faithfulness / groundedness**, answer relevance and citation correctness. **LLM-as-judge** and its failure modes. Read the RAGAS paper or docs for the definitions, then implement simple versions yourself.
- Experiment hygiene: a fixed dataset version, logged configs, and comparing runs.

### Tasks
1. **[YOU]** Hand-write 30 questions over your real corpus, with the relevant document and page(s). Include hard ones: acronyms, multi-hop, "difference between X and Y", and a few that have **no answer** in the corpus.
2. **[PAIR]** A synthetic generator: 100 or more questions from random chunks via Ollama, filtered for quality. Store them in `eval/datasets/*.jsonl`, versioned.
3. **[YOU]** `eval/run_eval.py`: run every retriever config, compute the metrics, and write `eval/runs/<timestamp>.json` plus a markdown table.
4. **[YOU]** Run the experiments and record the results in `docs/learning/journal.md`:
   - Fixed vs. structure-aware chunking; chunk sizes 256, 512 and 1024.
   - Semantic vs. keyword vs. hybrid; with and without the reranker; with and without contextual headers.
5. **[YOU]** Pick defaults **based on the numbers** and write ADR-0009, "Retrieval configuration, chosen by eval".
6. **[YOU]** Write the `rag-eval` skill (`.claude/skills/rag-eval/SKILL.md`) yourself. It should tell an agent how to run the evaluation, where results live, how to compare with the previous run, and to flag regressions above 2 points.

### Done when
- `make eval` prints a comparison table, and you can explain *why* the winner won.
- CI (or a pre-push hook) runs a 20-question smoke evaluation and fails on a recall regression.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Your Recall@5 is 0.92 on synthetic questions but your hand-written ones score 0.61. What's going on?
2. When would you optimize MRR over Recall@k?
3. What are two ways LLM-as-judge can mislead you?

**Context file:** `docs/context/eval.md`.

---

## Phase 10: RAG answering with citations (≈1.5 weeks)

**Goal:** `POST /ask` streams a grounded answer with numbered citations that map to document, page and chunk.

### Concepts to learn
- The **Ollama API**: `/api/chat`, streaming NDJSON, `options.num_ctx` (the default context is small!), temperature, and `format` for JSON output.
- **Prompt construction:** a system prompt with grounding rules, numbered context blocks `[1] (System Design Primer, p. 42) …`, and an instruction to cite as `[n]` and to say "I don't know" when the context is insufficient.
- **Context budgeting:** count tokens, fit the top chunks within `num_ctx` minus room for the answer, and order them sensibly (the "lost in the middle" effect).
- **Citation validation:** parse `[n]` from the output, drop invalid numbers, and map them to chunk metadata. Optionally check that each cited chunk actually supports its sentence.
- **Server-Sent Events (SSE)** with FastAPI `StreamingResponse`, and the event types `token`, `citations` and `done`.
- **Prompt injection from documents:** an uploaded document can contain "ignore previous instructions…". Use delimiters, an instruction hierarchy, and treat retrieved text as data.

### Tasks
1. **[PAIR]** An `LLMClient` interface and `OllamaClient` (streaming, timeouts, retry on connection errors). Add a `FakeLLM` for tests.
2. **[YOU]** A `PromptBuilder`: a pure function (question, chunks, budget) → messages. Unit test it heavily.
3. **[YOU]** A `CitationResolver`: answer text plus the chunk list → a structured `citations[]` with `{n, document_id, title, page_start, page_end, snippet}`.
4. **[YOU]** The `/ask` SSE endpoint: retrieve (Phase 8 defaults), build the prompt, stream tokens, then emit the citations event.
5. **[YOU]** Answer-level evaluation: extend `run_eval.py` with faithfulness (an LLM judge checks each claim against the cited chunks), citation precision and a "no-answer" correctness rate.
6. **[YOU]** An injection test: upload a document containing malicious instructions, ask a question that retrieves it, and assert the system prompt holds.
7. **[AI/PAIR]** Frontend chat: streaming text, `[n]` rendered as chips, and clicking a chip opens a side panel with the snippet and page. Stretch: `react-pdf` jumps to the page.

### Done when
- "What is the difference between the Saga pattern and Two-Phase Commit?" returns a correct answer that cites the right pages.
- A question with no answer in the corpus gets "I couldn't find this in your documents", not a hallucination.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Why might the model ignore half your context even though it "fits"?
2. How do you stop a model from citing `[7]` when you only gave it 5 chunks?
3. What's your defense against prompt injection from uploaded documents, and what's its weakness?

**Context file:** `docs/context/rag.md`.

---

## Phase 11: Conversation history and caching (≈1 week)

**Goal:** Multi-turn chat where follow-ups like "and what about its failure modes?" work, with Redis caching where it pays off.

### Concepts to learn
- **Query rewriting / condensation:** (history + follow-up) → a standalone search query. Retrieval needs the rewritten query; the LLM answer needs the history.
- History windowing: the last N turns plus a running summary of older ones.
- **Caching patterns:** cache-aside, TTLs, and **invalidation** (the hard part). What's safe to cache: query embeddings, yes. Final answers: only if keyed on the active versions of the collection.
- Redis data structures: strings with TTL, and `SET NX` for locks.

### Tasks
1. **[YOU]** `conversations` and `messages` tables. Store `citations` and `retrieval_debug` (rewritten query, chunk IDs, scores) on each assistant message. This is invaluable for debugging.
2. **[YOU]** A `QueryRewriter` (an LLM call with a tight prompt), skipped on the first turn. Add evaluation cases: 10 multi-turn conversations.
3. **[YOU]** History budget management inside the `PromptBuilder`.
4. **[YOU]** Redis query-embedding cache keyed by `(model, sha256(query))`.
5. **[YOU, stretch]** Answer cache keyed by `(collection_id, collection_index_version, normalized_question)`. Bump `collection_index_version` whenever any document in the collection changes, which gives you free invalidation.
6. **[AI]** Frontend: a conversation sidebar, rename and delete.

### Done when
- The multi-turn evaluation cases retrieve the right chunks after rewriting.
- The cache hit rate is visible in the logs.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Why not just embed the raw follow-up question?
2. Explain how the version counter makes cache invalidation trivial. What's the trade-off?

**Context file:** `docs/context/chat.md`.

---

## Phase 12: Document versioning and re-indexing (≈1 week) — *challenge 2*

**Goal:** Uploading a new version of a document indexes it in the background **while the old version keeps serving queries**, then switches atomically. Changing the embedding model re-indexes everything without downtime.

### Concepts to learn
- **Blue/green deployment** applied to data: build the new index next to the old one, then swap a pointer.
- **Atomic pointer swap:** `UPDATE documents SET active_version_id = :new WHERE id = :doc AND active_version_id = :expected_old`. That's compare-and-set again.
- **Stale job guard:** a slow job for v2 finishes after v3 was uploaded. It must *not* activate v2. Use monotonic `version_no` and check at activation time.
- MVCC and transaction isolation in Postgres: what a concurrent search sees during the swap.
- Garbage collection: delete superseded chunks later, never in the hot path.

### Tasks
1. **[YOU]** `POST /documents/{id}/versions` (upload a new version). Dedup against the current active version's hash first (no-op if unchanged).
2. **[YOU]** Activation step at the end of the pipeline: in one transaction, CAS the pointer, mark the old version `superseded`, and bump `collection_index_version`. If the CAS fails because a newer version is already active, mark this version `superseded` without activating it.
3. **[YOU]** A GC task: delete chunks of `superseded` versions older than X hours. Keep the version rows for history.
4. **[YOU]** **Model migration:** a `reindex_collection(model=...)` task embeds into a new column or table keyed by model, and the retriever reads the model in a `settings.active_embedding_model` switch. Document the procedure in an ADR.
5. **[YOU]** Race tests:
   - Searches running continuously during the v1 → v2 swap never return 0 results and never mix versions.
   - v2 is slow and v3 is fast, so v3 ends up active and v2 never gets activated.
6. **[AI]** Frontend: a version history per document, status badges, and "re-index" (admin).

### Done when
- Both race tests pass.
- ADR-0010, "Versioning and zero-downtime re-indexing", is written.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Why isn't "delete the old chunks, then insert the new ones" acceptable?
2. Walk through the v2-slow / v3-fast race with your code. Where exactly is it prevented?
3. How do you re-index 10,000 documents onto a new embedding model without downtime?

**Context file:** update `docs/context/ingestion.md` with a versioning section.

---

## Phase 13: AI summaries and flashcards (≈1 week)

**Goal:** Background-generated document summaries and flashcards that are grounded in their source chunks, with a spaced-repetition review flow.

### Concepts to learn
- Long-document summarization: **map-reduce** (summarize the chunks, then the summaries) vs. **refine** (iterative). Hierarchical summaries per section.
- **Structured output:** Ollama `format` with a JSON schema, validated with Pydantic. On failure, retry once with the validation error in the prompt.
- Flashcard quality: one fact per card, a question that isn't trivially guessable, and the source chunk linked back for verification.
- The **SM-2** spaced-repetition algorithm (as used by Anki): ease factor, intervals and grading 0–5.

### Tasks
1. **[YOU]** A `summarize_version` task using map-reduce by section, storing summary levels (section and document).
2. **[YOU]** `generate_flashcards` with a JSON schema, Pydantic validation, dedup of near-identical cards (by embedding similarity above 0.92), and `source_chunk_ids`.
3. **[YOU]** An SM-2 scheduler as a pure function with unit tests, plus the endpoints `GET /cards/due` and `POST /cards/{id}/review`.
4. **[YOU]** Grounding check: reject cards whose answer isn't supported by the source chunk (reuse your Phase 10 faithfulness judge).
5. **[AI/PAIR]** Frontend: a summary tab per document, and a flashcard review screen (flip card, grade buttons, "show source").

### Done when
- Generated summaries and cards are regenerated automatically when a new version is activated (hooked into Phase 12).
- SM-2 has unit tests for intervals over 5 reviews.

### Checkpoint questions
_Answer these yourself first. Then run `/quiz-me phase N` for the full L1–L4 quiz (the phase isn't done until you pass it)._
1. Map-reduce vs. refine: which do you lose global context with, and which is slower?
2. What do you do when the LLM returns invalid JSON three times?

**Context file:** `docs/context/study.md`.

---

## Phase 14: Frontend consolidation (≈1 week, plus slices along the way)

The frontend has been growing in slices since Phase 3. This phase cleans it up.

### Concepts to learn
- **Generated API types:** `openapi-typescript` from FastAPI's `/openapi.json`, so the backend is the single source of truth.
- **TanStack Query:** server state vs. client state, query keys, invalidation and polling (`refetchInterval` while a document is processing).
- Consuming SSE with `fetch` plus `ReadableStream` (`EventSource` can't send auth headers).
- Feature-folder architecture, error boundaries and suspense.
- Accessibility basics: keyboard navigation for chat and flashcards.

### Tasks
1. **[PAIR]** `npm run gen:api` generates `src/api/schema.d.ts` plus a typed fetch client.
2. **[YOU]** Refactor the features to TanStack Query, replacing ad-hoc `useEffect` fetches.
3. **[YOU]** A `useAskStream` hook that parses the SSE events into `{tokens, citations, status}`.
4. **[YOU]** Document status that updates live (polling, or SSE via Redis pub/sub as a stretch).
5. **[AI]** Styling and layout (Tailwind or CSS modules).
6. **[PAIR]** Component tests with Vitest and Testing Library for the chat and upload flows. One Playwright end-to-end test: upload, ready, ask, see a citation.

### Done when
- A backend schema change breaks the TypeScript build, rather than failing at runtime.
- The end-to-end test passes locally.

**Context file:** `docs/context/frontend.md`.

---

## Phase 15: Hardening, observability and polish (≈1 week)

### Concepts to learn
- Observability: structured logs, metrics and traces. **OpenTelemetry** auto-instrumentation for FastAPI, SQLAlchemy and Celery. The three golden signals.
- Rate limiting (a Redis token bucket), request timeouts, and back-pressure on the queue.
- Backups: `pg_dump` plus the storage folder, and a restore drill.
- Security review: uploads, auth, injection and the IDOR suite.

### Tasks
1. **[PAIR]** OpenTelemetry tracing, with spans for retrieve, rerank, LLM time-to-first-token and the total. View them locally in Jaeger (one compose service).
2. **[YOU]** Metrics: ingestion latency per stage, failure rate by error type, cache hit rate and retrieval latency p95.
3. **[YOU]** Per-user rate limits on `/ask` and upload.
4. **[YOU]** A backup and restore script, plus actually restoring it into a fresh database once.
5. **[YOU]** Run `/security-review`. Fix the findings yourself, using the review as a learning resource.
6. **[YOU]** Write `README.md` for a stranger: architecture diagram, setup, design decisions (link the ADRs), and evaluation results.

### Done when
- A trace for one `/ask` request shows where the time goes.
- The restore drill succeeds.
- The README alone gets a fresh machine running.

---

## Engineering challenges: where each is solved

| Challenge | Phases | Mechanisms |
|---|---|---|
| **Prevent duplicate processing** | 3, 4, 7 | Content hash plus a DB unique constraint, `IntegrityError` handling, compare-and-set job claiming, idempotent steps (upsert or delete-insert in one transaction), embedding cache by content hash |
| **Document updated while the old version is indexed** | 12 | Separate version rows, background build, atomic CAS pointer swap, stale-job guard via `version_no`, deferred GC, search reads only `active_version_id` |
| **No retrieval from inaccessible documents** | 2, 8, 10 | `AccessPolicy`, a required `allowed_collection_ids` in every retriever, pre-filtering in SQL, denormalized `collection_id` on chunks, 404 not 403, IDOR and leakage test suites, optional RLS, per-collection dedup scope (side channel) |
| **Evaluate retrieval accuracy** | 9, 10, 11 | Golden and synthetic datasets, Recall@k, MRR and nDCG, answer faithfulness and citation precision, multi-turn cases, a regression gate in CI |
| **Failed background jobs** | 4, 12 | State machine, transient vs. permanent errors, backoff with jitter, `acks_late`, a reaper for stuck jobs, outbox gap handling, the reprocess endpoint, visible error messages in the UI |

---

## AI tooling and skills inventory

**Created up front (in `.claude/skills/`):**

| Skill | Trigger | What it does |
|---|---|---|
| `start-phase` | `/start-phase N` | Reads PROGRESS plus only phase N of PLAN. Gives a concept primer, the reading list, warm-up questions and a task breakdown with help levels. |
| `tutor` | `/tutor`, "I'm stuck", "explain" | Socratic help with a 4-rung hint ladder. Won't write [YOU] code unless you say "show me". |
| `review-my-code` | `/review-my-code` | A mentor review: correctness, patterns, *why*. Asks questions, points to one concept to study, doesn't rewrite. |
| `checkpoint` | `/checkpoint`, end of session | Rewrites PROGRESS.md, appends to the journal, updates the context file and ADRs. Keeps future sessions cheap. |
| `quiz-me` | `/quiz-me [phase N \| concept \| ADR-NNNN]` | Leveled quiz from beginner (L1) to professional (L4) on a phase, concept or decision. It has deep, review and checkpoint modes, and appends every question and answer to the single gitignored `docs/learning/quiz-bank.md`. |

**You write these yourself, as learning exercises:**

| Skill | Phase | Purpose |
|---|---|---|
| `debug-pipeline` | 4 | How to diagnose a stuck or failed document |
| `rag-eval` | 9 | Run the evaluation, compare with the last run, flag regressions |
| `new-migration` | any time after 1 | Your checklist for writing safe Alembic migrations (indexes, `CONCURRENTLY`, downgrade) |
| `add-parser` | 5 | The steps to add a new file format: a test fixture, the parser, registration |

**Also consider:** a project subagent `.claude/agents/security-reviewer.md` in Phase 15 (you already have a similar `adversarial-reviewer` in voice-assistant), and the existing `superpowers:test-driven-development` skill for Phases 4–6.

---

## Reading list (primary sources first)

- FastAPI docs: "Concurrency and async / await", "Dependencies", "Testing".
- SQLAlchemy 2.0 "Unified Tutorial" and the asyncio docs.
- pgvector README (HNSW, filtering, iterative scans).
- Postgres docs, chapter 12: "Full Text Search".
- Celery docs: "Tasks", especially the Retrying, Idempotence and `acks_late` sections.
- microservices.io: Transactional Outbox, Saga (bonus: it's also in your test corpus).
- Lewis et al. 2020, *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*.
- Liu et al. 2023, *Lost in the Middle*.
- Cormack et al. 2009, *Reciprocal Rank Fusion*.
- Anthropic, "Introducing Contextual Retrieval" (contextual chunk headers).
- The RAGAS docs (metric definitions only; implement your own).
- Sentence-Transformers docs: "Semantic Search", "Cross-Encoders".
- Ollama API docs (`/api/chat`, structured outputs).
- *Designing Data-Intensive Applications* (Kleppmann), chapters 7 (Transactions) and 11 (Streams), for Phases 4 and 12.
