# KQL Multi-Database Research Documentation

This directory contains comprehensive research and implementation guidance for combining data from multiple KQL databases in Microsoft Fabric Real-Time Intelligence.

## Overview

The Fabric Platform Monitoring solution uses multiple separate KQL databases:
- **Activity Events**: User activity logs from audit APIs
- **Capacity Utilization**: Capacity metrics from Real Time Hub
- **Gateway Monitoring**: On-premises gateway logs
- **Platform Inventory**: Tenant inventory and workspace metadata

This research addresses the challenge of querying and combining data across these databases for unified monitoring and analytics.

## Documents

### 1. [KQL Multi-Database Union Strategies](./KQL-Multi-Database-Union-Strategies.md)
**Comprehensive Research Document** (55,887 characters, ~15 pages)

Complete technical research covering:
- **KQL Union Operations**: Syntax, performance, limitations
- **Dynamic Database References**: Parameterization constraints and workarounds
- **Alternative Approaches**: Materialized views, shortcuts, federation, ETL
- **Monitoring Patterns**: Enterprise monitoring solution patterns, time-series aggregation
- **Performance & Scalability**: Query optimization, caching, decision matrices

**Who should read this:**
- Architects designing multi-database solutions
- Engineers seeking deep understanding of cross-database queries
- Anyone evaluating consolidation vs. federation approaches

**Key findings:**
- Database names cannot be dynamically parameterized (security limitation)
- Union operations are performant for typical monitoring workloads (< 5M rows/db)
- Functions provide the best abstraction for cross-database logic
- Hot cache configuration is critical for performance

### 2. [Quick Reference: Multi-DB Patterns](./Quick-Reference-Multi-DB-Patterns.md)
**Practical Pattern Library** (12,139 characters, ~4 pages)

Ready-to-use query patterns:
- 10 common cross-database query patterns
- Performance optimization examples
- Common mistakes to avoid
- Dashboard parameter configuration
- Quick decision tree for when to use cross-database queries

**Who should read this:**
- Developers implementing cross-database queries
- Dashboard creators needing quick examples
- Anyone looking for copy-paste solutions

**Use cases:**
- Basic cross-database union
- Time-aligned metrics from multiple sources
- Event correlation across databases
- Anomaly detection across sources
- Health scoring from multiple metrics

### 3. [Implementation Template: Functions](./Implementation-Template-Functions.md)
**Deployment-Ready Code** (21,209 characters, ~7 pages)

Production-ready KQL functions:
- 7 complete cross-database functions
- Deployment instructions and checklist
- Dashboard integration examples
- Testing queries
- Performance optimization recommendations

**Who should read this:**
- Engineers ready to implement cross-database queries
- DevOps teams deploying monitoring solutions
- Anyone needing production-ready code

**Functions included:**
1. `GetPlatformEvents` - Unified event stream
2. `GetPlatformMetrics` - Time-series metrics
3. `GetWorkspaceSummary` - Per-workspace analytics
4. `GetCapacityHealth` - Capacity health scoring
5. `CorrelateCapacityAndActivity` - Event correlation
6. `GetPlatformOverview` - High-level summary
7. `GetActivityHotspots` - Identify activity hotspots

## Quick Start

### For Immediate Implementation

1. **Read:** [Quick Reference](./Quick-Reference-Multi-DB-Patterns.md) (10 minutes)
2. **Deploy:** [Implementation Template](./Implementation-Template-Functions.md) (30 minutes)
3. **Test:** Run validation queries from the template
4. **Integrate:** Update dashboards to use new functions

### For Strategic Planning

1. **Read:** [Comprehensive Research](./KQL-Multi-Database-Union-Strategies.md) (45 minutes)
2. **Evaluate:** Review alternative approaches (Section 3)
3. **Decide:** Use decision matrix (Section 5.4) to choose approach
4. **Plan:** Follow recommended implementation strategy (Section 6)

## Key Recommendations

### Immediate Actions
1. **Create a Core database** with cross-database functions
2. **Configure hot cache** to 7-14 days on all monitoring tables
3. **Standardize time filtering** across all databases
4. **Use functions** from Core database for all cross-database queries

### Dashboard Strategy
1. Use functions for abstraction and maintainability
2. Implement filter-based source selection (not parameterized database names)
3. Add dynamic granularity based on time range
4. Cache dashboard queries with 1-5 minute freshness

### Performance Monitoring
1. Track query duration for cross-database queries
2. Set alerts for queries exceeding 10 seconds
3. Review query patterns monthly

## Use Cases Covered

### Unified Monitoring
- Combine events from Activity, Capacity, and Gateway sources
- Time-aligned metrics across all databases
- Correlation of events across different sources

### Capacity Management
- Correlate capacity utilization with user activities
- Identify workspaces causing capacity pressure
- Track capacity health metrics

### Platform Analytics
- Platform-wide overview across all data sources
- Activity hotspot identification
- Anomaly detection across databases

### Workspace Analytics
- Per-workspace activity summary
- Capacity impact by workspace
- User engagement metrics

## Technical Constraints

### What Works
- ✅ Cross-database unions with `database("name").Table` syntax
- ✅ Functions encapsulating cross-database queries
- ✅ Filter-based parameter selection after union
- ✅ Time-series aggregation across databases
- ✅ Case-based database selection

### What Doesn't Work
- ❌ Parameterized database names: `database(variableName).Table`
- ❌ Materialized views with cross-database queries
- ❌ Dynamic query construction with database selection
- ❌ String interpolation in database() function

### Workarounds
- Use case statements with string literals
- Union all databases, then filter by source
- Create functions for common patterns
- Use shortcuts for virtual local access

## Performance Guidelines

### Query Performance Targets
- Dashboard queries: < 5 seconds
- Ad-hoc exploration: < 10 seconds
- Background aggregation: < 60 seconds

### When Performance Degrades
- Data volume exceeds 50M rows per database
- Query duration consistently > 10 seconds
- Dashboard refresh is too slow

**Consider:**
- ETL consolidation to unified database
- More aggressive pre-aggregation (materialized views)
- Tiered architecture (hot/warm/cold)

## Architecture Decision

### Current Approach: Query-Time Federation
**Pros:**
- No data duplication
- Flexible schema evolution
- Source databases remain independent
- Simple to implement and maintain

**Cons:**
- Query performance depends on data volume
- More complex dashboard queries
- Higher compute cost per query

**Best For:**
- Current requirements (multiple independent databases)
- Moderate data volumes (< 5M rows per db)
- Evolving schemas
- Regulatory separation needs

### Alternative: Consolidated Database
**Consider when:**
- Query performance is consistently poor
- Dashboard complexity becomes unmanageable
- Real-time latency requirements (< 1s)
- Historical analysis is primary use case

## Implementation Checklist

### Phase 1: Foundation (Week 1)
- [ ] Create Core database (or designate existing)
- [ ] Deploy 7 core functions from template
- [ ] Configure hot cache on all databases (7 days)
- [ ] Test functions with sample queries
- [ ] Document baseline query performance

### Phase 2: Dashboard Migration (Week 2-3)
- [ ] Identify dashboards using cross-database queries
- [ ] Update dashboards to use functions
- [ ] Add parameters for source selection
- [ ] Test dashboard performance
- [ ] A/B test old vs. new approach

### Phase 3: Optimization (Week 4)
- [ ] Create materialized views for hourly aggregations
- [ ] Add indexes if needed
- [ ] Fine-tune hot cache based on usage
- [ ] Set up performance monitoring
- [ ] Document query patterns

### Phase 4: Monitoring (Ongoing)
- [ ] Track query duration metrics
- [ ] Alert on slow queries (> 10s)
- [ ] Review monthly for optimization opportunities
- [ ] Update functions as schemas evolve

## Support and Maintenance

### Monthly Tasks
- Review function usage with `.show functions`
- Check query performance with `.show queries`
- Update functions based on schema changes
- Add new functions for emerging patterns

### When Schemas Change
1. Update affected functions
2. Test with existing dashboards
3. Update documentation
4. Communicate changes to dashboard users

### When Performance Degrades
1. Check query execution with `.show queries`
2. Review hot cache effectiveness
3. Consider adding materialized views
4. Evaluate if consolidation is needed

## References

### Microsoft Documentation
- [Union operator - Kusto](https://learn.microsoft.com/kusto/query/union-operator)
- [Cross-database queries - Kusto](https://learn.microsoft.com/kusto/query/cross-cluster-queries)
- [Database() function - Kusto](https://learn.microsoft.com/kusto/query/database-function)
- [Real-Time Intelligence - Fabric](https://learn.microsoft.com/fabric/real-time-intelligence/overview)
- [KQL Databases - Fabric](https://learn.microsoft.com/fabric/real-time-intelligence/eventhouse)

### Internal Resources
- [Platform Monitoring README](../README.md)
- [DatabaseSchema.kql files](../src/) in each database folder

## Version History

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-02-07 | Initial research and implementation guidance |

## Contributing

When updating these documents:
1. Test all code examples before committing
2. Update version history
3. Keep examples aligned across all three documents
4. Validate against current Fabric RTI capabilities

## Questions?

For questions about this research:
1. Review the comprehensive research document first
2. Check the quick reference for common patterns
3. Try the implementation template functions
4. Open an issue with specific questions

---

**Research Team**: Fabric Platform Monitoring Contributors
**Last Updated**: 2026-02-07
**Status**: Production-Ready
