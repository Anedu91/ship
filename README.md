# Ship 🚀

Autonomous shipping pipeline for AI coding agents. One command: requirements in, stacked PRs out.

## What it does

```
/ship <requirements>
   │
   ├── 1. Read & understand requirements
   ├── 2. Plan stacked PRs (<250 lines each)
   ├── 3. Execute each PR with coding agent
   ├── 4. Validate (ruff, mypy, tests)
   ├── 5. Push stacked PRs to GitHub
   └── 6. Report summary
```

You give it requirements. It gives you a stack of reviewed, validated, well-documented PRs.

## Supported tools

| Tool | Integration | How |
|------|-------------|-----|
| **Claude Code** | Plugin | `/ship:ship <requirements>` |
| **OpenClaw** | Skill | `/ship <requirements>` or conversational |
| **OpenCode** | Planned | — |
| **Codex** | Planned | — |

## Install

### Claude Code

```bash
# Load as local plugin
claude --plugin-dir /path/to/ship

# Or add to .claude/settings.json
{
  "plugins": ["/path/to/ship"]
}
```

### OpenClaw

Copy `openclaw/ship/` to your workspace `skills/` directory.

## Requirements

- **Graphite CLI** (`gt`) — for stacked PR management
- **GitHub CLI** (`gh`) — for PR creation and issue fetching
- **Project tooling** — ruff, mypy, pytest (Python) or equivalent for your stack

## How it works

### Planning

Ship breaks work into PRs of **<250 lines each**. Each PR:
- Is a complete, working increment
- Passes all validation independently
- Builds on the previous PR in the stack
- Has a single, clear responsibility

### Validation

Before pushing, each PR is validated against the project's toolchain:
- **Python**: `ruff check` → `ruff format --check` → `mypy` → `pytest`
- **Node**: `lint` → `typecheck` → `test`
- Falls back to whatever linter/test config exists

### PR descriptions

Every PR includes:
- What changed and why
- Implementation approach
- File-by-file change summary
- Validation status
- Stack position (PR X/N)

## Configuration

Create `.ship.yaml` in your repo root (optional):

```yaml
# Max lines per PR (default: 250)
maxLinesPerPR: 250

# Validation commands (auto-detected if not specified)
validate:
  - "uv run ruff check ."
  - "uv run ruff format --check ."
  - "uv run mypy src/"
  - "uv run pytest"

# Branch prefix for ship branches
branchPrefix: "ship/"
```

## License

MIT
