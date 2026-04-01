---
name: ship-plan
description: "This skill should be used as the fallback planner when the ship orchestrator cannot find a project-level planner. Breaks requirements into detailed stacked PR blueprints."
allowed-tools: Read, Write, Glob, Grep
---

# ship-plan — Fallback Planner

Read `ship-state.md` and produce a detailed `ship-plan.md` with PR blueprints.

## Input

Read `ship-state.md` from the repo root for requirements, constraints, and repo context.

Read `.ship.yaml` from the repo root (if exists) for configuration overrides:
- `maxLinesPerPR` (default: 250)
- `maxPRs` (default: 7) — hard cap on total PRs. The orchestrator also passes this value in the prompt.
- `branchPrefix` (default: `"ship/"`)
- `validate` (default: auto-detected, already in ship-state.md)

The orchestrator also provides a list of **available executors** from `.ship.yaml`:
```
Available executors:
- path: ".agents/backend-engineer.md", match: "Python, FastAPI, scrapers, CLI tools"
- path: ".agents/infra-engineer.md", match: "Terraform, Docker, CI/CD"
```

If no executors are provided, set `Executor: default` on all PRs.

## Executor Assignment

For each PR, assign the best-fit executor by matching the PR's work against the `match` descriptions. Selection is based on what the agent **can do**, not its name. A "backend-engineer" agent whose `match` includes "scrapers" is the right pick for scraper work.

If no executor matches a PR's work, set `Executor: default` (uses built-in ship-execute).

## Planning Rules

1. Each PR MUST have less than `maxLinesPerPR` lines changed (hard limit)
2. The total number of PRs MUST NOT exceed `maxPRs` (hard limit, default: 7)
3. Each PR is a complete, working increment — tests pass independently
4. Each PR has a single, clear responsibility
5. **Aim for 3–7 PRs total.** Prefer fewer well-scoped PRs over many tiny ones. Only split further when a single PR would exceed the line limit.
6. **Bundle implementation with its tests.** Each PR should include the function/module AND its corresponding tests together — never split tests into a separate PR.
7. If a single logical change exceeds the line limit, split it further

## Planning Order

Split work following this priority:
1. **Infrastructure first** — types, interfaces, configuration, database schemas
2. **Core logic + tests** — business logic, services, utilities, each bundled with their tests
3. **Integration + wiring** — API routes, UI components, glue code that connects the pieces

## Blueprint Detail

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

## Output

Write `ship-plan.md` to the repo root following the exact schema in `${CLAUDE_PLUGIN_ROOT}/skills/ship/references/contracts.md`.

All PRs start with `Status: pending` and include an `Executor:` field.

Do NOT begin implementation. The orchestrator handles phase transitions.
