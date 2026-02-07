# Implementation Template: Multi-Database Functions for Fabric Platform Monitoring

**Ready-to-deploy KQL functions for cross-database queries**

---

## Deployment Instructions

1. **Create a Core Database** (if not exists)
   - Name: `Platform Core` or use existing `Platform Inventory` database
   - This database will host all cross-database functions

2. **Deploy Functions**
   - Copy the `.create-or-alter` commands below
   - Execute in the Core database
   - Functions will be available immediately

3. **Update Dashboards**
   - Replace direct database queries with function calls
   - Add parameters as needed
   - Test with small time ranges first

---

## Core Function Library

### Function 1: GetPlatformEvents

**Purpose:** Unified event stream from all monitoring sources

**Parameters:**
- `startTime` (datetime): Start of time range
- `endTime` (datetime): End of time range
- `sources` (dynamic array): Filter by source - ["Activity", "Capacity", "Gateway"] or [] for all

```kql
.create-or-alter function with (
    folder = "Platform",
    docstring = "Unified platform events from Activity, Capacity, and Gateway databases"
)
GetPlatformEvents(
    startTime:datetime,
    endTime:datetime,
    sources:dynamic = dynamic([])
) {
    let activityData =
        database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | extend
            EventTime = CreationTime,
            Source = "Activity",
            EventType = Activity,
            EventId = Id,
            WorkspaceId = toguid(details.WorkspaceId),
            CapacityId = toguid(details.CapacityId)
        | project EventTime, Source, EventType, EventId, UserId, WorkspaceId, CapacityId, Details=details;

    let capacityData =
        database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | extend
            EventTime = windowStartTime,
            Source = "Capacity",
            EventType = "CapacitySummary",
            EventId = id,
            UserId = "",
            WorkspaceId = guid(null),
            CapacityId = capacityId
        | project EventTime, Source, EventType, EventId, UserId, WorkspaceId, CapacityId, Details=pack_all();

    let gatewayData =
        database("Gateway Monitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | extend
            EventTime = jobStartTime,
            Source = "Gateway",
            EventType = "GatewayJob",
            EventId = guid(null),
            UserId = "",
            WorkspaceId = guid(null),
            CapacityId = guid(null)
        | project EventTime, Source, EventType, EventId, UserId, WorkspaceId, CapacityId, Details=pack_all();

    let allData = activityData | union capacityData, gatewayData;

    allData
    | where array_length(sources) == 0 or Source in (sources)
}
```

**Usage:**

```kql
// Get all events from last 24 hours
GetPlatformEvents(ago(24h), now(), dynamic([]))
| summarize Count=count() by Source

// Get only Activity and Capacity events
GetPlatformEvents(ago(1h), now(), dynamic(["Activity", "Capacity"]))
| summarize Count=count() by bin(EventTime, 5m), Source
| render timechart
```

---

### Function 2: GetPlatformMetrics

**Purpose:** Time-series metrics aggregated across all sources

**Parameters:**
- `startTime` (datetime): Start of time range
- `endTime` (datetime): End of time range
- `granularity` (timespan): Time bin size (1m, 5m, 1h, 1d)

```kql
.create-or-alter function with (
    folder = "Platform",
    docstring = "Platform metrics across Activity, Capacity, and Gateway databases"
)
GetPlatformMetrics(
    startTime:datetime,
    endTime:datetime,
    granularity:timespan = 5m
) {
    let activityMetrics = database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | summarize
            ActivityCount = count(),
            UniqueUsers = dcount(UserId),
            UniqueWorkspaces = dcount(toguid(details.WorkspaceId))
          by Timestamp = bin(CreationTime, granularity);

    let capacityMetrics = database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | summarize
            CapacityEvents = count(),
            AvgUtilization = avg(utilizationInteractive + utilizationBackground),
            MaxUtilization = max(utilizationInteractive + utilizationBackground),
            AvgOverage = avg(overageTotalCapacityUnitMs)
          by Timestamp = bin(windowStartTime, granularity);

    let gatewayMetrics = database("Gateway Monitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | summarize
            GatewayJobs = count(),
            FailedJobs = countif(status == "Failed"),
            AvgJobDuration = avg(duration)
          by Timestamp = bin(jobStartTime, granularity);

    activityMetrics
    | join kind=fullouter capacityMetrics on Timestamp
    | join kind=fullouter gatewayMetrics on Timestamp
    | project
        Timestamp = coalesce(Timestamp, Timestamp1, Timestamp2),
        ActivityCount = coalesce(ActivityCount, 0L),
        UniqueUsers = coalesce(UniqueUsers, 0L),
        UniqueWorkspaces = coalesce(UniqueWorkspaces, 0L),
        CapacityEvents = coalesce(CapacityEvents, 0L),
        AvgUtilization = coalesce(AvgUtilization, 0.0),
        MaxUtilization = coalesce(MaxUtilization, 0.0),
        AvgOverage = coalesce(AvgOverage, 0.0),
        GatewayJobs = coalesce(GatewayJobs, 0L),
        FailedJobs = coalesce(FailedJobs, 0L),
        AvgJobDuration = coalesce(AvgJobDuration, 0.0)
    | order by Timestamp asc
}
```

**Usage:**

```kql
// Get 5-minute metrics for last 24 hours
GetPlatformMetrics(ago(24h), now(), 5m)
| render timechart

// Get hourly metrics for last 7 days
GetPlatformMetrics(ago(7d), now(), 1h)
| project Timestamp, ActivityCount, AvgUtilization, GatewayJobs
| render timechart
```

---

### Function 3: GetWorkspaceSummary

**Purpose:** Per-workspace summary across all data sources

**Parameters:**
- `startTime` (datetime): Start of time range
- `endTime` (datetime): End of time range
- `workspaceId` (guid): Specific workspace or null for all

```kql
.create-or-alter function with (
    folder = "Platform",
    docstring = "Workspace-level summary combining Activity and Inventory data"
)
GetWorkspaceSummary(
    startTime:datetime,
    endTime:datetime,
    workspaceId:guid = guid(null)
) {
    let workspaceFilter = iff(workspaceId == guid(null), true, false);

    let activityByWorkspace = database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | extend WsId = toguid(details.WorkspaceId)
        | where workspaceFilter or WsId == workspaceId
        | summarize
            Activities = count(),
            UniqueUsers = dcount(UserId),
            TopActivities = make_set(Activity, 10)
          by WorkspaceId = WsId;

    let workspaceInfo = database("Platform Inventory").Workspaces
        | where workspaceFilter or id == workspaceId
        | project WorkspaceId=id, WorkspaceName=name, CapacityId=capacityId, State=state, Type=type;

    activityByWorkspace
    | join kind=rightouter workspaceInfo on WorkspaceId
    | project
        WorkspaceId,
        WorkspaceName,
        CapacityId,
        State,
        Type,
        Activities = coalesce(Activities, 0L),
        UniqueUsers = coalesce(UniqueUsers, 0L),
        TopActivities = coalesce(TopActivities, dynamic([]))
    | order by Activities desc
}
```

**Usage:**

```kql
// Get summary for all workspaces in last 7 days
GetWorkspaceSummary(ago(7d), now(), guid(null))
| where Activities > 0
| take 20

// Get summary for specific workspace
GetWorkspaceSummary(ago(7d), now(), toguid("00000000-0000-0000-0000-000000000000"))
```

---

### Function 4: GetCapacityHealth

**Purpose:** Capacity health metrics with workspace and activity context

**Parameters:**
- `startTime` (datetime): Start of time range
- `endTime` (datetime): End of time range

```kql
.create-or-alter function with (
    folder = "Platform",
    docstring = "Capacity health metrics from Capacity and Inventory databases"
)
GetCapacityHealth(
    startTime:datetime,
    endTime:datetime
) {
    let capacityMetrics = database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | summarize
            AvgUtilization = avg(utilizationInteractive + utilizationBackground),
            MaxUtilization = max(utilizationInteractive + utilizationBackground),
            P95Utilization = percentile(utilizationInteractive + utilizationBackground, 95),
            TotalOverage = sum(overageTotalCapacityUnitMs),
            ThrottleEvents = countif(utilizationInteractive + utilizationBackground > 100.0),
            DataPoints = count()
          by CapacityId = capacityId, CapacityName = capacityName;

    let capacityInfo = database("Platform Inventory").Capacities
        | project CapacityId, Sku, Region, State;

    capacityMetrics
    | join kind=leftouter capacityInfo on CapacityId
    | extend HealthScore = round(
        100.0 - (AvgUtilization * 0.5) - (ThrottleEvents * 5.0),
        2
      )
    | project
        CapacityId,
        CapacityName,
        Sku,
        Region,
        State,
        AvgUtilization,
        MaxUtilization,
        P95Utilization,
        TotalOverage,
        ThrottleEvents,
        DataPoints,
        HealthScore
    | order by HealthScore asc
}
```

**Usage:**

```kql
// Get capacity health for last 24 hours
GetCapacityHealth(ago(24h), now())
| where HealthScore < 80
| project CapacityName, AvgUtilization, ThrottleEvents, HealthScore
```

---

### Function 5: CorrelateCapacityAndActivity

**Purpose:** Correlate capacity utilization spikes with user activities

**Parameters:**
- `startTime` (datetime): Start of time range
- `endTime` (datetime): End of time range
- `utilizationThreshold` (real): Minimum utilization % to correlate (default 80)
- `correlationWindow` (timespan): Time window for correlation (default 5m)

```kql
.create-or-alter function with (
    folder = "Platform",
    docstring = "Correlate capacity utilization events with user activities"
)
CorrelateCapacityAndActivity(
    startTime:datetime,
    endTime:datetime,
    utilizationThreshold:real = 80.0,
    correlationWindow:timespan = 5m
) {
    let highUtilization = database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | where utilizationInteractive + utilizationBackground > utilizationThreshold
        | project
            CapacityEventTime = windowStartTime,
            CapacityId = capacityId,
            CapacityName = capacityName,
            Utilization = utilizationInteractive + utilizationBackground;

    let activities = database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | extend CapacityId = toguid(details.CapacityId)
        | where isnotempty(CapacityId)
        | project
            ActivityTime = CreationTime,
            CapacityId,
            Activity,
            UserId,
            WorkspaceId = toguid(details.WorkspaceId);

    highUtilization
    | join kind=inner activities on CapacityId
    | where abs(datetime_diff('second', CapacityEventTime, ActivityTime)) <= totalseconds(correlationWindow)
    | summarize
        ActivityCount = count(),
        Activities = make_bag(pack(Activity, count())),
        UniqueUsers = dcount(UserId),
        TopUsers = make_set(UserId, 10),
        UniqueWorkspaces = dcount(WorkspaceId)
      by
        CapacityEventTime,
        CapacityId,
        CapacityName,
        Utilization
    | extend TimeDeltaSeconds = 0L  // Summarized, so delta is 0
    | project
        CapacityEventTime,
        CapacityId,
        CapacityName,
        Utilization,
        ActivityCount,
        Activities,
        UniqueUsers,
        UniqueWorkspaces,
        TopUsers
    | order by CapacityEventTime desc, Utilization desc
}
```

**Usage:**

```kql
// Find correlations for utilization > 90% in last 24 hours
CorrelateCapacityAndActivity(ago(24h), now(), 90.0, 5m)
| take 20

// Find correlations for utilization > 80% with 10-minute window
CorrelateCapacityAndActivity(ago(7d), now(), 80.0, 10m)
| project CapacityEventTime, CapacityName, Utilization, ActivityCount, UniqueUsers
```

---

### Function 6: GetPlatformOverview

**Purpose:** High-level platform overview with key metrics from all sources

**Parameters:**
- `startTime` (datetime): Start of time range
- `endTime` (datetime): End of time range

```kql
.create-or-alter function with (
    folder = "Platform",
    docstring = "High-level platform overview combining all monitoring sources"
)
GetPlatformOverview(
    startTime:datetime,
    endTime:datetime
) {
    let activitySummary = database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | summarize
            TotalActivities = count(),
            UniqueUsers = dcount(UserId),
            UniqueWorkspaces = dcount(toguid(details.WorkspaceId)),
            TopActivities = make_set(Activity, 5);

    let capacitySummary = database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | summarize
            UniqueCapacities = dcount(capacityId),
            AvgUtilization = avg(utilizationInteractive + utilizationBackground),
            MaxUtilization = max(utilizationInteractive + utilizationBackground),
            TotalOverage = sum(overageTotalCapacityUnitMs),
            ThrottleEvents = countif(utilizationInteractive + utilizationBackground > 100.0);

    let gatewaySummary = database("Gateway Monitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | summarize
            TotalJobs = count(),
            FailedJobs = countif(status == "Failed"),
            AvgJobDuration = avg(duration),
            UniqueGateways = dcount(gatewayId);

    print
        TimeRange = strcat(format_datetime(startTime, "yyyy-MM-dd HH:mm"), " to ", format_datetime(endTime, "yyyy-MM-dd HH:mm")),
        TotalActivities = toscalar(activitySummary | project TotalActivities),
        UniqueUsers = toscalar(activitySummary | project UniqueUsers),
        UniqueWorkspaces = toscalar(activitySummary | project UniqueWorkspaces),
        TopActivities = toscalar(activitySummary | project TopActivities),
        UniqueCapacities = toscalar(capacitySummary | project UniqueCapacities),
        AvgCapacityUtilization = round(toscalar(capacitySummary | project AvgUtilization), 2),
        MaxCapacityUtilization = round(toscalar(capacitySummary | project MaxUtilization), 2),
        CapacityThrottleEvents = toscalar(capacitySummary | project ThrottleEvents),
        TotalGatewayJobs = toscalar(gatewaySummary | project TotalJobs),
        FailedGatewayJobs = toscalar(gatewaySummary | project FailedJobs),
        GatewayFailureRate = round(
            toscalar(gatewaySummary | project FailedJobs) * 100.0 /
            toscalar(gatewaySummary | project TotalJobs),
            2
        ),
        UniqueGateways = toscalar(gatewaySummary | project UniqueGateways)
}
```

**Usage:**

```kql
// Get overview for last 24 hours
GetPlatformOverview(ago(24h), now())

// Get overview for last 7 days
GetPlatformOverview(ago(7d), now())
```

---

### Function 7: GetActivityHotspots

**Purpose:** Identify workspaces with high activity and their capacity impact

**Parameters:**
- `startTime` (datetime): Start of time range
- `endTime` (datetime): End of time range
- `topN` (int): Number of top workspaces to return (default 10)

```kql
.create-or-alter function with (
    folder = "Platform",
    docstring = "Identify activity hotspots and their capacity impact"
)
GetActivityHotspots(
    startTime:datetime,
    endTime:datetime,
    topN:int = 10
) {
    let workspaceActivity = database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | extend WorkspaceId = toguid(details.WorkspaceId)
        | where isnotempty(WorkspaceId)
        | summarize
            Activities = count(),
            UniqueUsers = dcount(UserId),
            ActivityTypes = dcount(Activity)
          by WorkspaceId;

    let workspaceInfo = database("Platform Inventory").Workspaces
        | project WorkspaceId=id, WorkspaceName=name, CapacityId=capacityId;

    let capacityUtilization = database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | summarize
            AvgUtilization = avg(utilizationInteractive + utilizationBackground),
            MaxUtilization = max(utilizationInteractive + utilizationBackground)
          by CapacityId = capacityId;

    workspaceActivity
    | top topN by Activities desc
    | join kind=inner workspaceInfo on WorkspaceId
    | join kind=leftouter capacityUtilization on CapacityId
    | project
        WorkspaceId,
        WorkspaceName,
        CapacityId,
        Activities,
        UniqueUsers,
        ActivityTypes,
        AvgCapacityUtilization = round(coalesce(AvgUtilization, 0.0), 2),
        MaxCapacityUtilization = round(coalesce(MaxUtilization, 0.0), 2)
    | order by Activities desc
}
```

**Usage:**

```kql
// Get top 10 activity hotspots in last 24 hours
GetActivityHotspots(ago(24h), now(), 10)

// Get top 20 activity hotspots in last 7 days
GetActivityHotspots(ago(7d), now(), 20)
| where AvgCapacityUtilization > 70
```

---

## Dashboard Integration Examples

### Example 1: Multi-Source Activity Timeline

```kql
// Dashboard parameters: _startTime, _endTime, _sources (array)

GetPlatformEvents(_startTime, _endTime, _sources)
| summarize Count=count() by bin(EventTime, 5m), Source, EventType
| render timechart with (title="Platform Activity Timeline")
```

### Example 2: Platform Health Dashboard

```kql
// Dashboard parameters: _startTime, _endTime

let metrics = GetPlatformMetrics(_startTime, _endTime, 5m);
let overview = GetPlatformOverview(_startTime, _endTime);

metrics
| project
    Timestamp,
    ActivityCount,
    UniqueUsers,
    AvgUtilization,
    GatewayJobs
| render timechart with (title="Platform Metrics")
```

### Example 3: Capacity Monitoring with Activity Context

```kql
// Dashboard parameters: _startTime, _endTime

let capacityHealth = GetCapacityHealth(_startTime, _endTime);
let correlations = CorrelateCapacityAndActivity(_startTime, _endTime, 80.0, 5m);

capacityHealth
| join kind=leftouter (
    correlations
    | summarize TotalActivityEvents=sum(ActivityCount) by CapacityId
) on CapacityId
| project
    CapacityName,
    AvgUtilization,
    ThrottleEvents,
    HealthScore,
    TotalActivityEvents = coalesce(TotalActivityEvents, 0L)
| order by HealthScore asc
```

---

## Testing Queries

Run these queries to validate function deployment:

```kql
// Test 1: GetPlatformEvents
GetPlatformEvents(ago(1h), now(), dynamic([]))
| take 10

// Test 2: GetPlatformMetrics
GetPlatformMetrics(ago(24h), now(), 1h)
| take 10

// Test 3: GetWorkspaceSummary
GetWorkspaceSummary(ago(7d), now(), guid(null))
| take 10

// Test 4: GetCapacityHealth
GetCapacityHealth(ago(24h), now())
| take 10

// Test 5: CorrelateCapacityAndActivity
CorrelateCapacityAndActivity(ago(24h), now(), 80.0, 5m)
| take 10

// Test 6: GetPlatformOverview
GetPlatformOverview(ago(24h), now())

// Test 7: GetActivityHotspots
GetActivityHotspots(ago(7d), now(), 10)
```

---

## Performance Optimization Recommendations

After deploying these functions, consider:

1. **Create Materialized Views** for frequently accessed aggregations:
   ```kql
   // In Activity Events database
   .create-or-alter materialized-view ActivitySummaryByHour on table ActivityEvents {
       ActivityEvents
       | summarize Count=count(), UniqueUsers=dcount(UserId)
         by Hour=bin(CreationTime, 1h), Activity
   }
   ```

2. **Configure Hot Cache** for each database:
   ```kql
   .alter table ActivityEvents policy caching hot = 7d
   .alter table CapacitySummary policy caching hot = 7d
   .alter table GatewayJobs policy caching hot = 7d
   ```

3. **Monitor Function Performance**:
   ```kql
   .show queries
   | where Text contains "GetPlatformEvents"
   | summarize AvgDuration=avg(Duration), Count=count() by User
   ```

---

## Maintenance

**Monthly Tasks:**
- Review function usage with `.show functions`
- Check query performance with `.show queries`
- Update functions based on schema changes
- Add new functions for emerging patterns

**As Needed:**
- Adjust hot cache policies based on query patterns
- Add materialized views for slow aggregations
- Optimize time ranges in functions

---

**Deployment Checklist:**

- [ ] Core database exists
- [ ] All 7 functions deployed
- [ ] Functions tested with sample queries
- [ ] Hot cache configured (7 days minimum)
- [ ] Dashboards updated to use functions
- [ ] Performance baseline established
- [ ] Documentation updated

---

**Version:** 1.0
**Last Updated:** 2026-02-07
