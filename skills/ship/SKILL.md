---
name: ship
description: "Autonomous shipping pipeline: reads requirements, plans stacked PRs (<250 lines each), executes each PR sequentially, validates, and pushes to GitHub. Invoke with /ship:ship <requirements>."
argument-hint: <requirements — file path, GitHub issue URL, or inline text>
model: sonnet
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*), Bash(gt:*), Bash(gh:*), Bash(cat:*), Bash(rm:*), Agent
---

# /ship — Autonomous Shipping Pipeline

Orchestrate a full development pipeline from requirements to pushed PRs. Run the entire pipeline end-to-end without asking for confirmation between phases.

Each phase is spawned as a subagent with configurable model and permission mode.

**CRITICAL**: When spawning every Agent subagent, you MUST pass the `mode` parameter explicitly. Default is `mode: "bypassPermissions"` unless overridden by `modes.<phase>` in the config. Never omit the `mode` parameter — omitting it causes the subagent to prompt the user for permissions, which blocks the pipeline.

`$ARGUMENTS` is the requirements input (file path, GitHub issue URL, or inline text).

## Phase 0: Initialize

1. Check for stale state files (`ship-state.md`, `ship-plan.md`) in the repo root. If found, ask whether to resume the previous run or start fresh. If starting fresh, delete them.

2. Load configuration from `.ship.yaml` in the repo root (if exists). Extract:
   - `maxLinesPerPR` (default: 250)
   - `maxPRs` (default: 7) — hard cap on number of PRs the planner can create
   - `branchPrefix` (default: `"ship/"`)
   - `validate` (default: auto-detect)
   - `agents.planners` — list of `{path, match}` entries (or single `agents.planner` shorthand)
   - `agents.executors` — list of `{path, match, model}` entries (or single `agents.executor` shorthand)
   - `models` — model per phase (defaults below)
   - `modes` — permission mode per phase (defaults below)

   Normalize shorthand: if `agents.planner` is a string, treat as `[{path: <value>, match: "default"}]`. Same for `agents.executor`.

3. Apply defaults for any missing config. If `.ship.yaml` doesn't exist, or any field is missing, use these defaults:

   ```yaml
   maxLinesPerPR: 250
   maxPRs: 7
   branchPrefix: "ship/"
   validate: # auto-detect
   models:
     read: sonnet
     plan: opus
     execute: sonnet
     push: sonnet
   agents:
     planners: []   # uses built-in ship-plan
     executors: []  # uses built-in ship-execute
   ```

   All phases run with `bypassPermissions` by default. To restrict a specific phase, set `modes.<phase>` (e.g., `modes.push: default` to require approval before pushing).

   Merge, don't replace: if `.ship.yaml` only sets `models.plan: sonnet`, all other models keep their defaults.

## Phase 1: Read Requirements

Spawn an Agent subagent:
- **Prompt**: Read `${CLAUDE_PLUGIN_ROOT}/skills/ship-read/SKILL.md` and follow its instructions. Input: `$ARGUMENTS`
- **model**: `models.read` from config (default: `"sonnet"`)
- **mode**: `modes.read` from config (default: `"bypassPermissions"`)

Result: `ship-state.md` exists in the repo root.

## Phase 2: Plan

Discover the planner:

1. If `agents.planners` has entries and the first matching file exists:
   → The planner prompt is: read that file and follow its instructions.
2. Otherwise:
   → The planner prompt is: read `${CLAUDE_PLUGIN_ROOT}/skills/ship-plan/SKILL.md` and follow its instructions.

Spawn an Agent subagent:
- **Prompt**: The planner prompt above, plus: "Here are the available executors: `<list from agents.executors with path, match, and model>`. Maximum PRs allowed: `<maxPRs>`." Do NOT include `ship-state.md` contents in the prompt — the planner will read it directly from the repo root.
- **model**: `models.plan` from config (default: `"opus"`)
- **mode**: `modes.plan` from config (default: `"bypassPermissions"`)

Result: `ship-plan.md` exists with detailed PR blueprints. All PRs have `Status: pending` and an `Executor:` assignment.

**Validate the plan**: Verify `ship-plan.md` has a `## Stack` section and each PR has Branch, Executor, Description, Files, and Estimated lines fields. If malformed, re-run the planner.

## Phase 3: Execute Stack

For each PR in `ship-plan.md`, in order:

### 3a. Execute PR (includes branch creation)

Update the PR's status in `ship-plan.md` to `in-progress`.

Discover the executor from the PR's `Executor:` field:

1. If `Executor:` is a path (not `"default"`) AND the file exists:
   → The executor prompt is: read that file and follow its instructions with this PR's blueprint.
2. Otherwise:
   → The executor prompt is: read `${CLAUDE_PLUGIN_ROOT}/skills/ship-execute/SKILL.md` and follow its instructions.

Resolve the model for this executor:
1. If the PR's executor has a `model` override in `agents.executors`, use that.
2. Otherwise, use `models.execute` (default: `sonnet`).

Spawn an Agent subagent:
- **Prompt**: The executor prompt above, plus: "First, create the branch by running `gt create <branch name> -m "feat: <PR title>"`. Then implement PR N: `<PR title>`. Read `ship-plan.md` from the repo root for the full blueprint and validation commands." Do NOT include the blueprint contents in the prompt — the executor will read it directly.
- **model**: resolved model above
- **mode**: `modes.execute` from config (default: `"bypassPermissions"`)

Result: Branch created, PR implemented, validated, committed. Status updated to `done` or `failed:<reason>`.

### 3b. Continue

If the PR failed, log the failure but continue to the next PR. Do not abort the stack.

Repeat 3a-3b for every PR in the plan.

## Phase 4: Push

Spawn an Agent subagent:
- **Prompt**: Read `${CLAUDE_PLUGIN_ROOT}/skills/ship-push/SKILL.md` and follow its instructions. The plan is in `ship-plan.md`.
- **model**: `models.push` from config (default: `"sonnet"`)
- **mode**: `modes.push` from config (default: `"bypassPermissions"`)

Result: Stack is pushed to GitHub. Each PR has a detailed description.

## Phase 5: Report

Output a summary (the orchestrator does this directly, no subagent needed):

```
## Ship Complete

Feature: <name from plan Summary>
PRs created: <count>
Total lines changed: <sum of estimated lines>

Stack:
1. <PR title> — <GitHub PR URL> (<lines> lines) [<status>]
2. <PR title> — <GitHub PR URL> (<lines> lines) [<status>]
...

Validation: <all passed | N failures documented>
```

Clean up: delete `ship-state.md` and `ship-plan.md` from the repo root.

## Additional Resources

For data contracts and schemas, consult:
- **`references/contracts.md`** — ship-state.md and ship-plan.md schemas, project-level agent contract
- **`references/pipeline-overview.md`** — full architecture, delegation logic, configuration reference, project-level agent authoring guide
