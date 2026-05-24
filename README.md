# agy-delegate

> A [Claude Code](https://claude.ai/code) skill that delegates an approved `task.md` checklist to the headless [Antigravity CLI](https://antigravity.google/product/antigravity-cli) (`agy`) for execution.

![Claude Code skill](https://img.shields.io/badge/Claude%20Code-skill-blueviolet)
![requires agy](https://img.shields.io/badge/requires-agy-orange)

---

## What it does

This skill is an **executor only**. Before invoking it, use Claude Code plan mode to write and approve a `task.md`. Then run `/agy-delegate` and the skill will:

1. Verify `task.md` exists and show it to you for final confirmation
2. Hand off to `agy` with the checklist as its prompt
3. Validate the `handoff.json` result written by agy
4. Clean up transient files on success, or preserve them for inspection on failure

Git branch management, codebase exploration, and code review are handled outside this skill — by you and Claude Code plan mode.

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| [Claude Code](https://claude.ai/code) | The CLI that runs skills |
| [`agy`](https://antigravity.google/product/antigravity-cli) (Antigravity CLI) | Must be in `$PATH` |
| `agy` trusted workspace | Add your project path to `trustedWorkspaces` in `~/.gemini/antigravity-cli/settings.json` |

---

## Installation

```bash
mkdir -p ~/.claude/skills/agy-delegate
cp SKILL.md ~/.claude/skills/agy-delegate/SKILL.md
```

Restart Claude Code (or run `/reload` if supported). The `/agy-delegate` command will be available in any project.

---

## Workflow

```
1. Plan mode (Claude Code)
   └─ Analyze codebase → write task.md → user approves

2. /agy-delegate
   └─ Verify task.md → show to user → delete stale handoff.json

3. agy (headless)
   └─ Execute each step → commit after each → write handoff.json

4. Validate
   └─ Check handoff.json status → report results
   └─ Success: delete task.md + handoff.json
   └─ Failure: keep both for inspection + --resume
```

---

## Usage

```bash
# Delegate the current task.md to agy
/agy-delegate

# Use a custom task file
/agy-delegate -t docs/task.md

# Resume from step 5 (e.g. step 4 failed, you fixed it manually)
/agy-delegate --resume 5

# Give agy more time for large tasks (default: 15m)
/agy-delegate --timeout 30m
```

---

## Flags

| Flag | Description | Default |
|------|-------------|---------|
| `-t <path>` | Use a custom task file instead of `task.md` | `task.md` |
| `--resume <step>` | Skip steps 1…N-1 and restart from step N | — |
| `--timeout <duration>` | Override the agy execution timeout. Accepts `5m`, `30m`, `1h`, etc. | `15m` |

---

## Expected `task.md` format

The skill expects a checklist structured like this (produced by Claude Code plan mode):

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

- [ ] Step N: Run the full quality suite — `make check` — confirm exit code 0.
  - Why: all checks must pass before the branch is mergeable.
  - Verify: `make check` exits 0.

- [ ] Step N+1: Write `handoff.json` at the project root.
```

---

## `handoff.json` contract

`agy` writes this file at the project root when it finishes. The skill validates it:

```json
{
  "status": "success",
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

A `status` of anything other than `"success"` stops cleanup and keeps both `task.md` and `handoff.json` on disk for inspection.

---

## Guardrails

- **Step 0 is never skipped** — pre-flight failures always block; no best-effort workarounds
- **`task.md` and `handoff.json` are preserved on failure** — use `--resume <step>` to continue after fixing the issue
- **Stale `handoff.json` is cleared at the start of every run** — a leftover receipt from a previous run cannot cause a false pass
- **This skill does not manage git** — branch creation, stashing, and merging are your responsibility

---

## License

MIT — see [LICENSE](LICENSE) or replace with your preferred license.
