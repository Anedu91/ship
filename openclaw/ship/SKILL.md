---
name: ship
description: "Autonomous shipping pipeline: reads requirements, plans stacked PRs (<250 lines each), executes with coding agents, validates (mypy/ruff/tests), and pushes to GitHub with detailed PR descriptions."
triggers:
  - ship
  - ship feature
  - ship this
  - implement and ship
---

# /ship — Autonomous Shipping Pipeline (OpenClaw)

OpenClaw adapter for the ship pipeline. The architecture is identical to the Claude Code plugin — a thin orchestrator with dynamic delegation.

## When invoked

1. Read the orchestrator skill at `skills/ship/SKILL.md` in the ship plugin directory
2. Follow the exact same pipeline described there
3. Use `sessions_spawn` with a coding agent for execution phases if needed
4. Report progress to the user via chat messages

## Differences from Claude Code

- Spawn subagents via `sessions_spawn` instead of Claude Code's native subagents
- Report progress to the user via chat messages
- Can be triggered conversationally ("ship this feature") not just via `/ship`
- When reading skill files, use the filesystem path to the plugin instead of `${CLAUDE_PLUGIN_ROOT}`

## Dynamic delegation

The same `.ship.yaml` configuration works in OpenClaw. Project-level planners and executors are discovered the same way — check `agents.planner` / `agents.executor` paths, fall back to built-in skills.

## Reference

- **Pipeline architecture**: `skills/ship/references/pipeline-overview.md`
- **Data contracts**: `skills/ship/references/contracts.md`
