Rowstore Indexes

# Design and implement indexes
## Terminology and basics
- **Page** - A contiguous, 8KB chunk of memory
- **B-Tree** - A data structure containing a tree of node and leaf pages.
- **Rowstore** - index structures (B-tree based) that are designed to store row data together
- **Clustered index** - Indexes where the the leaf pages contain the actual data in the table
    - A table can have only one clustered index
    - Index key columns are called the *clustering key*
    - Maximum key size is 900 B
- **Non-clustered index** - A separate index structure containing a partial copy of the table data, along with a pointer to the corresponding row for heaps, or the clustering key of the corresponding row for clustered index tables.
    - **Key columns** are used to for the index key
    - **Included columns** are included on the leaf pages so they don't need to be looked up
    - Maximum key size is 1700 B
- **Heap** - A table with no clustered index.  Data is stored in non-sequential pages
- **Simple index** - An index key with a single column.  Index node and leaf page data are sorted based on this column value
- **Composite index** - An index ky that spans multiple columns.  Index node and leaf page data are sorted by the first column, then the second column, etc. 
    - It is best to choose the most *selective* column as the lead column, i.e. the column with the most unique values across the rows of the table
- **Filtered index** - An index with a `WHERE` clause that limits the values copies by the index

## Index design
### During Database design phase
- Primary key constraints automatically create an index(clustered by default)
- Unique columns constraints automatically create an index(non-clustered by default)
- Primary/Unique indexes are used to enforce constraints, and should not be removed
- Columns used for foreign keys are often good candidates for an index.  Be sure to consider the selectivity of column
### After design phase
Creating indexes based on query, performance, and data requirements, during development and production phases
#### Common Search Paths
- Look for columns that are not used of primary keys, foreign keys, or uniqueness constraints, but are frequently used to filter queries
- Use `SET STATISTICS TIME ON` and `SET STATISTICS IO OFF` to see CPU and IO usage by queries.  Of particular interest is the number of logical reads necessary to execute the query
- Computed columns can be used for indexes as long as they are deterministic
WWE example (from book). Run with and without the index, and look at plan/stats to see plan 
```
SET STATISTICS TIME ON;
SET STATISTICS IO ON;

SELECT CustomerID, OrderId, OrderDate, ExpectedDeliveryDate
FROM  Sales.Orders
WHERE CustomerPurchaseOrderNumber = '16374';

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO

CREATE INDEX CustomerPurchaseOrderNumber ON Sales.Orders(CustomerPurchaseOrderNumber);
```
#### Join conditions
- Look for hash match joins in the query plan.

#### Sorts
- Indexes can be useful when a query needs to sort data
- By default, data stored in indexes is sorted in ASC order.
- Scans can go in either order, but ASC/DESC is important for composite keys
- Sorting can also enable the merge join operator for joining large data sets(which is usually cheaper than the hash match)

## Covering indexes
Key lookup operations are used when scanning or seeking on a non-clustered index in order to look up rows in the table that correspond to the non-clustered index key.  As the number of rows returned by a query increases, the cost of key lookup operators can significanly degrade performance.  In these cases, the optimizer may even prefer to do a clustered index or table scan rather than using the non-clustered index.  

To deal with this, nonclustered indexes can *include* additional columns from the table in the leaf pages.  An index that contains all the data needed for a query, either in key columns or included columns, is called a *covering index*.  When creating a covering index for a query, use columns used in `WHERE` and `JOIN` clauses as the index key, and columns used in the `SELECT` clause as the included columns.  Example:

```sql
DROP INDEX IF EXISTS PreferredName_Include_FullName ON Application.People; 
DROP INDEX IF EXISTS ContactPersonID_Include_OrderDate_ExpectedDeliveryDate on Sales.Orders

-- Look at query plan and stats with no covering index
SET STATISTICS TIME ON;
SET STATISTICS IO ON;

SELECT Orders.ContactPersonId, People.PreferredName
FROM  Sales.Orders
        JOIN Application.People
            ON People.PersonID = Orders.ContactPersonID
WHERE  People.PreferredName = 'Aakriti';

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO

-- Key on ContactPersonID (used in join), include values needed in the Select
CREATE NONCLUSTERED INDEX ContactPersonID_Include_OrderDate_ExpectedDeliveryDate
ON Sales.Orders ( ContactPersonID ) 
INCLUDE ( OrderDate,ExpectedDeliveryDate)
ON USERDATA;
GO

-- Key on PreferredName(used in WHERE), and include FullName(used in select)
-- Note that People.PersonID is automatically included, as it is the primary key
CREATE NONCLUSTERED INDEX PreferredName_Include_FullName 
ON Application.People (	PreferredName )
INCLUDE (FullName)
ON USERDATA;
GO

-- Look at query plan and stats with covering index
SET STATISTICS TIME ON;
SET STATISTICS IO ON;

SELECT OrderId, OrderDate, ExpectedDeliveryDate, People.FullName
FROM  Sales.Orders
        JOIN Application.People
            ON People.PersonID = Orders.ContactPersonID
WHERE  People.PreferredName = 'Aakriti';

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO

```
When creating covering indexes, be mindful of tradeoffs.  Including too many columns can result in large index structures, and maintainence of non-clusetered indexes adds overhead to data modification statements.

Note that SQL server may sometimes scan a non-clustered index that includes columns it needs for a query, because the non-clustered index likely is smaller than the clustered index.  For example, this query will scan the non-clustered index defined above on `Sales.Orders`.  However, it cannot use the included columns for seeks or sorts
```sql
SELECT OrderDate, ExpectedDeliveryDate
FROM  Sales.Orders
WHERE OrderDate > '2015-01-01';
```

## Best practices for clustered indexes
Charactistics to consider when choosing a clustered index:
- The clustered index contains all of the table data on leaf pages.  Perfomance will generally be better when queries can make optimal use of the clustered index
- The clustering key affects other rowstore indexes on the table.  A large clustering key could degrade performance of these other indexes.
- If the clustered index is not uniuqe, a 4B uniquifier is attached by the system
- If the value of clustering key columns is changed, all non-clustered rowstore indexes will also have to change
- Good to choose an increasing value, as it will be inserted at the end of the structure, leading to minimal page splits

Scenarios
- Columns that are used for single row fetches, often for modification.  Primary key is often used for this case, and an identity or sequence can ensure the desired 'increasing key' property
- Range queries - Having the data in order can assist in range-based queries
- Queries with large result sets - If these are run frequently, and return a lot of rows, it is best to perform these searches by a clustered index

Usually, the primary key is chosen as the clustered key.  There needs to be a compelling reason to use something else for a clustered index.  

Common Data types for keys
- Integer
    - Small
    - Can use identity or sequences to ensure that the keys always increase when new data is inserted
- GUID
    - Large(16B), and often are not generated in sequential order
    - Can be generated on the client side(good for concurrency)
