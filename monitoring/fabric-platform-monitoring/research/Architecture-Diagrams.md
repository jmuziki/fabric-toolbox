# Architecture Diagrams for Multi-Database KQL Strategies

## Current Architecture: Separate Databases

```
Fabric Platform Monitoring Eventhouse
│
├─── Activity Events Database
│    ├── ActivityEventsRaw (table)
│    ├── ActivityEvents (table)
│    ├── ItemsViews() (function)
│    └── ActivitySummaryByHour (materialized view)
│
├─── Capacity Utilization Database
│    ├── CapacityEventsRaw (table)
│    ├── CapacitySummary (table)
│    ├── CapacityState (table)
│    └── CapacityWorkloadSummary (table)
│
├─── Gateway Monitoring Database
│    ├── GatewayJobs (table)
│    ├── GatewayHeartbeat (table)
│    └── GatewaySystemCounters (table)
│
└─── Platform Inventory Database
     ├── Workspaces (table)
     ├── Capacities (table)
     ├── Items (table)
     └── TenantSettings (table)
```

## Problem: Cross-Database Queries in Dashboards

### Before: Direct Database Queries

```
Real-Time Dashboard
│
├─── Tile 1: Activity Count
│    └─── Query: database("Activity Events").ActivityEvents
│         | where CreationTime > ago(1h)
│         | count
│
├─── Tile 2: Capacity Utilization
│    └─── Query: database("Capacity Utilization").CapacitySummary
│         | where windowStartTime > ago(1h)
│         | summarize avg(utilizationInteractive)
│
└─── Tile 3: Combined View (❌ Complex!)
     └─── Query: database("Activity Events").ActivityEvents
          | where CreationTime > ago(1h)
          | extend Source = "Activity"
          | union (
              database("Capacity Utilization").CapacitySummary
              | where windowStartTime > ago(1h)
              | extend Source = "Capacity"
          )
          | summarize count() by Source
          // Repeated across many tiles - hard to maintain!
```

**Issues:**
- Duplicated cross-database logic
- Hard to maintain across many dashboards
- Difficult to optimize performance
- Complex queries in dashboard tiles

## Solution: Function-Based Abstraction

### Recommended Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Core Database                            │
│  (Platform Core or Platform Inventory)                      │
│                                                              │
│  Cross-Database Functions:                                  │
│  ├── GetPlatformEvents(startTime, endTime, sources)        │
│  ├── GetPlatformMetrics(startTime, endTime, granularity)   │
│  ├── GetWorkspaceSummary(startTime, endTime, workspaceId)  │
│  ├── GetCapacityHealth(startTime, endTime)                 │
│  ├── CorrelateCapacityAndActivity(...)                     │
│  ├── GetPlatformOverview(startTime, endTime)               │
│  └── GetActivityHotspots(startTime, endTime, topN)         │
│                                                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┬──────────────┐
        │              │              │              │
        ▼              ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Activity   │ │   Capacity   │ │   Gateway    │ │  Inventory   │
│    Events    │ │ Utilization  │ │  Monitoring  │ │   Database   │
│              │ │              │ │              │ │              │
│ - Raw tables │ │ - Raw tables │ │ - Raw tables │ │ - Workspaces │
│ - Parsed     │ │ - Summaries  │ │ - Jobs       │ │ - Capacities │
│ - MVs        │ │ - MVs        │ │ - Heartbeat  │ │ - Items      │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

### After: Function-Based Queries

```
Real-Time Dashboard
│
├─── Tile 1: Activity Count
│    └─── Query: GetPlatformMetrics(_startTime, _endTime, 5m)
│         | project Timestamp, ActivityCount
│         | render timechart
│
├─── Tile 2: Capacity Utilization
│    └─── Query: GetPlatformMetrics(_startTime, _endTime, 5m)
│         | project Timestamp, AvgUtilization
│         | render timechart
│
└─── Tile 3: Combined View (✅ Simple!)
     └─── Query: GetPlatformEvents(_startTime, _endTime, _sources)
          | summarize Count=count() by Source
          | render piechart
          // Clean, maintainable, optimized!
```

**Benefits:**
- Single source of truth for cross-database logic
- Easy to maintain and optimize
- Consistent query patterns
- Simple dashboard queries

## Query Flow Diagram

### GetPlatformEvents() Function Flow

```
Dashboard calls:
GetPlatformEvents(ago(24h), now(), dynamic(["Activity", "Capacity"]))
│
▼
┌─────────────────────────────────────────────────────────────┐
│ Function: GetPlatformEvents()                               │
│                                                              │
│  1. Query Activity Events Database                          │
│     ├─ Filter: CreationTime between (startTime..endTime)   │
│     ├─ Transform: Standardize schema                        │
│     └─ Extend: Source = "Activity"                         │
│                                                              │
│  2. Query Capacity Utilization Database                     │
│     ├─ Filter: windowStartTime between (startTime..endTime)│
│     ├─ Transform: Standardize schema                        │
│     └─ Extend: Source = "Capacity"                         │
│                                                              │
│  3. Query Gateway Monitoring Database                       │
│     ├─ Filter: jobStartTime between (startTime..endTime)   │
│     ├─ Transform: Standardize schema                        │
│     └─ Extend: Source = "Gateway"                          │
│                                                              │
│  4. Union all results                                       │
│     └─ All three queries execute in PARALLEL               │
│                                                              │
│  5. Filter by requested sources                             │
│     └─ where Source in (sources)                           │
│                                                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
              Return unified result set
                       │
                       ▼
            Dashboard renders visualization
```

## Performance Optimization Strategy

### Layered Caching Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Query Execution Layer                       │
│                                                              │
│  Dashboard → GetPlatformMetrics(ago(24h), now(), 5m)       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Function Logic Layer                           │
│                                                              │
│  Decides: Use raw tables or materialized views?            │
│  ├─ If granularity >= 1h → Use MVs (fast)                 │
│  └─ If granularity < 1h → Use raw tables (flexible)       │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Activity   │ │   Capacity   │ │   Gateway    │
│   Database   │ │   Database   │ │   Database   │
│              │ │              │ │              │
│ Hot Cache    │ │ Hot Cache    │ │ Hot Cache    │
│ (7 days)     │ │ (7 days)     │ │ (7 days)     │
│              │ │              │ │              │
│ MVs:         │ │ MVs:         │ │ MVs:         │
│ - Hourly     │ │ - Hourly     │ │ - Hourly     │
│   Summary    │ │   Summary    │ │   Summary    │
│              │ │              │ │              │
│ Raw Tables:  │ │ Raw Tables:  │ │ Raw Tables:  │
│ - Events     │ │ - Summary    │ │ - Jobs       │
└──────────────┘ └──────────────┘ └──────────────┘
        │              │              │
        ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│                Cold Storage (Azure Storage)                 │
│              Retention: 730 days (2 years)                  │
└─────────────────────────────────────────────────────────────┘
```

### Query Path Decision Tree

```
User requests data with time range T and granularity G
│
├─ Is T within hot cache? (< 7 days)
│  ├─ YES → Query hot cache (fast - SSD)
│  └─ NO → Query cold storage (slower - blob storage)
│
├─ Is G >= 1 hour?
│  ├─ YES → Use materialized views (pre-aggregated)
│  └─ NO → Use raw tables (flexible)
│
└─ Apply filters and projections
   ├─ Filter by time in EACH branch BEFORE union
   ├─ Project only needed columns EARLY
   └─ Union results from all databases
```

## Alternative Architectures Considered

### Option 1: Consolidated Database (NOT Recommended for Current Use Case)

```
┌─────────────────────────────────────────────────────────────┐
│           Single Unified Monitoring Database                │
│                                                              │
│  All Events Table:                                          │
│  ├─ Source: "Activity", "Capacity", "Gateway"              │
│  ├─ EventTime, EventType, UserId, WorkspaceId, Details     │
│  └─ Populated via:                                          │
│      ├─ Eventstream → Direct ingestion                      │
│      └─ Data Pipeline → ETL from source systems            │
└─────────────────────────────────────────────────────────────┘
```

**Pros:**
- Simplest queries (single table)
- Best performance for aggregations
- No cross-database overhead

**Cons:**
- ❌ Data duplication
- ❌ Complex ingestion pipeline
- ❌ Schema changes affect everything
- ❌ Loss of source database benefits (specialized retention, etc.)

### Option 2: Materialized View Federation (NOT Possible)

```
❌ DOES NOT WORK - Shown for reference only

┌─────────────────────────────────────────────────────────────┐
│              Core Database                                   │
│                                                              │
│  .create materialized-view UnifiedEvents on ??? {           │
│      database("Activity Events").ActivityEvents             │
│      | union database("Capacity").CapacitySummary           │
│  }                                                           │
│                                                              │
│  ERROR: Materialized views cannot reference database()      │
└─────────────────────────────────────────────────────────────┘
```

**Why it doesn't work:**
- Materialized views must be single-database
- Cannot use database() function in view definition
- Workaround: Use functions instead (recommended approach)

### Option 3: Shortcuts (Alternative Recommended Approach)

```
┌─────────────────────────────────────────────────────────────┐
│              Core Database                                   │
│                                                              │
│  Shortcuts (appear as local tables):                        │
│  ├─ ActivityEvents → database("Activity Events")           │
│  ├─ CapacitySummary → database("Capacity")                 │
│  └─ GatewayJobs → database("Gateway")                      │
│                                                              │
│  Queries appear single-database:                            │
│  ActivityEvents                                             │
│  | union CapacitySummary                                    │
│  | union GatewayJobs                                        │
└─────────────────────────────────────────────────────────────┘
```

**Pros:**
- Queries look like single-database
- Permissions still enforced at source
- Simpler syntax

**Cons:**
- Still cross-database under the hood
- Same performance characteristics
- Shortcuts must be maintained

## Comparison Matrix

| Approach | Complexity | Performance | Flexibility | Maintenance | Recommended |
|----------|-----------|-------------|-------------|-------------|-------------|
| **Functions** | Medium | Good | High | Medium | ✅ YES |
| **Consolidated DB** | High | Excellent | Low | High | ❌ No |
| **Direct Queries** | Low | Good | High | High | ⚠️ Only for simple cases |
| **Shortcuts** | Low | Good | High | Medium | ✅ Alternative |
| **Materialized Views** | N/A | N/A | N/A | N/A | ❌ Not Possible |

## Implementation Timeline

```
Week 1: Foundation
├─ Day 1-2: Create Core database and deploy functions
├─ Day 3-4: Configure hot cache and test functions
└─ Day 5: Document baseline performance

Week 2-3: Migration
├─ Day 1-3: Identify dashboards to migrate
├─ Day 4-7: Update dashboards to use functions
├─ Day 8-10: A/B test and validate
└─ Day 11-15: Complete migration

Week 4: Optimization
├─ Day 1-2: Create materialized views
├─ Day 3-4: Fine-tune hot cache
└─ Day 5: Set up performance monitoring

Ongoing: Maintenance
├─ Weekly: Review query performance
├─ Monthly: Optimize slow queries
└─ Quarterly: Review architecture decisions
```

---

**Document Purpose**: Visual reference for multi-database KQL strategies
**Last Updated**: 2026-02-07
