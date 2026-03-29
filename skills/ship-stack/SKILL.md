---
name: ship-stack
description: "This skill should be used when the ship orchestrator needs to create or manage Graphite stacked branches during the shipping pipeline."
allowed-tools: Bash(gt:*), Bash(git:*)
---

# ship-stack — Graphite Branch Management

Manage Graphite stacked branches for the ship pipeline.

## Create Branch for a PR

Given a branch name and PR title from the plan:

```bash
gt create <branch-name> -m "feat: <PR title>"
```

If the current branch is not tracked by Graphite, track it first:

```bash
gt track -p main
```

## Verify Stack Health

Before creating a new branch, verify the stack:

```bash
gt ls
```

Ensure the current branch is where expected in the stack. If not, navigate:

```bash
gt checkout <branch-name>
```

## Navigate Stack

Move between branches in the stack:

```bash
gt up          # move to child branch
gt down        # move to parent branch
gt top         # move to top of stack
gt bottom      # move to bottom of stack
```

## After Execution

After the executor commits changes to the current branch, no additional stack operations are needed. The orchestrator will call `ship-stack` again for the next PR, which creates the next branch on top of the current one.

## Important

- Always use `gt` commands, never raw `git branch` or `git checkout` for stack operations
- Branch names come from the plan's Branch field
- The orchestrator calls this skill once per PR, before the executor runs
