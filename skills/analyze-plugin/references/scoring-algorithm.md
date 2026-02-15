# Scoring Algorithm — Full Rubric

## Overview

Each capability is scored on 4 dimensions (0-10 each), combined with weighted average.

**Formula**: `Final = (Quality × 0.35) + (Context × 0.25) + (Utility × 0.25) + (Completeness × 0.15)`

**Score Range**: `0.0 - 10.0` (one decimal place)

---

## Dimension 1: Implementation Quality (35%)

Assess by reading the actual source (SKILL.md, agent prompt, hook config, command body).

| Score | Level | Criteria |
|-------|-------|----------|
| 9-10 | Exceptional | Precise, structured prompts. Handles edge cases. Progressive disclosure. Clear sections. |
| 7-8 | Strong | Clear instructions, good structure. Covers common cases. Minor gaps. |
| 5-6 | Adequate | Gets the job done. Some vagueness. Missing edge cases. |
| 3-4 | Weak | Vague or generic. Missing important cases. Unclear instructions. |
| 1-2 | Poor | Barely functional. Contradictory or confusing. |
| 0 | Broken | Non-functional, empty, or placeholder. |

### Positive Signals (raise score)
- Specific, actionable instructions
- Structured output formats defined
- Edge case handling
- Progressive disclosure (lean main + detailed references)
- Strong trigger descriptions (skills/agents)

### Negative Signals (lower score)
- "You should..." second-person style throughout
- Overly generic ("review the code for issues")
- Everything in one file (no progressive disclosure)
- Hardcoded paths instead of ${CLAUDE_PLUGIN_ROOT}
- Missing frontmatter or metadata

---

## Dimension 2: Context Efficiency (25%)

| Score | Char Count | Description |
|-------|-----------|-------------|
| 10 | < 500 | Ultra-lean |
| 9 | 500 - 1,000 | Very efficient |
| 8 | 1,000 - 1,500 | Lean |
| 7 | 1,500 - 2,000 | Moderate |
| 6 | 2,000 - 3,000 | Moderate-heavy |
| 5 | 3,000 - 4,000 | Heavy |
| 4 | 4,000 - 5,000 | Bloated |
| 3 | 5,000 - 7,000 | Very bloated |
| 2 | 7,000 - 10,000 | Excessive |
| 1 | 10,000 - 15,000 | Extreme |
| 0 | > 15,000 | Unacceptable |

### What Counts
- **Skills**: SKILL.md body only (references load separately)
- **Commands**: Full command markdown file
- **Hooks (prompt)**: Prompt string in hooks.json
- **Agents**: **DO NOT COUNT** — use default score of 7
- **MCP**: Negligible (~100 chars/tool)

### Adjustment Rule
If context efficiency is 0-4 but quality is 8+ and uses progressive disclosure, bump context score by 1.

---

## Dimension 3: Utility (25%)

| Score | Frequency | Examples |
|-------|-----------|---------|
| 10 | Multiple times daily | Code review, git commit, debugging |
| 9 | Daily | PR creation, test running, linting |
| 8 | Several times weekly | Planning, refactoring, documentation |
| 7 | Weekly | TDD, performance analysis |
| 6 | Several times monthly | Migration, deployment automation |
| 5 | Monthly | Framework patterns, API integration |
| 4 | Occasionally | Niche workflow |
| 3 | Rarely | Very specialized |
| 2 | Almost never | Hyper-specific scenario |
| 1 | One-time | Setup/bootstrap only |
| 0 | No practical use | Novelty, demo only |

---

## Dimension 4: Plugin Completeness (15%)

Assessed entirely from the local plugin directory — no external API calls required.

| Score | Level | Criteria |
|-------|-------|----------|
| 9-10 | Comprehensive | Has README, multiple component types (skills + agents + commands), reference docs, clear directory structure |
| 7-8 | Well-structured | Has README, 2+ component types, reasonable structure |
| 5-6 | Basic | Has README or 2+ component types (not both), basic structure |
| 3-4 | Minimal | Single component type, minimal documentation |
| 1-2 | Bare | Single component, no README, flat structure |
| 0 | Empty | Bare plugin.json with nothing else |

---

## Form-Based Comparison Rules

### Same-Form Competition
1. **Clear winner** (gap > 0.5): Higher score wins
2. **Tie zone** (gap <= 0.5): Deterministic tiebreak — higher Context Efficiency wins, then alphabetically earlier plugin name
3. **RUNNER-UP**: Within 2.0 of winner, above 5.0
4. **IGNORE**: Below 5.0, or 2.0+ behind winner

### Cross-Form: No Competition
- Skill vs Agent → complementary (both can be WINNER)
- Command vs Hook → complementary (different trigger mechanisms)
- Only same form competes: skill vs skill, agent vs agent, etc.

### Status Labels

| Status | Meaning | Override generated? |
|--------|---------|---------------------|
| WINNER | Best same-form implementation | No |
| RUNNER-UP | Good alternative, kept as backup | No |
| IGNORE | Outclassed or low quality | YES |

---

## Capability Categories

| Category | What Belongs |
|----------|-------------|
| Code Quality | Code review, linting, type checking, test analysis, silent failure detection, comment analysis, code simplification |
| Workflow | Git commit, PR creation/review, branch management, deployment, CI/CD, cleanup |
| Development Process | Debugging, TDD, planning, feature development, refactoring, architecture |
| Knowledge & Reference | Framework best practices, style guides, documentation, API references |
| Meta & Tooling | Hook creation, skill management, plugin management, learning/reflection, CLAUDE.md |
| Integration | MCP servers, external APIs, database connections, IDE integration |
| Content & Media | Video creation, diagram generation, documentation writing, presentations |

---

## Example: Full Scoring

**Plugin**: pr-review-toolkit
**Capability**: code-reviewer (agent)

| Dimension | Score | Reasoning |
|-----------|-------|-----------|
| Quality | 9 | Multi-pass review, confidence-based filtering, clear output |
| Context | 7 | Agent (default neutral — separate context) |
| Utility | 10 | Used on every PR, multiple times daily |
| Completeness | 9 | Multiple component types (6 agents), README, clear structure |

**Final**: (9 × 0.35) + (7 × 0.25) + (10 × 0.25) + (9 × 0.15) = 3.15 + 1.75 + 2.50 + 1.35 = **8.75**
