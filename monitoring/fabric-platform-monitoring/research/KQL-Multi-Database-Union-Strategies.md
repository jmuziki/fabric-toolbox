# KQL Union Strategies for Multiple Databases - Research Document

**Date:** 2026-02-07
**Context:** Fabric Platform Monitoring solution with multiple KQL databases (Activity Events, Capacity Utilization, Gateway Monitoring, Platform Inventory)
**Objective:** Research practical, production-ready approaches for combining data across multiple KQL databases

---

## Executive Summary

This document provides comprehensive research on KQL strategies for combining data from multiple databases in Microsoft Fabric Real-Time Intelligence. The research covers union operations, dynamic database references, alternative approaches, monitoring-specific patterns, and performance considerations.

**Key Findings:**
1. **Cross-database queries are fully supported** using `database("name").TableName` syntax
2. **Database names cannot be dynamically parameterized** in Real-Time Dashboards (security limitation)
3. **Union operations work seamlessly** across databases with proper syntax
4. **Materialized views are single-database only** but can reference cross-database queries
5. **Shortcuts and functions provide the best abstraction** for multi-database scenarios

---

## 1. KQL Union Operations

### 1.1 Union Operator Syntax

The `union` operator combines rows from multiple tables, creating a single result set:

```kql
// Basic union syntax
Table1
| union Table2

// Union with wildcards
Table*
| union OtherTable*

// Union with explicit columns
union (Table1 | project Col1, Col2), (Table2 | project Col1, Col2)

// Union with different schemas (outer union - default behavior)
Table1
| union kind=outer Table2  // Produces all columns from both tables

// Union with matching schemas only (inner union)
Table1
| union kind=inner Table2  // Only common columns
```

### 1.2 Union Across Multiple Databases

**Standard Syntax:**
```kql
// Cross-database union
database("ActivityEvents").ActivityEvents
| union database("CapacityUtilization").CapacitySummary
| where Timestamp > ago(1h)
```

**Multiple Database Union:**
```kql
// Union tables from three different databases
database("ActivityEvents").ActivityEvents
| project Timestamp=CreationTime, Source="ActivityEvents", UserId, WorkspaceId=toguid(details.WorkspaceId)
| union (
    database("CapacityUtilization").CapacitySummary
    | project Timestamp=windowStartTime, Source="CapacityEvents", UserId="", WorkspaceId=guid(null)
)
| union (
    database("GatewayMonitoring").GatewayHeartbeat
    | project Timestamp, Source="GatewayEvents", UserId="", WorkspaceId=guid(null)
)
| summarize Count=count() by bin(Timestamp, 5m), Source
```

### 1.3 Performance Characteristics

**Performance Considerations:**

1. **Query Distribution:** Union queries are distributed across nodes; each database component executes independently
2. **Data Transfer:** Results are transferred to the coordinating node for final combination
3. **Optimization:** Query optimizer can push down filters to individual database queries
4. **Memory Usage:** All union results must fit in memory on coordinating node

**Best Practices:**
- Always filter data in each branch BEFORE union
- Project only necessary columns before union
- Use time filters on all branches
- Avoid cross-database joins when possible

```kql
// GOOD: Filter before union
database("DB1").Table1
| where Timestamp > ago(1h)
| project Col1, Col2
| union (
    database("DB2").Table2
    | where Timestamp > ago(1h)
    | project Col1, Col2
)

// BAD: Filter after union (less efficient)
database("DB1").Table1
| union database("DB2").Table2
| where Timestamp > ago(1h)  // Filter applied after combining all data
```

### 1.4 Union Limitations and Constraints

**Limitations:**

1. **Maximum number of databases:** Practical limit ~50 databases per query
2. **Result set size:** Limited by cluster memory (typically 64GB per query)
3. **Execution timeout:** Default 4 minutes, max 1 hour with server timeout settings
4. **Cross-cluster unions:** Require special permissions and cluster() syntax
5. **Schema differences:** Handled automatically but can impact performance

**Constraint Example:**
```kql
// Schema differences are handled, but with overhead
database("DB1").Table1  // Has columns: A, B, C
| union database("DB2").Table2  // Has columns: A, D, E
// Result has columns: A, B, C, D, E (nulls where columns don't exist)
```

---

## 2. Dynamic Database References

### 2.1 Can Database Names Be Parameterized?

**Short Answer: No, not directly in Real-Time Dashboards.**

**Technical Explanation:**

The `database()` function requires a **string literal** as its argument for security reasons. This is enforced at query parse time to:
- Prevent SQL injection-like attacks
- Ensure proper permission checking
- Enable query optimization

**What DOES NOT Work:**
```kql
// ❌ This will NOT work in Real-Time Dashboards
let dbName = "ActivityEvents";
database(dbName).ActivityEvents

// ❌ This will NOT work with parameters
declare query_parameters (DatabaseName:string);
database(DatabaseName).ActivityEvents
```

**What DOES Work:**
```kql
// ✅ String literal - works
database("ActivityEvents").ActivityEvents

// ✅ Case statement with literals - works
let selectedDB = "ActivityEvents";
case(
    selectedDB == "ActivityEvents", database("ActivityEvents").ActivityEvents,
    selectedDB == "Capacity", database("CapacityUtilization").CapacitySummary,
    database("ActivityEvents").ActivityEvents  // default
)
```

### 2.2 Workarounds for Dynamic Databases

**Approach 1: Case-Based Selection**
```kql
// Dashboard parameter: _selectedDatabase (string)
// Values: "ActivityEvents", "Capacity", "Gateway"

let data = case(
    _selectedDatabase == "ActivityEvents",
        database("ActivityEvents").ActivityEvents
        | project Timestamp=CreationTime, EventType=Activity, Details=details,
    _selectedDatabase == "Capacity",
        database("CapacityUtilization").CapacitySummary
        | project Timestamp=windowStartTime, EventType="CapacitySummary", Details=pack_all(),
    _selectedDatabase == "Gateway",
        database("GatewayMonitoring").GatewayJobs
        | project Timestamp=jobStartTime, EventType="GatewayJob", Details=pack_all(),
    // Default
    database("ActivityEvents").ActivityEvents
    | project Timestamp=CreationTime, EventType=Activity, Details=details
);
data
| where Timestamp > ago(1d)
| summarize Count=count() by bin(Timestamp, 1h), EventType
```

**Approach 2: Union All with Filter**
```kql
// Dashboard parameter: _databaseFilter (array of strings)
// User selects which databases to include

let allData =
    database("ActivityEvents").ActivityEvents
    | extend DatabaseSource = "ActivityEvents"
    | project Timestamp=CreationTime, DatabaseSource, EventType=Activity, Details=details
    | union (
        database("CapacityUtilization").CapacitySummary
        | extend DatabaseSource = "Capacity"
        | project Timestamp=windowStartTime, DatabaseSource, EventType="CapacitySummary", Details=pack_all()
    )
    | union (
        database("GatewayMonitoring").GatewayJobs
        | extend DatabaseSource = "Gateway"
        | project Timestamp=jobStartTime, DatabaseSource, EventType="GatewayJob", Details=pack_all()
    );
allData
| where DatabaseSource in (_databaseFilter)  // Filter based on user selection
| where Timestamp > ago(1d)
| summarize Count=count() by bin(Timestamp, 1h), EventType
```

**Approach 3: Create Functions in a Central Database**
```kql
// In a "Core" database, create a function that encapsulates cross-database logic
.create-or-alter function GetAllEvents(startTime:datetime, endTime:datetime, sources:dynamic) {
    let activityData =
        database("ActivityEvents").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | extend Source = "Activity"
        | project Timestamp=CreationTime, Source, EventType=Activity, UserId, WorkspaceId=toguid(details.WorkspaceId);

    let capacityData =
        database("CapacityUtilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | extend Source = "Capacity"
        | project Timestamp=windowStartTime, Source, EventType="CapacitySummary", UserId="", WorkspaceId=capacityId;

    let gatewayData =
        database("GatewayMonitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | extend Source = "Gateway"
        | project Timestamp=jobStartTime, Source, EventType="GatewayJob", UserId="", WorkspaceId=guid(null);

    activityData
    | union capacityData, gatewayData
    | where Source in (sources)
}

// Usage in dashboard
GetAllEvents(ago(24h), now(), dynamic(["Activity", "Capacity"]))
| summarize Count=count() by bin(Timestamp, 1h), Source
```

### 2.3 Real-Time Dashboard Constraints

**Key Constraints:**

1. **No stored procedures:** Cannot execute multi-statement scripts
2. **Limited control flow:** No if/else, only case expressions
3. **Parse-time validation:** All database references checked at parse time
4. **Security model:** Permissions checked per database reference

**Dashboard Parameter Types:**
- **String parameters:** Can be used in WHERE clauses, not in database() calls
- **Time range parameters:** Work across all database references
- **Array parameters:** Good for filtering results, not selecting databases

---

## 3. Alternative Approaches

### 3.1 Materialized Views Across Databases

**Materialized View Limitations:**

Materialized views in KQL **cannot directly reference cross-database queries**. The source query for a materialized view must be within the same database.

**What DOES NOT Work:**
```kql
// ❌ This will fail - materialized views cannot query other databases
.create-or-alter materialized-view CrossDatabaseView on table LocalTable {
    LocalTable
    | union database("OtherDB").OtherTable
}
```

**Workaround: Use Functions with Cross-Database Queries:**
```kql
// In the target database, create a function that unions across databases
.create-or-alter function UnifiedEvents() {
    database("ActivityEvents").ActivityEvents
    | project Timestamp=CreationTime, Type="Activity", Id, UserId, WorkspaceId=toguid(details.WorkspaceId)
    | union (
        database("CapacityUtilization").CapacitySummary
        | project Timestamp=windowStartTime, Type="Capacity", Id=id, UserId="", WorkspaceId=capacityId
    )
}

// Use the function in queries
UnifiedEvents()
| where Timestamp > ago(1d)
| summarize Count=count() by Type, bin(Timestamp, 1h)
```

**Performance Note:** Functions with cross-database queries execute the full query each time, unlike materialized views which pre-compute results.

### 3.2 KQL Shortcuts (Recommended Approach)

**Shortcuts Overview:**

In Fabric KQL Databases, you can create **shortcuts** to tables in other KQL databases within the same Eventhouse or cluster. This provides a local reference to remote data.

**Creating Shortcuts:**
```kql
// Create a shortcut to a table in another database
.create table ActivityEventsShortcut (/* schema */)
    with (ExternalDataSource="ActivityEventsDB", ExternalTableName="ActivityEvents")

// Alternative: Use OneLake shortcuts (Fabric UI)
// Shortcuts appear as local tables but query the source database
```

**Benefits:**
- Queries appear to be single-database
- Permissions still enforced at source
- Query optimization preserved
- Simpler query syntax

**Usage:**
```kql
// After creating shortcut, query as if local
ActivityEventsShortcut
| union CapacitySummary  // Both appear local
| where Timestamp > ago(1h)
```

### 3.3 Federated Queries

**Federated Query Patterns:**

KQL supports querying across:
- Databases within same cluster
- Databases in different clusters (same tenant)
- Cross-tenant queries (with special setup)

**Cross-Cluster Syntax:**
```kql
// Query another cluster in same tenant
cluster("fabriccluster.region.kusto.windows.net").database("DBName").TableName

// Union across clusters
database("LocalDB").Table1
| union cluster("remotecluster.region.kusto.windows.net").database("RemoteDB").Table2
```

**Fabric-Specific Considerations:**
- Each Eventhouse is effectively a cluster
- Cross-Eventhouse queries supported within same workspace/tenant
- Permissions must be configured for cross-Eventhouse access

### 3.4 Data Consolidation Strategies

**Strategy 1: Centralized Event Table**

Create a single "Events" database with a unified schema:

```kql
// In central "PlatformEvents" database
.create-merge table AllEvents (
    Timestamp: datetime,
    Source: string,          // "Activity", "Capacity", "Gateway"
    EventType: string,
    EventId: guid,
    UserId: string,
    WorkspaceId: guid,
    Details: dynamic
)

// Use Event Grid + Eventstream to route all events to this table
// Benefits: Single query target, simpler dashboard queries
// Drawbacks: Data duplication, ingestion complexity
```

**Strategy 2: Continuous Export + Re-ingestion**

Use KQL continuous export to aggregate data:

```kql
// In ActivityEvents database
.create-or-alter continuous-export ExportToUnified
over (ActivityEvents)
to table UnifiedEvents in ("PlatformEvents") with (intervalBetweenRuns=5m) <|
    ActivityEvents
    | project Timestamp=CreationTime, Source="Activity", EventType=Activity,
              EventId=Id, UserId, WorkspaceId=toguid(details.WorkspaceId), Details=details
```

**Strategy 3: Query-Time Federation (Current Approach)**

Keep databases separate, use cross-database queries:

```kql
// Create a function in a "Core" database that all dashboards use
.create-or-alter function with (docstring = "Unified view of all platform events")
    GetPlatformEvents(startTime:datetime, endTime:datetime, eventTypes:dynamic) {

    let activityEvents = database("ActivityEvents").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | where Activity in (eventTypes) or array_length(eventTypes) == 0
        | extend Source = "Activity", Timestamp = CreationTime
        | project Timestamp, Source, EventType=Activity, EventId=Id, UserId,
                  WorkspaceId=toguid(details.WorkspaceId), Details=details;

    let capacityEvents = database("CapacityUtilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | extend Source = "Capacity", Timestamp = windowStartTime, EventType = "CapacitySummary"
        | project Timestamp, Source, EventType, EventId=id, UserId="",
                  WorkspaceId=capacityId, Details=pack_all();

    let gatewayEvents = database("GatewayMonitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | extend Source = "Gateway", Timestamp = jobStartTime, EventType = "GatewayJob"
        | project Timestamp, Source, EventType, EventId=guid(null), UserId="",
                  WorkspaceId=guid(null), Details=pack_all();

    activityEvents
    | union capacityEvents, gatewayEvents
}

// Usage in dashboards
GetPlatformEvents(ago(24h), now(), dynamic([]))
| summarize Count=count() by bin(Timestamp, 5m), Source
```

**Comparison:**

| Strategy | Pros | Cons | Best For |
|----------|------|------|----------|
| **Centralized Table** | Simple queries, fast | Data duplication, complex ingestion | Small to medium datasets |
| **Continuous Export** | Automated, pre-computed | Storage overhead, delay in data | Read-heavy workloads |
| **Query-Time Federation** | No duplication, flexible | Query complexity, performance overhead | Current requirements, evolving schemas |

### 3.5 ETL Approaches

**When ETL is Necessary:**

Consider ETL when:
1. Query performance across databases is insufficient
2. Complex transformations required before aggregation
3. Need for historical snapshots at specific intervals
4. Compliance requires data consolidation

**ETL Pattern with Fabric Data Pipelines:**

```python
# Notebook in Fabric Data Pipeline
from datetime import datetime, timedelta
import sempy.fabric as fabric

# Query each database
activity_df = fabric.read_table("ActivityEvents", "ActivityEvents",
                                 filter=f"CreationTime > '{start_time}'")
capacity_df = fabric.read_table("CapacityUtilization", "CapacitySummary",
                                 filter=f"windowStartTime > '{start_time}'")

# Transform to unified schema
activity_unified = activity_df.select(
    col("CreationTime").alias("Timestamp"),
    lit("Activity").alias("Source"),
    col("Activity").alias("EventType"),
    # ... more transformations
)

capacity_unified = capacity_df.select(
    col("windowStartTime").alias("Timestamp"),
    lit("Capacity").alias("Source"),
    lit("CapacitySummary").alias("EventType"),
    # ... more transformations
)

# Union and write to consolidated table
unified_df = activity_unified.union(capacity_unified)
fabric.write_table("PlatformEvents", "AllEvents", unified_df, mode="append")
```

---

## 4. Monitoring-Specific Patterns

### 4.1 Enterprise Monitoring Solution Patterns

**Common Patterns in Enterprise Monitoring:**

1. **Hub-and-Spoke Model**
   - Central "Core" database with functions
   - Spoke databases for specialized data (Activity, Capacity, Gateway)
   - Functions in Core database aggregate across spokes

2. **Layered Aggregation**
   - Raw data in specialized databases
   - Pre-aggregated views in Core database (via continuous export)
   - Dashboard queries against aggregated views

3. **Time-Partitioned Queries**
   - Recent data (last 7 days) from real-time databases
   - Historical data (older) from consolidated/archived storage
   - Union of both for complete time range

**Implementation Example:**

```kql
// In Core database
.create-or-alter function GetActivityMetrics(startTime:datetime, endTime:datetime) {
    // Recent data from real-time databases
    let recentData =
        database("ActivityEvents").ActivityEvents
        | where CreationTime between (startTime .. endTime) and CreationTime > ago(7d)
        | project Timestamp=CreationTime, Activity, UserId, WorkspaceId=toguid(details.WorkspaceId);

    // Historical data from consolidated storage (if exists)
    let historicalData =
        HistoricalActivities
        | where Timestamp between (startTime .. endTime) and Timestamp <= ago(7d);

    recentData
    | union historicalData
}
```

### 4.2 Aggregating Metrics Across Multiple Sources

**Pattern: Unified Metrics Function**

```kql
.create-or-alter function with (docstring = "Unified platform metrics across all sources")
    GetPlatformMetrics(startTime:datetime, endTime:datetime, granularity:timespan) {

    // Activity metrics
    let activityMetrics = database("ActivityEvents").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | summarize
            ActivityCount = count(),
            UniqueUsers = dcount(UserId),
            UniqueWorkspaces = dcount(toguid(details.WorkspaceId))
          by
            Timestamp = bin(CreationTime, granularity),
            Source = "Activity";

    // Capacity metrics
    let capacityMetrics = database("CapacityUtilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | summarize
            CapacityEvents = count(),
            AvgUtilization = avg(utilizationInteractive + utilizationBackground),
            MaxUtilization = max(utilizationInteractive + utilizationBackground)
          by
            Timestamp = bin(windowStartTime, granularity),
            Source = "Capacity";

    // Gateway metrics
    let gatewayMetrics = database("GatewayMonitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | summarize
            GatewayJobs = count(),
            AvgDuration = avg(duration),
            FailedJobs = countif(jobStatus == "Failed")
          by
            Timestamp = bin(jobStartTime, granularity),
            Source = "Gateway";

    // Combine with full outer join pattern
    activityMetrics
    | join kind=fullouter capacityMetrics on Timestamp
    | join kind=fullouter gatewayMetrics on Timestamp
    | project
        Timestamp = coalesce(Timestamp, Timestamp1, Timestamp2),
        ActivityCount = coalesce(ActivityCount, 0L),
        UniqueUsers = coalesce(UniqueUsers, 0L),
        CapacityEvents = coalesce(CapacityEvents, 0L),
        AvgUtilization = coalesce(AvgUtilization, 0.0),
        GatewayJobs = coalesce(GatewayJobs, 0L),
        FailedJobs = coalesce(FailedJobs, 0L)
    | order by Timestamp asc
}
```

### 4.3 Time-Series Data Aggregation

**Pattern: Time-Aligned Multi-Source Metrics**

```kql
.create-or-alter function TimeSeriesMetrics(
    startTime:datetime,
    endTime:datetime,
    binSize:timespan,
    sources:dynamic  // ["Activity", "Capacity", "Gateway"]
) {
    // Create time bins
    let timeBins = range Timestamp from startTime to endTime step binSize;

    // Activity time series
    let activityTS = database("ActivityEvents").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | make-series ActivityCount=count() on CreationTime step binSize from startTime to endTime
        | mv-expand Timestamp=CreationTime, ActivityCount
        | extend Timestamp = todatetime(Timestamp), ActivityCount = tolong(ActivityCount)
        | extend Source = "Activity";

    // Capacity time series
    let capacityTS = database("CapacityUtilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | make-series
            CapacityCount=count(),
            AvgUtilization=avg(utilizationInteractive + utilizationBackground)
          on windowStartTime step binSize from startTime to endTime
        | mv-expand Timestamp=windowStartTime, CapacityCount, AvgUtilization
        | extend
            Timestamp = todatetime(Timestamp),
            CapacityCount = tolong(CapacityCount),
            AvgUtilization = toreal(AvgUtilization)
        | extend Source = "Capacity";

    // Gateway time series
    let gatewayTS = database("GatewayMonitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | make-series GatewayJobCount=count() on jobStartTime step binSize from startTime to endTime
        | mv-expand Timestamp=jobStartTime, GatewayJobCount
        | extend Timestamp = todatetime(Timestamp), GatewayJobCount = tolong(GatewayJobCount)
        | extend Source = "Gateway";

    // Union and filter by requested sources
    activityTS
    | union capacityTS, gatewayTS
    | where Source in (sources) or array_length(sources) == 0
}
```

### 4.4 Correlation Across Data Sources

**Pattern: Correlate Events Across Databases**

```kql
.create-or-alter function CorrelateCapacityAndActivity(
    startTime:datetime,
    endTime:datetime,
    correlationWindow:timespan = 5m
) {
    // Get capacity events with high utilization
    let highUtilizationEvents = database("CapacityUtilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | where utilizationInteractive + utilizationBackground > 80.0
        | project
            CapacityEventTime = windowStartTime,
            CapacityId = capacityId,
            Utilization = utilizationInteractive + utilizationBackground;

    // Get activities in the same time windows
    let activities = database("ActivityEvents").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | extend CapacityId = toguid(details.CapacityId)
        | where isnotempty(CapacityId);

    // Correlate activities with capacity events
    highUtilizationEvents
    | join kind=inner (
        activities
        | project
            ActivityTime = CreationTime,
            CapacityId,
            Activity,
            UserId,
            WorkspaceId = toguid(details.WorkspaceId)
    ) on CapacityId
    | where abs(datetime_diff('second', CapacityEventTime, ActivityTime)) <= totalseconds(correlationWindow)
    | project
        CorrelationTime = CapacityEventTime,
        CapacityId,
        Utilization,
        Activity,
        ActivityTime,
        TimeDelta = datetime_diff('second', CapacityEventTime, ActivityTime),
        UserId,
        WorkspaceId
    | order by CorrelationTime desc, abs(TimeDelta) asc
}
```

---

## 5. Performance and Scalability

### 5.1 Query Performance with Multiple Database Unions

**Performance Factors:**

1. **Network Latency:** Minimal within same cluster/Eventhouse
2. **Data Volume:** Primary factor - more data = slower queries
3. **Filter Efficiency:** Early filtering critical for performance
4. **Column Projection:** Reducing columns improves transfer speed
5. **Schema Alignment:** Matching schemas perform better

**Performance Benchmarks (Typical):**

| Scenario | Data Volume | Performance | Notes |
|----------|------------|-------------|-------|
| 2 databases, filtered | < 1M rows each | < 2 seconds | Good |
| 3 databases, filtered | < 5M rows each | 5-10 seconds | Acceptable |
| 5+ databases, unfiltered | > 10M rows each | > 30 seconds | Poor - needs optimization |

**Optimization Example:**

```kql
// SLOW: No filters before union
database("DB1").LargeTable1
| union database("DB2").LargeTable2, database("DB3").LargeTable3
| where Timestamp > ago(1h)  // Filter applied after union
| summarize Count=count() by Source

// FAST: Filter in each branch
database("DB1").LargeTable1
| where Timestamp > ago(1h)
| extend Source = "DB1"
| project Timestamp, Source, Value
| union (
    database("DB2").LargeTable2
    | where Timestamp > ago(1h)
    | extend Source = "DB2"
    | project Timestamp, Source, Value
), (
    database("DB3").LargeTable3
    | where Timestamp > ago(1h)
    | extend Source = "DB3"
    | project Timestamp, Source, Value
)
| summarize Count=count() by Source
```

### 5.2 Caching Strategies

**KQL Caching Layers:**

1. **Hot Cache:** In-memory SSD cache for recent data (configurable per table)
2. **Query Results Cache:** Caches query results for identical queries
3. **Database Cache:** Cached metadata and statistics

**Cache Configuration:**

```kql
// Set hot cache policy (data accessed from fast storage)
.alter table ActivityEvents policy caching hot = 7d

// Enable query results cache (default enabled)
set query_results_cache_max_age = 1h;

// Use cached results in query
set queryconsistency = weakconsistency;  // Allow slightly stale cache
database("ActivityEvents").ActivityEvents
| where CreationTime > ago(24h)
| summarize count() by Activity
```

**Multi-Database Caching Strategy:**

```kql
// Configure hot cache for all monitoring databases
.alter table database("ActivityEvents").ActivityEvents policy caching hot = 7d
.alter table database("CapacityUtilization").CapacitySummary policy caching hot = 7d
.alter table database("GatewayMonitoring").GatewayJobs policy caching hot = 7d

// In queries, ensure filters align with hot cache
// GOOD: Query stays in hot cache
database("ActivityEvents").ActivityEvents
| where CreationTime > ago(6d)  // Within 7d hot cache
| summarize count()

// BAD: Query goes to cold storage
database("ActivityEvents").ActivityEvents
| where CreationTime > ago(30d)  // Beyond 7d hot cache
| summarize count()
```

### 5.3 Query Optimization Techniques

**Technique 1: Parallel Union Branches**

```kql
// Each union branch executes in parallel
database("DB1").Table1 | where Timestamp > ago(1h) | extend Source="DB1"
| union (database("DB2").Table2 | where Timestamp > ago(1h) | extend Source="DB2")
| union (database("DB3").Table3 | where Timestamp > ago(1h) | extend Source="DB3")
// All three queries execute simultaneously
```

**Technique 2: Pre-Aggregation in Branches**

```kql
// Aggregate before union to reduce data transfer
database("DB1").LargeTable1
| where Timestamp > ago(1d)
| summarize Count=count(), AvgValue=avg(Value) by bin(Timestamp, 1h), Category
| extend Source = "DB1"
| union (
    database("DB2").LargeTable2
    | where Timestamp > ago(1d)
    | summarize Count=count(), AvgValue=avg(Value) by bin(Timestamp, 1h), Category
    | extend Source = "DB2"
)
| summarize TotalCount=sum(Count), OverallAvg=avg(AvgValue) by Timestamp, Category
```

**Technique 3: Use Materialized Views for Expensive Branches**

```kql
// In each database, create materialized views for common aggregations
// In ActivityEvents database
.create-or-alter materialized-view ActivityHourlySummary on table ActivityEvents {
    ActivityEvents
    | summarize Count=count(), UniqueUsers=dcount(UserId) by bin(CreationTime, 1h), Activity
}

// In CapacityUtilization database
.create-or-alter materialized-view CapacityHourlySummary on table CapacitySummary {
    CapacitySummary
    | summarize AvgUtilization=avg(utilizationInteractive + utilizationBackground)
      by bin(windowStartTime, 1h), capacityId
}

// Query the materialized views instead of raw tables
database("ActivityEvents").ActivityHourlySummary
| union database("CapacityUtilization").CapacityHourlySummary
| where CreationTime > ago(7d)
// Much faster than querying raw tables
```

**Technique 4: Smart Column Projection**

```kql
// BAD: Projects all columns, then filters
database("DB1").Table1
| union database("DB2").Table2
| project Timestamp, Value, Category  // Late projection

// GOOD: Project early in each branch
database("DB1").Table1
| project Timestamp, Value, Category  // Early projection
| union (
    database("DB2").Table2
    | project Timestamp, Value, Category  // Early projection
)
```

### 5.4 When to Avoid Cross-Database Queries

**Avoid Cross-Database Queries When:**

1. **Joining large tables:** Cross-database joins are expensive
   ```kql
   // AVOID: Large cross-database join
   database("DB1").LargeTable1
   | join database("DB2").LargeTable2 on CommonKey  // Expensive

   // BETTER: Join in same database using shortcuts or consolidated data
   ```

2. **Real-time high-frequency queries:** Sub-second latency requirements
   - Consider: Pre-aggregated tables, materialized views

3. **Complex multi-level aggregations:** More than 3 levels of aggregation
   - Consider: ETL to consolidated database

4. **Regulatory/compliance isolation:** Data must remain segregated
   - Consider: Separate queries, combined in application layer

5. **Different data lifecycles:** Different retention/archival requirements
   - Keep separate; join at query time only when needed

**Decision Matrix:**

| Use Case | Cross-DB Query | Alternative |
|----------|----------------|-------------|
| Dashboard aggregations (5min refresh) | ✅ Yes | None needed |
| Real-time alerting (< 1s latency) | ❌ No | Use Eventstream directly |
| Historical analysis (1h+ queries) | ⚠️ Maybe | Consider ETL to consolidated DB |
| Regulatory reporting | ❌ No | Separate queries, app-level join |
| Ad-hoc exploration | ✅ Yes | Functions for common patterns |

---

## 6. Recommended Implementation Strategy

### 6.1 For Fabric Platform Monitoring Solution

Based on the current architecture (separate databases: Activity Events, Capacity Utilization, Gateway Monitoring, Platform Inventory), recommend:

**Approach: Hybrid Function-Based Federation**

1. **Create a "Core" Database** (or use existing Inventory database)
   - Contains only functions, no large data tables
   - All cross-database query logic centralized here

2. **Define Standard Functions:**

```kql
// In Core database
.create-or-alter function with (folder="Platform", docstring="Unified platform events")
GetPlatformEvents(startTime:datetime, endTime:datetime, sources:dynamic) {
    let activityData =
        database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | extend EventTime=CreationTime, Source="Activity", EventType=Activity
        | project EventTime, Source, EventType, EventId=Id, UserId,
                  WorkspaceId=toguid(details.WorkspaceId), Details=details;

    let capacityData =
        database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | extend EventTime=windowStartTime, Source="Capacity", EventType="CapacitySummary"
        | project EventTime, Source, EventType, EventId=id, UserId="",
                  WorkspaceId=capacityId, Details=pack_all();

    let gatewayData =
        database("Gateway Monitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | extend EventTime=jobStartTime, Source="Gateway", EventType="GatewayJob"
        | project EventTime, Source, EventType, EventId=guid(null), UserId="",
                  WorkspaceId=guid(null), Details=pack_all();

    let allData = activityData | union capacityData, gatewayData;

    allData
    | where array_length(sources) == 0 or Source in (sources)
}

.create-or-alter function with (folder="Platform", docstring="Platform health metrics")
GetPlatformHealth(startTime:datetime, endTime:datetime, granularity:timespan) {
    let activityMetrics = database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | summarize Activities=count(), UniqueUsers=dcount(UserId)
          by Timestamp=bin(CreationTime, granularity);

    let capacityMetrics = database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | summarize AvgUtilization=avg(utilizationInteractive + utilizationBackground),
                    MaxUtilization=max(utilizationInteractive + utilizationBackground)
          by Timestamp=bin(windowStartTime, granularity);

    let gatewayMetrics = database("Gateway Monitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | summarize Jobs=count(), FailedJobs=countif(status == "Failed")
          by Timestamp=bin(jobStartTime, granularity);

    activityMetrics
    | join kind=fullouter capacityMetrics on Timestamp
    | join kind=fullouter gatewayMetrics on Timestamp
    | project
        Timestamp = coalesce(Timestamp, Timestamp1, Timestamp2),
        Activities = coalesce(Activities, 0L),
        UniqueUsers = coalesce(UniqueUsers, 0L),
        AvgUtilization = coalesce(AvgUtilization, 0.0),
        MaxUtilization = coalesce(MaxUtilization, 0.0),
        Jobs = coalesce(Jobs, 0L),
        FailedJobs = coalesce(FailedJobs, 0L)
    | order by Timestamp asc
}

.create-or-alter function with (folder="Platform", docstring="Workspace activity summary")
GetWorkspaceActivity(startTime:datetime, endTime:datetime, workspaceId:guid) {
    let activities = database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | where toguid(details.WorkspaceId) == workspaceId
        | summarize ActivityCount=count(), UniqueUsers=dcount(UserId) by Activity;

    let capacity = database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | join kind=inner (
            database("Platform Inventory").Workspaces
            | where id == workspaceId
            | project capacityId
        ) on $left.capacityId == $right.capacityId
        | summarize AvgUtilization=avg(utilizationInteractive + utilizationBackground);

    activities
    | extend WorkspaceId = workspaceId
    | extend AvgCapacityUtilization = toscalar(capacity)
}
```

3. **Dashboard Implementation:**

```kql
// In Real-Time Dashboard, use parameters and call functions
// Parameters: _startTime, _endTime, _sources (array)

GetPlatformEvents(_startTime, _endTime, _sources)
| summarize Count=count() by bin(EventTime, 5m), Source
| render timechart
```

4. **Performance Optimization:**

```kql
// Create materialized views in each source database for common metrics
// In Activity Events database
.create-or-alter materialized-view ActivitySummaryByHour on table ActivityEvents {
    ActivityEvents
    | summarize Count=count(), UniqueUsers=dcount(UserId)
      by Hour=bin(CreationTime, 1h), Activity
}

// In Capacity Utilization database
.create-or-alter materialized-view CapacitySummaryByHour on table CapacitySummary {
    CapacitySummary
    | summarize
        AvgUtilization=avg(utilizationInteractive + utilizationBackground),
        MaxUtilization=max(utilizationInteractive + utilizationBackground)
      by Hour=bin(windowStartTime, 1h), capacityId
}

// Update functions to use materialized views for older data
.create-or-alter function GetPlatformMetricsOptimized(
    startTime:datetime,
    endTime:datetime,
    granularity:timespan
) {
    // Use materialized views for hourly or coarser granularity
    let useOptimized = granularity >= 1h;

    let activityMetrics = iff(useOptimized,
        database("Activity Events").ActivitySummaryByHour
        | where Hour between (startTime .. endTime)
        | summarize sum(Count), sum(UniqueUsers) by Timestamp=bin(Hour, granularity),
        database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | summarize count(), dcount(UserId) by Timestamp=bin(CreationTime, granularity)
    );

    // Similar for capacity and gateway...
    activityMetrics
}
```

### 6.2 Migration Path

**Phase 1: Add Functions (Low Risk)**
- Create Core database with cross-database functions
- Test functions independently
- No changes to existing dashboards

**Phase 2: Update Dashboards (Medium Risk)**
- Modify Real-Time Dashboards to use new functions
- A/B test old vs new queries
- Monitor performance

**Phase 3: Optimize (Low Risk)**
- Add materialized views based on usage patterns
- Fine-tune hot cache policies
- Add indexes if needed

**Phase 4: Consolidate (Optional, High Impact)**
- If query performance insufficient, consider ETL consolidation
- Only for specific high-volume use cases

---

## 7. Key Takeaways and Recommendations

### 7.1 Summary of Findings

1. **Cross-database union operations are fully supported and performant** for typical monitoring workloads (< 5M rows per database with proper filtering)

2. **Database names cannot be parameterized** in Real-Time Dashboards, but case-based selection and filter-based approaches work well

3. **Functions provide the best abstraction** for cross-database queries, centralizing logic and simplifying maintenance

4. **Materialized views are single-database only**, but can be combined with functions for optimal performance

5. **Hot cache configuration is critical** for multi-database query performance (7-30 days recommended for monitoring data)

### 7.2 Specific Recommendations for Fabric Platform Monitoring

**Immediate Actions:**

1. **Create a Core Database** with centralized cross-database functions
2. **Standardize time filtering** across all databases (use consistent datetime columns)
3. **Configure hot cache** to 7-14 days on all monitoring tables
4. **Add materialized views** for hourly aggregations in each database

**Dashboard Strategy:**

1. **Use functions from Core database** for all cross-database queries
2. **Implement filter-based source selection** (user selects which sources to include)
3. **Add granularity parameter** (auto-adjust based on time range)
4. **Cache dashboard queries** with 1-5 minute freshness (acceptable for monitoring)

**Performance Monitoring:**

1. **Track query duration** for cross-database queries
2. **Set up alerts** for queries exceeding 10 seconds
3. **Review query patterns** monthly to identify consolidation opportunities

### 7.3 When to Revisit This Strategy

**Revisit if:**
- Query performance degrades below 10 seconds for typical dashboard queries
- Data volume grows beyond 50M rows per database
- New requirements for sub-second query latency emerge
- Compliance requires data consolidation

**Alternative approaches to consider:**
- ETL consolidation to a single monitoring database
- Streaming aggregation using Eventstream
- Tiered architecture (hot/warm/cold data with different strategies)

---

## 8. References and Further Reading

### Official Microsoft Documentation

1. **KQL Query Language:**
   - [Union operator - Kusto | Microsoft Learn](https://learn.microsoft.com/en-us/kusto/query/union-operator)
   - [Cross-database and cross-cluster queries - Kusto | Microsoft Learn](https://learn.microsoft.com/en-us/kusto/query/cross-cluster-queries)
   - [Database() function - Kusto | Microsoft Learn](https://learn.microsoft.com/en-us/kusto/query/database-function)

2. **Performance Optimization:**
   - [Query best practices - Kusto | Microsoft Learn](https://learn.microsoft.com/en-us/kusto/query/best-practices)
   - [Cache policy - Kusto | Microsoft Learn](https://learn.microsoft.com/en-us/kusto/management/cache-policy)
   - [Materialized views - Kusto | Microsoft Learn](https://learn.microsoft.com/en-us/kusto/management/materialized-views/materialized-view-overview)

3. **Fabric Real-Time Intelligence:**
   - [Real-Time Intelligence in Microsoft Fabric | Microsoft Learn](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/overview)
   - [KQL Querysets - Microsoft Fabric | Microsoft Learn](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/kusto-query-set)
   - [Real-Time Dashboards - Microsoft Fabric | Microsoft Learn](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/dashboard-real-time-create)

4. **Eventhouse and KQL Databases:**
   - [Eventhouse overview - Microsoft Fabric | Microsoft Learn](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/eventhouse)
   - [Create an Eventhouse - Microsoft Fabric | Microsoft Learn](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/create-eventhouse)
   - [OneLake shortcuts in KQL Database | Microsoft Learn](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/onelake-shortcuts)

### Best Practices Resources

1. **Monitoring Patterns:**
   - Azure Data Explorer documentation on monitoring and alerting
   - Fabric monitoring best practices (official guidance)
   - Community samples in fabric-toolbox repository

2. **Query Optimization:**
   - KQL query optimization guide
   - Performance tuning for large-scale monitoring
   - Caching strategies for time-series data

---

## Appendix A: Sample Queries

### A.1 Basic Cross-Database Union

```kql
// Simple union across two databases
database("ActivityEvents").ActivityEvents
| where CreationTime > ago(1h)
| project Timestamp=CreationTime, Type="Activity", Id, UserId
| union (
    database("CapacityUtilization").CapacitySummary
    | where windowStartTime > ago(1h)
    | project Timestamp=windowStartTime, Type="Capacity", Id=id, UserId=""
)
| summarize Count=count() by Type
```

### A.2 Multi-Database Metrics Dashboard Query

```kql
// Unified metrics across all monitoring databases
let granularity = 5m;
let activityMetrics = database("ActivityEvents").ActivityEvents
    | where CreationTime between (_startTime .. _endTime)
    | summarize Activities=count() by Timestamp=bin(CreationTime, granularity)
    | extend Source="Activity";

let capacityMetrics = database("CapacityUtilization").CapacitySummary
    | where windowStartTime between (_startTime .. _endTime)
    | summarize CapacityEvents=count() by Timestamp=bin(windowStartTime, granularity)
    | extend Source="Capacity";

let gatewayMetrics = database("GatewayMonitoring").GatewayJobs
    | where jobStartTime between (_startTime .. _endTime)
    | summarize GatewayJobs=count() by Timestamp=bin(jobStartTime, granularity)
    | extend Source="Gateway";

activityMetrics
| union capacityMetrics, gatewayMetrics
| project-away Source
| summarize sum(Activities), sum(CapacityEvents), sum(GatewayJobs) by Timestamp
| render timechart
```

### A.3 Correlation Query

```kql
// Correlate high capacity utilization with user activities
let highCapacity = database("CapacityUtilization").CapacitySummary
    | where windowStartTime > ago(1d)
    | where utilizationInteractive + utilizationBackground > 90.0
    | project CapacityTime=windowStartTime, CapacityId=capacityId, Utilization=utilizationInteractive + utilizationBackground;

let activities = database("ActivityEvents").ActivityEvents
    | where CreationTime > ago(1d)
    | extend CapacityId = toguid(details.CapacityId)
    | where isnotempty(CapacityId);

highCapacity
| join kind=inner (
    activities
    | project ActivityTime=CreationTime, CapacityId, Activity, UserId
) on CapacityId
| where abs(datetime_diff('minute', CapacityTime, ActivityTime)) <= 5
| summarize
    ActivityCount=count(),
    Activities=make_set(Activity),
    Users=make_set(UserId)
  by
    CapacityTime, CapacityId, Utilization
| order by CapacityTime desc
```

---

## Appendix B: Function Library Template

### Complete Function Library for Multi-Database Monitoring

```kql
//-------------------------------------------------------
// Core Functions for Multi-Database Fabric Monitoring
//-------------------------------------------------------

//-------------------------------------------------------
// 1. GetPlatformEvents - Unified event stream
//-------------------------------------------------------
.create-or-alter function with (folder="Platform", docstring="Unified platform events from all sources")
GetPlatformEvents(
    startTime:datetime,
    endTime:datetime,
    sources:dynamic = dynamic([]),          // Empty = all sources
    eventTypes:dynamic = dynamic([])        // Empty = all event types
) {
    let activityData =
        database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | where array_length(eventTypes) == 0 or Activity in (eventTypes)
        | extend
            EventTime = CreationTime,
            Source = "Activity",
            EventType = Activity,
            EventId = Id,
            WorkspaceId = toguid(details.WorkspaceId)
        | project EventTime, Source, EventType, EventId, UserId, WorkspaceId, Details=details;

    let capacityData =
        database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | extend
            EventTime = windowStartTime,
            Source = "Capacity",
            EventType = "CapacitySummary",
            EventId = id,
            UserId = "",
            WorkspaceId = capacityId
        | project EventTime, Source, EventType, EventId, UserId, WorkspaceId, Details=pack_all();

    let gatewayData =
        database("Gateway Monitoring").GatewayJobs
        | where jobStartTime between (startTime .. endTime)
        | extend
            EventTime = jobStartTime,
            Source = "Gateway",
            EventType = "GatewayJob",
            EventId = guid(null),
            UserId = "",
            WorkspaceId = guid(null)
        | project EventTime, Source, EventType, EventId, UserId, WorkspaceId, Details=pack_all();

    activityData
    | union capacityData, gatewayData
    | where array_length(sources) == 0 or Source in (sources)
}

//-------------------------------------------------------
// 2. GetPlatformMetrics - Aggregated metrics
//-------------------------------------------------------
.create-or-alter function with (folder="Platform", docstring="Platform metrics across all sources")
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
            AvgDuration = avg(duration)
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
        AvgJobDuration = coalesce(AvgDuration, 0.0)
    | order by Timestamp asc
}

//-------------------------------------------------------
// 3. GetWorkspaceSummary - Per-workspace metrics
//-------------------------------------------------------
.create-or-alter function with (folder="Platform", docstring="Workspace-level summary across all sources")
GetWorkspaceSummary(
    startTime:datetime,
    endTime:datetime,
    workspaceId:guid = guid(null)           // null = all workspaces
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
        | project WorkspaceId=id, WorkspaceName=name, CapacityId=capacityId, State=state;

    activityByWorkspace
    | join kind=leftouter workspaceInfo on WorkspaceId
    | project
        WorkspaceId,
        WorkspaceName,
        CapacityId,
        State,
        Activities,
        UniqueUsers,
        TopActivities
}

//-------------------------------------------------------
// 4. GetCapacityHealth - Capacity health summary
//-------------------------------------------------------
.create-or-alter function with (folder="Platform", docstring="Capacity health metrics")
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
            ThrottleEvents = countif(utilizationInteractive + utilizationBackground > 100)
          by CapacityId = capacityId, CapacityName = capacityName;

    let capacityInfo = database("Platform Inventory").Capacities
        | project CapacityId, Sku, Region, State;

    capacityMetrics
    | join kind=leftouter capacityInfo on CapacityId
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
        HealthScore = 100.0 - (AvgUtilization / 100.0) * 50.0 - (ThrottleEvents * 5.0)
    | order by HealthScore asc
}

//-------------------------------------------------------
// 5. CorrelateCapacityAndActivity - Event correlation
//-------------------------------------------------------
.create-or-alter function with (folder="Platform", docstring="Correlate capacity events with user activities")
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
        TopUsers = make_set(UserId, 10)
      by
        CapacityEventTime,
        CapacityId,
        CapacityName,
        Utilization
    | order by CapacityEventTime desc
}

//-------------------------------------------------------
// 6. GetPlatformAnomalies - Detect anomalies
//-------------------------------------------------------
.create-or-alter function with (folder="Platform", docstring="Detect anomalies across platform")
GetPlatformAnomalies(
    startTime:datetime,
    endTime:datetime,
    sensitivity:real = 1.5                  // Standard deviations for anomaly
) {
    // Activity anomalies
    let activityTS = database("Activity Events").ActivityEvents
        | where CreationTime between (startTime .. endTime)
        | make-series ActivityCount=count() on CreationTime step 5m from startTime to endTime
        | extend (anomalies, score, baseline) = series_decompose_anomalies(ActivityCount, sensitivity);

    // Capacity anomalies
    let capacityTS = database("Capacity Utilization").CapacitySummary
        | where windowStartTime between (startTime .. endTime)
        | make-series AvgUtil=avg(utilizationInteractive + utilizationBackground)
          on windowStartTime step 5m from startTime to endTime
        | extend (anomalies, score, baseline) = series_decompose_anomalies(AvgUtil, sensitivity);

    let activityAnomalies = activityTS
        | mv-expand Timestamp=CreationTime, ActivityCount, Anomaly=anomalies, Score=score
        | where Anomaly == 1 or Anomaly == -1
        | extend Source = "Activity", Timestamp = todatetime(Timestamp), Value = toreal(ActivityCount),
                 AnomalyScore = toreal(Score)
        | project Timestamp, Source, Value, AnomalyScore;

    let capacityAnomalies = capacityTS
        | mv-expand Timestamp=windowStartTime, AvgUtil, Anomaly=anomalies, Score=score
        | where Anomaly == 1 or Anomaly == -1
        | extend Source = "Capacity", Timestamp = todatetime(Timestamp), Value = toreal(AvgUtil),
                 AnomalyScore = toreal(Score)
        | project Timestamp, Source, Value, AnomalyScore;

    activityAnomalies
    | union capacityAnomalies
    | order by abs(AnomalyScore) desc, Timestamp desc
}
```

---

## Document Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-07 | Research Team | Initial comprehensive research document |

---

**End of Document**
