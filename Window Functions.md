## Window Aggregate Functions
Aggregate functions(SUM, COUNT, AVG, MIN, MAX) applied to a window of rows defined by the OVER clause.  Allow you to mix detail and aggregated data in the same query.  The OVER clause allows you to define a set of rows for the function per each underlying row.

Example
```sql
SELECT custid, orderid, val,
  SUM(val) OVER(PARTITION BY custid) AS custtotal,
  SUM(val) OVER() AS grandtotal
FROM Sales.OrderValues;
```
Returns all customer/order combinations, along with a total for all that customers orders, and grandtotal for all orders

OVER clause also supports a window frame option.  This allows you to define an ordering within the partition, then confine the frame of rows beween two delimiters.  Two window frame units are available, ROWS and RANGE.  With the ROWS unit, delimiters available are
- `UNBOUNDED PRECEDING` or `FOLLOWING`
- `CURRENT ROW`
- `<n> ROWS PRECEDING` or `FOLLOWING`

The RANGE unit is not well supported by T-SQL, supporting only UNBOUNDED and CURRENT ROW.  When using the same delimiters, the RANGE option includes peers(tied rows in terms of ordering), while the ROWS option doesnt. 

If a window order is specified, but no frame clause is specified, the default window framing is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW.  The ROWS option is better optimized than RANGE, so an explicit frame should generally be specified

Example (computing a running sum of orders for a customer, ordered by date with ID as a tie-breaker)
```sql
SELECT custid, orderid, orderdate, val,
  SUM(val) OVER(PARTITION BY custid
                ORDER BY orderdate, orderid
                ROWS BETWEEN UNBOUNDED PRECEDING
                         AND CURRENT ROW) AS runningtotal
FROM Sales.OrderValues;
```

The optimal index to support window functions is one created on the partitioning and ordering elements as keys, and includes the rest of the elements from the query for coverage(POC index - partitioning, ordering, and covering)

Note, window functions can only be used in SELECT or ORDER BY clauses.  If you need to filter on the value of a window function, use a table expression.

## Window Ranking Functions
Allow you to rank rows within a partition based on a specified ordering(order clause is mandatory).
- `ROW_NUMBER()` - assigns a unique incrementing integer, starting with 1 to rows in the partition.  Note, if ordering is not unique, then ROW_NUMBER() is not deterministic
- `RANK()` - Returns the number of rows in the partition that have a lower ordering value than the current, plus 1.  Always deterministic
- `DENSE_RANK()` - Returns the number of distinct ordering values that are lower than the current, plus one.  There are no gaps in the values(unlike RANK).  Always deterministic  
- `NTILE(N)` - Arrange the rows within the partition in N equally sized tiles, and returns which tile the row is part of.  Not deterministic if ordering is not uniquce  

Example
```sql
SELECT custid, orderid, val,
  ROW_NUMBER() OVER(ORDER BY val) AS rownum,
  RANK()       OVER(ORDER BY val) AS rnk,
  DENSE_RANK() OVER(ORDER BY val) AS densernk,
  NTILE(100)   OVER(ORDER BY val) AS ntile100
FROM Sales.OrderValues;
```

custid | orderid | val | rownum | rnk | densernk | ntile100
------ | ------- | --- | ------ | --- | -------- | --------
12 | 10782 | 12.50 | 1 | 1 | 1 | 1
27 | 10807 | 18.40 | 2 | 2 | 2 | 1
66 | 10586 | 23.80 | 3 | 3 | 3 | 1
76 | 10767 | 28.00 | 4 | 4 | 4 | 1
54 | 10898 | 30.00 | 5 | 5 | 5 | 1
88 | 10900 | 33.75 | 6 | 6 | 6 | 1
48 | 10883 | 36.00 | 7 | 7 | 7 | 1
41 | 11051 | 36.00 | 8 | 7 | 7 | 1
71 | 10815 | 40.00 | 9 | 9 | 8 | 1
38 | 10674 | 45.00 | 10 | 10 | 9 | 2
53 | 11057 | 45.00 | 11 | 10 | 9 | 2
75 | 10271 | 48.00 | 12 | 12 | 10 | 2

## Window Offset Functions
Returns an element from a single row that is in a given offset from the current row in the window partition, or from the first or last row in the window frame
- `LAG(col, [N], [default])` - Supports window partition and ordering.  Returns the value of col from the Nth row preceding the the current row in the partition.  N defaults to 1.  If no row exists, then NULL is returned, or the default value if specified
- `LEAD(col)` - - Supports window partition and ordering.  Returns the value of col from the Nth row following the the current row in the partition.  N defaults to 1.  If no row exists, then NULL is returned, or the default value if specified
- `FIRST_VALUE(col)` - Supports partition, order, and frame clauses.  Returns the value of col in the first row of the window frame
- `LAST_VALUE(col)` - Supports partition, order, and frame clauses.  Returns the value of col in the last row of the window frame


Example: Returns cust oder information, along with values of previous and next orders for the customer(based on date and order id)
```sql
SELECT custid, orderid, orderdate, val,
  LAG(val)  OVER(PARTITION BY custid
                 ORDER BY orderdate, orderid) AS prev_val,
  LEAD(val) OVER(PARTITION BY custid
                 ORDER BY orderdate, orderid) AS next_val
FROM Sales.OrderValues;
```

Example: Returns cust order info, along with values for the first and last order for that customer(based on date and order id)
```sql
SELECT custid, orderid, orderdate, val,
  FIRST_VALUE(val)  OVER(PARTITION BY custid
                         ORDER BY orderdate, orderid
                         ROWS BETWEEN UNBOUNDED PRECEDING
                                  AND CURRENT ROW) AS first_val,
  LAST_VALUE(val) OVER(PARTITION BY custid
                       ORDER BY orderdate, orderid
                       ROWS BETWEEN CURRENT ROW
                                 AND UNBOUNDED FOLLOWING) AS last_val
FROM Sales.OrderValues
ORDER BY custid, orderdate, orderid;
```