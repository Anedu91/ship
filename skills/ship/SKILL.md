---
name: ship
description: "Autonomous shipping pipeline: reads requirements, plans stacked PRs (<250 lines each), executes each PR sequentially, validates, and pushes to GitHub. Invoke with /ship:ship <requirements>."
argument-hint: <requirements — file path, GitHub issue URL, or inline text>
model: sonnet
allowed-tools: Read, Glob, Grep, Agent
---

# /ship — Autonomous Shipping Pipeline

Orchestrate a full development pipeline from requirements to pushed PRs. Run the entire pipeline end-to-end without asking for confirmation between phases.

The orchestrator does NOT execute any bash commands. It only reads files, spawns subagents, and reports results.

**CRITICAL**: When spawning every Agent subagent, you MUST pass the `mode` parameter explicitly. Default is `mode: "bypassPermissions"` unless overridden by `modes.<phase>` in the config. Never omit the `mode` parameter — omitting it causes the subagent to prompt the user for permissions, which blocks the pipeline.

`$ARGUMENTS` is the requirements input (file path, GitHub issue URL, or inline text).

## Phase 0: Initialize

1. Check for stale `ship-plan.md` or `ship-state.md` in the repo root. If found, ask whether to resume the previous run or start fresh. If starting fresh, tell the first subagent to delete them.

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

3. Apply defaults for any missing config:

   ```yaml
   maxLinesPerPR: 250
   maxPRs: 7
   branchPrefix: "ship/"
   validate: # auto-detect
   models:
     plan: opus
     execute: sonnet
     push: sonnet
   agents:
     planners: []   # uses built-in ship-plan
     executors: []  # uses built-in ship-execute
   ```

   All phases run with `bypassPermissions` by default. To restrict a specific phase, set `modes.<phase>` (e.g., `modes.push: default` to require approval before pushing).

   Merge, don't replace: if `.ship.yaml` only sets `models.plan: sonnet`, all other models keep their defaults.

## Phase 1: Plan

The planner reads requirements, scans the repo, and produces the full PR blueprint in a single step.

Discover the planner:

1. If `agents.planners` has entries and the first matching file exists:
   → The planner prompt is: read that file and follow its instructions.
2. Otherwise:
   → The planner prompt is: read `${CLAUDE_PLUGIN_ROOT}/skills/ship-plan/SKILL.md` and follow its instructions.

Spawn an Agent subagent:
- **Prompt**: The planner prompt above, plus: "Requirements input: `$ARGUMENTS`. Here are the available executors: `<list from agents.executors with path, match, and model>`. Maximum PRs allowed: `<maxPRs>`. Max lines per PR: `<maxLinesPerPR>`. Branch prefix: `<branchPrefix>`."
- **model**: `models.plan` from config (default: `"opus"`)
- **mode**: `modes.plan` from config (default: `"bypassPermissions"`)

Result: The planner returns the full plan as structured text in its response. The orchestrator captures this output.

**Parse the plan**: Extract each PR section from the planner's output. Each PR should have: title, branch name, executor assignment, description, file operations, estimated lines, and validation commands. Store these in memory for Phase 2.

**Validate the plan**: Verify the output has a `## Stack` section and each PR has Branch, Executor, Description, Files, and Estimated lines fields. If malformed, re-run the planner.

## Phase 2: Execute Stack

For each PR extracted from the plan, in order:

### 2a. Execute PR

Discover the executor from the PR's `Executor:` field:

1. If `Executor:` is a path (not `"default"`) AND the file exists:
   → The executor prompt is: read that file and follow its instructions with this PR's blueprint.
2. Otherwise:
   → The executor prompt is: read `${CLAUDE_PLUGIN_ROOT}/skills/ship-execute/SKILL.md` and follow its instructions.

Resolve the model for this executor:
1. If the PR's executor has a `model` override in `agents.executors`, use that.
2. Otherwise, use `models.execute` (default: `sonnet`).

Spawn an Agent subagent:
- **Prompt**: The executor prompt above, plus the **full PR blueprint inline**:
  ```
  Implement this PR:

  Branch: <branch name>
  Title: <PR title>
  Description: <description>
  Files:
    - <file operations from the plan>
  Validation commands:
    - <command 1>
    - <command 2>
  ```
  Do NOT reference ship-plan.md — the executor receives everything it needs in this prompt.
- **model**: resolved model above
- **mode**: `modes.execute` from config (default: `"bypassPermissions"`)

Result: Branch created, PR implemented, validated, committed. The executor returns success or failure with details.

### 2b. Record & Continue

Record the result (success/failure + details) for the final report. If the PR failed, log the failure but continue to the next PR. Do not abort the stack.

Repeat 2a-2b for every PR in the plan.

## Phase 3: Push

Delegate pushing to the ship-push subagent.

Spawn an Agent subagent:
- **Prompt**: Read `${CLAUDE_PLUGIN_ROOT}/skills/ship-push/SKILL.md` and follow its instructions. Then provide inline:
  ```
  Feature summary: <from plan Summary>
  Total PRs: <count>

  PR 1: <title>
  - Branch: <branch name>
  - Description: <description>
  - Files: <file operations summary>
  - Estimated lines: <number>
  - Validation: <pass or fail with reason>

  PR 2: <title>
  ...
  ```
- **model**: `models.push` from config (default: `"sonnet"`)
- **mode**: `modes.push` from config (default: `"bypassPermissions"`)

Result: The push agent returns the list of PRs with their GitHub URLs and status.

## Phase 4: Report

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

## Additional Resources

For data contracts and the PR blueprint schema, consult:
- **`references/contracts.md`** — PR blueprint schema, project-level agent contract
- **`references/pipeline-overview.md`** — full architecture, delegation logic, configuration reference
