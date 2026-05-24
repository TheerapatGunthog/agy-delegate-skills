# agy-delegate

A Claude Code skill that delegates an approved `task.md` checklist to the headless [Antigravity CLI](https://github.com/google-gemini/gemini-cli) (`agy`) for execution, then validates the result and cleans up transient files.

**This skill is an executor only.** Project analysis and implementation planning happen in Claude Code plan mode — not inside this skill.

---

## Workflow

```
1. User: run Claude Code plan mode
         → Claude analyzes the project, writes task.md, user approves

2. User: /agy-delegate
         → skill verifies task.md exists
         → delegates to agy (headless execution)
         → validates handoff.json
         → on success: deletes task.md + handoff.json
         → on failure: keeps both for inspection / --resume
```

---

## Prerequisites

- **`agy` installed** and in `PATH` (`agy --version` must succeed)
- Current directory listed in `trustedWorkspaces` inside `~/.gemini/antigravity-cli/settings.json`, or `toolPermission` set to `always-proceed`
- A `task.md` already written and approved via Claude Code plan mode

---

## Usage

```
/agy-delegate                    # Delegate existing task.md to agy
/agy-delegate -t docs/task.md    # Use a custom task file
/agy-delegate --resume 5         # Re-run agy starting from step 5
/agy-delegate --timeout 30m      # Override execution timeout (default: 15m)
```

### Flags

| Flag | Description |
|---|---|
| `-t <path>` | Use a custom task file instead of `task.md` at the project root |
| `--resume <step>` | Restart agy from a specific step number (skips earlier steps) |
| `--timeout <duration>` | Override the agy timeout (e.g. `5m`, `30m`, `1h`; default: `15m`) |

---

## Cleanup Contract

| Outcome | `task.md` | `handoff.json` |
|---|---|---|
| Success | deleted | deleted |
| Failure / partial | kept | kept |

On success both files are deleted: `handoff.json` is transient metadata and leaving it risks stale validation on the next run; `task.md` is deleted so a future `/agy-delegate` cannot silently re-execute the old plan — the user must go back through plan mode.

On failure or partial completion both files are kept so the user can inspect what happened and use `--resume <step>` to continue from a specific step.

---

## task.md Format

The skill expects a checklist produced by Claude Code plan mode. Minimum required structure:

```markdown
# Task Checklist

## Context
<goal, affected files, constraints>

## Allowed Files
- src/foo.py
- tests/test_foo.py

## Steps

- [ ] Step 1: ...
  - Why: ...
  - Verify: `<command>`

- [ ] Step N: Write handoff.json
  ...
```

agy commits after each step using [conventional commit](https://www.conventionalcommits.org/) format and runs each step's `Verify` command before marking it done.

---

## handoff.json Schema

agy writes this file at the end of a run. The skill reads and validates it:

```json
{
  "status": "success | failure | partial",
  "steps_completed": 4,
  "steps_total": 4,
  "notes": "brief summary",
  "errors": [],
  "scope_violations": []
}
```
