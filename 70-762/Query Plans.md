Query Plans

# Analyze and troubleshoot query plans
## Resources
- https://www.red-gate.com/simple-talk/sql/performance/execution-plan-basics/
- https://www.red-gate.com/products/dba/sql-monitor/entrypage/execution-plans
- https://docs.microsoft.com/en-us/sql/relational-databases/showplan-logical-and-physical-operators-reference?view=sql-server-2017
- https://docs.microsoft.com/en-us/sql/relational-databases/query-processing-architecture-guide?view=sql-server-2017
- https://docs.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store?view=sql-server-2017
- https://bertwagner.com/2019/08/27/how-i-troubleshoot-sql-server-execution-plans/
- https://sqlserverfast.com/epr/

## Query Processing
1. Parsing - checks syntax, outputs a parse tree that represents the logical steps necessary to process the query
2. Algebrizer 
    - resolves names of objects, tables, and columns referenced in the query.
    - Identifies all the types of the objects referenced 
    - Determines the location of aggregates in the query(i.e. *aggregate binding*).
    - Outputs a binary called the **query processor tree**, which is passed to the query optimizer
3. Optimizer
    - Uses column and index statistics to estimate the best(i.e. lowest cost) execution plan for the query
    - Evaluates different plans, looking for the one with lowest cost in terms of CPU and I/O 
    - For simple queries, it may produce a trivial plan
    - Given an infinite amount of time, and complete, up-to-date statistics, the optimizer will produce *the* optimal plan.  However, it attempts to find the best plan in the least amount of time possible based on the statistics that are available.
    - Execution plans are stored in the plan cache for reuse
4. Query exection
    - Performed by the storage engine
    - Estimated plan is subject to change during actual execution, e.g. if the statistics were out of date
    - The Actual Execution plan shows what actually happened during query processing
    
    
### Plan cache
Compiled query plans are cached for reuse.  To see cached query plans, use the following

```sql
SELECT [cp].[refcounts] 
, [cp].[usecounts] 
, [cp].[objtype] 
, [st].[dbid] 
, [st].[objectid] 
, [st].[text] 
, [qp].[query_plan] 
FROM sys.dm_exec_cached_plans cp 
CROSS APPLY sys.dm_exec_sql_text ( cp.plan_handle ) st 
CROSS APPLY sys.dm_exec_query_plan ( cp.plan_handle ) qp ;
```

## Capturing query plans
- Interactive method
- Using Extended Events
- Using SQL Trace
    - Server Side
    - Client Side

## Analyze graphical query plan
Check for the following
- Query Plan Optimization
    - Right click first operator, and select properties.  Look for 'Reason for early termination of statement optimization' 
- Operators
- Arrow width - indicates how many records were processed by the operator
- Operator cost - look for operations with highest cost
- Warnings

Operators to watch out for
- Table Scan - reading from a heap, row by row.  Can perform slowly when run against a large table
- Clustered index scan - scanning a clustered index.  Index seeks are much better
    - Use of a where clause that can be used with the index  
- Non clustered index seek plus key(or RID) lookup - When a non-clustered index is used, but the engine needs to look up the original row to read column values in WHERE, SELECT, or JOIN clauses.  Consider modifying the index to cover more attributes
- Sort - used when ordering results by a column that does is not included in a clustered index
    - Consider adding sort columns to a clustered index, or filtering the query to reduce number of records to sort.
    - Also look at estimated number of rows and memory grants.  If estimate is off, the engine will spill over to tempDB
- Hash Match (Aggegate)
- Hash Match (Inner Join)

### Using the Query Store
Stores historical information about query plans used.  Can be used to force plans

## Actual vs Estimated plans

## Azure SQL Database Query Performance Insight

