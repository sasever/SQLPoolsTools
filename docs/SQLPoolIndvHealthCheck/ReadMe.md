# Performance Tuning on Azure Synapse SQL Pools
*Individual Health Check Scrips and Guidance for Manual Health Checking Azure Synapse Sql Pools Tables*

Performance tuning on any data processing platform may easily become rearranging deck chairs on the Titanic, if you do not focus on the correct issues. You need to aim and put effort for the work which brings the most outcome. For example,

* You may have tables in very bad condition, but does not get frequently read or written and has no effect in processing timelines
* You may have tables in very bad condition, and does not get frequently read or written but becomes a bottleneck for multiple subsequent processes.

The effect of which one you will put an effort will obviously be different.

## How to approach Performance Tuning on Azure Synapse SQL Pools.
If you have a poor performing read query or poor performing write operation on Azure Synapse SQL Pools, there might be a few different reasons resulting it. But solutions usually are hidden in how healthily your tables are placed in the system and how you are reading/writing these tables.

Azure Synapse SQL Pools, gains its strength in fast query performance first from the [MPP architecture](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql/overview-architecture) and second from [Compressed ColumnStore index (CCI)](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview?toc=/azure/synapse-analytics/toc.json&bc=/azure/synapse-analytics/breadcrumb/toc.json&view=azure-sqldw-latest&preserve-view=true) storage technology and the usage of [CCI best practices](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql/data-load-columnstore-compression). 

### 1. Analyzing  Read Performance
### 1.1. Checking the Execution
When analyzing read performance for a specific query you should first understand what operation(s) in query processing causes you how much time.
You can get this information by combining two DMV's, [SYS.DM_PDW_EXEC_REQUESTS](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-pdw-exec-requests-transact-sql?view=azure-sqldw-latest) and [SYS.DM_PDW_REQUEST_STEPS](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-pdw-request-steps-transact-sql?view=azure-sqldw-latest).

```sql
with queries as 
(	SELECT  request_id,command,resource_class,[label],[status]
	 FROM    sys.dm_pdw_exec_requests a
	WHERE 1=1 
    /*Filter Area: You can filter with either one
       or multiple of below example filters  */
	  and a.start_time> '2020-10-21 11:00:00.000'
	  and a.request_id in ('QID37052391') /*Sample Query ID*/
      and command like '%MYSCHEMA.MYTABLE%'
	  and resource_class is not null
	  and ISNULL([label],'OTHER')='<MYQUERYLABEL>'
) 
SELECT  qq.resource_class, 
        upper(qq.command) as query_text,
        qq.status main_status,
        st.* 
 FROM sys.dm_pdw_request_steps st, queries qq
WHERE st.request_id = qq.request_id
order by st.request_id desc,st.step_index asc
OPTION (LABEL = 'PLANQUERY')
```
***How to use this query:** an important best practice is always adding unique labels to your ad-hoc or Reporting/ETL Tool generated queries so that it will help you to find the exact query you are looking for. If that is  not the case you can use various filters to access the desired outcome. Keep in mind that command column does only contain first 4000 characters of your query.*

If you can not find the query you are looking for most probably this happens because these DMVs have [capacity limits](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-service-capacity-limits#metadata). You may not be able to access to the exact query you are looking for because of that reason. Therefor using a log analytics workspace to store query logs is a better practice *(The 4000 character limitation is still valid with Command_s column)*. You can use below query to retrieve the same result from log analytics.

```csharp          
AzureDiagnostics 
|where Category == "ExecRequests"  
//Filter Area: You can filter with either one
//       or multiple of below example filters 
   and Label_s == '<MYQUERYLABEL>'
   and StartTime_t > ago(48h)  //Filtering time
   and Command_s contains_cs 'MYSCHEMA.MYTABLE' 
   and RequestId_s in ('QID23966359')  //Sample Query ID
   and Status_s =='Completed'  
| project RequestId_s,MainST=StartTime_t,MainQuery=Command_s,MainStatus=Status_s
|join  (AzureDiagnostics 
        |where Category == "RequestSteps"    
           and StartTime_t > ago(48h) //Filtering time
        |project RequestId_s,StepIndex_d,OperationType_s,LocationType_s,DistributionType_s,RowCount_d,StartTime_t,EndTime_t,Status_s,Command_s
        ) on $left.RequestId_s == $right.RequestId_s
|summarize   MainStatus=min(MainStatus) ,Status_s=min(Status_s) , RowCount_d=max(RowCount_d),EndTime_t=max(EndTime_t),Command_s=max(Command_s) 
 by RequestId_s,MainST,MainQuery,StepIndex_d,OperationType_s,LocationType_s,DistributionType_s,StartTime_t
|project Duration_M=datetime_diff('Second',EndTime_t,StartTime_t)/60.0,RequestId_s,MainST,MainQuery,MainStatus,StepIndex_d,OperationType_s,LocationType_s,DistributionType_s,Status_s,RowCount_d,StartTime_t,EndTime_t,Command_s 
|order by Duration_M,RequestId_s desc,StepIndex_d asc
```

After you identify long running steps, check whether they are one of the below: 

| Operation | Description |
| --- | --- |
| ShuffleMoveOperation | Redistributes data for compatible join or aggregation |
| BroadcastMoveOperation | Table needs to be temporarily replicated for join compatibility |
| RoundRobinMoveOperation | Data gets redistributed as round_robin, whenever target table is round_robin, this operation happens |


If you see one or multiple of above operations this means you need to look at how the tables joined or filtered in the query are distributed. 

###  1.2. Checking the Distribution Layout
Below query can help you in understanding the distribution layout as an overview.

```sql
SELECT
  DISTINCT
  t.object_id,
  s.name AS SchemaName,
  t.name AS TableName,
  t.Is_external,
  tdp.distribution_policy_desc AS distributionPolicyDesc,
  CASE WHEN cdp.distribution_ordinal = 1 THEN c.name ELSE NULL END AS DistCol,
  CASE WHEN cdp.distribution_ordinal = 1 THEN ty.name ELSE NULL END AS DistColDataType,
  CASE WHEN cdp.distribution_ordinal = 1 THEN c.max_length ELSE NULL END DistColDataLength,
  CASE WHEN cdp.distribution_ordinal = 1 THEN c.precision ELSE NULL END DistColDataPrecision,
  CASE WHEN cdp.distribution_ordinal = 1 THEN c.scale ELSE NULL END DistColDataScale,
  i.type_desc AS StorageType,
  CASE WHEN count(p.rows) > 1 THEN 'YES' ELSE 'NO' END AS IsPartitioned,
  count(p.rows) AS NumPartitions,
  sum(p.rows) AS NumRows
FROM
  sys.tables AS t
  INNER JOIN sys.schemas AS s ON t.schema_id = s.schema_id
  INNER JOIN sys.pdw_table_distribution_properties AS tdp ON t.object_id = tdp.object_id
  INNER JOIN sys.columns AS c ON t.object_id = c.object_id
  INNER JOIN sys.types ty ON C.system_type_id = ty.system_type_id
  INNER JOIN sys.pdw_column_distribution_properties AS cdp ON c.object_id = cdp.object_id
  AND c.column_id = cdp.column_id
  INNER JOIN sys.partitions p ON t.object_id = p.object_id
  INNER JOIN sys.indexes i ON t.object_id = i.object_id
WHERE 1=1
  AND (
    tdp.distribution_policy_desc <> 'HASH'
    OR cdp.distribution_ordinal = 1
  )
  AND i.index_ID < 2
  AND is_external = '0'
  AND ty.name != 'sysname'
GROUP BY
    t.object_id,  s.name,  t.name,  tdp.distribution_policy_desc,
    cdp.distribution_ordinal,  c.name,  i.type_desc,  Is_external,  ty.Name
	,c.max_length, c.precision, c.scale
UNION
SELECT
  t.object_id,
  s.name AS SchemaName,
  t.name AS TableName,
  Is_external,
  NULL AS distributionPolicyDesc,
  NULL AS DistColDataType,
  NULL AS DistCol,
  NULL AS DistColDataLength,
  NULL AS DistColDataPrecision,
  NULL AS DistColDataScale,
  'HADOOP' StorageType,
  'NO' AS IsPartitioned,
  0 AS NumPartitions,
  0 AS NumRows
FROM
  sys.tables AS t
  INNER JOIN sys.schemas AS s ON t.schema_id = s.schema_id
WHERE
  Is_External = '1'
```  

***How to evaluate the output of above query:** This query provides you the information about the table distribution lay-out. Check whether one of the tables are external, check whether the two tables getting joined are distributed with the join key, whether the data type, length precision scale are same for the join key. Check whether the tables are round robin distributed, Check the number of rows for each table and see whether you have  less than 60M row tables getting broadcasted that takes too much time. Also StorageType is another important identifier, if your Table has way more than 60M rows and it is not stored as Clustered Columnstore Index, the query performance is normal to be low.*

If everything seems perfect by lay out, then you may either have the data distributed unevenly in the distributions and the session is waiting for the more data containing distributions to finish, or if the data is stored as Clustered Columnstore Index, the Row Group quality may be low.

###  1.3. Checking the Distribution Data Skew
Data being distributed unevenly within distributions is an important problem. If your data is less than 60M it may be normal to have uneven distribution but still may require to be handled, for example by changing the distribution methodology. The bigger your tables get, the more it will be important for the data to be distributed as even as possible. With below script you can check the skew. Script provides information about skew between MAXIMUM number containing distributution and MEDIAN of the distribution row count as DISTR_max_med_skew, between MAXIMUM number containing distributution and AVERAGE of the distribution row count as DISTR_max_avg_skew, between MAXIMUM number containing distributution and MINIMUM of the distribution row count as DISTR_max_min_skew.



```sql
WITH base
AS
(
SELECT 
  s.name                                                               AS  [schemaname]
, t.name                                                               AS  [tablename]
, QUOTENAME(s.name)+'.'+QUOTENAME(t.name)                              AS  [two_part_name]
, nt.[name]                                                            AS  [node_table_name]
, ROW_NUMBER() OVER(PARTITION BY nt.[name] ORDER BY (SELECT NULL))     AS  [node_table_name_seq]
, tp.[distribution_policy_desc]                                        AS  [distribution_policy_name]
, c.[name]                                                             AS  [distribution_column]
, nt.[distribution_id]                                                 AS  [distribution_id]
, i.[index_id]                                                         AS  [index_id]
, i.[type]                                                             AS  [index_type]
, i.[type_desc]                                                        AS  [index_type_desc]
, i.[name]		                                                       AS  [index_name]
, nt.[pdw_node_id]                                                     AS  [pdw_node_id]
, pn.[type]                                                            AS  [pdw_node_type]
, pn.[name]                                                            AS  [pdw_node_name]
, di.name                                                              AS  [dist_name]
, di.position                                                          AS  [dist_position]
, nps.[partition_number]                                               AS  [partition_nmbr]
, nps.[reserved_page_count]                                            AS  [reserved_space_page_count]
, nps.[reserved_page_count] - nps.[used_page_count]                    AS  [unused_space_page_count]
, nps.[in_row_data_page_count] 
    + nps.[row_overflow_used_page_count] 
    + nps.[lob_used_page_count]                                        AS  [data_space_page_count]
, nps.[reserved_page_count] 
 - (nps.[reserved_page_count] - nps.[used_page_count]) 
 - ([in_row_data_page_count] 
         + [row_overflow_used_page_count]+[lob_used_page_count])       AS  [index_space_page_count]
, nps.[row_count]                                                      AS  [approx_row_count]
, t.[object_id]                                                        AS  [object_id]
from 
    sys.schemas s
INNER JOIN sys.tables t
    ON s.[schema_id] = t.[schema_id]
INNER JOIN sys.indexes i
    ON  t.[object_id] = i.[object_id]
INNER JOIN sys.pdw_table_distribution_properties tp
    ON t.[object_id] = tp.[object_id]
INNER JOIN sys.pdw_table_mappings tm
    ON t.[object_id] = tm.[object_id]
INNER JOIN sys.pdw_nodes_tables nt
    ON tm.[physical_name] = nt.[name]
INNER JOIN sys.dm_pdw_nodes pn
    ON  nt.[pdw_node_id] = pn.[pdw_node_id]
	  AND pn.[type] = 'COMPUTE' -
INNER JOIN sys.pdw_distributions di
    ON  nt.[distribution_id] = di.[distribution_id]
INNER JOIN sys.dm_pdw_nodes_db_partition_stats nps
    ON nt.[object_id] = nps.[object_id]
    AND nt.[pdw_node_id] = nps.[pdw_node_id]
    AND i.[index_id] = nps.[index_id]
    AND nt.[distribution_id] = nps.[distribution_id]
LEFT OUTER JOIN (select * from sys.pdw_column_distribution_properties where distribution_ordinal = 1) cdp
    ON t.[object_id] = cdp.[object_id]
LEFT OUTER JOIN sys.columns c
    ON cdp.[object_id] = c.[object_id]
    AND cdp.[column_id] = c.[column_id]
)
, size
AS
(
SELECT
   object_id
,  [schemaname]
,  [tablename]
,  [two_part_name]
,  [node_table_name]
,  [node_table_name_seq]
,  [distribution_policy_name]
,  [distribution_column]
,  [distribution_id]
,  [index_id]
,  [index_type]
,  [index_type_desc]
,  [index_name]
,  [pdw_node_id]
,  [pdw_node_type]
,  [pdw_node_name]
,  [dist_name]
,  [dist_position]
,  [reserved_space_page_count]
,  [unused_space_page_count]
,  [data_space_page_count]
,  [index_space_page_count]
,  [approx_row_count]
 ,PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY [approx_row_count])  OVER (PARTITION BY object_id) AS DISTR_median_row_count
FROM base
)
select s.object_id,s.schemaname, s.tablename, 
	DISTR_max_row_count,DISTR_min_row_count,DISTR_avg_row_count,DISTR_median_row_count,
    case when isnull(DISTR_max_row_count,0)>0 then (DISTR_max_row_count * 1.00 - DISTR_min_row_count * 1.00) /DISTR_max_row_count * 1.00  else -1 end AS DISTR_max_min_skew,
	case when isnull(DISTR_max_row_count,0)>0 then(DISTR_max_row_count * 1.00 - DISTR_avg_row_count * 1.00) / DISTR_max_row_count * 1.00  else -1 end AS DISTR_max_avg_skew,
	case when isnull(DISTR_max_row_count,0)>0 then(DISTR_max_row_count * 1.00 - DISTR_median_row_count * 1.00) / DISTR_max_row_count * 1.00  else -1 end AS DISTR_max_med_skew
 from (
SELECT object_id,schemaname, 
	tablename, 
	DISTR_median_row_count, 
	MAX([approx_row_count]) as DISTR_max_row_count,
	MIN([approx_row_count]) as DISTR_min_row_count,
	AVG([approx_row_count]) as DISTR_avg_row_count
FROM size
WHERE 1=1
  and table_name  in ('MYTABLENAME')
GROUP BY object_id,schemaname, 
		tablename ,DISTR_median_row_count) as s
```
***How to evaluate the output of above query:** If there is a way more than 10% Skew for all three metrics then you should be reconsidering how the tables are distributed. Unless DISTR_max_min_skew is 1, which means that at least 1 distribution doesn't have any rows for this table, DISTR_max_med_skew being high has more priority, because this can cause longer processing times  although more than half of the distributions has already done their jobs*

###  1.4. Checking the Compressed Columnstore Index Row Groups

**1.4.1. Checking the Compressed Columnstore Index Row Group Status**

Below query helps you to understand some essential [Compressed Columnstore Index health criterions](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql/data-load-columnstore-compression). You can find the answers of how many compressed and open row groups the table has, what is the maximum ideal number of compressed row groups.

```sql
SELECT
        t.object_id                                              AS object_id
,       s.name                                                   AS [schemaname]
,       t.name                                                   AS [tablename]
,       COUNT(DISTINCT rg.[partition_number])                    AS [table_partition_count]
,       SUM(rg.[total_rows])                                     AS [row_count_total]
,       SUM(rg.[total_rows])/COUNT(DISTINCT rg.[distribution_id])               AS [row_count_per_distribution_MAX]
,       CEILING    ((SUM(rg.[total_rows])*1.0/COUNT(DISTINCT rg.[distribution_id]))/1048576) AS [rowgroup_per_distribution_MAX_IDEAL]
,       CEILING    (SUM(rg.[total_rows])*1.0/1048576)           AS [rowgroup_count_MAX_IDEAL]
,       SUM(CASE WHEN rg.[State] = 1 THEN 1   ELSE 0    END)    AS [OPEN_rowgroup_count]
,       SUM(CASE WHEN rg.[State] = 1 THEN rg.[total_rows]     ELSE 0    END)    AS [OPEN_rowgroup_rows_TOTAL]
,       AVG(CASE WHEN rg.[State] = 1 THEN rg.[total_rows]     ELSE 0    END)    AS [OPEN_rowgroup_rows_AVG]
,       MIN(CASE WHEN rg.[State] = 1 THEN rg.[total_rows]     ELSE NULL END)    AS [OPEN_rowgroup_rows_MIN]
,       MAX(CASE WHEN rg.[State] = 1 THEN rg.[total_rows]     ELSE NULL END)    AS [OPEN_rowgroup_rows_MAX]
,       AVG(CASE WHEN rg.[State] = 1 THEN rg.[total_rows]     ELSE NULL END)    AS [OPEN_rowgroup_rows_AVG]
,       SUM(CASE WHEN rg.[State] = 3 THEN 1   ELSE 0    END)    AS [COMPRESSED_rowgroup_count]
,       SUM(CASE WHEN rg.[State] = 3 THEN rg.[total_rows]     ELSE 0    END)    AS [COMPRESSED_rowgroup_rows]
,       SUM(CASE WHEN rg.[State] = 3 THEN rg.[deleted_rows]   ELSE 0    END)    AS [COMPRESSED_rowgroup_rows_DELETED]
,       MIN(CASE WHEN rg.[State] = 3 THEN rg.[total_rows]     ELSE NULL END)    AS [COMPRESSED_rowgroup_rows_MIN]
,       MAX(CASE WHEN rg.[State] = 3 THEN rg.[total_rows]     ELSE NULL END)    AS [COMPRESSED_rowgroup_rows_MAX]
,       AVG(CASE WHEN rg.[State] = 3 THEN rg.[total_rows]     ELSE NULL END)    AS [COMPRESSED_rowgroup_rows_AVG]
FROM    sys.[pdw_nodes_column_store_row_groups] rg
JOIN    sys.[pdw_nodes_tables] nt                   ON  rg.[object_id]          = nt.[object_id]
                                                    AND rg.[pdw_node_id]        = nt.[pdw_node_id]
                                                    AND rg.[distribution_id]    = nt.[distribution_id]
JOIN    sys.[pdw_table_mappings] mp                 ON  nt.[name]               = mp.[physical_name]
JOIN    sys.[tables] t                              ON  mp.[object_id]          = t.[object_id]
JOIN    sys.[schemas] s                             ON t.[schema_id]            = s.[schema_id]
GROUP BY t.object_id, s.[name], t.[name];
```


***How to evaluate the output of above query:** Azure Synapse SQL pools by default has 60 distributions, and normally if everything is at in desired state we do not expect having more than 60 Open Row groups. If we have too many open row groups this means we either have singleton insert/update operations or sessions are trying to  insert/update in very small batches (less than 50K per distribution).  Having frequent updates/deletes(COMPRESSED_rowgroup_rows_DELETED) is also not something desired for the compressed Row group health since it will add additional steps to the read operations. In these cases table load pattern should be revised.*

*If COMPRESSED_rowgroup_count is too big comparing to rowgroup_count_MAX_IDEAL, this means there are some health issues in how the row groups are build. For that case the health of row groups should also be checked.*

**1.4.2. Checking the Compressed Columnstore Index Row Group Health**

Ideally an Azure Synapse SQL pools Table, which is stored as Compressed Columnstore Index, we want each compressed row group to have 1 million rows. If I should share an exact number, it should be [1,048,576](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-memory-optimizations-for-columnstore-compression#target-size-for-rowgroups)

When a Compressed row group has less than 1,048,576  rows, it gets marked as 'TRIMMED'. [The following trim reasons indicate  trimming of the rowgroup:](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-memory-optimizations-for-columnstore-compression#how-to-monitor-rowgroup-quality
)

* **BULKLOAD:** Incoming batch of rows for the load for that distribution had less than or not as exact multiples of 1 million and greater than 50K rows. 
* **MEMORY_LIMITATION:** Memory of the loading session is less than the required working memory for compressing 1 million rows of that table.
* **DICTIONARY_SIZE:** There is at least one string column with wide and/or high cardinality strings that causes the maximum dictionary size which is is limited to 16 MB, to get filled before the rowgroup reaches to 1 million rows.


```sql
with base as(
	select
		   tb.object_id
	     , sm.name as schemaname
		 , tb.[name] AS tablename
	     , rg.[row_group_id] AS [row_group_id]
	     , rg.[state] AS [state] 
	     , rg.[state_desc] AS [state_desc]
		 , rg.[total_rows]
	     , rg.[trim_reason_desc] AS trim_reason_desc
	     , mp.[physical_name] AS physical_name
	FROM sys.[schemas] sm
	JOIN sys.[tables] tb ON sm.[schema_id] = tb.[schema_id]
	JOIN sys.[pdw_table_mappings] mp ON tb.[object_id] = mp.[object_id]
	JOIN sys.[pdw_nodes_tables] nt ON nt.[name] = mp.[physical_name]
	JOIN sys.[dm_pdw_nodes_db_column_store_row_group_physical_stats] rg ON rg.[object_id] = nt.[object_id]
	 AND rg.[pdw_node_id] = nt.[pdw_node_id]
	 AND rg.[distribution_id] = nt.[distribution_id]
     AND tb.name = 'MYTABLENAME'
     AND sm.name = 'MYTABLEsSCHEMA'
)
select 
    object_id,
    schemaname,
    tablename,
    sum( case when trim_reason_desc='DICTIONARY_SIZE' then 1 else 0 end)DICTIONARY_SIZE_Trimmed_RG,
    min( case when trim_reason_desc='DICTIONARY_SIZE' then [total_rows] end)DICTIONARY_SIZE_Trimmed_RG_max_size,
    min( case when trim_reason_desc='DICTIONARY_SIZE' then [total_rows] end)DICTIONARY_SIZE_Trimmed_RG_min_size,
    min( case when trim_reason_desc='DICTIONARY_SIZE' then [total_rows] end)DICTIONARY_SIZE_Trimmed_RG_avg_size,
    sum( case when trim_reason_desc='BULKLOAD' then 1 else 0 end)BULKLOAD_Trimmed_RG,
    max( case when trim_reason_desc='BULKLOAD' then [total_rows] end)BULKLOAD_Trimmed_RG_max_size,
    min( case when trim_reason_desc='BULKLOAD' then [total_rows] end)BULKLOAD_Trimmed_RG_min_size,
    avg( case when trim_reason_desc='BULKLOAD' then [total_rows] end)BULKLOAD_Trimmed_RG_avg_size,
    sum( case when trim_reason_desc='MEMORY_LIMITATION' then 1 else 0 end)MEMORY_LIMITATION_Trimmed_RG,
    max( case when trim_reason_desc='MEMORY_LIMITATION' then [total_rows] end)MEMORY_LIMITATION_Trimmed_RG_max_size,
    min( case when trim_reason_desc='MEMORY_LIMITATION' then [total_rows] end)MEMORY_LIMITATION_Trimmed_RG_min_size,
    avg( case when trim_reason_desc='MEMORY_LIMITATION' then [total_rows] end)MEMORY_LIMITATION_Trimmed_RG_avg_size
from base  group by object_id,tablename,schemaname;
```
***How to evaluate the output of above query:** If your row groups are trimmed with one or many of the above reasons, and the AVG number of rows and MIN number of rows per trimmed rowgroup is very low comparing to 1 million
you should check the recommendations [here](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-memory-optimizations-for-columnstore-compression). If BULKLOAD or  MEMORY_LIMITATION are the only or main trimming reasons you may first try to recreate the table or a copy of a table wih a CTAS(CREATE TABLE AS SELECT) statement with a better [resource class](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/resource-classes-for-workload-management) if MEMORY_LIMITATION especially exists. Control the latest health condition and retry the slow query to see the result.* 

*If the main trimming reason is DICTIONARY_SIZE there is  at least one very high cardinality and/or very big sized character or big numeric/date column. Check unnecessarily big columns try to reduce them. You may check whether it is worth the effort by creating a copy of the table without the candidate columns and trying the slow query to see the result.*

You can use below query to check the columns of the table(s) you are interested.

```sql
select s.name schemaname
      ,t.name as tablename
	  ,c.name as columnname
	  ,c.column_id as columnId
	  ,ty.name as typename
      CASE WHEN c.max_length>=30 THEN 'YES' ELSE 'NO' END as tooLongColumn
	  ,c.max_length  
	  ,c.precision
	  ,c.scale
	  ,c.is_nullable
	  ,c.collation_name
from sys.schemas s 
join sys.tables t       on s.schema_id=t.schema_id
join sys.all_columns c  on t.object_id=c.object_id
join [sys].[types] ty   on ty.system_type_id=c.system_type_id
where  t.name = 'MYTABLENAME'
  and  s.name ='MYTABLEsSCHEMANAME'
  and  TY.name != 'sysname'
order by t.name,c.name,c.max_length;
```

The engine will create compressed row groups if there are greater than 50,000 rows being inserted (as opposed to inserting into the delta store) but sets the trim reason to BULKLOAD. In this scenario, consider increasing your batch load to include more rows. Also, reevaluate your partitioning scheme to ensure it's not too granular as row groups can't span partition boundaries. You can check [Partition Implementation Samples](docs/SQLPoolPartitioning/ReadMe.md) for a sample guideline and [Azure Synapse analytics Partitioning Documentation](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-tables-partition) for deeper information


### 1.5. Checking whether the statistics are up to date

Everything about your tables may be perfect, but the Optimizer may not be aware about it and act differently,build an improper plan, if the statistics of your table are not up to date. Creating the correct statistics for different workloads are also as important as having overall statistics. If your tables are used by subsequent processes to produce other outputs/tables during your ETL/ELT, maintaining all necessary statistics after every major change on your tables is a good practice to follow. you can find detailed information on [how to create statistics here](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql/develop-tables-statistics) in Microsoft Azure Synapse documentation.

```sql
WITH STATS
AS
(
SELECT TB.OBJECT_ID,
       sm.[name] AS [schemaname], 
	   tb.[name] AS [tablename], 
	   st.[name] AS [stats_name], 
	   st.[has_filter] AS [stats_is_filtered], 
	   STATS_DATE(st.[object_id], st.[stats_id]) AS [stats_last_updated_date], 
	   st.[user_created] 
FROM sys.objects AS ob
	 JOIN sys.stats AS st ON ob.[object_id] = st.[object_id] 
	 JOIN sys.tables AS tb ON st.[object_id] = tb.[object_id]
	 JOIN sys.schemas AS sm ON tb.[schema_id] = sm.[schema_id]
WHERE 1 = 1 AND 
	  STATS_DATE(st.[object_id], st.[stats_id]) IS NOT NULL )
select OBJECT_ID,[schemaname],[tablename]
	  ,max([stats_last_updated_date]) newest_active_stat_date
	  ,min([stats_last_updated_date]) oldest_active_stat_date
	  ,count(*) number_of_stats 
	  ,SUM(cast([user_created] as int)) NUM_OF_USERCREATED_STATS
	  ,SUM(cast([stats_is_filtered] as int)) NUM_OF_FILTERED_STATS
from STATS
group by  OBJECT_ID,[schemaname],[tablename]
```

These checks can be used all together or individually to examine performance problems. An automated and merged version is provided [here](docs/SQLPoolAutoHealthCheck/ReadMe.md) in this repository.
