# agy-delegate

> A [Claude Code](https://claude.ai/code) skill that turns a plain-English task description into a fully executed, reviewed, and mergeable git branch — delegated to the headless [Antigravity CLI](https://github.com/google-gemini/aistudio-agy) (`agy`).

![Claude Code skill](https://img.shields.io/badge/Claude%20Code-skill-blueviolet)
![requires agy](https://img.shields.io/badge/requires-agy-orange)
![requires git](https://img.shields.io/badge/requires-git-lightgrey)

---

## What it does

Type `/agy-delegate "Add retry logic to the orchestrator"` and the skill will:

1. Explore your codebase (reads `CLAUDE.md`, `graphify-out/`, or discovers autonomously)
2. Write a precise `task.md` checklist with exact file paths, rationale, verify commands, and paired test steps
3. Spawn an isolated `auto/<task>` git branch
4. Hand off to `agy` with your full project context injected into the prompt
5. Validate the result (`handoff.json`), scan for secrets, check scope, run your quality suite, verify commit format
6. Show you the full diff and wait for your explicit merge approval
7. Optionally create a `gh pr create --draft` after merge

Nothing merges automatically. Every gate requires your sign-off.

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| [Claude Code](https://claude.ai/code) | The CLI that runs skills |
| [`agy`](https://github.com/google-gemini/aistudio-agy) (Antigravity CLI) | Must be in `$PATH` |
| `agy` trusted workspace | Add your project path to `trustedWorkspaces` in `~/.gemini/antigravity-cli/settings.json` |
| Git repository | `git init` if needed — the skill will not auto-initialize |
| [`gh`](https://cli.github.com/) CLI | Optional — only used for the PR draft feature |

---

## Installation

```bash
mkdir -p ~/.claude/skills/agy-delegate
cp SKILL.md ~/.claude/skills/agy-delegate/SKILL.md
```

Restart Claude Code (or run `/reload` if supported). The `/agy-delegate` command will be available in any project.

---

## Usage

```bash
# Basic — describe what you want built
/agy-delegate "Implement latency budget checks in graph/pipeline.py and verify"

# --plan: write task.md first, review it, then decide whether to run agy
/agy-delegate --plan "Add retry logic to orchestrator"

# --timeout: give agy more time for large tasks (default: 15m)
/agy-delegate --timeout 30m "Refactor the entire auth module"

# -t: use a custom task file instead of the default task.md
/agy-delegate -t docs/task.md "Add retry parameter to the orchestrator state dict"

# --resume: re-run agy on a branch that failed or was interrupted
/agy-delegate --resume auto/add-retry-logic-to-orchestrator

# --resume with step: restart from a specific step (e.g. step 5 failed, fix it, resume)
/agy-delegate --resume auto/add-retry-logic-to-orchestrator:5
```

---

## Flags

| Flag | Description | Default |
|------|-------------|---------|
| `-t <path>` | Use a custom task file instead of `task.md` | `task.md` |
| `--plan` | Dry-run gate: generates `task.md`, shows it to you, waits for `proceed` or `abort` before delegating | off |
| `--resume <branch>[:<step>]` | Skip branch creation and task.md writing; re-run agy on the given branch. Add `:<N>` to restart from step N. | — |
| `--timeout <duration>` | Override the agy execution timeout. Accepts `5m`, `30m`, `1h`, etc. | `15m` |

---

## How it works

```
Step 0   Pre-flight
         └─ agy installed? git repo? trusted workspace? parse flags.

Step 1   Spawn branch
         └─ auto-stash uncommitted changes → create auto/<task-name> branch

Step 2   Explore codebase
         └─ CLAUDE.md → graphify-out/ → .graphify/ → autonomous discovery
         └─ outputs [Exploration Summary] before writing the plan

Step 3   Write task.md
         └─ exact file paths · rationale · verify commands · paired test steps
         └─ ## Allowed Files scope list populated from exploration
         └─ if --plan: show to user and wait for "proceed" / "abort"

Step 4   Delegate to agy
         └─ injects: language/stack, branch, quality cmd, test convention, constraints
         └─ if --resume N: tells agy to skip steps 1…N-1

Step 5   Validate & review
         └─ 0. secrets scan (blocks merge on any hit)
         └─ 1. quality suite (make check / pytest / cargo test / …)
         └─ 2. env var declarations
         └─ 3. CLAUDE.md invariants
         └─ 4. lock-file consistency
         └─ 5. scope guard (diff vs Allowed Files list)
         └─ 6. commit message format (conventional commits)
         └─ 7. full diff shown to user

Step 6   Merge & cleanup
         └─ git merge --no-ff on explicit user approval
         └─ pop stash · delete feature branch
         └─ optional: gh pr create --draft
```

---

## Generated `task.md` format

The skill writes a checklist like this into `task.md` before delegating:

```markdown
# Task Checklist

## Context
<2-3 sentences: goal, affected subsystem, invariants agy must not break.>
Test coverage: every logic change must have a corresponding test. List test files explicitly.

## Allowed Files
<!-- agy must only create or modify files in this list -->
- src/pipeline/latency.py
- tests/test_latency.py

## Steps

- [ ] Step 1: **Add `check_latency_budget` to `src/pipeline/latency.py`** — implement the budget check function
  - Why: the orchestrator needs to enforce per-node latency limits
  - Verify: `grep -q "check_latency_budget" src/pipeline/latency.py`
  - Test: Add `test_budget_exceeded` in `tests/test_latency.py`

- [ ] Step 2: **Call `check_latency_budget` in `src/pipeline/runner.py`** — invoke on each node result
  - Why: the check is only useful if called in the execution path
  - Verify: `python -m pytest tests/test_latency.py -k test_budget_exceeded`
  - Test: Add `test_runner_enforces_budget` in `tests/test_latency.py`

- [ ] Step N: Run the full quality suite — `make check` — confirm exit code 0.
  - Why: all checks must pass before the branch is mergeable.
  - Verify: `make check` exits 0.

- [ ] Step N+1: Write `handoff.json` at the project root.
```

---

## `handoff.json` contract

`agy` writes this file at the project root when it finishes. The skill validates it before offering a merge:

```json
{
  "status": "success",
  "branch": "auto/add-latency-budget-checks",
  "steps_completed": 4,
  "steps_total": 4,
  "notes": "Implemented check_latency_budget and wired it into the runner.",
  "scope_violations": [],
  "errors": []
}
```

| Field | Values |
|-------|--------|
| `status` | `"success"` / `"failure"` / `"partial"` |
| `scope_violations` | Files agy needed to touch that were outside `## Allowed Files` |
| `errors` | Non-empty when `status` is `"failure"` or `"partial"` |

A `status` of anything other than `"success"` blocks the merge and keeps the branch intact for inspection.

---

## Safety guardrails

- **Never auto-merge** — always surfaces the full `git diff` and waits for explicit user approval
- **Secrets block merge** — any credential-looking string in the diff stops the workflow immediately
- **Scope violations surfaced** — files changed outside `## Allowed Files` are flagged before merge
- **Never `git stash drop`** — only `git stash pop`; lost stash = lost work
- **Branch preserved on failure** — `auto/*` branches are never deleted without your confirmation
- **Timeout preserves branch** — if agy times out, the branch stays and the last log path is reported
- **Step 0 is never skipped** — pre-flight failures always block; no best-effort workarounds

---

## Better results with project context

The quality of the generated `task.md` improves significantly when the skill can read prior knowledge about your codebase. In order of preference:

1. **`CLAUDE.md`** — run `claude init` in your project root to generate one. The skill reads it fully, including any `@`-referenced files.
2. **`graphify-out/`** — run `/graphify` (the graphify Claude Code skill) to build a knowledge graph. The skill reads `summary.md`, entity files, and relationship files from this directory.
3. **Autonomous** — if neither exists, the skill explores the codebase itself, but file paths and function names in the checklist will be less precise.

---

## License

MIT — see [LICENSE](LICENSE) or replace with your preferred license.
