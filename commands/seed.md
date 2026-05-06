---
description: Show the current Ouroboros seed (or draft, if interviewing)
allowed-tools: Read
---

Display the current Ouroboros specification.

1. If `.ouroboros/seed.json` exists, read it and pretty-print:
   - Goal
   - Acceptance criteria (numbered)
   - Constraints
   - Ontology (term → definition table)
   - Assumptions exposed
   - Ambiguity score
   - `created_at`, `locked`

2. Otherwise, read `.ouroboros/state.json`:
   - If `seedDraft` is non-null, print "Draft (NOT locked)" with the same
     fields as above plus the running ambiguity hint.
   - Otherwise, tell the user no seed exists and to run `/interview <goal>`.

Do not modify any files.
