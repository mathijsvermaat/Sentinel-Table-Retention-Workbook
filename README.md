# Sentinel Table Retention Workbook

An Azure Monitor / Microsoft Sentinel workbook for reviewing table retention, increasing total retention for one or many tables, and forecasting Data Lake storage costs.

**Current version: 1.0**

Download [SentinelDataLakeRetention.workbook.json](SentinelDataLakeRetention.workbook.json) and import it into a workbook. No scripts, deployments, Azure Functions, Logic Apps, Automation accounts, or helper identities are required.

## Features

| Tab | Purpose |
|---|---|
| **Tables** | Search and select tables, review proposed changes, and apply retention-only updates. |
| **Costs** | Forecast current, proposed, and additional monthly Lake storage costs from observed table volume. |
| **Verify** | Select one or more tables to read current settings and inspect their recent Azure Activity Log events, initiators, details, and source links. |

- **One workflow for one or many tables:** every prepared change uses a batch containing one retention-only PATCH per eligible table.
- **Retention only:** requests contain `totalRetentionInDays`, not schema, subtype, plan, or analytics-retention settings.
- **Explicit confirmation:** review table names and target retention before any update.
- **Increase-only preview:** tables already at or above the target are skipped.
- **Readable layout:** icon-led headings, status indicators, friendly column labels, and optional detailed guidance.
- **Native authentication:** requests use your portal identity and Azure RBAC. No credentials are embedded.

> **Important:** this workbook changes Azure resource settings. Longer retention can increase storage costs. Start with a small, noncritical selection and verify the results before broader use.

## Prerequisites

- A Microsoft Sentinel Data Lake-integrated Log Analytics workspace.
- Permission to read and update that workspace's table settings. Microsoft documents the relevant permissions in [Configure table settings](https://learn.microsoft.com/azure/sentinel/manage-table-tiers-retention).
- Workspace query access for the selection and review queries.
- Read access to the workspace's `Usage` table for cost forecasts.
- Activity Log read access for recent table operations in Verify. If unavailable, the activity panel reports an error independently; current table settings can still be read with table-read permission.

Defender portal access alone does not necessarily grant the Azure permissions required by the table API. Current updates do not require deployment-write permissions.

## Import the workbook

1. Download [SentinelDataLakeRetention.workbook.json](SentinelDataLakeRetention.workbook.json). If viewing it on GitHub, use **Raw** or **Download raw file**.
2. In Microsoft Defender, open **Microsoft Sentinel > Threat management > Workbooks** and add a workbook. You can also use Azure Monitor Workbooks.
3. Select **Edit > Advanced editor**.
4. Replace the workbook definition with the downloaded JSON, then select **Apply** and **Done Editing**.
5. Select your subscription and workspace.
6. Save a copy in your environment if desired.

Set **Detailed guidance** to **Show** to find **Version information**, including the current version (**1.0**) and a version-history table. Public release history starts at 1.0. Reimport the JSON when upgrading an existing copy; close any old confirmation pane and reselect tables after upgrading.

A single fixed batch action prevents a stale button from combining the batch API version with a newly selected table URL.

### Workspace & target rendering error in Defender

The Subscription picker explicitly uses `typeSettings.includeAll: true` to list all subscriptions accessible to your portal identity. Without this setting, Defender can call its default-subscription provider when no Sentinel workspace context is available, throwing `Cannot read properties of null (reading 'subscriptionId')` and displaying **Error** for the entire **Workspace & target** block.

If an older copy shows this error, reimport the updated JSON. This setting does not grant permissions or change retention; an empty subscription list still requires checking the signed-in account and its Azure access.

## Update retention

1. Open **Tables** and enter **Target total retention (days)**.
2. Search the inventory and check the tables you want to change.
3. Review each table's decision and the **Selection readiness** summary.
4. Select a prepared-change row, then select **Review retention change**.
5. Inspect the table names, target, and request body. Confirm only if they match your intention.
6. Refresh **Read back this change** and verify every expected table.
7. Refresh the inventory and reselect remaining tables before another change.

**Total retention includes analytics retention.** A target of 365 means 365 days overall, not an additional 365 days.

### One workflow, automatic batching

| Eligible tables needing an increase | Request |
|---|---|
| 1 | One POST to the ARM batch endpoint, containing one PATCH. |
| 2-20 | One POST to the ARM batch endpoint, containing one PATCH per table. |
| More than 20 | Numbered batches of up to 20 tables, with one confirmation per prepared row. |

One selection can contain up to **800 tables**, producing at most 40 batches. Nothing beyond a batch boundary is silently omitted. If several tables are selected but only one needs an increase, the workbook prepares a one-request batch.

Every table update contains only:

```json
{
  "properties": {
    "totalRetentionInDays": 365
  }
}
```

Every confirmation uses the fixed endpoint `POST /batch?api-version=2020-06-01`. Each inner PATCH URL uses the Tables API version `2026-03-01`. The batch API version is never sent directly to a table endpoint. Read-back is required even for a one-request batch; an outer success is not proof that its inner PATCH succeeded.

### Eligibility and safeguards

- Supported input: **30-4,383 whole days**, subject to the service's table-specific restrictions.
- Eligible **Analytics** and **Auxiliary** tables use the same retention-only path, including `Any`, `Classic`, and `DataCollectionRuleBased` subtypes.
- **Basic-plan, search-result, and restored tables are excluded** from this Lake-focused workflow.
- XDR-only tables outside the selected workspace are not listed.
- Missing required metadata, stale workspace selections, or conflicting snapshots block request preparation.
- Tables already at or above the target are skipped; the workbook does not intentionally model retention reductions.

**Search only filters the display.** It does not clear earlier selections. Inspect the exact table names in the preview and confirmation.

**An open confirmation pane is a snapshot.** If you change the selection, workspace, or target, close the pane and review the newly prepared change before confirming. Consistent batch routing prevents mixed methods and API versions; it does not make an already-open pane track later selections.

**The preview is not an atomic concurrency guard.** Another administrator could increase retention after your preview was generated. Confirming a stale request could overwrite that newer value. Refresh before confirming and avoid concurrent administration.

## Verify every change

An outer batch **HTTP 200 does not mean every PATCH succeeded**. A batch can contain successful and failed operations.

After submitting:

1. Refresh **Read back this change** before refreshing the inventory.
2. Check that every expected table appears.
3. Confirm total retention matches the reviewed target and provisioning state is **Succeeded**.
4. Compare analytics retention, inheritance, plan, and subtype with their pre-change values.
5. Refresh the same tables in Defender and confirm the settings there.

The read-back is a fresh table GET, **not a captured write receipt**. The native workbook action does not export its response into workbook parameters, so the workbook cannot display member write statuses as an execution report.

Missing rows, mismatched values, read errors, or incomplete provisioning require investigation. A matching current value does not prove which request caused it.

### Verify tab and Activity Log

In **Verify**, select **one or more tables** (up to 800). Current settings and recent activity load for the whole selection. There is no mode selector, historical-deployment view, or run ID to enter. Search does not clear previous selections; verify the checked rows and clear/reselect when changing workspaces. Invalid or stale cross-workspace selections are blocked.

**Current table settings** uses a fresh workspace-table GET, filtered to all selected names. Compare each table's total retention with the target above and check provisioning state, analytics retention, inheritance, plan, and subtype. Check that every expected table is present. If a row is missing or the read fails, refresh/reselect and investigate; an empty or partial result is not success.

**Recent table activity** loads automatically after selection, with a default 7-day lookback. Choose 1, 7, 30, or 90 days:

- Uses the native Activity Log API; no diagnostic setting or log export is required.
- Shows the table name, Azure's operation description, status, initiator, available description/error details, event stage, and correlation ID.
- **View event JSON** opens the full record returned by the Activity Log API. It can contain identity/network details; redact it before sharing. Azure does not always include the old/new retention values; blank fields mean the service did not supply them, not that a specific retention change has been proven.
- **Open Azure Activity Log** opens the workspace's original Azure history. It is not an event-specific deep link: adjust the portal time range and use the displayed event time/correlation ID to find the original record.
- Events can take **3-20 minutes** to appear.
- `tables/write` includes PATCH, PUT, deployments, and other administrators' changes. It does not identify your last workbook click automatically.
- The API request is scoped to the workspace, then filtered to the selected table resource IDs case-insensitively. Only the first returned page is shown; some selected-table events may be on later pages.
- A warning appears **only when a continuation link exists**. The previous empty "Activity history coverage" grid has been removed. Shorten the lookback or use Azure Activity Log for full history.

The settings and activity panels are independent: an Activity Log permissions error does not prove the PATCH failed. **Current batched PATCH updates create no deployment records**, so historical deployment lookup has been removed.

There is **no automatic retry or rollback**. If a batch partially succeeds, investigate and retry only the tables that still need a change.

## Forecast Data Lake storage costs

1. Open **Costs**.
2. Choose **USD - US dollar ($)** or **EUR - Euro (€)** and enter the customer's **Lake storage rate** in that currency per billed GB-month.
3. Choose **7, 30, or 90 full UTC days** of observed volume; the default is 30.
4. Select the tables to forecast. This selection is independent of the update selection.
5. Change the target retention and compare the spend columns below.

| Spend column | Meaning |
|---|---|
| **Est. current Lake spend** | Estimated monthly Lake storage with the current retention settings, not actual invoiced spend. |
| **Forecast Lake spend** | Estimated total monthly Lake storage with the proposed retention settings. **This is the column to use for the expected Lake-storage spend after extending retention.** |
| **Monthly difference** | Forecast minus estimated current spend: the additional monthly Lake-storage cost. |

The forecast reads actual table retention from ARM and hourly data volume from `Usage`. It does not add ingestion charges.

### Currency and decimal separators

- **No exchange-rate conversion:** EUR/USD selects the denomination of the rate and results. Enter the actual price in that currency. Switching to EUR does not convert a previously entered USD price; update the rate if the prices differ.
- **Rate input always uses a dot:** enter `0.03`, not `0,03`, with no currency symbol or thousands separator.
- **Displayed spend follows the portal's regional decimal format**, not the selected currency. Thousands grouping is disabled to reduce ambiguity. English/US formatting shows `$0.0005`; a comma-decimal locale can show `€0,0005`.
- Spend is shown with **2-4 decimal places**, so small values such as `0.0005` remain visible. Smaller amounts may round to zero at the displayed precision. Missing estimates remain blank with an explanatory status, not a currency-formatted zero.
- Money remains numeric in query results for sorting and export. Headers identify the currency and monthly period.

```text
Average GB/day = observed GB / observation days
Analytics paid Lake days = max(total retention - analytics retention, 0)
Lake-only paid days = total retention
Monthly storage cost = average GB/day x paid Lake days / 6 x storage rate
```

The divisor **6** is Microsoft's documented uniform Lake storage billing compression factor. Enter the compressed/billed GB-month rate; do not divide the rate by six again.

### Example

For 60 GB observed over 30 days, 90-day analytics retention, and an **illustrative** rate of 0.03 per billed GB-month:

| Target total retention | Paid Lake days | Estimated monthly storage |
|---|---:|---:|
| 180 days | 90 | 0.90 |
| 270 days | 180 | 1.80 |
| 365 days | 275 | 2.75 |

These are example calculations, not a price quote. With EUR selected and a EUR rate, the results are euros; with USD selected and a USD rate, they are dollars. No currency conversion occurs.

### Forecast limitations

- Observed volume is `sum(Usage.Quantity) / 1000` in decimal GB. **It is not physical table size or measured Lake occupancy.**
- The forecast assumes future volume continues at the observed daily average.
- Both current and proposed costs are steady-state estimates, not invoices. Additional stored data accumulates gradually.
- Increasing retention does not restore deleted data or backfill data from before Lake onboarding.
- Ingestion, processing, queries, analytics storage, tax, and changes to discounts are excluded.
- Missing volume is shown as **NO ESTIMATE**, not zero cost. Partial or stale coverage is flagged.
- If any selected table cannot be estimated, the combined monetary total is withheld.

## Validation and support

Earlier validation included real two-table batched PATCH, an identical repeat, and fresh read-back. In that controlled batch test, full resource comparison showed changes only to total retention and its derived archive-retention value; schema, subtype, plan, analytics retention, inheritance, and protection settings stayed unchanged.

The batch workflow passed schema/preservation checks and live read-only query tests for batch boundaries, request contents, invalid input, and stale or tampered reviews. Defender confirmation previews were checked for rapid one-to-many and many-to-one selection changes, including delayed readiness responses. These tests submitted no retention updates, and both test tables remained unchanged.

Query tests covered exact batch boundaries, selection validation, request contents, stale reviews, and cost arithmetic. This is not certification of every table type, connector, or tenant configuration.

The batch approach follows an example in the [Azure-Sentinel Archive Log Tool workbook](https://github.com/Azure/Azure-Sentinel/blob/master/Tools/Archive-Log-Tool/ArchiveLogsTool-Workbook/Basic-Archive-Logs-Tool.json). That repository example is practical prior art, **not a complete published REST support contract**. Test in your environment and retain the read-back checks.

For issues, use this repository's GitHub Issues and include the workbook revision, relevant status/error, and sanitized reproduction steps. **Do not attach access tokens, cookies, credentials, or an unredacted browser HAR.**

## References and acknowledgements

- [Microsoft Sentinel table settings](https://learn.microsoft.com/azure/sentinel/manage-table-tiers-retention)
- [Microsoft Sentinel pricing and billing](https://learn.microsoft.com/azure/sentinel/billing)
- [Log Analytics retention and PUT/PATCH behavior](https://learn.microsoft.com/azure/azure-monitor/logs/data-retention-configure)
- [Usage table reference](https://learn.microsoft.com/azure/azure-monitor/reference/tables/usage)
- [Workbook ARM actions](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-link-actions)
- [Azure Activity Log](https://learn.microsoft.com/azure/azure-monitor/platform/activity-log)
- [Original Sentinel retention workbook](https://github.com/Azure/Azure-Sentinel/blob/master/Workbooks/ArchivingBasicLogsRetention.json)
- [Nicolas Suter's table-retention article](https://nicolasuter.ch/sentinel-table-retention-bicep/)

Microsoft's upstream workbook patterns are MIT-licensed. The required notice is included below.

## License

MIT License

Copyright (c) Microsoft Corporation. All rights reserved.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
