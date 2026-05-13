---
name: smoke-test-skill
description: >
  Post-release smoke testing skill that replaces 1-2 days of manual QA effort.
  Reads smoke test scenarios from a CSV file or Azure DevOps tickets/test plans,
  executes every scenario against the live app via the Playwright MCP server,
  and produces a structured pass/fail report with screenshots. Designed for
  production-safe, repeatable smoke testing after every release.

  Trigger on: "run smoke tests", "smoke test after release", "post-release QA",
  "run release validation", "execute smoke scenarios", "smoke test from CSV",
  "run all smoke tests", "validate release", "post-deployment smoke test",
  "smoke test the release", "QA smoke suite", "run regression smoke",
  "check release health", "smoke test suite", "automated smoke testing".

  Always use this skill when the user mentions smoke testing, post-release validation,
  or wants to replace manual QA test execution with automation — even if they say
  "just run through the smoke scenarios" or "check if the release broke anything".
---

# Smoke Test Skill

Autonomous post-release smoke testing skill. Reads test scenarios from **CSV** or
**Azure DevOps** (tickets, test plans, or test suites), executes them live against
the app via the Playwright MCP server, and delivers a structured HTML + JSON report.

Replaces 1–2 days of manual QA smoke testing with a fully automated, repeatable run.

---

## ⚙️ ONE-TIME SETUP — Do This First

### Step 0A — Create environment config

Create `.github/inputs/smoke-testing/smoke-test-skill.env.json` (add to `.gitignore`):

```json
{
  "app": {
    "baseUrl": "https://your-app.com",
    "loginUrl": "https://your-app.com/login",
    "defaultPath": "/dashboard"
  },
  "credentials": {
    "username": "smoke-test-user@company.com",
    "password": "SmokeTestPass123!",
    "authType": "form"
  },
  "ado": {
    "organization": "https://dev.azure.com/your-org",
    "project": "YourProject",
    "pat": ""
  },
  "smoke": {
    "stopOnFirstFailure": false,
    "maxParallelScenarios": 1,
    "defaultTimeoutMs": 10000,
    "screenshotOnPass": true,
    "screenshotOnFailure": true,
    "headless": false,
    "outputDir": ".github/outputs/smoke-test/{runId}",
    "reportTitle": "Post-Release Smoke Test Report"
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
| `stopOnFirstFailure` | Halt entire run on first failure | `false` recommended for full smoke coverage |
| `screenshotOnPass` | Take screenshots during passing scenario steps | `true` recommended for first runs; set `false` to speed up large suites |
| `outputDir` | Where results are saved | `.github/outputs/smoke-test/{runId}` (resolved at runtime) |

### Step 0B — Gitignore

```bash
echo ".github/inputs/smoke-testing/smoke-test-skill.env.json" >> .gitignore
echo ".tmp/smoke/" >> .gitignore
echo ".github/outputs/smoke-test/" >> .gitignore
```

### Step 0C — Create page map (shared with UI test skill if both are used)

```bash
cat > page-map.json << 'EOF'
{"_readme":"Auto-maintained by Test Skills. Commit this file — it is not sensitive.","_version":1,"pages":{}}
EOF
```

### Step 0D — Add to AGENTS.md

Add to your existing `AGENTS.md` at repo root:

```markdown
## Smoke Test Skill Rules

- Never hardcode credentials in test files — always read from `.github/inputs/smoke-testing/smoke-test-skill.env.json`
- Prefer data-testid selectors over CSS classes or XPath
- Call mcp_microsoft_pla_browser_snapshot before every click, fill, or type — resolve element targets from live snapshot output
- Always take a screenshot before and after each major action using mcp_microsoft_pla_browser_take_screenshot
- If login fails, stop the entire run — do not attempt any scenarios
- Save passing selectors to page-map.json after every run
- Maximum 3 snapshot attempts to locate an element before marking the step as FAILED
- Never click "Delete", "Remove", or destructive actions on production URLs
- baseUrl must contain "staging", "test", "qa", "dev", "uat", "preview", or "localhost" — refuse prod URLs
- All browser interactions go through MCP tools — never generate standalone spec files for execution
- On failure: screenshot first, then record failure reason, then continue (unless stopOnFirstFailure is true)
```

---

## MCP Tools Reference

The Playwright MCP server exposes these tools. Use them directly — no CLI, no spec files.

| Tool | When to use |
|------|-------------|
| `mcp_microsoft_pla_browser_navigate` | Go to a URL |
| `mcp_microsoft_pla_browser_snapshot` | **Read page state** — call before EVERY interaction to find element targets |
| `mcp_microsoft_pla_browser_take_screenshot` | Capture visual evidence at key steps |
| `mcp_microsoft_pla_browser_click` | Click a button, link, or element |
| `mcp_microsoft_pla_browser_fill_form` | Fill multiple form fields at once (preferred over type for forms) |
| `mcp_microsoft_pla_browser_type` | Type into a single field |
| `mcp_microsoft_pla_browser_select_option` | Choose a dropdown/select option |
| `mcp_microsoft_pla_browser_press_key` | Send a keyboard key (Enter, Tab, Escape, etc.) |
| `mcp_microsoft_pla_browser_hover` | Hover over an element (for tooltips/menus) |
| `mcp_microsoft_pla_browser_evaluate` | Run JavaScript for assertions (URL checks, row counts, etc.) |
| `mcp_microsoft_pla_browser_wait_for` | Wait for text to appear/disappear or a fixed time |
| `mcp_microsoft_pla_browser_navigate_back` | Go back in browser history |
| `mcp_microsoft_pla_browser_tabs` | List open tabs / switch tabs |

> **Snapshot-first rule**: Always call `mcp_microsoft_pla_browser_snapshot` before
> any `click`, `fill_form`, or `type` call. Read the snapshot output to find the
> exact `target` reference for the element. **Never guess element references.**

---

## Prerequisites

```bash
# Playwright MCP server — must already be running
# No additional Playwright installation needed for MCP-based execution

# Azure CLI + DevOps extension (for fetching tickets and test plans)
az login
az extension add --name azure-devops

# jq for JSON processing
# macOS: brew install jq
# Linux: apt install jq
# Windows (Git Bash): winget install jqlang.jq
```

> **Windows users**: Run terminal commands in **Git Bash**, not CMD or PowerShell.

---

## Input Formats

The agent accepts test scenarios from **two sources**. Read → [references/csv-format.md] for CSV details and [references/ado-format.md] for ADO integration.

### Source A — CSV File

A flat CSV where each row is one smoke test scenario:

```csv
id,module,scenario,steps,expected_result,start_url,priority,tags,depends_on,data_setup,data_teardown
ST-001,Auth,Login with valid credentials,"1. Enter valid username and password|2. Click Login",User lands on dashboard,/user/login,P0,auth;login,,,
ST-002,Auth,Login with invalid credentials,"1. Enter invalid credentials|2. Click Login",Error message shown,/user/login,P0,auth;negative,,,
```

> `start_url` is a **reset point** — the agent navigates here silently before step 1. It guarantees a clean browser state between scenarios regardless of where the previous one ended. It is NOT a test step.

→ Full column spec, validation rules, priority definitions, and extended examples: **read `references/csv-format.md`**

### Source B — Azure DevOps Test Plan / Suite / Work Items

The agent can pull scenarios from:
- A **Test Plan** (all test cases in a plan)
- A **Test Suite** (specific suite within a plan)
- A **Query** (work items tagged for smoke testing)
- A **Sprint** (all test cases for current sprint)
- A **Bug Work Item** — repro steps are read from `Microsoft.VSTS.TCM.ReproSteps`
  and used as the primary execution sequence

See [references/ado-format.md] for fetch commands.

---

## Execution Flow

```
Step 1  →  Load Config
Step 2  →  Load Scenarios (CSV or ADO)
Step 3  →  Filter & Prioritize
Step 4  →  Review Run Plan (Human Gate)
Step 5  →  Login to App
Step 6  →  Execute Scenarios (loop)
Step 7  →  Update Page Map
Step 8  →  Generate Report  → smoke-report.json + smoke-report.html (same folder)
Step 9  →  Human Review Gate → Post to ADO (optional)
Step 10 →  Save Passing Specs for CI (optional)
Step 11 →  Cleanup
```

---

## Step 1 — Load Config

Read `.github/inputs/smoke-testing/smoke-test-skill.env.json`. Extract:

```
BASE_URL         ← config.app.baseUrl
LOGIN_URL        ← config.app.loginUrl
DEFAULT_PATH     ← config.app.defaultPath
USERNAME         ← config.credentials.username
PASSWORD         ← config.credentials.password
STOP_ON_FAIL     ← config.smoke.stopOnFirstFailure
TIMEOUT_MS       ← config.smoke.defaultTimeoutMs
SCREENSHOT_PASS  ← config.smoke.screenshotOnPass
SCREENSHOT_FAIL  ← config.smoke.screenshotOnFailure
HEADLESS         ← config.smoke.headless  (default: false)
OUTPUT_DIR       ← config.smoke.outputDir  (with {runId} replaced at runtime)
```

Generate `RUN_ID` = `smoke-{YYYYMMDD}-{HHmm}` (e.g. `smoke-20250515-1430`).
Resolve `OUTPUT_DIR` by replacing `{runId}` with the generated run ID.

**Safety check** — refuse if `BASE_URL` does not contain:
`staging`, `test`, `qa`, `dev`, `localhost`, `127.0.0.1`, `uat`, `preview`

> If the URL looks like production, show a warning and ask the user to explicitly
> confirm with: "I confirm this is safe to smoke test: [URL]"

Create run directories:
```bash
mkdir -p .tmp/smoke/$RUN_ID
mkdir -p $OUTPUT_DIR/screenshots/pass
mkdir -p $OUTPUT_DIR/screenshots/fail
```

For PowerShell:
```powershell
New-Item -ItemType Directory -Force -Path ".tmp/smoke/$RUN_ID", "$OUTPUT_DIR/screenshots/pass", "$OUTPUT_DIR/screenshots/fail" | Out-Null
```

```
✅ Config loaded
   App:     {BASE_URL}
   Run ID:  {RUN_ID}
   Output:  {OUTPUT_DIR}
```

---

## Step 2 — Load Scenarios

### 2A — From CSV

```bash
# Read the user-supplied CSV file
# Parse into internal scenario array (see schema below)
```

Parse each CSV row into:
```json
{
  "id": "ST-001",
  "module": "Auth",
  "scenario": "Login with valid credentials",
  "steps": ["Enter valid username and password", "Click Login"],
  "expected_result": "User lands on dashboard",
  "start_url": "/user/login",
  "priority": "P0",
  "tags": ["auth", "login"],
  "depends_on": [],
  "data_setup": null,
  "data_teardown": null,
  "status": "PENDING"
}
```

### 2B — From ADO

Read **`references/ado-format.md`** for all `az` CLI commands covering:
- Single work item (bug or story)
- Test suite / test plan
- Work item query
- Current sprint
- How to parse ADO items into the scenario schema

> **Bug tickets**: Use `Microsoft.VSTS.TCM.ReproSteps` as the primary execution sequence. For all other work item types, derive steps from `AcceptanceCriteria` then `Description`.

Parse test case descriptions, acceptance criteria, and repro steps into the same scenario schema.
Save parsed scenarios to `.tmp/smoke/{RUN_ID}/scenarios.json`.

```
✅ Scenarios loaded
   Total:    {N}
   P0:       {count}
   P1:       {count}
   P2:       {count}
   Modules:  {comma-separated list}
```

---

## Step 3 — Filter & Prioritize

Ask the user (or accept as command-line args):

**Filter options:**
- Run **all** scenarios
- Run **P0 only** (critical, must never fail post-release)
- Run **P0 + P1** (standard smoke suite)
- Run by **module** (e.g. only `Auth` and `Reports`)
- Run by **tag** (e.g. `auth`, `checkout`)
- Run a **specific ID list** (e.g. `ST-001,ST-005,ST-012`)

Sort execution order: P0 first → P1 → P2 → alphabetical within priority.

Save filtered + sorted list to `.tmp/smoke/{RUN_ID}/run-plan.json`:

```json
{
  "runId": "smoke-20250515-1430",
  "filter": "P0+P1",
  "totalScenarios": 18,
  "estimatedDuration": "~22 minutes",
  "scenarios": [ ... ]
}
```

---

## Step 4 — Review Run Plan (Human Gate)

Display run plan summary:

```
━━━ Smoke Test Run Plan ━━━
  Run ID:     smoke-20250515-1430
  App URL:    https://staging.myapp.com
  Filter:     P0 + P1
  Scenarios:  18

  By Module:
    Auth          3  (P0: 2, P1: 1)
    Dashboard     4  (P0: 2, P1: 2)
    Reports       5  (P0: 1, P1: 3, P2: 1)
    Checkout      6  (P0: 3, P1: 2, P2: 1)

  Est. duration: ~22 minutes
  Output dir:    .github/outputs/smoke-test/smoke-20250515-1430/
━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If `HEADLESS: true` → skip this gate and proceed automatically.

Otherwise **ask**: "Start smoke test run? [Y/n/edit]"

- **Y** → proceed
- **n** → abort
- **edit** → let user modify filter and re-run Step 3

---

## Step 5 — Login to App

```
mcp_microsoft_pla_browser_navigate   →  {LOGIN_URL}
mcp_microsoft_pla_browser_snapshot   →  find username, password, submit fields
if SCREENSHOT_PASS: mcp_microsoft_pla_browser_take_screenshot → {OUTPUT_DIR}/screenshots/pass/00-login-page.png
mcp_microsoft_pla_browser_fill_form  →  username={USERNAME}, password={PASSWORD}
mcp_microsoft_pla_browser_click      →  submit button (target resolved from snapshot)
mcp_microsoft_pla_browser_wait_for   →  expected post-login text or URL
if SCREENSHOT_PASS: mcp_microsoft_pla_browser_take_screenshot → {OUTPUT_DIR}/screenshots/pass/01-post-login.png
```

If login fails (expected post-login text/URL not found after submit):
- Screenshot → `{OUTPUT_DIR}/screenshots/fail/LOGIN-FAILED.png`
- Write `LOGIN_FAILED` result to run summary
- **Stop the entire run** — do not attempt any scenarios

```
✅ Login successful — {USERNAME} @ {BASE_URL}
```

---

## Step 6 — Execute Scenarios

For each scenario in the run plan:

### 6A — Pre-scenario Setup

```
[ST-001] Auth › Login with valid credentials  [P0]
```

**Dependency check** — before doing anything, check `depends_on`:
- If any ID in `depends_on` has `status: FAILED` or `status: SKIPPED` → mark this scenario `SKIPPED` with `skipReason: "dependency {ID} failed"` and move to the next scenario immediately. Do NOT execute any steps.
- If `depends_on` is empty or all dependencies passed → proceed.

**Data setup** — if `data_setup` is set, execute it now and store the result as `{SETUP_DATA}`. Steps may reference `{SETUP_DATA}` tokens (e.g. `{SETUP_DATA.memberId}`).

**Navigate to start URL** — silently reset browser state:
```
mcp_microsoft_pla_browser_navigate → {BASE_URL}{start_url}
mcp_microsoft_pla_browser_snapshot → verify page loaded (read output before proceeding)
```

### 6B — Selector Resolution

Before every interaction, use this priority order to find the element:

```
1. page-map.json hit        →  stored selector from a prior run (fastest, use directly)
2. snapshot discovery       →  call snapshot, read live accessibility tree, extract target ref
3. data-testid attribute    →  [data-testid="submit-btn"]     (most reliable)
4. ARIA role + name         →  role=button[name="Apply"]
5. Visible text             →  text=Submit
6. CSS selector             →  #submit-btn, .my-button
```

**Maximum 3 snapshot attempts** per element. If the element is still not found
after 3 attempts, call `mcp_microsoft_pla_browser_take_screenshot` for visual
inspection, record the step as FAILED, and move to the next scenario.

### 6C — Execute Steps (Snapshot-First Rule)

For each plain-English step in `scenario.steps`:

1. **Interpret** the step — determine which MCP tool applies:

| Step pattern | MCP tool |
|---|---|
| "Go to / Navigate to {path}" | `mcp_microsoft_pla_browser_navigate` |
| "Click {element}" | snapshot → `mcp_microsoft_pla_browser_click` |
| "Fill / Enter / Type {value} in {field}" | snapshot → `mcp_microsoft_pla_browser_fill_form` |
| "Select {option} from {dropdown}" | snapshot → `mcp_microsoft_pla_browser_select_option` |
| "Wait for {text/element}" | `mcp_microsoft_pla_browser_wait_for` |
| "Press {key}" | `mcp_microsoft_pla_browser_press_key` |
| "Verify / Assert / Check {condition}" | snapshot + `mcp_microsoft_pla_browser_evaluate` |
| "Hover over {element}" | snapshot → `mcp_microsoft_pla_browser_hover` |
| "Upload {file}" | snapshot → `mcp_microsoft_pla_browser_file_upload` |
| "Scroll to {element}" | `mcp_microsoft_pla_browser_evaluate` (scrollIntoView) |

2. **Snapshot before every interaction** — always call `mcp_microsoft_pla_browser_snapshot` before `click`, `fill_form`, or `type`. Read the output carefully. Extract the exact `target` reference before proceeding.
3. **Execute** the MCP tool with the resolved target
4. **Screenshot** on each step if `screenshotOnPass: true` → `{OUTPUT_DIR}/screenshots/pass/{id}-step-{N}.png`; if `false`, skip all pass screenshots

### 6D — Assertion (Expected Result Check)

After all steps, evaluate `scenario.expected_result`:

```
mcp_microsoft_pla_browser_snapshot → read final page state
```

| Result type | How to check |
|---|---|
| "Text X visible" | Find text in snapshot output |
| "URL is {path}" | `mcp_microsoft_pla_browser_evaluate`: `window.location.pathname` |
| "Element {X} visible" | Find element in snapshot |
| "Error message shown" | Find error-class element in snapshot |
| "File downloaded" | Check download via evaluate or wait_for |
| "Toast/notification appears" | `wait_for` → snapshot |
| "Table has {N} rows" | `evaluate`: `document.querySelectorAll('tr').length` |
| "No error on page" | Snapshot: no error text/class visible |
| "Modal opened" | Snapshot: modal/dialog element present |
| "Redirect to {path}" | Evaluate: `window.location.pathname === '{path}'` |

### 6E — Record Result

```json
{
  "id": "ST-001",
  "module": "Auth",
  "scenario": "Login with valid credentials",
  "priority": "P0",
  "status": "PASSED",
  "duration": "12s",
  "stepsExecuted": 3,
  "failedAt": null,
  "failureReason": null,
  "skipReason": null,
  "rootFailure": null,
  "dependsOn": [],
  "selectorKnown": true,
  "screenshots": [
    "screenshots/pass/ST-001-step-1.png",
    "screenshots/pass/ST-001-step-3.png"
  ],
  "assertionChecked": "User lands on dashboard",
  "assertionResult": "PASSED"
}
```

On **failure**:
- Take screenshot → `{OUTPUT_DIR}/screenshots/fail/{id}-FAILED.png`
- Set `status: "FAILED"`, record `failedAt` step and `failureReason`
- Run `data_teardown` if set — always, regardless of pass/fail
- Any scenario with this ID in its `depends_on` will be auto-`SKIPPED` with `rootFailure: "{this-id}"`
- If `stopOnFirstFailure: true` → halt run after this scenario
- Otherwise → continue to next scenario

On **skip** (cascade):
- Set `status: "SKIPPED"`, `skipReason: "dependency {ID} failed"`, `rootFailure: "{ID}"`
- No browser interaction — move immediately to next scenario
- Skipped scenarios do NOT count as failures in the release decision

**Progress output** (live, as each completes):

```
[✅ PASS]  ST-001  Auth › Login with valid credentials          (12s)
[✅ PASS]  ST-002  Auth › Login with invalid credentials        (9s)
[❌ FAIL]  ST-003  Members › Create member                      (15s) ← Save button not found
[⏭ SKIP]  ST-004  Members › Edit member                        (0s)  ← dependency ST-003 failed
[⏭ SKIP]  ST-005  Members › Delete member                      (0s)  ← dependency ST-003 failed
[✅ PASS]  ST-006  Reports › Generate a report                  (34s)
```

---

## Step 7 — Update Page Map

For every scenario that **PASSED**, record discovered selectors into `page-map.json`.
For steps where `selectorKnown` was false and execution succeeded, save the working selector.

```json
{
  "_readme": "Auto-maintained by Test Agents. Commit this file — it is not sensitive.",
  "_version": 1,
  "pages": {
    "login": {
      "url": "/login",
      "lastUpdated": "2025-05-15",
      "elements": {
        "usernameField": "#email",
        "passwordField": "#password",
        "submitButton": "button[type='submit']"
      }
    },
    "dashboard": {
      "url": "/dashboard",
      "lastUpdated": "2025-05-15",
      "elements": {
        "widget1": "[data-testid='revenue-widget']",
        "widget2": "[data-testid='users-widget']"
      }
    }
  }
}
```

Write updated `page-map.json` to disk. This file is safe to commit — it contains no secrets.

---

## Step 8 — Generate Report

Generate both **JSON** and **HTML** reports. Read [references/report-template.md] for HTML structure.

### 8A — Summary JSON

Write to `{OUTPUT_DIR}/smoke-report.json`:

```json
{
  "runId": "smoke-20250515-1430",
  "releaseVersion": "{if provided by user}",
  "appUrl": "https://staging.myapp.com",
  "executedBy": "Smoke Skill Agent",
  "startTime": "2025-05-15T14:30:00Z",
  "endTime": "2025-05-15T14:52:11Z",
  "duration": "22m 11s",
  "filter": "P0+P1",
  "summary": {
    "total": 18,
    "passed": 15,
    "failed": 3,
    "skipped": 2,
    "cascadeSkips": 2,
    "passRate": "83.3%"
  },
  "byPriority": {
    "P0": { "total": 8, "passed": 8, "failed": 0 },
    "P1": { "total": 10, "passed": 7, "failed": 3 }
  },
  "byModule": {
    "Auth": { "total": 3, "passed": 3, "failed": 0 },
    "Dashboard": { "total": 4, "passed": 2, "failed": 2 },
    "Reports": { "total": 5, "passed": 5, "failed": 0 },
    "Checkout": { "total": 6, "passed": 5, "failed": 1 }
  },
  "failedScenarios": [
    {
      "id": "ST-003",
      "scenario": "Dashboard loads widgets",
      "priority": "P1",
      "failureReason": "Widget 3 (.revenue-widget) not found in snapshot after 10s",
      "screenshot": "screenshots/fail/ST-003-FAILED.png"
    }
  ],
  "releaseDecision": "CONDITIONAL_PASS"
}
```

**Release decision logic** — based on **root failures only**, not cascade skips:

| Condition | Decision |
|---|---|
| Any P0 `FAILED` (root) | `P0_FAIL` — block release |
| P0 `FAILED` + downstream `SKIPPED` | Still `P0_FAIL` — skips don't soften the verdict |
| Only P1/P2 `FAILED` (root), no P0 failures | `CONDITIONAL_PASS` |
| Only `SKIPPED` scenarios, zero root failures | `CONDITIONAL_PASS` — flag skips for investigation |
| All `PASSED` (skips = 0) | `PASS` |

> A skipped scenario is never counted as a failure. It is always reported separately in the run summary under `cascadeSkips`.

→ Full decision matrix, ADO comment formats, escalation contacts, and re-run playbook: **read `references/release-decision.md`**

### 8B — HTML Report

> **MANDATORY** — Do not skip. The HTML report is the primary deliverable shared with the team. Both `smoke-report.json` and `smoke-report.html` must exist in `{OUTPUT_DIR}` before proceeding to Step 9.

**Before writing the file, you MUST call `read_file` on `.github/skills/smoke-testing/references/report-template.md`** to load the full HTML template, inline CSS, and token reference. Do not generate HTML from memory.

Steps:
1. Call `read_file` on `references/report-template.md` — load the template now
2. Replace all `{PLACEHOLDER}` tokens using values from `smoke-report.json`
3. For each failed scenario, read the screenshot from `{OUTPUT_DIR}/screenshots/fail/{id}-FAILED.png` and base64-encode it inline
4. Write the completed HTML to **`{OUTPUT_DIR}/smoke-report.html`** — same folder as `smoke-report.json`

The HTML must be **self-contained** — no external CDN, inline CSS only, base64-encoded screenshots for failures.

---

## Step 9 — Human Review Gate → Post to ADO

Display the final summary:

```
━━━ Smoke Test Run Complete ━━━
  Run ID:       smoke-20250515-1430
  Duration:     22m 11s
  Scenarios:    18
  ✅ Passed:    13 (72.2%)
  ❌ Failed:     3  (root failures)
  ⏭ Skipped:    2  (cascade — blocked by root failures)
  P0 Status:    ✅ ALL PASSED
  P1 Status:    ❌ 3 FAILED (root)

  Release Decision: ⚠️  CONDITIONAL PASS
  (P0 clean — P1 root failures logged as known issues)

  Root Failures:
    ST-003  Members › Create member                  [P1]  ← root
    ST-009  Checkout › Apply promo code              [P1]  ← root
    ST-014  Reports › Export to Excel                [P1]  ← root

  Cascade Skips (not counted as failures):
    ST-004  Members › Edit member                   [P1]  ← blocked by ST-003
    ST-005  Members › Delete member                 [P1]  ← blocked by ST-003

  Report: .github/outputs/smoke-test/smoke-20250515-1430/smoke-report.html
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If `HEADLESS: true` → skip ADO post (no credentials configured for unattended posting).

Otherwise **ask**: "Post this smoke test report to ADO? [Y/n]"

If Y and `WORK_ITEM_ID` or `TEST_RUN_ID` is available:

```bash
# Post as comment to the release work item
az boards work-item comment add \
  --id "$RELEASE_WORK_ITEM_ID" \
  --text "$SMOKE_REPORT_COMMENT" \
  --output none

# Or update ADO Test Run results
az test run publish \
  --project "$ADO_PROJECT" \
  --build-id "$BUILD_ID" \
  --run-title "Smoke Test - $RUN_ID"
```

---

## Step 10 — Save Passing Scenarios as Specs (Optional, for CI)

If `HEADLESS: true` → skip this step entirely.

Otherwise, if any scenarios **PASSED** and the user wants them in CI, ask:
"Save passing scenarios as Playwright specs in `{OUTPUT_DIR}/specs/` for future CI runs? [Y/n]"

If approved, generate a `.spec.ts` file per passing scenario using the exact selectors
discovered during the MCP run and recorded in `page-map.json`:

```typescript
// Auto-generated by Smoke Test Agent
// Scenario: {id} — {scenario}
// Run ID:   {RUN_ID}
// Generated: {TIMESTAMP}
// Executed via Playwright MCP — this spec is for CI reproduction only

import { test, expect } from '@playwright/test';
import * as fs from 'fs';

const config = JSON.parse(fs.readFileSync('.github/inputs/smoke-testing/smoke-test-skill.env.json', 'utf-8'));
const BASE_URL  = config.app.baseUrl;
const LOGIN_URL = config.app.loginUrl;
const USERNAME  = config.credentials.username;
const PASSWORD  = config.credentials.password;

test('{scenario}', async ({ page }) => {

  // Login
  await page.goto(LOGIN_URL);
  await page.fill('#email', USERNAME);          // selector from page-map.json
  await page.fill('#password', PASSWORD);
  await page.click("button[type='submit']");
  await page.waitForURL(`**${config.app.defaultPath}**`, { timeout: 15000 });

  // Step 1 — Navigate to starting URL
  await page.goto(`${BASE_URL}{url_path}`);
  await page.screenshot({ path: '{OUTPUT_DIR}/screenshots/pass/{id}-step-00-start.png' });

  // ... steps using discovered selectors from page-map.json ...

});
```

Save to `{OUTPUT_DIR}/specs/{id}-{scenario-slug}.spec.ts`.

> Review passing specs and copy them to a permanent location (e.g. `tests/smoke/`)
> before committing. Selectors used must be the exact ones discovered during the MCP run.
> Never commit credentials — specs always read from `.github/inputs/smoke-testing/smoke-test-skill.env.json`.

---

## Step 11 — Cleanup

```bash
rm -rf .tmp/smoke/$RUN_ID
```

Report, screenshots, and specs remain in `{OUTPUT_DIR}` permanently (until gitignored cleanup).

```
━━━ Smoke Test Agent — Run Complete ━━━
  Result:   {CONDITIONAL_PASS / PASS / P0_FAIL}
  Report:   {OUTPUT_DIR}/smoke-report.html
  JSON:     {OUTPUT_DIR}/smoke-report.json
  Specs:    {OUTPUT_DIR}/specs/ (if generated)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Modes

| Mode | Steps | When to use |
|------|-------|-------------|
| **Full run** | 1→11 | Post-release smoke test |
| **P0 only** | 1→11, P0 filter | Critical regression check |
| **Single module** | 1→11, module filter | After a targeted fix |
| **Dry run** | 1→4 only | Preview plan without executing |
| **Re-run failures** | Load last results → 6→9 | Re-test only failed scenarios |
| **Report only** | Load existing results → 8→9 | Regenerate / re-post report |
| **CSV validate** | 2 only | Check if CSV is well-formed |
| **Generate specs only** | 10 only | Convert existing run results to CI specs |

---

## Selector Strategy

```
1. page-map.json hit        →  stored selector from prior run (fastest)
2. snapshot discovery       →  call mcp_microsoft_pla_browser_snapshot, read live
                               accessibility tree, extract exact target reference
3. data-testid attribute    →  [data-testid="submit"]        (most reliable)
4. ARIA role + name         →  role=button[name="Apply"]
5. Visible text             →  text=Submit
6. CSS selector             →  #submit-btn, .my-button
```

**How snapshot-based discovery works:**
1. Call `mcp_microsoft_pla_browser_snapshot`
2. Read the returned accessibility tree carefully
3. Identify the element matching the step description
4. Extract its `target` reference (e.g. `ref=s1e23` or a CSS selector)
5. Pass that reference as `target` in the next `click`, `fill_form`, or `type` call
6. Record the working selector in `page-map.json` for future runs

**Tip for your team**: Add `data-testid` attributes to key UI elements as you build them.
This allows the agent to skip snapshot discovery entirely and execute faster.

Maximum **3 snapshot attempts** per element before reporting failure.

---

## File Structure After a Run

```
your-repo/
├── page-map.json                      ← commit this (no secrets)
├── .github/inputs/smoke-testing/
│   ├── smoke-test-skill.env.json      ← gitignored, never commit
│   └── smoke-scenarios.csv           ← your scenario source (commit)
├── AGENTS.md                          ← includes smoke test rules
├── .tmp/smoke/{runId}/                ← deleted after run (gitignored)
└── .github/outputs/smoke-test/
    └── smoke-20250515-1430/
        ├── smoke-report.html          ← shareable report
        ├── smoke-report.json          ← machine-readable
        ├── specs/                     ← CI specs (if generated in Step 10)
        │   └── ST-001-login-valid.spec.ts
        └── screenshots/
            ├── pass/
            │   ├── 00-login-page.png
            │   ├── 01-post-login.png
            │   ├── ST-001-step-1.png
            │   └── ST-001-step-3.png
            └── fail/
                ├── LOGIN-FAILED.png
                ├── ST-003-FAILED.png
                └── ST-009-FAILED.png
```

> **Neither `.tmp/smoke/` nor `.github/outputs/smoke-test/` should be committed.**
> Only commit `page-map.json` and `.github/inputs/smoke-testing/smoke-scenarios.csv`.
> Review passing specs from `{OUTPUT_DIR}/specs/` and copy to `tests/smoke/` before committing.

---

## Reference Files

Read these at the step indicated — do not load all of them upfront.

| File | Read at | Contents |
|---|---|---|
| `references/csv-format.md` | Step 2A | Full column spec, validation rules, priority definitions, extended examples |
| `references/ado-format.md` | Step 2B | All `az` CLI fetch commands — work items, test suites, test plans, queries, sprints |
| `references/report-template.md` | Step 8B (**mandatory read_file call required**) | Complete self-contained HTML template, inline CSS, token reference, base64 screenshot embedding |
| `references/release-decision.md` | Step 8A | Decision matrix, ADO comment formats, escalation contacts, re-run playbook |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Login fails | Wrong creds or URL | Check `loginUrl` and credentials in env.json |
| Session expires mid-run | Long run, short session timeout | Add re-login logic between modules |
| Element not found after 3 snapshots | Dynamic rendering / timing | Add `mcp_microsoft_pla_browser_wait_for` before snapshot |
| CSV parse error | Bad delimiters or encoding | Check CSV is UTF-8, pipes not commas in steps column |
| ADO fetch fails | PAT expired | Regenerate PAT with Test Management (read/write) scope |
| Screenshots blank | MCP server not ready after navigate | Add `wait_for` 2s after navigate before screenshot |
| All scenarios fail | Wrong base URL or broken session | Check URL, verify re-login, confirm app health |
| Page map not growing | Step 7 skipped | Always run Step 7 — commit page-map.json after each run |
| Repro steps not found | Bug ticket missing TCM field | Check ADO field `Microsoft.VSTS.TCM.ReproSteps` is populated |
