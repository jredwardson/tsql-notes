## Pivoting Data
Groups and aggregates data, transitioning it from a state of rows to a state of columns.  Need to identify three elements
- What do you want to see on the rows(*grouping* element)
- What do you want to see on the columns(*spreading* element)
- What do you want to see in the intersection of each row and column(*aggregation* element)

General Form
```sql
WITH PivotData AS
(
    SELECT
        <grouping col>,
        <spreading col>,
        <aggregation col>
    FROM <source table>
)
SELECT <select list>
FROM PivotData
    PIVOT ( <aggregate function> (<aggregation column) 
        FOR < spreading column> IN (<distinct spreading vals>)) AS P        
```

Example
```sql
WITH PivotData AS
(
  SELECT
    custid,    -- grouping column
    shipperid, -- spreading column
    freight    -- aggregation column
  FROM Sales.Orders
)
SELECT custid, [1], [2], [3]
FROM PivotData
  PIVOT(SUM(freight) FOR shipperid IN ([1],[2],[3]) ) AS P;
```

custid	| 1 | 2 | 3
------- |---|---|---
23 | 11.26 | 101.95 | 524.73
46 | 68.84 | 388.07 | 277.50
69 | 48.62 | 8.29 | 7.56
29 | 20.05 | 10.14 | 7.79
75 | 45.31 | 500.40 | 12.96
15 | 121.62	66.20 | NULL
9 | 341.16 | 419.57 | 597.14
... | ... | ... | ...

Notes:
- The grouping element is identified by elimination.  This is why it's recommended to prepare the data using a CTE
- Will return NULL if no aggregate column data is available for grouping/spreading column.  Use ISNULL or COALESCE in select list
- Aggregate and spreading columns cannot directly be results of expressions(use the CTE if needed)
- COUNT(*) is not allowed as aggregate function.  Can use a dummy column, e.g. `1 AS agg_col` in CTE, then `count(agg_col)` in PIVOT  
- PIVOT operator can only use one aggregate function
- The IN clause accepts a static list of spreading values, and does not support subqueries.  Use dynamic sql if values are not know ahead of time

## Unpivot
Rotate the data from a state of columns to a state of rows.  In every unpivoting task, identify the three elements involved
- The name you want to assign to the target values column
- The name you want to assign to the target names column
- The set of source columns you are unpivoting

General Form
```sql
SELECT <column list>, <names column>, <values column>
FROM <source table>
    UNPIVOT ( <values column> FOR <names column> IN (<source columns>)) AS U;
```

Example
```sql
DROP TABLE IF EXISTS Sales.FreightTotals;
GO

WITH PivotData AS
(
  SELECT
    custid,    -- grouping column
    shipperid, -- spreading column
    freight    -- aggregation column
  FROM Sales.Orders
)
SELECT *
INTO Sales.FreightTotals
FROM PivotData
  PIVOT( SUM(freight) FOR shipperid IN ([1],[2],[3]) ) AS P;

SELECT * FROM Sales.FreightTotals;

-- unpivot data
SELECT custid, shipperid, freight
FROM Sales.FreightTotals
  UNPIVOT( freight FOR shipperid IN([1],[2],[3]) ) AS U;
```

custid | shipperid | freight
------ | --------- | -------
23 | 1 | 11.26
23 | 2 | 101.95
23 | 3 | 524.73
46 | 1 | 68.84
46 | 2 | 388.07
46 | 3 | 277.50
... | ... | ...

Notes
- UNPIVOT filters out rows with NULLs in the value column.  In order to preserve nulls, prepare the source table using a CTE using ISNULL or COALESCE to replace nulls
- Names column is defined as NVARCHAR(128)
- Values column is defined as the same type as that of the source columns.  The types of all columns being unpivoted must be the same
- UNPIVOT requires a static list of columns to unpivot.  Use dynamic sql if the column names are not know in advance
- UNPIVOT is limited to unpivoting only one values column