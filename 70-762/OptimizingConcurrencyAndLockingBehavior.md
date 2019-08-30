# Optimizing Concurrency and Locking Behavior

SQL Server uses locks to control the effect of concurrent transactions on one another.
An administrator's job is to improve concurrency by properly managing locking behavior.
Understand how to uncover performance problems related to locks and lock escalations.
Additionally, know how to use the tools available for identifying when and why
deadlocks happen and the possible steps to take to prevent deadlocks from arising.

* Troubleshoot locking issues
* Identify lock escalation behaviors
* Capture and analyze deadlock graphs
* Identify ways to remediate deadlocks



## Troubleshooting Locking issues

DMV = Dynamic Management Views: Helpful to view information about locks

 1.**sys.dm_tran_locks** Use this DMV to view all current locks, the lock resources, lock
mode, and other related information.
The sys.dm_tran_locks DMV provides you with information about existing locks and locks that
have been requested but not yet granted in addition to details about the resource for which
the lock is requested. You can use this DMV only to view information at the current point in
time. It does not provide access to historical information about locks.

| resource_type | resource_subtype | resource_database_id | resource_description                                                                                                                                                                                                                                             | resource_associated_entity_id | resource_lock_partition | request_mode | request_type | request_status | request_reference_count | request_lifetime | request_session_id | request_exec_context_id | request_request_id | request_owner_type           | request_owner_id | request_owner_guid                   | request_owner_lockspace_id | lock_owner_address |
|---------------|------------------|----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|-------------------------|--------------|--------------|----------------|-------------------------|------------------|--------------------|-------------------------|--------------------|------------------------------|------------------|--------------------------------------|----------------------------|--------------------|
| DATABASE      |                  | 16                   |                                                                                                                                                                                                                                                                  | 0                             | 14                      | S            | LOCK         | GRANT          | 1                       | 0                | 76                 | 0                       | 0                  | SHARED_TRANSACTION_WORKSPACE | 0                | 00000000-0000-0000-0000-000000000000 | 0x00000088637DD530:0:0     | 0x0000008AE618D7C0 |
| DATABASE      |                  | 4                    |                                                                                                                                                                                                                                                                  | 0                             | 1                       | S            | LOCK         | GRANT          | 1                       | 0                | 55                 | 0                       | 0                  | SHARED_TRANSACTION_WORKSPACE | 0                | 00000000-0000-0000-0000-000000000000 | 0x0000008B9626A1C0:0:0     | 0x0000008B90EC0780 |
| DATABASE      |                  | 4                    |                                                                                                                                                                                                                                                                  | 0                             | 9                       | S            | LOCK         | GRANT          | 1                       | 0                | 51                 | 0                       | 0                  | SHARED_TRANSACTION_WORKSPACE | 0                | 00000000-0000-0000-0000-000000000000 | 0x0000008B9580A2E0:0:0     | 0x0000008B944ABF40 |
| DATABASE      |                  | 4                    |                                                                                                                                                                                                                                                                  | 0                             | 10                      | S            | LOCK         | GRANT          | 1                       | 0                | 65                 | 0                       | 0                  | SHARED_TRANSACTION_WORKSPACE | 0                | 00000000-0000-0000-0000-000000000000 | 0x0000008B9580E2E0:0:0     | 0x0000008B944AEC00 |
| DATABASE      |                  | 6                    |                                                                                                                                                                                                                                                                  | 0                             | 13                      | S            | LOCK         | GRANT          | 1                       | 0                | 57                 | 0                       | 0                  | SHARED_TRANSACTION_WORKSPACE | 0                | 00000000-0000-0000-0000-000000000000 | 0x000000884A05FA50:0:0     | 0x0000008609969F80 |
| DATABASE      |                  | 6                    |                                                                                                                                                                                                                                                                  | 0                             | 14                      | S            | LOCK         | GRANT          | 1                       | 0                | 82                 | 0                       | 0                  | SHARED_TRANSACTION_WORKSPACE | 0                | 00000000-0000-0000-0000-000000000000 | 0x00000088637DD9C0:0:0     | 0x00000087371B3EC0 |
| DATABASE      |                  | 6                    |                                                                                                                                                                                                                                                                  | 0                             | 14                      | S            | LOCK         | GRANT          | 1                       | 0                | 78                 | 0                       | 0                  | SHARED_TRANSACTION_WORKSPACE | 0                | 00000000-0000-0000-0000-000000000000 | 0x00000089A81760A0:0:0     | 0x0000008782907480 |
| DATABASE      |                  | 6                    |                                                                                                                                                                                                                                                                  | 0                             | 14                      | S            | LOCK         | GRANT          | 1                       | 0                | 70                 | 0                       | 0                  | SHARED_TRANSACTION_WORKSPACE | 0                | 00000000-0000-0000-0000-000000000000 | 0x00000088637DD4A0:0:0     | 0x00000086B796F280 |
| DATABASE      |                  | 7                    |                                                                                                                                                                                                                                                                  | 0                             | 4                       | S            | LOCK         | GRANT          | 1                       | 0                | 64                 | 0                       | 0                  | SHARED_TRANSACTION_WORKSPACE | 0                | 00000000-0000-0000-0000-000000000000 | 0x00000086CC769810:0:0     | 0x0000008A5456DD80 |

Example:
```
BEGIN TRANSACTION;
 SELECT RowId, ColumnText
 FROM Examples.LockingA
 WITH (HOLDLOCK, ROWLOCK);
```
Another transaction in a separate session:
```
BEGIN TRANSACTION;
 UPDATE Examples.LockingA
 SET ColumnText = 'Row 2 Updated'
 WHERE RowId = 2; 
```
Now use sys.dm_tran_locks DMV to view some details about the current locks:
```
SELECT
 request_session_id as s_id,
 resource_type,
 resource_associated_entity_id,
 request_status,
 request_mode
FROM sys.dm_tran_locks
WHERE resource_database_id = db_id('PR_DEV');
```
| s_id | resource_type | resource_associated_entity_id | request_status | request_mode |
|------|---------------|-------------------------------|----------------|--------------|
| 1    | DATABASE      | 0                             | GRANT          | S            |
| 2    | DATABASE      | 0                             | GRANT          | S            |
| 1    | PAGE          | 7.20576E+16                   | GRANT          | IS           |
| 2    | PAGE          | 7.20576E+16                   | GRANT          | IX           |
| 1    | KEY           | 7.20576E+16                   | GRANT          | RangeS-S     |
| 1    | KEY           | 7.20576E+16                   | GRANT          | RangeS-S     |
| 1    | KEY           | 7.20576E+16                   | GRANT          | RangeS-S     |
| 1    | KEY           | 7.20576E+16                   | GRANT          | RangeS-S     |
| 1    | KEY           | 7.20576E+16                   | GRANT          | RangeS-S     |
| 2    | KEY           | 7.20576E+16                   | WAIT           | X            |
| 1    | OBJECT        | 933578364                     | GRANT          | IS           |


Use resource_associated_entity_id column as the hobt_id for sys.partitions to find which table is currently being locked
```
SELECT
 object_name(object_id) as Resource,
 object_id,
 hobt_id
FROM sys.partitions
WHERE hobt_id=72057594041729024;
```
|Resource | object_id | hobt_id           |
|---------|-----------|-------------------|
|LockingA | 933578364 | 72057594041729024 |

When troubleshooting blocking situations, look for CONVERT in the request_status column
in this DMV. This value indicates the request was granted a lock mode earlier, but now needs
to upgrade to a different lock mode and is currently blocked. 


 2.**sys.dm_os_waiting_tasks** Use this DMV to see which tasks are waiting for a resource.
* Whereas the earlier query showing existing locks is helpful for learning how SQL Server acquires locks, this table
returns information that is more useful on a day-today basis for uncovering blocking chains. For example, in the query 
below you can see that session 2 is blocked by session 1.

```
SELECT
 t1.resource_type AS res_typ,
 t1.resource_database_id AS res_dbid,
 t1.resource_associated_entity_id AS res_entid,
 t1.request_mode AS mode,
 t1.request_session_id AS s_id,
 t2.blocking_session_id AS blocking_s_id
FROM sys.dm_tran_locks as t1
INNER JOIN sys.dm_os_waiting_tasks as t2
 ON t1.lock_owner_address = t2.resource_address; 
```

```
--ALl you need to do is this statement to release both locks.
ROLLBACK TRANSACTION;
```
 3.**sys.dm_os_wait_stats** Use this DMV to see how often processes are waiting while locks are taken.
 Not too useful. Just identifies long term trends but doesn't show you details about the locked resources.


## Identify lock escalation behaviors
In general, maintaining locks requires memory resources. SQL Server uses row level locking by default, 
however it is far more optimal for the SQL Engine to convert a large number of row locks into a single 
table lock, and in the process release memory held by the row locks.

**Lock Escalation** is an optimization technique used by SQL Server and it is basically how it handles 
locking for large updates. When SQL Server is going to modify a large number of rows – it is more efficient 
for the database engine to take fewer, larger locks (table lock) instead of dealing with a large number 
of individual locks (row locks).

**Lock Hierarchy** in SQL Server starts at Database level and goes down to Row level —
Database –> Table –> Page –> Row

**Lock Escalation Threshold**: As per MSDN documentation, Lock escalation is triggered when a Transact-SQL 
statement acquires at least 5,000 locks on a single reference of a table.
```
select name, lock_escalation_desc
from sys.tables
```
| name                           | lock_escalation_desc |
|--------------------------------|----------------------|
| OrgAddress                     | TABLE                |
| ProvOrganization               | TABLE                |
| OrgNetwork                     | TABLE                |
| ProvServiceLocation            | TABLE                |
| Network                        | TABLE                |
| ProvSpecialty                  | TABLE                |
| OrgAddress                     | TABLE                |
| ServiceLocation                | TABLE                |
| Organization                   | TABLE                |
| ServiceLocationLanguage        | TABLE                |

While the Database Engine checks for possible escalations at every 1250 newly acquired locks, a lock 
escalation will occur if and only if a Transact-SQL statement has acquired at least 5000 locks on a 
single reference of a table.

You can reduce the locking issues by —
* Keeping transactions shorter.
* Reducing lock footprint of expensive queries by doing performance tuning and making them efficient.
* Breaking up large operations into batches.


## Capture and analyze deadlock graphs
Usually the process of locking and unlocking SQL Server is fast enough to allow many users to
read and write data with the appearance that it occurs simultaneously. However, sometimes
two sessions block each other and neither can complete, which is a situation known as **deadlocking**.

```
BEGIN TRANSACTION;
 UPDATE Examples.LockingA
 SET ColumnText = 'Row 1 Updated'
 WHERE RowId = 1;
 WAITFOR DELAY '00:00:05';
 UPDATE Examples.LockingB;
 SET ColumnText = 'Row 1 Updated Again'
 WHERE RowId = 1;
```
And in a second session...:
```
BEGIN TRANSACTION;
 UPDATE Examples.LockingB
 SET ColumnText = 'Row 1 Updated'
 WHERE RowId = 1;
 WAITFOR DELAY '00:00:05';
 UPDATE Examples.LockingA;
 SET ColumnText = 'Row 1 Updated Again'
 WHERE RowId = 1; 
```

This is how deadlocks are created: Both transactions need the same table resources. Both transactions 
can successfully update a row without conflict and have an exclusive lock on the updated data.
Then they each try to update data in the table that the other transaction had updated, but
each transaction is blocked while waiting for the other transaction’s exclusive lock to be released. 
Neither transaction can ever complete and release its lock, thereby causing a deadlock.
When SQL Server recognizes this condition, it terminates one of the transactions and
rolls it back. It usually chooses the transaction that is least expensive to rollback based on the
number of transaction log records. At that point, the aborted transaction’s locks are released
and the remaining open transaction can continue.

Deadlocks are not typically going to happen while you watch, so to know how and why they occur, we use deadlock graphs

![Deadlock Graph Example](https://imgur.com/a/jpxr77r)

In the deadlock graph, you see the tables and queries involved
in the deadlock, which process was terminated, and which locks led to the deadlock. The
ovals at each end of the deadlock graph contain information about the processes running the
deadlocked queries. The terminated process displays in the graph with an x superimposed on
it. Hover your mouse over the process to view the statement associated with it. The rectangles
labeled Key Lock identify the database object and index associated with the locking. Lines in
the deadlock graph show the relationship between processes and database objects. A request
relationship displays when a process waits for a resource while an owner relationship displays
when a resource waits for a process.

The book sucks at explaining deadlock graphs.

https://www.sqlshack.com/understanding-graphical-representation-sql-server-deadlock-graph/

1. The processes involved
2. The deadlock victim
3. The resources involved
4. The lock mode of the held and requested locks
5. The queries involved

There are multiple ways to go about troubleshooting deadlocks which include:

* Trace flags 1222, 1204 _These will be on the test_

* Profiler (trace events)

* Extended events


## Identify ways to remediate deadlocks
Deadlocks are less likely to occur if transactions can release resources as quickly as possible.
You can also lock up additional resources to avoid contention between multiple transactions.
For example, you can use a hint to lock a table although this action can also cause blocking.
Usually the best way to resolve a deadlock is to rerun the transaction. For this reason,
you should enclose a transaction in a TRY/CATCH block and add retry logic. Let’s revise the
previous example to prevent the deadlock. Start two new sessions and add the statements in to 
both sessions.


```
--Add retry logic to avoid deadlock
DECLARE @Tries tinyint
SET @Tries = 1
WHILE @Tries <= 3
BEGIN

    BEGIN TRANSACTION
    BEGIN TRY
        UPDATE Examples.LockingB
            SET ColumnText = 'Row 3 Updated'
            WHERE RowId = 3;
        WAITFOR DELAY '00:00:05';
        UPDATE Examples.LockingA
        SET ColumnText = 'Row 3 Updated Again'
            WHERE RowId = 3;
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        SELECT ERROR_NUMBER() AS ErrorNumber;
        ROLLBACK TRANSACTION;
        SET @Tries = @Tries + 1;
        CONTINUE;
    END CATCH
END
```
Next, execute each session. This time the deadlock occurs again, but the CATCH block
captured the deadlock. SQL Server does not automatically roll back the transaction when
you use this method, so you should include a ROLLBACK TRANSACTION in the CATCH block.
The @@TRANCOUNT variable resets to zero in both transactions. As a result, SQL Server no
longer cancels one of the transactions and you can also see the error number generated for
the deadlock victim:

|ErrorNumber|
|-----------|
|1205       |

Re-execution of the transaction might not be possible if the cause of the deadlock is still
locking resources. To handle those situations, you could need to consider the following methods 
as alternatives for resolving deadlocks.

 * Use SNAPSHOT or READ_COMMITTED_SNAPSHOT isolation levels. Either of these options avoid 
 most blocking problems without the risk of dirty reads. However, both of these options 
 require plenty of space in tempdb.
 * Use the NOLOCK query hint if one of the transactions is a SELECT statement, but only 
 use this method if the trade-off of a deadlock for dirty reads is acceptable.
 * Add a new covering nonclustered index to provide another way for SQL Server to read data 
 without requiring access to the underlying table. This approach works only if the other transaction 
 participating in the deadlock does not use any of the covering index keys. The trade-off is 
 the additional overhead required to maintain the index.
 * Proactively prevent a transaction from locking a resource that eventually gets locked by
 another transaction by using the HOLDLOCK or UPDLOCK query hints. 
