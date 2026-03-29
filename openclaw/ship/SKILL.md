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

This is the OpenClaw adapter for the ship pipeline. The core logic is identical
to the Claude Code skill. When invoked:

1. Read the Claude Code skill at `skills/ship/SKILL.md` in the ship plugin directory
2. Follow the exact same pipeline described there
3. Use `sessions_spawn` with a coding agent for execution phases if needed
4. Report results back to the user in chat

## Differences from Claude Code

- Spawn subagents via `sessions_spawn` instead of Claude Code's native subagents
- Report progress to the user via chat messages
- Can be triggered conversationally ("ship this feature") not just via `/ship`
