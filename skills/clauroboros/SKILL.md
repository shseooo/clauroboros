---
name: clauroboros
description: Spec-first coding workflow. Apply when working on any non-trivial coding task — building, fixing, refactoring, evaluating, or debugging. Enforces the Karpathy coding harness (think before coding, simplicity, surgical changes, goal-driven execution) and the Ouroboros loop (interview → seed → evaluate → drift → unstuck → ralph). Read .ouroboros/ state files at the start of every coding session if they exist.
---

# Ouroboros — coding harness + spec-first loop

## Coding harness (apply on EVERY coding task)

Source: https://github.com/forrestchang/andrej-karpathy-skills/blob/main/CLAUDE.md

### 1. Think Before Coding
**Don't assume. Don't hide confusion. Surface tradeoffs.**
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity First
**Minimum code that solves the problem. Nothing speculative.**
- No features beyond what was asked.
- No abstractions for single-use code.
- No flexibility/configurability that wasn't requested.
- No error handling for impossible scenarios.
- If you wrote 200 lines and it could be 50, rewrite it.

### 3. Surgical Changes
**Touch only what you must. Clean up only your own mess.**
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- Mention unrelated dead code — don't delete it.
- Remove orphans your changes created; don't remove pre-existing dead code.
- Test: every changed line traces directly to the user's request.

### 4. Goal-Driven Execution
**Define success criteria. Loop until verified.**
- "Add validation" → "Write tests for invalid inputs, then make them pass."
- "Fix the bug" → "Write a test that reproduces it, then make it pass."
- "Refactor X" → "Ensure tests pass before and after."
- For multi-step tasks, state a brief plan with verify steps.

---

## Ouroboros loop (when an Ouroboros session is active)

The Ouroboros workflow turns vague goals into verified codebases through a
specification-first loop. Each phase has its own slash command.

### State files (always under `.ouroboros/` in the project root)

```
.ouroboros/
├── seed.json        # canonical immutable spec (read this as authority)
├── seed.yaml        # human / Ouroboros-compatible mirror of seed.json
├── state.json       # mutable session state — see schema below
└── interview.jsonl  # append-only transcript of seed_set / finalize events
```

**state.json schema:**
```json
{
  "seedDraft": null,
  "interview": null,
  "persona": null,
  "personaTurnsLeft": 0,
  "ralph": null,
  "acGrades": [],
  "drift": null
}
```

When any field is active:
- `seedDraft`: `{ goal, acceptance_criteria[], constraints[], ontology{}, assumptions_exposed[], ambiguity }`
- `interview`: `{ active, turns, goal }`
- `persona`: one of `inverter | first-principles | naive-newcomer | adversary | architect`
- `ralph`: `{ active, turn, cap }`
- `acGrades`: `[ { index, criterion, verdict: pass|fail|n/a, evidence, at } ]`
- `drift`: `{ goal, constraint, ontology, weighted, notes, at }`

**seed.json schema (locked once finalized):**
```json
{
  "goal": "string",
  "acceptance_criteria": ["string"],
  "constraints": ["string"],
  "ontology": { "term": "definition" },
  "assumptions_exposed": ["string"],
  "ambiguity": 0.0,
  "created_at": "ISO timestamp",
  "locked": true
}
```

### Phases

| Slash command       | Purpose                                                                          |
| ------------------- | -------------------------------------------------------------------------------- |
| `/ooo-interview`        | Socratic Q&A that exposes hidden assumptions; produces a draft seed              |
| `/ooo-seed`             | Show the current seed (or draft, if interviewing)                                |
| `/ooo-evaluate`         | Mechanical (test runner) + semantic (grade each AC) verification gate            |
| `/ooo-drift`            | Self-assess goal/constraint/ontology drift vs the locked seed                    |
| `/ooo-unstuck`          | Switch to a lateral-thinking persona for the next 3 turns                        |
| `/ooo-ralph`            | Self-driven evaluate-and-continue loop until ACs converge or a hard turn cap     |
| `/ooo-status`           | Print interview / seed / persona / ralph / drift state                           |
| `/ooo-reset`            | Clear session state (preserves the locked seed)                                  |

### The five lateral personas (used by `/ooo-unstuck`)

| ID                | Lens                                                                          |
| ----------------- | ----------------------------------------------------------------------------- |
| `inverter`        | Assume the OPPOSITE of every requirement — what's load-bearing vs ornamental? |
| `first-principles`| Strip to physical/mathematical/logical fundamentals; reconstruct from axioms  |
| `naive-newcomer`  | Pretend you've never seen this codebase — surface every implicit assumption   |
| `adversary`       | Actively try to BREAK the current solution — edge cases, races, holes         |
| `architect`       | Zoom out — redraw module boundaries; what if seams were elsewhere?            |

### Mechanical evaluator detection (used by `/ooo-evaluate`)

Detect (in order) and run:
1. `package.json` with `scripts.test` → `npm test --silent`
2. `pyproject.toml` or `pytest.ini` → `pytest -q`
3. `Makefile` with a `test:` target → `make test`
4. `Cargo.toml` → `cargo test --quiet`
5. `go.mod` → `go test ./...`
6. None of the above → mark mechanical stage as `skipped`

### Drift weighting

`weighted = 0.5 * goal + 0.3 * constraint + 0.2 * ontology`. Threshold for OK
is `weighted ≤ 0.30`. Be honest; over-report rather than under-report.

### YAML emission rules for seed.yaml

The file is a *mirror* — `seed.json` is canonical. Always double-quote string
scalars in `seed.yaml` to avoid quoting bugs. Top-level keys appear in this
order: `created_at`, `locked`, `ambiguity`, `goal`, `acceptance_criteria`,
`constraints`, `ontology`, `assumptions_exposed`.

---

## Coordination rules

- At session start, if `.ouroboros/state.json` exists, READ it before answering. Honor any active persona, interview, or ralph cap.
- During an interview, never write production code. Ask one question per turn.
- During ralph, after each work step run `/ooo-evaluate`. Stop and emit the literal token `CONVERGED` when all ACs pass. Stop on cap.
- The seed is **immutable** once locked. If you discover the seed is wrong, STOP and report — do not silently re-scope. Either revise via a new interview or accept the drift.
