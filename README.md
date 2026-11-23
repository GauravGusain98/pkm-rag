# PKM-RAG

A local, private knowledge base. You upload PDFs, Word documents, Markdown, plain text and code files, then ask questions and get answers that cite the source document and page number. Everything runs on your own machine, including the LLM.

> **Status: early. Phase 0 of 16.** At the moment the repo has the plan, the architecture decisions and the local infrastructure (Postgres and Redis). There's no application code yet. See [docs/PROGRESS.md](docs/PROGRESS.md) for where things stand.

This is also a learning project. I'm building it to learn the technologies and patterns properly, so the repo includes the build plan, decision records and the AI-assistant setup I use to work through it.

## Planned features

- Upload PDF, DOCX, MD, TXT and source-code files, with deduplication by content hash
- Background parsing, chunking and embedding, so uploads never block the API
- Semantic, keyword and hybrid search
- RAG answers that stream in and cite pages and headings
- Per-user collections, with access control enforced in SQL before ranking
- Document versioning and re-indexing
- A retrieval evaluation harness, so changes to chunking or ranking are measured rather than guessed
- AI-generated summaries and flashcards

## Stack

| Layer | Choice |
|---|---|
| API | FastAPI, SQLAlchemy 2.0 (async), Alembic |
| Database | PostgreSQL 16 with pgvector and full-text search |
| Jobs | Celery, with Redis as the broker |
| Cache | A separate Redis instance (LRU, no persistence) |
| Embeddings | Sentence-Transformers, run in the worker |
| LLM | Ollama, run natively on the host (default `llama3.2:3b`) |
| Frontend | React and TypeScript (Vite, TanStack Query) |
| File storage | Local, content-addressed (`storage/<sha256[:2]>/<sha256>`) |

## Architecture

```
 React + TS ──REST/SSE──▶ FastAPI ──enqueue──▶ Redis (queue) ──▶ Celery workers
                            │  ▲                                  parse → chunk → embed
                            │  └── Redis (cache)                        │
                            ▼                                           ▼
                 PostgreSQL + pgvector + FTS ◀───────────────────────────┘
                            ▲
                 Ollama (native on host) ◀── answer generation, summaries
```

The full target design is in [docs/PLAN.md](docs/PLAN.md#1-architecture-overview). [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) describes only what has actually been built.

### Key decisions

Each one has an Architecture Decision Record (ADR) in [docs/adr/](docs/adr/):

- [ADR-0001](docs/adr/0001-pgvector-vs-qdrant.md): pgvector instead of a separate vector database, so vectors sit next to the access-control data and filtering is one SQL query.
- [ADR-0002](docs/adr/0002-ollama-native.md): Ollama runs natively on macOS because Docker on a Mac can't use the Metal GPU.
- [ADR-0005](docs/adr/0005-redis-topology.md): two Redis instances, so the job queue (durable, no eviction) and the cache (disposable, LRU) fail independently.

## Running it locally

What works today is the infrastructure.

**Prerequisites:** Docker, [Ollama](https://ollama.com), and later [uv](https://docs.astral.sh/uv/) with Python 3.12 and Node 22.

```bash
cp .env.example .env          # local dev defaults, adjust if you like
docker compose up -d          # Postgres + pgvector (5432), redis-queue (6380), redis-cache (6379)
docker compose ps             # all three should show as healthy

ollama pull llama3.2:3b       # a 3B model runs on an 8 GB machine
```

`make` targets (`make up`, `make api`, `make worker`, `make test`, `make eval`) and the backend and frontend arrive in the next phases.

## Roadmap

| Phase | Topic |
|---|---|
| 0 | Foundations: infrastructure, ADRs, tooling *(in progress)* |
| 1 | FastAPI skeleton, database and migrations |
| 2 | Authentication and access control |
| 3 | Upload, storage and deduplication |
| 4 | Background processing with Celery |
| 5 | Parsing to a common document representation |
| 6 | Chunking |
| 7 | Embeddings and pgvector |
| 8 | Semantic, keyword and hybrid search |
| 9 | Retrieval evaluation |
| 10 | RAG answering with citations |
| 11 | Conversation history and caching |
| 12 | Document versioning and re-indexing |
| 13 | AI summaries and flashcards |
| 14 | Frontend consolidation |
| 15 | Hardening, observability and polish |

The details of each phase are in [docs/PLAN.md](docs/PLAN.md).

## Repository layout

```
docs/
  PLAN.md           phase-by-phase build plan
  PROGRESS.md       current phase and next steps
  ARCHITECTURE.md   what exists in the code today
  adr/              architecture decision records
  context/          per-subsystem contracts (filled in as subsystems land)
  learning/         learning journal
AGENTS.md           guide for AI coding assistants working in this repo
.claude/            Claude Code settings and project skills (tutor, quiz, review, checkpoint)
docker-compose.yml  Postgres + pgvector and the two Redis instances
```

## Working with AI assistants

[AGENTS.md](AGENTS.md) sets the rules for any AI assistant in this repo. Each task in the plan is tagged **[YOU]**, **[PAIR]** or **[AI]**, which decides whether the assistant only mentors, writes scaffolding with TODOs, or writes the code itself. The file also lists the invariants that must never break, such as "every retrieval path takes `allowed_collection_ids` explicitly".
