## Views
A view is a reusable named query, or table expression, whose definition is stored as an object in the database.  Views are defined using the `CREATE VIEW` or `CREATE OR ALTER VIEW` statements, which must be the first statement in the batch.  As with other table expressions, the view query must meed the following conditions
- All columns must have names
- All column names must be unique
- The inner query is not allowed to have an ORDER BY clause, unless this clause supports a TOP or OFFSET-FETCH filter

Example view definition
```sql
USE TSQLV4;
GO
CREATE OR ALTER VIEW Sales.OrderTotals
  WITH SCHEMABINDING
AS

SELECT
  O.orderid, O.custid, O.empid, O.shipperid,  O.orderdate,
  O.requireddate, O.shippeddate,
  SUM(OD.qty) AS qty,
  CAST(SUM(OD.qty * OD.unitprice * (1 - OD.discount))
       AS NUMERIC(12, 2)) AS val
FROM Sales.Orders AS O
  INNER JOIN Sales.OrderDetails AS OD
    ON O.orderid = OD.orderid
GROUP BY
  O.orderid, O.custid, O.empid, O.shipperid, O.orderdate,
  O.requireddate, O.shippeddate;
GO
```
With this definition, you can query the view just like you would query any table.  

View attributes
- `SCHEMABINDING` - Prevents structural changes to underlying objects while the view exists
- `ENCRYPTION` - Obfuscates the object definition that is stored internally.  Otherwise, the definition can be found using the  `OBJECT_DEFINITION` function.

You can issue insert/update/delete statements against a view, which will modify data in the underlying tables.  If the inner query joins multiple tables, modification statements are only allowed to affect one table at a time.

### Indexed Views
When you query a view, SQL server expands the view definition and issues the query against the underlying tables.  It is possible to create a clustered index on the view, which will improve read performance at the cost of increased write costs to underlying tables, and the storage space required for the index.  To create an index on a view
- The view must have the `SCHEMABINDING` attribute
- If the query is a grouped query, it must include the `COUNT_BIG(*)` aggregate
- You cannot manipulate the result of an aggregate calculation in the view select list
- The first index created must be clustered and unique

## User defined functions
UDFs are routines that accept parameters, apply calculations, and returns either a scalar or table-valued result.  There are some limitations.  In a UDF, you cannot
- Use error handling
- Modify data
- Use Data Definition Language
- Use temporary tables
- Use dynamic SQL

### Scalar
Example
```sql
CREATE OR ALTER FUNCTION dbo.SubtreeTotalSalaries(@mgr AS INT)
  RETURNS MONEY
WITH SCHEMABINDING
AS
BEGIN
  DECLARE @totalsalary AS MONEY;

  WITH EmpsCTE AS
  (
    SELECT empid, salary
    FROM dbo.Employees
    WHERE empid = @mgr

    UNION ALL

    SELECT S.empid, S.salary
    FROM EmpsCTE AS M
      INNER JOIN dbo.Employees AS S
        ON S.mgrid = M.empid
  )
  SELECT @totalsalary = SUM(salary)
  FROM EmpsCTE;

  RETURN @totalsalary;
END;
```
### Table
Inline example
```sql
CREATE OR ALTER FUNCTION dbo.GetPage(@pagenum AS BIGINT, @pagesize AS BIGINT)
  RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
  WITH C AS
  (
    SELECT ROW_NUMBER() OVER(ORDER BY orderdate, orderid) AS rownum,
      orderid, orderdate, custid, empid
    FROM Sales.Orders
  )
  SELECT rownum, orderid, orderdate, custid, empid
  FROM C
  WHERE rownum BETWEEN (@pagenum - 1) * @pagesize + 1 AND @pagenum * @pagesize;
```

Multi-statement example
```sql
CREATE FUNCTION dbo.GetSubtree (@mgrid AS INT, @maxlevels AS INT = NULL)
RETURNS @Tree TABLE
(
  empid    INT          NOT NULL PRIMARY KEY,
  mgrid    INT          NULL,
  empname  VARCHAR(25)  NOT NULL,
  salary   MONEY        NOT NULL,
  lvl      INT          NOT NULL,
  sortpath VARCHAR(892) NOT NULL,
  INDEX idx_lvl_empid_sortpath NONCLUSTERED(lvl, empid, sortpath)
)
WITH SCHEMABINDING
AS
BEGIN
  DECLARE @lvl AS INT = 0;

  -- insert subtree root node into @Tree
  INSERT INTO @Tree(empid, mgrid, empname, salary, lvl, sortpath)
    SELECT empid, NULL AS mgrid, empname, salary, @lvl AS lvl, '.' AS sortpath
    FROM dbo.Employees
    WHERE empid = @mgrid;

  WHILE @@ROWCOUNT > 0 AND (@lvl < @maxlevels OR @maxlevels IS NULL)
  BEGIN
    SET @lvl += 1;

    -- insert children of nodes from prev level into @Tree
    INSERT INTO @Tree(empid, mgrid, empname, salary, lvl, sortpath)
      SELECT S.empid, S.mgrid, S.empname, S.salary, @lvl AS lvl,
        M.sortpath + CAST(S.empid AS VARCHAR(10)) + '.' AS sortpath
      FROM dbo.Employees AS S
        INNER JOIN @Tree AS M
          ON S.mgrid = M.empid AND M.lvl = @lvl - 1;
  END;
  
  RETURN;
END;
```
## Stored Procedures