---

description: Execute YAML-driven campaign creation QA scenarios via Playwright MCP
---

# Campaign QA Command

Execute repeatable, YAML-defined campaign creation test scenarios against the Tatari2 campaign form using Playwright MCP browser automation.

## Usage

```
/tatari-qa:campaign-qa <scenario_name>
```

Where `<scenario_name>` is the filename (without `.yaml` extension) from `script/qa/campaign_creation/scenarios/`.

Example:
```
/tatari-qa:campaign-qa awareness_cpm_daily_prospecting
```

## Prerequisites

- Local dev environment running (`aws-vault exec tatari-ro -- make dev`)
- philo-fe running at `http://local.tatari.tools:3000`
- Playwright MCP browser available (Playwright tools: `mcp__playwright__*`)
- For Tier 2/3 scenarios: local data setup scripts must have been run:
  ```bash
  docker compose exec -it devserver poetry run python script/programmatic_segment_ingester.py ingest-beeswax-segments
  docker compose exec -it devserver poetry run python script/ops/setup_local_dev_for_beeswax_sandbox.py
  ```

## Process

### Step 0: Load and Parse Scenario

1. Read the scenario file:

   ```
   Read: script/qa/campaign_creation/scenarios/{scenario_name}.yaml
   ```

2. Parse all fields from the YAML. Compute runtime values:
   - **Campaign name**: Replace `{scenario}` with the scenario's `name` field, `{timestamp}` with current date/time in `YYYYMMDD-HHmmss` format
   - **Relative dates**: Convert `+Nd` format to absolute dates (e.g., `+7d` = 7 days from today). Format as `YYYY-MM-DD` for internal use.
   - **null values**: Skip that field / accept the form default

3. Print a summary of the scenario being executed:

   ```
   Executing scenario: {name}
   Description: {description}
   Tags: {tags}
   Goal: {goal.type} / KPI: {goal.kpi} / Target: {goal.target}
   Budget: ${budget.amount} {budget.type}
   Audience: {audience.strategy}
   Submit: {submit.action}
   ```

### Step 1: Navigate to Campaign Creation Form

Navigate the browser to the campaign creation page:

```
mcp__playwright__browser_navigate
  url: http://local.tatari.tools:3000/c/tatari_client_mode_2_debug/campaigns/new
```

**If navigation fails with `browserType.launchPersistentContext: Failed to launch`** (error log shows "Opening in existing browser session"), an existing Playwright Chromium instance is blocking. Kill it and retry:

```bash
pkill -f "Google Chrome for Testing.*mcp-chromium"
```

Then retry the `browser_navigate` call.

Wait for the page to load using `browser_wait_for` with text "Campaign Name". If the wait times out and the URL has redirected to `tatari.okta.com`, the user needs to authenticate:

1. Inform the user: "The browser redirected to Okta login. Please approve the MFA push notification on your phone."
2. Wait for the user to confirm login
3. Re-navigate to the campaign creation URL

Once the form loads, take an accessibility snapshot to verify the "Campaign Name" textbox is visible.

### Step 2: Fill Campaign Name

```
mcp__playwright__browser_fill_form
  textbox "Campaign Name": "{computed_campaign_name}"
```

### Step 3: Set Campaign Goal

The goal section has radio buttons. **CRITICAL: Click the CONTAINER element, NOT the radio input itself.** Styled labels intercept pointer events on radio inputs, causing TimeoutError.

Take a snapshot to find the goal radio containers:

```
mcp__playwright__browser_snapshot
```

**Goal type mapping:**

| YAML `goal.type` | Form Label to Click |
|-------------------|-------------------|
| `awareness` | Container for "Raise brand awareness" |
| `conversion` | Container for "Drive conversions or visits" |

Use `browser_click` on the container ref (the parent element wrapping the radio + label), NOT the radio ref.

**If `goal.type` is `awareness`:**
- Click the "Raise brand awareness" container
- The KPI will automatically be CPM (only option for awareness)

**If `goal.type` is `conversion`:**
- Click the "Drive conversions or visits" container
- Then select the KPI (see Step 4)

### Step 4: Set KPI

**Only needed for conversion goal type.** Awareness automatically uses CPM.

For conversion goals, the KPI options appear as radio buttons:

| YAML `goal.kpi` | Form Label |
|-----------------|------------|
| `cpa` | "Conversions (CPA)" |
| `cpv` | "Site visitors (CPV)" |
| `roas` | "Return on ad spend (ROAS)" - may be feature-flag gated |

Click the **container** for the desired KPI radio button (same container-click pattern as Step 3).

### Step 5: Set Conversion Event (if needed)

**Skip this step if `goal.conversion_event` is null.**

The conversion event is a React-Select dropdown. React-Select options do NOT appear in accessibility snapshots.

**Interaction pattern:**

1. Take snapshot to find the conversion event combobox:
   ```
   mcp__playwright__browser_snapshot
   ```

2. Click the combobox to open it:
   ```
   mcp__playwright__browser_click
     ref: [combobox ref]
   ```

3. **If `goal.conversion_event` is `"first"`:** The first option is already focused when the dropdown opens. Press Enter directly (do NOT press ArrowDown first — that would skip to the second option):
   ```
   mcp__playwright__browser_press_key
     key: Enter
   ```

4. **If `goal.conversion_event` is a specific name** (e.g., `"Purchase"`): Type to filter, then select the focused result:
   ```
   mcp__playwright__browser_type
     ref: [combobox ref]
     text: "{event_name}"
   ```
   Wait 500ms for filtering, then press Enter to select the first matching result (already focused):
   ```
   mcp__playwright__browser_press_key
     key: Enter
   ```

### Step 6: Set KPI Target Value

Fill the target value input. The label changes based on KPI:

| KPI | Target Label |
|-----|-------------|
| `cpm` | "Target: CPM" |
| `cpa` | "Target: CPA" |
| `cpv` | "Target: CPV" |
| `roas` | "Target: ROAS" |

```
mcp__playwright__browser_fill_form
  textbox "Target: {KPI}": "{goal.target}"
```

If the exact label isn't found in the snapshot, look for a spinbutton or textbox near the KPI section.

### Step 7: Set Budget Amount

```
mcp__playwright__browser_fill_form
  textbox "Budget": "{budget.amount}"
```

**Important:** After filling the budget, the form auto-splits the amount 80/20 between Streaming TV and Premium Online Video on blur. No need to set channel split manually unless `channel_split` specifies custom percentages.

### Step 8: Set Budget Type

**Default is "Daily".** Only change if `budget.type` is `lifetime`.

| YAML `budget.type` | Form Label |
|--------------------|------------|
| `daily` | "Daily" (default, may not need clicking) |
| `lifetime` | "Lifetime" |

Click the **container** for the budget type radio (same container-click pattern).

**If `budget.type` is `lifetime`:** An end date field will appear and becomes required (see Step 10).

### Step 9: Set Start Date

1. Take snapshot to find the start date field:
   ```
   mcp__playwright__browser_snapshot
   ```

2. Click the start date textbox to open the calendar picker:
   ```
   mcp__playwright__browser_click
     ref: [start date textbox ref]
   ```

3. The calendar opens showing the current month. Navigate to the target month if needed:
   - Use "Move one month forward" / "Move one month back" buttons
   - Calculate how many months to navigate from current month to target month

4. Click the target day cell:
   ```
   mcp__playwright__browser_click
     ref: [cell with the target day number]
   ```

**Date calculation:** If `budget.start_date` is `+7d`, compute today + 7 days. If it's `YYYY-MM-DD`, use that date directly.

### Step 10: Set End Date (if needed)

**Skip this step if `budget.end_date` is null.**

End date is **required for lifetime budgets** and optional for daily.

Follow the same calendar interaction pattern as Step 9 but for the end date field:

1. Click the end date textbox
2. Navigate months if needed
3. Click the target day cell

**Note:** The end date calendar correctly disables dates before the start date.

### Step 11: Set Audience Strategy

**CRITICAL: Do NOT use `browser_click` with a snapshot ref for audience radios.** Clicking via ref can silently fail (radio appears clicked but doesn't actually check), and the resulting snapshot can exceed 450K characters due to the interests tree expanding in the DOM.

**Always use `browser_run_code` for audience strategy selection and verification:**

| YAML `audience.strategy` | Text to Click |
|--------------------------|---------------|
| `prospecting` | "Demographic and interests targeting" |
| `lal` | "Look-a-like based on your existing customers" |
| `retargeting` | "Retargeting existing visitors" |

```javascript
mcp__playwright__browser_run_code
  code: async (page) => {
    // Click the audience strategy label text
    await page.locator('text=Demographic and interests targeting').first().click();
    await page.waitForTimeout(500);

    // Verify selection
    const radios = await page.evaluate(() => {
      const results = [];
      document.querySelectorAll('input[type="radio"]').forEach(r => {
        const text = r.parentElement?.textContent?.substring(0, 60) || '';
        if (text.includes('Demographic') || text.includes('Look-a-like') || text.includes('Retargeting')) {
          results.push({ text: text.substring(0, 40), checked: r.checked });
        }
      });
      return results;
    });
    return radios;
  }
```

Replace `'Demographic and interests targeting'` with the appropriate text from the table above based on the scenario's `audience.strategy`.

**Do NOT take a `browser_snapshot` after this step** — the expanded demographics/interests panel will overflow the context window. Use `browser_run_code` for any subsequent audience-section interactions.

### Step 12: Configure Audience Details

#### For Prospecting (`audience.strategy: prospecting`)

The demographics panel opens with defaults:
- Gender: All (default)
- Age: 18-74 (default range)
- Income: full range (default)
- Interests: empty (default)

**Demographics configuration:**

If `audience.demographics.gender` is not `all`:

| Value | Form Label |
|-------|------------|
| `all` | "All" (default) |
| `female` | "Female" |
| `male` | "Male" |

Click the **container** for the gender radio.

**Age and Income sliders:** If `audience.demographics.age_min`, `age_max`, `income_min`, or `income_max` are specified, use `browser_run_code` to set slider values programmatically:

```javascript
// Example for age slider
async (page) => {
  // Find the age slider inputs and set values
  const sliders = await page.locator('[role="slider"]').all();
  // Set min/max values as needed
}
```

**CRITICAL: Do NOT expand the interests tree.** Leave `audience.interests` empty to avoid context overflow from the massive hierarchical tree. The interests tree has hundreds of expandable groups that can exceed tool output limits.

#### For Look-a-like (`audience.strategy: lal`)

After selecting LAL, the form shows provider options. Accept defaults unless the scenario specifies otherwise.

#### For Retargeting (`audience.strategy: retargeting`)

After selecting retargeting:

1. The form shows "Add audience" / "Add exclusion" buttons and funnel categories
2. For each entry in `audience.retargeting.events`:
   - Click the appropriate funnel category button (upper/mid/lower/conversion)
   - Select the event (similar React-Select pattern as conversion events)
3. Set lookback days if specified (spinbutton)

**Note:** Retargeting requires conversion events to be set up in the local DB (setup scripts).

### Step 13: Configure Screen Targeting (if specified)

**Skip if `screen_targeting` is null or all values are `true` (defaults).**

If any screen type needs to be unchecked:

```
mcp__playwright__browser_fill_form
  checkbox "TV": {screen_targeting.tv}
  checkbox "Mobile": {screen_targeting.mobile}
  checkbox "Computer": {screen_targeting.computer}
```

### Step 14: Attach Creatives

1. Take snapshot to find the "Attach creatives" button:
   ```
   mcp__playwright__browser_snapshot
   ```

2. Click "Attach creatives" to open the drawer:
   ```
   mcp__playwright__browser_click
     ref: [Attach creatives button ref]
   ```

3. Wait for the drawer to load, then take snapshot:
   ```
   mcp__playwright__browser_snapshot
   ```

4. **If `creatives.select` is `first_available`:**
   - Find the first enabled checkbox (under "Ready to air" section)
   - Check it:
     ```
     mcp__playwright__browser_fill_form
       checkbox [first creative name]: true
     ```

5. **If `creatives.select` is a list of names:**
   - Check each named creative's checkbox

6. Click the footer "Attach creatives" button to confirm. **IMPORTANT:** The footer button's ref goes stale after checking checkboxes (React re-renders the footer). Always use `browser_run_code` instead of `browser_click` with a ref:

   ```javascript
   mcp__playwright__browser_run_code
     code: async (page) => {
       const buttons = await page.getByRole('button', { name: 'Attach creatives' }).all();
       // The footer confirm button is the last "Attach creatives" button
       await buttons[buttons.length - 1].click();
       return `Clicked footer button (${buttons.length} total)`;
     }
   ```

7. **Close the creatives drawer** before continuing to prevent snapshot overflow in subsequent steps. Use `browser_run_code`:

   ```javascript
   mcp__playwright__browser_run_code
     code: async (page) => {
       const closeBtn = page.getByRole('button', { name: 'Close creatives drawer' });
       if (await closeBtn.isVisible().catch(() => false)) {
         await closeBtn.click();
         await page.waitForTimeout(500);
       }
       return 'Drawer closed';
     }
   ```

### Step 15: Set Click-Through URL

The click-through URL is a React-Select dropdown.

1. Take snapshot to find the click-through URL combobox:
   ```
   mcp__playwright__browser_snapshot
   ```

2. Click the combobox:
   ```
   mcp__playwright__browser_click
     ref: [click-through URL combobox ref]
   ```

3. **If `click_through_url.select` is `first`:** The first option is already focused when the dropdown opens. Press Enter directly (do NOT press ArrowDown first — that skips to the second option):
   ```
   mcp__playwright__browser_press_key
     key: Enter
   ```

4. **If `click_through_url.select` is a specific URL text:**
   ```
   mcp__playwright__browser_type
     ref: [combobox ref]
     text: "{url_text}"
   ```
   Wait 500ms for filtering, then press Enter to select the first match (already focused):
   ```
   mcp__playwright__browser_press_key
     key: Enter
   ```

### Step 16: Configure IP Filtering (if specified)

**Skip if `ip_filtering` is null.**

Look for the "Smart IP Filtering" toggle/radio and set accordingly.

### Step 17: Configure Location (if specified)

**Skip if `location` is null (defaults to United States / all).**

Location is a searchable multi-select. If specific states or DMAs are needed, type to search and select.

### Step 18: Submit the Form

Take a final snapshot to verify the form state before submission:

```
mcp__playwright__browser_snapshot
```

**IMPORTANT: Save the snapshot to a file to avoid context overflow:**

```
mcp__playwright__browser_snapshot
  filename: /tmp/campaign-qa-pre-submit.yaml
```

Then read just the relevant portions if needed.

#### Draft Save (`submit.action: draft`)

Click "Save changes" button:

```
mcp__playwright__browser_click
  ref: [Save changes button ref]
```

#### Full Submit (`submit.action: full`)

Click "Finish" button:

```
mcp__playwright__browser_click
  ref: [Finish button ref]
```

### Step 19: Verify Assertions

Wait 2-3 seconds for navigation, then verify:

#### `assertions.redirect_to_campaigns_list: true`

Check that the URL now contains `/campaigns` (not `/campaigns/new`):

```javascript
mcp__playwright__browser_run_code
  code: async (page) => { return page.url(); }
```

Verify the URL matches `http://local.tatari.tools:3000/c/tatari_client_mode_2_debug/campaigns` (without `/new`).

#### `assertions.no_validation_errors: true`

If still on the form (not redirected), take a snapshot and check for validation error indicators:
- Look for `img "Attention"` elements
- Look for error text like "Please choose a conversion event", "Budget is required", etc.

### Step 20: Report Results

Output a structured result summary:

```
Campaign QA Result
==================
Scenario: {name}
Tags: {tags}
Status: PASS / FAIL

Campaign Name: {computed_campaign_name}
Goal: {goal.type} / {goal.kpi}
Budget: ${budget.amount} {budget.type}
Audience: {audience.strategy}
Submit Action: {submit.action}

Assertions:
  - Redirect to campaigns list: PASS / FAIL
  - No validation errors: PASS / FAIL

{If FAIL: include error details, screenshot path, or snapshot excerpt}
```

## Critical Interaction Patterns Reference

These patterns are **essential** for reliable Playwright MCP automation of this form:

| Element | Pattern | Why |
|---------|---------|-----|
| Radio buttons (goal, budget) | Click **container** element via ref, NOT the radio input | Styled labels intercept pointer events, causing TimeoutError |
| Radio buttons (audience) | Use **`browser_run_code`** with text-based locator — never use ref-based click | Ref clicks silently fail; snapshot causes 450K+ char overflow from interests tree |
| React-Select dropdowns | Click combobox → Enter (for first option) or type + Enter (for specific option) | First option is auto-focused; ArrowDown skips to second. Options don't appear in snapshots |
| Date pickers | Click textbox, navigate months if needed, click day cell | Calendar is fully accessible |
| Creatives drawer | Click opener → check checkboxes → use **`browser_run_code`** to click footer confirm button | Footer refs go stale after checkbox interaction; two buttons with same label |
| Budget | Fill amount, auto-splits 80/20 on blur | No need to set channel split manually |
| Interests tree | **DEFAULT TO EMPTY - never expand** | Context overflow risk — hundreds of nested items |
| Browser launch | Kill existing Chromium before navigating if launch fails | Previous MCP session can block the user data directory |

## Context Window Management Rules

**CRITICAL:** Follow these rules to avoid context overflow:

1. **Never take full-page snapshots after expanding interests/audience sections** - the tree is enormous
2. **Use `browser_run_code` + `page.evaluate()` for targeted DOM queries** instead of full snapshots when checking specific values
3. **Close the creatives drawer before taking the next snapshot** - the drawer adds significant snapshot size
4. **Save large snapshots to file** using the `filename` parameter, then read specific sections with Grep
5. **Between major form sections**, take targeted snapshots rather than full-page ones if the form has grown complex

## Error Recovery

If a step fails:

1. **Browser launch failure** (`Failed to launch the browser process`): An existing Playwright Chromium session is running. Kill it with `pkill -f "Google Chrome for Testing.*mcp-chromium"` and retry.
2. **Okta redirect on navigation**: The user is not authenticated. Ask them to approve the Okta MFA push notification, then re-navigate.
3. **TimeoutError on radio click**: You're clicking the radio input, not the container. Take a snapshot, find the parent container element, and click that instead.
4. **Audience radio click silently fails**: The ref-based click succeeded but the radio didn't check. Use `browser_run_code` with text-based locator (`page.locator('text=...')`) instead.
5. **Audience snapshot causes context overflow** (450K+ chars): The interests tree expanded in the DOM. Never take `browser_snapshot` after clicking audience radios — use `browser_run_code` + `page.evaluate()` instead.
6. **Creatives footer button ref not found**: React re-rendered the footer after checkbox interaction, invalidating refs. Use `browser_run_code` to find and click the button by role.
7. **React-Select selects wrong option**: The first option is auto-focused on open. Pressing ArrowDown skips to the second. For `select: first`, press Enter directly without ArrowDown.
8. **React-Select shows no options**: The local DB may not have the required data. Check if setup scripts have been run.
9. **Calendar navigation off-by-one**: Count months carefully. The calendar shows the current month by default.
10. **Creatives drawer empty**: No "Ready to air" creatives exist. This is a data issue, not an automation issue.
11. **Validation errors on save**: Read the error messages carefully. Usually means a required field was missed. Take a snapshot of the full form and identify the missing field.
12. **Context overflow**: If any snapshot response is truncated, save to file and grep for specific elements.
