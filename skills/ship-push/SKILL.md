---
name: ship-push
description: "This skill should be used when the ship orchestrator needs to push the completed PR stack to GitHub and create PR descriptions."
---

# ship-push — Push Stack & Create PRs

Push the completed stack to GitHub and ensure each PR has a detailed description.

## Step 1: Submit Stack

Push the entire stack at once:

```bash
gt submit --no-interactive
```

If `gt submit` fails, try with `--force`:

```bash
gt submit --no-interactive --force
```

## Step 2: Get PR Numbers

After submission, get the PR numbers for each branch:

```bash
gt ls
```

For each branch in the stack, get its PR number:

```bash
gh pr list --head <branch-name> --json number --jq '.[0].number'
```

## Step 3: Create PR Descriptions

For each PR in `ship-plan.md`, generate a description using this template:

```markdown
## What
<PR description from the plan>

## Why
<How this PR contributes to the overall feature, referencing the plan's Summary>

## How
<Implementation approach — summarize the file operations from the blueprint>

## Changes
- `<file>`: <what changed and why>
- `<file>`: <what changed and why>

## Validation
- [x/fail] <validation command 1>
- [x/fail] <validation command 2>

## Stack
PR <N>/<total> for: <plan Summary>
```

Update each PR:

```bash
gh pr edit <number> --body-file /tmp/ship-pr-<N>.md
```

## Step 4: Return Results

Output the list of PRs with their URLs:

```
1. <PR title> — <URL> (<lines> lines) [<status>]
2. <PR title> — <URL> (<lines> lines) [<status>]
```

The orchestrator uses this output for the final report.
