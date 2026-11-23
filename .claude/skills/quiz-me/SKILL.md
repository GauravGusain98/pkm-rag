---
name: quiz-me
description: Use when the learner wants to test or deepen their understanding in the PKM-RAG project ("/quiz-me", "quiz me", "quiz me on phase 4", "quiz me on pgvector", "quiz me on ADR-0007", "review time"). Runs a leveled quiz from beginner to professional on a phase, concept or design decision, and records every question and answer in the single quiz bank file.
---

# Quiz me

The goal is real understanding of the concepts and the *decisions* behind this project, from "what is it" up to "defend it in a design review".

## The quiz bank (one file, forever)
- Path: `docs/learning/quiz-bank.md`. It's **gitignored, so never commit it** and never `git add -f` it.
- Exactly one file holds all quizzes. If it's missing, create it with the header below. Otherwise only append. Never create another quiz file, and never rewrite or delete past entries.
- The file grows large, so **don't read it whole.** Use grep (e.g. `grep -n "Verdict: wrong\|Verdict: partial" docs/learning/quiz-bank.md`, or grep by `concept:` or `Topic:`).
- **Write each question to the file as soon as it's answered**, not at the end, so an interrupted quiz isn't lost.

Header, written on creation only:
```
# Quiz bank
<!-- Single file for all quizzes. Gitignored. Appended by the quiz-me and checkpoint skills. Grep it, don't read it whole. -->
```

## Modes
Pick from the request. If it's unclear, ask with one short question.

| Mode | Trigger | Size |
|---|---|---|
| **deep** (default when a topic is named) | "quiz me on phase 4", "on pgvector", "on ADR-0007" | 15–25 questions covering the whole concept map, across all 4 levels. Can span sessions. |
| **review** (default with no topic) | "/quiz-me", "review time" | 8 questions of spaced repetition across finished phases |
| **checkpoint** | called by the `checkpoint` skill | 2–3 questions about the current phase |

## Levels (every deep quiz climbs all four)
1. **L1 Beginner (what and why):** definitions, what problem it solves, what breaks without it. *"What is an embedding, and why can't we just use keyword search?"*
2. **L2 Intermediate (how and use):** mechanics, APIs, configuration, applying it in this codebase. *"Write the SQL for a k-NN query in pgvector with cosine distance."*
3. **L3 Advanced (edge cases and failure modes):** races, limits, gotchas, debugging, predicting the outcome. *"Your filtered HNSW query returns 3 of 10 rows. Why? Fix it two ways."*
4. **L4 Professional (design, scale, trade-offs):** production incidents, scaling, alternatives, defending a decision. *"At 50M chunks, would you keep pgvector? What would you measure first, and what would make you switch?"*

## Deep quiz procedure
1. **Build a concept map for the topic:**
   - Phase topic: that phase's "Concepts to learn", tasks and checkpoint questions in `docs/PLAN.md` (grep `## Phase N:` and read only that section), plus its `docs/context/*.md` and related ADRs.
   - Concept topic: everything in PLAN.md and the context files touching it, plus its prerequisites.
   - Decision topic (ADR): the context, the options rejected, the consequences, and the concepts the decision depends on.
   - Show the map as a short list (8–15 concepts) and ask whether to add or remove anything.
2. **Check prior coverage:** grep the quiz bank for this `Topic:` and these `concept:` values. Skip concepts already answered correctly at L3 or L4. Prioritize past `wrong` and `partial` answers.
3. **Ask one question at a time** and wait for the answer. Start at L1 and move up a level once the learner answers a concept correctly. If they get one wrong, ask one more question at the same level on that concept before moving on. Aim to cover every concept on the map at L1–L2 and the central ones at L3–L4.
4. **Mix the question types:** explain-why, predict-the-outcome, spot-the-bug (a short snippet in this project's style), write-a-snippet (SQL, Python, a config line), compare-A-vs-B, defend-the-decision (from the ADRs), and design-it (L4).
5. **After each answer:**
   - Verdict: `correct` / `partial` / `wrong` / `skipped` ("I don't know" is fine and is recorded as `skipped`).
   - Feedback in 1–4 sentences: what was right, what was missing, and the key insight.
   - A model answer: the complete answer a senior engineer would give.
   - A source: a primary doc or section, an ADR, or `file:line` in this repo.
   - Append the entry to the quiz bank (format below).
6. **To pause or resume:** if the learner stops, append `_Paused after QN; remaining concepts: …_`. On the next deep quiz on the same topic, grep for that line and resume from there.
7. **Finish:** append the summary block. Update the "Concepts I'm still shaky on" list in `docs/PROGRESS.md`: remove anything answered correctly at L3 or above, and add the misses.

## Review quiz procedure
Build a pool from the quiz bank's `wrong`, `partial` and `skipped` entries (re-ask them in new wording, never verbatim), the PROGRESS "shaky" list, the journal's "needed full solution" notes, and the checkpoint questions of finished phases. Weight it 50% past misses, 30% the latest phase and 20% older phases. Use mixed levels. Record entries the same way.

## Entry format (append exactly this)
```
## Quiz YYYY-MM-DD · Topic: <phase N: name | concept | ADR-NNNN> · Mode: <deep|review|checkpoint>

### Q1 · L2 Intermediate · concept: <short-kebab-name> · type: <predict|explain|bug|snippet|compare|defend|design>
**Question:** …
**My answer:** <the learner's answer, verbatim or faithfully condensed>
**Verdict:** correct | partial | wrong | skipped
**Feedback:** …
**Model answer:** …
**Source:** …

### Summary
- Score: X / N (L1 a/b · L2 a/b · L3 a/b · L4 a/b)
- Strong: …
- Revisit: <concepts> with <sources>
- Next quiz suggestion: …
```

## Don't
- Give the answer before the learner has tried, or hint inside the question.
- Ask trivia (version numbers, exact flag spellings) unless it matters in practice.
- Ask more than one question per message.
