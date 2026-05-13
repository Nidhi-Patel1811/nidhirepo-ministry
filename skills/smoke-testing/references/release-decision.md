# Release Decision Reference

Rules for determining the release decision after a smoke test run, and the escalation playbook for each outcome.

---

## Decision Matrix

| Condition | Decision | Label | Action |
|---|---|---|---|
| Any `P0` scenario `FAILED` | `P0_FAIL` | 🚫 Do Not Release | Block release immediately. Escalate to team lead. |
| No `P0` failures, one or more `P1`/`P2` failures | `CONDITIONAL_PASS` | ⚠️ Conditional Pass | Release allowed with known issues logged as bugs. |
| All scenarios `PASSED` | `PASS` | ✅ Clear to Release | Release approved. No action required. |

---

## Decision Logic (Pseudocode)

```
if any scenario where priority == "P0" and status == "FAILED":
    releaseDecision = "P0_FAIL"
elif any scenario where status == "FAILED":
    releaseDecision = "CONDITIONAL_PASS"
else:
    releaseDecision = "PASS"
```

---

## P0_FAIL — Do Not Release

**Triggered by**: Any P0 scenario failing.

**Immediate actions:**
1. Stop the release pipeline (do not deploy)
2. Post smoke report to the release work item in ADO (Step 9)
3. Notify the team lead and the developer who owns the failing module
4. Create a blocking bug in ADO linked to the release work item:
   - Title: `[SMOKE BLOCKER] {scenario} — {failureReason}`
   - Priority: P0 / Severity: Critical
   - Tag: `smoke-blocker`, `release-blocker`
5. Re-run smoke tests only after a fix is deployed to the QA environment
6. Do **not** close the release work item until a full `PASS` run is achieved

**ADO comment format:**
```
🚫 SMOKE TEST FAILED — DO NOT RELEASE

Run ID:   {RUN_ID}
App:      {APP_URL}
Decision: P0_FAIL

P0 Failures:
  {list of failed P0 scenarios with reasons}

Release is BLOCKED. Failing scenarios must be fixed before release.
Report: {OUTPUT_DIR}/smoke-report.html
```

---

## CONDITIONAL_PASS — Release with Known Issues

**Triggered by**: All P0 scenarios pass, but one or more P1 or P2 scenarios fail.

**Actions:**
1. Release may proceed — P0 (critical) paths are healthy
2. Post smoke report to the release work item in ADO (Step 9)
3. For each failing P1/P2 scenario, create a bug in ADO:
   - Title: `[SMOKE] {scenario} — {failureReason}`
   - Priority: P1 (for P1 failures) / P2 (for P2 failures)
   - Tag: `smoke-finding`, `post-release-bug`
   - Link to the release work item
4. Add the failures to the release notes as "Known Issues"
5. Schedule a follow-up smoke run after the P1 bugs are fixed

**ADO comment format:**
```
⚠️ SMOKE TEST — CONDITIONAL PASS

Run ID:   {RUN_ID}
App:      {APP_URL}
Decision: CONDITIONAL_PASS

P0 Status: ✅ All passed
P1/P2 Failures ({count}):
  {list of failing scenarios with reasons}

Release may proceed. Failures logged as post-release bugs.
Report: {OUTPUT_DIR}/smoke-report.html
```

---

## PASS — Clear to Release

**Triggered by**: All scenarios pass (no failures of any priority).

**Actions:**
1. Release approved
2. Post smoke report to the release work item in ADO (Step 9, optional)
3. Close any previously opened `smoke-blocker` bugs from earlier runs
4. Update the release work item status to reflect QA approval

**ADO comment format:**
```
✅ SMOKE TEST PASSED — CLEAR TO RELEASE

Run ID:   {RUN_ID}
App:      {APP_URL}
Decision: PASS

Scenarios: {TOTAL} / {TOTAL} passed (100%)
All P0, P1, and P2 scenarios passed.

Report: {OUTPUT_DIR}/smoke-report.html
```

---

## Partial Run Decisions

When a run is filtered (e.g. P0-only or single module), the decision applies only to the scenarios that were executed.

| Filter used | Decision scope |
|---|---|
| `P0 only` | Only P0 failures counted — no P1/P2 in scope |
| `module:Auth` | Only Auth failures counted |
| `P0+P1` | Standard — full release decision applies |
| `All` | Full release decision applies |

Always note the filter used in the ADO comment so the team knows the coverage.

---

## Escalation Contacts

Update this section for your team:

| Decision | Who to notify | How |
|---|---|---|
| `P0_FAIL` | Team lead + module owner | Slack `#releases` channel + ADO comment |
| `CONDITIONAL_PASS` | QA lead + PM | ADO comment on release work item |
| `PASS` | PM + release manager | ADO comment (optional) |

---

## Re-Run After Fix

After a `P0_FAIL` or `CONDITIONAL_PASS`, once fixes are deployed:

1. Run smoke test in `Re-run failures` mode (Step 6 only for failed scenarios)
2. If all previously-failed scenarios now pass → upgrade decision to `PASS` or `CONDITIONAL_PASS`
3. Post updated report to same ADO work item as a follow-up comment
4. Note in comment: `Re-run after fix — previously failed scenarios now passing`
