# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

tatari-qa is a Claude Code plugin for Tatari-specific QA automation. It provides Playwright-powered browser testing commands for Tatari2 workflows (campaign creation, etc.).

This plugin is intentionally separate from [Conductor](https://github.com/tatari-tv/conductor) — Conductor provides general-purpose engineering workflow tools, while tatari-qa contains Tatari-specific QA automation that spans multiple repos (philo, philo-fe).

## Architecture

### Plugin Structure

```text
tatari-qa/
├── .claude-plugin/    # Plugin manifest (plugin.json, marketplace.json)
├── commands/          # Slash commands (markdown with prompts)
├── CLAUDE.md          # This file
└── README.md          # Overview and setup
```

### Commands

Commands live in `commands/*.md` as markdown files with YAML frontmatter. Each file becomes a slash command (`/tatari-qa:<command-name>`).

### Scenario Files

YAML scenario files live in the **philo** repo at `script/qa/campaign_creation/scenarios/`. This plugin reads them at runtime — it does not bundle its own scenarios.

## Prerequisites

- Playwright MCP server configured in Claude Code
- Local Tatari2 dev environment (philo + philo-fe)
- For campaign QA: `aws-vault exec tatari-ro -- make dev` and philo-fe at `http://local.tatari.tools:3000`

## Key Patterns

### Playwright MCP Interaction

This plugin relies heavily on Playwright MCP tools (`mcp__playwright__*`). Key patterns discovered through live testing:

- **Radio buttons**: Click the container element, not the radio input itself (styled labels intercept pointer events)
- **React-Select**: First option is auto-focused on open; press Enter directly (ArrowDown skips to second)
- **Context overflow**: Never expand the interests tree; use `browser_run_code` for audience section interactions
- **Stale refs**: Use `browser_run_code` instead of ref-based clicks after React re-renders (e.g., creatives footer)

### Adding New Commands

1. Create `commands/<command-name>.md` with YAML frontmatter
2. Follow existing command patterns for Playwright interaction
3. Include error recovery section for known gotchas
4. Document context window management strategies
