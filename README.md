# Ship

Autonomous shipping pipeline for AI coding agents. One command: requirements in, stacked PRs out.

## What it does

```
/ship <requirements>
   │
   ├── 1. Plan: read requirements + scan repo + blueprint PRs
   ├── 2. Execute each PR sequentially (branch, implement, validate, commit)
   ├── 3. Push stacked PRs to GitHub + create PR descriptions
   ├── 4. Report summary
   └── Done
```

## Architecture

Ship is a pure orchestrator that delegates all work to specialized subagents. It executes zero bash commands — only reads config, spawns agents, and reports results. Planning and execution are **dynamically delegated** — projects can provide their own planner and executor. No shared state files — data is passed inline between phases.

```
/ship <requirements>
  │
  ├─ Phase 1: Plan   (1 subagent)
  │   ├─ Read requirements + scan repo
  │   └─ Output PR blueprints as text
  │
  ├─ Phase 2: Execute (N subagents, sequential)
  │   └─ FOR EACH PR:
  │       ├─ Receives blueprint inline from orchestrator
  │       └─ gt create branch → implement → validate → commit
  │
  ├─ Phase 3: Push   (1 subagent)
  │   ├─ Receives feature summary + PR list inline
  │   └─ gt submit + gh pr edit with descriptions
  │
  └─ Phase 4: Report (orchestrator directly, text only)
```

**Total subagents: N + 2** (1 planner + N executors + 1 pusher)

| Skill | Role | Dynamic? |
|-------|------|----------|
| `ship` | Orchestrator — pure delegation, zero commands | No |
| `ship-plan` | Planner — read requirements + blueprint PRs | Replaceable |
| `ship-execute` | Executor — implement a single PR | Replaceable |
| `ship-push` | Pusher — push stack + create PR descriptions | No |
| `ship-stack` | Graphite branch management (reference) | No |

## Supported tools

| Tool | Integration | How |
|------|-------------|-----|
| **Claude Code** | Plugin | `/ship:ship <requirements>` |
| **OpenClaw** | Skill | `/ship <requirements>` or conversational |
| **OpenCode** | Planned | — |
| **Codex** | Planned | — |

## Install

### Claude Code

```bash
# Load as local plugin
claude --plugin-dir /path/to/ship

# Or add to .claude/settings.json
{
  "plugins": ["/path/to/ship"]
}
```

### OpenClaw

Copy `openclaw/ship/` to your workspace `skills/` directory.

## Requirements

- **Graphite CLI** (`gt`) — for stacked PR management
- **GitHub CLI** (`gh`) — for PR creation and issue fetching
- **Project tooling** — ruff, mypy, pytest (Python) or equivalent for your stack

## Configuration

Create `.ship.yaml` in your repo root (optional):

```yaml
# Max lines per PR (default: 250)
maxLinesPerPR: 250

# Branch prefix for ship branches (default: "ship/")
branchPrefix: "ship/"

# Validation commands (auto-detected if not specified)
validate:
  - "uv run ruff check ."
  - "uv run ruff format --check ."
  - "uv run mypy src/"
  - "uv run pytest"

# Model per phase (options: opus, sonnet, haiku)
models:
  plan: opus          # heavy thinking, architecture decisions
  execute: sonnet     # mechanical, follow the blueprint
  push: sonnet        # push stack + PR descriptions

# Permission mode per phase (all default to bypassPermissions)
# Only set to RESTRICT a phase. Options: default, acceptEdits, auto
# modes:
#   push: default    # require approval before pushing

# Project-level agents (optional)
agents:
  # Single agent shorthand
  planner: ".ship/plan.md"
  executor: ".agents/backend-engineer.md"

  # Or multiple agents with capability descriptions
  # Per-agent model override takes precedence over models.execute
  executors:
    - path: ".agents/backend-engineer.md"
      match: "Python, FastAPI, SQLAlchemy, scrapers, CLI tools"
      model: sonnet
    - path: ".agents/infra-engineer.md"
      match: "Terraform, Docker, CI/CD, deployment"
      model: haiku
    - path: ".agents/frontend-engineer.md"
      match: "React, TypeScript, UI components"
```

## Project-level agents

Ship's planner and executor can be replaced per-project. This lets you encode project-specific knowledge — architecture patterns, code conventions, splitting strategies, validation — that the generic fallbacks don't know.

### How it works

1. Ship reads `.ship.yaml` from the repo root
2. The orchestrator passes the available executors list to the planner
3. The planner assigns the best-fit executor per PR based on `match` descriptions
4. During execution, Ship loads the assigned agent for each PR
5. If no agent matches, Ship uses its built-in fallback skills

### Writing a planner

A planner is a markdown file with instructions for breaking work into PRs. It must:
- Read requirements (provided inline by the orchestrator)
- Scan the repo for context
- Output the PR blueprint as structured text (NOT write to a file)
- Assign executors per PR via the `Executor:` field

Example (`.ship/plan.md`):
```markdown
Break work following this project's layered architecture:
1. Database models and migrations first
2. Service layer (business logic)
3. API routes and middleware
4. Tests bundled with each layer

Use SQLAlchemy for models, Pydantic for schemas.
Each PR should touch at most one layer.
...
```

### Writing an executor

An executor is a markdown file with instructions for implementing a single PR. It receives its blueprint inline from the orchestrator. It must:
- Implement changes following the blueprint
- Run validation and fix failures (up to 3 retries)
- Commit and report success/failure

Example (`.ship/execute.md`):
```markdown
Implement the PR blueprint following project conventions:
- Models inherit from Base in src/db/base.py
- Services use dependency injection via Depends
- Routes go in src/api/v1/ and are registered in __init__.py
- Run: uv run ruff check . && uv run mypy src/ && uv run pytest
...
```

## How it works

### Planning

Ship breaks work into PRs of **<250 lines each**. Each PR:
- Is a complete, working increment
- Passes all validation independently
- Builds on the previous PR in the stack
- Has a single, clear responsibility
- Includes method-level detail in its blueprint

### Validation

Before committing, each PR is validated against the project's toolchain:
- **Python**: `ruff check` → `ruff format --check` → `mypy` → `pytest`
- **Node**: `lint` → `typecheck` → `test`
- Falls back to whatever linter/test config exists
- Retries up to 3 times on failure

### PR descriptions

Every PR includes:
- What changed and why
- Implementation approach
- File-by-file change summary
- Validation status
- Stack position (PR X/N)

## License

MIT
