Columnstore Indexes

# Columnstore Indexes
## Structure and concepts
- **Row Group** - A group of rows that have been compressed into the column store format at the same time.  Each row group contains up to 1,048,478 rows.
- **Column Segment** - A column of data within the rowgroup.  Each rowgroup contains one column segment for each column in the table/index.  Column segments also stores metadata that can be used by the query processor when retreiving data(e.g. max value)
- **Delta rowgroup** - A clustered index used only with columnstore indexes.  Stores modified rows until the number of rows reaches a threshold, and are then moved to the columnstore.  Inserted rows are stored here.  Delete operations mark the row as removed from the column segment.  Updates are treated as a delete followed by an insert
- **Deltastore** - The collection of the delta rowgroups temporarily stores modified in a heap structure until it can be moved into a compressed row group.  Inserts are stored here.  
- **Tuple mover** - is a background process that moves data from the delta store to compressed rowgroups, and manages columnstore maintenance
- **Clustered Columnstore** - A columnstore used as the physical storage for an entire table
- **Non-clustered columnstore** - A secondary columnstore structure that can be created on a rowstore table.  
- **Batch-mode execution** - A query processing method used to process multiple rows together
## Use cases
- Reporting scenarios when dealing with a large amount of data
- Useful when you need a small number of columns, but a large number of rows
- Not generally useful for picking out a small set of rows(rows are not ordered in any particular way)
- Cannot be used for `varchar(max)`, `nvarchar(max)`, `rowversion`, `sql_variant`, `xml`, or CLR types (e.g. `hierarchyid`)
- Clustered columnstore indexes are generally used for dimensional data warehouses
- Non-clustered columnstore indexes are generally used for analytics on OLTP tables
- Not very useful for small tables(should have hundreds of thousands of rows at least to be useful)

*** Data warehouses
- Fact tables - Contain dimension keys, degenerate dimensions, and measures.  Dimension keys are usually used for grouping, degenerate dimensions for filtering, and measures for aggregation
- Dimensions tables - Contain information about the dimensions.  Often of much lower cardinality compared to fact tables
- Clustered columnstores are generally good for large fact tables
    - All rows stored in a highly compressed way
    - Batch execution mode
    - Can be used to support a variety of reporting queries
    - Non-clustered can still be used if needed(e.g. if a datatype is not supported), but you lose the benefit of compressing the entire table

*** OLTP Reporting
Data Warehouses are generally updated on a schedule.  If more real-time analytics are needed, non-clustered columnstore indexes can be used on OLTP tables.  Be sure to consider how many reporting queries need to be supported, and how flexible the reporting needs to be
    - If the same query is run over and over, a covering index might be better
    - If many different reporting queries are needed, a single columnstore can replace multiple non-clustered rowstore indexes

Guidelines
- Target only the analytically useful columns in the columnstore to reduce the amount of data duplicated in the index
- Delay adding rows to compressed rowgroups.  Columnstore maintenance is generally optimized for large data loads, as usually seen in OLAP.
    - In OLAP, a row may be updated many times soon after it is created, but then becomes relatively static
    - Configured using `WITH (COMPRESSION_DELAY = <Delay minutes>` on the columnstore index definition
- Use filters on columnstore indexes to target the data rows of interest, and filter out rows that are frequently updated

## Using non-clustered indexes with columnstore indexes
General index design principles

- **Columnstore Indexes** - Great for working with large data sets, particularly for aggregation. Not great for looking up a single row, as the index is not ordered 
    - **Clustered** - Compresses the table to greatly reduce memory and disk footprint of data. 
    - **Nonclustered** - Addition to typical table structure, ideal when the columns included cover the needs of the query. 
- **Rowstore Indexes** - Best used for seeking a row, or a set of rows in order. 
    - **Clustered** - Physically reorders table’s data in an order that is helpful. Useful for the primary access path where you fetch rows along with the rest of the row data, or for scanning data in a given order. 
    - **Nonclustered** - Structure best used for finding a single row. Not great for scanning unless all of the data needed is in the index keys, or is in included in the leaf pages of the index

If you need to fetch a small number of rows from a fact table, e.g. for ETL operations, it may be useful to defined a non-clustered rowstore index on the necessary columns, e.g. surrogate key or degenerate dimension columns.

## Columnstore maintentance
- `REORGANIZE` - Runs the tuple mover immediately.  This is an online operation.  Also used to combine smaller rowgroups 
- `REBUILD` - Offline process that complete rebuilds and recompresses the index.  Organizes and compresses data in the best possible fashion

Ways of loading data into a columnstore:
- **Bulk Load Into a Clustered Columnstore** - Different from bulk loading data into a rowstore table, you can load bulk amounts of data into a clustered columnstore index by using an `INSERT INTO <TargetTable> WITH (TABLOCK) ... SELECT ... FROM <SourceName>` statement. 
    - Bulk inserts go into delta store if less than 1,024,000 rows, otherwise are directly compressed into row groups
- **Other Batch Operations** - Loading data where you don’t meet the requirements of the Bulk Load pattern
    - Frequents inserts/updates/deletes can cause fragmentation 

View rowgroup/delta group statistics:
```sql
SELECT state_desc, total_rows, deleted_rows, 
       transition_to_compressed_state_desc as transition
FROM sys.dm_db_column_store_row_group_physical_stats   
WHERE object_id  = OBJECT_ID('Fact.SaleLimited');
GO
```

Force tuple mover to compress rows into rowgroups
```sql
ALTER INDEX CColumnStore ON Fact.SaleLimited REORGANIZE 
                                WITH (COMPRESS_ALL_ROW_GROUPS = ON);
```                                
