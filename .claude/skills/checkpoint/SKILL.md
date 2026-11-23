---
name: checkpoint
description: Use at the end of a work session or when a phase task finishes in the PKM-RAG project ("/checkpoint", "wrap up", "save progress", "I'm done for today"). Updates PROGRESS, the journal, context files and ADRs so the next session starts cheaply.
---

# Checkpoint

Goal: the next session (yours or an agent's) gets oriented from `AGENTS.md` plus `PROGRESS.md` in under 2k tokens.

## Steps
1. Gather what happened: `git status`, `git diff --stat` and `git log --oneline -10`, plus this conversation. Don't re-read whole files.
2. **Rewrite** `docs/PROGRESS.md` (replace it, don't append; keep it under 60 lines):
   - Current phase, status, today's date.
   - Done in this phase (a checklist with ✓).
   - Next up: the 1–3 concrete next actions, specific enough to start cold ("write the CAS claim in `workers/tasks/ingest.py:claim_job`").
   - Open questions and blockers.
   - Concepts still shaky (carry them over and add new ones).
3. **Mini-quiz:** run the `quiz-me` skill in **checkpoint** mode, with 2–3 questions about the current phase or today's work, one at a time. Each question and answer is appended to `docs/learning/quiz-bank.md` (the single gitignored quiz file).
4. Append a journal entry to `docs/learning/journal.md` using its template. Record the learner's answers *in their words*, and note any "needed full solution" moments.
5. If a subsystem's contract, invariants or gotchas changed, update `docs/context/<subsystem>.md` (keep it under 60 lines).
6. If a decision was made (a library, schema shape, scope, trade-off), draft an ADR from `docs/adr/0000-template.md` and ask the learner to write the "What I learned" section.
7. If the real architecture changed, update `docs/ARCHITECTURE.md` (current state only).
8. If the phase's "Done when" criteria are all met, mark the phase complete in PROGRESS only **after** the learner passes a deep quiz on it (`/quiz-me phase N`, scoring at least 70% overall with at least one correct L3 answer). Suggest a `/quiz-me` review quiz if 2 or more phases have finished since the last one.
9. Suggest a commit message. Don't commit unless asked.

## Don't
- Let PROGRESS.md grow into a log. History belongs in the journal and git.
