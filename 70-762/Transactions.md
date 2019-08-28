Transactions

# Transaction Concepts

## Resources

- <https://jimgray.azurewebsites.net/papers/theTransactionConcept.pdf>
- <http://www.sommarskog.se/error_handling/Part1.html>
- <https://docs.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide?view=sql-server-2017>

## Overview

A transaction is sequence of operations performed as a single unit of work. A common example is a transfer of funds from a checking account to a savings account.  This involves two operations, deducting from the checking, and adding to the savings.  The four characteristics of transactions are known as ACID:

-   **Atomic**: A transation must be an atomic unit of work, i.e. all or nothing.  In the bank transfer example, we would want both of the operations to succeed, or neither to succeed.
-   **Consistent**: A transaction takes the database from one consistent state to another consistent state, i.e. all data integrity, referential integrity, and other constraints are satisfied.  In the bank transfer example, a transaction that would result in a negative saving balance must fail.
-   **Isolated**: The effects of one transaction are isolated from all other transactions.  Also called serializabilty.  Serial execution is the easiest way to guarantee isolation, but results in low concurrency.  In the bank example, many customers should be able to make transfers simultaneously
-   **Durable**: The changes made by a transaction are permanently saved once the transaction complete, even if the database shuts down unexpectedly. In the bank transfer example, once the transaction is complete, the new savings and checking balances must be made permanent 

## Transactions in SQL

- The SQL `COMMIT` statement completes and persists a transation, 
- The `ROLLBACK` statement removes the transaction and restores the database to its prior state.  Most SQL engines will default to a `ROLLBACK` when an error occurs.  
- Savepoints allow the programmer to save parts of the transaction, and only rollback certain pieces on failure.

# SQL Server Transactions
## Explicit and Implicit transactions

-   Auto-commit - Any single statement that changes data is automatically an atomic transaction.  SQL Server performs a rollback if a failure occurs
-   Implicit
    -   `SET IMPLICIT_TRANSACTIONS ON` (Off by default)
    -   Transaction is automatically started when you execute `ALTER TABLE`, `BEGIN TRANSACTION`, `CREATE`, `DELETE`, `DROP`, `FETCH`, `GRANT`, `INSERT`, `OPEN`, `REVOKE`, `SELECT`, `TRUNCATE`, and `UPDATE`
    -   Transaction must be manually committed or rolled back
    -   Usually not a good idea
-   Explicit
    -   Transactions are started using `BEGIN TRANSACTION` and completed using `COMMIT/ROLLBACK TRANSACTION`
    -   `CREATE/DROP/ALTER DATBASE` cannot be used
    -   `CREATE/DROP/ALTER FULLTEXT CATALOG/INDEX` cannot be used
    -   `BACKUP/RECONFIGURE/RESTORE` cannot be used
    -   Transactions can be nested, although this only increases a counter, `@@TRANCOUT`
        - Inner transactions are not isolated
        - `COMMIT` statement decreases counter by 1.  Only when the counter goes from 1 to 0 is the transaction actually committed
        - `ROLLBACK` undoes the transaction and sets the counter to 0, regardless of it's initial value

## ACID Support

-   Atomicity - Supported via transaction control.
    -   `XACT_ABORT` - Set this to ON for more consistent behavior regarding transactions.  When ON, most errors will cause a transaction to abort and rollback.  Without this, you need to use try/catch to handle errors.  It's best practice to use both
-   Consistency - Supported via constraints
-   Isolation - Supported via locking and row-versioning mechanisms, based on isolation levels
    -   Default isolation level is `READ COMMITTED` (See [Isolation Levels](:/b29d1100e16d4e73828593a3e78afbb3) )
-   Durability 
    -   SQL Server guarantees full transaction durability by default via the transaction log.  This is a write-ahead log that
        -   Contains data changes, page allocations and de-allocations, and index changes, each marked by a Log Sequence Number(LSN) for the transaction
        -   Is written to disk when transaction comits or if the log buffer fills up
    -   On system failure, changes can always be recreated from transaction log if they have not yet been written to disk
    -   SQL Server also supports delayed durable transactions, i.e. lazy commits
        -   Increases throughput due to less contention for log IO
        -   Once a transaction is written to the log, SQL Server reports success
        -   Buffer is only flished to disk when it becomes full, or a buffer flsh event occurs(`sp_flush_log`)
        -   Tradeoff between potential data loss and reduced latency/contention
        -   Useful for OLAP, but not for OLTP

## Savepoints
- Save the changes made in a transaction using `SAVE TRANSACTION <name>`
- Use `ROLLBACK TRANSACTION <name>` to undo the changes made since the last savepoint with that name.  In this case, the rollback does not reset `@@TRANCOUNT`
