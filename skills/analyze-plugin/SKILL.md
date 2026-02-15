---
name: analyze-plugin
description: This skill should be used when the user asks to "analyze a plugin", "score a plugin", "evaluate a plugin", "compare plugins", "rate a plugin", "check plugin quality", or wants to understand a Claude Code plugin's capabilities, context cost, and quality score. Provides the scoring algorithm and analysis methodology for the plugin-analyzer registry.
---

# Plugin Analysis Methodology

Analyze, score, and register Claude Code plugins into the plugin-analyzer registry. Each plugin is decomposed into individual **capabilities**, scored on a 0-10 scale, and compared against same-form entries to determine winners.

## Core Concepts

### Capabilities, Not Plugins
Never track plugins as monolithic units. Decompose each plugin into discrete **capabilities** — individual things it can do. A plugin with 5 skills, 2 agents, and 3 commands has up to 10 capabilities to evaluate independently.

### Form-Based Comparison
Different forms serve different purposes and are **complementary, not competing**:

| Form | Purpose | Competes With |
|------|---------|---------------|
| Skill | Knowledge injection — teaches Claude how | Other skills for same domain |
| Agent | Autonomous execution — does the work | Other agents for same task |
| Command | Explicit user action — triggered manually | Other commands for same action |
| Hook | Automatic guardrail — fires on events | Other hooks for same event |

A Code Review SKILL and a Code Review AGENT are both valuable — they serve different roles. Only compare within the same form.

## Analysis Workflow

### Step 1: Scan Plugin Structure

Read the plugin to inventory all components:

1. Read `.claude-plugin/plugin.json` for metadata (name, version, description, repo URL)
2. Scan `skills/` — each subdirectory with `SKILL.md` is a skill capability (form = "skill")
3. Scan `agents/` — each `.md` file is an agent capability (form = "agent")
4. Scan `commands/` — each `.md` file is a command capability (form = "command")
5. Read `hooks/hooks.json` — each hook entry is a hook capability (form = "hook")
6. Read `.mcp.json` — each MCP server is an MCP capability (form = "mcp")

For each component, read its content to assess quality and measure character count.

### Step 2: Classify Each Capability

Assign each capability to exactly one category:

| Category | Covers |
|----------|--------|
| Code Quality | Code review, linting, type checking, test analysis, silent failure detection |
| Workflow | Git commit, PR creation, branch management, deployment, CI/CD |
| Development Process | Debugging, TDD, planning, feature development, refactoring |
| Knowledge & Reference | Best practices, documentation, patterns, style guides |
| Meta & Tooling | Hook creation, skill management, plugin management, learning capture |
| Integration | MCP servers, external service connections, API wrappers |
| Content & Media | Video creation, diagram generation, documentation writing |

If a capability does not fit any category, create a new one.

### Step 3: Score Each Capability

Apply the 4-dimension weighted scoring model. For the full rubric, consult `references/scoring-algorithm.md`.

| Dimension | Weight | What to Assess |
|-----------|--------|---------------|
| Implementation Quality | 35% | Prompt clarity, edge cases, structure |
| Context Efficiency | 25% | Character count vs. value delivered |
| Utility | 25% | How frequently needed in daily work |
| Plugin Completeness | 15% | Component variety, README, docs, structure (local-only) |

**Formula**: `Score = (Quality × 0.35) + (Context × 0.25) + (Utility × 0.25) + (Completeness × 0.15)`

**Agent exception**: Since agents run in separate context, use default Context score of 7.

### Step 4: Measure Context Cost

For each component, count characters that load into context:

- **Skills**: Count chars in `SKILL.md` body (references load separately)
- **Commands**: Count chars in the command `.md` file
- **Hooks with prompts**: Count chars in the `prompt` field (always-on)
- **Hooks with commands**: Negligible (~50 chars)
- **Agents**: **SKIP** — loads in separate subagent context
- **MCP servers**: Negligible

Classify as **always-on** (hooks, CLAUDE.md rules) or **on-demand** (skills, commands).

### Step 5: Match Canonical Names

Check `registry/capabilities.md` for existing alias lines:
1. Scan `> aliases:` lines for the component name
2. Match found → Use that capability's canonical name
3. No match → Create a new canonical name, add component name as first alias

### Step 6: Update Registry Files

After scoring, update all registry files:

1. **plugins/<plugin-name>.md** — Full per-plugin analysis with profile, capability table, scoring breakdown
2. **plugins.json** — Add/update plugin entry with profile metrics
3. **capabilities.md** — Group by category → capability → form. Compare same-form entries. Higher score = WINNER.
4. **~/.claude/CLAUDE.md** — For IGNORE-status capabilities, inject override rules into managed section (marker-bounded), grouped by category + form

## Per-Plugin File Template

```markdown
# Plugin: <name>
> Source: <source>
> Installed: <path>
> Version: <version>
> Analyzed: <date>
> Plugin Completeness: <X>/10

## Plugin Profile
| Metric | Value |
|--------|-------|
| Total Capabilities | X |
| Winners | Y |
| Win Rate | Z% |
| Avg Score | X.X |
| Best Score | X.X |
| Always-on Context | X chars |
| On-demand Context | X chars |

## Capabilities
| # | Capability | Category | Form | Score | Context (chars) | Load | Status |
|---|-----------|----------|------|-------|-----------------|------|--------|

## Scoring Breakdown
### 1. Capability Name (Score)
- Quality: X — reasoning
- Context: X — reasoning
- Utility: X — reasoning
- Completeness: X — reasoning
```

## Winner Selection Logic

Within same-form entries for a capability:
1. Score gap > 0.5 → Higher score becomes WINNER
2. Score gap <= 0.5 → Deterministic tiebreak: higher Context Efficiency wins, then alphabetically earlier plugin name
3. Within 2.0 of winner, above 5.0 → RUNNER-UP
4. Below 5.0 or 2.0+ behind → IGNORE

Cross-form entries never compete. Both a winning skill and a winning agent can coexist.

## Additional Resources

- **`references/scoring-algorithm.md`** — Full scoring rubric with detailed examples for each score level, context efficiency scale, and edge case handling
