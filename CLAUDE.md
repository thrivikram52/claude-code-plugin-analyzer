# plugin-analyzer

This plugin analyzes installed Claude Code plugins, scores their capabilities, and maintains a registry of per-form winners.

## Commands

- `/plugin-analyzer:init` — Full scan and rebuild of the registry
- `/plugin-analyzer:update` — Incremental update (new/removed plugins only)
- `/plugin-analyzer:status` — Read-only dashboard of the registry

## Override Injection

This plugin writes auto-generated override rules into `~/.claude/CLAUDE.md` within a clearly marked section:
```
<!-- BEGIN PLUGIN-ANALYZER OVERRIDES -->
...rules...
<!-- END PLUGIN-ANALYZER OVERRIDES -->
```

These rules suppress IGNORE-status capabilities in favor of winners. Run `/plugin-analyzer:init` or `/plugin-analyzer:update` to regenerate.
