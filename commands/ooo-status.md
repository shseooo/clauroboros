---
description: Show interview / seed / persona / ralph / drift status
allowed-tools: Read
---

Print the current Ouroboros state. Read-only.

1. `.ouroboros/seed.json` → if present, report `locked, ACs count, ambiguity`.
   Else → `absent`.
2. `.ouroboros/state.json#interview` → `active (turns=N, goal=...)` or `idle`.
3. `state.json#persona` + `personaTurnsLeft` → `<id> (<N> turns left)` or `—`.
4. `state.json#ralph` → `on (turn=X / cap=Y)` or `off`.
5. If a locked seed exists, tally `state.json#acGrades`:
   `<P> pass / <F> fail / <N> n-a / <U> ungraded` (U = total ACs - graded entries).
6. `state.json#drift` → if present, print `weighted=<W>` plus components and timestamp.

Format as a compact code-fenced block. No prose.
