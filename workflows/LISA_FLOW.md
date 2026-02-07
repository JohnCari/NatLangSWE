# Lisa Flow

> Autonomous AI development with structured specs and self-healing tests

**Repository:** [github.com/JohnCari/lisa-flow](https://github.com/JohnCari/lisa-flow)

---

## Overview

Lisa Flow combines [GitHub Spec Kit](https://github.com/github/spec-kit)'s structured phases with [Geoffrey Huntley's Ralph Loop](https://ghuntley.com/loop/) philosophy. Give it a feature description — it specs, plans, implements, and polishes — then runs a self-healing test loop until everything passes.

---

## Setup

1. **Claude Code** — Install natively (not via npm)
2. **Spec Kit** — Install from [github/spec-kit](https://github.com/github/spec-kit)
3. **`/frontend-design` skill** — Install from the Anthropic Marketplace
4. **Context7 MCP** — Install from [Upstash](https://github.com/upstash/context7)
5. **Agent Teams** (recommended, optional) — Parallelizes PLAN, IMPLEMENT, and TEST phases

### Enabling Agent Teams

Agent Teams is experimental and disabled by default. Enable via `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Or export directly:

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

Restart Claude Code after enabling.

---

## Usage

```bash
lisa-flow/lisa-flow.sh <feature> [retries]
```

| Parameter | Description |
|-----------|-------------|
| `"feature text"` | Describe feature directly as a string |
| `@path/to/file` | Read feature description from a file |
| `[retries]` | Optional number of test iterations (default: 5) |

### Examples

```bash
lisa-flow/lisa-flow.sh "Build auth API"
lisa-flow/lisa-flow.sh @path/to/spec.md 10
```

Tests are automatically included — no need to specify "with tests".

---

## Workflow

```
SPECIFY → PLAN → TASKS → IMPLEMENT → TEST
```

| Phase | Description | Agent Teams |
|-------|-------------|-------------|
| **Specify** | Creates comprehensive spec including TDD tests | — |
| **Plan** | Technical implementation plan — parallelizes research across APIs, architecture, dependencies | Parallel |
| **Tasks** | Generates structured task list from plan | — |
| **Implement** | Builds the feature — spawns teammates for independent tasks, uses `/frontend-design` for UI | Parallel |
| **Test** | Self-healing loop with parallel checks — repeats until all pass or max retries | Parallel |

### Test Phase — Parallel Checks

The test phase runs up to `[retries]` iterations, each spawning parallel teams for:

| Check | Focus |
|-------|-------|
| **Functional** | Run all tests, fix failures (never modify test expectations) |
| **Code quality** | Fix bugs, dead code, magic numbers |
| **Security** | OWASP vulnerability checks |
| **Performance** | Optimization passes |

Exits when all checks pass or max retries is reached.

### Context7 Integration

Context7 MCP is automatically injected into PLAN and IMPLEMENT prompts, ensuring accurate library documentation:

1. `mcp__context7__resolve-library-id` — Resolve library name to ID
2. `mcp__context7__query-docs` — Query docs with the ID and a specific question

---

## Integration with Feature-Sliced Hexagonal

Lisa Flow is the **inner workflow** each terminal runs within [Feature-Sliced Hexagonal](../patterns/FEATURE_SLICED_HEXAGONAL.md):

```
1. IDEATION     → Describe idea
2. ANALYSIS     → Create skeleton if needed
3. BREAKDOWN    → Split into features, assign terminals
4. PARALLEL     → Each terminal runs lisa-flow.sh
                   └── SPECIFY → PLAN → TASKS → IMPLEMENT → TEST
5. INTEGRATION  → Merge all features
```

- Feature-Sliced Hexagonal prevents conflicts (file ownership)
- Lisa Flow gives each terminal a structured TDD workflow
- Agent Teams parallelize work within each terminal

---

## Logging

- Logs written to `logs/lisa-flow_YYYY-MM-DD_HH-MM-SS.log`
- All phase output captured with timing
- Auto-cleanup of logs older than 7 days
- Summary displayed at end with per-phase timings and overall status

---

## Inspiration

| Source | Contribution |
|--------|--------------|
| [Ralph Loop](https://ghuntley.com/loop/) | Self-healing test loop philosophy |
| [GitHub Spec Kit](https://github.com/github/spec-kit) | Structured spec → plan → tasks → implement phases |
| [Context7 MCP](https://github.com/upstash/context7) | Accurate library documentation during development |

---

## License

CC0 — Public Domain
