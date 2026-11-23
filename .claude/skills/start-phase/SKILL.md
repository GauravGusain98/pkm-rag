---
name: start-phase
description: Use when the user starts a new phase of the PKM-RAG build plan ("/start-phase 4", "let's begin phase 4", "what's next"). Gives a concept primer, reading list, warm-up questions and a task breakdown for one phase, without loading the whole plan.
---

# Start a phase

Goal: orient the learner in a phase cheaply, and make them think before they code.

## Steps
1. Read `docs/PROGRESS.md`. If no phase number was given, use "Current phase". If the previous phase's "Done when" items aren't all done, say so and ask whether to finish them first.
2. Read **only** that phase's section of `docs/PLAN.md`: grep for `^## Phase N:` and read until the next `^## `. Don't read other phases.
3. Read `docs/context/*.md` only for subsystems this phase depends on (at most 2).
4. Respond with exactly these sections, kept tight:
   - **Why this phase matters** (2–3 sentences tied to the project's goal).
   - **Concept primer:** for each concept in the phase, 2–4 sentences of plain explanation plus one concrete example from *this* project. Don't explain how to implement the [YOU] tasks.
   - **Read first:** the 1–3 primary sources from the phase or the PLAN reading list, with what to look for in each.
   - **Warm-up questions:** 3 questions the learner should try to answer before coding. Don't give the answers. Offer to discuss them.
   - **Task breakdown:** the phase's tasks as a checklist, with help-level tags ([YOU], [PAIR], [AI]) and a suggested order.
   - **First step:** the one smallest concrete action to take now.
5. Update `docs/PROGRESS.md`: set the current phase and status `in progress`, and copy the task checklist into "Next up".
6. If this phase names a context file to create, create it from the template in `docs/context/README.md` with empty sections. The learner fills it in as they build.

## Don't
- Write any [YOU] code.
- Paste the plan section back verbatim. Summarize and add value.
