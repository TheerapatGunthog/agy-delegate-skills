---
name: agy-delegate
description: "Automatically delegate the execution of a task checklist in task.md to the Antigravity CLI (agy) to write the code, make incremental git commits, and run verification check suites."
trigger: /agy-delegate
---

# /agy-delegate

Draft a technical implementation plan inside `task.md`, spawn a safe isolated Git branch, and automatically trigger the headless Antigravity CLI (`agy`) agent to write the code and run the verification pipeline.

## Usage

```
/agy-delegate "Implement latency budget checks in graph/pipeline.py and verify"
/agy-delegate -t docs/task.md "Add retry parameter to the orchestrator state dict"
/agy-delegate --resume auto/existing-branch
/agy-delegate --resume auto/existing-branch:5
/agy-delegate --plan "Add retry logic to orchestrator"
/agy-delegate --timeout 30m "Refactor the entire auth module"
```

**Flags:**
- `-t <path>` — Use a custom task file instead of the default `task.md`
- `--resume <branch>[:<step>]` — Re-run agy on an existing branch's task.md; optional `:<step>` restarts from that step number (e.g. `--resume auto/my-branch:5`)
- `--plan` — Write `task.md` and show it to you for approval before handing off to agy (dry-run gate)
- `--timeout <duration>` — Override the default agy execution timeout (default: `15m`; e.g. `5m`, `30m`, `1h`)

---

## What You Must Do When Invoked

Work through every step below **in order**. Do not skip steps or combine them out of sequence. All execution must be handled with safety boundaries.

---

### Step 0 — Pre-flight Checks

Run all checks before touching any files or git state. Abort immediately if any check fails.

1. **Check agy is installed:**
   ```bash
   agy --version
   ```
   If this fails, abort and tell the user:
   > `agy` (Antigravity CLI) is not installed or not in PATH. Install it and ensure the current project directory is listed in `trustedWorkspaces` inside `~/.gemini/antigravity-cli/settings.json`.

2. **Check git repository:**
   ```bash
   git rev-parse --is-inside-work-tree 2>/dev/null
   ```
   If this fails (not a git repo), offer to initialize one:
   > This directory is not a git repository. Run `git init && git add -A && git commit -m "Initial commit"` first, or run `/agy-delegate` again after initializing.
   Do not auto-initialize — let the user decide.

3. **Check trustedWorkspaces coverage:**
   Read `~/.gemini/antigravity-cli/settings.json`. If `toolPermission` is NOT `always-proceed` AND the current directory is NOT listed in `trustedWorkspaces`, warn:
   > Warning: Current directory is not in agy's `trustedWorkspaces`. The `--dangerously-skip-permissions` flag may pause for interactive prompts. Add this path to `trustedWorkspaces` in `~/.gemini/antigravity-cli/settings.json` before proceeding.
   Ask the user whether to continue anyway or abort.

4. **If `--resume <branch>[:<step>]` flag is set:**
   - Parse the branch name and optional step number from the argument (e.g. `auto/my-branch:5` → branch=`auto/my-branch`, `resume_from_step=5`).
   - Verify the branch exists: `git show-ref --verify refs/heads/<branch>`
   - Verify the task file exists on that branch.
   - If a step number was provided, count the total checklist items in task.md. If `resume_from_step > steps_total`, abort:
     > `resume_from_step` (N) exceeds `steps_total` (M) in task.md. Check the step number and try again.
   - If all checks pass, skip Steps 1 and 2 entirely and jump to Step 3.

5. **Parse `--timeout <duration>` flag** — store the value as `<timeout>` (default: `15m`). Accept formats: `5m`, `30m`, `1h`, `90m`. This value is used in Step 4.

6. **Parse `--plan` flag** — if present, set `plan_mode=true`. This activates the user-review gate at the end of Step 3 (see Step 3 item 5).

---

### Step 1 — Check Workspace & Spawn Git Branch (Safety First)

> Skip this step if `--resume` flag was passed and Step 0 verified the branch.

1. Record the name of the current branch (you will need it for the merge step):
   ```bash
   git rev-parse --abbrev-ref HEAD
   ```

2. Check for uncommitted changes:
   ```bash
   git status --porcelain
   ```
   If there are uncommitted changes, **auto-stash** them with a labeled stash:
   ```bash
   git stash push -m "agy-delegate pre-run stash"
   ```
   Note the stash ref — you must pop it after merge.

3. **Generate a safe branch name** from the task prompt using this algorithm:
   - Lowercase the prompt
   - Replace all spaces and non-alphanumeric characters with `-`
   - Collapse consecutive `-` into one
   - Truncate to 50 characters
   - Strip any trailing `-`
   - Prefix with `auto/`
   - Example: `"Add retry logic to orchestrator"` → `auto/add-retry-logic-to-orchestrator`

4. **Check for branch name collision:**
   ```bash
   git show-ref --verify refs/heads/<branch-name>
   ```
   If the branch already exists, append `-2`, then `-3`, etc. until a free name is found.

5. Create and checkout the branch:
   ```bash
   git checkout -b <branch-name>
   ```

---

### Step 2 — Codebase Exploration & Project Intelligence

> Skip this step if `--resume` flag was passed.

Before writing `task.md`, gather ground-truth knowledge about the project. The quality of the checklist depends entirely on what you learn here. Work through 2a → 2b → 2c in order.

#### 2a — Primary Sources (try in order)

1. **Check for `CLAUDE.md`** at the project root:
   - If found: read it fully (including any `@`-referenced files). Extract: project structure, language/framework, test conventions, quality command, forbidden patterns, active modules.
   - Mark findings as `[source: CLAUDE.md]`.

2. **Check for Graphify knowledge graph** — search in order:
   - `graphify-out/` directory at the project root (produced by the `/graphify` skill)
   - `graph.json` at the project root
   - `.graphify/graph.json`
   - `.graphify/` directory (any `.json` files inside)

   If `graphify-out/` exists, read the files inside it — look for `summary.md`, `entities.md`, any relationship or module files. If only a `graph.json` exists, read it directly. Extract: module relationships, key entities, architectural patterns relevant to the task. Mark findings as `[source: graphify]`.

3. **If neither `CLAUDE.md` nor a graphify graph exists**, ask the user:
   > No `CLAUDE.md` or Graphify knowledge graph found in this project. Options:
   > - **A)** Run `claude init` to generate a `CLAUDE.md` (recommended — gives agy better context about project conventions)
   > - **B)** Run `/graphify` to build a knowledge graph first (recommended for large or complex codebases)
   > - **C)** Skip — analyze the project autonomously right now
   >
   > Which would you prefer? (A / B / C)

   - If **A**: run `claude init` to generate `CLAUDE.md`, then re-read the generated file and continue to 2b.
   - If **B**: stop. Tell the user: "Run `/graphify` in a new session, then re-run `/agy-delegate` once the graph is built."
   - If **C**: proceed to 2b with no primary source — autonomous exploration will be the only input.

#### 2b — Autonomous Exploration (always run to fill gaps)

Run these checks regardless of whether 2a found a primary source. They verify accuracy and fill in anything CLAUDE.md or graphify didn't cover for the specific task at hand.

1. **Project structure overview:**
   ```bash
   find . -maxdepth 3 -not -path './.git/*' -type f \
     \( -name "*.py" -o -name "*.ts" -o -name "*.tsx" -o -name "*.go" \
        -o -name "*.rs" -o -name "*.java" -o -name "*.rb" \) | sort | head -80
   ```

2. **Detect language/framework and quality command** — check in order:
   - `Makefile` with a `check` target → `make check`
   - `package.json` → inspect for test script; use `npm test` / `yarn test` / `pnpm test`
   - `pyproject.toml` or `setup.py` → `pytest`
   - `Cargo.toml` → `cargo test`
   - `go.mod` → `go test ./...`
   - Record the exact quality command — it will be injected into the agy prompt in Step 4.

3. **Find files relevant to the task** — grep for the key terms from the user's prompt:
   ```bash
   grep -r "<key-term>" . --include="*.py" --include="*.ts" --include="*.go" -l 2>/dev/null | head -20
   ```
   Read the 2–3 most relevant files in full (not just filenames). Identify the exact functions, classes, or config keys that need to change.

4. **Check test structure** — find and read one representative test file:
   ```bash
   find . -type f \( -name "test_*.py" -o -name "*.test.ts" -o -name "*_test.go" \) | head -10
   ```
   Note: naming convention (`test_foo.py` vs `foo.test.ts`), fixture patterns, assertion style. New tests must follow the same conventions.

5. **Check recent activity** — understand what's been touched lately:
   ```bash
   git log --oneline -10
   git diff HEAD~1 --name-only
   ```

#### 2c — Exploration Summary

Before writing `task.md`, output a brief summary block to the user:

```
[Exploration Summary]
- Source:           CLAUDE.md | graphify | autonomous
- Language/stack:   <detected>
- Quality command:  <exact command>
- Relevant files:   <list with paths>
- Test convention:  <e.g. test_*.py beside source, or __tests__/ directory>
- Key constraints:  <any rules from CLAUDE.md or discovered invariants>
```

This summary is the ground truth that drives Step 3. If any field is "unknown", note it explicitly — do not guess file paths.

---

### Step 3 — Prepare the Checklist Document

> Skip this step if `--resume` flag was passed.

1. Locate the checklist file (default: `task.md` at the project root).
2. If the user specified a custom task file path using the `-t` option, use that path instead.
3. **Delete any existing `handoff.json`** before writing the new task file, so a stale result from a previous run cannot pass validation:
   ```bash
   rm -f handoff.json
   ```
4. Analyze the user's requirements and write a detailed, step-by-step technical implementation checklist. Mark all items as unchecked. Always include a final step to run the full quality suite and a step to write `handoff.json`.

   **Checklist quality requirements — every step MUST include:**
   - **Exact file path(s)** to create, edit, or delete (e.g. `src/pipeline/latency.py`, `tests/test_latency.py`)
   - **Exact change** — what to add, modify, delete, or move (function name, class, config key, line range if helpful)
   - **Why** — one sentence of rationale or constraint so agy understands intent
   - **Verification** — the concrete command or assertion that confirms the step succeeded (e.g. `python -m pytest tests/test_latency.py -k test_budget_exceeded`, `grep -q "latency_budget" src/pipeline/latency.py`)
   - **Test coverage** — every step that adds or modifies logic MUST be immediately followed by (or include) a paired test sub-step naming the exact test file path and exact test function/case to add or update. Steps that only modify config, docs, or non-logic files are exempt.

   Write as if handing off to a developer who has **never seen this codebase and cannot ask questions**. Every step must be actionable with zero external context.

   ```markdown
   # Task Checklist

   ## Context
   <2-3 sentences describing the overall goal, the affected subsystem, and any key constraints or invariants agy must not break.>
   Test coverage: every logic change must have a corresponding test. List test files explicitly.

   ## Allowed Files
   <!-- agy must only create or modify files in this list. If a change requires a file not listed here, record it in handoff.json under "scope_violations" and stop — do not commit the out-of-scope file. -->
   - <file1 — from Step 2 exploration>
   - <file2>
   - <corresponding test file(s)>

   ## Steps

   - [ ] Step 1: **<verb> `<file-path>`** — <exact change description>
     - Why: <rationale or constraint>
     - Verify: `<command that exits 0 on success>`
     - Test: Add/update `<test-function>` in `<test-file-path>`

   - [ ] Step 2: **<verb> `<file-path>`** — <exact change description>
     - Why: <rationale or constraint>
     - Verify: `<command that exits 0 on success>`
     - Test: Add/update `<test-function>` in `<test-file-path>`

   - [ ] Step N: Run the full quality suite — `make check` (or the project-appropriate command) and confirm exit code 0.
     - Why: All checks must pass before the branch is mergeable.
     - Verify: `make check` exits 0.

   - [ ] Step N+1: Write a summary JSON to `handoff.json` at the project root:
     ```json
     {
       "status": "success",
       "branch": "<branch-name>",
       "steps_completed": <N+1>,
       "steps_total": <N+1>,
       "notes": "<brief description of what was done>",
       "scope_violations": []
     }
     ```
     Set `"status": "failure"` and add an `"errors": ["<message>"]` array if any step failed. Add file paths to `"scope_violations"` if any out-of-scope files were needed.
   ```

   The `steps_total` field in `handoff.json` must equal the total number of checklist items. Instruct agy to write `"status": "failure"` and add an `"errors"` array if any step fails.

5. **If `plan_mode=true` (`--plan` flag was passed):** display the generated `task.md` contents to the user and ask:
   > `task.md` has been written. Review the checklist above. Type **proceed** to hand off to agy, or **abort** to cancel (the branch will be deleted and stash restored).

   - If the user types `proceed`: continue to Step 4.
   - If the user types `abort`: run `git checkout <original-branch>`, `git branch -D <branch-name>`, `git stash pop` (if stashed), delete `task.md`, and exit cleanly.

---

### Step 4 — Delegate to Antigravity CLI (Headless Execution)

1. Trigger the headless Antigravity CLI (`agy`) from the current directory. Use `--print-timeout <timeout>` (from the `--timeout` flag, default `15m`) to prevent infinite hangs. Inject the full project context discovered in Step 2 directly into the prompt:
   ```bash
   agy --dangerously-skip-permissions --print-timeout <timeout> \
     -p "You are working on a <language/stack> project on branch <branch-name>.
   Quality check command: <quality-command>.
   Test convention: <test-convention from Step 2b.4>.
   Key constraints: <constraints extracted from CLAUDE.md or autonomous exploration; write 'none known' if empty>.
   If a CLAUDE.md exists in this project, follow all rules defined in it.
   Follow the checklist in task.md exactly — implement each step, commit after each item using conventional commit format (feat:/fix:/test:/refactor:/chore:/docs:), and run the verification command listed in each step before marking it done.
   Only create or modify files listed under '## Allowed Files' in task.md. If you need to touch a file not on that list, record it in handoff.json under 'scope_violations' and stop — do not commit the out-of-scope file.
   After all steps pass, run <quality-command> and confirm exit code 0.
   Then write handoff.json at the project root with fields: status ('success'/'failure'/'partial'), branch, steps_completed, steps_total, notes, scope_violations (array). Add an 'errors' array if any step failed."
   ```

   **If `resume_from_step` is set** (from `--resume <branch>:<step>`), append to the prompt:
   > "Steps 1 through <resume_from_step - 1> are already complete — skip them. Begin at step <resume_from_step>."

2. Wait synchronously for the command to complete.

3. **If the command exits with a non-zero exit code:**
   - Do not attempt to merge or clean up the branch.
   - Find the most recent log file in `~/.gemini/antigravity-cli/log/` and read the last 50 lines to identify the failure.
   - Report the error to the user clearly.
   - Tell the user the branch `<branch-name>` is preserved for inspection.
   - Stop here.

---

### Step 5 — Validate Handoff & Code Review

#### 5a — Validate handoff.json

1. Check that `handoff.json` exists at the project root. If it does not exist despite a zero exit code, warn:
   > `handoff.json` was not created. Treating as partial success — will NOT auto-merge. Please inspect the branch manually.
   Stop here.

2. Read `handoff.json`. Validate against the expected schema:
   ```json
   {
     "status": "success" | "failure" | "partial",
     "branch": "<string>",
     "steps_completed": <int>,
     "steps_total": <int>,
     "notes": "<string>",
     "errors": ["<string>"]
   }
   ```
   - If `status` is not `"success"` → stop, report `errors` to user, keep branch.
   - If `steps_completed < steps_total` → stop, report partial completion, keep branch.

#### 5b — Code Review

Run the following checks. If any fail, report them to the user and do NOT proceed to merge:

0. **Secrets scan** — before reviewing anything else, run:
   ```bash
   git diff <original-branch>...HEAD -- . ':(exclude)*.lock' \
     | grep -iE '(password|secret|api_key|apikey|token|private_key|-----BEGIN|AWS_|GITHUB_TOKEN)' \
     | head -20
   ```
   If any matches are found, **stop immediately**:
   > Potential secrets detected in the diff. Do NOT merge until these lines are reviewed and removed or rotated. Affected lines shown above.
   Keep the branch and do not proceed further.

1. **Quality suite passes** — discover and run the project's quality check command:
   - If `Makefile` exists and has a `check` target → `make check`
   - Else if `package.json` exists → `npm test` (or `yarn test` / `pnpm test` based on which lock file is present)
   - Else if `pyproject.toml` or `setup.py` exists → `pytest`
   - Else if `Cargo.toml` exists → `cargo test`
   - Else if `go.mod` exists → `go test ./...`
   - Else ask the user what command to run before proceeding.
   Confirm exit code 0.

2. **New env vars declared** — if any `.env` or config file changed, verify `.env.example` (or equivalent) was updated.

3. **Project-specific invariants** — read `CLAUDE.md` (and any files it references with `@`) in the current project root. Extract any coding rules, forbidden patterns, or required conventions stated there. Verify the diff does not violate them. If no `CLAUDE.md` exists, skip this sub-check.

4. **Lock-file consistency** — if a dependency manifest changed (`pyproject.toml`, `package.json`, `Cargo.toml`, `go.mod`, `Gemfile`, etc.), verify its lock file (`uv.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Cargo.lock`, `go.sum`, `Gemfile.lock`, etc.) was also updated and committed.

5. **Scope guard** — read the `## Allowed Files` list from `task.md`. Get the list of files changed on this branch:
   ```bash
   git diff <original-branch>...HEAD --name-only
   ```
   If any changed file is not in the `## Allowed Files` list AND is not in `handoff.json`'s `scope_violations` array, report it as a blocking issue:
   > Out-of-scope file modified: `<path>`. This was not in the allowed file list. Review the change before merging.

6. **Commit message format** — verify every commit on this branch uses a conventional commit prefix:
   ```bash
   git log <original-branch>..HEAD --format="%s" \
     | grep -vE '^(feat|fix|test|refactor|chore|docs|style|perf|ci|build)(\(.+\))?!?: '
   ```
   If any commit messages do not match, list them and warn the user (non-blocking — user may squash before merging):
   > The following commits do not follow conventional commit format: `<list>`. Consider squashing or amending before merge.

7. **Diff sanity** — run `git diff <original-branch>...HEAD` and display the full diff to the user before asking for merge approval.

#### 4c — Summary Report

Present to the user:
- All commits made (output of `git log <original-branch>..HEAD --oneline`)
- The full diff (`git diff <original-branch>...HEAD`)
- The `handoff.json` contents
- Results of each code review check above (pass/fail)

---

### Step 6 — Merge & Cleanup

Only proceed here after the user explicitly confirms they want to merge.

1. Switch back to the original branch:
   ```bash
   git checkout <original-branch>
   ```

2. Merge with a no-fast-forward merge commit:
   ```bash
   git merge --no-ff auto/<task-name> -m "feat: merge agy-delegate/<task-name>"
   ```
   If merge fails due to conflicts, abort the merge (`git merge --abort`), switch back to the feature branch, and ask the user to resolve conflicts manually.

3. Delete the feature branch after a successful merge:
   ```bash
   git branch -d auto/<task-name>
   ```

4. If changes were stashed in Step 1, restore them:
   ```bash
   git stash pop
   ```
   If the pop causes conflicts, report them to the user — do not silently drop the stash.

5. Confirm success:
   > Branch `auto/<task-name>` merged into `<original-branch>` and deleted. All done.

6. **Optional PR draft** — ask the user:
   > Would you like to create a draft pull request? (y/N)

   If yes, check whether `gh` is installed:
   ```bash
   gh --version 2>/dev/null
   ```
   If `gh` is not installed, note: "`gh` CLI not found — skipping PR draft." and stop.

   If installed, run:
   ```bash
   gh pr create --draft \
     --title "<conventional-commit-style title derived from task.md Context>" \
     --body "$(cat <<'EOF'
   ## Summary
   <bullet points from task.md Context block and handoff.json notes field>

   ## Changes
   <output of: git log --oneline for the merged commits>

   ## Test plan
   - [ ] <each Verify command from task.md steps>

   🤖 Generated by agy-delegate
   EOF
   )"
   ```
   Return the PR URL to the user.

---

## Guardrails

These rules are absolute — no exceptions:

- **Never merge if `make check` has a non-zero exit code.** A passing test suite is a hard gate.
- **Never delete the `auto/*` branch before the user confirms the merge.** The branch is the rollback path.
- **Never `git stash drop`.** Only `git stash pop`. Lost stash = lost work.
- **Never auto-merge.** Always surface the full git diff and wait for explicit user approval.
- **If `status` in `handoff.json` is `"failure"` or `"partial"`, keep the branch.** Do not clean up — the user may need to inspect or retry.
- **If agy times out** (exit code from timeout), the branch stays. Report the timeout and the last known log file path.
- **Never skip Step 0.** Pre-flight failures are always blocking — do not proceed with best-effort workarounds.
