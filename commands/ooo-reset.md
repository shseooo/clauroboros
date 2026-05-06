---
description: Clear Ouroboros session state (preserves the locked seed)
allowed-tools: Read, Write, Edit
---

Reset Ouroboros session state. **Does NOT delete the locked seed.**

1. Confirm with the user: "Clear interview / persona / ralph / acGrades /
   drift? `seed.yaml` is preserved."

2. On yes, overwrite `.ouroboros/state.json` with the empty schema:
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

3. Leave `.ouroboros/seed.json`, `.ouroboros/seed.yaml`, and
   `.ouroboros/interview.jsonl` untouched.

4. Report what was cleared.
