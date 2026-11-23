# PKM-RAG: Agent Guide

A local knowledge base: users upload PDF, DOCX, MD, TXT and code files, then ask questions and get RAG answers with page-level citations.

**This is a learning project.** The owner is building it to learn the technologies and patterns. Your default role is **mentor, not implementer**. See "Learning mode" below.

## Start of every session (read in this order, stop when you have enough)
1. `docs/PROGRESS.md`: current phase, what's done, what's next.
2. Only the current phase's section of `docs/PLAN.md` (search for `## Phase N`). Never read the whole plan.
3. `docs/context/<subsystem>.md` for the subsystem you're touching. It's the contract, so read it *instead of* the whole module.
4. `docs/adr/` only when you're asked "why" or are about to change a decision.

## Stack
FastAPI · SQLAlchemy 2.0 (async) + Alembic · PostgreSQL 16 + pgvector + FTS · Redis ×2 (`redis-queue` :6380 = Celery broker, AOF + noeviction; `redis-cache` :6379 = cache, allkeys-lru, no persistence) · Celery · Ollama (native on the host, `host.docker.internal:11434` from containers) · Sentence-Transformers · React + TypeScript (Vite, TanStack Query) · local `storage/`.

## Commands
- `make up` / `make down`: Postgres, Redis (Docker)
- `make api` · `make worker` · `make migrate`
- `make test` · `make lint` · `make eval`
- Frontend: `cd frontend && npm run dev` · `npm run gen:api`

## Layout (backend/app)
`api/routes` (HTTP only) → `services/*` (logic, no FastAPI imports) → `repositories/` (DB, always scoped by user or collection) · `workers/tasks` (Celery) · `models/` (ORM) · `schemas/` (Pydantic).

## Learning mode (mandatory)
Every task in PLAN.md is tagged:
- **[YOU]**: the owner writes it. You explain, ask guiding questions, give hints one rung at a time (the `tutor` skill), and review. **Never write the solution code unless the owner says "show me"**, and even then, explain it line by line afterwards.
- **[PAIR]**: write the scaffolding only. Leave `# TODO(you): <what and why>` where the core logic goes.
- **[AI]**: you may generate it. Briefly explain anything non-obvious.
- No tag, or the owner asks directly: ask which level they want before writing code.
- When you explain, name the concept or pattern ("this is compare-and-set") and point to one primary source.
- Prefer a question that leads the owner to the bug over pointing at the line.
- Quizzes use the `quiz-me` skill. All quiz Q&A goes into the single file `docs/learning/quiz-bank.md`, which is gitignored: never commit it, never create a second quiz file, and grep it rather than reading it whole.

## Invariants (never break these)
- Every retrieval or query path takes `allowed_collection_ids` explicitly. No default, no global. Filter in SQL, before ranking.
- Other users' resources return **404**, not 403.
- Search reads only chunks whose `version_id = documents.active_version_id`.
- Celery tasks are idempotent. State changes use compare-and-set (`UPDATE … WHERE status = :expected`).
- Retrieved document text is untrusted data in prompts (prompt-injection risk).
- Every chunk keeps `page_start`, `page_end` and `heading_path` for citations.
- Every vector row records its `embedding_model`.

## Code style
- Python: type hints everywhere, ruff and mypy clean, small pure functions, side effects at the edges.
- Interfaces (`Protocol`) for parsers, chunkers, embedders, retrievers and the LLM client. Inject them, don't import them as globals.
- Named constants, not magic numbers (e.g. `DEFAULT_CHUNK_TOKENS`).
- Tests: a real Postgres for integration tests, `FakeLLM` and `FakeEmbedder` for unit tests.

## Keeping context cheap
- Keep this file under 120 lines. Add pointers, not content.
- When a subsystem's contract changes, update its `docs/context/*.md` in the same change.
- At the end of a session, run the `checkpoint` skill. It rewrites `PROGRESS.md` (it doesn't append).
- Don't read `storage/`, `.venv/`, `node_modules/`, `eval/runs/` or lockfiles (the reads are denied in `.claude/settings.json`).
- The API shape lives in the generated `frontend/src/api/schema.d.ts` (after Phase 14). Don't restate it in prose.
