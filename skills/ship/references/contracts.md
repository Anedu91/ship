# Ship Data Contracts

The ship pipeline communicates through **inline prompt passing** — no shared state files. The orchestrator is the central hub that receives the plan and distributes PR blueprints to executors.

## PR Blueprint Schema

The planner outputs this structured text. The orchestrator parses it and passes each PR section inline to its executor.

```markdown
# Ship Plan

## Summary
<one paragraph describing the full feature>

## Configuration
- Max lines per PR: <number>
- Branch prefix: <prefix>
- Validation commands:
  - <command 1>
  - <command 2>

## Stack

### PR 1: <title>
- Branch: `<prefix>/<short-name>`
- Executor: <path to executor agent, or "default" for built-in>
- Description: <what this PR does and why>
- Files:
  - `path/to/file.py` [CREATE]: <what goes here — classes, functions with signatures>
  - `path/to/other.py` [MODIFY]: <exact changes — add method X to class Y, modify function Z signature>
  - `path/to/config.yaml` [MODIFY]: <add key X with value Y>
- Estimated lines: <number>

### PR 2: <title>
- Branch: `<prefix>/<short-name>`
- Executor: <path to executor agent, or "default" for built-in>
- Description: <what this PR does and why>
- Depends on: PR 1
- Files:
  - `path/to/file.py` [MODIFY]: <exact changes>
- Estimated lines: <number>
```

### File operation types

| Operation | Meaning |
|-----------|---------|
| `[CREATE]` | New file. Blueprint specifies full structure: classes, functions, exports. |
| `[MODIFY]` | Existing file. Blueprint specifies exact changes: add/remove/change with method-level detail. |
| `[DELETE]` | Remove file. Blueprint explains why. |
| `[RENAME]` | Move/rename. Blueprint specifies old and new path. |

### Blueprint detail level

The planner MUST provide enough detail that the executor can work mechanically:

**Insufficient:**
```
- `src/models/user.py` [CREATE]: User model
```

**Sufficient:**
```
- `src/models/user.py` [CREATE]: SQLAlchemy model `User` with fields: id (UUID, primary key), email (String, unique, indexed), name (String), created_at (DateTime, server_default=now). Add `UserCreate(BaseModel)` and `UserUpdate(BaseModel)` Pydantic schemas.
```

## Executor Input Contract

Each executor receives its blueprint **inline in the prompt** from the orchestrator:

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

The executor does NOT read any state files. It implements, validates, commits, and reports success/failure.

## .ship.yaml — Project Configuration

Optional file in the target repo root. Defines project-level overrides.

```yaml
# Max lines changed per PR (default: 250)
maxLinesPerPR: 250

# Max total PRs (default: 7)
maxPRs: 7

# Branch prefix for ship branches (default: "ship/")
branchPrefix: "ship/"

# Validation commands (auto-detected if omitted)
validate:
  - "uv run ruff check ."
  - "uv run ruff format --check ."
  - "uv run mypy src/"
  - "uv run pytest"

# Model per phase (default shown)
# Options: opus, sonnet, haiku
models:
  plan: opus
  execute: sonnet

# Permission mode per phase (all default to bypassPermissions)
# Only set if you want to RESTRICT a phase.
# Options: bypassPermissions, acceptEdits, auto, default
# modes:
#   push: default    # example: require approval before pushing

# Project-level agents (optional — falls back to ship built-ins)
agents:
  # Planners — list of available planners. First match wins.
  planners:
    - path: ".ship/plan.md"
      match: "default"

  # Executors — list of available executors with capability descriptions.
  executors:
    - path: ".agents/backend-engineer.md"
      match: "Python, FastAPI, SQLAlchemy, Pydantic, scrapers, CLI tools"
      model: sonnet
    - path: ".agents/infra-engineer.md"
      match: "Terraform, Docker, CI/CD, deployment, infrastructure"
      model: haiku
```

### Agent selection flow

1. The orchestrator reads all entries from `agents.executors` (and `agents.planners`)
2. The orchestrator passes the available agents list to the planner
3. The planner assigns an executor per PR by matching the PR's work against `match` descriptions
4. The planner writes the chosen agent's `path` into each PR's `Executor:` field
5. During execution, the orchestrator reads the `Executor:` field and loads that agent

If no executor matches or the field is `"default"`, the built-in `ship-execute` skill is used.

### Single-agent shorthand

```yaml
agents:
  executor: ".agents/backend-engineer.md"
  planner: ".ship/plan.md"
```

### Project-level agent contract

**Planner**: Must read requirements (provided inline by orchestrator), scan the repo, and output the plan in the schema above as text. Must NOT write any files.

**Executor**: Must receive its PR blueprint inline from the orchestrator prompt. Implement changes, run validation, commit, and report success/failure. Must NOT read or write state files.

**Pusher**: Must receive feature summary and PR list inline from the orchestrator prompt. Push the stack, create PR descriptions, and return PR URLs.
