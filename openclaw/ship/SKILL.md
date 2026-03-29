---
name: ship
description: "Autonomous shipping pipeline: reads requirements, plans stacked PRs (<250 lines each), executes with coding agents, validates, and pushes to GitHub with detailed PR descriptions."
triggers:
  - ship
  - ship feature
  - ship this
  - implement and ship
---

# /ship — Autonomous Shipping Pipeline (OpenClaw)

You are an orchestrator. When triggered, you run a full development pipeline from requirements to pushed PRs. You do NOT ask for confirmation between phases — you run end-to-end and report results.

## Setup

The ship skill files live at a known location. At the start of every run, resolve the ship plugin root:

```
SHIP_ROOT = /home/anedu/Documents/programing/ship
```

All skill references below are relative to `SHIP_ROOT/skills/`.

## Input

The user provides one of:
- A GitHub issue URL (e.g., "ship github.com/Anedu91/aureum_rag/issues/5")
- A file path to requirements
- Inline text describing what to build
- A conversational request (e.g., "ship this feature: add user auth to aureum_rag")

You must also determine the **target repo** — the repo where code will be written. Infer from:
1. The GitHub issue URL (if provided)
2. Explicit mention in the conversation ("in aureum_rag")
3. Ask the user if unclear

All commands run in the target repo directory, NOT in the ship plugin directory.

## Phase 0: Initialize

1. `cd` into the target repo directory
2. Check for stale state files (`ship-state.md`, `ship-plan.md`). If found, ask user: resume or start fresh?
3. Read `.ship.yaml` from the target repo root (if exists) and merge with defaults:

```yaml
maxLinesPerPR: 250
branchPrefix: "ship/"
validate: # auto-detect
models:
  read: anthropic/claude-sonnet-4-6
  plan: anthropic/claude-opus-4-6
  execute: anthropic/claude-sonnet-4-6
  stack: anthropic/claude-sonnet-4-6
  push: anthropic/claude-sonnet-4-6
agents:
  planners: []
  executors: []
```

Map any shorthand model names: `opus` → `anthropic/claude-opus-4-6`, `sonnet` → `anthropic/claude-sonnet-4-6`.

## Phase 1: Read Requirements

Read `SHIP_ROOT/skills/ship-read/SKILL.md` and follow its instructions directly (no subagent — this is lightweight).

Input: the user's requirements.
Output: `ship-state.md` exists in the target repo root.

Report to user: "📋 Requirements parsed. Planning PRs..."

## Phase 2: Plan

Discover the planner:
1. If `.ship.yaml` has `agents.planners` entries and the file exists → read that file
2. Otherwise → read `SHIP_ROOT/skills/ship-plan/SKILL.md`

This is the heavy thinking phase. Spawn a subagent:

```
sessions_spawn:
  task: |
    You are a planner for a shipping pipeline. Read and follow the instructions 
    in <planner_file>. 
    
    Available executors: <list from .ship.yaml agents.executors>
    
    The requirements are in ship-state.md in the repo root.
    Write ship-plan.md following the schema in <SHIP_ROOT>/skills/ship/references/contracts.md
  model: <models.plan>
  cwd: <target_repo_path>
  mode: run
```

Wait for completion. Validate that `ship-plan.md` has `## Stack` section with proper PR blueprints.

Report to user: "📐 Plan ready. X PRs planned. Executing..."

## Phase 3: Execute Stack

For each PR in `ship-plan.md`, in order:

### 3a. Create Branch

Run directly (lightweight, no subagent):
```bash
cd <target_repo>
gt create <branch-name> -m "feat: <PR title>"
```

If `gt` is not available or fails, fall back to:
```bash
git checkout -b <branch-name>
```

### 3b. Execute PR

Update status in `ship-plan.md` to `in-progress`.

Discover executor from the PR's `Executor:` field:
1. If `Executor:` is a path and file exists → read that file for instructions
2. Otherwise → use `SHIP_ROOT/skills/ship-execute/SKILL.md`

Resolve model:
1. If the executor entry has a `model` override → use that
2. Otherwise → use `models.execute`

Spawn a coding subagent:

```
sessions_spawn:
  task: |
    You are implementing a single PR. Read and follow the instructions in <executor_file>.
    
    PR Blueprint:
    <full PR section from ship-plan.md>
    
    Validation commands:
    <commands from Configuration section>
    
    After implementation:
    1. Run all validation commands
    2. Fix failures (up to 3 retries)  
    3. git add -A && git commit --amend -m "feat: <PR title>"
    4. Update Status in ship-plan.md to "done" or "failed:<reason>"
  model: <resolved_model>
  cwd: <target_repo_path>
  mode: run
```

Wait for completion.

Report to user: "✅ PR 1/N: <title> — <done|failed>"

### 3c. Continue

If failed, log it. Move to next PR. Do NOT abort.

## Phase 4: Push

Read `SHIP_ROOT/skills/ship-push/SKILL.md` and follow its instructions directly.

Run:
```bash
gt submit --no-interactive
```

For each PR, create a detailed description using `gh pr edit`.

If `gt submit` fails, fall back to:
```bash
git push origin <branch> && gh pr create --base <parent-branch> --head <branch> --title "feat: <title>" --body-file /tmp/ship-pr-N.md
```

Report to user: "🚀 Stack pushed. Creating PR descriptions..."

## Phase 5: Report

Send the user a summary:

```
## Ship Complete 🚀

Feature: <name>
PRs created: <count>

Stack:
1. <title> — <url> (<lines> lines) [done]
2. <title> — <url> (<lines> lines) [done]

Validation: all passed
```

Clean up: delete `ship-state.md` and `ship-plan.md`.

## Fallback Strategy

If `gt` (Graphite) is not available:
- Use plain `git checkout -b` for branches
- Use `gh pr create` instead of `gt submit`
- PRs won't be stacked (flat branches off main) but the pipeline still works

If `gh` is not available:
- Push branches but skip PR creation
- Report branch names instead of PR URLs

## Error Handling

- If a phase fails completely, report the error and stop. Don't push a half-built stack.
- If a single PR fails validation after retries, mark it failed and continue to the next.
- If planning produces an invalid plan, retry planning once. If still invalid, report and stop.

## Key Differences from Claude Code Version

| Aspect | Claude Code | OpenClaw |
|--------|------------|----------|
| Subagents | `Agent` tool per phase | `sessions_spawn` for plan + execute only |
| Lightweight phases | Subagent | Direct execution (read, stack, push) |
| Model control | Per-subagent | Per `sessions_spawn` |
| Plugin root | `${CLAUDE_PLUGIN_ROOT}` | Hardcoded `SHIP_ROOT` path |
| Invocation | `/ship:ship <args>` | Conversational or explicit |
| Progress | Silent until done | Reports each phase to user in chat |
