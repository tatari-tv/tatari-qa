# tatari-qa

Tatari-specific QA automation plugin for [Claude Code](https://claude.ai/code). Playwright-powered browser testing for Tatari2 workflows.

## Why a Separate Plugin?

This plugin lives outside [Conductor](https://github.com/tatari-tv/conductor) because:

- **Tatari-specific**: Hardcoded to Tatari2 URLs, Okta auth, and internal form structures
- **Cross-repo**: QA workflows span philo and philo-fe — a single-repo `.claude/commands/` file can't reach both
- **Conductor stays general**: Conductor provides engineering workflow tools usable by any team

## Installation

```bash
/plugin marketplace add tatari-tv/tatari-qa
/plugin install tatari-qa@tatari-qa
```

## Available Commands

| Command | Description |
|---------|-------------|
| `/tatari-qa:bootstrap` | Bootstrap the full QA dogfooding environment |
| `/tatari-qa:campaign-qa <scenario>` | Execute YAML-driven campaign creation QA scenarios |

## Prerequisites

- [Claude Code](https://claude.ai/code) with Playwright MCP server configured
- Local Tatari2 dev environment running
- philo-fe at `http://local.tatari.tools:3000`

## Campaign QA

Run pre-defined campaign creation scenarios against the local Tatari2 campaign form:

```
/tatari-qa:campaign-qa awareness_cpm_daily_prospecting
```

Scenario YAML files live in the philo repo at `script/qa/campaign_creation/scenarios/`. See the [setup guide](https://github.com/tatari-tv/philo/blob/master/script/qa/campaign_creation/README.md) in philo for details.

### Supported Scenarios

13 pre-built scenarios across 3 tiers:

- **Tier 1** (no extra setup): Awareness + CPM combinations
- **Tier 2** (needs conversion events): CPA, CPV, ROAS scenarios
- **Tier 3** (needs segment data): Retargeting scenarios

## Dogfooding

Run `/tatari-qa:bootstrap` to set up the full campaign QA dogfooding environment. The agent discovers your local repos, checks out the correct feature branches, validates prerequisites, runs setup scripts, and reports readiness.

This is for pre-merge dogfooding while these PRs are open:
- philo [#17588](https://github.com/tatari-tv/philo/pull/17588) — Beeswax sandbox setup automation
- philo [#17593](https://github.com/tatari-tv/philo/pull/17593) — Execute queued orders script
- philo-fe [#8391](https://github.com/tatari-tv/philo-fe/pull/8391) — Campaign QA scenarios

Once the PRs merge to main, the branch checkout steps become no-ops (the agent detects main has everything).

See [Prerequisites](#prerequisites) for base requirements.

## License

MIT
