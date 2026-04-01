---
name: ship-execute
description: "This skill should be used as the fallback executor when the ship orchestrator cannot find a project-level executor. Implements a single PR from a ship blueprint mechanically."
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(gt:*), Bash(git:*), Bash(uv:*), Bash(npm:*), Bash(make:*), Bash(pytest:*), Bash(ruff:*), Bash(mypy:*)
---

# ship-execute — Fallback Executor

Implement a single PR from its blueprint in `ship-plan.md`.

## Input

The orchestrator provides the PR number, title, and branch name. Read `ship-plan.md` from the repo root for the full PR blueprint (under `### PR N: <title>`) and the Configuration section (validation commands).

## Step 0: Create Branch

Before implementing, create the Graphite branch for this PR:

```bash
gt create <branch-name> -m "feat: <PR title>"
```

If the current branch is not tracked by Graphite, track it first: `gt track -p main`.

## Implementation Process

1. **Read the blueprint** — read `ship-plan.md` and find the PR section. Understand every file operation specified
2. **Read existing files** — for `[MODIFY]` operations, read the current file first to understand context
3. **Implement each file operation**:
   - `[CREATE]` — create the file with the structure specified in the blueprint
   - `[MODIFY]` — make exactly the changes described. Do not refactor or "improve" surrounding code
   - `[DELETE]` — remove the file
   - `[RENAME]` — move the file to its new path
4. **Match existing patterns** — follow the codebase's naming, style, and conventions

## Implementation Rules

- Implement exactly what the blueprint specifies. Do not add features, refactor unrelated code, or make "improvements" beyond the blueprint.
- If the blueprint is insufficient to implement (missing critical details), document what's missing in the PR status and continue with best judgment.
- Do not modify files outside the blueprint unless strictly necessary for the code to compile/run.

## Validation

After implementation, run the validation commands from the Configuration section:

```
For each validation command:
  Run the command
  If it fails:
    Read the error output
    Fix the issue
    Re-run validation
  Repeat up to 3 times total
```

If validation still fails after 3 attempts, document the failures and continue.

## Commit

After validation passes (or after 3 failed attempts):

```bash
git add -A
git commit --amend -m "feat: <PR title>"
```

## Update Status

Update the PR's Status field in `ship-plan.md`:
- If validation passed: `Status: done`
- If validation failed after retries: `Status: failed:<brief reason>`

Do NOT proceed to the next PR. The orchestrator handles sequencing.
