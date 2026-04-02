# Ship Pipeline Overview

## Architecture

Ship is a thin orchestrator that delegates all work to specialized subagents. No shared state files — the orchestrator passes data inline between phases. The orchestrator itself executes zero bash commands; it only reads config, spawns agents, and reports results.

```
/ship <requirements>
  │
  ├─ Phase 1: Plan    (1 subagent)
  │   ├─ Reads requirements (GitHub issue / file / inline text)
  │   ├─ Scans repo for context
  │   ├─ Produces PR blueprint as structured text
  │   ├─ project: .ship.yaml agents.planner path
  │   └─ fallback: ship-plan skill
  │
  ├─ Phase 2: Execute  (N subagents, sequential)
  │   └─ FOR EACH PR:
  │       ├─ Receives blueprint inline from orchestrator
  │       ├─ gt create branch → implement → validate → commit
  │       ├─ planner-assigned: Executor field in PR blueprint
  │       └─ fallback: ship-execute skill
  │
  ├─ Phase 3: Push    (1 subagent)
  │   ├─ Receives feature summary + PR list inline
  │   ├─ gt submit + gh pr edit with descriptions
  │   └─ Returns PR URLs
  │
  └─ Phase 4: Report  (orchestrator directly, text only)
```

**Total subagents: N + 2** (1 planner + N executors + 1 pusher). The orchestrator executes zero bash commands.

## Data Flow

```
Requirements (inline)
    │
    ▼
  Planner ──outputs──▶ Plan text (captured by orchestrator)
    │
    ▼
  Orchestrator parses plan, extracts per-PR sections
    │
    ├─▶ Executor 1 (receives PR 1 blueprint inline)
    ├─▶ Executor 2 (receives PR 2 blueprint inline)
    └─▶ Executor N (receives PR N blueprint inline)
    │
    ▼
  Pusher (receives feature summary + PR list inline)
    │
    ▼
  Orchestrator outputs final report (text only)
```

No files are written or read between phases. The orchestrator holds the plan in memory and passes relevant sections inline to each subagent.

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
4. During execution, the orchestrator reads each PR's `Executor:` field and loads that agent

Selection is based on what the agent **can do** (its `match` field), not its name.

### Planner Discovery

```
DISCOVER_PLANNER:
  1. Read agents.planners from .ship.yaml (or agents.planner shorthand)
  2. IF entries exist AND first matching file exists:
       → Read file, use its instructions
  3. ELSE:
       → Read ${CLAUDE_PLUGIN_ROOT}/skills/ship-plan/SKILL.md
```

## Subagent Architecture

Each phase is spawned as a subagent via the Agent tool. This provides:

1. **Model control** — use Opus for planning (heavy thinking), Sonnet for execution (mechanical). The orchestrator itself runs on Sonnet to minimize cost.
2. **Permission bypass** — subagents run with `bypassPermissions` by default (passed explicitly as `mode` parameter)
3. **Context isolation** — each executor only sees its own PR blueprint, not the entire plan
4. **Pure orchestrator** — the orchestrator executes zero bash commands. All side effects (file creation, git operations, pushing) happen inside subagents.

The orchestrator spawns each phase with:
- **model**: from `models.<phase>` config, or per-agent `model` override
- **mode**: from `modes.<phase>` config, default `"bypassPermissions"`. MUST always be passed explicitly.

### Model resolution for executors

1. If the executor entry in `agents.executors` has a `model` field → use that
2. Otherwise → use `models.execute` from config
3. Otherwise → default `sonnet`

## Stacked PRs Are Sequential

Each PR branch is created from the previous PR's branch using Graphite (the executor runs `gt create` before implementing). PR 2 depends on PR 1's code. There is no parallel execution of stacked PRs.

The planner does the heavy thinking. The executor follows blueprints mechanically — including branch creation.

## Configuration Reference

`.ship.yaml` in repo root (all fields optional):

| Field | Default | Description |
|-------|---------|-------------|
| `maxLinesPerPR` | 250 | Max lines changed per PR |
| `maxPRs` | 7 | Max total PRs the planner can create |
| `branchPrefix` | `"ship/"` | Prefix for branch names |
| `validate` | auto-detected | List of validation commands |
| `models.plan` | `opus` | Model for planning (heavy thinking) |
| `models.execute` | `sonnet` | Default model for execution |
| `models.push` | `sonnet` | Model for pushing PRs |
| `modes.<phase>` | `bypassPermissions` | Override permission mode for a phase |
| `agents.planners` | built-in `ship-plan` | List of `{path, match}` planners |
| `agents.planner` | — | Shorthand: single planner path |
| `agents.executors` | built-in `ship-execute` | List of `{path, match, model}` executors |
| `agents.executor` | — | Shorthand: single executor path |

## Writing Project-Level Agents

### Planner

A project-level planner is a markdown file with instructions for breaking work into PRs. It receives requirements inline from the orchestrator and outputs the plan as structured text.

It should encode project-specific knowledge:
- Architecture patterns (e.g., "models first, then services, then API routes")
- PR sizing heuristics for this codebase
- Naming conventions and file organization
- Domain-specific splitting strategies

The planner MUST:
- Read requirements from the input provided by the orchestrator
- Scan the repo for context
- Assign the best-fit executor per PR via the `Executor:` field
- Output the plan following the exact schema in `contracts.md`
- Provide method-level detail in file blueprints
- NOT write any files to disk

### Executor

A project-level executor is a markdown file with instructions for implementing a single PR. It receives its blueprint inline from the orchestrator prompt.

It should encode project-specific knowledge:
- Code style and patterns
- Testing conventions
- Framework-specific implementation details

The executor MUST:
- Implement changes following the blueprint in the prompt
- Run validation commands from the prompt
- Fix failures (up to 3 retries)
- Stage and commit
- Report success or failure
- NOT read or write state files
