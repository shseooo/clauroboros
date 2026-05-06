---
description: Start an Ouroboros Socratic interview for the given goal
argument-hint: <goal>
allowed-tools: Read, Write, Edit, Bash
---

You are starting an Ouroboros interview. The goal is: **$1**

Apply the Karpathy coding harness from the `ouroboros` skill (think before
coding · simplicity · surgical · goal-driven). Do NOT write production code
during the interview.

## Steps

1. Ensure `.ouroboros/` exists. If `.ouroboros/state.json` is missing, create
   it with the empty schema documented in the `ouroboros` skill. If
   `.ouroboros/seed.json` already exists and `locked: true`, ask the user
   whether to start a NEW draft (the existing seed.yaml is preserved either
   way). Stop if they say no.

2. Initialize the interview in `state.json`:
   - `seedDraft = { goal: "$1", acceptance_criteria: [], constraints: [], ontology: {}, assumptions_exposed: [], ambiguity: 1.0 }`
   - `interview = { active: true, turns: 0, goal: "$1" }`

3. Append a `start` event to `.ouroboros/interview.jsonl`:
   `{"kind":"start","at":"<ISO>","goal":"$1"}`

4. Begin the Socratic interview. **Ask ONE question at a time.** Each
   question must target the most load-bearing hidden assumption you can
   identify. Cover (across multiple turns):
   - Ontology — the precise meaning of every domain term
   - Acceptance criteria — observable, verifiable conditions of done
   - Hard constraints — non-negotiables (perf, deps, runtime, security)
   - Anti-cases — what this is NOT for
   - Edge cases — boundary inputs that could break the design
   - Success metrics — how we'll measure goodness post-ship
   - Out-of-scope — explicit non-goals

5. After each user answer:
   - Read `state.json`, merge new learnings into `seedDraft`:
     - Add ACs to `acceptance_criteria`
     - Add constraints, ontology terms, exposed assumptions
     - Update `ambiguity` (0 = perfectly specified, 1 = total fog)
   - Increment `interview.turns`
   - Append a `seed_set` event to `interview.jsonl`
   - Write state.json back

6. **Finalize** when both `ambiguity ≤ 0.2` AND `acceptance_criteria.length ≥ 5`:
   - Build the final seed object with `created_at` (current ISO), `locked: true`
   - Write `.ouroboros/seed.json` (canonical)
   - Write `.ouroboros/seed.yaml` — mirror with double-quoted scalars,
     keys in this order: `created_at, locked, ambiguity, goal,
     acceptance_criteria, constraints, ontology, assumptions_exposed`
   - Clear `state.json#interview` and `state.json#seedDraft`
   - Append a `finalize` event to `interview.jsonl`
   - Report the locked seed summary to the user

If at any point you discover the goal itself is incoherent, STOP and report
to the user — do not paper over with more questions.
