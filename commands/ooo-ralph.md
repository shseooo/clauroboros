---
description: Run a self-driven evaluate-and-continue loop until ACs converge or cap is hit
argument-hint: [on|off|<cap-N>]
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

Run the Ouroboros Ralph loop: keep evaluating and fixing until every
acceptance criterion passes or a hard turn cap is reached.

## Argument handling

- `off` → set `state.json#ralph = null`. Report and stop.
- `<integer>` → start with that cap.
- empty → start with cap = 8 (default).

## Pre-conditions

- `.ouroboros/seed.json` must exist with `locked: true`. If not, stop and
  tell the user.

## Loop

Update `state.json#ralph = { active: true, turn: 0, cap: <N> }`.

Then **execute the following loop yourself** within this single command run.
Claude Code has no native per-turn hook, so Ralph runs as one extended
command rather than across user turns:

```
while state.json#ralph.turn < cap:
  state.json#ralph.turn += 1, persist
  
  Run the equivalent of /ooo-evaluate (mechanical + per-AC grading).
  
  if all ACs pass:
    write state.json#ralph = null, persist
    output the literal token: CONVERGED
    break
  
  Identify the smallest delta that would flip the most failing ACs to pass.
  Apply that change surgically (Karpathy harness §3).
  Continue.

if cap hit without converging:
  write state.json#ralph = null, persist
  report: "Ralph hit cap=<N> without converging. <K> ACs still failing."
  list each failing AC with current evidence
  do NOT continue silently
```

## Hard rules
- Never disable or weaken acceptance criteria to make the loop "converge."
  If an AC is wrong, STOP and tell the user — they should re-run
  `/ooo-interview` to revise the seed.
- Apply the Karpathy harness on every iteration: surgical changes, no
  speculative refactors, no scope creep.
- Each iteration must produce a verifiable result before continuing.
