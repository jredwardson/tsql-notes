# Deadlocking

A deadlock occurs when two or more tasks permanently block each other by each task having a lock on a resource which the other tasks are trying to lock. For example:
- Transaction A acquires a share lock on row 1.
- Transaction B acquires a share lock on row 2.
- Transaction A now requests an exclusive lock on row 2, and is blocked until transaction B finishes and releases the share lock it has on row 2.
- Transaction B now requests an exclusive lock on row 1, and is blocked until transaction A finishes and releases the share lock it has on row 1

This condition is known as a cyclic dependency.  A depends on B continue, but B depends on A to continue.

When a deadlock occurs, SQL Server will kill one of the processes with error 1250 

## Lock DMVs
- `sys.dm_tran_locks` - view all current locks, resources, modes, etc
- `sys.dm_os_waiting_tasks` - See which tasks are waiting for a resource
- `sys.dm_os_wait_stats` - See how often processes are waiting on locks

## Capturing deadlock info
SQL Server profiler or Extended events to capture a deadlock graph

Trace flags 1204 and 1222