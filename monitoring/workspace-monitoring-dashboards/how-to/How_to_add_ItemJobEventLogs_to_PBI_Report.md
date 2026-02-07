# How to Add ItemJobEventLogs to Power BI Workspace Monitoring Report

## Overview

This guide walks you through adding **ItemJobEventLogs** data to the Fabric Workspace Monitoring Power BI Report template. ItemJobEventLogs captures job-level execution events for Pipelines, Notebooks, Lakehouses, Warehouses, Dataflows, CopyJobs, MLExperiments, and 20+ other Fabric item types.

Following this guide, you will add three new report pages:
1. **Item Jobs Overview** - High-level metrics and trends
2. **Item Jobs Failure Analysis** - Deep dive into failures
3. **Item Jobs Performance** - Duration analysis and performance insights

## Prerequisites

- Power BI Desktop installed
- Access to the Monitoring Eventhouse KQL database in your Fabric workspace
- The Workspace Monitoring Power BI template (.pbit file) imported and connected to your data source

## Step 1: Add ItemJobEventLogs Data Source

### 1.1 Create a New KQL Query

1. In Power BI Desktop, open the Workspace Monitoring report
2. Click **Home** > **Transform data** > **Transform data** to open Power Query Editor
3. Click **New Source** > **More...**
4. Search for and select **Azure Data Explorer (Kusto)**
5. Click **Connect**

### 1.2 Configure Connection

1. Enter your **Monitoring Eventhouse Query URI** (same as used for other tables)
2. Click **OK**
3. Select **DirectQuery** mode (recommended for large datasets)
4. Click **OK**

### 1.3 Add Base Query for Overview

Create a new query named **`ItemJobEventLogs_Overview`** with this KQL:

```kql
ItemJobEventLogs
| where Timestamp > ago(30d)
| summarize 
    TotalJobs = count(),
    SucceededJobs = countif(JobStatus == "Completed"),
    FailedJobs = countif(JobStatus == "Failed"),
    InProgressJobs = countif(JobStatus == "InProgress"),
    AvgDurationMs = avg(DurationMs),
    P95DurationMs = percentile(DurationMs, 95)
  by WorkspaceName, Timestamp = bin(Timestamp, 1h)
```

### 1.4 Add Additional Queries

Create these additional queries following the same process:

**Query: `ItemJobEventLogs_ByItemKind`**
```kql
ItemJobEventLogs
| where Timestamp > ago(30d)
| summarize 
    TotalJobs = count(),
    FailedJobs = countif(JobStatus == "Failed"),
    AvgDurationMs = avg(DurationMs)
  by ItemKind, JobType
| order by TotalJobs desc
```

**Query: `ItemJobEventLogs_Timeline`**
```kql
ItemJobEventLogs
| where Timestamp > ago(30d)
| summarize 
    JobCount = count(),
    FailCount = countif(JobStatus == "Failed")
  by bin(Timestamp, 1h), ItemKind
```

**Query: `ItemJobEventLogs_FailureDetails`**
```kql
ItemJobEventLogs
| where Timestamp > ago(30d) and JobStatus == "Failed"
| project Timestamp, ItemName, ItemKind, JobType, WorkspaceName, 
         DurationMs, ExecutingPrincipalId, JobStartTime, JobEndTime,
         JobInstanceId, JobInvokeType
| order by Timestamp desc
```

**Query: `ItemJobEventLogs_TopItems`**
```kql
ItemJobEventLogs
| where Timestamp > ago(30d)
| summarize 
    TotalRuns = count(),
    Failures = countif(JobStatus == "Failed"),
    AvgDurationMs = avg(DurationMs),
    MaxDurationMs = max(DurationMs),
    FailRate = round(100.0 * countif(JobStatus == "Failed") / count(), 2)
  by ItemName, ItemKind, ItemId
| order by TotalRuns desc
| take 50
```

**Query: `ItemJobEventLogs_ScheduledVsOnDemand`**
```kql
ItemJobEventLogs
| where Timestamp > ago(30d)
| summarize 
    RunCount = count(),
    AvgDurationMs = avg(DurationMs),
    FailRate = round(100.0 * countif(JobStatus == "Failed") / count(), 2)
  by JobInvokeType, ItemKind
```

### 1.5 Apply and Close

1. Click **Close & Apply** in Power Query Editor
2. Wait for the data to load

## Step 2: Create Item Jobs Overview Page

### 2.1 Add New Page

1. Click the **+** icon at the bottom to add a new report page
2. Rename the page to **"Item Jobs Overview"**

### 2.2 Add Stat Cards (KPIs)

Add the following card visuals at the top of the page:

**Card 1: Total Jobs**
- Visual: Card
- Field: `ItemJobEventLogs_Overview[TotalJobs]` (use SUM aggregation)
- Position: Top-left
- Size: 200px × 150px

**Card 2: Success Rate %**
- Visual: Card
- Field: Create a measure:
  ```DAX
  Success Rate % = 
  DIVIDE(
      SUM(ItemJobEventLogs_Overview[SucceededJobs]),
      SUM(ItemJobEventLogs_Overview[TotalJobs]),
      0
  ) * 100
  ```
- Format: Percentage with 1 decimal place
- Position: Next to Total Jobs
- Size: 200px × 150px

**Card 3: Failed Jobs**
- Visual: Card
- Field: `ItemJobEventLogs_Overview[FailedJobs]` (SUM)
- Conditional formatting: Red if value > 0
- Position: Next to Success Rate
- Size: 200px × 150px

**Card 4: Avg Duration**
- Visual: Card
- Field: `ItemJobEventLogs_Overview[AvgDurationMs]` (AVERAGE)
- Format: Display as seconds: Create measure:
  ```DAX
  Avg Duration (sec) = AVERAGE(ItemJobEventLogs_Overview[AvgDurationMs]) / 1000
  ```
- Position: Next to Failed Jobs
- Size: 200px × 150px

**Card 5: P95 Duration**
- Visual: Card
- Field: `ItemJobEventLogs_Overview[P95DurationMs]` (AVERAGE)
- Format: Display as seconds
- Position: Next to Avg Duration
- Size: 200px × 150px

### 2.3 Add Time Series Charts

**Area Chart: Job Count Over Time by Item Kind**
- Visual: Area chart
- X-axis: `ItemJobEventLogs_Timeline[Timestamp]`
- Y-axis: `ItemJobEventLogs_Timeline[JobCount]` (SUM)
- Legend: `ItemJobEventLogs_Timeline[ItemKind]`
- Position: Below cards, left half
- Size: 600px × 300px

**Line Chart: Failure Count Over Time**
- Visual: Line chart
- X-axis: `ItemJobEventLogs_Timeline[Timestamp]`
- Y-axis: `ItemJobEventLogs_Timeline[FailCount]` (SUM)
- Line color: Red
- Position: Below cards, right half
- Size: 600px × 300px

### 2.4 Add Bar Chart and Table

**Bar Chart: Jobs by Item Kind**
- Visual: Clustered bar chart
- Y-axis: `ItemJobEventLogs_ByItemKind[ItemKind]`
- X-axis: `ItemJobEventLogs_ByItemKind[TotalJobs]` (SUM)
- Position: Middle section, left
- Size: 600px × 300px

**Table: Top 50 Items by Run Count**
- Visual: Table
- Columns:
  - `ItemJobEventLogs_TopItems[ItemName]`
  - `ItemJobEventLogs_TopItems[ItemKind]`
  - `ItemJobEventLogs_TopItems[TotalRuns]`
  - `ItemJobEventLogs_TopItems[Failures]`
  - `ItemJobEventLogs_TopItems[AvgDurationMs]`
  - `ItemJobEventLogs_TopItems[MaxDurationMs]`
  - `ItemJobEventLogs_TopItems[FailRate]`
- Position: Bottom section, full width
- Size: 1200px × 400px
- Format: Add conditional formatting for FailRate (red if > 5%)

## Step 3: Create Item Jobs Failure Analysis Page

### 3.1 Add New Page

1. Add another new page
2. Rename to **"Item Jobs - Failure Analysis"**

### 3.2 Add Stat Cards

**Card 1: Total Failures**
- Visual: Card
- Field: `ItemJobEventLogs_FailureDetails[ItemName]` (COUNT)
- Conditional formatting: Red background
- Position: Top-left
- Size: 250px × 150px

**Card 2: Most Failing Item**
- Visual: Card
- Field: Create measure:
  ```DAX
  Most Failing Item = 
  TOPN(1, 
      SUMMARIZE(
          ItemJobEventLogs_FailureDetails,
          ItemJobEventLogs_FailureDetails[ItemName]
      ),
      COUNTROWS(ItemJobEventLogs_FailureDetails),
      DESC
  )
  ```
- Position: Next to Total Failures
- Size: 250px × 150px

**Card 3: Most Failing Item Kind**
- Visual: Card
- Field: Similar measure for ItemKind
- Position: Next to Most Failing Item
- Size: 250px × 150px

### 3.3 Add Charts

**Bar Chart: Failures by Item Kind**
- Visual: Clustered bar chart
- Y-axis: `ItemJobEventLogs_FailureDetails[ItemKind]`
- X-axis: Count of failures
- Position: Middle section
- Size: 600px × 300px

**Line Chart: Failure Trend Over Time**
- Visual: Line chart
- X-axis: `ItemJobEventLogs_Timeline[Timestamp]`
- Y-axis: `ItemJobEventLogs_Timeline[FailCount]` (SUM)
- Line color: Red
- Position: Top-right
- Size: 600px × 300px

### 3.4 Add Failure Details Table

**Table: Detailed Failure List**
- Visual: Table
- Columns:
  - `ItemJobEventLogs_FailureDetails[Timestamp]`
  - `ItemJobEventLogs_FailureDetails[ItemName]`
  - `ItemJobEventLogs_FailureDetails[ItemKind]`
  - `ItemJobEventLogs_FailureDetails[JobType]`
  - `ItemJobEventLogs_FailureDetails[WorkspaceName]`
  - `ItemJobEventLogs_FailureDetails[DurationMs]`
  - `ItemJobEventLogs_FailureDetails[ExecutingPrincipalId]`
  - `ItemJobEventLogs_FailureDetails[JobInvokeType]`
- Position: Bottom section, full width
- Size: 1200px × 500px
- Enable drill-through to see more details

## Step 4: Create Item Jobs Performance Page

### 4.1 Add New Page

1. Add another new page
2. Rename to **"Item Jobs - Performance"**

### 4.2 Add Performance Cards

**Card 1: Avg Duration**
- Visual: Card
- Field: `ItemJobEventLogs_Overview[AvgDurationMs]` (AVERAGE) / 1000 (convert to seconds)

**Card 2: P95 Duration**
- Visual: Card
- Field: `ItemJobEventLogs_Overview[P95DurationMs]` (AVERAGE) / 1000

**Card 3: Max Duration**
- Visual: Card
- Field: Create measure:
  ```DAX
  Max Duration (sec) = 
  MAX(ItemJobEventLogs_TopItems[MaxDurationMs]) / 1000
  ```

### 4.3 Add Performance Charts

**Bar Chart: Avg Duration by Item Kind**
- Visual: Clustered bar chart
- Y-axis: `ItemJobEventLogs_ByItemKind[ItemKind]`
- X-axis: `ItemJobEventLogs_ByItemKind[AvgDurationMs]` (AVERAGE)
- Position: Left side
- Size: 600px × 400px

**Table: Scheduled vs On-Demand Comparison**
- Visual: Table
- Columns:
  - `ItemJobEventLogs_ScheduledVsOnDemand[JobInvokeType]`
  - `ItemJobEventLogs_ScheduledVsOnDemand[ItemKind]`
  - `ItemJobEventLogs_ScheduledVsOnDemand[RunCount]`
  - `ItemJobEventLogs_ScheduledVsOnDemand[AvgDurationMs]`
  - `ItemJobEventLogs_ScheduledVsOnDemand[FailRate]`
- Position: Right side
- Size: 600px × 400px

### 4.4 Add Slowest Jobs Table

**Query for Slowest Jobs** (Add to Power Query):
```kql
ItemJobEventLogs
| where Timestamp > ago(30d)
| top 20 by DurationMs desc
| project Timestamp, ItemName, ItemKind, JobType, DurationMs, JobStatus
```

**Table: Slowest Jobs**
- Visual: Table
- Show top 20 jobs by duration
- Position: Bottom section, full width
- Size: 1200px × 400px

## Step 5: Add Slicers and Filters

### 5.1 Add Workspace Slicer

On each page, add a slicer for WorkspaceName:
- Visual: Slicer
- Field: `ItemJobEventLogs_Overview[WorkspaceName]`
- Position: Top of page
- Enable "Select All" option

### 5.2 Add Date Range Filter

Add a date slicer:
- Visual: Date slicer
- Field: `ItemJobEventLogs_Overview[Timestamp]`
- Position: Top of page, next to workspace slicer
- Set default to "Last 30 days"

## Step 6: Format and Style

### 6.1 Apply Consistent Theme

1. Match the color scheme of existing pages:
   - Primary color: Blue (#0078D4)
   - Success: Green (#107C10)
   - Warning: Yellow (#FFB900)
   - Error: Red (#D13438)

2. Use consistent fonts:
   - Titles: Segoe UI Semibold, 14pt
   - Cards: Segoe UI, 28pt
   - Tables: Segoe UI, 10pt

### 6.2 Add Page Navigation

1. Add a navigation bar at the top linking to all Item Jobs pages
2. Use buttons with icons for better UX

## Step 7: Enable Drill-Through

### 7.1 Create Drill-Through Page

1. Create a new page named "Item Job Details"
2. Right-click the page tab > **Edit drill through filters**
3. Add `ItemJobEventLogs_FailureDetails[ItemId]` as drill-through field
4. Add detailed information about the selected item

### 7.2 Configure Drill-Through Actions

On the Failure Analysis table:
1. Right-click on ItemName column
2. Enable drill-through to "Item Job Details" page

## Step 8: Test and Validate

### 8.1 Test Filters

1. Select different workspaces in the slicer
2. Verify all visuals update correctly
3. Test date range filtering

### 8.2 Verify Calculations

1. Check that totals match across pages
2. Verify success rate calculations
3. Confirm failure counts are accurate

### 8.3 Performance Check

1. Monitor query performance
2. If slow, consider adding aggregations
3. Review DirectQuery optimization options

## Best Practices

1. **Use DirectQuery** for ItemJobEventLogs to handle large datasets
2. **Create DAX measures** for complex calculations rather than calculated columns
3. **Add tooltips** to explain metrics to end users
4. **Enable cross-filtering** between visuals on the same page
5. **Set appropriate refresh schedules** if using Import mode
6. **Document custom measures** for future maintenance

## Troubleshooting

### Issue: No data showing
- **Solution**: Verify Monitoring Eventhouse connection is active
- Check that ItemJobEventLogs table has data for the selected time range

### Issue: Slow performance
- **Solution**: Use DirectQuery mode
- Add filters to reduce data volume
- Consider creating aggregated tables for historical data

### Issue: Calculations incorrect
- **Solution**: Review measure formulas
- Check for NULL handling in calculations
- Verify time zone conversions if needed

## Additional Resources

- [Workspace Monitoring Overview](https://learn.microsoft.com/en-us/fabric/fundamentals/workspace-monitoring-overview)
- [Power BI DirectQuery Best Practices](https://learn.microsoft.com/en-us/power-bi/connect-data/desktop-directquery-about)
- [KQL Query Language Reference](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/)

---

**Template Version:** 2025.8.2+ItemJobEventLogs

**Last Updated:** February 2026

For questions or issues, please create an issue in the [Fabric Toolbox GitHub repository](https://github.com/microsoft/fabric-toolbox).
