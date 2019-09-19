Statistics and Indexes

# Optimize statistics and indexes

## Statistic Accuracy and Impact on Query Plans/Performance
- A column's *cardinality* is the number of rows containing a specific value.  For a unique value, the cardinality is 1.  
- Selectivity
 
To see statistics, use
```sql
DBCC SHOW_STATISTICS(<table-name>, <index-name>)
```
This shows
- Metadata about statistics
    - Last update date/time
    - Number of rows in table
    - Number of rows sampled
    - Number of steps in histogram
    - Index Density
    - Avg key length
    - Whether index contains string summary statistics
    - Filtered predicate if applicable
    - Number of non-filtered rows
- Index densities for each combination of columns in the index
- Histogram of up to 200 sample values in the first key column
    - `RANGE_HI_KEY` - Histogram sample values
    - `RANGE_ROWS` - Number of rows between previous and current sample value
    - `EQ_ROWS` - Number of rows equal to current sample
    - `DISTINCT_RANGE_ROWS` - Number of distinct values between current step and previous step
    - `AVG_RANGE_ROWS` - Average number of rows for each distinct value within the range

When SQL Server generates a execution plan, it uses the histogram to estimate number of rows that match a where clause for a single constant expression, or the densities when the expression uses two or more columns.  Out of date statistics can cause the engine to generate a suboptimal plan(e.g. using an index seek rather than a scan if the statistics indicate a smaller number of rows than actually exist)

### Automatic statistic updates
- Statistics are generated when you add an index to a table that contains data, or run the `UPDATE STATISTICS` command
- `ALTER DATABASE SET AUTO_UPDATE_STATISTICS <ON/OFF>` - Updates statistics as needed by using a counter on modifications to column values.  Counter is incremented when rows are inserted or deleted, or when an indexed column is updated.  Reset to 0 when statistics are generated.  When stats are updated, it acquires compile locks and statements may require recompilation.
- `ALTER DATABASE SET AUTO_UPDATE_STATISTICS_ASYNC <ON/OFF>` - Updates statistics in a background thread so as not block query execution.  Optimizer may choose sub-optimal plan until statistics are updated.
- `ALTER DATABASE SET AUTO_CREATE_STATISTICS <ON/OFF>` - SQL Server creates statistics on individual columns in query predicates during query execution.

Stat update thresholds
- Empty table - One or more rows added to an empty table
- Table with fewer than 500 rows - When more than 500 rows are added 
- Table with more than 500 rows - When the number of rows added exceeds a dynamic percentage of total rows( percentage decreases as number of total table rows increases)


Use `sys.stats` catalog view to see if statistic were auto created:
```sql
SELECT 
    OBJECT_NAME(object_id) AS ObjectName,
	name, 
	auto_created
FROM sys.stats
WHERE auto_created = 1 AND
    object_id IN 
        (SELECT object_id FROM sys.objects WHERE type = 'U');
```

## Design statistics maintenance tasks
SQL Server automatically creates and udpates stats for all indexes and columns used in a `WHERE` or `JOIN ON` clause.  In cases where these updates are
- Too frequent, so that they degrade performance
- Too infrequent for a table subject to high-volume data changes

In these cases, you can disable automatic statistics options, and implement a maintenance plan.  (See https://docs.microsoft.com/en-us/sql/relational-databases/maintenance-plans/maintenance-plans?view=sql-server-2017).  This uses SQL Server Agent to run a scheduled maintenance job.  The maintenance plan wizard contains an Update Statistics Task contains the following options
- Databases (note tempdb is never included)
    - All
    - System
    - User
    - Specific Databases
- Updates
    - All existing statistics
    - Column stats only
    - Index stats only
- Scan type
    - Full Scan
    - Sample By

This generates a script that uses `UPDATE STATISTICS`, e.g.
```sql
USE WideWorldImporters;
GO
UPDATE STATISTICS [Application].[Cities] 
WITH FULLSCAN
GO
```
See https://docs.microsoft.com/en-us/sql/t-sql/statements/update-statistics-transact-sql?view=sql-server-2017 for more info.

## Use DMVs to review current index usage and identify missing indexes
Indexes should be reviewed periodically to ensure that they are still useful, or whether any are ignored or missing.

### Current Index Usage
- `sys.dm_db_index_usage_stats` - review use of indexes to resolve queries
- `sys.dm_db_index_physical_stats` - Check overall status of indexes in the DB

Review current index usage:
```sql
SELECT 
    OBJECT_NAME(ixu.object_id, DB_ID('WideWorldImporters')) AS [object_name] ,
    ix.[name] AS index_name ,
    ixu.user_seeks + ixu.user_scans + ixu.user_lookups AS user_reads,
	ixu.user_updates AS user_writes
FROM sys.dm_db_index_usage_stats ixu
INNER JOIN WideWorldImporters.sys.indexes ix ON 
    ixu.[object_id] = ix.[object_id] AND 
	ixu.index_id = ix.index_id 
WHERE ixu.database_id = DB_ID('WideWorldImporters')
ORDER BY user_reads DESC;
```

Find unused indexes:
```sql
SELECT 
    OBJECT_NAME(ix.object_id) AS ObjectName ,
    ix.name
FROM sys.indexes AS ix
INNER JOIN sys.objects AS o ON 
    ix.object_id = o.object_id
WHERE ix.index_id NOT IN ( 
    SELECT ixu.index_id
    FROM sys.dm_db_index_usage_stats AS ixu
    WHERE 
	    ixu.object_id = ix.object_id AND 
		ixu.index_id = ix.index_id AND 
		database_id = DB_ID() 
	) AND 
	o.[type] = 'U'
ORDER BY OBJECT_NAME(ix.object_id) ASC ;
```

Indexes that are updated but never used
```sql
SELECT 
    o.name AS ObjectName ,
    ix.name AS IndexName ,
    ixu.user_seeks + ixu.user_scans + ixu.user_lookups AS user_reads ,
    ixu.user_updates AS user_writes ,
    SUM(p.rows) AS total_rows
FROM sys.dm_db_index_usage_stats ixu
INNER JOIN sys.indexes ix ON 
    ixu.object_id = ix.object_id AND 
	ixu.index_id = ix.index_id
INNER JOIN sys.partitions p ON 
    ixu.object_id = p.object_id AND 
	ixu.index_id = p.index_id
INNER JOIN sys.objects o ON 
    ixu.object_id = o.object_id
WHERE 
    ixu.database_id = DB_ID() AND 
	OBJECTPROPERTY(ixu.object_id, 'IsUserTable') = 1 AND 
	ixu.index_id > 0
GROUP BY 
 o.name ,
 ix.name ,
 ixu.user_seeks + ixu.user_scans + ixu.user_lookups ,
 ixu.user_updates
HAVING ixu.user_seeks + ixu.user_scans + ixu.user_lookups = 0
ORDER BY ixu.user_updates DESC,
 o.name ,
 ix.name ;
```

Review index fragmentation.  Focus on indexes with fragmentation > 15% and page count > 500. When fragmentation is between 15 and 30 percent, reorganize the index and when greater than 30 percent, rebuild the index.
```sql
DECLARE  @db_id SMALLINT, @object_id INT;
SET @db_id = DB_ID(N'WideWorldImporters');
SET @object_id = OBJECT_ID(N'WideWorldImporters.Sales.Orders');

SELECT
   ixs.index_id AS idx_id,
   ix.name AS ObjectName, 
   index_type_desc, 
   page_count AS pg_ct, 
   avg_page_space_used_in_percent AS AvgPageSpacePct, 
   fragment_count AS frag_ct,
   avg_fragmentation_in_percent AS AvgFragPct
FROM sys.dm_db_index_physical_stats 
    (@db_id, @object_id, NULL, NULL , 'Detailed') ixs
INNER JOIN sys.indexes ix ON 
    ixs.index_id = ix.index_id AND 
	ixs.object_id = ix.object_id
ORDER BY avg_fragmentation_in_percent DESC;
```

### Missing indexes
When the optimizer creates an execution plan, it tracks up to 500 indexes that could have been used.  These can be reviewed using
- `sys.dm_db_missing_index_details` - IDentify columns used for equality and inequality predicates
- `sys.dm_db_missing_index_groups` - Cross-reference between details and group stats
- `sys.dm_db_missing_index_group_stats` - Retrieve metrics on groups of missing indexes

```tsql
SELECT
    (user_seeks + user_scans) * avg_total_user_cost * (avg_user_impact * 0.01) AS IndexImprovement,
    id.statement,
    id.equality_columns,
    id.inequality_columns,
    id.included_columns
FROM sys.dm_db_missing_index_group_stats AS igs
INNER JOIN sys.dm_db_missing_index_groups AS ig
    ON igs.group_handle = ig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details AS id
    ON ig.index_handle = id.index_handle
ORDER BY IndexImprovement DESC;
```

## Consolidate overlapping indexes
When two indexes are similar, e.g. one includes an additional column, consider consolidating the indexes in order to reduce maintenance overhead
```sql
WITH IndexColumns AS (
    SELECT 
	    '[' + s.Name + '].[' + T.Name + ']' AS TableName,
        ix.name AS IndexName,  
        c.name AS ColumnName, 
        ix.index_id,
        ixc.index_column_id,
        COUNT(*) OVER(PARTITION BY t.OBJECT_ID, ix.index_id) AS ColumnCount
    FROM sys.schemas AS s
    INNER JOIN sys.tables AS t ON 
	    t.schema_id = s.schema_id
    INNER JOIN sys.indexes AS ix ON 
	    ix.OBJECT_ID = t.OBJECT_ID
    INNER JOIN sys.index_columns AS ixc ON  
	    ixc.OBJECT_ID = ix.OBJECT_ID AND 
		ixc.index_id = ix.index_id
    INNER JOIN sys.columns AS c ON  
	    c.OBJECT_ID = ixc.OBJECT_ID AND 
		c.column_id = ixc.column_id
    WHERE 
        ixc.is_included_column = 0 AND
        LEFT(ix.name, 2) NOT IN ('PK', 'UQ', 'FK')
)
SELECT DISTINCT 
    ix1.TableName, 
	ix1.IndexName AS Index1, 
	ix2.IndexName AS Index2
FROM IndexColumns AS ix1
INNER JOIN IndexColumns AS ix2 ON 
    ix1.TableName = ix2.TableName AND 
	ix1.IndexName <> ix2.IndexName AND 
	ix1.index_column_id = ix2.index_column_id AND  
	ix1.ColumnName = ix2.ColumnName AND 
	ix1.index_column_id < 3 AND 
	ix1.index_id < ix2.index_id AND 
	ix1.ColumnCount <= ix2.ColumnCount
ORDER BY ix1.TableName, ix2.IndexName; 
```
