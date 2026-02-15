---
description: Display the current status of the plugin registry — all tracked plugins, per-form winners, and context budget
allowed-tools: ["Read", "Glob"]
---

# Plugin Registry Status

Present a comprehensive read-only overview of the plugin analyzer registry.

## Execution Steps

### 1. Read Registry Files

Read from `${CLAUDE_PLUGIN_ROOT}/registry/`:
- `plugins.json` — list of all tracked plugins with profiles
- `capabilities.md` — category-wise capabilities with per-form winners

Also read `~/.claude/CLAUDE.md` and extract the managed section between `<!-- BEGIN PLUGIN-ANALYZER OVERRIDES -->` and `<!-- END PLUGIN-ANALYZER OVERRIDES -->` markers for active override rules.

### 2. Present Summary Dashboard

```
PLUGIN ANALYZER — STATUS
=========================

Plugins Tracked: X
Total Capabilities: Y
Winners: W | Runner-ups: R | Ignored: I

Context Budget:
  Always-on:  X,XXX chars (loaded every session)
  On-demand:  X,XXX chars (loaded when triggered)
```

### 3. Present Plugin Table

List all tracked plugins with profiles:

```
TRACKED PLUGINS
| Plugin | Source | Capabilities | Win Rate | Avg Score | Always-on | On-demand |
|--------|--------|-------------|----------|-----------|-----------|-----------|
```

Highlight plugins with 0% win rate (all capabilities outclassed — consider uninstalling).

### 4. Present Winner Board

For each category, show winners grouped by form:

```
WINNER BOARD

Code Quality:
  Skills:
    Code Review ............. superpowers:requesting-code-review (6.2)
  Agents:
    Code Review ............. pr-review-toolkit:code-reviewer (8.7)
    Test Coverage ........... pr-review-toolkit:pr-test-analyzer (8.1)

Workflow:
  Commands:
    Git Commit .............. commit-commands:commit (8.5)
    Push & PR ............... commit-commands:commit-push-pr (8.0)
```

### 5. Present Contested Capabilities

Show capabilities where multiple same-form entries compete:

```
CONTESTED CAPABILITIES
| Capability | Form | Winner (Score) | Runner-up (Score) | Ignored |
|-----------|------|----------------|-------------------|---------|
```

### 6. Present Active Overrides

Read and display override rules from the managed section in `~/.claude/CLAUDE.md`.

### 7. Empty State

If the registry is empty (no plugins tracked), display:
```
No plugins tracked yet. Run /plugin-analyzer:init to analyze your installed plugins.
```
