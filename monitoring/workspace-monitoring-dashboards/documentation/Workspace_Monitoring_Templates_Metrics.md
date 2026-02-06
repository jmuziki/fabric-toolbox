# Template metrics 
**within the Fabric Workspace Monitoring templates**

## General

This article describes all the common metrics within the Real-Time Dashboard and Power BI templates for Workspace Monitoring.

## References

#### Workspace Monitoring feature
- [What is workspace monitoring](https://learn.microsoft.com/en-us/fabric/fundamentals/workspace-monitoring-overview)

#### Semantic model logs

- [Semantic model operation logs](https://learn.microsoft.com/en-us/fabric/enterprise/powerbi/semantic-model-operations)
- [Semantic model 'ExecutionMetrics'](https://learn.microsoft.com/en-us/power-bi/transform-model/log-analytics/desktop-log-analytics-configure?tabs=refresh#events-and-schema)

#### Eventhouse logs

- [Eventhouse query logs](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/monitor-logs-query)
- [Eventhouse metrics](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/monitor-metrics)

-----


The following tables describe the defined metrics for the RT-Dashboard and Power BI report templates.

### Semantic model related metrics

Source table: **SemanticModelLogs**
|Source column|Metric name(s) in templates|Description|
|--|--|--|
|OperationId|# Operations|Unique count of operations <br> _Filtered for queries and refreshes only._|
|OperationId, Status |Operation Success Ratio|Percentage of successful unique operations. Higher is better. <br>_Filtered for queries and refreshes only._ |
|ExecutingUser|# Users/<br>Active Users|The user running the operation.|
|OperationId|# Queries/<br>Query Operations| Unique count of DAX/MDX query operations.|
|OperationId|# Refreshes/<br>Refresh Operations|Unique count of refresh operations.|
|OperationId, Status |# Errors|Unique count of failed operations.|
|executionDelayMs|Total Execution Delay Time/<br>Execution Delay|Total time spent waiting for Analysis Services engine thread pool thread availability.|
|capacityThrottlingMs|Total Throttling Time/<br>Capacity Throttling|Total time the request got delayed due to capacity throttling.|
|capacityThrottlingMs|Max Throttling Time/<br>Max Capacity Throttling|Max time the request got delayed due to capacity throttling.|
|CpuTimeMs|Total CPU Time|Total amount of CPU time (transferred in selected unit) used by the event/operation.|
|CpuTimeMs|Max CPU Time|Max amount of CPU time (transferred in selected unit) used by the event/operation.|
|queryProcessingCpuTimeMs|Total Query CPU Time|Total CPU time spent by tasks on Analysis Services query thread pool thread.|
|queryProcessingCpuTimeMs|Max Query CPU Time|Max CPU time spent by tasks on Analysis Services query thread pool thread.|
|mEngineCpuTimeMs|Total Power Query CPU Time/<br>Power Query CPU Time|Total CPU time spent by PowerQuery engine (Mashup engine) by the event/operation.|
|mEngineCpuTimeMs|Max Power Query CPU Time|Max CPU time spent by PowerQuery engine (Mashup engine) by the event/operation.|
|vertipaqJobCpuTimeMs|Total VertiPaq CPU Time/<br>VertiPaq CPU Time|Total CPU time spent by Vertipaq engine.|
|vertipaqJobCpuTimeMs|Max VertiPaq CPU Time|Max CPU time spent by Vertipaq engine.|
|CpuTimeMs|Total Progress CPU Time/<br>Progress CPU Time|Total amount of CPU time (transferred in selected unit) used by the refresh/progress event. <br> _Filtered for refresh sequences only._|
|durationMs|Total Duration Time|Total duration of the execution.|
|durationMs|Max Duration|Max duration of selected executions.|
|approximatePeakMemConsumptionKB|Total Memory Peak/Total Operation Memory Peak|Approximate peak total memory consumption during the request.|
|approximatePeakMemConsumptionKB|Max Memory Peak/Max Operation Memory Peak|Max of approximate total memory peak consumption during the request of selected operations.|
|mEnginePeakMemoryKB|Total Power Query Memory Peak|Approximate peak memory commit size (transferred in selected unit) across all PowerQuery engine mashup containers.|
|mEnginePeakMemoryKB|Max Power Query Memory Peak|Max of approximate peak memory commit size across all PowerQuery engine mashup containers of the selected operations.|
|extracted from EventText|Total Processed Objects/<br>Progress Object Count|Total count of progressed count of object (table rows, hierarchy objects etc.)|
|queryResultRows|Query Result Rows|Total number of rows returned as a result of the DAX query.|
|directQueryTotalTimeMs|Total DirectQuery Time|Total time spent on executing and reading all DirectQuery queries during the request.|
|directQueryConnectionTimeMs|DirectQuery Connection Time|Total time spent on creating new DirectQuery connection during the request|
|directQueryIterationTimeMs|DirectQuery Iteration Time|Total time spent on iterating the results returned by the DirectQuery queries.|
|externalQueryTimeoutMs|External Query Execution Time|Timeout associated with queries to external datasources.|
|directQueryRequestCount|DirectQuery Request Count|Total number of DirectQuery storage engine queries executed by the DAX engine.|
|directQueryTotalRows|DirectQuery Rows|Total number of DirectQuery storage engine queries executed by the DAX engine.|
|datasourceConnectionThrottleTimeMs|Datasource Connection Throttle Time|Total throttle time after hitting the datasource connection limit. Learn more about maximum concurrent connections [here](https://learn.microsoft.com/en-us/fabric/enterprise/powerbi/service-premium-what-is#semantic-model-sku-limitation).|
|tabularConnectionTimeoutMs|Tabular Connection Timeout|Timeout associated with external tabular datasource connections (e.g. SQL).|
|directQueryTimeoutMs|DirectQuery Timeout|Timeout associated with DirectQuery queries.|

----------------

### Item Job Event Logs Metrics

Source table: **ItemJobEventLogs**

|Source Column|Metric Name|Description|
|--|--|--|
|JobStatus|Total Jobs / Job Count|Count of all job executions across all item types (Pipelines, Notebooks, Lakehouses, Warehouses, Dataflows, etc.)|
|JobStatus|Success Rate|Percentage of jobs with "Completed" status. Higher is better. Calculated as (Succeeded / Total) * 100|
|JobStatus|Succeeded Jobs|Count of jobs with JobStatus = "Completed"|
|JobStatus|Failed Jobs|Count of jobs with JobStatus = "Failed"|
|JobStatus|In Progress Jobs|Count of jobs with JobStatus = "InProgress"|
|DurationMs|Avg Duration|Average job execution time in milliseconds. Calculated from job start to end time|
|DurationMs|P95 Duration|95th percentile job execution time in milliseconds. Useful for identifying outliers and setting SLA thresholds|
|DurationMs|Max Duration|Maximum job execution time observed in the selected time window|
|ItemKind|Jobs by Item Kind|Distribution of jobs across Fabric item types (Pipeline, Notebook, Lakehouse, Warehouse, CopyJob, DataflowFabric, MLExperiment, etc.)|
|JobType|Jobs by Job Type|Breakdown by specific job type (Data Pipeline, RunNotebook, TableMaintenance, SqlAnalyticsEndpoint, Refresh, etc.)|
|JobInvokeType|Scheduled vs On-Demand|Breakdown of jobs by trigger type: "Scheduled" (time-based) or "OnDemand" (manually triggered)|
|ItemName, JobStatus|Failure Rate by Item|Per-item failure percentage. Calculated as (Failed jobs for item / Total jobs for item) * 100|
|ItemKind, JobStatus|Failure Rate by Item Kind|Failure percentage aggregated by item type|
|ExecutingPrincipalId|Jobs by User|Distribution of jobs by executing principal (user or service principal who triggered the job)|
|ExecutingPrincipalType|Jobs by Principal Type|Breakdown by "User" vs "ServicePrincipal"|
|JobStartTime, JobEndTime|Job Duration Trend|Time series of job execution times to identify performance trends|
|Timestamp|Job Volume Over Time|Count of job executions over time, useful for capacity planning and identifying usage patterns|
|WorkspaceName|Jobs by Workspace|Distribution of jobs across different workspaces (for multi-workspace monitoring scenarios)|

#### Supported ItemKind and JobType Combinations

The ItemJobEventLogs table captures events from multiple Fabric item types. Common combinations include:

|Item Kind|Job Types|Use Case|
|--|--|--|
|Pipeline|Data Pipeline|Data pipeline orchestration executions|
|Notebook|RunNotebook, RunNotebookInteractive, PipelineRunNotebook|Notebook job executions (scheduled, interactive, or called from pipelines)|
|Lakehouse|TableMaintenance, TableLoad, LakehouseOperation, LivyBatch, LivySession|Lakehouse operations including table loads, maintenance jobs, and Spark sessions|
|Warehouse|DatamartBatch, SqlAnalyticsEndpoint|Warehouse batch operations and SQL endpoint queries|
|CopyJob|CopyJob|Data copy/movement operations|
|DataflowFabric|Refresh, Publish|Dataflow refresh and publish operations|
|MLExperiment|MLExperimentRun|Machine learning experiment runs|

#### Advanced Reliability & Dependency Metrics

**Reliability & SLA Metrics:**

|Source Column|Metric Name|Description|
|--|--|--|
|JobStatus|Overall Availability %|Percentage of successful jobs across all item types. SLA target: 99%|
|JobStatus|Items Below SLA|Count of items with availability < 99%|
|JobStatus, Timestamp|MTBF (Mean Time Between Failures)|Average time between consecutive failures for an item. Higher is better.|
|JobStartTime, JobEndTime|MTTR (Mean Time to Recovery)|Average time from job start to end for failed jobs. Lower is better.|
|JobStatus|Error Budget|Remaining failure allowance to maintain 99% SLA target|
|JobStatus|SLA Health Status|Overall platform health: Excellent (>99.5%), Good (99-99.5%), Warning (97-99%), Critical (<97%)|
|Timestamp|Availability Trend|Time series of platform availability to detect degradation|

**Dependency & Cascade Metrics:**

|Source Column|Metric Name|Description|
|--|--|--|
|ItemName, Timestamp|Co-Failure Count|Number of times two items fail within the same 5-minute window, indicating possible dependencies|
|Timestamp, JobStatus|Cascade Events|Time windows with 3+ failures within 15 minutes, suggesting cascading failures from a single root cause|
|ItemName|Blast Radius|Maximum number of downstream items affected when a critical item fails|
|ItemName|Impact Score|Calculated as FailureCount × UniqueTimeWindows. High scores indicate items that fail frequently across different time periods|
|ItemName|Most Impactful Failure|Identifies which single item's failure caused the most downstream co-failures|
|Timestamp|Failure Propagation|Time series showing how failures spread through the workspace during cascade events|
|ItemName pairs|Co-Failure Pairs|Top item pairs that consistently fail together, revealing hidden dependencies|
|ItemName|Critical Dependency Paths|Items with highest failure impact based on co-failure analysis|

**Use Cases:**

**Cascade Detection:** When multiple items fail within a short time window, the Dependency Analysis page helps identify: (1) Which item failed first (likely root cause), (2) Which items failed as a consequence, (3) The blast radius of the initial failure, (4) Whether this is a recurring pattern.

**Dependency Mapping:** Co-failure analysis reveals hidden dependencies not visible in code. Example: If "Notebook_ProcessOrders" and "Pipeline_OrderETL" always fail together, there's likely a dependency even if not explicitly defined. Use this to improve error handling or add explicit dependency declarations.

**Reliability Improvement:** The Impact Score helps prioritize which items to fix first. An item with 50 failures across 10 different hours has higher impact (score: 500) than an item with 100 failures all in one hour (score: 100), because the former affects more scenarios.

----------------

## Other helpful resources

##### Microsoft Fabric features
- [Workspace monitoring overview](https://learn.microsoft.com/en-us/fabric/fundamentals/workspace-monitoring-overview)
- [Enable workspace monitoring](https://learn.microsoft.com/en-us/fabric/fundamentals/enable-workspace-monitoring)

##### Workspace Monitoring Templates
- [Documentation - Real-Time Dashboard template for Workspace Monitoring](/monitoring/workspace-monitoring-dashboards/documentation/Workspace_Monitoring_RTI_Dashboard.md)
- [Documentation - Power BI template for Workspace Monitoring](/monitoring/workspace-monitoring-dashboards/documentation/Workspace_Monitoring_PBI_Report.md)

##### Some other Fabric Toolbox assets
- [Overview - Fabric Cost Analysis](/monitoring/fabric-cost-analysis/README.md)
- [Overview - FUAM solution accelerator for tenant level monitoring](/monitoring/fabric-unified-admin-monitoring/README.md)
- [Overview - Semantic Model MCP Server](https://github.com/microsoft/fabric-toolbox/tree/main/tools/SemanticModelMCPServer)
- [Overview - Semantic Model Audit tool](/tools/SemanticModelAudit/README.md)

##### Semantic Link & Semantic Link Lab
- [What is semantic link?](https://learn.microsoft.com/en-us/fabric/data-science/semantic-link-overview)
- [Overview - Semantic Link Labs](https://github.com/microsoft/semantic-link-labs/blob/main/README.md)

----------------



