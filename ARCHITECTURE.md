# Architecture — claude-code-plugin-analyzer

## System Overview

claude-code-plugin-analyzer is a Claude Code plugin that scans installed plugins, decomposes them into capabilities, scores each capability, and maintains a registry of per-form winners. It operates as a **read-only analyzer** of the plugin cache — it never modifies installed plugins.

```
┌─────────────────────────────────────────────────────────────┐
│                      USER WORKFLOW                           │
│                                                             │
│  Install plugins manually → /plugin-analyzer:init or :update│
│                           → /plugin-analyzer:status          │
└──────────┬─────────────────────────────┬────────────────────┘
           │                             │
           ▼                             ▼
┌────────────────────┐       ┌────────────────────┐
│  PLUGIN SCANNER    │       │  REGISTRY READER   │
│  (init / update)   │       │  (status)          │
│                    │       │                    │
│  Reads from:       │       │  Reads from:       │
│  ~/.claude/        │       │  registry/         │
│  plugins/cache/    │       │  CLAUDE.md          │
└────────┬───────────┘       └────────────────────┘
         │
         ▼
┌────────────────────┐
│  SCORING ENGINE    │
│  (analyze-plugin   │
│   skill)           │
│                    │
│  4-dimension       │
│  weighted model    │
│  Form-based        │
│  comparison        │
└────────┬───────────┘
         │
         ▼
┌───────────────────────────────────────────────────┐
│                REGISTRY (output)                   │
│                                                   │
│  plugins.json         Plugin metadata + profiles  │
│  plugins/*.md         Per-plugin detailed analysis│
│  capabilities.md      Winners by category + form  │
│                       + aliases + context summary  │
│                                                   │
│  ~/.claude/CLAUDE.md  Override rules injected     │
│                       into managed section         │
└───────────────────────────────────────────────────┘
```

## Plugin Structure

```
claude-code-plugin-analyzer/
├── .claude-plugin/
│   └── plugin.json                     # Plugin manifest
│
├── commands/
│   ├── init.md                         # /plugin-analyzer:init
│   ├── update.md                       # /plugin-analyzer:update
│   └── status.md                       # /plugin-analyzer:status
│
├── skills/
│   └── analyze-plugin/
│       ├── SKILL.md                    # Scoring methodology
│       └── references/
│           └── scoring-algorithm.md    # Full scoring rubric
│
├── registry/                           # Generated data (committed to git)
│   ├── plugins.json                    # All tracked plugins with profiles
│   ├── capabilities.md                 # Winners by category + form + aliases
│   └── plugins/                        # Per-plugin analysis files
│       ├── pr-review-toolkit.md
│       ├── superpowers.md
│       └── ...
│
├── CLAUDE.md                              # Auto-generated ignore rules (loaded by Claude Code)
├── REQUIREMENTS.md
├── ARCHITECTURE.md
├── README.md
└── .gitignore
```

### File Responsibilities

| File | Role | Written By | Read By |
|------|------|-----------|---------|
| `.claude-plugin/plugin.json` | Plugin identity | Manual | Claude Code |
| `commands/*.md` | Action instructions for Claude | Manual | Claude (on command invoke) |
| `skills/analyze-plugin/SKILL.md` | Scoring methodology knowledge | Manual | Claude (during analysis) |
| `skills/analyze-plugin/references/scoring-algorithm.md` | Detailed scoring rubric | Manual | Claude (when deep detail needed) |
| `registry/plugins.json` | Master plugin list + profiles | init/update | update, status |
| `registry/capabilities.md` | Capability winners + aliases + context | init/update | status, Claude (alias matching) |
| `registry/plugins/*.md` | Per-plugin detailed analysis | init/update | status, user review |
| `CLAUDE.md` (plugin root) | Plugin description + override injection docs | Manual | Claude Code (every session) |
| `~/.claude/CLAUDE.md` | Override rules injected into managed section | init/update | Claude Code (every session) |

## Data Models

### plugins.json

Tracks all analyzed plugins with their profiles.

```json
{
  "version": "1.0.0",
  "last_updated": "2026-02-15",
  "plugins": [
    {
      "name": "pr-review-toolkit",
      "source": "claude-plugins-official",
      "installed_path": "~/.claude/plugins/cache/claude-plugins-official/pr-review-toolkit/abc123",
      "version": "1.0.0",
      "analyzed_date": "2026-02-15",
      "total_capabilities": 6,
      "winners": 5,
      "win_rate": 0.83,
      "avg_score": 7.9,
      "best_score": 8.7,
      "always_on_chars": 450,
      "on_demand_chars": 0
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| name | string | Plugin name from plugin.json |
| source | string | Parent directory name (marketplace name or "standalone") |
| installed_path | string | Absolute path to plugin's installed location |
| version | string | Plugin version from plugin.json |
| analyzed_date | string | ISO date when last analyzed |
| total_capabilities | number | Count of all scored capabilities |
| winners | number | Count of capabilities with WINNER status (per-form) |
| win_rate | number | winners / total_capabilities (0.0 - 1.0) |
| avg_score | number | Mean score across all capabilities |
| best_score | number | Highest individual capability score |
| always_on_chars | number | Total chars from always-on components |
| on_demand_chars | number | Total chars from on-demand components |

### capabilities.md

Single file combining: capability registry, canonical name aliases, context budget summary, and per-form winner tables.

```markdown
# Capability Registry

> Plugins: 6 | Capabilities: 22
> Winners: 18 | Runner-ups: 3 | Ignored: 1
> Context: 1,200 always-on / 12,400 on-demand

---

## Code Quality

### Code Review
> aliases: code-reviewer, review-code, pr-review, requesting-code-review

#### Skills
| Plugin | Score | Context (chars) | Status |
|--------|-------|-----------------|--------|
| superpowers | 6.2 | 1,800 | WINNER |

**Active skill**: superpowers:requesting-code-review

#### Agents
| Plugin | Score | Context (chars) | Status |
|--------|-------|-----------------|--------|
| pr-review-toolkit | 8.7 | — | WINNER |
| another-plugin | 7.2 | — | RUNNER-UP |

**Active agent**: pr-review-toolkit:code-reviewer
**Why**: Multi-pass review with confidence-based filtering

---

## Workflow

### Git Commit
> aliases: commit, create-commit, smart-commit

#### Commands
| Plugin | Score | Context (chars) | Status |
|--------|-------|-----------------|--------|
| commit-commands | 8.5 | 1,200 | WINNER |

**Active command**: commit-commands:commit (sole entry)
```

**Structure rules**:
- Header: global summary (plugin count, capability count, context totals)
- Level 2 heading (##): Category
- Level 3 heading (###): Capability name (canonical)
- Blockquote after capability name: aliases for dedup matching
- Level 4 heading (####): Form type (Skills, Agents, Commands, Hooks)
- Table under each form: same-form entries compared
- Only show form subsections that have entries (skip empty forms)

### Per-Plugin Analysis (registry/plugins/<name>.md)

```markdown
# Plugin: pr-review-toolkit

> Source: claude-plugins-official
> Installed: ~/.claude/plugins/cache/claude-plugins-official/pr-review-toolkit/abc123
> Version: 1.0.0
> Analyzed: 2026-02-15
> Plugin Completeness: 9/10

## Plugin Profile

| Metric | Value |
|--------|-------|
| Total Capabilities | 6 |
| Winners | 5 |
| Win Rate | 83% |
| Avg Score | 7.9 |
| Best Score | 8.7 |
| Always-on Context | 450 chars |
| On-demand Context | 0 chars |

## Capabilities

| # | Capability | Category | Form | Score | Context (chars) | Load | Status |
|---|-----------|----------|------|-------|-----------------|------|--------|
| 1 | Code Review | Code Quality | agent | 8.7 | — | — | WINNER |
| 2 | Test Coverage | Code Quality | agent | 8.1 | — | — | WINNER |
| 3 | Type Analysis | Code Quality | agent | 7.5 | — | — | WINNER |
| 4 | Silent Failures | Code Quality | agent | 7.8 | — | — | WINNER |
| 5 | Comment Analysis | Code Quality | agent | 7.2 | — | — | WINNER |
| 6 | Code Simplification | Code Quality | agent | 6.5 | — | — | RUNNER-UP |

## Scoring Breakdown

### 1. Code Review (8.7)
- Quality: 9 — Multi-pass review, confidence-based filtering
- Context: 7 — Agent (default neutral, separate context)
- Utility: 10 — Daily use, every PR
- Completeness: 9 — Multiple component types, README, reference docs, clear structure
```

### Override Injection into ~/.claude/CLAUDE.md

Overrides are injected into the user's global `~/.claude/CLAUDE.md` within a managed section:

```markdown
# ... user's existing content (untouched) ...

<!-- BEGIN PLUGIN-ANALYZER OVERRIDES -->
## Plugin Overrides — Managed by claude-code-plugin-analyzer

> Auto-generated by /plugin-analyzer:init or /plugin-analyzer:update.
> Do not edit manually — changes will be overwritten.

### Code Quality — Agents
- Do NOT use `code-reviewer` agent from `another-plugin`. Use `pr-review-toolkit:code-reviewer` agent instead.

### Workflow — Commands
(no overrides)
<!-- END PLUGIN-ANALYZER OVERRIDES -->
```

**Injection rules**:
1. Read `~/.claude/CLAUDE.md` (create if it doesn't exist)
2. Find `<!-- BEGIN PLUGIN-ANALYZER OVERRIDES -->` and `<!-- END PLUGIN-ANALYZER OVERRIDES -->` markers
3. If markers exist → replace everything between them
4. If markers don't exist → append the full block (markers + content) at the end of the file
5. Never modify any content outside the markers
6. Override rules are grouped by **category + form**, only for IGNORE-status entries

## Command Architecture

### /plugin-analyzer:init — Full Rebuild

```
Allowed tools: Read, Write, Edit, Bash, Glob, Grep

Flow:
┌──────────────────┐
│ 1. DISCOVER      │  Glob: ~/.claude/plugins/cache/**/.claude-plugin/plugin.json
│    Find all      │  Filter: skip claude-code-plugin-analyzer itself
│    installed     │  Output: list of { name, source, path }
│    plugins       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. CLEAR         │  Write: reset plugins.json to empty
│    Reset         │  Bash: rm registry/plugins/*.md (keep .gitkeep)
│    registry      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 3. ANALYZE       │  For each plugin:
│    Score each    │    Read: plugin.json, skills, agents, commands, hooks, mcp
│    plugin        │    Apply: 4-dimension scoring (skill knowledge)
│                  │    Classify: category + form for each capability
│                  │    Match: aliases in capabilities.md
│                  │    Write: registry/plugins/<name>.md
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. BUILD         │  Aggregate all capabilities across all plugins
│    capabilities  │  Group by: category → capability name → form
│    .md           │  Compare within same form → WINNER/RUNNER/IGNORE
│                  │  Include: aliases, context summary header
│                  │  Write: registry/capabilities.md
│                  │  Write: registry/plugins.json (all entries)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. INJECT        │  Read: capabilities.md (IGNORE entries)
│    Overrides     │  Read: ~/.claude/CLAUDE.md (existing content)
│                  │  Inject into managed section (marker-based)
│                  │  Group rules by category + form
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 6. REPORT        │  Output: summary dashboard to user
└──────────────────┘
```

### /plugin-analyzer:update — Incremental

```
Allowed tools: Read, Write, Edit, Bash, Glob, Grep

Flow:
┌──────────────────┐
│ 1. DISCOVER      │  Same as init step 1
│    + DIFF        │  Read: plugins.json (existing tracked list)
│                  │  Compute: NEW / REMOVED / EXISTING
└────────┬─────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌──────────┐
│ NEW    │ │ REMOVED  │  EXISTING → skip (scores pinned)
│ Analyze│ │ Clean up │
│ & add  │ │ & remove │
└────┬───┘ └────┬─────┘
     │          │
     └────┬─────┘
          │
          ▼
┌──────────────────┐
│ REBUILD          │  Must fully rebuild capabilities.md because
│ capabilities.md  │  removals can change winners
│ + inject         │  Uses PINNED scores for existing plugins
│ overrides        │  (only new plugins get fresh scores)
│                  │  Re-inject overrides into ~/.claude/CLAUDE.md
│                  │  Same as init steps 4-5
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ REPORT           │  "2 new, 1 removed, 8 unchanged"
│                  │  Show any winner changes
└──────────────────┘
```

### /plugin-analyzer:status — Read Only

```
Allowed tools: Read, Glob (read-only only)

Flow:
┌──────────────────┐
│ READ             │  Read: plugins.json, capabilities.md,
│ all registry     │        ~/.claude/CLAUDE.md (managed section)
│ files            │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ FORMAT           │  Summary dashboard
│ & PRESENT        │  Plugin table with profiles (win rate)
│                  │  Winner board by category + form
│                  │  Contested capabilities
│                  │  Active overrides
└──────────────────┘
```

## Plugin Discovery Algorithm

Finding plugins in `~/.claude/plugins/cache/` requires handling nested directories:

```
~/.claude/plugins/cache/
├── claude-plugins-official/          # Marketplace (multiple plugins)
│   ├── pr-review-toolkit/
│   │   └── abc123/                   # Hash directory
│   │       └── .claude-plugin/
│   │           └── plugin.json       ← FOUND
│   ├── commit-commands/
│   │   └── def456/
│   │       └── .claude-plugin/
│   │           └── plugin.json       ← FOUND
│   └── ...
├── standalone-plugin/                # Standalone
│   └── ghi789/
│       └── .claude-plugin/
│           └── plugin.json           ← FOUND
└── ...
```

**Discovery steps**:
1. Glob for `~/.claude/plugins/cache/**/.claude-plugin/plugin.json`
2. For each found `plugin.json`, the plugin root is the parent of `.claude-plugin/`
3. Extract source name: the directory directly under `plugins/cache/`
4. Extract plugin name: from `plugin.json` `name` field
5. Filter out: `claude-code-plugin-analyzer` (self)

**Output**: List of `{ name, source, path }` tuples for all installed plugins.

## Scoring Engine

The scoring engine is the `analyze-plugin` **skill**. Commands (init, update) trigger analysis; the skill provides the methodology Claude follows.

### Scoring Flow per Plugin

```
Plugin path
    │
    ├─→ Read plugin.json → name, version, source
    │
    ├─→ Glob skills/*/SKILL.md → list of skills
    │   └─→ For each: read, count chars, score, form = "skill"
    │
    ├─→ Glob agents/*.md → list of agents
    │   └─→ For each: read, score (context = 7 default), form = "agent"
    │
    ├─→ Glob commands/*.md → list of commands
    │   └─→ For each: read, count chars, score, form = "command"
    │
    ├─→ Read hooks/hooks.json → list of hooks
    │   └─→ For each: if prompt-based, count prompt chars, score, form = "hook"
    │
    ├─→ Read .mcp.json → list of MCP servers
    │   └─→ For each: note server, minimal scoring, form = "mcp"
    │
    └─→ Aggregate:
        ├─→ Match each to canonical name via aliases in capabilities.md
        ├─→ If alias match found → use EXISTING category (sticky)
        ├─→ If no alias match → classify into category, create new alias entry
        ├─→ Compare within same form only (deterministic tiebreak)
        ├─→ Compute plugin profile (win rate, totals)
        └─→ Write per-plugin analysis file
```

### Form-Based Comparison Logic

```
For each capability name (e.g., "Code Review"):
    For each form (skill, agent, command, hook):
        Collect all entries of this form
        If count == 1:
            Mark as WINNER (sole entry)
        If count > 1:
            Sort by score descending
            Highest → WINNER
            Within 0.5 of winner → TIEBREAK:
                1. Higher Context Efficiency score wins
                2. If still tied: alphabetically earlier plugin name wins
            Within 2.0 of winner and above 5.0 → RUNNER-UP
            Below 5.0 or more than 2.0 behind → IGNORE
```

Cross-form entries never compete. A capability can have multiple WINNERs across different forms (e.g., one winning skill AND one winning agent for "Code Review").

## Key Design Decisions

### D1: Capability-Centric, Not Plugin-Centric
Plugins are containers. Capabilities are what compete. A plugin's value is derived from how many of its capabilities win.

### D2: Form-Based Comparison Only
Skills compete with skills. Agents compete with agents. Different forms serve different purposes (knowledge injection vs autonomous execution vs guardrails vs explicit actions) and are complementary, not competing.

### D3: Automated Scoring, No User Approval
Scoring is fully automated. Users influence outcomes by running init/update and reviewing status.

### D4: Single capabilities.md File
Aliases (for dedup), context budget summary, and winner tables all live in `capabilities.md`. Per-plugin context detail lives in `registry/plugins/*.md`.

### D5: Overrides Injected into Global ~/.claude/CLAUDE.md
Losing capabilities are suppressed through behavioral rules injected into `~/.claude/CLAUDE.md` using marker-bounded sections. This ensures overrides are loaded in every session (global scope), affect cross-plugin behavior, and never clobber user content. Users keep all plugins intact.

### D6: Registry Committed to Git
All registry files are version-controlled, enabling history, rollback, and snapshots.

### D7: Self-Exclusion
Plugin-analyzer always skips analyzing itself to prevent circular reference.

### D8: Manual Plugin Management
Users install and remove plugins manually. Plugin-analyzer only analyzes what's installed.

### D9: Local-Only Scoring (No External APIs)
All scoring dimensions must be assessable from the local plugin directory. "Plugin Completeness" replaced "Repo Health" because GitHub API access is unreliable and unavailable in offline contexts. This guarantees scoring works on any machine.

### D10: Deterministic Tiebreaking
When scores are within 0.5 (tie zone), the winner is determined by: (1) higher Context Efficiency score, then (2) alphabetically earlier plugin name. This eliminates dependency on processing order and ensures the same input always produces the same output.

### D11: Category Stickiness
Once a capability is assigned to a category in `capabilities.md`, that assignment persists across runs. This prevents alias entries from splitting across categories when Claude makes slightly different categorization judgments between sessions.

### D12: Score Pinning on Update
`/update` only scores new plugins — existing plugins retain their pinned scores from the last `/init` or initial analysis. This prevents override flip-flops caused by LLM scoring variance (~±0.5 between identical runs).
