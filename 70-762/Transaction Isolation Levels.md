# Handling Concurrency with Transaction Isolation Levels

- Concurrency <> Performance
- Concurrency = Running multiple transactions at once.
- Performance = Speed of successfully completing transactions.

## Locks

Exclusive Lock (X) - No other locks can be placed on the resource until the lock is released.

Shared Lock (S) - Shared and update locks can be placed on resources locked with shared locks.

Update Lock (U) - Shared locks can be placed on resources with update locks. When the statement is ready to perform the update, this will be converted to an exclusive lock.

## Isolation Levels
Specify how read operations should behave in the presense of concurrent write operation.  Lower isolation levels improve concurrency at the cost of increased anomolies.

READ UNCOMMITTED
* How it Works
	* Does not issue shared locks when reading data.
	* Ignores exclusive locks when reading data.
* What You'll See
	- Committed and uncommitted data
	- High concurrency (doesn't block other transactions and isn't blocked by other transactions).
	- Good performance.
	- Dirty reads - Uncommitted changes.
	- Lost updates - Concurrent transactions updating the same data can cause data to be overwritten.
	- Non-repeatable reads - Concurrent transactions can modify resources between each other's statements.
	- Double rows - Updates to rows that move them around the table can cause them to be read twice by a concurrent read.
	- Phantom rows - Other transactions can insert rows between statements.

READ COMMITTED
* How it Works
	- Acquires shared locks when reading data (blocked by exclusive locks).
	- Row locks are released when the next row is read. ***
	- Page locks are released when the next page is read. ***
	- Table locks are released at the end of the statement.
* What You'll See
	- Committed data
	- Slightly dimished concurrency (minimal locking behavior).
	- Slightly dimished performance.
	- Lost updates.
	- Non-repeatable reads.
	- Double rows.
	- Phantom rows.
	
REPEATABLE READ
* How it Works
	- Aquires shared locks when reading data (blocked by exclusive locks).
	- Shared locks are released after the transaction completes.
* What You'll See
	- Committed data.
	- Blocks to updates/deletes on the same resources until the transaction completes.
	- Dimished concurrency (locks are held longer).
	- Dimished performance.
	- Phantom rows.
	
SERIALIZABLE
* How it Works
	- Acquires shared locks for the entire range of keys read by each statement in the transaction.
	- Shared locks are released after the transaction completes.
* What You'll See
	- Committed data.
	- Blocks to inserts/updates/deletes on the same resources until the transaction completes.
	- Significantly dimished concurrency (lots of locks held for the whole transaction).
	- Significantly diminished performance.
	
SNAPSHOT
* How it Works
	- As transactions read rows, a copy of the original is stored in tempdb with a version number.
	- Changes to the original data will cause the snapshot transaction to fail and rollback.
	- Optimistic reads/writes.
* What You'll See
	- Transaction-level consistency.
	- High concurrency.
	- Good performance when there are little to no update conflicts.
	- Terrible performance with many update conflicts (errors/rollbacks).

READ COMMITTED SNAPSHOT
* How it Works
	- Lighter weight row versioning.
	- Optimistic reads, pessimistic writes.
* What You'll See
	- Statement-level consistency.
	- Lost updates.
	- Non-repeatable reads.
	- Phantom rows.
