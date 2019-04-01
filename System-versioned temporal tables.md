## System-versioned temporal tables

Allow you to track historical changes to values in a table.  SQL server supports system-versioned tables, meaning that the system transaction time determines the effective time of the change.  Elements required for to mark it as a temporal table
- A primary key constraint
- Two DATETIME2 columns to store start and end of the validity period
- Start column marked with `GENERATED ALWAYS AS ROW START`
- End column marked with `GENERATED ALWAYS AS ROW END`
- Start/end colums designated using `PERIOD FOR SYSTEM_TIME (<start>, <end>)`
- Table option `SYSTEM_VERSIONING` set to ON
- A linked history table, which can be created automatically

Example table setup
```sql
CREATE TABLE dbo.Products
(
  productid   INT          NOT NULL
    CONSTRAINT PK_dboProducts PRIMARY KEY(productid),
  productname NVARCHAR(40) NOT NULL,
  supplierid  INT          NOT NULL,
  categoryid  INT          NOT NULL,
  unitprice   MONEY        NOT NULL,
-- below are additions related to temporal table
  validfrom   DATETIME2(3)
    GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
  validto     DATETIME2(3)
    GENERATED ALWAYS AS ROW END   HIDDEN NOT NULL,
  PERIOD FOR SYSTEM_TIME (validfrom, validto)
)
WITH ( SYSTEM_VERSIONING = ON ( HISTORY_TABLE = dbo.ProductsHistory ) );
```
### Modifying Data
When data is inserted, updated, or deleted from the table, SQL server tracks changes in the History table.  Times are in UTC time zone, and all changes in a given transaction are given an effective time based on start time for that transaction.
- Insert - SQL server sets the start column to the transaction's start time and the end time to the maximum point in time for the data type.  No rows need to be written to the history table
- Delete - SQL server moves the affected rows to the history table, and sets the end column to the start time of the transaction
- Update - Handled as a delete plus an insert.  The current table will have the new values, with start time set to the change time, and the history table will hold the old state, with the end colum set to the change time.  Note that if you update the same row multiple times in a single transaction, you will end up with history records that have a degenerate interval(start/end time are the same)

### Querying data
Use the FOR SYSTEM_TIME clause to query rows that are valid at certain points in time.  General form
```sql
SELECT <select list>
FROM <temporal table>
    FOR SYSTEM_TIME <subclause>
WHERE <filter>
...
```

Subclauses:
- `AS OF @dt` - returns rows where `validfrom <= @dt` and `validto > @dt`
- `FROM @start TO @end` - rows where `validfrom < @end and validto > @start`
- `BETWEEN @start TO @end` - rows where `validfrom =< @end and validto > @start`
- `CONTAINED IN(@start, @end)` - returns rows where `validfrom >= @start and validto <= @end`