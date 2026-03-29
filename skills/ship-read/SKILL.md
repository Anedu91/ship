---
name: ship-read
description: "This skill should be used when the ship orchestrator needs to parse requirements from GitHub issues, markdown files, or inline text into a structured ship-state.md file."
---

# ship-read — Parse Requirements

Parse `$ARGUMENTS` into a structured `ship-state.md` file in the repo root.

## Step 1: Detect Input Type

Determine the input type from `$ARGUMENTS`:

- **GitHub issue URL** (contains `github.com` or matches `#<number>` or `<org>/<repo>#<number>`):
  Fetch with `gh issue view <number> --json title,body,comments,labels`

- **File path** (path exists on disk):
  Read the file completely.

- **Inline text** (anything else):
  Use the text directly as the requirements.

## Step 2: Extract Requirements

From the raw input, identify:

1. **Goal** — what needs to be built (one paragraph)
2. **Constraints** — tech stack, patterns, conventions mentioned or implied
3. **Acceptance criteria** — how to know it's done. If not explicit, infer from the goal.

## Step 3: Scan Repo Context

Read the target repo to populate context:

1. **Tech stack** — check for `pyproject.toml`, `package.json`, `Cargo.toml`, `go.mod`, `Makefile`
2. **Validation commands** — if `.ship.yaml` has `validate`, use those. Otherwise auto-detect:
   - `pyproject.toml` → `uv run ruff check . && uv run ruff format --check . && uv run mypy src/ && uv run pytest`
   - `package.json` → `npm run lint && npm run typecheck && npm test`
   - `Makefile` → `make check` or `make test`
3. **Project structure** — list key directories and their purpose relevant to the goal
4. **Relevant files** — files likely to be touched based on the requirements

## Step 4: Write ship-state.md

Write the structured output to `ship-state.md` in the repo root following the schema in `${CLAUDE_PLUGIN_ROOT}/skills/ship/references/contracts.md`.

Do NOT proceed to planning. The orchestrator handles phase transitions.
