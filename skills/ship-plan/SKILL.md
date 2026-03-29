---
name: ship-plan
description: "This skill should be used as the fallback planner when the ship orchestrator cannot find a project-level planner. Breaks requirements into detailed stacked PR blueprints."
---

# ship-plan — Fallback Planner

Read `ship-state.md` and produce a detailed `ship-plan.md` with PR blueprints.

## Input

Read `ship-state.md` from the repo root for requirements, constraints, and repo context.

Read `.ship.yaml` from the repo root (if exists) for configuration overrides:
- `maxLinesPerPR` (default: 250)
- `branchPrefix` (default: `"ship/"`)
- `validate` (default: auto-detected, already in ship-state.md)

## Planning Rules

1. Each PR MUST have less than `maxLinesPerPR` lines changed (hard limit)
2. Each PR is a complete, working increment — tests pass independently
3. Each PR has a single, clear responsibility
4. More small PRs are preferred over fewer large ones
5. If a single logical change exceeds the line limit, split it further

## Planning Order

Split work following this priority:
1. **Infrastructure first** — types, interfaces, configuration, database schemas
2. **Core logic** — business logic, services, utilities
3. **Integration** — API routes, UI components, wiring
4. **Tests** — test files, test utilities (unless tests are small enough to include with their PR)

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

All PRs start with `Status: pending`.

Do NOT begin implementation. The orchestrator handles phase transitions.
