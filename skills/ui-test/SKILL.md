---
name: ui-test-skill
description: >
  Autonomous UI testing agent that reads an Azure DevOps ticket (or pasted text),
  understands the acceptance criteria, and executes tests directly using the
  Playwright MCP server — no spec files or CLI needed. Agent navigates the live app,
  interacts with elements, takes screenshots, and reports pass/fail back to the ADO
  work item or PR. Uses a self-growing page-map.json to remember selectors across
  runs. Uses browser snapshot for element discovery.
  Trigger on: "test this ticket", "run UI tests for ticket", "test this feature",
  "generate UI test", "playwright test for ticket", "test AC on UI",
  "automated UI test", "test this on browser", "QA this ticket".
  Human approves before any comment is posted to ADO.

triggers:
  - test this ticket
  - run UI tests for ticket
  - test this feature on UI
  - generate UI test
  - playwright test for ticket
  - test AC on UI
  - automated UI test
  - QA this ticket
  - test this on browser
  - ui test agent
---

# UI Test Agent Skill

Autonomous agent that turns a ticket description into a running UI test executed
**directly via the Playwright MCP server**. No spec files, no CLI, no `npx playwright test`.
The agent controls the browser in real time using MCP tools, observes the live app,
and saves results permanently.

---

## ⚙️ ONE-TIME CONFIGURATION — Do This First

Before running any tests, set up your environment config file.
This is the **only place** you put your app URL and login credentials.

> **Playwright MCP server must already be running.** The agent uses it directly —
> no `npx playwright test`, no spec files, no Node.js CLI required.

### Step 0A — Create your env config

Create a file called `ui-test-agent.env.json` in your **repo root**
(add it to `.gitignore` immediately — never commit credentials):

```json
{
  "app": {
    "baseUrl": "https://your-staging-app.com",
    "loginUrl": "https://your-staging-app.com/login",
    "defaultPath": "/dashboard"
  },
  "credentials": {
    "username": "your-test-user@company.com",
    "password": "your-test-password",
    "authType": "form"
  },
  "ado": {
    "organization": "",
    "project": "",
    "pat": ""
  },
  "options": {
    "screenshotOnStep": true,
    "screenshotDir": ".github/outputs/ui-test/{workItemId}/screenshots",
    "snapshotDepth": 5
  }
}
```

**Where to put what:**

| Field | What to put | Example |
|---|---|---|
| `baseUrl` | Your deployed/staging app URL | `https://staging.myapp.com` |
| `loginUrl` | Full URL of your login page | `https://staging.myapp.com/auth/login` |
| `defaultPath` | Page to land on after login | `/dashboard` or `/home` |
| `username` | Test account email/username | `qauser@company.com` |
| `password` | Test account password | `TestPass123!` |
| `authType` | How login works | `form` / `sso` / `token` |
| `organization` | (Optional) Your ADO org URL | Leave blank if Azure CLI is already configured |
| `project` | (Optional) Your ADO project name | Leave blank if Azure CLI is already configured |
| `pat` | (Optional) ADO Personal Access Token | Leave blank if Azure CLI is already configured |
| `screenshotOnStep` | Take screenshot after each step | `true` recommended |
| `screenshotDir` | Where screenshots are saved | `.github/outputs/ui-test/{workItemId}/screenshots` (resolved at runtime) |

### Step 0B — Add to .gitignore

```bash
echo "ui-test-agent.env.json" >> .gitignore
echo ".tmp/" >> .gitignore
```

### Step 0C — Create page map (starts empty, grows automatically)

```bash
# Create the page map file — agent fills this over time
cat > page-map.json << 'EOF'
{
  "_readme": "Auto-maintained by UI Test Agent. Commit this file — it is not sensitive.",
  "_version": 1,
  "pages": {}
}
EOF
```

---

## Prerequisites

```bash
# Playwright MCP server — must already be running (user confirmed this)
# No additional Playwright installation needed for MCP-based execution

# Azure CLI + DevOps extension (for fetching tickets)
az login
az extension add --name azure-devops

# jq for JSON processing (used when fetching ADO tickets in terminal)
# macOS: brew install jq
# Linux: apt install jq
# Windows (Git Bash): winget install jqlang.jq
```

> **Windows users**: Run terminal commands in **Git Bash**, not CMD or PowerShell.

---

## AGENTS.md — Add This Section

Add to your existing `AGENTS.md` at repo root:

```markdown
## UI Test Agent Rules

- Never hardcode credentials in test files — always read from ui-test-agent.env.json
- Prefer data-testid selectors over CSS classes or XPath
- Use mcp_microsoft_pla_browser_snapshot before every click or fill — find selectors from live snapshot
- Always take a screenshot before and after each major action using mcp_microsoft_pla_browser_take_screenshot
- If login fails, stop and report — do not attempt to bypass auth
- Save passing test summaries and selectors to page-map.json
- Update page-map.json with any new selectors discovered during a run
- Maximum 3 snapshot attempts to locate a selector before reporting failure
- Never click "Delete", "Remove", or destructive actions on production URLs
- baseUrl must contain "staging", "test", "qa", "dev", or "localhost" — refuse prod URLs
- All browser interactions go through MCP tools — never generate standalone spec files for execution
```

---

## MCP Tools Reference

The Playwright MCP server exposes these tools. Use them directly — no CLI, no spec files.

| Tool | When to use |
|------|-------------|
| `mcp_microsoft_pla_browser_navigate` | Go to a URL |
| `mcp_microsoft_pla_browser_snapshot` | **Read page state** — do this before every interaction to find elements |
| `mcp_microsoft_pla_browser_take_screenshot` | Capture visual evidence |
| `mcp_microsoft_pla_browser_click` | Click a button, link, or element |
| `mcp_microsoft_pla_browser_fill_form` | Fill multiple form fields at once |
| `mcp_microsoft_pla_browser_type` | Type into a single field |
| `mcp_microsoft_pla_browser_select_option` | Choose a dropdown/select option |
| `mcp_microsoft_pla_browser_press_key` | Send a keyboard key (Enter, Tab, Escape, etc.) |
| `mcp_microsoft_pla_browser_hover` | Hover over an element (for tooltips/menus) |
| `mcp_microsoft_pla_browser_evaluate` | Run JavaScript on the page for assertions |
| `mcp_microsoft_pla_browser_wait_for` | Wait for text to appear/disappear or a fixed time |
| `mcp_microsoft_pla_browser_navigate_back` | Go back in browser history |
| `mcp_microsoft_pla_browser_tabs` | List open tabs / switch tabs |

> **Snapshot-first rule**: Always call `mcp_microsoft_pla_browser_snapshot` before
> any `click`, `fill_form`, or `type` call. Read the snapshot output to find the
> exact `target` reference for the element. Never guess element references.

---

## Step 1 — Load Config

Read `ui-test-agent.env.json` from the repo root using `read_file` tool.
Extract these values for use in subsequent steps:

```
BASE_URL       ← config.app.baseUrl
LOGIN_URL      ← config.app.loginUrl
DEFAULT_PATH   ← config.app.defaultPath
USERNAME       ← config.credentials.username
PASSWORD       ← config.credentials.password
AUTH_TYPE      ← config.credentials.authType
SCREENSHOT_DIR ← .github/outputs/ui-test/{WORK_ITEM_ID}/screenshots  (derived after ticket is fetched)
OUTPUT_DIR    ← .github/outputs/ui-test/{WORK_ITEM_ID}
```

> **Note on `TICKET_REPRO`**: Repro steps are extracted from the ADO field
> `Microsoft.VSTS.TCM.ReproSteps` (Bug work item type). Claude uses these in Step 4 to
> generate a test plan that directly reproduces the reported steps, not just validates the AC.

**Safety check** — refuse to run if `BASE_URL` does not contain one of:
`staging`, `test`, `qa`, `dev`, `localhost`, `127.0.0.1`

If the URL looks like production, ask the user to confirm before proceeding.

```
✅ Config loaded
   App:   {BASE_URL}
   Auth:  {AUTH_TYPE} as {USERNAME}
```

Directories are created **after** the work item ID is known (Step 2). Do NOT create them here yet.
Both `.tmp/ui-test/{WORK_ITEM_ID}/` and `.github/outputs/ui-test/{WORK_ITEM_ID}/` are created together in Step 2.

---

## Step 2 — Fetch Ticket from ADO

Run in terminal:

```bash
WORK_ITEM_ID=<NUMBER>   # set by user or prompt for it

az boards work-item show \
  --id "$WORK_ITEM_ID" \
  --output json > .tmp/ui-test/$WORK_ITEM_ID/ticket.json

TICKET_TITLE=$(jq -r '.fields["System.Title"]' .tmp/ui-test/$WORK_ITEM_ID/ticket.json)
TICKET_DESC=$(jq -r '.fields["System.Description"] // ""' .tmp/ui-test/$WORK_ITEM_ID/ticket.json \
  | sed 's/<[^>]*>//g' | tr -s ' \n')
TICKET_AC=$(jq -r '.fields["Microsoft.VSTS.Common.AcceptanceCriteria"] // ""' .tmp/ui-test/$WORK_ITEM_ID/ticket.json \
  | sed 's/<[^>]*>//g' | tr -s ' \n')
TICKET_REPRO=$(jq -r '.fields["Microsoft.VSTS.TCM.ReproSteps"] // ""' .tmp/ui-test/$WORK_ITEM_ID/ticket.json \
  | sed 's/<[^>]*>//g' | tr -s ' \n')
TICKET_TYPE=$(jq -r '.fields["System.WorkItemType"]' .tmp/ui-test/$WORK_ITEM_ID/ticket.json)

echo "✅ [$TICKET_TYPE] $TICKET_TITLE"
```

If the user pastes ticket text manually instead, capture it as `TICKET_DESC`, `TICKET_AC`,
and `TICKET_REPRO`. Set `WORK_ITEM_ID="manual"`.

Save summary to `.tmp/ui-test/{WORK_ITEM_ID}/ticket-summary.txt` — include all four fields:
```
TITLE: {TICKET_TITLE}
TYPE:  {TICKET_TYPE}
DESCRIPTION: {TICKET_DESC}
REPRO STEPS: {TICKET_REPRO}
ACCEPTANCE CRITERIA: {TICKET_AC}
```

Then create the per-run directories (both temp and output):
```bash
mkdir -p .tmp/ui-test/$WORK_ITEM_ID .github/outputs/ui-test/$WORK_ITEM_ID/screenshots
```

For PowerShell:
```powershell
New-Item -ItemType Directory -Force -Path ".tmp/ui-test/$WORK_ITEM_ID", ".github/outputs/ui-test/$WORK_ITEM_ID/screenshots" | Out-Null
```

---

## Step 3 — Load Page Map

Read `page-map.json` using `read_file`. Extract known pages and their element selectors.
These will be used as the first-choice selectors before falling back to snapshot discovery.

```
✅ Page map loaded
   Known pages: {comma-separated page keys or "none yet"}
```

If `page-map.json` does not exist, create it:
```json
{"_readme":"Auto-maintained by UI Test Agent","_version":1,"pages":{}}
```

---

## Step 4 — Analyze Ticket and Generate Test Plan

**[AGENT STEP — Claude reads ticket + page map and produces a structured MCP test plan]**

Claude receives:
- Ticket title, description, acceptance criteria
- **Repro steps** (from `Microsoft.VSTS.TCM.ReproSteps`) — used as the primary sequence for Bug tickets
- App base URL and login URL
- All known pages and selectors from page-map.json

For **Bug** tickets: generate test steps that follow the repro steps exactly, then assert
the acceptance criteria is met (i.e. the bug is fixed). For other work item types, derive
steps from the description and acceptance criteria.

Claude outputs a JSON test plan that maps every step to a specific MCP tool call.
Save to `.tmp/ui-test/{WORK_ITEM_ID}/test-plan.json`:

> Substitute `{WORK_ITEM_ID}` with the actual ID in all `screenshotLabel` paths inside the plan.

```json
{
  "testName": "reports-date-filter-last-30-days",
  "ticketId": "12345",
  "ticketTitle": "Date filter on Reports page — Last 30 days option",
  "requiresLogin": true,
  "steps": [
    {
      "id": 1,
      "description": "Navigate to Reports page",
      "mcpTool": "mcp_microsoft_pla_browser_navigate",
      "params": { "url": "{BASE_URL}/reports" },
      "screenshotLabel": "navigate-reports",
      "pageMapKey": "reports"
    },
    {
      "id": 2,
      "description": "Take snapshot to find date filter element",
      "mcpTool": "mcp_microsoft_pla_browser_snapshot",
      "params": {},
      "note": "Read output to find target reference for date filter dropdown"
    },
    {
      "id": 3,
      "description": "Click Date Filter dropdown",
      "mcpTool": "mcp_microsoft_pla_browser_click",
      "params": {
        "target": "<resolved from snapshot>",
        "element": "Date filter dropdown"
      },
      "screenshotLabel": "date-filter-open",
      "pageMapKey": "reports.dateFilter",
      "selectorKnown": false
    },
    {
      "id": 4,
      "description": "Select Last 30 days option",
      "mcpTool": "mcp_microsoft_pla_browser_click",
      "params": {
        "target": "text=Last 30 days",
        "element": "Last 30 days option"
      },
      "screenshotLabel": "last-30-selected",
      "pageMapKey": "reports.dateFilterOption_last30",
      "selectorKnown": false
    },
    {
      "id": 5,
      "description": "Assert results table is visible",
      "mcpTool": "mcp_microsoft_pla_browser_snapshot",
      "params": {},
      "assertion": "Results table must appear in snapshot output",
      "screenshotLabel": "results-visible",
      "pageMapKey": "reports.resultsTable"
    }
  ],
  "estimatedDuration": "45s"
}
```

**Rules for the test plan:**
- Every interaction step (click, fill, type) must be preceded by a snapshot step
- Use `selectorKnown: true` only when the selector exists in page-map.json
- Login is always the first executed action when `requiresLogin: true`
- Assertions use either a snapshot step (check element appears in output) or
  `mcp_microsoft_pla_browser_evaluate` for JavaScript checks

---

## Step 5 — Review Test Plan (Human Gate)

Display the test plan summary to the user:

```
━━━ Test Plan Review ━━━
  Test name:      {testName}
  Ticket:         #{ticketId} — {ticketTitle}
  Steps:          {count}
  Requires login: {requiresLogin}
  Est. duration:  {estimatedDuration}

Steps:
  [1] NAVIGATE        — Navigate to Reports page
  [2] SNAPSHOT        — Find date filter element
  [3] CLICK           — Click Date Filter dropdown  [discover via snapshot]
  [4] CLICK           — Select Last 30 days         [discover via snapshot]
  [5] SNAPSHOT/ASSERT — Assert results table visible [discover via snapshot]
━━━━━━━━━━━━━━━━━━━━━━━
```

**Ask the user**: "Approve this test plan? [Y/n]"

- If **n**: stop and let the user edit `.tmp/ui-test/{WORK_ITEM_ID}/test-plan.json`, then re-run from Step 5.
- Output directory for this run: `.github/outputs/ui-test/{WORK_ITEM_ID}/`
- If **Y**: proceed to Step 6.

---

## Step 6 — Execute Test via MCP Tools

**[AGENT STEP — Claude directly calls MCP tools to execute each step]**

This is the core execution loop. Claude calls MCP tools in sequence according to
the test plan. No spec files are generated. The browser is controlled live.

### 6A — Login (if requiresLogin: true)

```
mcp_microsoft_pla_browser_navigate  →  {LOGIN_URL}
mcp_microsoft_pla_browser_snapshot  →  find username and password fields
mcp_microsoft_pla_browser_fill_form →  fill username={USERNAME}, password={PASSWORD}
mcp_microsoft_pla_browser_take_screenshot → .github/outputs/ui-test/{WORK_ITEM_ID}/screenshots/00-login-page.png
mcp_microsoft_pla_browser_click     →  submit button (from snapshot)
mcp_microsoft_pla_browser_wait_for  →  text expected on dashboard / default path
mcp_microsoft_pla_browser_take_screenshot → .github/outputs/ui-test/{WORK_ITEM_ID}/screenshots/01-post-login.png
```

If login fails (dashboard text not found in snapshot after submit):
- Take a screenshot, report the failure, and **stop immediately**.

### 6B — Execute Each Test Step

For each step in the test plan, execute the corresponding MCP tool call.

**Selector resolution order:**
1. `pageMapKey` exists in page-map.json → use stored selector directly
2. `selectorKnown: false` → call `mcp_microsoft_pla_browser_snapshot` first,
   read output, identify the element reference, then proceed
3. If element not found in snapshot after 3 attempts → report failure with screenshot

**After each step with `screenshotLabel`:**
```
mcp_microsoft_pla_browser_take_screenshot → .github/outputs/ui-test/{WORK_ITEM_ID}/screenshots/step-{id}-{screenshotLabel}.png
```

**Assertion steps:**
- Snapshot-based: call `mcp_microsoft_pla_browser_snapshot`, check that the
  asserted element or text appears in the output. If absent → test FAILED.
- JS-based: call `mcp_microsoft_pla_browser_evaluate` with the check function.

### 6C — Track Results

Maintain a running result object:

```json
{
  "testName": "reports-date-filter-last-30-days",
  "result": "PASSED",
  "stepsTotal": 5,
  "stepsPassed": 5,
  "stepsFailed": 0,
  "failedSteps": [],
  "screenshots": ["step-01-navigate-reports.png", "..."],
  "duration": "38s"
}
```

Save to `.tmp/ui-test/{WORK_ITEM_ID}/results.json` when execution ends.
Also copy to `.github/outputs/ui-test/{WORK_ITEM_ID}/results.json` for persistence.

---

## Step 7 — Update Page Map with Discovered Selectors

**[AGENT STEP — Claude updates page-map.json with selectors found during execution]**

For every step where `selectorKnown: false` and execution succeeded, record the
selector that worked. Update `page-map.json`:

```json
{
  "_readme": "Auto-maintained by UI Test Agent. Commit this file — it is not sensitive.",
  "_version": 1,
  "pages": {
    "reports": {
      "url": "/reports",
      "lastUpdated": "2025-04-29",
      "elements": {
        "dateFilter": "#date-filter-dropdown",
        "dateFilterOption_last30": "text=Last 30 days",
        "resultsTable": "[data-testid='results-table']"
      }
    }
  }
}
```

Use `replace_string_in_file` or `create_file` to write the updated page-map.json.
Commit this file — it contains no secrets.

---

## Step 8 — Human Review Gate (Before Posting to ADO)

Display the full result summary and the exact comment that would be posted:

```
━━━ Review Gate — Preview ADO Comment ━━━

## {EMOJI} UI Test Result — {RESULT}
**Ticket:** #{WORK_ITEM_ID} — {TICKET_TITLE}
**Test:** `{testName}`
**App URL:** {BASE_URL}
**Result:** {RESULT}
**Duration:** {duration}

### Steps executed
- [1] NAVIGATE — Navigate to Reports page ✅
- [2] SNAPSHOT — Find date filter element ✅
- [3] CLICK    — Click Date Filter dropdown ✅
- [4] CLICK    — Select Last 30 days ✅
- [5] ASSERT   — Results table visible ✅

### Screenshots
step-01-navigate-reports.png, step-03-date-filter-open.png, ...

---
*Generated by UI Test Agent | Reviewed by: {git user email} | {timestamp}*

──────────────────────────────────────────────────────
```

**Ask the user**: "Post this comment to ADO Work Item #{WORK_ITEM_ID}? [Y/n]"

If approved and `WORK_ITEM_ID` is not "manual", run in terminal:
```bash
az boards work-item comment add \
  --id "$WORK_ITEM_ID" \
  --text "$ADO_COMMENT" \
  --output none
```

---

## Step 9 — Save Passing Test as Spec (Optional, for CI)

If the test **PASSED** and the user wants it in CI, ask:
"Save this test as a Playwright spec in .github/outputs/ui-test/{WORK_ITEM_ID}/ for future CI runs? [Y/n]"

If approved, generate a `.spec.ts` file from the executed steps:

```typescript
// Auto-generated by UI Test Agent
// Ticket: #{WORK_ITEM_ID} — {TICKET_TITLE}
// Generated: {TIMESTAMP}
// Executed via Playwright MCP — this spec is for CI reproduction only

import { test, expect } from '@playwright/test';
import * as fs from 'fs';

const config = JSON.parse(fs.readFileSync('ui-test-agent.env.json', 'utf-8'));
const BASE_URL  = config.app.baseUrl;
const LOGIN_URL = config.app.loginUrl;
const USERNAME  = config.credentials.username;
const PASSWORD  = config.credentials.password;

test('{testName}', async ({ page }) => {

  // Login
  await page.goto(LOGIN_URL);
  await page.fill('[name="username"], [name="email"], #username, #email', USERNAME);
  await page.fill('[name="password"], #password', PASSWORD);
  await page.click('[type="submit"]');
  await page.waitForURL(`**${config.app.defaultPath}**`, { timeout: 15000 });

  // Step 1 — Navigate to /reports
  await page.goto(`${BASE_URL}/reports`);
  await page.screenshot({ path: '.github/outputs/ui-test/{WORK_ITEM_ID}/screenshots/step-01-navigate-reports.png' });

  // Step 3 — Click Date Filter dropdown
  await page.click('#date-filter-dropdown');   // selector discovered during MCP run
  await page.screenshot({ path: '.github/outputs/ui-test/{WORK_ITEM_ID}/screenshots/step-03-date-filter-open.png' });

  // Step 4 — Select Last 30 days
  await page.click('text=Last 30 days');
  await page.screenshot({ path: '.github/outputs/ui-test/{WORK_ITEM_ID}/screenshots/step-04-last-30-selected.png' });

  // Step 5 — Assert results table visible
  await expect(page.locator("[data-testid='results-table']")).toBeVisible({ timeout: 10000 });
  await page.screenshot({ path: '.github/outputs/ui-test/{WORK_ITEM_ID}/screenshots/step-05-results-visible.png' });

});
```

Selectors used in the spec must be the exact ones discovered during the MCP run
and recorded in page-map.json. Save to `.github/outputs/ui-test/{WORK_ITEM_ID}/{testName}.spec.ts`.

Clean up temp files:
```bash
rm -rf .tmp/ui-test/$WORK_ITEM_ID
```

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  UI Test Agent — Run Complete
  Result:   {RESULT}
  Ticket:   #{WORK_ITEM_ID}
  Spec:     .github/outputs/ui-test/{WORK_ITEM_ID}/{testName}.spec.ts (if saved)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Modes Reference

| Mode | What it does | When to use |
|------|-------------|-------------|
| **Full run** | Steps 1→9 in order | Normal ticket testing |
| **Plan only** | Steps 1→5, stop | Review plan before executing |
| **Re-execute** | Steps 6→8 | Re-run using existing test-plan.json |
| **Update map only** | Step 7 only | After manually discovering selectors |
| **Post result only** | Step 8 only | Post existing result to ADO |
| **Generate spec only** | Step 9 only | Convert MCP run results to CI spec |

---

## Selector Strategy (How Agent Picks Elements)

The agent uses this priority order to find UI elements on every step:

```
1. page-map.json (pageMapKey exists)  →  stored CSS / data-testid from prior run  (fastest)
2. mcp_microsoft_pla_browser_snapshot →  read live accessibility tree, find element ref
3. data-testid attribute              →  [data-testid="submit-btn"]   (most reliable)
4. ARIA role + accessible name        →  role=button[name="Apply"]
5. Visible text                       →  text=Last 30 days
6. CSS selector                       →  #date-filter, .my-button
```

**How snapshot-based discovery works:**

1. Call `mcp_microsoft_pla_browser_snapshot`
2. Read the returned accessibility tree
3. Identify the element matching the step description
4. Extract its `target` reference (e.g. `ref=s1e23` or a CSS selector)
5. Pass that reference as `target` in the next `click`, `fill_form`, or `type` call
6. Record the discovered selector in page-map.json for future runs

**Tip for your team**: Add `data-testid` attributes to key UI elements as you build them.
This allows the agent to skip snapshot discovery entirely and execute faster.

---

## File Structure After Setup

```
your-repo/
├── ui-test-agent.env.json          ← YOUR CONFIG (gitignored, never commit)
├── page-map.json                   ← Selector registry (commit this)
├── AGENTS.md                       ← Includes UI test rules
├── .tmp/
│   └── ui-test/
│       └── {workItemId}/       ← Temp files scoped to this ticket (gitignored)
│           ├── ticket.json
│           ├── ticket-summary.txt
│           ├── test-plan.json
│           └── results.json
├── .github/
│   └── outputs/
│       └── ui-test/                ← Final result files (gitignored)
│           └── {workItemId}/       ← One folder per ticket/run
│               ├── screenshots/    ← All screenshots from the run
│               │   ├── 00-login-page.png
│               │   └── step-01-*.png
│               ├── results.json
│               └── {testName}.spec.ts
└── .gitignore                      ← Must include ui-test-agent.env.json
```

> **Neither `.tmp/` nor `.github/outputs/ui-test/` should be committed** — both are gitignored.
> Only commit `page-map.json`. Each ticket run gets its own isolated folder pair:
> `.tmp/ui-test/{workItemId}/` (temp, deleted after run) and `.github/outputs/ui-test/{workItemId}/`
> (persistent screenshots, results, and optional spec). Review passing specs and copy them
> to a permanent location (e.g. `tests/`) before committing.

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `Login failed` | Wrong credentials or URL | Check `loginUrl` and credentials in env.json |
| `Element not found in snapshot` | Element not rendered yet, or selector wrong | Call `mcp_microsoft_pla_browser_wait_for` before snapshot, or use `mcp_microsoft_pla_browser_take_screenshot` to visually inspect |
| `Snapshot shows wrong page` | Navigation didn't complete | Add `mcp_microsoft_pla_browser_wait_for` with expected page text after navigate |
| `ADO post failed` | PAT expired or missing permissions | Regenerate PAT with Work Items (read/write) scope |
| `Page map not growing` | Step 7 skipped | Always run Step 7 after each successful test |
| `MCP tool returns error` | Browser not open or MCP server not running | Confirm Playwright MCP server is running; use `mcp_microsoft_pla_browser_navigate` to open first URL |

---

## Quick Reference — Config File Location

```
WHERE TO PUT YOUR APP URL:
  ui-test-agent.env.json → "app" → "baseUrl"

WHERE TO PUT YOUR LOGIN URL:
  ui-test-agent.env.json → "app" → "loginUrl"

WHERE TO PUT YOUR CREDENTIALS:
  ui-test-agent.env.json → "credentials" → "username" / "password"

WHERE TO PUT YOUR ADO TOKEN:
  ui-test-agent.env.json → "ado" → "pat"

NEVER commit ui-test-agent.env.json to git.
ALWAYS commit page-map.json — it has no secrets.
```
