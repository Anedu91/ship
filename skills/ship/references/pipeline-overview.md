# Ship Pipeline Overview

## Architecture

Ship is a thin orchestrator that delegates work to specialized skills. Planning and execution are dynamically delegated to project-level agents when configured, falling back to built-in defaults.

```
/ship <requirements>
  │
  ├─ ship-read       (fixed)    → parse requirements → ship-state.md
  │
  ├─ planner         (dynamic)  → read ship-state.md → ship-plan.md
  │   ├─ project: .ship.yaml agents.planner path
  │   └─ fallback: ship-plan skill
  │
  ├─ FOR EACH PR (sequential):
  │   ├─ ship-stack  (fixed)    → gt create <branch>
  │   ├─ executor    (dynamic)  → implement blueprint → validate → commit
  │   │   ├─ planner-assigned: Executor field in PR blueprint
  │   │   └─ fallback: ship-execute skill
  │   └─ update ship-plan.md status
  │
  ├─ ship-push       (fixed)    → gt submit + PR descriptions
  │
  └─ cleanup → delete state files → output summary
```

## Dynamic Delegation

### Multi-Agent Selection

Projects can register multiple executors (and planners), each with a `match` description of its capabilities:

```yaml
agents:
  executors:
    - path: ".agents/backend-engineer.md"
      match: "Python, FastAPI, SQLAlchemy, scrapers, CLI tools"
    - path: ".agents/infra-engineer.md"
      match: "Terraform, Docker, CI/CD, deployment"
```

**Selection flow:**

1. The orchestrator reads all available agents from `.ship.yaml`
2. The orchestrator passes the full list to the planner
3. The planner assigns the best-fit executor per PR by matching the PR's work against `match` descriptions
4. The planner writes the agent's `path` into each PR's `Executor:` field
5. During execution, the orchestrator reads each PR's `Executor:` field and loads that agent

Selection is based on what the agent **can do** (its `match` field), not its name. An agent named "backend-engineer" with `match: "Python, scrapers"` is the right pick for Python scraper work.

If no executor matches or `Executor: default`, the built-in `ship-execute` skill is used.

### Planner Discovery

```
DISCOVER_PLANNER:
  1. Read agents.planners from .ship.yaml (or agents.planner shorthand)
  2. IF entries exist AND first matching file exists:
       → Read file, use its instructions
  3. ELSE:
       → Read ${CLAUDE_PLUGIN_ROOT}/skills/ship-plan/SKILL.md
```

Project-level agents receive the same inputs and must produce the same outputs as built-in skills. See `contracts.md` for exact schemas.

## Subagent Architecture

Each phase is spawned as a subagent via the Agent tool. This provides:

1. **Model control** — use Opus for planning (heavy thinking), Sonnet for execution (mechanical), Haiku for stack ops (lightweight)
2. **Permission bypass** — subagents run with `bypassPermissions` by default, eliminating approval prompts
3. **Context isolation** — each phase gets its own context window, preventing overflow on large features

The orchestrator spawns each phase with:
- **model**: from `models.<phase>` config, or per-agent `model` override
- **mode**: from `modes.<phase>` config

### Model resolution for executors

Per-agent model overrides take precedence:

1. If the executor entry in `agents.executors` has a `model` field → use that
2. Otherwise → use `models.execute` from config
3. Otherwise → default `sonnet`

This lets you run a simple infra agent on Haiku while a complex backend agent uses Sonnet.

## Stacked PRs Are Sequential

Each PR branch is created from the previous PR's branch using Graphite. PR 2 depends on PR 1's code. There is no parallel execution of stacked PRs.

The planner does the heavy thinking. The executor follows blueprints mechanically. This means sequential execution is fast — each PR is just implementation + validation, not design.

## State Files

| File | Created by | Consumed by | Lifecycle |
|------|-----------|-------------|-----------|
| `ship-state.md` | ship-read | planner | Deleted after plan is written |
| `ship-plan.md` | planner | executor, ship-push | Deleted after report |

Both files are gitignored. If stale state files exist on startup, the orchestrator asks whether to resume or start fresh.

## Configuration Reference

`.ship.yaml` in repo root (all fields optional):

| Field | Default | Description |
|-------|---------|-------------|
| `maxLinesPerPR` | 250 | Max lines changed per PR |
| `branchPrefix` | `"ship/"` | Prefix for branch names |
| `validate` | auto-detected | List of validation commands |
| `models.read` | `sonnet` | Model for reading requirements |
| `models.plan` | `opus` | Model for planning (heavy thinking) |
| `models.execute` | `sonnet` | Default model for execution |
| `models.stack` | `haiku` | Model for branch operations |
| `models.push` | `sonnet` | Model for pushing PRs |
| `modes.read` | `bypassPermissions` | Permission mode for reading |
| `modes.plan` | `bypassPermissions` | Permission mode for planning |
| `modes.execute` | `bypassPermissions` | Permission mode for execution |
| `modes.stack` | `bypassPermissions` | Permission mode for branch ops |
| `modes.push` | `bypassPermissions` | Permission mode for pushing |
| `agents.planners` | built-in `ship-plan` | List of `{path, match}` planners |
| `agents.planner` | — | Shorthand: single planner path |
| `agents.executors` | built-in `ship-execute` | List of `{path, match, model}` executors |
| `agents.executor` | — | Shorthand: single executor path |

## Writing Project-Level Agents

### Planner

A project-level planner is a markdown file with instructions for breaking work into PRs. It should encode project-specific knowledge:

- Architecture patterns (e.g., "models first, then services, then API routes")
- PR sizing heuristics for this codebase
- Naming conventions and file organization
- Domain-specific splitting strategies

The planner MUST:
- Read `ship-state.md` for requirements and repo context
- Read the available executors list provided by the orchestrator
- Assign the best-fit executor per PR via the `Executor:` field
- Write `ship-plan.md` following the exact schema in `contracts.md`
- Provide method-level detail in file blueprints

### Executor

A project-level executor is a markdown file with instructions for implementing a single PR. It should encode project-specific knowledge:

- Code style and patterns
- Testing conventions
- Validation commands and how to interpret failures
- Framework-specific implementation details

The executor MUST:
- Read the PR blueprint section it receives
- Implement changes following the blueprint
- Run validation commands from the plan's Configuration section
- Fix failures (up to 3 retries)
- Stage, commit, and update status in `ship-plan.md`

### Example: Python project

`.ship/plan.md`:
```markdown
Break work following this project's layered architecture:
1. Database models and migrations first
2. Service layer (business logic)
3. API routes
4. Tests for each layer

Use SQLAlchemy for models, Pydantic for schemas. Each PR should touch
at most one layer. Always include type hints.
...
```

`.ship/execute.md`:
```markdown
Implement the PR blueprint. Follow these project conventions:
- All models inherit from `Base` in `src/db/base.py`
- Services go in `src/services/` with dependency injection via `Depends`
- Routes go in `src/api/v1/` and are registered in `src/api/v1/__init__.py`
- Run validation: `uv run ruff check . && uv run mypy src/ && uv run pytest`
...
```
