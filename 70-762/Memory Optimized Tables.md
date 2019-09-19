Memory Optimized Tables

# Memory optimized tables and natively stored procedures
 
A memory optimized table is a highly optimized data structure that sql server uses to store data completely in memory without paging to disk.  The table has one representation in active memory, and secondary copy on disk, whcih is used for recovery after shutdown and restart of the server or database 

## Use cases for memory optimized tables 
- High data ingestion rate
    - High number of inserts, either in steady stream or burst
    - Bottlenecks from locking and latching
    - Last-page contention when many threads attempt to access the same page in a B-Tree
    - Example scenarios are IOT, financial trading, manufacturing, telemetry, and security monitoring
- High volume, high performance reads
    - High number of concurrent read requests with periodic inserts and updates),
    - Overheads from parsing, compiling, and optimizing statements 
    - Example scenarios are retail, online banking, and online gaming
    - Memory optimized tables can reduce contention, while natively compiled sprocs enable faster code execution
- Complex business logic in stored procedures
    - Application requires intensive data processing and performs high volume inserts, updates, and deletes. 
    - Supply chains, online catalogs
- Real-time data access 
    - Applications where minimal latency is a priority, e.g. online gaming and financial trading. 
- Session state managements
    - Storage of session data, e.g. in Web applications
    - Frequent reads and writes on a small amount of data
    - Multiple servers may attempt to read/update data for the same session
- Heavy reliance on temp tables, table variables, and table valued parameters
    - Temp tables, table variables, and TVPs all require read/write operations on tempdb, which may incur IO overhead
    - Memory optimized temp-tables, table variables, and TVPs can be used to eliminate tempDB contention and IO overhead
- ETL operations
    - Often require staging tablesas a starting point, or for intermediate steps. 
    - Can experience overhead due to IO operations and query processing
    - Memory optimized tables, and non-durable tables can be used to eliminate these overheads
    - 
## Creating in-memory tables
Memory optimized tables require SQL Server 2016 Enterprise, Developer, or Evaluation versions.  Maximum sise is 2 TB.  When a memory-optimized table is created, SQL Server generates C source and a DLL files that are used to execute DML statements against the table.

Create a database, defining the filegroup as `CONTAINS MEMORY_OPTIMIZED_DATA`

```sql
CREATE DATABASE Exam762_IMOLTP 
ON PRIMARY 
(    
    NAME = Exam762_IMOLTP_data,
    FILENAME = 'C:\data\ExamBook762Ch3_IMOLTP.mdf', size=500MB 
),  
FILEGROUP Exam762_IMOLTP_FG CONTAINS MEMORY_OPTIMIZED_DATA 
(      
    NAME = Exam762_IMOLTP_FG_Container,      
    FILENAME = 'C:\data\ExamBook762Ch3_IMOLTP_FG_Container' 
)  
LOG ON 
(     
    NAME = Exam762_IMOLTP_log,      
    FILENAME = 'C:\data\ExamBook762Ch3_IMOLTP_log.ldf', size=500MB 
);  
GO
```
Create a table using the `WITH (MEMORY_OPTIMIZED = ON)` option at the end:
```sql
USE Exam762_IMOLTP; 
GO  

CREATE SCHEMA Examples; 
GO 

CREATE TABLE Examples.Order_Disk (
    OrderId INT NOT NULL PRIMARY KEY NONCLUSTERED,     
    OrderDate DATETIME NOT NULL,     
    CustomerCode NVARCHAR(5) NOT NULL 
); 
GO   

CREATE TABLE Examples.Order_IM (     
    OrderID INT NOT NULL PRIMARY KEY NONCLUSTERED,     
    OrderDate DATETIME NOT NULL,     
    CustomerCode NVARCHAR(5) NOT NULL 
) WITH (MEMORY_OPTIMIZED = ON); 
GO  
```


## Optimize performance of in-memory tables 
### Natively compiled stored procedure 
A Procedure compiled at the time of creation(rather than at first execution with standard sprocs).  These are translated into C code, then into machine code that is stored in a dynamic link library(DLL) in the default data directory.  Natively compiled sprocs can only access memory-optimized tables.  To create a natively-compiled sproc, you must use the `WITH NATIVE_COMPILATION` and `SCHEMABINDING` options.
```
CREATE PROCEDURE Examples.OrderInsert_NC      
    @OrderID INT,      
    @CustomerCode NVARCHAR(10)  
WITH NATIVE_COMPILATION, SCHEMABINDING   
AS    
BEGIN ATOMIC  
WITH (TRANSACTION ISOLATION LEVEL = SNAPSHOT, LANGUAGE = N'English')       
    DECLARE @OrderDate DATETIME = getdate();       
    INSERT INTO Examples.Order_IM (OrderId, OrderDate, CustomerCode)      
    VALUES (@OrderID, @OrderDate, @CustomerCode);   
END;
GO
```
Transactions are supported using the `BEGIN ATOMIC` block.  This reates an atomic block, which is a block of T-SQL statements that succeed or fail together. Starting an atomic block creates a transaction if one does not het exist or creates a savepoint if one does exist.  Must include options defining the isolation level and language, e.g. 
`BEGIN ATOMIC WITH(TRANSACTION ISOLATION LEVEL = SNAPSHOT, LANGUAGE = N’English’)`

Limitations of natively compiled sprocs 
- No Tembdb access
    -Instead you can create non-durable memory optimized tables or memory optimized types or table variables 
- No Cursors
    - Use set based logic or while loops 
- No Case statement 
    - Use a table variable with a column to filter 
- No Merge statement 
    - Must use explicit INSERT,UPDATE,DELETE
- No select into clause 
    - Use `INSERT INTO <table> SELECT` 
- No `FROM` clause in a `UPDATE` statememts
    - Can use a memory-optimized type and a trigger instead 
- No `PERCENT` or `WITH TIES` in `TOP` clause 
- No `DISTINCT` with aggregation functions 
- No `INTERSECT`, `EXCEPT`, `APPLY`, `PIVOT`, `UNPIVOT`, `LIKE` or `CONTAINS` operators
- No CTEs
- No Multi-row insert statements 
    - Need to split up inserts 
- No `EXECUTE WITH RECOMPILE`
- Cannot reference Views
     

### Indexes 
Memory optimized tables can have up to eight *non-clustered* indexes all of which are covering indexes.  These indexes only exists in memory and do not include data, rather they point to a row in memory.  The following options are supported
- Non clustered Hash
    - Use when you have many queries that perform point lookups, i.e. equi-joins
    - Must specify a bucket count between two and three times the expected number of distinct values in the indexed column.  A higher bucket count uses more memory, but retrieves data more quickly
- Columnstore 
    - Good for analytics, order by, window function, large table scans 
- Nonclustered B-tree 
    - Good for < or >, order by, inequalities
     
### Offload analytics to readable secondary
Good for when you need to support both OLTP and analytics 

### Durability
Memory-optimized tables have two durability options:
- Durable: default option that treats data like disc table
    - `WITH(MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA)`
- Non-durable: Only persists the table schema not data.
    - No logging overhead 
    - Faster writes
    - Appropriate for transient data, e.g. session state mgmt or ETL staging tables
    - `DURABILITY = SCHEMA ONLY`
    

## Determine best case usage scenarios for natively compiled stored procs
- Applications for which obtaining the best possible performance is a requirement 
- Queries that execute frequently 
- Aggregation 
- Nested loop join 
- Multi-statement select,inserts,updates, or deletes 
- Complex expressions 
- Procedural logic such as conditionals and loops 

## Execution stats for natively compiled stored procedures
By default, many statistics are not collected by DMVs such as `sys.dm_exec_query_stats` or `sys.dm_exec_procedure_stats` 

To enable collection of execution stats, use one of the following
- `EXEC Sys.sp_xtp_control_proc_exec_stats` to enable statistics collection at Procedure level
- `EXEC Sys.sp_xtp_control_query_exec_stats` to enable statistics collection at the Query level 

Example usage once these are enabled:
```sql
SELECT
    OBJECT_NAME(PS.object_id) AS obj_name,
    cached_time as cached_tm,
    last_execution_time as last_exec_tm,
    execution_count as ex_cnt,
    total_worker_time as wrkr_tm,
    total_elapsed_time as elpsd_tm 
FROM sys.dm_exec_procedure_stats PS 
INNER JOIN sys.all_sql_modules SM ON SM.object_id = PS.object_id 
WHERE SM.uses_native_compilation = 1;

SELECT     
    st.objectid as obj_id,     
    OBJECT_NAME(st.objectid) AS obj_nm,     
    SUBSTRING(st.text,          
        (QS.statement_start_offset / 2 ) + 1,         
        ((QS.statement_end_offset - QS.statement_start_offset) / 2) + 1)              
        AS 'Query',
    QS.last_execution_time as last_exec_tm,     
    QS.execution_count as ex_cnt 
FROM sys.dm_exec_query_stats QS 
CROSS APPLY sys.dm_exec_sql_text(sql_handle) st 
INNER JOIN sys.all_sql_modules SM ON SM.object_id = st.objectid 
WHERE SM.uses_native_compilation = 1
```

