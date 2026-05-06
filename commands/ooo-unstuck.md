---
description: Switch to a lateral-thinking persona for the next 3 turns
argument-hint: [inverter|first-principles|naive-newcomer|adversary|architect]
allowed-tools: Read, Write, Edit
---

Activate a lateral-thinking persona to break out of a stuck state.

## If `$1` is empty

Ask the user to pick one of:
- `inverter` — Assume the OPPOSITE of every requirement. What changes? What
  requirement turns out to be load-bearing vs ornamental?
- `first-principles` — Strip to physical/mathematical/logical fundamentals.
  Forget the existing approach entirely; reconstruct from axioms.
- `naive-newcomer` — Pretend you've never seen this codebase. Explain it from
  zero, asking dumb questions out loud. Surface every implicit assumption.
- `adversary` — Actively try to BREAK the current solution. Edge cases,
  races, malformed inputs, security holes.
- `architect` — Zoom out. Forget current module boundaries — redraw them.
  What if the seams were elsewhere?

## If `$1` is a valid persona id

1. Update `state.json`:
   ```json
   { "persona": "$1", "personaTurnsLeft": 3 }
   ```

2. Apply the chosen persona's lens to the most recent task or problem the
   user mentioned. Generate fresh insights, contradictions, or attack
   vectors. Be concrete; cite files and lines.

3. Note: `personaTurnsLeft` is informational — Claude Code has no native
   per-turn hook, so the user must remember to keep applying the lens (or
   re-run `/ooo-unstuck`). The `ouroboros` skill instructs the assistant to
   re-read `state.json` at session start to pick the persona back up.
