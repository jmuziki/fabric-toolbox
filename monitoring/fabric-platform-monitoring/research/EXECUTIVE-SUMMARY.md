# KQL Multi-Database Strategies - Executive Summary

## Research Overview

**Date:** 2026-02-07
**Branch:** claude/update-template-multi-kql-databases
**Context:** Fabric Platform Monitoring with multiple KQL databases

## What Was Researched

### 1. KQL Union Operations
- Cross-database union syntax and capabilities
- Performance characteristics with multiple databases
- Limitations (max databases, result size, timeouts)
- Best practices for filtering and projection

### 2. Dynamic Database References
- Constraints: Database names CANNOT be parameterized in Real-Time Dashboards
- Security requirement: database() requires string literals
- Workarounds: Case statements, union-then-filter, function abstraction

### 3. Alternative Approaches
- Materialized views: Single-database only
- KQL Shortcuts: Recommended for local-like access
- Federated queries: Supported within same tenant
- ETL consolidation: When query performance is insufficient

### 4. Monitoring-Specific Patterns
- Hub-and-spoke model with central function database
- Time-series aggregation across sources
- Event correlation across databases
- Composite health metrics

### 5. Performance and Scalability
- Query performance targets: < 5s for dashboards, < 10s for ad-hoc
- Hot cache critical: 7-30 days recommended
- Pre-aggregation with materialized views for hourly+ data
- Decision matrix for when to avoid cross-database queries

## Key Findings

### ✅ What Works Well

1. **Cross-Database Unions**
   - Fully supported with `database("name").Table` syntax
   - Performant for < 5M rows per database
   - Query optimizer pushes filters down to each branch

2. **Function-Based Abstraction**
   - Create functions in Core database
   - Encapsulate cross-database logic
   - Simplify dashboard queries
   - Centralize maintenance

3. **Filter-Based Selection**
   - Union all sources, filter by source column
   - Works with dashboard parameters
   - Flexible for user selection

### ❌ What Doesn't Work

1. **Parameterized Database Names**
   - `database(variableName).Table` - NOT supported
   - Security enforcement at parse time
   - No dynamic database selection

2. **Cross-Database Materialized Views**
   - Materialized views must be single-database
   - Cannot reference `database()` in view definition
   - Use functions instead

3. **Dynamic Query Construction**
   - No string interpolation for database names
   - Limited control flow in dashboards
   - Must use case statements with literals

### ⚠️ Important Constraints

1. **Real-Time Dashboard Limitations**
   - Parse-time validation of database references
   - No stored procedures or multi-statement scripts
   - Parameters cannot influence database selection directly

2. **Performance Considerations**
   - All union results must fit in memory on coordinator
   - Data transfer overhead between databases
   - Query timeout: 4 minutes default, 1 hour max

## Recommended Implementation

### Hybrid Function-Based Federation

**Architecture:**
```
Core Database (Platform Core or Inventory)
├── Functions (cross-database query logic)
│   ├── GetPlatformEvents()
│   ├── GetPlatformMetrics()
│   ├── GetWorkspaceSummary()
│   └── ... (7 total functions)
│
├── Activity Events Database
│   └── Raw activity logs + materialized views
│
├── Capacity Utilization Database
│   └── Capacity metrics + materialized views
│
├── Gateway Monitoring Database
│   └── Gateway logs + jobs
│
└── Platform Inventory Database
    └── Workspaces, capacities, items
```

**Benefits:**
- No data duplication
- Flexible schema evolution
- Source databases remain independent
- Simple dashboard queries via functions

**Trade-offs:**
- Query performance depends on data volume
- Higher compute cost per query
- More complex implementation

### Implementation Steps

1. **Week 1: Foundation**
   - Create Core database
   - Deploy 7 core functions
   - Configure hot cache (7 days)
   - Test and validate

2. **Week 2-3: Migration**
   - Update dashboards to use functions
   - Add source selection parameters
   - A/B test performance
   - Validate results

3. **Week 4: Optimization**
   - Add materialized views for hourly aggregations
   - Fine-tune hot cache
   - Set up performance monitoring
   - Document patterns

## Deliverables

### 📄 Research Documents

1. **KQL-Multi-Database-Union-Strategies.md** (1,872 lines)
   - Comprehensive technical research
   - All 5 research areas covered in depth
   - Production patterns and best practices
   - Complete function library in appendix

2. **Quick-Reference-Multi-DB-Patterns.md** (405 lines)
   - 10 ready-to-use query patterns
   - Common mistakes and how to avoid them
   - Performance optimization tips
   - Dashboard parameter examples

3. **Implementation-Template-Functions.md** (717 lines)
   - 7 production-ready KQL functions
   - Deployment instructions and checklist
   - Dashboard integration examples
   - Testing and validation queries

4. **README.md** (338 lines)
   - Overview and document guide
   - Quick start instructions
   - Implementation checklist
   - Support and maintenance guidance

### Total: ~3,332 lines of documentation

## Code Samples Provided

### Functions
1. `GetPlatformEvents()` - Unified event stream from all sources
2. `GetPlatformMetrics()` - Time-series metrics across databases
3. `GetWorkspaceSummary()` - Per-workspace analytics
4. `GetCapacityHealth()` - Capacity health scoring
5. `CorrelateCapacityAndActivity()` - Event correlation
6. `GetPlatformOverview()` - High-level summary
7. `GetActivityHotspots()` - Activity hotspot identification

### Query Patterns
- Basic cross-database union
- Parameter-based filtering
- Time-aligned metrics
- Correlation across databases
- Pre-aggregation for performance
- Dynamic time granularity
- Workspace-centric queries
- Health score calculation
- Anomaly detection

## Performance Guidelines

### Query Performance Targets
| Query Type | Target | Action if Exceeded |
|------------|--------|-------------------|
| Dashboard queries | < 5s | Add materialized views |
| Ad-hoc exploration | < 10s | Review filters and projections |
| Background aggregation | < 60s | Consider ETL consolidation |

### Optimization Checklist
- [ ] Filter by time in EACH branch before union
- [ ] Project only necessary columns early
- [ ] Configure hot cache (7-30 days)
- [ ] Create materialized views for hourly+ aggregations
- [ ] Use functions for complex cross-database logic
- [ ] Monitor query duration and set alerts

## When to Revisit This Strategy

**Triggers for Re-evaluation:**
1. Query performance consistently exceeds 10 seconds
2. Data volume grows beyond 50M rows per database
3. Dashboard complexity becomes unmanageable
4. Real-time latency requirements emerge (< 1s)
5. Compliance requires data consolidation

**Alternative Approaches to Consider:**
- ETL consolidation to single monitoring database
- Streaming aggregation using Eventstream
- Tiered architecture (hot/warm/cold data)
- Hybrid approach (recent + historical)

## Success Metrics

### Implementation Success
- [ ] All 7 functions deployed and tested
- [ ] Dashboard queries use functions consistently
- [ ] Query performance meets targets (< 5s)
- [ ] Hot cache configured on all databases
- [ ] Performance monitoring in place

### Operational Success
- [ ] Dashboard refresh time acceptable to users
- [ ] Query failures < 1% of executions
- [ ] Function updates don't break dashboards
- [ ] Team understands multi-database patterns
- [ ] Documentation kept up to date

## Next Steps

### Immediate Actions
1. Review research documents with team
2. Decide on Core database location
3. Deploy functions from template
4. Test with sample dashboards
5. Create performance baseline

### Follow-up (30 days)
1. Review query performance metrics
2. Identify slow queries for optimization
3. Add materialized views as needed
4. Update functions based on usage patterns
5. Train team on multi-database patterns

### Ongoing Maintenance
1. Monthly performance review
2. Update functions when schemas change
3. Add new functions for emerging patterns
4. Monitor for consolidation triggers
5. Keep documentation current

## References

### Internal Documents
- [Comprehensive Research](./KQL-Multi-Database-Union-Strategies.md)
- [Quick Reference](./Quick-Reference-Multi-DB-Patterns.md)
- [Implementation Template](./Implementation-Template-Functions.md)
- [Research README](./README.md)

### Microsoft Documentation
- [KQL Union Operator](https://learn.microsoft.com/kusto/query/union-operator)
- [Cross-Database Queries](https://learn.microsoft.com/kusto/query/cross-cluster-queries)
- [Fabric Real-Time Intelligence](https://learn.microsoft.com/fabric/real-time-intelligence/overview)

## Contact

For questions about this research:
1. Review documentation in this directory
2. Check examples in implementation template
3. Open issue with specific questions
4. Tag fabric-platform-monitoring team

---

## Research Conclusion

Cross-database queries in Fabric Real-Time Intelligence are **production-ready and performant** for the Fabric Platform Monitoring use case, with proper implementation:

1. **Use function-based abstraction** in a Core database
2. **Filter early** in each union branch
3. **Configure hot cache** appropriately (7-30 days)
4. **Monitor performance** and optimize as needed
5. **Consider consolidation** only when query performance is consistently poor

The hybrid function-based federation approach provides the best balance of:
- **Flexibility** - Independent database evolution
- **Performance** - Query optimization opportunities
- **Maintainability** - Centralized query logic
- **Cost** - No data duplication

This strategy is recommended for implementation and should support the Fabric Platform Monitoring solution's requirements for the foreseeable future.

---

**Status:** ✅ Research Complete - Ready for Implementation
**Confidence:** High - Based on Fabric RTI capabilities and monitoring best practices
**Risk:** Low - Well-understood patterns with clear alternatives if needed
