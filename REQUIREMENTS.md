# Requirements — plugin-analyzer

## Problem Statement

The Claude Code ecosystem has a rapidly growing number of plugins, each bundling multiple capabilities (skills, agents, commands, hooks, MCP servers). Many plugins offer overlapping capabilities — for example, "code review" might exist as an agent in one plugin, a skill in another, and a hook in a third.

Without a systematic approach, users end up with:
- Duplicate capabilities consuming unnecessary context
- No clarity on which implementation is best within each form (skill vs skill, agent vs agent)
- No visibility into total context budget consumed by plugins
- No way to track what's installed, what's winning, and what's dead weight

## Goals

1. **Analyze** all installed Claude Code plugins by decomposing them into individual capabilities
2. **Score** each capability on a consistent 0-10 scale using a weighted 4-dimension model
3. **Compare** same-form capabilities across plugins (skill vs skill, agent vs agent) to determine winners
4. **Track** context budget (always-on vs on-demand) across all plugins
5. **Generate** CLAUDE.md override rules to suppress losing capabilities automatically

## User Workflow

```
1. User discovers and installs plugins MANUALLY
   (using `claude plugin add <url>` or marketplace install)

2. User runs /plugin-analyzer:init (first time)
   → Scans all installed plugins
   → Analyzes, scores, and registers everything
   → Builds the complete registry from scratch

3. User installs or removes plugins over time

4. User runs /plugin-analyzer:update (subsequent times)
   → Detects NEW plugins (installed but not yet in registry)
   → Detects REMOVED plugins (in registry but no longer installed)
   → Analyzes only new plugins, cleans up removed ones
   → Recalculates winners and overrides

5. User runs /plugin-analyzer:status (anytime)
   → Views the full registry: plugins, winners, context budget, overrides
```

## Functional Requirements

### FR-1: `/plugin-analyzer:init` Command

**Purpose**: Build (or rebuild) the entire registry from scratch.

**Behavior**:
1. Scan `~/.claude/plugins/cache/` recursively to discover all installed plugins
   - A valid plugin contains `.claude-plugin/plugin.json`
   - Plugins may be nested: `<source>/<plugin-name>/<hash>/`
   - Must handle both marketplace bundles (multiple plugins under one source) and standalone plugins
2. Clear existing registry data (plugins.json, per-plugin files, capabilities.md)
3. For each discovered plugin:
   a. Read plugin metadata from `plugin.json`
   b. Inventory all components (skills, agents, commands, hooks, MCP servers)
   c. Measure context cost (character count) of each component
   d. Score each capability on 4 dimensions (see Scoring Algorithm)
   e. Classify each capability into a category
   f. Identify the component's form (skill, agent, command, hook, MCP)
   g. Write per-plugin analysis file to `registry/plugins/<plugin-name>.md`
   h. Add plugin entry to `registry/plugins.json`
4. Build `registry/capabilities.md`:
   - Group capabilities by category, then by capability name, then by form
   - Compare same-form entries to determine winners
   - Include aliases, context summary, and override information
5. Inject override rules into `~/.claude/CLAUDE.md`:
   - Read existing `~/.claude/CLAUDE.md` content
   - Find or create the managed section between `<!-- BEGIN PLUGIN-ANALYZER OVERRIDES -->` and `<!-- END PLUGIN-ANALYZER OVERRIDES -->` markers
   - Replace only the content between markers with ignore rules for all IGNORE-status capabilities
   - Preserve all user content outside the markers
6. Report results to user

**Edge Cases**:
- No plugins installed → Report empty state, create empty registry
- Plugin has no `.claude-plugin/plugin.json` → Skip with warning
- Plugin has zero recognizable components → Skip with warning
- Plugin-analyzer's own directory found → Skip self (do not analyze itself)

### FR-2: `/plugin-analyzer:update` Command

**Purpose**: Incrementally update the registry for newly installed or removed plugins.

**Behavior**:
1. Scan `~/.claude/plugins/cache/` to discover all currently installed plugins
2. Read `registry/plugins.json` to get the list of currently tracked plugins
3. Compute the diff:
   - **NEW**: Installed but not in registry → Analyze and register
   - **REMOVED**: In registry but not installed → Remove from registry
   - **EXISTING**: In both → Skip (already analyzed)
4. For each NEW plugin: perform full analysis (same as init step 3)
5. For each REMOVED plugin:
   a. Delete `registry/plugins/<plugin-name>.md`
   b. Remove from `plugins.json`
   c. Remove all its capabilities from `capabilities.md`
6. Rebuild `capabilities.md` winners (removal of a plugin might change winners)
7. Re-inject override rules into `~/.claude/CLAUDE.md` (same marker-based approach as init step 5)
8. Report: X new plugins analyzed, Y plugins removed, Z unchanged

**Edge Cases**:
- No changes detected → Report "Registry is up to date"
- Registry doesn't exist yet → Suggest running `/plugin-analyzer:init` first
- A previously-tracked plugin's path changed (reinstalled) → Treat as removed + new

### FR-3: `/plugin-analyzer:status` Command

**Purpose**: Display the current state of the registry (read-only).

**Behavior**:
1. Read all registry files
2. Present:
   a. **Summary dashboard**: Total plugins, capabilities, winners count, context budget
   b. **Plugin table**: Each plugin with capability count, win rate, avg score, context cost
   c. **Winner board**: Category-by-category, form-by-form view of winning capabilities
   d. **Contested capabilities**: Where multiple same-form entries compete (show scores)
   e. **Active overrides**: Current CLAUDE.md ignore rules

**Edge Cases**:
- Empty registry → Display "No plugins tracked. Run /plugin-analyzer:init to start."

## Scoring Algorithm

### Capability-Level Scoring

Each capability is scored on 4 dimensions, each 0-10:

| Dimension | Weight | Assessment Method |
|-----------|--------|-------------------|
| Implementation Quality | 35% | Read source: prompt clarity, structure, edge cases, progressive disclosure |
| Context Efficiency | 25% | Inverse of character count (see scale below) |
| Utility | 25% | How frequently the capability is needed in daily developer work |
| Plugin Completeness | 15% | Local assessment only (see scale below) |

**Formula**: `Score = (Quality × 0.35) + (Context × 0.25) + (Utility × 0.25) + (Completeness × 0.15)`

**Score range**: 0.0 - 10.0 (one decimal place)

### Plugin Completeness Scale

Assessed entirely from the local plugin directory — no external API calls.

| Score | Criteria |
|-------|----------|
| 9-10 | Has README, multiple component types (skills + agents + commands), reference docs, clear directory structure |
| 7-8 | Has README, 2+ component types, reasonable structure |
| 5-6 | Has README or 2+ component types (not both), basic structure |
| 3-4 | Single component type, minimal documentation |
| 1-2 | Single component, no README, flat structure |
| 0 | Bare plugin.json with nothing else |

### Context Efficiency Scale

| Score | Character Count |
|-------|----------------|
| 10 | < 500 |
| 9 | 500 - 1,000 |
| 8 | 1,000 - 1,500 |
| 7 | 1,500 - 2,000 |
| 6 | 2,000 - 3,000 |
| 5 | 3,000 - 4,000 |
| 4 | 4,000 - 5,000 |
| 3 | 5,000 - 7,000 |
| 2 | 7,000 - 10,000 |
| 1 | 10,000 - 15,000 |
| 0 | > 15,000 |

### What Counts as Context

| Component Type | Context Cost | Load Type |
|---------------|-------------|-----------|
| Skills (SKILL.md body) | Counted | On-demand (when triggered) |
| Commands (.md file) | Counted | On-demand (when invoked) |
| Hooks (prompt-based) | Counted (prompt string) | Always-on |
| Hooks (command-based) | Negligible (~50 chars) | Always-on |
| Agents | **NOT counted** | Separate subagent context |
| MCP server tool definitions | Negligible (~100 chars/tool) | Always-on |

### Agent Scoring Exception

Since agents run in separate context and are excluded from the context budget, use a **default Context Efficiency score of 7** (neutral) for all agent capabilities.

### Form-Based Comparison

Capabilities are compared **only within the same form type**. Different forms serve different purposes and are complementary, not competing.

| Form | Purpose | Competes With |
|------|---------|---------------|
| Skill | Knowledge injection — teaches Claude how | Other skills for same domain |
| Agent | Autonomous execution — does the work | Other agents for same task |
| Command | Explicit user action — triggered manually | Other commands for same action |
| Hook | Automatic guardrail — fires on events | Other hooks for same event |

**Examples**:
- Code Review SKILL (superpowers) vs Code Review SKILL (another-plugin) → COMPETE
- Code Review AGENT (pr-review-toolkit) vs Code Review AGENT (another-plugin) → COMPETE
- Code Review SKILL vs Code Review AGENT → COMPLEMENTARY (both can be WINNER)

### Winner Selection Rules (Within Same Form)

1. **Clear winner** (score gap > 0.5): Higher score wins → WINNER
2. **Tie zone** (score gap <= 0.5): Break tie deterministically:
   a. Higher Context Efficiency score wins (prefer lighter plugins)
   b. If still tied: alphabetically earlier plugin name wins
3. **RUNNER-UP**: Within 2.0 points of winner, above 5.0
4. **IGNORE**: Below 5.0, or more than 2.0 points behind winner

### Plugin-Level Profile (Derived, Not Scored)

Plugins are NOT scored with a single number. Instead, a profile is derived:

| Metric | Derivation |
|--------|-----------|
| Total capabilities | Count of all capabilities in the plugin |
| Winners count | How many of its capabilities are WINNER (per-form) |
| Win rate | Winners / Total capabilities |
| Best score | Highest individual capability score |
| Average score | Mean of all capability scores |
| Always-on context | Sum of always-on component chars |
| On-demand context | Sum of on-demand component chars |

## Capability Categories

| Category | What Belongs |
|----------|-------------|
| Code Quality | Code review, linting, type checking, test analysis, silent failure detection, comment analysis, code simplification |
| Workflow | Git commit, PR creation/review, branch management, deployment, CI/CD, cleanup tasks |
| Development Process | Debugging, TDD, planning, feature development, refactoring, architecture design |
| Knowledge & Reference | Framework best practices, style guides, documentation lookup, API references |
| Meta & Tooling | Hook creation, skill management, plugin management, learning/reflection, CLAUDE.md management |
| Integration | MCP servers, external APIs, database connections, IDE integration |
| Content & Media | Video creation, diagram generation, documentation writing, presentation tools |

New categories may be created if a capability truly doesn't fit any existing category.

## Capability Aliases

Each capability in `capabilities.md` includes an `aliases` line listing known component names that map to it. This ensures consistent deduplication across analysis sessions.

```markdown
### Code Review
> aliases: code-reviewer, review-code, pr-review, requesting-code-review
```

**Matching logic during analysis**:
1. Scan all alias lines in `capabilities.md` for the component name
2. Match found → Use that capability's **existing** canonical name **and existing category** (sticky assignment)
3. No match → Create a new capability entry, add the component name as first alias

### Category Stickiness

Once a capability is assigned a category (stored in `capabilities.md`), that assignment is **permanent** unless manually changed by the user. This prevents category drift across runs.

- During `/init`: first run assigns categories freely; subsequent runs read existing `capabilities.md` aliases first
- During `/update`: new capabilities get fresh category assignments; existing capabilities keep their stored category
- **Rationale**: If "code-reviewer" is categorized under "Code Quality" in run 1, it must stay there in run 2. Otherwise aliases split across categories and dedup breaks.

## Score Pinning

Scores are **pinned once assigned** and only change when explicitly re-scored.

### Pinning Rules

1. `/init` always re-scores everything (full rebuild)
2. `/update` only scores NEW plugins — existing plugins keep their pinned scores
3. A plugin's scores change only when:
   - `/init` is run (full rescan)
   - The plugin is detected as REMOVED + re-added (new hash = reinstalled)

### Rationale

LLM-based scoring is inherently non-deterministic. Without pinning, running `/update` could shift scores by ±0.5-1.0 between runs, causing winner/override flip-flops. Pinning ensures the registry is **stable** — overrides don't randomly change between sessions.

## Non-Functional Requirements

- **NFR-1**: Plugin-analyzer must never modify, delete, or interfere with installed plugins. It is read-only with respect to `~/.claude/plugins/cache/`.
- **NFR-2**: Plugin-analyzer must skip analyzing itself to avoid circular reference.
- **NFR-3**: Scoring is fully automated — no user approval needed for individual scores.
- **NFR-4**: All registry files must be human-readable markdown or JSON.
- **NFR-5**: The `status` command must be read-only (only Read and Glob tools).
- **NFR-6**: Override rules are injected into `~/.claude/CLAUDE.md` within a managed section bounded by `<!-- BEGIN PLUGIN-ANALYZER OVERRIDES -->` and `<!-- END PLUGIN-ANALYZER OVERRIDES -->` markers. Only content between markers is modified; all user content is preserved.

## Out of Scope

- **Plugin installation/uninstallation**: User handles this manually
- **Plugin auto-update**: Plugin-analyzer does not fetch or update plugin repos
- **URL handling**: No GitHub URL input, no cloning, no fetching
- **Real-time monitoring**: Runs on-demand only (init/update/status)
- **Multi-user support**: Single-user tool
- **Cross-form comparison**: Skills never compete with agents, commands never compete with hooks
