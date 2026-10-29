# Sentinel Table Retention Workbook

## Overview

This workbook helps security and operations teams manage total retention across tables in a Microsoft Sentinel workspace without opening and updating each table separately.

Use it to review existing settings, select the tables to update, estimate the Data Lake storage impact, and verify the results. The workflow runs inside Azure Workbooks using your portal identity; no scripts, deployments, or helper services are required.

**Current version: 1.0**

> [!NOTE]
> For guidance on deciding which data to retain and for how long, see the [Sentinel Maturity Model retention guidance](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/guidance/retention.md). The workbook helps apply and verify those decisions; it does not choose a retention policy for your organization.

## Key features and use cases

- **Table retention management:** search and select eligible tables, review the exact scope, and confirm retention-only updates.
- **Storage cost planning:** compare estimated current and proposed monthly Lake storage costs using observed volume and your storage rate.
- **Change verification:** read current settings for multiple tables and inspect recent Azure Activity Log events.
- **One consistent workflow:** eligible updates use the same batch method, with explicit confirmation and no additional infrastructure.

## Workbook views

| View | What you can do |
|------|-----------------|
| **Tables** | Select tables, review proposed changes, and submit a prepared batch. |
| **Costs** | Forecast current, proposed, and additional monthly Lake storage costs. This view does not change settings. |
| **Verify** | Read current table settings and investigate recent activity, including available error details and source links. |

## Prerequisites

> [!WARNING]
> Applying an update changes Azure resource settings, and longer retention can increase storage costs. Start with a small, noncritical selection and verify the results before broader use.

- A Microsoft Sentinel Data Lake-integrated Log Analytics workspace.
- Permission to read and update the workspace's table settings. See [Configure table settings](https://learn.microsoft.com/azure/sentinel/manage-table-tiers-retention) for the required permissions.
- Workspace query access for the selection and review queries.
- Read access to the workspace's `Usage` table for cost forecasts.
- Activity Log read access for recent table operations in **Verify**.

Defender portal access alone does not grant the required Azure permissions. Updates use your portal identity and Azure RBAC; no deployment-write permission or helper identity is needed.

## Deployment and first use

1. Download [SentinelDataLakeRetention.json](SentinelDataLakeRetention.json) using **Raw** or **Download raw file** on GitHub.
2. In Microsoft Defender, open **Microsoft Sentinel > Threat management > Workbooks** and add a workbook. Azure Monitor Workbooks is also supported.
3. Select **Edit > Advanced editor**.
4. Paste the downloaded JSON, then select **Apply** and **Done Editing**.
5. Select your subscription and workspace, and confirm that the table inventory loads.
6. Save a copy in your environment. Review a small selection and its cost forecast before confirming any updates.

### Initial settings

- **Target total retention:** defaults to 365 days. Total retention includes analytics retention; it is not an additional 365 days. Input accepts 30-4,383 whole days, subject to table-specific service restrictions.
- **Volume observation window:** choose 7, 30, or 90 full UTC days; the default is 30.
- **Storage pricing:** choose USD or EUR and enter your Lake storage rate per billed GB-month. Use a dot decimal, such as `0.03`.
- **Recent activity:** defaults to 7 days, with 1-, 7-, 30-, and 90-day options.
- **Detailed guidance:** select **Show** to see additional help and the **Version information** table. Public release history starts at 1.0.

When upgrading an existing copy, reimport the JSON, close any old confirmation pane, and reselect the tables.

### Apply a retention change

1. Open **Tables**, enter the target, and check the tables you want to update.
2. Review each table's decision and the **Selection readiness** summary.
3. Select a prepared-change row, then select **Review retention change**.
4. Check the table names, target, and request body. Confirm only if they match your intention.
5. Refresh **Read back this change** and verify every expected table before refreshing the inventory.
6. Refresh the inventory and reselect any remaining tables before another change.

Eligible updates are grouped into batches of up to **20 tables**, with one confirmation per prepared row. A selection of 45 eligible tables produces batches of 20, 20, and 5. One selection can contain up to **800 tables**; nothing beyond the first batch is silently omitted.

## Understanding the results

### Tables: decisions and scope

| Decision | Meaning |
|----------|---------|
| **INCREASE** | The table is eligible and its current total retention is below the target. |
| **SKIPPED** | No update is needed or supported for this table. Read the row's reason; tables already at or above the target keep their current setting. |
| **BLOCKED** | Invalid input, missing metadata, or a stale/conflicting selection prevents request preparation. Refresh and review the reported reason. |

Only eligible tables in the selected workspace can be updated. Search-result and restored tables are excluded.

**Search only filters the display; it does not clear earlier selections.** Check the exact table names in the preview and confirmation. The **Costs** and **Verify** views have their own selections.

### Costs: estimated monthly storage spend

Choose the observation window, currency, storage rate, and tables to forecast. The estimate uses observed data volume and the table retention settings.

| Spend column | Meaning |
|--------------|---------|
| **Est. current Lake spend** | Estimated monthly Lake storage with the current settings, not actual invoiced spend. |
| **Forecast Lake spend** | Estimated monthly Lake storage with the proposed settings. |
| **Monthly difference** | Forecast minus estimated current spend. |

These are steady-state estimates, not the next invoice. Additional retained data accumulates gradually. Missing volume produces **NO ESTIMATE**, not zero cost; the combined monetary total is withheld if any selected table cannot be estimated.

USD/EUR selects the denomination of the rate and results; it does not convert prices. Enter the matching rate when changing currency. Displayed decimal separators follow your portal's regional format, with 2-4 decimal places.

### Verify: current settings and supporting activity

> [!IMPORTANT]
> A successful batch notification does not prove that every table changed. An outer HTTP 200 response can contain individual failures. Always read back the settings.

After submitting an update:

1. Refresh **Read back this change**, or select the tables in **Verify**.
2. Check that every expected table is present and total retention matches the reviewed target.
3. Check that provisioning state is **Succeeded** and analytics retention and other table properties are unchanged.
4. Refresh the same tables in Defender and confirm the settings there.

**Recent table activity** loads automatically for the selected tables. It shows Azure's operation description, status, initiator, and available error details and correlation IDs. No diagnostic setting or log export is required.

- **View event JSON** opens the returned activity record. It may contain identity/network details; redact these before sharing.
- **Open Azure Activity Log** opens the workspace's history. Use the event time and correlation ID to find the original record; the link is not event-specific.
- Azure may omit old/new retention values, and table-write events can also come from other administrators or tools. A matching current value does not prove which request caused it.

An empty or partial read-back is not success. Investigate missing rows, mismatched settings, read errors, or incomplete provisioning.

## Troubleshooting and limitations

| Symptom or limitation | What to check |
|-----------------------|---------------|
| No workspace listed, or **Workspace & target** shows an error | Reimport the latest workbook and check the signed-in account's subscription/workspace access. |
| No prepared change | Review the target, selected rows, and readiness reasons. Refresh the inventory and reselect; some tables may already meet the target or be ineligible. |
| **NO ESTIMATE** or no combined cost total | Check the storage rate, query permissions, observation window, and `Usage` coverage. Missing data is not zero usage. |
| Unexpected currency or decimal formatting | Enter the rate with a dot decimal. Match the rate to the selected currency; no exchange-rate conversion is performed. Output uses your regional format. |
| No recent activity | Events can take 3-20 minutes to appear. Check the lookback and Activity Log permissions; activity access is independent of table-settings access. |
| Incomplete activity history | Only the first API page is displayed. A warning indicates more history is available; shorten the lookback or open Azure Activity Log. |
| Some tables did not change | A batch can partially succeed. Investigate the failed or missing results, then refresh and retry only the tables that still need an update. There is no automatic retry or rollback. |

**Before confirming:** an open confirmation pane is a snapshot. If you change the selection, workspace, or target, close it and review the newly prepared change. The preview does not lock the settings against another administrator's changes; refresh first and avoid concurrent administration.

**Forecast limitations:** the model assumes future volume follows the observed daily average. It does not measure physical Lake occupancy and excludes ingestion, processing, queries, analytics storage, tax, and changes to discounts. Increasing retention does not restore deleted data or backfill data from before Lake onboarding.

For issues, include the workbook version, relevant status/error, and sanitized reproduction steps in [GitHub Issues](https://github.com/mathijsvermaat/Sentinel-Table-Retention-Workbook/issues). Do not share access tokens, cookies, credentials, or an unredacted browser HAR.

## How it works / technical details

### Data sources

| Source | Purpose |
|--------|---------|
| Azure Resource Graph | Discover accessible Log Analytics workspaces. |
| ARM Tables API | Read table settings and submit explicitly confirmed retention updates. |
| `Usage` | Aggregate observed table volume for storage forecasts. |
| Azure Activity Log | Provide supporting operation/status evidence for selected tables. |

### Retention requests

Each prepared change uses `POST /batch?api-version=2020-06-01`. Each inner request is a PATCH to its table URL using API version `2026-03-01`, with only this property:

```json
{
  "properties": {
    "totalRetentionInDays": 365
  }
}
```

Schema, table plan, and analytics-retention settings are not submitted. The same batch endpoint is used for every selection. Read-back uses a fresh GET, not a captured receipt from the write; the workbook does not display individual batch-response statuses as an execution report.

### Storage calculation

```text
Observed GB = sum(Usage.Quantity in MBytes) / 1000
Average GB/day = observed GB / full UTC observation days
Proposed total retention = max(current total retention, target)
Paid Lake days = max(total retention - applicable analytics allowance, 0)
Monthly storage cost = average GB/day x paid Lake days / 6 x storage rate
```

For tables mirrored from analytics, the allowance is their analytics retention. For Lake-only tables, it is zero, so all retained days are modeled as billable.

The divisor **6** is Microsoft's documented Lake storage billing compression factor. Enter the rate per billed GB-month; do not divide it by six again.

For a table currently at **90 days total and 90 days analytics retention**, 60 GB observed over 30 days, and an **illustrative** rate of 0.03 per billed GB-month:

| Target total retention | Paid Lake days | Estimated monthly storage |
|------------------------|---------------:|--------------------------:|
| 180 days | 90 | 0.90 |
| 270 days | 180 | 1.80 |
| 365 days | 275 | 2.75 |

These are example calculations, not a price quote. Amounts use the currency of the entered rate.

## Related resources

- [Sentinel Maturity Model retention guidance](https://github.com/mathijsvermaat/Sentinel-Maturity/blob/main/guidance/retention.md)
- [Microsoft Sentinel table settings](https://learn.microsoft.com/azure/sentinel/manage-table-tiers-retention)
- [Microsoft Sentinel pricing and billing](https://learn.microsoft.com/azure/sentinel/billing)
- [Log Analytics retention and PUT/PATCH behavior](https://learn.microsoft.com/azure/azure-monitor/logs/data-retention-configure)
- [Tables - Update REST API](https://learn.microsoft.com/en-us/rest/api/loganalytics/tables/update?view=rest-loganalytics-2026-03-01)
- [Usage table reference](https://learn.microsoft.com/azure/azure-monitor/reference/tables/usage)
- [Workbook ARM actions](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-link-actions)
- [Azure Activity Log](https://learn.microsoft.com/azure/azure-monitor/platform/activity-log)
- [Original Sentinel retention workbook](https://github.com/Azure/Azure-Sentinel/blob/master/Workbooks/ArchivingBasicLogsRetention.json)
- [Microsoft's native batch workbook example](https://github.com/Azure/Azure-Sentinel/blob/master/Tools/Archive-Log-Tool/ArchiveLogsTool-Workbook/Basic-Archive-Logs-Tool.json)

[Nicolas Suter's article on managing table retention with Bicep](https://nicolasuter.ch/sentinel-table-retention-bicep/). His guide is also a useful starting point for teams that prefer to manage retention as code.