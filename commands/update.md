---
name: update
description: Detect newly installed or removed plugins, analyze only new ones, clean up removed ones, and recalculate winners
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Update Plugin Registry

Compare currently installed plugins against the registry, analyze new plugins, remove entries for uninstalled plugins, and recalculate winners.

## When to Use

- After installing new plugins
- After removing/uninstalling plugins
- As a quick sync (faster than full init)

## Execution Steps

### 1. Discover + Diff

**Discover** all installed plugins (same as init step 1):
- Glob `~/.claude/plugins/cache/**/.claude-plugin/plugin.json`
- Build list of `{ name, source, path }`
- Skip self (`plugin-analyzer`)

**Read** `${CLAUDE_PLUGIN_ROOT}/registry/plugins.json` for currently tracked plugins.

If `plugins.json` doesn't exist or is empty, suggest: "Registry not initialized. Run `/plugin-analyzer:init` first."

**Compute diff**:
- **NEW**: Plugin name found in installed but not in plugins.json → needs analysis
- **REMOVED**: Plugin name in plugins.json but not found installed → needs cleanup
- **EXISTING**: Plugin name in both → skip (already analyzed)

Match by plugin `name` field (not by path, since paths may change with reinstalls).

### 2. Analyze NEW Plugins

For each NEW plugin, perform full analysis (same as init step 3):
1. Read metadata, inventory components
2. Score each capability on 4 dimensions
3. Classify, assign canonical name, identify form
4. Compute plugin profile
5. Write per-plugin file to `${CLAUDE_PLUGIN_ROOT}/registry/plugins/<plugin-name>.md`

### 3. Clean Up REMOVED Plugins

For each REMOVED plugin:
1. Delete `${CLAUDE_PLUGIN_ROOT}/registry/plugins/<plugin-name>.md`
2. Remove its entry from the plugins list (will be written in step 4)

### 4. Rebuild capabilities.md

Must fully rebuild because removals can change winners:

1. Read ALL per-plugin files in `${CLAUDE_PLUGIN_ROOT}/registry/plugins/`
2. Collect every capability across all remaining plugins
3. Group by: category → capability name → form
4. Compare within same form → assign WINNER/RUNNER-UP/IGNORE
5. Preserve existing aliases (from old capabilities.md), add new ones
6. Compute global summary
7. Write `${CLAUDE_PLUGIN_ROOT}/registry/capabilities.md`
8. Write `${CLAUDE_PLUGIN_ROOT}/registry/plugins.json` with updated list

### 5. Re-inject Overrides

Re-inject override rules into `~/.claude/CLAUDE.md` using the marker-based approach (same as init step 5). Only the managed section between `<!-- BEGIN PLUGIN-ANALYZER OVERRIDES -->` and `<!-- END PLUGIN-ANALYZER OVERRIDES -->` is modified.

### 6. Report Changes

```
PLUGIN ANALYZER — UPDATE COMPLETE
===================================

New plugins analyzed: X
  - plugin-a (5 capabilities, 4 winners)
  - plugin-b (3 capabilities, 3 winners)

Removed plugins: Y
  - plugin-c (was providing: Code Review agent, Git Commit command)

Unchanged: Z

Winner Changes:
  [Capability] [Form]: old-winner → new-winner (score: X.X → Y.Y)

Context Budget Change:
  Always-on: X,XXX → Y,YYY (±delta)
  On-demand: X,XXX → Y,YYY (±delta)
```

If nothing changed: "Registry is up to date. No new or removed plugins detected."
