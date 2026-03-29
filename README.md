# Ship

Autonomous shipping pipeline for AI coding agents. One command: requirements in, stacked PRs out.

## What it does

```
/ship <requirements>
   │
   ├── 1. Read & understand requirements
   ├── 2. Plan stacked PRs (<250 lines each)
   ├── 3. Execute each PR sequentially
   ├── 4. Validate (linters, types, tests)
   ├── 5. Push stacked PRs to GitHub
   └── 6. Report summary
```

## Architecture

Ship is a thin orchestrator that delegates to specialized skills. Planning and execution are **dynamically delegated** — projects can provide their own planner and executor.

```
/ship <requirements>
  │
  ├─ ship-read       (fixed)    → parse requirements
  ├─ planner         (dynamic)  → plan PRs (project or fallback)
  ├─ FOR EACH PR:
  │   ├─ ship-stack  (fixed)    → create graphite branch
  │   └─ executor    (dynamic)  → implement + validate (project or fallback)
  ├─ ship-push       (fixed)    → push stack + PR descriptions
  └─ report + cleanup
```

| Skill | Role | Dynamic? |
|-------|------|----------|
| `ship` | Orchestrator — runs the pipeline | No |
| `ship-read` | Parse requirements from issues, files, or text | No |
| `ship-plan` | Fallback planner — break work into PR blueprints | Replaceable |
| `ship-execute` | Fallback executor — implement a single PR | Replaceable |
| `ship-stack` | Graphite branch management | No |
| `ship-push` | Push stack and create PR descriptions | No |

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
  read: sonnet        # lightweight parsing
  plan: opus          # heavy thinking, architecture decisions
  execute: sonnet     # mechanical, follow the blueprint
  stack: haiku        # simple branch operations
  push: sonnet        # template-based PR descriptions

# Permission mode per phase (default: bypassPermissions)
# Options: bypassPermissions, acceptEdits, auto, default
modes:
  execute: bypassPermissions

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
3. The planner assigns the best-fit executor per PR based on `match` descriptions, not agent names
4. During execution, Ship loads the assigned agent for each PR
5. If no agent matches, Ship uses its built-in fallback skills

### Writing a planner

A planner is a markdown file with instructions for breaking work into PRs. It must:
- Read `ship-state.md` (requirements + repo context)
- Read the available executors list and assign one per PR
- Write `ship-plan.md` following the [plan schema](skills/ship/references/contracts.md)

Example (`.ship/plan.md`):
```markdown
Break work following this project's layered architecture:
1. Database models and migrations first
2. Service layer (business logic)
3. API routes and middleware
4. Tests for each layer

Use SQLAlchemy for models, Pydantic for schemas.
Each PR should touch at most one layer.
...
```

### Writing an executor

An executor is a markdown file with instructions for implementing a single PR. It must:
- Read the PR blueprint it receives
- Implement, validate, and commit
- Update the PR status in `ship-plan.md`

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
