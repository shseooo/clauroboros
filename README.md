# clauroboros

[English](./README.md) | [한국어](./README.ko.md) | [日本語](./README.ja.md)

Native [Ouroboros](https://github.com/Q00/ouroboros) loop as a Claude Code
plugin.

This is **not** a wrapper around the `ouroboros` Python CLI. The spec-first
loop (interview → seed → evaluate → drift → unstuck → ralph) is implemented
as Claude Code slash commands and a skill. State lives in `.ouroboros/` in
your project; the [Karpathy coding harness](https://github.com/forrestchang/andrej-karpathy-skills/blob/main/CLAUDE.md)
is bundled in the skill and triggers on every coding task.

## Install

From a git remote:
```sh
claude plugin marketplace add shseooo/clauroboros
claude plugin install clauroboros@clauroboros-cc
```

From a local checkout (development):
```sh
claude plugin marketplace add ./clauroboros
claude plugin install clauroboros@clauroboros-cc
```

## What it ports

| Concept    | Claude Code mechanism                                                                                  |
| ---------- | ------------------------------------------------------------------------------------------------------ |
| Seed       | `.ouroboros/seed.json` (canonical) + `seed.yaml` (mirror); locked once finalized.                     |
| Interview  | `/ooo-interview <goal>` slash command instructs the agent to ask Socratic questions one at a time.        |
| Evaluate   | `/ooo-evaluate` runs the test suite (auto-detected) and grades each AC by inspecting the codebase.        |
| Drift      | `/ooo-drift` prompts a self-assessment scored as `0.5*goal + 0.3*constraint + 0.2*ontology`.              |
| Unstuck    | `/ooo-unstuck [persona]` activates one of 5 lateral-thinking personas (recorded in state.json).            |
| Ralph      | `/ooo-ralph [N]` runs an inline self-driven evaluate-and-fix loop with a hard cap.                        |
| Harness    | Karpathy guidelines bundled in `skills/clauroboros/SKILL.md`; auto-loads on coding-related triggers.    |

## Glossary

- **Seed** — Immutable spec: goal + acceptance criteria + constraints + ontology
  + exposed assumptions. Once locked, every command treats it as authority.
  Stored as `seed.json` (canonical) + `seed.yaml` (mirror).
- **Acceptance Criterion (AC)** — A single verifiable statement that defines
  part of "done". Each AC is graded `pass` / `fail` / `n/a` by either
  mechanical tests or by inspecting the codebase. A seed needs ≥ 5 ACs
  before it can lock.
- **Ambiguity score** — 0–1 measure of how fuzzy the spec still is during
  the interview. The seed locks only when ambiguity ≤ 0.2.
- **Constraint** — Non-functional bound the solution must respect
  (performance, security, compatibility, dependency policy, etc.).
- **Ontology** — Term → definition map that pins down project-specific
  vocabulary, so every command and AC refers to the same concept.
- **Persona** — One of 5 lateral-thinking lenses for breaking out of stuck
  states: `inverter`, `first-principles`, `naive-newcomer`, `adversary`,
  `architect`. Recorded in `state.json` so the next session can pick up.
- **Drift** — Divergence between current work and the locked seed, weighted
  as `0.5*goal + 0.3*constraint + 0.2*ontology`. ≤ 0.30 is OK; over = DRIFTED.
- **Ralph** — Self-driven evaluate-and-fix loop with a hard turn cap.
  Continues until every AC passes (CONVERGED) or the cap is hit.
- **Hard cap** — Maximum number of `/ooo-ralph` iterations before a forced stop
  (default 8). Prevents runaway loops; if ACs haven't converged at the cap,
  ralph stops and reports honestly instead of weakening ACs to fake success.
- **Scope creep** — Gradual divergence from the locked seed: extra features,
  broadened goal, redefined terms, or constraints quietly relaxed. Detected
  via `/ooo-drift`. The seed is a boundary, not a suggestion — if the goal truly
  changed, stop and re-run `/ooo-interview` rather than expanding silently.
- **Karpathy harness** — Coding behavioral guidelines (think first, surgical
  changes, define success criteria, expose assumptions) bundled into the
  skill and applied on every coding task.

## Slash commands

| Command                   | Effect                                                                  |
| ------------------------- | ----------------------------------------------------------------------- |
| `/ooo-interview <goal>`       | Start a Socratic interview to crystallize a seed.                       |
| `/ooo-seed`                   | Print the locked seed (or current draft).                              |
| `/ooo-evaluate`               | Run mechanical tests + grade each AC with file-level evidence.         |
| `/ooo-drift`                  | Self-assess drift vs the locked seed.                                  |
| `/ooo-unstuck [id]`           | Switch to a lateral persona (5 options).                               |
| `/ooo-ralph [on\|off\|N]`     | Run an inline evaluate-and-fix loop with a hard cap.                   |
| `/ooo-status`                 | Show interview / seed / persona / ralph / drift state.                 |
| `/ooo-reset`                  | Clear session state (the locked seed is preserved).                    |

## When to use each command

| Situation                                                  | Command                  |
| ---------------------------------------------------------- | ------------------------ |
| Starting a new feature/task with a fuzzy goal              | `/ooo-interview <goal>`      |
| Want to inspect the current spec (locked or draft)         | `/ooo-seed`                  |
| Just finished a meaningful work step — verify against ACs  | `/ooo-evaluate`              |
| Suspect scope creep after several edits                    | `/ooo-drift`                 |
| Stuck, looping, or generating low-quality solutions        | `/ooo-unstuck [persona]`     |
| Want autonomous push-to-green within a hard cap            | `/ooo-ralph [N]`             |
| Forgot where you are in the loop                           | `/ooo-status`                |
| Want to clear in-progress state but keep the seed          | `/ooo-reset`                 |

### Picking a persona for `/ooo-unstuck`

| You are stuck because…                       | Persona            |
| -------------------------------------------- | ------------------ |
| The design feels overcomplicated             | `inverter`         |
| You've patched on top of a bad foundation    | `first-principles` |
| The codebase is foreign and assumptions hide | `naive-newcomer`   |
| You can't tell if the solution is correct    | `adversary`        |
| Module boundaries feel wrong                 | `architect`        |

### Loop-level guidance

- Run `/ooo-interview` **before writing any code** for a non-trivial task. The
  cost of crystallizing a seed up front is far less than rewriting later.
- Run `/ooo-evaluate` as the gate before claiming "done" — never self-declare
  done without a graded AC tally.
- Run `/ooo-drift` periodically (every few edits, or after a refactor) — it's
  cheap and catches scope creep early.
- Use `/ooo-ralph` when the remaining work is mechanical (failing ACs with clear
  fixes), **not** when ACs are wrong. If an AC is wrong, stop and re-run
  `/ooo-interview` to revise the seed; never weaken ACs to converge.

## Notes on Claude Code's harness limits

Claude Code does not expose per-turn system-prompt injection or cross-turn
auto follow-up messages. So:
- **Karpathy harness** lives in a SKILL whose description triggers on coding
  keywords, so it auto-loads on relevant tasks. It is also referenced from
  every command body to ensure it stays applied.
- **Persona persistence** is by file (state.json) only. The skill instructs
  the agent to re-read state.json at session start. There is no native turn
  hook to auto-decrement.
- **Ralph** runs as a single extended command rather than across multiple
  user turns. The agent self-loops inside the command up to the cap.

## State files

```
.ouroboros/
├── seed.json        # canonical seed (read by every command as authority)
├── seed.yaml        # human / Ouroboros-compatible mirror
├── state.json       # interview / persona / ralph / acGrades / drift
└── interview.jsonl  # append-only event log
```

## Files in this plugin

```
clauroboros/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── commands/
│   ├── ooo-interview.md
│   ├── ooo-seed.md
│   ├── ooo-evaluate.md
│   ├── ooo-drift.md
│   ├── ooo-unstuck.md
│   ├── ooo-ralph.md
│   ├── ooo-status.md
│   └── ooo-reset.md
└── skills/
    └── clauroboros/
        └── SKILL.md
```

## Example workflow

```
1. /ooo-interview "build a todo CLI"
   → agent runs a Socratic interview, one question at a time
   → each answer updates seedDraft in .ouroboros/state.json
   → locks seed.json + seed.yaml once ambiguity ≤ 0.2 and AC ≥ 5

2. (write code)

3. /ooo-evaluate
   → auto-detects + runs npm test / pytest / make test / cargo test / go test
   → inspects the codebase per AC and records verdict (pass/fail/n-a) with evidence
   → prints pass / fail tally

4. /ooo-drift
   → self-assesses goal / constraint / ontology divergence
   → weighted ≤ 0.30 is OK, otherwise DRIFTED

5. (when stuck) /ooo-unstuck adversary
   → activates a persona that tries to break the current solution

6. /ooo-ralph 8
   → self-loops evaluate → fix-failing-AC within cap=8
   → prints CONVERGED and stops automatically once all pass
```

## Notes

- Zero runtime dependencies; the agent uses Claude Code's built-in
  `Read` / `Write` / `Edit` / `Bash` / `Grep` / `Glob` tools to manage
  `.ouroboros/` state.
- Seed YAML is hand-written by the agent following the schema in `SKILL.md`.
  Always double-quoted scalars, fixed key order. Reads come from the JSON
  sidecar so quoting bugs don't break the loop.
- Ralph hard-caps at the user's chosen N (default 8). It will never disable
  ACs to "converge" — it stops and reports if it can't pass them honestly.
