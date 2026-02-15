# plugin-analyzer

Analyze, score, and compare installed Claude Code plugins. Decompose plugins into capabilities, determine per-form winners, track context budget, and auto-generate override rules.

## How It Works

1. **You install plugins manually** (using `claude plugin add <url>`)
2. **Run `/plugin-analyzer:init`** — scans all installed plugins, scores every capability
3. **Capabilities are compared within the same form** — skill vs skill, agent vs agent (never skill vs agent)
4. **Winners are determined** — highest score per form wins
5. **Overrides are generated** — CLAUDE.md rules to suppress losing capabilities

## Commands

| Command | What it does |
|---------|-------------|
| `/plugin-analyzer:init` | Scan ALL installed plugins, build/rebuild entire registry from scratch |
| `/plugin-analyzer:update` | Detect new/removed plugins, analyze only changes |
| `/plugin-analyzer:status` | Show full registry — plugins, winners, context budget |

## Scoring Algorithm

Each capability is scored on 4 dimensions (0-10), weighted average:

| Dimension | Weight | What |
|-----------|--------|------|
| Implementation Quality | 35% | Prompt clarity, structure, edge cases |
| Context Efficiency | 25% | Character count vs value delivered |
| Utility | 25% | How frequently needed in daily work |
| Plugin Completeness | 15% | Component variety, README, docs, structure (local-only) |

**Formula**: `Score = (Quality x 0.35) + (Context x 0.25) + (Utility x 0.25) + (Completeness x 0.15)`

## Form-Based Comparison

Different forms serve different purposes and are complementary:

| Form | Purpose | Competes With |
|------|---------|---------------|
| Skill | Knowledge injection | Other skills |
| Agent | Autonomous execution | Other agents |
| Command | Explicit user action | Other commands |
| Hook | Automatic guardrail | Other hooks |

A Code Review **skill** and a Code Review **agent** can both be winners — they never compete.

## Registry Structure

```
registry/
  plugins.json           # All tracked plugins with profiles
  capabilities.md        # Winners by category + form + aliases + context
  plugins/
    plugin-name.md       # Per-plugin detailed analysis
```

## Installation

```bash
claude plugin add /path/to/plugin-analyzer
```

## Quick Start

```bash
# 1. Install the plugin
claude plugin add /path/to/plugin-analyzer

# 2. Restart Claude Code

# 3. Run init to analyze all your installed plugins
/plugin-analyzer:init

# 4. View the results
/plugin-analyzer:status
```
