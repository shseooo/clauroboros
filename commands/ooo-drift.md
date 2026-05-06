---
description: Self-assess drift (goal/constraint/ontology) vs the locked seed
allowed-tools: Read, Write, Edit
---

Estimate Ouroboros drift vs the locked seed and record it.

## Pre-conditions
- `.ouroboros/seed.json` must exist with `locked: true`. If not, drift is
  undefined — tell the user and stop.

## Steps

1. Read `seed.json` and inspect the current state of the project (what has
   been built so far).

2. Score three components, each in `[0, 1]` (0 = perfectly aligned,
   1 = totally drifted):
   - **goal** (weight 0.5) — has the work strayed from the goal?
   - **constraint** (weight 0.3) — are any locked constraints violated?
   - **ontology** (weight 0.2) — are any defined terms used inconsistently?

3. Compute `weighted = 0.5 * goal + 0.3 * constraint + 0.2 * ontology`.

4. Record in `state.json#drift`:
   ```json
   { "goal": <number>, "constraint": <number>, "ontology": <number>,
     "weighted": <number>, "notes": "<short explanation>", "at": "<ISO>" }
   ```

5. Verdict: `OK` if `weighted ≤ 0.30`, else `DRIFTED`.

6. Report the components, weighted score, and verdict. Be honest;
   over-report rather than under-report drift.
