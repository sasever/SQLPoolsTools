# Synapse SQLPoolsTools Automated Health Check

Health checking and performance tuning in an Azure Synapse SQL Pools Gen 2 environment involves multiple observations.

1.  Understanding the scheduled ETL/ELT workflows and dependencies
1.  Understanding the scheduled reporting workflows and dependencies
1.  Understanding the Ad-Hoc query reporting patterns 
1.  Understanding whether the tables are  storing the data in a healthy way. 
 
 The Automated health check script addresses the 4th observation and provides guidance in how to solve issues founded in the tables in alignment with possible remarks on first three observation.

 
You can reach the store procedure from here:

[sp_health_check_report.sql](./sp_health_check_report.sql)

## Health Check Sources

To perform the analysis below Dynamic Management or Catalog Views (DMVs) from Azure Synapse SQL Pools Gen 2 are used:

* [SYS.SCHEMAS](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/schemas-catalog-views-sys-schemas?view=sql-server-ver15)
* [SYS.TABLES](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-tables-transact-sql?view=sql-server-ver15)
* [SYS.ALL_COLUMNS](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-all-columns-transact-sql?view=sql-server-ver15)
* [SYS.COLUMNS](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-columns-transact-sql?view=sql-server-ver15)
* [SYS.STATS](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-stats-transact-sql?view=sql-server-ver15)
* [SYS.TYPES](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-types-transact-sql?view=sql-server-ver15)
* [SYS.PDW_TABLE_MAPPINGS](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-pdw-table-mappings-transact-sql?view=aps-pdw-2016-au7)
* [SYS.PDW_DISTRIBUTIONS](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-pdw-distributions-transact-sql?view=aps-pdw-2016-au7)
* [SYS.DM_PDW_NODES](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-pdw-nodes-transact-sql?view=aps-pdw-2016-au7)
* [SYS.PDW_NODES_TABLES](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-pdw-nodes-tables-transact-sql?view=aps-pdw-2016-au7)
* [SYS.DM_PDW_NODES_DB_COLUMN_STORE_ROW_GROUP_PHYSICAL_STATS](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-column-store-row-group-physical-stats-transact-sql?view=sql-server-ver15)
* [SYS.DM_PDW_NODES_DB_PARTITION_STATS](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-partition-stats-transact-sql?view=sql-server-ver15)
* [SYS.PDW_TABLE_DISTRIBUTION_PROPERTIES](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-pdw-table-distribution-properties-transact-sql?view=aps-pdw-2016-au7)
* [SYS.PDW_COLUMN_DISTRIBUTION_PROPERTIES](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-pdw-column-distribution-properties-transact-sql?view=aps-pdw-2016-au7)
* [SYS.PARTITIONS](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-partitions-transact-sql?view=sql-server-ver15)
* [SYS.INDEXES](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-indexes-transact-sql?view=sql-server-ver15)

## Health Check Artifacts

 The DMVs listed on the [**Health Check Sources**](./ReadMe.md#health-check-sources) section are used to calculate **7** tables containing several different metrics to identify a tables' health condition. From these 7 tables,
 the process builds first a BASE table to merge all calculated info, then calculates, Scores, Flags, Importance Score and Verbal Report.
 
1. **HC_TABLE_PATTERN :** Contains column type and size statistics per table 
1. **HC_COLUMN_STORE_DENSITY :** Contains general statistics on the types and numbers of row groups and row counts of the CLUSTERED COLUMNSTORE INDEX tables.
1. **HC_GENERAL_ROWGROUP_HEALTH :** Contains trimming metrics of the CLUSTERED COLUMNSTORE INDEX tables.
1. **HC_DISTRIBUTION_LAYOUT :** Contains distribution policy related information adn table specs like partitioning, being external, index/storage type and number of rows
1. **HC_DISTRIBUTION_SKEW_INFO:** Contains MAX to MIN/AVG/MEDIAN number of row containing distribution's skew information
1. **HC_TABLE_STATS_INFO :** Contains info around statistics age and count
1. **HC_BASE :** Merged version of above tables on object_id
1. **HC_SCORES_FLAGS :** calculates Scores and Flags to identify the health
    + All calculated scores are between 1 and 0. 
    + 1 represents the worst condition for the scored property/behavior
    + 0 represents the best condition for the scored property/behavior
    + Some of the ratios are subtracted from 1 to fit into above conditions
1. **HC_WITH_IMPORTANCE :** calculates Importance Score, which is used to mark the tables which needs more/immediate attention.
    + The scores and reverted versions of flags are be summed to calculate importance score on the next step.
    + Importance Score Range is:  Max: 12 Min:0
    + Importance Score should be observed as the less the merrier
1. **HC_REPORT :** Produces a Health Check Report per table with verbal description of the table's condition. Uses all calculated Scores,Flags and other report metrics to build the suggestions.

## Execution of the Health Check 
Heath Check store procedure has three execution mods, ***FULL, SCHEMA, TABLE*** controlled by **@run_type** input parameter
   * **FULL** Performs a Full Health Check on  all existing Table objects in SYS.TABLES DMV.
   * **SCHEMA** Performs a Full Health Check on all existing Table objects in a given SCHEMA
   * **TABLE** Performs a  Health Check on given Table object

You can control whether you want a new REPORT table to be created for the execution or use an existing one by **report_type** input parameter
   * **CTAS** Creating new report table on every execution, It does not clean up the previously created tables on another date.
   * **INSERT** Inserts to an existing REPORT table on every execution.

You can control whether you want the staging tables created for the report calculation process to be dropped or not by **stage_cleanse_type** input parameter
   * **DROP** Drops the staging tables after the execution 
   * **KEEP** Keeps the staging tables after the execution 

The Input Parameters that the procedure accepts are as follows:
* **run_type :** accepted values: FULL,SCHEMA,TABLE
* **report_type :**  accepted values: CTAS,INSERT
* **stage_cleanse_type :** accepted values: DROP,KEEP
* **op_schema_name :** schema name that will contain created tables for the report calculation
* **hc_schema_name :** schema name that will be scanned for report, used only for SCHEMA and TABLE run type
* **hc_table_name :** table name that will be scanned for report, used only for  TABLE run type

To call the procedure in the **FULL** execution mode:

```sql
DECLARE @run_type varchar(10) ='FULL' /*accepted values: FULL,SCHEMA,TABLE*/
DECLARE @report_type [varchar](10) = 'INSERT' /*accepted values: CTAS,INSERT*/
DECLARE @stage_cleanse_type [varchar](10) = 'DROP'/*accepted values: DROP,KEEP*/
DECLARE @op_schema_name varchar(100) = 'dbo'

EXEC dbo.[sp_health_check_report]  @run_type, @report_type, @stage_cleanse_type, @op_schema_name, NULL, NULL
```
When the sp is called with **FULL** run type and **CTAS** report type, all the table artifacts are created with a **FULL_yyyyMMdd** suffix.
When the sp is called with **FULL** run type and **INSERT** report type, staging the table artifacts are created with a **FULL_yyyyMMdd** suffix, final report table, if it does not exists or it is the first execution, will be created as **HC_REPORT**, and next executions with INSERT will insert to this table.

To call the procedure in the **SCHEMA** execution mode:

```sql
DECLARE @run_type varchar(10) ='SCHEMA' /*accepted values: FULL,SCHEMA,TABLE*/
DECLARE @report_type [varchar](10) = 'CTAS' /*accepted values: CTAS,INSERT*/
DECLARE @stage_cleanse_type [varchar](10) = 'DROP'/*accepted values: DROP,KEEP*/
DECLARE @op_schema_name varchar(100) = 'dbo'
DECLARE @hc_schema_name varchar(100) = 'my_non_performing_schema'

EXEC dbo.[sp_health_check_report]  @run_type, @report_type, @stage_cleanse_type, @op_schema_name, @hc_schema_name, NULL
```
When the sp is called with **SCHEMA** execution mode, all the table artifacts are created with a **SCHEMA_yyyyMMdd** suffix.

To call the procedure in the **TABLE** execution mode:

```sql
DECLARE @run_type varchar(10) ='TABLE' /*accepted values: FULL,SCHEMA,TABLE*/
DECLARE @report_type [varchar](10) = 'CTAS' /*accepted values: CTAS,INSERT*/
DECLARE @stage_cleanse_type [varchar](10) = 'DROP'/*accepted values: DROP,KEEP*/
DECLARE @op_schema_name varchar(100) = 'dbo'
DECLARE @hc_schema_name varchar(100) = 'my_non_performing_tables_schema'
DECLARE @hc_table_name varchar(1000) = 'my_nonperforming_table'

EXEC dbo.sp_health_check_report  @run_type, @report_type, @stage_cleanse_type, @op_schema_name, @hc_schema_name, @hc_table_name
```
When the sp is called with **TABLE** execution mode, all the table artifacts are created with a **_TABLE_yyyyMMdd** suffix.

**Important Note:** The SP does not handle the cleanup/housekeeping of the report artifacts.
you can add a block to clean up all artifacts except the HC_REPORT tables to the procedure, or add an additional clean up methodology.
