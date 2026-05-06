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
| Interview  | `/interview <goal>` slash command instructs the agent to ask Socratic questions one at a time.        |
| Evaluate   | `/evaluate` runs the test suite (auto-detected) and grades each AC by inspecting the codebase.        |
| Drift      | `/drift` prompts a self-assessment scored as `0.5*goal + 0.3*constraint + 0.2*ontology`.              |
| Unstuck    | `/unstuck [persona]` activates one of 5 lateral-thinking personas (recorded in state.json).            |
| Ralph      | `/ralph [N]` runs an inline self-driven evaluate-and-fix loop with a hard cap.                        |
| Harness    | Karpathy guidelines bundled in `skills/clauroboros/SKILL.md`; auto-loads on coding-related triggers.    |

## Slash commands

| Command                   | Effect                                                                  |
| ------------------------- | ----------------------------------------------------------------------- |
| `/interview <goal>`       | Start a Socratic interview to crystallize a seed.                       |
| `/seed`                   | Print the locked seed (or current draft).                              |
| `/evaluate`               | Run mechanical tests + grade each AC with file-level evidence.         |
| `/drift`                  | Self-assess drift vs the locked seed.                                  |
| `/unstuck [id]`           | Switch to a lateral persona (5 options).                               |
| `/ralph [on\|off\|N]`     | Run an inline evaluate-and-fix loop with a hard cap.                   |
| `/status`                 | Show interview / seed / persona / ralph / drift state.                 |
| `/reset`                  | Clear session state (the locked seed is preserved).                    |

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
│   ├── interview.md
│   ├── seed.md
│   ├── evaluate.md
│   ├── drift.md
│   ├── unstuck.md
│   ├── ralph.md
│   ├── status.md
│   └── reset.md
└── skills/
    └── clauroboros/
        └── SKILL.md
```

## Example workflow

```
1. /interview "build a todo CLI"
   → agent runs a Socratic interview, one question at a time
   → each answer updates seedDraft in .ouroboros/state.json
   → locks seed.json + seed.yaml once ambiguity ≤ 0.2 and AC ≥ 5

2. (write code)

3. /evaluate
   → auto-detects + runs npm test / pytest / make test / cargo test / go test
   → inspects the codebase per AC and records verdict (pass/fail/n-a) with evidence
   → prints pass / fail tally

4. /drift
   → self-assesses goal / constraint / ontology divergence
   → weighted ≤ 0.30 is OK, otherwise DRIFTED

5. (when stuck) /unstuck adversary
   → activates a persona that tries to break the current solution

6. /ralph 8
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
