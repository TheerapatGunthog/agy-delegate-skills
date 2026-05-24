---
name: agy-delegate
description: "Executor-only skill: hands an approved task.md (produced by Claude Code plan mode) to the Antigravity CLI (agy) for headless execution, validates the handoff result, and cleans up transient files on success."
trigger: /agy-delegate
---

# /agy-delegate

Pick up a pre-approved `task.md` (written by Claude Code plan mode) and hand it off to the headless Antigravity CLI (`agy`) for execution. This skill is an **executor only** — project analysis and task planning happen in Claude Code plan mode before this skill is invoked.

## Usage

```
/agy-delegate                          # Delegate existing task.md to agy
/agy-delegate -t docs/task.md          # Use a custom task file
/agy-delegate --resume 5               # Re-run agy starting from step 5
/agy-delegate --timeout 30m            # Override timeout (default: 15m)
```

**Flags:**
- `-t <path>` — Use a custom task file instead of the default `task.md` at the project root
- `--resume <step>` — Restart agy from a specific step number (e.g. `--resume 5` skips steps 1–4)
- `--timeout <duration>` — Override the default agy execution timeout (default: `15m`; accepts `5m`, `30m`, `1h`, etc.)

---

## What You Must Do When Invoked

Work through every step below **in order**. Do not skip steps or combine them out of sequence.

---

### Step 0 — Pre-flight Checks

Run all checks before touching anything. Abort immediately if any check fails.

1. **Check agy is installed:**
   ```bash
   agy --version
   ```
   If this fails, abort and tell the user:
   > `agy` (Antigravity CLI) is not installed or not in PATH. Install it and ensure the current project directory is listed in `trustedWorkspaces` inside `~/.gemini/antigravity-cli/settings.json`.

2. **Check trustedWorkspaces coverage:**
   Read `~/.gemini/antigravity-cli/settings.json`. If `toolPermission` is NOT `always-proceed` AND the current directory is NOT listed in `trustedWorkspaces`, warn:
   > Warning: Current directory is not in agy's `trustedWorkspaces`. The `--dangerously-skip-permissions` flag may pause for interactive prompts. Add this path to `trustedWorkspaces` in `~/.gemini/antigravity-cli/settings.json` before proceeding.
   Ask the user whether to continue anyway or abort.

3. **Parse `--timeout <duration>` flag** — store the value as `<timeout>` (default: `15m`). Accept formats: `5m`, `30m`, `1h`, `90m`. Used in Step 2.

4. **Parse `--resume <step>` flag** — if present, store as `resume_from_step` (integer). Used in Step 2.

5. **Parse `-t <path>` flag** — if present, use that path as the task file. Otherwise default to `task.md` at the project root.

---

### Step 1 — Verify task.md Exists

1. Check that the task file (from Step 0 flag parsing) exists and is non-empty:
   ```bash
   test -s <task-file-path>
   ```

2. **If the file does not exist or is empty**, abort with:
   > No `task.md` found (or the file is empty). This skill is an executor — it does not generate plans.
   >
   > **To create a task plan:** Use Claude Code plan mode (type `/plan` or use the Plan button in the UI). Claude will analyze the project and write a `task.md` for your approval. Then re-run `/agy-delegate`.

3. **If `--resume <step>` was passed**, read the task file and count the total checklist items (lines matching `- [ ]` or `- [x]`). If `resume_from_step` exceeds `steps_total`, abort:
   > `resume_from_step` (N) exceeds `steps_total` (M) in task.md. Check the step number and try again.

4. Read the task file and display its contents to the user for confirmation before proceeding.

5. **Delete any existing `handoff.json`** so a stale result from a prior run cannot pass validation:
   ```bash
   rm -f handoff.json
   ```

---

### Step 2 — Delegate to Antigravity CLI (Headless Execution)

1. Trigger the headless Antigravity CLI from the current directory:
   ```bash
   agy --dangerously-skip-permissions --print-timeout <timeout> \
     -p "Follow the checklist in task.md exactly.
   Implement each step, commit after each item using conventional commit format (feat:/fix:/test:/refactor:/chore:/docs:), and run the verification command listed in each step before marking it done.
   Only create or modify files listed under '## Allowed Files' in task.md. If you need to touch a file not on that list, record it in handoff.json under 'scope_violations' and stop — do not commit the out-of-scope file.
   After all steps pass, run the quality command specified in task.md and confirm exit code 0.
   Then write handoff.json at the project root with fields: status ('success'/'failure'/'partial'), steps_completed, steps_total, notes, scope_violations (array). Add an 'errors' array if any step failed."
   ```

   **If `resume_from_step` is set**, append to the prompt:
   > "Steps 1 through <resume_from_step - 1> are already complete — skip them. Begin at step <resume_from_step>."

2. Wait synchronously for the command to complete.

3. **If the command exits with a non-zero exit code:**
   - Find the most recent log file in `~/.gemini/antigravity-cli/log/` and read the last 50 lines to identify the failure.
   - Report the error to the user clearly.
   - Stop here.

---

### Step 3 — Validate Handoff

1. Check that `handoff.json` exists at the project root. If it does not exist despite a zero exit code, warn:
   > `handoff.json` was not created. Treating as partial success — please inspect the output manually.
   Stop here.

2. Read `handoff.json`. Validate against the expected schema:
   ```json
   {
     "status": "success" | "failure" | "partial",
     "steps_completed": <int>,
     "steps_total": <int>,
     "notes": "<string>",
     "errors": ["<string>"],
     "scope_violations": ["<string>"]
   }
   ```
   - If `status` is not `"success"` → report `errors` and `scope_violations` to the user. Stop here.
   - If `steps_completed < steps_total` → report partial completion. Stop here.

3. On success, present to the user:
   - The `handoff.json` contents
   - A reminder to review agy's output before committing or merging any changes

4. **Cleanup on success** — delete both transient files:
   ```bash
   rm -f handoff.json
   rm -f <task-file-path>
   ```
   `handoff.json` is a completion receipt; leaving it risks a future run passing stale validation. `task.md` is deleted so a future `/agy-delegate` cannot silently re-execute the old plan — the user must go through plan mode again. If the task file was specified via `-t <path>`, delete that path, not the default `task.md`.

---

## Guardrails

- **Never skip Step 0.** Pre-flight failures are always blocking.
- **Cleanup is conditional on outcome:**
  - **Success** → delete both `task.md` (or the `-t` path) and `handoff.json` after presenting results to the user.
  - **Failure or partial** → keep both files. The user needs them to inspect the failure and use `--resume <step>` to continue.
  - The `rm -f handoff.json` at the start of Step 1 clears stale state before a run and is separate from this rule.
- **If agy times out**, report the timeout and the last known log file path. Stop here.
- **If `status` in `handoff.json` is `"failure"` or `"partial"`**, do not report success. Surface all errors to the user.
- **This skill does not manage git.** Branch creation, stashing, and merging are the user's responsibility outside this skill. If git safety is needed, create a branch before invoking `/agy-delegate`.
