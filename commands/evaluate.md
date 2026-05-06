---
description: Run the Ouroboros 3-stage evaluation gate (mechanical + semantic)
allowed-tools: Read, Write, Edit, Bash
---

Run a full Ouroboros evaluation pass.

## Pre-conditions
- `.ouroboros/seed.json` must exist with `locked: true`. If not, tell the
  user to run `/interview <goal>` first and stop.

## Stage 1 — Mechanical

Detect (in order, first match wins) and run:
1. `package.json` with `scripts.test` → `npm test --silent`
2. `pyproject.toml` or `pytest.ini` → `pytest -q`
3. `Makefile` with a `test:` target → `make test`
4. `Cargo.toml` → `cargo test --quiet`
5. `go.mod` → `go test ./...`
6. None of the above → stage is `skipped`

Capture exit code and the last 50 lines of combined stdout/stderr. Report.

## Stage 2 — Semantic (per-AC grading)

For each acceptance criterion in `seed.json#acceptance_criteria` (1-indexed):
1. Inspect the codebase using `Read` / `Grep` / `Glob` to find evidence.
2. Render a verdict: `pass` | `fail` | `n/a`.
3. Append/update an entry in `state.json#acGrades`:
   `{ index, criterion, verdict, evidence: "<one line: file path, test name, or reason>", at: "<ISO>" }`
   - Replace any prior grade for the same `index` (don't duplicate).

When done, write `state.json` back.

## Report

Output a tally: `N pass / M fail / K n-a / U ungraded`. List failing ACs
explicitly with their evidence. Apply the Karpathy harness — be terse, no
fluff, no premature conclusions. If mechanical stage failed, surface the test
output up front; do NOT downgrade real test failures to "n/a".
