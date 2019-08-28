## Transactions
A *transaction* defines a unit of work against the database such that all statements in the transaction either succeed or all fail.  Some examples of where transactions are useful:    
- You need to add data to a Sales Order table and a Sales Order Lines table.  If there are any error in either insert, you want to ensure that no data is added to either table
- You need to deduct money from a Checking Account Table and Add it to a Savings account.  In this case, you want to ensure that both operations either succeed or fail together so the accounts stay in a consistent state.

Transactional statements in TSQL include
- DML(Data Modification Language) statements like `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE TABKE`, `MERGE`    
- Most DDL(Data Definition Language) statements like `CREATE`, `DROP`, `ALTER TABLE`

Transaction Modes in TSQL
- Autocommit mode - Any time a transactional statement is issued, a transaction is started and committed.  So if only you are modifying a single table, no need to specify transactions
- Explicit transactions - Transactions that you control using `BEGIN TRAN`, `COMMIT TRAN`, or `ROLLBACK TRAN` Statements
- Implicit transactions - Enable using `SET IMPLICIT_TRANSACTIONS ON`.  In this mode, a transaction is automatically created any time you issue a transactional statement.  However, you have to remember to commit or rollback that transaction, or it will be left open.  For this reason, you generally want to avoid this mode

Explicit Transactions
- Use `BEGIN TRANSACTION` or `BEGIN TRAN` to start a transaction.
- Use `COMMIT TRANSACTION` or `COMMIT TRAN` to complete the transaction
- USE `ROLLBACK TRANSACTION` or `ROLLBACK TRAN` to abort and undo the transaction
- Use `@@TRANCOUNT` function to investigate current transaction status.  If greater than 0, then you are inside a transaction.

Nested Transactions
- Each time `BEGIN TRAN` is invoked, `@@TRANCOUNT` is increased by 1
- Each time `COMMIT TRAN` is invoked, `@@TRANCOUNT` is decreased by 1.  The transaction is only really committed if this causes `@@TRANCOUNT` to go from 1 to 0
- `ROLLBACK TRAN` aborts all nested transaction and sets `@@TRANCOUNT` to 0

## Throw vs RAISERROR

Property           | THROW  | RAISERROR
------------------ | ------ | ----------
Can rethrow original system error | Y | N
Activates catch block             | Y | Y for 10 < severity < 20
Always aborts batch when not using TRY-CATCH | Y | N
Aborts/dooms transactions with XACT_ABORT OFF | N | N
Aborts/dooms transactions with XACT_ABORT ON | Y | N
If error number is passed, it must be defined in sys.messages | N | Y
Supports printf parameter markers directly | N | Y
Supports indicating severity | N (always sev 16) | Y
Supports WITH LOG to log error to error and application logs | N | Y
Supports NOWAIT to send messages immediately to the client | N | Y
Preceding statement needs to be terminated | Y | N
