Concurrency


# Concurrency

A database with a high-level of concurrency can support a high-number of simulataneous processes that do not interfere with eachother.  Problems can occur when simultaneous processes attempt to read from/write to the same data.  Some of the common problems include
- **Dirty Reads** AKA uncommitted dependencies, can occur when an uncommitted transaction updates a row at the same time that another transaction reads that row with its new value. The first transaction might change the row again before committing, and consequently the second transaction has invalid data.  In the following example, transaction 2 might print 'Greetings', 'Hellloooooo', or 'La la la', depending on the ordering of statements.
```sql
CREATE TABLE T1 (id int, val varchar(20))
INSERT INTO T1 (1, 'Greetings')

-- Transaction 1
BEGIN TRAN
UPDATE T1 SET val = 'Hellloooooo' where id = 1
UPDATE T1 SET val = 'La la la' where id = 1
COMMIT TRAN

-- Transaction 2
BEGIN TRAN
SELECT val FROM T1 where id = 1
PRINT val
COMMIT TRAN
```
- **Non-Repeatable Reads** AKA inconsistent analysis, can occur when data is read more than once within the same transaction while another transaction updates the same data between read operations.  In the following example, transaction 2 may print different values due to the change made by transaction 1
```sql
CREATE TABLE T1 (id int, val varchar(20))
INSERT INTO T1 (1, 'Hellloooooo')

-- Transaction 1
BEGIN TRAN
UPDATE T1 SET val = 'La la la' where id = 1
COMMIT TRAN

-- Transaction 2
BEGIN TRAN
SELECT val FROM T1 where id = 1
PRINT val
SELECT val FROM T1 where id = 1
PRINT val
COMMIT TRAN
``` 
- **Phantom reads** can occur when one transaction reads the same data multiple times while another transaction inserts or updates a row between read operations.  This situation occurs only when the query uses a predicate.  
```sql
CREATE TABLE sayings (id INT, name VARCHAR(100))

--Transaction 1  
BEGIN TRAN;  
SELECT ID FROM dbo.sayings  
WHERE ID > 5 and ID < 10;  
--The INSERT statement from the second transaction occurs here.  
SELECT ID FROM dbo.sayings  
WHERE ID > 5 and ID < 10;  
COMMIT;

--Transaction 2  
BEGIN TRAN;  
INSERT INTO dbo.employee  
  (Id, Name) VALUES(6 ,'We dont need no stinkin badges');  
COMMIT;  
```
- **Lost updates** Occurs when simultaneous transactions read the same row and then update the row based on the values originally selected. Whichever of these transactions is committed first becomes a lost update because it was replaced by the update in the other transaction.  In the following example, transaction 1 and 2 are executed simultaneously.  If they are fully serialized, then T1 will have a value of 10 for ID 1.  Without proper concurrency mechanisms, the value could be 9, 10, or 11, depending on the ordering of statements, and which transaction completes first. 
```sql
CREATE TABLE T1 (id int, val int)
INSERT INTO T1 (1, 10)

-- Transaction 1
BEGIN TRAN
SELECT @val = val FROM T1 where id = 1
UPDATE T1 SET val = @val + 1
COMMIT TRAN

-- Transaction 2
BEGIN TRAN
SELECT @val = val FROM T1 where id = 1
UPDATE T1 SET val = @val - 1
COMMIT TRAN
```
- **Missing/double reads** can occur when one transaction is scanning an index, while another transaction makes an update that changes the index key column.  In this case, the row may be moved to a position ahead of, or behind the scan.  In the first case, this would cause a double read, and in the second it would cause a missing read.
```sql
CREATE TABLE T1 (id int, val int)
INSERT INTO T1 
VALUES (1, 1), (2, 1), (3, 2), (4, 3), (5, 5)

CREATE NONCLUSTERED INDEX idx1 ON T1 (val)

-- Transaction 1
BEGIN TRAN
SELECT val from T1
COMMIT TRAN

-- Transaction 2
BEGIN TRAN
UPDATE T1 SET val = 8 WHERE id = 2
COMMIT TRAN
```
## Concurrency Control
- **Pessimistic concurrency control**:  Uses locks to isolate transactions. When a transaction performs an action that acquires a lock on a resource, other transactions cannot perform conflicting actions on that resource until the lock is released.  Mainly used in environments where there is high contention for data, where the cost of protecting data with locks is less than the cost of rolling back transactions if concurrency conflicts occur.
- **Optimistic concurrency control**: Transactions do not lock data when they read it. When a transaction attempts to change data, the system checks to see if another transaction changed the data after it was read, and raises an error if so. Typically, the transaction receiving the error rolls back the transaction and starts over. Mainly used in environments where there is low contention for data, and where the cost of occasionally rolling back a transaction is lower than the cost of locking data when read.
## Locks

SQL Server uses locks to support concurrency while maintaining isolation.  The locks used by a transaction depend on its isolation level.
### Lock Resources
SQL Server can lock resources at multiple levels of granularity.  Locking resources at the most granular level improves concurrency, at the cost of more overhead for managing the locks.

A lock hierarchy may be established to fully protect a given resource

| Resource | Description |
| ---------|-------------|
| RID      | A row identifier used to lock a single row within a heap. |
| KEY      | A key or key-range lock within an index, depending on isolation level. |
| PAGE     | An 8-kilobyte (KB) page in a database, such as data or index pages.  Locked when a transaction reads all rows on the page, or during page-level maintenance |
| EXTENT   | A contiguous group of eight pages, such as data or index pages.  Typically locked during space allocation/deallocation |
| HoBT     | A heap or B-tree. A lock protecting an entire B-tree (index) or all the data pages in a heap. |
| TABLE    | The entire table, including all data and indexes. |
| FILE     | A database file. |
| APPLICATION | An application-specified resource, using `sp_getapplock`. |
| METADATA | System metadata locks to protect catalog information |
| ALLOCATION_UNIT | An allocation unit. |
| DATABASE | The entire database.  Gets a shared lock to indicate that it is in use so that another process cannot drop/restore/etc |

### Lock Modes:
- **Shared (S)**: Also known as a read lock. Used for read operations and is released as soon as data has been read from the locked resource.  Other transactions may also hold a shared lock on the resource, but may not modify the resource. In order to hold the lock for the entire transaction, use the `HOLDLOCK` table hint, or set the `REPEATABLE_READ` or `SERIALIZABLE` transaction isolation levels.
- **Update (U)**: Used on a resource that might be updated in order to prevent a common type of deadlocking.  Only one update (U) lock can exist on a resource at a time. When a transaction modifies the resource, the update (U) lock is converted to an exclusive (X) lock.  
- **Exclusive (X)**: Protects a resource during `INSERT`, `UPDATE`, or `DELETE` operations to prevent multiple concurrent changes. While the lock is held, no other transaction can read or modify the data, unless a statement uses the `NOLOCK` hint or a transaction runs under the read uncommitted isolation level.
- **Intent**:  Used to establish a lock hierarchy.  Signals an intent to aquire locks at a lower level.  This prevents other transactions from modifying the higher level resources, and improves efficiency by detecting lock conflicts at higher levels in the hierarchy
    - **Intent shared (IS)**: Protects requested or acquired shared (S) locks on *some* resources lower in the lock hierarchy.
    - **Intent exclusive (IX)**: Protects requested or acquired exclusive (X) and shared (S) locks on *some* resources lower in the hierarchy
    - **Shared with intent exclusive (SIX)**: Protects requested or acquired shared (S) locks on *all* resources lower in the hierarchy and intent exclusive (IX) locks on *some* resources lower in the hierarchy. Only one shared with intent exclusive (SIX) lock can exist at a time for a resource to prevent other transactions from modifying it. However, concurrent intent shared (IS) locks are allowed
    - **Intent update (IU)**: Used on page resources only to protect requested or acquired update (U) locks on *all* lower-level resources and converts it to an intent exclusive (IX) lock if a transaction performs an update operation. 
    - **Shared intent update (SIU)** A combination of shared (S) and intent update (IU) locks and occurs when a transaction acquires each lock separately but holds them at the same time. 
    - **Update intent exclusive (UIX)** A combination of update (U) and intent exclusive (IX) locks that a transaction acquires separately but holds at the same time.  
- **Schema** Used when an operation depends the table’s schema. There are two types of schema locks: 
    - **Schema modification (Sch-M)** This lock mode prevents other transactions from reading from or writing to a table during a Data Definition Language (DDL) operation, such as removing a column. Some Data Manipulation Language (DML) operations, such as truncating a table, also require a schema modification (Sch-M) lock. 
    - **Schema stability (Sch-S)**: Used during query compilation and execution to block concurrent DDL operations and concurrent DML operations requiring a schema modification (Sch-M) lock from accessing a table. 
- **Bulk Update (BU)**: This lock mode is used for bulk copy operations to allow multiple threads to bulk load data into the same table at the same time and to prevent other transactions that are not bulk loading data from accessing the table. SQL Server acquires it when the table lock on bulk load table option is set by using sp_tableoption or when you use a `TABLOCK` hint
- **Key-range**: A key-range lock is applied to a range of rows that is read by a query with the `SERIALIZABLE` isolation level to prevent phantom reads
    - **RangeS-S** This lock mode is a shared range, shared resource lock used for a serializable range scan   
    -  **RangeS-U** This lock mode is a shared range, update resource lock used for a serializable update scan. 
    - **RangeI-N** This lock mode is an insert range, null resource lock that SQL Server acquires to test a range before inserting a new key into an index. 
    - **RangeX-X** This lock mode is an exclusive range, exclusive resource lock used when updating a key in a range

### Lock Compatibility
|   | IS | S | U | IX | SIX | X |
|---|----|---|---|----|-----|---|
|IS |  Y | Y | Y | Y  | Y   | N |
| S |  Y | Y | Y | N  | N   | N |
| U |  Y | Y | N | N  | N   | N |
|IX |  Y | N | N | Y  | N   | N |
|SIX|  Y | N | N | N  | N   | N |
| X |  N | N | N | N  | N   | N |

### Lock Escalation
SQL Server uses a dynamic locking strategy to determine the most cost-effective locks(in terms of granularity). The SQL Server Database Engine automatically determines what locks are most appropriate when the query is executed, based on the characteristics of the schema and query.

Lock escalation occurs when > 40 % memory pool is required by locks, or at least 5000 locks are taken in a single statement for single resource.  Intent locks are converted to full locks if possible and lower level locks are released.  

See `LOCK_ESCALATION`
