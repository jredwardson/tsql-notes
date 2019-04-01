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
