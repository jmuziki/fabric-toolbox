# Design Review Summary: Multi-Database KQL Support

**Status:** ❌ **NO-GO** - Requires significant design iteration

**Confidence:** 47% (Target: 95%+)

---

## TL;DR

The proposed design to add multi-database support via union statements in 23 base queries has **critical flaws** that would lead to:

- ⚠️ **Performance degradation** (no benchmarking done)
- ⚠️ **Maintenance nightmare** (23 duplicated union queries)
- ⚠️ **Data model errors** (WorkspaceName column doesn't exist)
- ⚠️ **Security gaps** (no permission model)
- ⚠️ **Breaking changes** (untested backward compatibility)

**The design contradicts comprehensive research already completed** that recommends a function-based federation approach.

---

## Critical Issues (Must Fix)

### 1. Architecture Mismatch ❌

**Problem:** Design proposes union-based modifications to 23 base queries

**Solution:** Use function-based federation (7-10 functions in Core database)

**Why:**
- Aligns with completed research recommendations
- Reduces maintenance from 23 queries to 7 functions
- Enables centralized optimization
- Follows DRY principle

### 2. Data Model Error ❌

**Problem:** Design assumes `WorkspaceName` column exists for filtering

**Reality:** Source tables use `WorkspaceId` (GUID), not `WorkspaceName` (string)

**Solution:**
- Filter by WorkspaceId before union
- Provide user-friendly name-to-ID parameter query
- Join with Platform Inventory for display names only

**Impact:** 10-50x performance improvement from pre-union filtering

### 3. No Performance Validation ❌

**Problem:** No benchmarking, testing, or performance thresholds defined

**Solution:**
- Test with 1, 2, 3, 5 databases
- Define acceptable latency (< 10s P95)
- Implement monitoring and alerting
- Document optimization strategy

### 4. No Permission Model ❌

**Problem:** Cross-database access control not designed or implemented

**Solution:**
- Pre-check database permissions
- Show clear warning for inaccessible databases
- Audit all cross-database access
- Provide "Test Connection" functionality

### 5. Backward Compatibility Risk ❌

**Problem:** Vague "empty dataSources array" claim without implementation

**Solution:**
- Define explicit default behavior
- Provide migration script
- Test old dashboards in new version
- Document upgrade path with rollback

---

## Recommended Approach

### Option 1: Function-Based Federation (RECOMMENDED) ⭐

**Architecture:**
```
Workspace Monitoring Core Database
├── GetSemanticModelLogs(startTime, endTime, workspaceIds)
├── GetEventhouseLogs(startTime, endTime, workspaceIds)
├── GetItemJobLogs(startTime, endTime, workspaceIds)
└── ... (7-10 total functions)
```

**Dashboard base queries:**
```kql
// OLD (proposed):
SemanticModelLogs
| union database("DB2").SemanticModelLogs, database("DB3").SemanticModelLogs
| where WorkspaceName in (_WorkspaceName)  // Column doesn't exist!

// NEW (recommended):
database('Workspace Monitoring Core').GetSemanticModelLogs(
    _startTime,
    _endTime,
    _workspaceIds  // Uses IDs, not names
)
```

**Benefits:**
- ✅ Single source of truth for cross-database logic
- ✅ Easy to optimize performance centrally
- ✅ Aligns with completed research
- ✅ Reduces code duplication from 23 to 7-10 functions

**Effort:** 6-8 weeks for proper implementation

**Confidence:** 95%

### Option 2: KQL Shortcuts (ALTERNATIVE)

Create shortcuts to remote tables in consolidated database:

```
Consolidated Database:
├── SemanticModelLogs_WS1 → Shortcut to Workspace1 Monitoring
├── SemanticModelLogs_WS2 → Shortcut to Workspace2 Monitoring
└── SemanticModelLogs_WS3 → Shortcut to Workspace3 Monitoring
```

**Benefits:**
- ✅ Simpler query syntax
- ✅ Centralized management
- ✅ Better query optimization

**Drawbacks:**
- ⚠️ Requires admin to manage shortcuts
- ⚠️ One more database to maintain

**Effort:** 1 week for setup + testing

**Confidence:** 85%

---

## Timeline to Implementation-Ready

### Current State → 95% Confidence: 6-8 weeks

**Week 1-2: Foundation**
- Create Core database
- Deploy 3-5 core functions
- Test with 2-3 databases
- Fix WorkspaceId filtering

**Week 3-4: Validation**
- Performance benchmarking
- Permission model implementation
- Backward compatibility testing
- User acceptance testing

**Week 5-6: Completion**
- Deploy remaining functions
- Update all 23 base queries
- Add monitoring dashboard
- Complete documentation

**Week 7-8: Rollout**
- Phased deployment (10% → 25% → 50% → 100%)
- Monitor performance and errors
- Gather user feedback
- Iterate as needed

---

## What Must Happen Before Implementation

### Critical Path (Cannot Skip):

1. ✅ **Adopt function-based architecture**
   - Aligns with research
   - Reduces complexity
   - Enables optimization

2. ✅ **Fix data model issues**
   - Use WorkspaceId (GUID) not WorkspaceName (string)
   - Pre-union filtering for performance
   - Document inventory dependency

3. ✅ **Complete performance testing**
   - Benchmark single vs. multi-database
   - Define acceptable thresholds
   - Implement optimization strategy

4. ✅ **Define permission model**
   - Pre-check database access
   - Clear error messages
   - Audit logging

5. ✅ **Prove backward compatibility**
   - Test old dashboards
   - Define migration path
   - Document rollback procedure

---

## Key Decisions

| Decision | Status | Rationale |
|----------|--------|-----------|
| Use functions instead of unions | ✅ RECOMMENDED | Aligns with research, reduces complexity |
| Filter by WorkspaceId not WorkspaceName | ✅ REQUIRED | Column exists, enables pre-union filtering |
| Performance test before implementation | ✅ REQUIRED | High risk of degradation without testing |
| Implement permission pre-check | ✅ REQUIRED | Security and user experience |
| Phased rollout (not big bang) | ✅ RECOMMENDED | Reduces blast radius of issues |

---

## Risk Assessment

| Risk | Current Level | Acceptable Level | Mitigation |
|------|---------------|------------------|------------|
| Performance | 🔴 HIGH | 🟢 LOW | Benchmark + optimize |
| Security | 🔴 HIGH | 🟢 LOW | Permission model + audit |
| Maintainability | 🔴 HIGH | 🟢 LOW | Function-based architecture |
| Breaking Changes | 🔴 HIGH | 🟢 LOW | Backward compatibility testing |
| Scalability | 🟡 MEDIUM | 🟢 LOW | Scale testing + documentation |

---

## Questions for Design Team

1. **Why was the union-based approach chosen over function-based federation?**
   - Research explicitly recommends functions
   - Union approach creates 23 duplicated query modifications

2. **Has the WorkspaceName column been validated to exist in source tables?**
   - Base queries show WorkspaceId (GUID), not WorkspaceName
   - This will cause query failures

3. **What is the acceptable query latency for multi-database scenarios?**
   - Need defined thresholds (e.g., < 10s P95)
   - No benchmarking has been done

4. **How will permissions be handled when users lack access to some databases?**
   - Current design has no permission model
   - Will queries fail silently or show errors?

5. **What is the migration path for existing dashboards?**
   - "Empty dataSources array" is vague
   - Need tested migration procedure

---

## Next Actions

### Immediate (This Week):

1. ⏸️ **PAUSE implementation work**
   - Do not modify 23 base queries yet
   - Avoid creating technical debt

2. 📖 **Review research documentation**
   - Read `/monitoring/fabric-platform-monitoring/research/`
   - Align team on function-based approach

3. 📅 **Schedule design iteration workshop**
   - Duration: 4 hours
   - Participants: Architect, Lead Dev, Product Owner
   - Goal: Align on recommended architecture

4. 📝 **Create revised design document**
   - Function-based architecture
   - WorkspaceId filtering
   - Performance requirements
   - Permission model

---

## Success Criteria for Revised Design

Before implementation can begin:

- ✅ Function-based architecture documented and approved
- ✅ WorkspaceId filtering implemented and tested
- ✅ Performance benchmarks completed with acceptable results
- ✅ Permission model implemented with clear error handling
- ✅ Backward compatibility tested with 10+ existing dashboards
- ✅ Monitoring dashboard created for query health
- ✅ Phased rollout plan documented and approved
- ✅ Team confidence level reaches 95%+

---

## References

**Full Design Review:** [DESIGN-REVIEW-Multi-Database-Support.md](./DESIGN-REVIEW-Multi-Database-Support.md)

**Research Documentation:**
- Executive Summary: `/monitoring/fabric-platform-monitoring/research/EXECUTIVE-SUMMARY.md`
- Implementation Template: `/monitoring/fabric-platform-monitoring/research/Implementation-Template-Functions.md`
- KQL Strategies: `/monitoring/fabric-platform-monitoring/research/KQL-Multi-Database-Union-Strategies.md`

---

**Review Date:** 2026-02-07
**Next Review:** After design iteration (2 weeks)
**Status:** ❌ NO-GO (requires iteration to achieve GO)
