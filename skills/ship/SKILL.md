---
name: ship
description: "Autonomous shipping pipeline: reads requirements, plans stacked PRs (<250 lines each), executes each PR sequentially, validates, and pushes to GitHub. Invoke with /ship:ship <requirements>."
argument-hint: <requirements — file path, GitHub issue URL, or inline text>
---

# /ship — Autonomous Shipping Pipeline

Orchestrate a full development pipeline from requirements to pushed PRs. Run the entire pipeline end-to-end without asking for confirmation between phases.

`$ARGUMENTS` is the requirements input (file path, GitHub issue URL, or inline text).

## Phase 0: Initialize

1. Check for stale state files (`ship-state.md`, `ship-plan.md`) in the repo root. If found, ask whether to resume the previous run or start fresh. If starting fresh, delete them.

2. Load configuration from `.ship.yaml` in the repo root (if exists). Extract:
   - `maxLinesPerPR` (default: 250)
   - `branchPrefix` (default: `"ship/"`)
   - `validate` (default: auto-detect)
   - `agents.planners` — list of `{path, match}` entries (or single `agents.planner` shorthand)
   - `agents.executors` — list of `{path, match}` entries (or single `agents.executor` shorthand)

   Normalize shorthand: if `agents.planner` is a string, treat as `[{path: <value>, match: "default"}]`. Same for `agents.executor`.

## Phase 1: Read Requirements

Read `${CLAUDE_PLUGIN_ROOT}/skills/ship-read/SKILL.md` and follow its instructions with `$ARGUMENTS` as input.

Result: `ship-state.md` exists in the repo root.

## Phase 2: Plan

Discover the planner:

1. If `agents.planners` has entries and the first matching file exists:
   → Read that file and follow its instructions.
2. Otherwise:
   → Read `${CLAUDE_PLUGIN_ROOT}/skills/ship-plan/SKILL.md` and follow its instructions.

**Pass to the planner:**
- `ship-state.md` (requirements + repo context)
- The list of available executors from `agents.executors` (each with `path` and `match` description). The planner uses this to assign an executor per PR.

The planner reads `ship-state.md` and writes `ship-plan.md`, including an `Executor:` field on each PR.

Result: `ship-plan.md` exists with detailed PR blueprints. All PRs have `Status: pending` and an `Executor:` assignment.

**Validate the plan**: Verify `ship-plan.md` has a `## Stack` section and each PR has Branch, Executor, Description, Files, and Estimated lines fields. If malformed, re-run the planner.

## Phase 3: Execute Stack

For each PR in `ship-plan.md`, in order:

### 3a. Create Branch

Read `${CLAUDE_PLUGIN_ROOT}/skills/ship-stack/SKILL.md` and follow its instructions to create the graphite branch for this PR.

### 3b. Execute PR

Update the PR's status in `ship-plan.md` to `in-progress`.

Discover the executor from the PR's `Executor:` field:

1. If `Executor:` is a path (not `"default"`) AND the file exists:
   → Read that file and follow its instructions with this PR's blueprint.
2. Otherwise:
   → Read `${CLAUDE_PLUGIN_ROOT}/skills/ship-execute/SKILL.md` and follow its instructions.

Pass the executor:
- The full PR section from `ship-plan.md` (everything under `### PR N: <title>`)
- The Configuration section (validation commands)

Result: PR is implemented, validated, committed. Status updated to `done` or `failed:<reason>`.

### 3c. Continue

If the PR failed, log the failure but continue to the next PR. Do not abort the stack.

Repeat 3a-3c for every PR in the plan.

## Phase 4: Push

Read `${CLAUDE_PLUGIN_ROOT}/skills/ship-push/SKILL.md` and follow its instructions.

Result: Stack is pushed to GitHub. Each PR has a detailed description.

## Phase 5: Report

Output a summary:

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
