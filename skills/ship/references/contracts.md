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

# Project-level agents (optional — falls back to ship built-ins)
agents:
  planner: ".ship/plan.md"
  executor: ".ship/execute.md"
```

### Project-level agent contract

Project-level planner and executor files are markdown instruction sets. They MUST:

1. **Planner**: Read `ship-state.md`, write `ship-plan.md` following the schema above exactly.
2. **Executor**: Read the PR blueprint section passed to it, implement changes, run validation, update the PR's Status field in `ship-plan.md`.

The schemas above are the contract. Deviation breaks the pipeline.
