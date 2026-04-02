---
name: ship-execute
description: "This skill should be used as the fallback executor when the ship orchestrator cannot find a project-level executor. Implements a single PR from an inline blueprint."
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(gt:*), Bash(git:*), Bash(uv:*), Bash(npm:*), Bash(make:*), Bash(pytest:*), Bash(ruff:*), Bash(mypy:*)
---

# ship-execute — Fallback Executor

Implement a single PR from the blueprint provided in the prompt.

## Input

The orchestrator provides everything inline in the prompt:
- **Branch name** and **PR title**
- **Description** of what this PR does
- **File operations** with method-level detail
- **Validation commands** to run after implementation

You do NOT need to read any state files. Everything you need is in the prompt.

## Step 0: Create Branch

Before implementing, create the Graphite branch for this PR:

```bash
gt create <branch-name> -m "feat: <PR title>"
```

If the current branch is not tracked by Graphite, track it first: `gt track -p main`.

## Implementation Process

1. **Understand the blueprint** — read the file operations provided in the prompt
2. **Read existing files** — for `[MODIFY]` operations, read the current file first to understand context
3. **Implement each file operation**:
   - `[CREATE]` — create the file with the structure specified
   - `[MODIFY]` — make exactly the changes described. Do not refactor or "improve" surrounding code
   - `[DELETE]` — remove the file
   - `[RENAME]` — move the file to its new path
4. **Match existing patterns** — follow the codebase's naming, style, and conventions

## Implementation Rules

- Implement exactly what the blueprint specifies. Do not add features, refactor unrelated code, or make "improvements" beyond the blueprint.
- If the blueprint is insufficient to implement (missing critical details), use best judgment and document assumptions.
- Do not modify files outside the blueprint unless strictly necessary for the code to compile/run.

## Validation

After implementation, run the validation commands from the prompt:

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

## Result

Output whether the PR succeeded or failed:
- If validation passed: report success
- If validation failed after retries: report failure with brief reason

Do NOT proceed to the next PR. The orchestrator handles sequencing.
