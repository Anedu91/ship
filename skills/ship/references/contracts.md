# Ship Data Contracts

All components in the ship pipeline communicate through two state files in the repo root. Both are gitignored and deleted after a successful run.

## ship-state.md — Requirements & Context

Created by `ship-read`. Consumed by the planner.

```markdown
# Ship State

## Requirements

### Goal
<what needs to be built — one paragraph>

### Constraints
- <tech stack requirements>
- <patterns to follow>
- <conventions to match>

### Acceptance Criteria
- <criterion 1>
- <criterion 2>

### Source
- Type: <github-issue | file | inline>
- Reference: <URL, file path, or "inline">

## Repo Context

### Tech Stack
- Language: <python | typescript | go | etc.>
- Package manager: <uv | npm | pnpm | cargo | etc.>
- Framework: <if applicable>

### Validation Commands
<from .ship.yaml or auto-detected>
- <command 1>
- <command 2>

### Project Structure
<key directories and their purpose, relevant to the goal>

### Relevant Files
- `path/to/file` — <why it's relevant>
```

## ship-plan.md — PR Blueprints

Created by the planner. Consumed by the executor, `ship-stack`, and `ship-push`.

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
- Status: <pending | in-progress | done | failed:<reason>>
- Files:
  - `path/to/file.py` [CREATE]: <what goes here — classes, functions with signatures>
  - `path/to/other.py` [MODIFY]: <exact changes — add method X to class Y, modify function Z signature>
  - `path/to/config.yaml` [MODIFY]: <add key X with value Y>
- Estimated lines: <number>
- Validation notes: <any special concerns for this PR>

### PR 2: <title>
- Branch: `<prefix>/<short-name>`
- Executor: <path to executor agent, or "default" for built-in>
- Description: <what this PR does and why>
- Status: pending
- Depends on: PR 1
- Files:
  - `path/to/file.py` [MODIFY]: <exact changes>
- Estimated lines: <number>
- Validation notes: <notes>
```

### File operation types

| Operation | Meaning |
|-----------|---------|
| `[CREATE]` | New file. Blueprint specifies full structure: classes, functions, exports. |
| `[MODIFY]` | Existing file. Blueprint specifies exact changes: add/remove/change with method-level detail. |
| `[DELETE]` | Remove file. Blueprint explains why. |
| `[RENAME]` | Move/rename. Blueprint specifies old and new path. |

### Status values

| Status | Set by | Meaning |
|--------|--------|---------|
| `pending` | Planner | Not yet started |
| `in-progress` | Orchestrator | Currently being executed |
| `done` | Executor | Implementation complete, validation passed |
| `failed:<reason>` | Executor | Failed after 3 retry attempts |

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

## PR Blueprint (single PR)

The orchestrator extracts one PR section from `ship-plan.md` and passes it to the executor. The executor receives the full PR section (everything under `### PR N: <title>`) plus the Configuration section for validation commands.

## PR Blueprint (executor assignment)

The planner assigns an executor to each PR via the `Executor:` field. The orchestrator reads this field and loads the corresponding agent file. If set to `"default"`, the built-in `ship-execute` skill is used.

The planner picks executors by matching each PR's content against the `match` descriptions in `.ship.yaml`. Selection is based on what the agent **can do** (its `match` field), not its name.

## .ship.yaml — Project Configuration

Optional file in the target repo root. Defines project-level overrides.

```yaml
# Max lines changed per PR (default: 250)
maxLinesPerPR: 250

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
  read: sonnet
  plan: opus
  execute: sonnet
  stack: haiku
  push: sonnet

# Permission mode per phase (all default to bypassPermissions)
# Only set if you want to RESTRICT a phase.
# Options: bypassPermissions, acceptEdits, auto, default
# modes:
#   push: default    # example: require approval before pushing

# Project-level agents (optional — falls back to ship built-ins)
agents:
  # Planners — list of available planners. First match wins.
  # If omitted or empty, uses built-in ship-plan.
  planners:
    - path: ".ship/plan.md"
      match: "default"

  # Executors — list of available executors with capability descriptions.
  # The planner reads these and assigns the best fit per PR.
  # Per-agent model override takes precedence over models.execute.
  # If omitted or empty, uses built-in ship-execute for all PRs.
  executors:
    - path: ".agents/backend-engineer.md"
      match: "Python, FastAPI, SQLAlchemy, Pydantic, scrapers, CLI tools"
      model: sonnet
    - path: ".agents/infra-engineer.md"
      match: "Terraform, Docker, CI/CD, deployment, infrastructure"
      model: haiku
    - path: ".agents/frontend-engineer.md"
      match: "React, TypeScript, UI components, styling"
```

### Agent selection flow

1. The orchestrator reads all entries from `agents.executors` (and `agents.planners`)
2. The orchestrator passes the available agents list to the planner
3. The planner assigns an executor per PR by matching the PR's work against `match` descriptions
4. The planner writes the chosen agent's `path` into each PR's `Executor:` field
5. During execution, the orchestrator reads the `Executor:` field and loads that agent

If no executor matches or the field is `"default"`, the built-in `ship-execute` skill is used.

### Single-agent shorthand

For projects with just one custom executor or planner, use the shorthand:

```yaml
agents:
  executor: ".agents/backend-engineer.md"
  planner: ".ship/plan.md"
```

This is equivalent to a single-entry list with `match: "default"`. The planner will use this executor for all PRs.

### Project-level agent contract

Project-level planner and executor files are markdown instruction sets. They MUST:

1. **Planner**: Read `ship-state.md` and the available agents list. Write `ship-plan.md` following the schema above exactly, including an `Executor:` field per PR.
2. **Executor**: Read the PR blueprint section passed to it, implement changes, run validation, update the PR's Status field in `ship-plan.md`.

The schemas above are the contract. Deviation breaks the pipeline.
