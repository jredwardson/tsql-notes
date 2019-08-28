## Table Expressions

In SQL, a query generally returns a table as output.  Table expressions allow you to define a table based on a query that can then be used in other queries.  In TSQL, there are three types of table expressions
- Derived Tables - a table expression defined in the FROM clause of a query
- CTEs - a table expression defined before a query using the WITH clause.
- Views - a table expression saved in the database that can be reused in many places

As an example of a derived table, consider the following:
```sql
-- two products with lowest prices per category
SELECT categoryid, productid, productname, unitprice
FROM (SELECT
        ROW_NUMBER() OVER(PARTITION BY categoryid
                          ORDER BY unitprice, productid) AS rownum,
        categoryid, productid, productname, unitprice
      FROM Production.Products) AS D
WHERE rownum <= 2;
```

The query inside the parenthesis in the `FROM` clause is a derived table.  Filtering on the result of a window function is a common use case for derived tables, because window functions cannot be directly used in the `WHERE` clause(they are only valid in `SELECT` and `ORDER BY`).  This query can also be written using a CTE as follows

```sql
WITH C AS
(
  SELECT
    ROW_NUMBER() OVER(PARTITION BY categoryid
                      ORDER BY unitprice, productid) AS rownum,
    categoryid, productid, productname, unitprice
  FROM Production.Products
)
SELECT categoryid, productid, productname, unitprice
FROM C
WHERE rownum <= 2;
```

In practice, there is not much of a difference between using a CTE vs a derived tables.  However, using CTEs often makes the query cleaner and more readable, especially in cases where you need to use to the table expression multiple times in the same query.  CTEs also have a recursive form, which is not possible with derived tables.

### Recursive CTEs
A recursive CTE is a CTE that references itself.  A common use-case is to return hierarchical data, such as an organization chart or a bill of materials.  For an example, consider the following scenario.  We create an employees table that stores the employee ID, the employee name, and the ID of the employee's manager:

```sql
SET NOCOUNT ON;
USE TSQLV4;
DROP TABLE IF EXISTS dbo.Employees;
GO
CREATE TABLE dbo.Employees
(
  empid   INT         NOT NULL CONSTRAINT PK_Employees PRIMARY KEY,
  mgrid   INT         NULL
    CONSTRAINT FK_Employees_Employees REFERENCES dbo.Employees,
  empname VARCHAR(25) NOT NULL,  
  CHECK (empid <> mgrid)
);

INSERT INTO dbo.Employees(empid, mgrid, empname, salary)
  VALUES(1, NULL, 'David'),
        (2, 1, 'Eitan'),
        (3, 1, 'Ina'),
        (4, 2, 'Seraph'),
        (5, 2, 'Jiru'),
        (6, 2, 'Steve'),
        (7, 3, 'Aaron'),
        (8, 5, 'Lilach'),
        (9, 7, 'Rita'),
        (10, 5, 'Sean'),
        (11, 7, 'Gabriel'),
        (12, 9, 'Emilia' ),
        (13, 9, 'Michael'),
        (14, 9, 'Didi');
GO
```

In this example, David is the 'big bad'; Eitan and Ina are managed by David; Seraph, Jiru and Steve are directly managed by Eitan, and so on.  We would like to construct a query that will return all the employees down the hierarchy from a given manager.  Using a recursive CTE, this looks like

```sql
DECLARE @mgrId INT = 2; -- We will look at the hierarchy under Eitan in this example

WITH EmpCTE AS (

    -- Anchor member: retrieves employee data for the given ID
    SELECT EmpId, 
           MgrId, 
           CAST(EmpName AS VARCHAR(100)) AS EmpName,  -- Cast here to ensure same data types in anchor and recursive members
           0 AS Level, 
           CAST(EmpName AS VARCHAR(100)) AS SortPath
    FROM dbo.Employees
    WHERE empId = @mgrId

    UNION ALL

    -- Recursive member
    SELECT E.EmpId,  
           E.MgrId, 
           CAST(REPLICATE('--', M.level + 1) + E.EmpName AS VARCHAR(100)),  -- Replicate is used to give us a nice visual representation of the hierarchy
           M.Level + 1 AS Level,                                            -- This shows the employee's level in the hierarchy
           CAST(M.SortPath + '\' + E.EmpName AS VARCHAR(100))               -- This allows us to sort based on hierarchical ordering
    FROM EmpCTE M                                    -- We reference the CTE, making it recursive
    INNER JOIN dbo.Employees E ON E.mgrId = M.empId  -- Get all the employees for the current manager(s)
)
SELECT * From EmpCTE ORDER BY SortPath
```

This query results in the following output.

EmpId | MgrId | EmpName | Level | SortPath
----- | ----- | ------- | ----- | --------
2 | 1 | Eitan | 0 | Eitan
5 | 2 | --Jiru | 1 | Eitan\Jiru
8 | 5 | ----Lilach | 2 | Eitan\Jiru\Lilach
10 | 5	| ----Sean | 2 | Eitan\Jiru\Sean
4 | 2 | --Seraph | 1 | Eitan\Seraph
6 | 2 | --Steve | 1 | Eitan\Steve

To break down how this query works
* First, SQL Server executes the anchor member, which gives the following result set (call it `T0`)

EmpId | MgrId | EmpName | Level | SortPath
----- | ----- | ------- | ----- | --------
2 | 1 | Eitan | 0 | Eitan

* Then, SQL Server invokes the recursive member, using `T0` as the input for EmpCTE.  This results in the following result set(the employees directly under Eitan), `T1`

EmpId | MgrId | EmpName | Level | SortPath
----- | ----- | ------- | ----- | --------
5 | 2 | --Jiru | 1 | Eitan\Jiru
4 | 2 | --Seraph | 1 | Eitan\Seraph
6 | 2 | --Steve | 1 | Eitan\Steve

* SQL Server invokes the recursive member again, this time with `T1` as the input.  This results in the following set, `T2`(The employees directly under Jiru, Seraph, or Steve.  In this case, only Jiru has employees under him)

EmpId | MgrId | EmpName | Level | SortPath
----- | ----- | ------- | ----- | --------
8  | 5 | ----Lilach | 2 | Eitan\Jiru\Lilach
10 | 5 | ----Sean | 2 | Eitan\Jiru\Sean

* SQL Server invokes the recursive member again, with `T2` as the input.  This results in an empty set, because Lilach and Sean have employees under them.  
* Once an empty set is reached, the recursion is complete, and SQL server returns `T0 UNION ALL T1 UNION ALL T2`  


