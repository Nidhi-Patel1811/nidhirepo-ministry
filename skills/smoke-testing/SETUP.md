# Smoke Test Skill — Setup Guide

End-to-end setup for the `smoke-test-skill`. Follow this once per repo before running your first smoke test.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Folder Structure](#2-folder-structure)
3. [Step-by-Step Setup](#3-step-by-step-setup)
4. [What to Commit vs Ignore](#4-what-to-commit-vs-ignore)
5. [Sample Prompts](#5-sample-prompts)
6. [Running Your First Smoke Test](#6-running-your-first-smoke-test)
7. [Known Quirks & Tips](#7-known-quirks--tips)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Prerequisites

### Required

| Requirement | Version | Why |
|---|---|---|
| **VS Code** | Latest stable | Hosts GitHub Copilot Chat agent |
| **GitHub Copilot Chat** | Latest extension | Runs the skill agent |
| **Playwright MCP Server** | Latest | Browser automation — all `mcp_microsoft_pla_browser_*` tools |
| **Node.js** | **18.x or higher** | Required by Playwright MCP server |

### Optional (for Azure DevOps integration)

| Requirement | Version | Why |
|---|---|---|
| **Azure CLI** | Latest | Fetch ADO tickets, test plans, post results |
| **Azure DevOps CLI extension** | Latest | `az boards`, `az test` commands |
| **jq** | Any | JSON parsing in shell scripts |

---

### Install Node.js 18.x

```bash
# macOS (Homebrew)
brew install node@18
brew link node@18 --force

# Windows (winget)
winget install OpenJS.NodeJS.LTS

# Linux
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify
node --version   # must be v18.x.x or higher
```

---

### Install & Configure Playwright MCP Server

The MCP server runs as a persistent background process. VS Code communicates with it through the GitHub Copilot Chat agent via `mcp_microsoft_pla_browser_*` tools.

**Step 1 — Install the MCP server package**

```bash
npm install -g @playwright/mcp
# or
npx @playwright/mcp --version
```

**Step 2 — Configure it in VS Code**

Add to your VS Code `settings.json` (or `.vscode/mcp.json` in the repo):

```json
{
  "mcp": {
    "servers": {
      "playwright": {
        "command": "npx",
        "args": ["@playwright/mcp@latest"],
        "env": {}
      }
    }
  }
}
```

**Step 3 — Verify tools are available**

Open VS Code Copilot Chat in Agent mode. The following tools should be discoverable:

```
mcp_microsoft_pla_browser_navigate
mcp_microsoft_pla_browser_snapshot
mcp_microsoft_pla_browser_take_screenshot
mcp_microsoft_pla_browser_click
mcp_microsoft_pla_browser_fill_form
mcp_microsoft_pla_browser_type
mcp_microsoft_pla_browser_press_key
mcp_microsoft_pla_browser_wait_for
```

If these are missing, the MCP server is not running. Check the Output panel → "MCP" channel.

---

### Install Azure CLI (optional)

```bash
# macOS
brew install azure-cli

# Windows
winget install Microsoft.AzureCLI

# Linux
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Add DevOps extension
az extension add --name azure-devops

# Log in
az login
az devops configure --defaults organization=https://dev.azure.com/YOUR-ORG project=YOUR-PROJECT
```

---

## 2. Folder Structure

```
your-repo/
│
├── page-map.json                              ← Shared selector cache (commit this)
│
├── .github/
│   ├── skills/
│   │   └── smoke-testing/
│   │       ├── SKILL.md                       ← Skill definition (the "brain")
│   │       ├── SETUP.md                       ← This file
│   │       └── references/
│   │           ├── csv-format.md              ← Full CSV column spec & examples
│   │           ├── ado-format.md              ← ADO fetch commands
│   │           ├── report-template.md         ← HTML report template
│   │           └── release-decision.md        ← Release decision matrix
│   │
│   ├── inputs/
│   │   └── smoke-testing/
│   │       ├── smoke-scenarios.csv            ← Your test scenarios (commit this)
│   │       └── smoke-test-skill.env.json      ← Credentials & config (GITIGNORE — never commit)
│   │
│   └── outputs/
│       └── smoke-test/
│           └── smoke-{YYYYMMDD}-{HHmm}/       ← One folder per run (gitignore outputs)
│               ├── smoke-report.html          ← Shareable HTML report
│               ├── smoke-report.json          ← Machine-readable results
│               └── screenshots/
│                   ├── pass/                  ← Pass screenshots (if screenshotOnPass: true)
│                   └── fail/                  ← Failure screenshots (always captured)
│
└── .tmp/
    └── smoke/                                 ← Temp working files, deleted after run (gitignore)
```

> **`page-map.json`** lives at the project root (not in `.github/inputs/`) because it is a living
> shared artifact used by both `smoke-testing` and `ui-test` skills and grows over time.

---

## 3. Step-by-Step Setup

### Step 1 — Create the inputs folder

```bash
mkdir -p .github/inputs/smoke-testing
```

### Step 2 — Create your environment config

Create `.github/inputs/smoke-testing/smoke-test-skill.env.json`:

```json
{
  "app": {
    "baseUrl": "https://qa6.yourapp.com",
    "loginUrl": "https://qa6.yourapp.com/user/login?destination=",
    "defaultPath": "/dashboard"
  },
  "credentials": {
    "username": "your-smoke-user",
    "password": "YourSmokePassword123!",
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
    "outputDir": "outputs/smoke-test/{runId}",
    "reportTitle": "Post-Release Smoke Test Report"
  }
}
```

> **`authType` values**: `form` (username/password form), `sso` (skip login — already authenticated), `token` (bearer token injected via `localStorage`)

### Step 3 — Create your scenario CSV

Create `.github/inputs/smoke-testing/smoke-scenarios.csv`:

```csv
id,module,scenario,steps,expected_result,start_url,priority,tags,depends_on,data_setup,data_teardown
ST-001,Auth,Login with valid credentials,"1. Enter valid username and password|2. Click Login button|3. Wait for dashboard to load",User is redirected to dashboard and username is visible in the header,/user/login,P0,auth;login,,,
ST-002,Auth,Login with invalid credentials,"1. Enter invalid username and wrong password|2. Click Login button|3. Wait for error message",Error message displayed and user remains on login page,/user/login,P0,auth;negative,,,
ST-003,People,Search for a person,"1. Type a name in the search box|2. Wait for results to load|3. Verify at least one result appears",Search results table shows at least one matching record,/members,P1,people;search,,,
```

> See [references/csv-format.md](references/csv-format.md) for the full 11-column specification, validation rules, `depends_on` usage, and examples with `data_setup`/`data_teardown`.

### Step 4 — Create the page map

```bash
# Mac/Linux
echo '{"_readme":"Auto-maintained by Test Skills. Commit this file.","_version":1,"pages":{}}' > page-map.json

# Windows PowerShell
'{"_readme":"Auto-maintained by Test Skills. Commit this file.","_version":1,"pages":{}}' | Set-Content page-map.json
```

### Step 5 — Update `.gitignore`

```bash
# Add these lines to your .gitignore
echo ".github/inputs/smoke-testing/smoke-test-skill.env.json" >> .gitignore
echo ".tmp/smoke/" >> .gitignore
echo "outputs/smoke-test/" >> .gitignore
```

### Step 6 — Add rules to `AGENTS.md`

Add this block to your project's `AGENTS.md` at the repo root:

```markdown
## Smoke Test Skill Rules

- Never hardcode credentials — always read from `.github/inputs/smoke-testing/smoke-test-skill.env.json`
- Prefer data-testid selectors over CSS classes or XPath
- Call mcp_microsoft_pla_browser_snapshot before every click, fill, or type
- If login fails, stop the entire run — do not attempt any scenarios
- Save passing selectors to page-map.json after every run
- Maximum 3 snapshot attempts per element before marking the step as FAILED
- Never click "Delete", "Remove", or destructive actions on production URLs
- baseUrl must contain "staging", "test", "qa", "dev", "uat", "preview", or "localhost" — refuse prod URLs
```

---

## 4. What to Commit vs Ignore

| File / Folder | Commit? | Reason |
|---|---|---|
| `.github/skills/smoke-testing/` | ✅ Yes | Skill definition — safe, no secrets |
| `.github/inputs/smoke-testing/smoke-scenarios.csv` | ✅ Yes | Scenario definitions — no secrets |
| `.github/inputs/smoke-testing/smoke-test-skill.env.json` | ❌ **No** | Contains credentials — gitignore |
| `page-map.json` | ✅ Yes | Selector cache — no secrets, improves future runs |
| `outputs/smoke-test/` | ❌ No | Generated per run — gitignore |
| `.tmp/smoke/` | ❌ No | Temp files — deleted after run, gitignore |

---

## 5. Sample Prompts

### Run all scenarios from CSV

```
Run smoke tests
```

```
Run the full smoke suite from .github/inputs/smoke-testing/smoke-scenarios.csv against qa6.yourapp.com
```

### Run P0 only (fastest, pre-release gate check)

```
Run P0 smoke tests only
```

```
Smoke test P0 scenarios before releasing to production
```

### Run a specific module

```
Smoke test only the Auth module
```

```
Run smoke tests for the People and Reports modules
```

### Run after a specific release / ticket

```
Post-release smoke test after deploying ticket #274359
```

```
Validate release smoke-20260507 — run P0 + P1 against qa6.sccrmdev.com
```

### Run from Azure DevOps

```
Run smoke tests from ADO test plan #12345
```

```
Smoke test the current sprint's test cases from ADO
```

### Trigger phrases the skill auto-detects

Any of these will invoke the skill without needing `@smoke-test-skill`:

> "run smoke tests" · "smoke test after release" · "post-release QA" · "run release validation" ·
> "check release health" · "validate release" · "post-deployment smoke test" · "QA smoke suite"

---

## 6. Running Your First Smoke Test

1. Open **VS Code** with this repo
2. Ensure the **Playwright MCP server** is running (check the MCP output channel)
3. Open **GitHub Copilot Chat** in Agent mode (`@workspace`)
4. Type:

   ```
   Run smoke tests
   ```

5. The skill will:
   - Load config from `.github/inputs/smoke-testing/smoke-test-skill.env.json`
   - Parse scenarios from `.github/inputs/smoke-testing/smoke-scenarios.csv`
   - Display a run plan and ask for confirmation
   - Log in to the app and execute each scenario
  - Generate `smoke-report.html` and `smoke-report.json` in `outputs/smoke-test/{runId}/`
   - Display a pass/fail summary with release decision

6. Open `smoke-report.html` in a browser to review results

---

## 7. Known Quirks & Tips

### React/Vue forms: use `pressSequentially` not `fill`

`fill()` sets the value directly without firing DOM events. React's `onChange` won't trigger, and the search/form won't respond. The skill handles this automatically via `mcp_microsoft_pla_browser_type` with `slowly: true` — equivalent to `pressSequentially()`. If you're writing manual scenarios, use step wording like "Type the search term character by character" to hint at this.

### Login redirects to dashboard (session still active)

If the app redirects away from the login page immediately (session still active from a previous run), ST-001-style "login with valid credentials" tests still **pass** — the expected outcome is "user is on dashboard", which is satisfied. The skill checks the final page state, not whether the login form was filled.

### `start_url` is a reset point, not a test step

Do NOT include "Go to login page" or "Navigate to /members" as a step in your CSV. Set it in `start_url` instead. The skill navigates there silently before step 1. Including it as a step counts it as a test action and generates an unnecessary screenshot.

### `page-map.json` improves speed over time

After a run, the skill saves discovered selectors to `page-map.json`. On subsequent runs, it skips snapshot discovery for known elements and uses the cached selector directly. Commit this file — it gets more valuable with every run.

### `screenshotOnPass: false` for large suites

For suites with 50+ scenarios, set `screenshotOnPass: false` to cut runtime by 30–40%. Failure screenshots are always captured regardless of this flag.

### `depends_on` — when to use it

Only use `depends_on` when scenario B literally cannot execute without scenario A creating data first (e.g. create-then-edit flows). Do NOT use it for logical grouping. Cascaded skips are not failures and don't block release, but they reduce coverage visibility.

---

## 8. Troubleshooting

### MCP tools not available in Copilot Chat

- Check VS Code Output → "MCP" channel for errors
- Ensure `node --version` returns 18.x or higher
- Confirm `@playwright/mcp` is installed: `npx @playwright/mcp --version`
- Restart VS Code after changing MCP settings

### "baseUrl looks like production" warning

The skill refuses to run against URLs that don't contain `staging`, `test`, `qa`, `dev`, `uat`, `preview`, or `localhost`. If your QA URL doesn't match these patterns, confirm explicitly:

```
I confirm this is safe to smoke test: https://release-preview.yourapp.com
```

### Snapshot returns empty or minimal accessibility tree

Some apps render content inside iframes or shadow DOM. Try:
- Waiting longer: add a "Wait 2 seconds" step before the assertion
- Using `mcp_microsoft_pla_browser_evaluate` to directly query the DOM
- Adding `data-testid` attributes to key elements in the app code

### Login fails: "submit button not found"

The login form selector may have changed. Check `page-map.json` — if it has a stale `submitButton` entry, delete the login page entry from `page-map.json` and re-run. The skill will re-discover selectors from snapshot.

### Screenshots not appearing in report

- Verify `screenshotOnPass: true` and `screenshotOnFailure: true` in env config
- Check that the output directory was created before execution (Step 1 creates it)
- On Windows, ensure paths use forward slashes in the config: `outputs/smoke-test/{runId}`

### ADO post fails: "Unauthorized"

- Run `az login` and re-authenticate
- Ensure `az extension add --name azure-devops` was run
- If using a PAT, add it to `smoke-test-skill.env.json` under `ado.pat`
