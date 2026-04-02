---
name: ship-push
description: "This skill should be used when the ship orchestrator needs to push the completed PR stack to GitHub and create PR descriptions."
allowed-tools: Read, Write, Bash(gt:*), Bash(gh:*), Bash(cat:*), Bash(rm:*)
---

# ship-push — Push Stack & Create PRs

Push the completed stack to GitHub and create PR descriptions.

## Input

The orchestrator provides everything inline in the prompt:
- **Feature summary** — one paragraph describing the full feature
- **PR list** — for each PR: title, branch name, description, file operations summary, estimated lines, validation result (pass/fail)
- **Total PR count**

You do NOT need to read any state files. Everything you need is in the prompt.

## Step 1: Submit Stack

Push the entire stack at once:

```bash
gt submit --no-interactive --publish
```

If `gt submit` fails, retry with `--force`:

```bash
gt submit --no-interactive --publish --force
```

After submission, mark any draft PRs as ready:

```bash
gh pr ready <number>
```

## Step 2: Get PR Numbers

For each branch in the stack, get its PR number:

```bash
gh pr list --head <branch-name> --json number --jq '.[0].number'
```

## Step 3: Create PR Descriptions

For each PR, generate a description using this template:

```markdown
## What
<PR description from the prompt>

## Why
<How this PR contributes to the overall feature, referencing the feature summary>

## How
<Implementation approach — summarize the file operations>

## Changes
- `<file>`: <what changed and why>

## Validation
- [x/fail] <validation command 1>
- [x/fail] <validation command 2>

## Stack
PR <N>/<total> for: <feature summary>
```

Write each description to a temp file and update the PR:

```bash
gh pr edit <number> --body-file /tmp/ship-pr-<N>.md
```

Clean up temp files after.

## Step 4: Return Results

Output the list of PRs with their URLs:

```
1. <PR title> — <URL> (<lines> lines) [<status>]
2. <PR title> — <URL> (<lines> lines) [<status>]
```

The orchestrator uses this output for the final report.
