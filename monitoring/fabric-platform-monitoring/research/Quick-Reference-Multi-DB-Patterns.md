# Quick Reference: Multi-Database KQL Patterns

**Quick reference guide for common multi-database query patterns in Fabric Platform Monitoring**

---

## Pattern 1: Basic Cross-Database Union

**Use Case:** Combine data from two or more databases

```kql
// Simple union
database("ActivityEvents").ActivityEvents
| where CreationTime > ago(1h)
| extend Source = "Activity"
| union (
    database("CapacityUtilization").CapacitySummary
    | where windowStartTime > ago(1h)
    | extend Source = "Capacity"
)
| summarize Count=count() by Source
```

**Performance Tips:**
- Filter BEFORE union
- Project only needed columns
- Use same time range in all branches

---

## Pattern 2: Dashboard Parameter-Based Filtering

**Use Case:** Let users select which data sources to include

```kql
// Dashboard parameter: _sources (multi-select: "Activity", "Capacity", "Gateway")

let allData =
    database("ActivityEvents").ActivityEvents
    | where CreationTime between (_startTime .. _endTime)
    | extend Source = "Activity", Timestamp = CreationTime
    | project Timestamp, Source, Id, UserId
    | union (
        database("CapacityUtilization").CapacitySummary
        | where windowStartTime between (_startTime .. _endTime)
        | extend Source = "Capacity", Timestamp = windowStartTime
        | project Timestamp, Source, Id=id, UserId=""
    )
    | union (
        database("GatewayMonitoring").GatewayJobs
        | where jobStartTime between (_startTime .. _endTime)
        | extend Source = "Gateway", Timestamp = jobStartTime
        | project Timestamp, Source, Id=jobId, UserId=""
    );

allData
| where Source in (_sources)  // Apply user filter
| summarize Count=count() by bin(Timestamp, 5m), Source
| render timechart
```

---

## Pattern 3: Time-Aligned Metrics from Multiple Databases

**Use Case:** Create aligned time-series metrics

```kql
let granularity = 5m;

let activityMetrics = database("ActivityEvents").ActivityEvents
    | where CreationTime between (_startTime .. _endTime)
    | summarize Count=count() by Timestamp=bin(CreationTime, granularity);

let capacityMetrics = database("CapacityUtilization").CapacitySummary
    | where windowStartTime between (_startTime .. _endTime)
    | summarize AvgUtil=avg(utilizationInteractive + utilizationBackground)
      by Timestamp=bin(windowStartTime, granularity);

activityMetrics
| join kind=fullouter capacityMetrics on Timestamp
| project
    Timestamp = coalesce(Timestamp, Timestamp1),
    ActivityCount = coalesce(Count, 0L),
    AvgUtilization = coalesce(AvgUtil, 0.0)
| order by Timestamp asc
| render timechart
```

---

## Pattern 4: Function-Based Abstraction

**Use Case:** Centralize cross-database logic

```kql
// Create function in Core database
.create-or-alter function GetPlatformEvents(startTime:datetime, endTime:datetime) {
    database("ActivityEvents").ActivityEvents
    | where CreationTime between (startTime .. endTime)
    | extend Source = "Activity"
    | project Timestamp=CreationTime, Source, EventType=Activity, UserId
    | union (
        database("CapacityUtilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | extend Source = "Capacity"
        | project Timestamp=windowStartTime, Source, EventType="CapacitySummary", UserId=""
    )
}

// Use in dashboards
GetPlatformEvents(ago(24h), now())
| summarize Count=count() by Source
```

---

## Pattern 5: Correlation Across Databases

**Use Case:** Find relationships between events in different databases

```kql
// Find activities during high capacity utilization
let highCapacity = database("CapacityUtilization").CapacitySummary
    | where windowStartTime > ago(1d)
    | where utilizationInteractive + utilizationBackground > 90
    | project CapTime=windowStartTime, CapId=capacityId, Util=utilizationInteractive + utilizationBackground;

let activities = database("ActivityEvents").ActivityEvents
    | where CreationTime > ago(1d)
    | extend CapId = toguid(details.CapacityId)
    | where isnotempty(CapId)
    | project ActTime=CreationTime, CapId, Activity, UserId;

highCapacity
| join kind=inner activities on CapId
| where abs(datetime_diff('minute', CapTime, ActTime)) <= 5  // 5-minute window
| summarize
    ActivityCount=count(),
    Activities=make_set(Activity)
  by CapTime, CapId, Util
| order by CapTime desc
```

---

## Pattern 6: Pre-Aggregation for Performance

**Use Case:** Reduce data transfer by aggregating in each branch

```kql
// SLOW: Union raw data, then aggregate
database("DB1").LargeTable
| union database("DB2").LargeTable
| summarize Count=count() by Category

// FAST: Aggregate first, then union
database("DB1").LargeTable
| summarize Count=count() by Category
| extend Source = "DB1"
| union (
    database("DB2").LargeTable
    | summarize Count=count() by Category
    | extend Source = "DB2"
)
| summarize TotalCount=sum(Count) by Category
```

---

## Pattern 7: Dynamic Time Granularity

**Use Case:** Adjust granularity based on time range

```kql
let granularity = case(
    _endTime - _startTime >= 30d, 1d,
    _endTime - _startTime >= 7d, 1h,
    _endTime - _startTime >= 1d, 5m,
    1m
);

database("ActivityEvents").ActivityEvents
| where CreationTime between (_startTime .. _endTime)
| union (
    database("CapacityUtilization").CapacitySummary
    | where windowStartTime between (_startTime .. _endTime)
)
| summarize Count=count() by bin(Timestamp, granularity)
```

---

## Pattern 8: Workspace-Centric Multi-Source Query

**Use Case:** Get all data related to a specific workspace

```kql
let wsId = _workspaceId;  // Dashboard parameter

let activities = database("ActivityEvents").ActivityEvents
    | where toguid(details.WorkspaceId) == wsId
    | where CreationTime > ago(7d)
    | summarize ActivityCount=count(), UniqueUsers=dcount(UserId) by Activity;

let wsInfo = database("PlatformInventory").Workspaces
    | where id == wsId
    | project WorkspaceName=name, CapacityId=capacityId, State=state;

let capacityMetrics = database("CapacityUtilization").CapacitySummary
    | where windowStartTime > ago(7d)
    | where capacityId in ((wsInfo | project CapacityId))
    | summarize AvgUtilization=avg(utilizationInteractive + utilizationBackground);

activities
| extend WorkspaceId = wsId
| extend WorkspaceName = toscalar(wsInfo | project WorkspaceName)
| extend AvgCapacityUtil = toscalar(capacityMetrics)
```

---

## Pattern 9: Health Score Across Multiple Sources

**Use Case:** Calculate composite health metrics

```kql
let startTime = ago(1d);
let endTime = now();

// Activity health: Are users active?
let activityHealth = database("ActivityEvents").ActivityEvents
    | where CreationTime between (startTime .. endTime)
    | summarize TotalActivities=count()
    | extend ActivityHealthScore = min(TotalActivities / 10000.0 * 100, 100.0);

// Capacity health: Low utilization, no throttling
let capacityHealth = database("CapacityUtilization").CapacitySummary
    | where windowStartTime between (startTime .. endTime)
    | summarize
        AvgUtil = avg(utilizationInteractive + utilizationBackground),
        ThrottleEvents = countif(utilizationInteractive + utilizationBackground > 100)
    | extend CapacityHealthScore = max(100.0 - AvgUtil - (ThrottleEvents * 10), 0.0);

// Gateway health: Low failure rate
let gatewayHealth = database("GatewayMonitoring").GatewayJobs
    | where jobStartTime between (startTime .. endTime)
    | summarize TotalJobs=count(), FailedJobs=countif(status == "Failed")
    | extend FailureRate = FailedJobs * 100.0 / TotalJobs
    | extend GatewayHealthScore = max(100.0 - FailureRate * 10, 0.0);

// Combine
print
    OverallHealth = (
        toscalar(activityHealth | project ActivityHealthScore) +
        toscalar(capacityHealth | project CapacityHealthScore) +
        toscalar(gatewayHealth | project GatewayHealthScore)
    ) / 3.0
```

---

## Pattern 10: Anomaly Detection Across Sources

**Use Case:** Detect unusual patterns across multiple databases

```kql
let sensitivity = 1.5;

// Activity anomalies
let activityTS = database("ActivityEvents").ActivityEvents
    | where CreationTime > ago(7d)
    | make-series ActivityCount=count() on CreationTime step 5m
    | extend (anomalies, score, baseline) = series_decompose_anomalies(ActivityCount, sensitivity)
    | mv-expand Timestamp=CreationTime, ActivityCount, Anomaly=anomalies, Score=score
    | where Anomaly != 0
    | extend Source = "Activity";

// Capacity anomalies
let capacityTS = database("CapacityUtilization").CapacitySummary
    | where windowStartTime > ago(7d)
    | make-series AvgUtil=avg(utilizationInteractive + utilizationBackground)
      on windowStartTime step 5m
    | extend (anomalies, score, baseline) = series_decompose_anomalies(AvgUtil, sensitivity)
    | mv-expand Timestamp=windowStartTime, AvgUtil, Anomaly=anomalies, Score=score
    | where Anomaly != 0
    | extend Source = "Capacity";

activityTS
| union capacityTS
| project Timestamp=todatetime(Timestamp), Source, AnomalyScore=toreal(Score)
| order by abs(AnomalyScore) desc
| take 20
```

---

## Common Mistakes to Avoid

### Mistake 1: Filtering After Union

```kql
// BAD
database("DB1").Table | union database("DB2").Table
| where Timestamp > ago(1h)  // Combines ALL data first

// GOOD
database("DB1").Table | where Timestamp > ago(1h)
| union (database("DB2").Table | where Timestamp > ago(1h))
```

### Mistake 2: Using Variables for Database Names

```kql
// DOES NOT WORK
let dbName = "ActivityEvents";
database(dbName).Table  // Error: database() requires string literal

// WORKS
database("ActivityEvents").Table
```

### Mistake 3: Cross-Database Materialized Views

```kql
// DOES NOT WORK
.create materialized-view MV on table LocalTable {
    LocalTable | union database("OtherDB").Table  // Error
}

// WORKS: Use function instead
.create function UnifiedView() {
    LocalTable | union database("OtherDB").Table
}
```

### Mistake 4: Late Column Projection

```kql
// BAD: Projects all columns, then selects
database("DB1").Table
| union database("DB2").Table
| project Col1, Col2

// GOOD: Project early in each branch
database("DB1").Table | project Col1, Col2
| union (database("DB2").Table | project Col1, Col2)
```

---

## Performance Checklist

- [ ] Filter by time in EACH branch before union
- [ ] Project only necessary columns early
- [ ] Use hot cache for frequently accessed time ranges (7-30 days)
- [ ] Create materialized views for hourly+ aggregations
- [ ] Use functions to centralize complex cross-database queries
- [ ] Pre-aggregate in branches when possible
- [ ] Test query performance with realistic data volumes
- [ ] Monitor query duration and set alerts for slow queries (>10s)

---

## Dashboard Parameter Configuration

**Recommended Parameters for Multi-Database Dashboards:**

1. **Time Range** (duration)
   - Variable: `_startTime`, `_endTime`
   - Default: Last 1 hour to 24 hours

2. **Data Sources** (string array)
   - Variable: `_sources`
   - Values: `["Activity", "Capacity", "Gateway"]`
   - Default: All selected

3. **Granularity** (string)
   - Variable: `_granularity`
   - Values: `["1m", "5m", "1h", "1d"]`
   - Use with `totimespan(_granularity)`

4. **Workspace Filter** (string array from query)
   - Variable: `_workspaces`
   - Source: `database("PlatformInventory").Workspaces | project id`

---

## Quick Decision Tree

**Should I use cross-database queries?**

```
Is data in multiple databases? --> NO --> Use single database query
    |
    YES
    |
    Is query latency < 10s acceptable? --> NO --> Consider ETL/consolidation
    |
    YES
    |
    Is data volume < 5M rows per DB? --> NO --> Use pre-aggregated views
    |
    YES
    |
    Use cross-database union with proper filtering
```

---

## Additional Resources

- [Full Research Document](./KQL-Multi-Database-Union-Strategies.md) - Comprehensive guide
- [KQL Union Operator - Microsoft Docs](https://learn.microsoft.com/kusto/query/union-operator)
- [Cross-Database Queries - Microsoft Docs](https://learn.microsoft.com/kusto/query/cross-cluster-queries)

---

**Last Updated:** 2026-02-07
