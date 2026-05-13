# Report Template Reference

Full self-contained HTML report template for the Smoke Test Skill.
The agent generates `smoke-report.html` at Step 8B using this structure.

---

## Requirements

- **Self-contained**: No external CDN links — all CSS is inline
- **Screenshots**: Failure screenshots embedded as base64 `<img>` tags
- **No JavaScript required**: Pure HTML/CSS, works offline and in email
- **Print-friendly**: Table layout suitable for PDF export

---

## How to Generate

At Step 8B, read `{OUTPUT_DIR}/smoke-report.json` (already written at Step 8A) and render it into the template below. Replace all `{PLACEHOLDER}` tokens with actual values from the JSON.

For failure screenshots, read the file from `{OUTPUT_DIR}/screenshots/fail/{id}-FAILED.png` and base64-encode it:

```bash
# Bash
base64 -w 0 path/to/screenshot.png

# PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("path\to\screenshot.png"))
```

---

## HTML Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Smoke Test Report — {RUN_ID}</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; font-size: 14px; color: #1a1a1a; background: #f5f5f5; padding: 24px; }
    .container { max-width: 1100px; margin: 0 auto; background: #fff; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); overflow: hidden; }

    /* Header */
    .header { padding: 24px 32px; border-bottom: 1px solid #e5e5e5; display: flex; justify-content: space-between; align-items: flex-start; }
    .header h1 { font-size: 20px; font-weight: 700; color: #111; }
    .header .meta { font-size: 13px; color: #666; margin-top: 4px; }
    .badge { display: inline-block; padding: 6px 14px; border-radius: 20px; font-size: 13px; font-weight: 600; }
    .badge.pass       { background: #d1fae5; color: #065f46; }
    .badge.cond-pass  { background: #fef3c7; color: #92400e; }
    .badge.fail       { background: #fee2e2; color: #991b1b; }

    /* Summary cards */
    .summary { display: grid; grid-template-columns: repeat(4, 1fr); gap: 16px; padding: 24px 32px; background: #fafafa; border-bottom: 1px solid #e5e5e5; }
    .card { background: #fff; border: 1px solid #e5e5e5; border-radius: 8px; padding: 16px; text-align: center; }
    .card .number { font-size: 32px; font-weight: 700; }
    .card .label  { font-size: 12px; color: #666; margin-top: 4px; text-transform: uppercase; letter-spacing: 0.05em; }
    .card.total  .number { color: #374151; }
    .card.passed .number { color: #059669; }
    .card.failed .number { color: #dc2626; }
    .card.rate   .number { color: #2563eb; }

    /* Tables */
    .section { padding: 24px 32px; border-bottom: 1px solid #e5e5e5; }
    .section h2 { font-size: 15px; font-weight: 600; margin-bottom: 16px; color: #111; }
    table { width: 100%; border-collapse: collapse; font-size: 13px; }
    th { text-align: left; padding: 8px 12px; background: #f9fafb; border-bottom: 2px solid #e5e5e5; font-weight: 600; color: #374151; }
    td { padding: 10px 12px; border-bottom: 1px solid #f0f0f0; vertical-align: top; }
    tr:last-child td { border-bottom: none; }
    tr:hover td { background: #f9fafb; }

    /* Status badges inline */
    .status { display: inline-block; padding: 2px 8px; border-radius: 12px; font-size: 12px; font-weight: 600; }
    .status.passed { background: #d1fae5; color: #065f46; }
    .status.failed { background: #fee2e2; color: #991b1b; }
    .status.skipped { background: #f3f4f6; color: #6b7280; }

    /* Priority */
    .priority { display: inline-block; padding: 2px 8px; border-radius: 4px; font-size: 11px; font-weight: 700; }
    .priority.P0 { background: #fee2e2; color: #991b1b; }
    .priority.P1 { background: #fef3c7; color: #92400e; }
    .priority.P2 { background: #f3f4f6; color: #6b7280; }

    /* Failures */
    .failure-block { background: #fff5f5; border: 1px solid #fecaca; border-radius: 8px; padding: 16px; margin-bottom: 16px; }
    .failure-block .failure-title { font-weight: 600; color: #dc2626; margin-bottom: 8px; }
    .failure-block .failure-reason { font-size: 13px; color: #374151; margin-bottom: 12px; }
    .failure-block img { max-width: 100%; border: 1px solid #e5e5e5; border-radius: 4px; margin-top: 8px; }

    /* Footer */
    .footer { padding: 16px 32px; background: #fafafa; font-size: 12px; color: #9ca3af; display: flex; justify-content: space-between; }
  </style>
</head>
<body>
<div class="container">

  <!-- Header -->
  <div class="header">
    <div>
      <h1>Smoke Test Report</h1>
      <div class="meta">
        Run ID: <strong>{RUN_ID}</strong> &nbsp;|&nbsp;
        App: <strong>{APP_URL}</strong> &nbsp;|&nbsp;
        Branch: <strong>{RELEASE_VERSION}</strong> &nbsp;|&nbsp;
        Duration: <strong>{DURATION}</strong>
      </div>
      <div class="meta" style="margin-top:6px">
        Started: {START_TIME} &nbsp;|&nbsp; Completed: {END_TIME}
      </div>
    </div>
    <!-- RELEASE_DECISION: replace class with 'pass', 'cond-pass', or 'fail' -->
    <span class="badge {DECISION_CLASS}">{RELEASE_DECISION_LABEL}</span>
  </div>

  <!-- Summary Cards -->
  <div class="summary">
    <div class="card total">
      <div class="number">{TOTAL}</div>
      <div class="label">Total Scenarios</div>
    </div>
    <div class="card passed">
      <div class="number">{PASSED}</div>
      <div class="label">Passed</div>
    </div>
    <div class="card failed">
      <div class="number">{FAILED}</div>
      <div class="label">Failed</div>
    </div>
    <div class="card rate">
      <div class="number">{PASS_RATE}</div>
      <div class="label">Pass Rate</div>
    </div>
  </div>

  <!-- Priority Breakdown -->
  <div class="section">
    <h2>By Priority</h2>
    <table>
      <thead>
        <tr><th>Priority</th><th>Total</th><th>Passed</th><th>Failed</th><th>Status</th></tr>
      </thead>
      <tbody>
        <tr>
          <td><span class="priority P0">P0</span></td>
          <td>{P0_TOTAL}</td><td>{P0_PASSED}</td><td>{P0_FAILED}</td>
          <td>{P0_FAILED == 0 ? "✅ All passed" : "❌ " + P0_FAILED + " failed"}</td>
        </tr>
        <tr>
          <td><span class="priority P1">P1</span></td>
          <td>{P1_TOTAL}</td><td>{P1_PASSED}</td><td>{P1_FAILED}</td>
          <td>{P1_FAILED == 0 ? "✅ All passed" : "⚠️ " + P1_FAILED + " failed"}</td>
        </tr>
        <tr>
          <td><span class="priority P2">P2</span></td>
          <td>{P2_TOTAL}</td><td>{P2_PASSED}</td><td>{P2_FAILED}</td>
          <td>{P2_FAILED == 0 ? "✅ All passed" : "ℹ️ " + P2_FAILED + " failed"}</td>
        </tr>
      </tbody>
    </table>
  </div>

  <!-- Module Breakdown -->
  <div class="section">
    <h2>By Module</h2>
    <table>
      <thead>
        <tr><th>Module</th><th>Total</th><th>Passed</th><th>Failed</th></tr>
      </thead>
      <tbody>
        <!-- Repeat for each module in byModule -->
        <tr>
          <td>{MODULE_NAME}</td>
          <td>{MODULE_TOTAL}</td>
          <td>{MODULE_PASSED}</td>
          <td>{MODULE_FAILED}</td>
        </tr>
      </tbody>
    </table>
  </div>

  <!-- All Scenarios -->
  <div class="section">
    <h2>All Scenarios</h2>
    <table>
      <thead>
        <tr><th>ID</th><th>Module</th><th>Scenario</th><th>Priority</th><th>Status</th><th>Duration</th></tr>
      </thead>
      <tbody>
        <!-- Repeat for each scenario in results, sorted P0 → P1 → P2 -->
        <tr>
          <td>{SCENARIO_ID}</td>
          <td>{SCENARIO_MODULE}</td>
          <td>{SCENARIO_NAME}</td>
          <td><span class="priority {PRIORITY}">{PRIORITY}</span></td>
          <td><span class="status {STATUS_CLASS}">{STATUS}</span></td>
          <td>{DURATION}s</td>
        </tr>
      </tbody>
    </table>
  </div>

  <!-- Failures Section (only if FAILED > 0) -->
  <!-- Omit this section entirely if no failures -->
  <div class="section">
    <h2>❌ Failures</h2>
    <!-- Repeat for each failed scenario -->
    <div class="failure-block">
      <div class="failure-title">{SCENARIO_ID} — {SCENARIO_NAME} <span class="priority {PRIORITY}">{PRIORITY}</span></div>
      <div class="failure-reason"><strong>Failed at step:</strong> {FAILED_AT}</div>
      <div class="failure-reason"><strong>Reason:</strong> {FAILURE_REASON}</div>
      <!-- Embed screenshot as base64 if available -->
      <img src="data:image/png;base64,{BASE64_SCREENSHOT}" alt="Failure screenshot for {SCENARIO_ID}" />
    </div>
  </div>

  <!-- Footer -->
  <div class="footer">
    <span>Generated by Smoke Test Skill &nbsp;|&nbsp; Run ID: {RUN_ID}</span>
    <span>Report generated at {GENERATED_AT}</span>
  </div>

</div>
</body>
</html>
```

---

## Token Reference

| Token | Source in smoke-report.json |
|---|---|
| `{RUN_ID}` | `.runId` |
| `{APP_URL}` | `.appUrl` |
| `{RELEASE_VERSION}` | `.releaseVersion` |
| `{DURATION}` | `.duration` |
| `{START_TIME}` | `.startTime` |
| `{END_TIME}` | `.endTime` |
| `{RELEASE_DECISION_LABEL}` | `PASS` → "✅ Clear to Release" / `CONDITIONAL_PASS` → "⚠️ Conditional Pass" / `P0_FAIL` → "🚫 Do Not Release" |
| `{DECISION_CLASS}` | `PASS` → `pass` / `CONDITIONAL_PASS` → `cond-pass` / `P0_FAIL` → `fail` |
| `{TOTAL}` | `.summary.total` |
| `{PASSED}` | `.summary.passed` |
| `{FAILED}` | `.summary.failed` |
| `{PASS_RATE}` | `.summary.passRate` |
| `{P0_TOTAL}` | `.byPriority.P0.total` |
| `{MODULE_NAME}` | key from `.byModule` |
| `{SCENARIO_ID}` | `.scenarios[n].id` |
| `{STATUS_CLASS}` | `PASSED` → `passed` / `FAILED` → `failed` / `SKIPPED` → `skipped` |
| `{BASE64_SCREENSHOT}` | base64-encoded PNG from `screenshots/fail/{id}-FAILED.png` |
| `{GENERATED_AT}` | current timestamp when writing the file |
