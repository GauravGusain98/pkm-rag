# Progress

<!-- Rewritten (not appended) by the `checkpoint` skill. Keep under 60 lines. -->

**Current phase:** 0, Foundations and AI context setup
**Status:** in progress
**Last session:** 2026-11-20 (phase kickoff)

## Machine notes
- Apple M1, **8 GB RAM**: use a 3B LLM (`llama3.2:3b` / `qwen2.5:3b`), not 8B. Give the Docker VM about 2–3 GB.
- Python: system 3.9 is too old, so use `uv python install 3.12`. Installed: uv 0.7, Docker 28 (start Docker Desktop), Ollama 0.40 (server up, no models), Node 22.

## Done in this phase
- ✓ git repo initialized, AI context files and skills scaffolded
- ✓ docker-compose: postgres (healthy), redis-queue (AOF + noeviction), redis-cache (allkeys-lru, no persistence), all with healthchecks. Decision: option C, two Redis instances (separate failure domains). Reasoning comments done. ADR-0005 written (your 'What I learned' section is still open)

## Next up (Phase 0 tasks)
1. [x] [YOU] Review AGENTS.md: cut what you don't believe, add your own rules (task 6)
2. [x] [PAIR] docker-compose (task 2).
3. [ ] [AI] Ollama: pull a 3B model and verify `/api/tags` (task 3)
4. [ ] [PAIR] backend/ with uv on Python 3.12: ruff, mypy, pytest, pytest-asyncio, pre-commit (task 4)
5. [ ] [AI] Makefile: up, down, api, worker, test, lint, migrate, eval (task 7)
6. [ ] [YOU] ADRs 0001–0004 (task 5): 0001 ✓, 0002 ✓ (2026-11-23), 0003 Celery vs BackgroundTasks next, then 0004 content-addressed storage
7. [ ] [YOU] `GET /health` checking db, redis and ollama (Done-when)
8. [ ] First commit, then `/quiz-me phase 0`

## Open questions / blockers
- (none)

## Concepts I'm still shaky on
- AOF mechanics: append every write command, replay on restart, fsync policy (quiz ADR-0005, Q3)
- LRU vs. LFU, and cache pollution from scans (Q11)
- Fail open for optional dependencies (the cache); sync CPU work inside `async def` blocks the event loop (Q12, which comes back in Phase 1)
- Dual writes across Postgres and Redis: commit-then-enqueue, the outbox, idempotent reaper (Q13, which comes back in Phase 4)
- Defending decisions on cost and failure isolation, not on future scale (Q14)
- Docker networking: OK at L2, not yet tested at L3 (Q8/Q9)
