# Subsystem context files

One file per subsystem (`ingestion.md`, `workers.md`, `parsing.md`, `chunking.md`, `embeddings.md`, `retrieval.md`, `eval.md`, `rag.md`, `chat.md`, `study.md`, `frontend.md`, `auth.md`, `backend-core.md`).

Agents read these **instead of** the source code, to save tokens. Keep each under 60 lines, using this template:

```
# <Subsystem>
## Purpose (1–2 lines)
## Entry points (functions, tasks, endpoints: file:symbol)
## Contract (inputs → outputs, the key types)
## Invariants (must always be true)
## Gotchas (things that bit you)
## Related ADRs
```

Create a file when its phase starts, and update it in the same change as any contract change.
