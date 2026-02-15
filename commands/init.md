---
name: init
description: Scan ALL installed Claude Code plugins, analyze every capability, build (or rebuild) the entire registry from scratch
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Initialize / Rebuild the Plugin Registry

Scan every installed Claude Code plugin, decompose into capabilities, score them, determine per-form winners, and generate the complete registry from scratch.

## When to Use

- **First time setup**: Bootstrap the registry with all currently installed plugins
- **Periodic rebuild**: Re-scan everything to catch new installs, removed plugins, version changes
- **Fresh start**: Wipe the registry and rebuild from the current plugin state

## Execution Steps

### 1. Discover All Installed Plugins

Find all installed plugins by scanning for plugin manifests:

```bash
# Glob for all plugin.json files in the cache
~/.claude/plugins/cache/**/.claude-plugin/plugin.json
```

For each found `plugin.json`:
- The plugin root is the parent directory of `.claude-plugin/`
- Read the `name` field from `plugin.json` to identify the plugin
- Extract the source name (the directory directly under `plugins/cache/`)
- **Skip** any plugin named `plugin-analyzer` (self-exclusion)
- **Skip** directories without `.claude-plugin/plugin.json` (not valid plugins)

Build a list of `{ name, source, path }` for all valid plugins.

### 2. Clear Existing Registry

Start clean:

1. Reset `${CLAUDE_PLUGIN_ROOT}/registry/plugins.json`:
   ```json
   {
     "version": "1.0.0",
     "last_updated": "<today's date>",
     "plugins": []
   }
   ```
2. Delete all `.md` files in `${CLAUDE_PLUGIN_ROOT}/registry/plugins/` (keep .gitkeep)
3. Capabilities.md and overrides will be rebuilt from results

### 3. Analyze Each Plugin

For each discovered plugin, use the analyze-plugin skill methodology:

1. **Read metadata**: `plugin.json` → name, version, description, repository
2. **Inventory components**:
   - Glob `skills/*/SKILL.md` → skills (form = "skill")
   - Glob `agents/*.md` → agents (form = "agent")
   - Glob `commands/*.md` → commands (form = "command")
   - Read `hooks/hooks.json` → hooks (form = "hook")
   - Read `.mcp.json` → MCP servers (form = "mcp")
3. **For each component**:
   - Read content, measure character count
   - Score on 4 dimensions (Quality, Context, Utility, Completeness)
   - Calculate weighted score
   - Classify into a capability category (use existing category from capabilities.md aliases if match found — category stickiness)
   - Assign canonical capability name (match aliases in existing capabilities.md, or create new)
   - Identify form type
4. **Compute plugin profile**: total capabilities, always-on chars, on-demand chars
5. **Write per-plugin file**: `${CLAUDE_PLUGIN_ROOT}/registry/plugins/<plugin-name>.md`

For Plugin Completeness scoring: assess from local directory only — README presence, number of component types, reference docs, directory structure.

### 4. Build capabilities.md

After ALL plugins are analyzed:

1. Collect every capability across all plugins
2. Group by: category → capability name → form
3. Within each form group, compare scores:
   - Highest score → WINNER
   - Within 0.5 of winner → deterministic tiebreak: higher Context Efficiency wins, then alphabetically earlier plugin name
   - Within 2.0 of winner and above 5.0 → RUNNER-UP
   - Below 5.0 or more than 2.0 behind → IGNORE
4. Cross-form entries are NOT compared (skill vs agent = complementary)
5. Include aliases line under each capability name
6. Compute global summary: plugin count, capability count, context totals
7. Write `${CLAUDE_PLUGIN_ROOT}/registry/capabilities.md`
8. Write `${CLAUDE_PLUGIN_ROOT}/registry/plugins.json` with all plugin entries

### 5. Inject Overrides into ~/.claude/CLAUDE.md

Inject override rules into the user's global `~/.claude/CLAUDE.md`:

1. Read existing `~/.claude/CLAUDE.md` (create if it doesn't exist)
2. Find `<!-- BEGIN PLUGIN-ANALYZER OVERRIDES -->` and `<!-- END PLUGIN-ANALYZER OVERRIDES -->` markers
3. If markers exist → replace everything between them with new override rules
4. If markers don't exist → append the full block (markers + content) at the end of the file
5. For every capability with IGNORE status, add an override rule within the managed section
6. Group rules by category + form
7. Format: `- Do NOT use \`component-name\` [form] from \`plugin-name\`. Use \`winner-plugin:component\` [form] instead.`
8. **Never modify any content outside the markers**

### 6. Report Results

Present a summary:

```
PLUGIN ANALYZER — INIT COMPLETE
================================

Plugins discovered: X
Plugins analyzed: Y (skipped: Z)
Total capabilities: N

Per-Plugin Summary:
| Plugin | Capabilities | Win Rate | Avg Score | Always-on | On-demand |
|--------|-------------|----------|-----------|-----------|-----------|

Winner Board (by category + form):
  [Category] → [Form]: [Capability] → [Winner] (Score)

Contested (same-form competition): X capabilities
Overrides Generated: Y

Context Budget:
  Always-on:  X,XXX chars
  On-demand:  X,XXX chars
```

Suggest next steps:
- Run `/plugin-analyzer:status` for detailed view
- Review overrides in `~/.claude/CLAUDE.md` (managed section)
