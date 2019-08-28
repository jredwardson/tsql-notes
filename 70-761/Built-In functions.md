## Type Conversion
- CAST(source_expression AS target_type)
- CONVERT(target_type, source_expression, style_number)
- PARSE(expr AS type USING style)
- TRY_CAST, TRY_CONVERT, TRY_PARSE - return null on failure
- FORMAT(source, fmt_string) - returns a character string based on a .Net format string

## Date and time functions
### Current date/time
- GETDATE() - Current date time in system timezone, returns DATETIME type
- CURRENT_TIMESTAMP() - Same as GETDATE, but standard
- GETUTCDATE() - Gets current UTC time, returns DATETIME type
- SYSDATETIME() - Returns current datetime in system timezone, returning DATETIME2 type
- SYSUTCDATETIME() - Returns current UTC datetime, returns DATETIME2 type
- SYSDATETIMEOFFSET() - Returns current time using DATETIMEOFFSET data type
### Date and time parts
- DATEPART(part, date)
- YEAR(date)
- MONTH(date)
- DAY(date)
- DATENAME(part, date)
- DATEFROMPARTS
- DATETIMEFROMPARTS
- DATETIME2FROMPARTS
- DATETIMEOFFSETFROMPARTS
- SMALLDATETIMEFROMPARTS
- TIMEFROMPARTS
- EOMONTH(datetime)
### Add and Diff
- DATEADD(interval, number, date)
- DATEDIFF(interval, date1, date2)
### Offset
- SWITCHOFFSET(datetimeoffset, desired offset) - Adjusts a DT offset to the desired offset
- TODATETIMEOFFSET - Construct a DT offset from a date/time plus an offset
- AT TIME ZONE - Uses named time zones instead of offsets(so you don't have to worry about Daylight savings)
