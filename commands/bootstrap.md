---
description: Bootstrap the full campaign QA dogfooding environment across philo, philo-fe, and tatari-qa
---

# Bootstrap Command

Set up the full campaign QA dogfooding environment. This command discovers your local repos, checks out the correct feature branches, validates prerequisites, runs setup scripts, and reports readiness status.

Use this when you want to dogfood the campaign QA pipeline and need all repos on the right branches with the environment configured.

## Configuration

### Branch Mappings

| Repo | Target Branch | Source |
|------|--------------|--------|
| philo | `feat/campaign-qa-dogfood` | Combined branch: PR #17588 + PR #17593 |
| philo-fe | `feat/campaign-qa-scenarios` | PR #8391 |
| tatari-qa | `main` | Plugin is on main |

### Repo Discovery Paths

Try these locations in order for each repo:

| Repo | Primary | Fallback (relative to tatari-qa) |
|------|---------|----------------------------------|
| philo | `~/dev/philo` | `../philo` |
| philo-fe | `~/dev/philo-fe` | `../philo-fe` |

If neither location exists, ask the user for the path.

## Process

### Step 1: Discover Repositories

For each repo (philo, philo-fe), check if it exists:

```bash
# Try primary path first
test -d ~/dev/philo/.git
# Then fallback
test -d ../philo/.git
```

For tatari-qa, use the current working directory (this plugin's repo).

**If a repo is not found at either location:**
- Ask the user: "I couldn't find the {repo} repository at ~/dev/{repo} or ../{repo}. What's the path to your local {repo} checkout?"

Print discovered paths for confirmation:

```
Repository Locations
====================
philo:      ~/dev/philo
philo-fe:   ~/dev/philo-fe
tatari-qa:  ~/dev/tatari-qa (current)
```

### Step 2: Check Git State

For each repo, inspect current state:

```bash
git -C {repo_path} status --porcelain
git -C {repo_path} branch --show-current
```

**Decision tree for each repo:**

1. **Already on target branch** → Report "already on {branch}" and skip checkout for this repo
2. **On different branch, clean working tree** → Proceed to checkout in Step 3
3. **On different branch, dirty working tree** → Warn the user:
   ```
   Warning: {repo} has uncommitted changes on branch {current_branch}:
   {list of modified files}

   Options:
   1. Stash changes and continue (recommended)
   2. Abort — I'll clean up manually
   ```
   - If user wants to stash: run `git -C {repo_path} stash` and note the repo for a reminder later
   - If user wants to abort: stop and explain how to resume after cleaning up

Print a status summary after checking all repos.

### Step 3: Checkout Branches

For each repo that needs a branch change:

```bash
git -C {repo_path} fetch origin
git -C {repo_path} checkout {target_branch}
```

**If the branch doesn't exist locally but does on remote:**
```bash
git -C {repo_path} checkout -b {target_branch} origin/{target_branch}
```

**If the branch is not found anywhere (PRs may have merged):**

Check if the expected files already exist on `origin/main`:

- For philo: check if `script/ops/beeswax_sandbox_setup.sh` and `script/qa/execute_queued_orders.py` exist
- For philo-fe: check if `script/qa/campaign_creation/scenarios/` directory exists

```bash
git -C {repo_path} ls-tree origin/main -- {expected_file_path}
```

If files exist on main:
```
The feature branch has been merged to main. Checking out main instead.
```
Then `git -C {repo_path} checkout main && git -C {repo_path} pull`.

If files don't exist on main or remote:
```
The branch {target_branch} doesn't exist yet. This combined branch needs to be created first.
For philo: see Phase 1 of the implementation plan or ask Nova.
```

Verify each checkout succeeded:
```bash
git -C {repo_path} branch --show-current
```

Print summary:

```
Branch Status
=============
philo:      feat/campaign-qa-dogfood     [checked out]
philo-fe:   feat/campaign-qa-scenarios   [already on branch]
tatari-qa:  main                         [already on branch]
```

### Step 4: Validate Prerequisites

Run each check and collect results. Stop at the first critical failure.

#### 4a. Docker Running

```bash
docker info > /dev/null 2>&1
```

If not running:
```
Docker isn't running. Start Docker Desktop, then re-run /tatari-qa:bootstrap.
```
**This is a critical failure — stop here.**

#### 4b. Devserver Container Running

```bash
docker compose -f {philo_path}/docker-compose.yaml ps devserver --format json 2>/dev/null
```

If not running:
```
The philo devserver container isn't running. Start it with:
  cd {philo_path} && aws-vault exec tatari-ro -- make dev

Then re-run /tatari-qa:bootstrap.
```
**This is a critical failure — stop here.**

#### 4c. Beeswax Credentials

Read `{philo_path}/components/philo/config/local_creds.py` and check for `BEESWAX_USERNAME` with a real value.

Invalid values (treat as missing):
- Empty string `''` or `""`
- Placeholder text: `your_email@company.com`, `changeme`, `example.com`, `<your beeswax staging email>`
- Not present in the file at all

If missing or placeholder:
```
Beeswax staging credentials are not configured in:
  {philo_path}/components/philo/config/local_creds.py

Set BEESWAX_USERNAME and BEESWAX_PASSWORD to your Beeswax staging account credentials.
Ask in #proj-tatari2 Slack channel if you need access.
```
**This is a critical failure — stop here.**

#### 4d. Sandbox API Settings

Read `{philo_path}/components/philo/config/local_settings.py` and check for `BEESWAX_API` pointing to `tatarisbx`:

```python
# Look for a line matching: BEESWAX_API.*tatarisbx
```

If missing:
```
Beeswax sandbox API URL is not configured. Add this to:
  {philo_path}/components/philo/config/local_settings.py

BEESWAX_API = "https://tatarisbx.api.beeswax.com/rest/v2"
```
**This is a critical failure — stop here.**

Print prerequisite status:

```
Prerequisites
=============
Docker:           PASS - running
Devserver:        PASS - running
Beeswax creds:   PASS - configured
Sandbox settings: PASS - configured
```

### Step 5: Run Beeswax Sandbox Setup

The setup scripts are safe to re-run (upsert-style logic) but take ~30-60 seconds.

Ask the user:
```
Run Beeswax sandbox setup scripts? This takes ~30-60 seconds. (Y/n)
```

If user says no (or if invoked with `--skip-setup`): skip to Step 6.

**Run segment ingester first** (order matters):
```bash
docker compose -f {philo_path}/docker-compose.yaml exec devserver poetry run python script/programmatic_segment_ingester.py ingest-beeswax-segments
```

**Important:** Do NOT use the `-it` flag — Claude Code runs in a non-interactive context.

If this fails, diagnose the error:
- Beeswax credentials wrong → check `local_creds.py` values
- Network issues → check VPN / internet
- Docker container not running → re-check Step 4b

**Then run sandbox setup:**
```bash
docker compose -f {philo_path}/docker-compose.yaml exec devserver poetry run python script/ops/setup_local_dev_for_beeswax_sandbox.py
```

If segment ingester succeeded but sandbox setup fails:
```
Segment ingester completed but sandbox setup failed. You can re-run /tatari-qa:bootstrap to retry.
```

### Step 6: Verify philo-fe is Running

Check if philo-fe dev server is accessible:

```bash
curl -s -o /dev/null -w "%{http_code}" http://local.tatari.tools:3000/ 2>/dev/null
```

If returns any HTTP response: philo-fe is running.

If connection refused or timeout:
```
philo-fe isn't running at http://local.tatari.tools:3000.
Start it with: cd {philo_fe_path} && make dev

You can start it after bootstrap completes — it's not required for the setup steps.
```

**This is a non-blocking check** — report the status but don't halt.

### Step 7: Final Status Report

If any repos were stashed in Step 2, remind the user:
```
Reminder: You have stashed changes in {repo}. Run:
  cd {repo_path} && git stash pop
```

Print comprehensive status:

```
QA Dogfooding Environment Status
=================================

Repositories:
  philo:      {philo_path}       [{branch}]   {status}
  philo-fe:   {philo_fe_path}    [{branch}]   {status}
  tatari-qa:  {tatari_qa_path}   [{branch}]   {status}

Prerequisites:
  Docker:           {status}
  Devserver:        {status}
  Beeswax creds:    {status}
  Sandbox settings: {status}
  philo-fe:         {status}

Setup:
  Segment ingester: {status}
  Sandbox setup:    {status}

Ready to dogfood! Run a campaign QA scenario:
  /tatari-qa:campaign-qa awareness_cpm_daily_prospecting

Note: The execute-queued-orders pipeline step (CM-9203) is not yet
integrated. After campaign creation, run it manually:
  docker compose -f {philo_path}/docker-compose.yaml exec devserver poetry run python script/qa/execute_queued_orders.py --latest
```

Adjust the final message based on what actually happened:
- If setup was skipped: note that setup was skipped
- If philo-fe isn't running: note it needs to be started before running scenarios
- If everything passed: show the "Ready to dogfood!" message

## Error Recovery

| Issue | Solution |
|-------|----------|
| Branch not found | PRs may not have merged yet, or combined branch needs to be created. Ask in #proj-tatari2. |
| Docker not running | Start Docker Desktop and re-run `/tatari-qa:bootstrap` |
| Devserver not running | `cd {philo_path} && aws-vault exec tatari-ro -- make dev` |
| Beeswax creds missing | Set `BEESWAX_USERNAME` and `BEESWAX_PASSWORD` in `components/philo/config/local_creds.py`. Ask in #proj-tatari2 for access. |
| Sandbox settings missing | Add `BEESWAX_API = "https://tatarisbx.api.beeswax.com/rest/v2"` to `components/philo/config/local_settings.py` |
| Setup script fails | Check Beeswax credentials, network, and docker state. Re-run `/tatari-qa:bootstrap` to retry. |
| Dirty working tree | Stash or commit changes, then re-run `/tatari-qa:bootstrap` |
| philo-fe not running | `cd {philo_fe_path} && make dev` — can be started after bootstrap |
