---
name: ship-plan
description: "This skill should be used as the fallback planner when the ship orchestrator cannot find a project-level planner. Reads requirements, scans the repo, and produces detailed stacked PR blueprints."
allowed-tools: Read, Write, Glob, Grep, Bash(gh:*), Bash(cat:*)
---

# ship-plan — Planner

Read requirements, scan the repo for context, and produce a detailed PR blueprint.

## Input

The orchestrator provides:
- **Requirements input** — a GitHub issue URL, file path, or inline text
- **Available executors** — list of `{path, match, model}` from `.ship.yaml`
- **Configuration** — `maxLinesPerPR`, `maxPRs`, `branchPrefix`

## Step 1: Read Requirements

Determine the input type:

- **GitHub issue URL** (contains `github.com` or matches `#<number>` or `<org>/<repo>#<number>`):
  Fetch with `gh issue view <number> --json title,body,comments,labels`

- **File path** (path exists on disk):
  Read the file completely.

- **Inline text** (anything else):
  Use the text directly.

From the raw input, identify:

1. **Goal** — what needs to be built (one paragraph)
2. **Constraints** — tech stack, patterns, conventions mentioned or implied
3. **Acceptance criteria** — how to know it's done. If not explicit, infer from the goal.

## Step 2: Scan Repo Context

Read the target repo to understand the codebase:

1. **Tech stack** — check for `pyproject.toml`, `package.json`, `Cargo.toml`, `go.mod`, `Makefile`
2. **Validation commands** — if `.ship.yaml` has `validate`, use those. Otherwise auto-detect:
   - `pyproject.toml` → `uv run ruff check . && uv run ruff format --check . && uv run mypy src/ && uv run pytest`
   - `package.json` → `npm run lint && npm run typecheck && npm test`
   - `Makefile` → `make check` or `make test`
3. **Relevant files** — files likely to be touched based on the requirements

## Step 3: Plan the Stack

### Executor Assignment

For each PR, assign the best-fit executor by matching the PR's work against the `match` descriptions. Selection is based on what the agent **can do**, not its name.

If no executors are provided or none match, set `Executor: default` (uses built-in ship-execute).

### Planning Rules

1. Each PR MUST have less than `maxLinesPerPR` lines changed (hard limit)
2. The total number of PRs MUST NOT exceed `maxPRs` (hard limit, default: 7)
3. Each PR is a complete, working increment — tests pass independently
4. Each PR has a single, clear responsibility
5. **Aim for 3–7 PRs total.** Prefer fewer well-scoped PRs over many tiny ones. Only split further when a single PR would exceed the line limit.
6. **Bundle implementation with its tests.** Each PR should include the function/module AND its corresponding tests together — never split tests into a separate PR.
7. If a single logical change exceeds the line limit, split it further

### Planning Order

Split work following this priority:
1. **Infrastructure first** — types, interfaces, configuration, database schemas
2. **Core logic + tests** — business logic, services, utilities, each bundled with their tests
3. **Integration + wiring** — API routes, UI components, glue code that connects the pieces

### Blueprint Detail

For each PR, specify files with operation type and method-level detail:

- `[CREATE]` — full structure: classes, functions with signatures, exports
- `[MODIFY]` — exact changes: add method X to class Y, change return type of Z
- `[DELETE]` — why the file is being removed
- `[RENAME]` — old path and new path

The blueprint must be detailed enough that an executor can implement mechanically without making architectural decisions.

**Insufficient detail:**
```
- `src/api/users.py` [CREATE]: User API routes
```

**Required detail:**
```
- `src/api/users.py` [CREATE]: FastAPI router with endpoints:
  - POST /users — create user, accepts UserCreate body, returns User
  - GET /users/{id} — get user by id, returns User or 404
  - PATCH /users/{id} — update user, accepts UserUpdate body, returns User
  All endpoints use UserService dependency injection.
```

## Step 4: Output the Plan

Output the plan as structured text following this exact format. Do NOT write it to a file — output it directly so the orchestrator captures it.

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
  - `path/to/other.py` [MODIFY]: <exact changes — add method X to class Y>
- Estimated lines: <number>

### PR 2: <title>
...
```

Do NOT begin implementation. Do NOT write any files to disk. The orchestrator handles phase transitions.
