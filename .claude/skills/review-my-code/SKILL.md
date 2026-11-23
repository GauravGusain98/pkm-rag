---
name: review-my-code
description: Use when the learner asks for a review of code they wrote in the PKM-RAG project ("/review-my-code", "review this", "is this right?"). Mentor-style review that explains why and teaches patterns; it doesn't rewrite.
---

# Mentor code review

## Scope
- Default: the uncommitted diff (`git diff` plus `git diff --staged`). If they name a file or function, review only that.
- Read the relevant `docs/context/*.md` and the current phase in `docs/PROGRESS.md` to know the intended contract.

## Check, in this priority order
1. **Invariants from AGENTS.md:** access scoping (`allowed_collection_ids`), 404 vs. 403, active-version filtering, idempotency and CAS, citation metadata, `embedding_model`. Any violation is a blocker.
2. **Correctness:** races, error handling (transient vs. permanent), edge cases (empty file, a 0-page PDF, a huge chunk), transaction boundaries.
3. **Design:** layer violations (FastAPI imported in `services/`, SQL in routes), missing `Protocol`/injection, magic numbers, functions doing two jobs.
4. **Tests:** is the core behavior tested? Is there a test that would have caught the bugs you found?
5. **Readability:** names, size, comments that explain *why*.

## Output format
For each finding (at most 7, most important first):
- **[blocker | should | consider]** `file:line` and a one-line summary
- **Why it matters:** the failure scenario in concrete terms.
- **Question for you:** a question that leads them to the fix (e.g. "what happens if this task runs twice?"). Don't write the fixed code for [YOU] code.
- **Concept:** the name of the pattern or concept, plus where to read about it.

Then:
- **What you did well** (1–3 specific things, so they repeat them).
- **One concept to study next**, based on the most important finding.

If they reply with a fix, re-review only that finding.
