# CSV Format Reference

Full specification for `smoke-scenarios.csv` — the scenario input file for the Smoke Test Skill.

---

## File Requirements

- **Encoding**: UTF-8 (no BOM)
- **Line endings**: LF or CRLF both accepted
- **Header row**: required, must be the first line
- **Delimiter**: comma `,`
- **Steps delimiter**: pipe `|` (within the quoted steps field)
- **Quoting**: fields containing commas or pipes must be wrapped in double quotes `"`

---

## Column Definitions

| Column | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Unique scenario ID. Format: `ST-NNN` (e.g. `ST-001`). Must be unique across the file. |
| `module` | string | ✅ | Functional area. Used for grouping and filtering. Examples: `Auth`, `Dashboard`, `Reports`, `Giving`, `Groups`, `People` |
| `scenario` | string | ✅ | Human-readable name for the test scenario. Keep under 80 chars. |
| `steps` | string | ✅ | Pipe-separated ordered steps in plain English. Each step maps to one MCP browser action. Wrap in double quotes when using pipes. |
| `expected_result` | string | ✅ | What must be true for the scenario to PASS. Written as an assertion statement. |
| `start_url` | string | ✅ | URL path the agent navigates to **before step 1** to reset browser state (e.g. `/login`, `/members`). This is a pre-condition, not a test step. Do NOT write "Go to" in steps for this — the agent handles it silently. |
| `priority` | string | ✅ | One of: `P0`, `P1`, `P2`. Controls execution order and release decision. |
| `tags` | string | optional | Semicolon-separated tags for filtering (e.g. `auth;login`, `reports;export`). Leave blank if not needed. |
| `depends_on` | string | optional | Comma-separated IDs of scenarios that must pass before this one runs (e.g. `ST-010`). If any dependency failed or was skipped, this scenario is auto-skipped. Leave blank for independent scenarios. |
| `data_setup` | string | optional | API call or fixture name to execute before steps. Result stored as `{SETUP_DATA}` — usable as tokens in steps. Use for creating temp test data. Leave blank if not needed. |
| `data_teardown` | string | optional | Cleanup to run after the scenario **always** (pass or fail). Use to delete records created by `data_setup` or within the scenario. Leave blank if not needed. |

---

## Priority Definitions

| Priority | Meaning | Release impact |
|---|---|---|
| `P0` | Must-pass. Core app functionality. | Any P0 failure → **DO NOT RELEASE** |
| `P1` | Important but non-blocking. | P1-only failures → `CONDITIONAL_PASS` |
| `P2` | Nice to verify. Edge cases. | P2-only failures → `PASS` (logged as known issues) |

---

## Steps Format

Steps are pipe-separated plain English instructions inside a double-quoted field.

**Rules:**
- Each step maps to one browser action
- Number your steps: `1. Do X|2. Do Y|3. Verify Z`
- Use action verbs: `Go to`, `Click`, `Enter`, `Type`, `Select`, `Verify`, `Wait for`, `Upload`
- Reference UI elements by their visible label, placeholder, or button text
- Do NOT use CSS selectors in steps — the agent resolves those from snapshots

**Step verb → MCP tool mapping:**

| Step starts with | Maps to |
|---|---|
| `Go to` / `Navigate to` | `mcp_microsoft_pla_browser_navigate` |
| `Click` | `mcp_microsoft_pla_browser_click` |
| `Enter` / `Fill` / `Type` | `mcp_microsoft_pla_browser_fill_form` or `type` |
| `Select` | `mcp_microsoft_pla_browser_select_option` |
| `Wait for` | `mcp_microsoft_pla_browser_wait_for` |
| `Verify` / `Assert` / `Check` | `mcp_microsoft_pla_browser_evaluate` + snapshot |
| `Hover over` | `mcp_microsoft_pla_browser_hover` |
| `Upload` | `mcp_microsoft_pla_browser_file_upload` |
| `Press` | `mcp_microsoft_pla_browser_press_key` |

## Data Strategy — Keeping Scenarios Independent

The gold rule: **no scenario should rely on another scenario's side effects for its test data.**

| Scenario type | Recommended strategy |
|---|---|
| **Create** | Create → verify → delete in the **same scenario** using `data_teardown`. Atomic, leaves no trace. |
| **Edit** | Use a permanent fixture record. Edit it, then restore the original value in `data_teardown`. |
| **Delete** | Use `data_setup` to create a temp record via API, then delete it in the scenario. |
| **Multi-step flow** | Use `depends_on` only for genuine UI state chaining (e.g. a wizard). Never for test data. |

> `depends_on` is for **UI flow dependencies** (step 2 of a wizard needs step 1's session state).  
> It is **not** a substitute for proper test data setup. If you find yourself using `depends_on` for data, use `data_setup` instead.

---

## Dependency & Cascade Skip Rules

- If a scenario in `depends_on` has `status: FAILED` or `status: SKIPPED`, this scenario is auto-skipped.
- Skipped scenarios are reported under `cascadeSkips` — they do **not** count as failures.
- The release decision is based on **root failures only**, not cascade skips.
- `data_teardown` runs **regardless** of pass/fail/skip for the scenario that defined it.

---



```csv
id,module,scenario,steps,expected_result,start_url,priority,tags,depends_on,data_setup,data_teardown
ST-001,Auth,Login with valid credentials,"1. Enter valid username and password|2. Click Login button|3. Wait for dashboard to load",User is redirected to main dashboard and username is visible in header,/user/login,P0,auth;login,,,
ST-002,Auth,Login with invalid credentials,"1. Enter invalid username and wrong password|2. Click Login button|3. Wait for error message",Error message is displayed and user remains on login page,/user/login,P0,auth;negative,,,
ST-003,Auth,Logout,"1. Click user avatar in header|2. Click Logout|3. Wait for login page",User is redirected to login page and session is cleared,/,P0,auth;logout,,,
ST-004,People,Search for a person,"1. Type a name in the search box|2. Wait for search results to load|3. Verify at least one result appears",Search results table is visible with at least one matching person record,/members,P1,people;search,,,
ST-005,People,Create then delete a person,"1. Click Add button|2. Fill in First Name and Last Name|3. Click Save|4. Verify success toast|5. Click Delete on new record|6. Confirm deletion",Person created successfully then removed — no trace left,/members,P1,people;create,,create_temp_member,delete_temp_member
ST-006,People,Edit a person,"1. Click on fixture member 'Test User'|2. Edit First Name to 'Test User Edited'|3. Click Save|4. Verify success toast|5. Restore First Name to 'Test User'|6. Save again",Changes saved and restored — fixture record unchanged,/members,P1,people;edit,,,
ST-007,Groups,View group list,"1. Wait for groups to load|2. Verify at least one group appears",Groups list is visible with at least one group,/groups,P1,groups,,,
ST-008,Dashboard,Dashboard loads,"1. Wait for dashboard widgets to load|2. Verify no error messages",Dashboard page is visible with no error messages,/dashboard,P1,dashboard,,,
ST-009,Reports,Open reports page,"1. Wait for reports list to load|2. Verify reports page title visible",Reports page title visible and list loads without error,/reports/myreports,P1,reports,,,
ST-010,Giving,Open giving page,"1. Wait for page to load|2. Verify giving input form is visible",Giving input page loads without error,/utilities/input_giving,P2,giving,,,
```

---

## Validation Rules

The agent validates the CSV at Step 2A before executing. These will cause a parse error:

| Rule | Bad example | Fix |
|---|---|---|
| `id` must be unique | Two rows both `ST-001` | Renumber duplicates |
| `priority` must be `P0`, `P1`, or `P2` | `High`, `Critical` | Use `P0` |
| `steps` must have at least one step | Empty steps field | Add at least `1. Verify page loaded` |
| `start_url` must start with `/` | `login` | Use `/login` |
| Unquoted field with pipe | `1. Click\|2. Type` without quotes | Wrap entire field in `"..."` |
| Pipe inside a step text | `Click "Save\|Cancel"` | Escape or rephrase to avoid pipe in step content |
| `depends_on` ID that doesn't exist | `depends_on: ST-999` | Ensure referenced ID exists in the same CSV |
| Using `depends_on` for test data sharing | ST-011 depends on ST-010 to create a record it then edits | Use `data_setup` instead — keep data creation atomic |

---

## Filtering Examples

These filter expressions work in Step 3 of the skill:

```
All             →  run everything
P0              →  only rows where priority = P0
P0+P1           →  rows where priority = P0 or P1
module:Auth     →  rows where module = Auth
tag:login       →  rows where tags contains "login"
id:ST-001,ST-005,ST-012   →  only those specific IDs
```

---

## Common Mistakes

1. **Using `url_path` (old name)** — the column is now `start_url`. Update any existing CSVs.
2. **Using `/people` instead of `/members`** — check the actual app URL before setting `start_url`
3. **Forgetting quotes around steps** — any step field with `|` must be quoted
4. **Writing selectors in steps** — steps should be plain English; let the agent resolve selectors
5. **P0 for everything** — reserve P0 for flows that would block all users if broken (login, core navigation)
6. **Expected result too vague** — "Page loads" is harder to assert than "Dashboard heading is visible"
7. **Using `depends_on` for data** — if ST-011 needs a record created by ST-010, use `data_setup` on ST-011 instead. `depends_on` is for UI flow chaining only.
8. **Forgetting `data_teardown`** — every scenario that creates data must clean it up, or it pollutes subsequent runs
