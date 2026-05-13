# ADO Format Reference

How to load smoke test scenarios from Azure DevOps — test plans, test suites, work item queries, sprints, and bug repro steps.

---

## Prerequisites

```bash
# Azure CLI with DevOps extension
az login
az extension add --name azure-devops

# Set defaults (optional — avoids repeating --org and --project on every command)
az devops configure --defaults organization=https://dev.azure.com/your-org project=YourProject

# jq for JSON parsing
# Windows (Git Bash): winget install jqlang.jq
# macOS: brew install jq
# Linux: apt install jq
```

---

## Source Option 1 — Single Work Item (Bug or Story)

Use this when verifying a specific bug fix or story.

```bash
WORK_ITEM_ID=<NUMBER>
RUN_ID=smoke-$(date +%Y%m%d-%H%M)

az boards work-item show \
  --id "$WORK_ITEM_ID" \
  --output json > .tmp/smoke/$RUN_ID/ticket-$WORK_ITEM_ID.json

# Extract core fields
TICKET_TITLE=$(jq -r '.fields["System.Title"]' .tmp/smoke/$RUN_ID/ticket-$WORK_ITEM_ID.json)
TICKET_TYPE=$(jq -r  '.fields["System.WorkItemType"]' .tmp/smoke/$RUN_ID/ticket-$WORK_ITEM_ID.json)
TICKET_DESC=$(jq -r  '.fields["System.Description"] // ""' .tmp/smoke/$RUN_ID/ticket-$WORK_ITEM_ID.json \
  | sed 's/<[^>]*>//g' | tr -s ' \n')
TICKET_AC=$(jq -r    '.fields["Microsoft.VSTS.Common.AcceptanceCriteria"] // ""' .tmp/smoke/$RUN_ID/ticket-$WORK_ITEM_ID.json \
  | sed 's/<[^>]*>//g' | tr -s ' \n')
TICKET_REPRO=$(jq -r '.fields["Microsoft.VSTS.TCM.ReproSteps"] // ""' .tmp/smoke/$RUN_ID/ticket-$WORK_ITEM_ID.json \
  | sed 's/<[^>]*>//g' | tr -s ' \n')
```

**Work item type → field to use for steps:**

| Work Item Type | Primary field for steps | Fallback |
|---|---|---|
| `Bug` | `Microsoft.VSTS.TCM.ReproSteps` | `System.Description` |
| `User Story` | `Microsoft.VSTS.Common.AcceptanceCriteria` | `System.Description` |
| `Task` | `System.Description` | — |
| `Test Case` | `Microsoft.VSTS.TCM.Steps` (XML) | `System.Description` |

> **Bug tickets**: Always prefer `TICKET_REPRO` — these are the exact human-written steps to reproduce the issue and verify the fix.

---

## Source Option 2 — Test Suite

Fetch all test cases from a specific test suite within a test plan.

```bash
PLAN_ID=<TEST_PLAN_ID>
SUITE_ID=<TEST_SUITE_ID>

az test case list \
  --plan-id "$PLAN_ID" \
  --suite-id "$SUITE_ID" \
  --output json > .tmp/smoke/$RUN_ID/suite-$SUITE_ID.json

# List test case IDs
jq -r '.[].testCase.id' .tmp/smoke/$RUN_ID/suite-$SUITE_ID.json
```

Then fetch each test case individually:

```bash
for CASE_ID in $(jq -r '.[].testCase.id' .tmp/smoke/$RUN_ID/suite-$SUITE_ID.json); do
  az boards work-item show \
    --id "$CASE_ID" \
    --output json > .tmp/smoke/$RUN_ID/case-$CASE_ID.json
done
```

---

## Source Option 3 — Full Test Plan

Fetch all suites within a plan, then all test cases within each suite.

```bash
PLAN_ID=<TEST_PLAN_ID>

az test plan show \
  --plan-id "$PLAN_ID" \
  --output json > .tmp/smoke/$RUN_ID/plan-$PLAN_ID.json

# List all suites
az test suite list \
  --plan-id "$PLAN_ID" \
  --output json > .tmp/smoke/$RUN_ID/suites-$PLAN_ID.json

# For each suite, fetch test cases (loop from Source Option 2)
for SUITE_ID in $(jq -r '.[].id' .tmp/smoke/$RUN_ID/suites-$PLAN_ID.json); do
  az test case list \
    --plan-id "$PLAN_ID" \
    --suite-id "$SUITE_ID" \
    --output json > .tmp/smoke/$RUN_ID/suite-$SUITE_ID.json
done
```

---

## Source Option 4 — Work Item Query

Fetch work items matching a saved ADO query (e.g. a query tagged `smoke-test`).

```bash
QUERY_ID=<QUERY_GUID_OR_PATH>

az boards query \
  --id "$QUERY_ID" \
  --output json > .tmp/smoke/$RUN_ID/query-results.json

# Extract work item IDs
for WI_ID in $(jq -r '.workItems[].id' .tmp/smoke/$RUN_ID/query-results.json); do
  az boards work-item show \
    --id "$WI_ID" \
    --output json > .tmp/smoke/$RUN_ID/wi-$WI_ID.json
done
```

---

## Source Option 5 — Current Sprint

Fetch all work items in the current sprint for smoke validation.

```bash
TEAM=<YOUR_TEAM_NAME>

az boards iteration work-item list \
  --team "$TEAM" \
  --iteration-path "@currentIteration" \
  --output json > .tmp/smoke/$RUN_ID/sprint-items.json

# Filter to bugs and test cases only
jq '[.[] | select(.fields["System.WorkItemType"] | test("Bug|Test Case"))]' \
  .tmp/smoke/$RUN_ID/sprint-items.json
```

---

## Parsing ADO Items into Scenario Schema

After fetching, convert each ADO work item into the standard scenario schema used by Step 6:

```bash
# For a Bug ticket — use ReproSteps as steps
parse_bug_to_scenario() {
  local FILE=$1
  local ID=$(jq -r '.fields["System.Id"] // .id' "$FILE")
  local TITLE=$(jq -r '.fields["System.Title"]' "$FILE")
  local REPRO=$(jq -r '.fields["Microsoft.VSTS.TCM.ReproSteps"] // ""' "$FILE" \
    | sed 's/<[^>]*>//g' | sed 's/&nbsp;/ /g' | tr -s ' \n')

  echo "{
    \"id\": \"ADO-$ID\",
    \"module\": \"Bug\",
    \"scenario\": \"$TITLE\",
    \"steps\": [\"$REPRO\"],
    \"expected_result\": \"Bug is not reproducible — fix verified\",
    \"url_path\": \"/\",
    \"priority\": \"P1\",
    \"tags\": [\"ado\", \"bug\"],
    \"status\": \"PENDING\"
  }"
}
```

> The agent will further parse multi-line repro steps into individual step entries by splitting on numbered list markers (`1.`, `2.`, etc.) or line breaks.

---

## Posting Results Back to ADO

After the run completes (Step 9), post the smoke report as a comment on the release work item:

```bash
# Format report as plain text for ADO comment
SMOKE_REPORT_COMMENT="Smoke Test Run: $RUN_ID
Result: $RELEASE_DECISION
Passed: $PASSED_COUNT / $TOTAL_COUNT ($PASS_RATE)
Failed: $FAILED_COUNT

$([ "$FAILED_COUNT" -gt 0 ] && echo "Failures:" && jq -r '.failedScenarios[] | "  - \(.id): \(.scenario) — \(.failureReason)"' $OUTPUT_DIR/smoke-report.json)

Full report: $OUTPUT_DIR/smoke-report.html"

az boards work-item comment add \
  --id "$RELEASE_WORK_ITEM_ID" \
  --text "$SMOKE_REPORT_COMMENT" \
  --output none
```

---

## Updating ADO Test Run Results

If your project uses ADO Test Plans with an active test run:

```bash
az test run publish \
  --project "$ADO_PROJECT" \
  --build-id "$BUILD_ID" \
  --run-title "Smoke Test - $RUN_ID" \
  --output none
```

---

## Troubleshooting ADO Fetches

| Problem | Cause | Fix |
|---|---|---|
| `ERROR: Please run 'az login'` | Not authenticated | Run `az login` |
| `Extension 'azure-devops' not installed` | Missing extension | `az extension add --name azure-devops` |
| `Resource not found` on work item | Wrong ID or no access | Confirm ID in ADO browser, check permissions |
| PAT expired | Token timeout | Regenerate PAT with `Work Items (read/write)` and `Test Management (read/write)` scopes |
| `ReproSteps` field is null | Bug has no repro steps written | Ask reporter to fill in the field in ADO |
| `AcceptanceCriteria` field is null | Story has no AC | Derive steps from `System.Description` instead |
| HTML tags in field values | ADO rich-text fields | Strip with `sed 's/<[^>]*>//g'` (already in commands above) |
