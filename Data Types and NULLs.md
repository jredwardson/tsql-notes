## ISNULL vs COALESCE

Aspect               | ISNULL     | COALESCE
------               |-------     |----------
Number of parameters | 2          | >= 2
Standard             | NO         | Yes
Data Type of result  | Type of first, then type of second, or INT if both are untyped NULL | Type with the highest precedence.  Error if all inputs are Untyped NULL
Nullability          | If any input is non-nullable, result is NOT NULL | If all inputs are non-nullable, result is NOT NULL
Might execute subquery more than once?  | No | Yes

## Data Types
Use the smallest type that fits your needs!
### Exact numeric
- BIGINT (8 Bytes)
- INT (4 Bytes)
- SMALLINT (2 Bytes)
- TINYINT (1 Byte)
- BIT (1 Bit)
- DECIMAL/NUMERIC (p, s)
    - P(precision) is total number of digits
    - S(Scale) is number of digits to right of decimal place
- MONEY (8 Bytes) and SMALLMONEY (4 Bytes)
    - Accurate to 4 decimal places

### Approximate Numeric
- FLOAT(n) - n is effectively 24 (7 digits/4 Bytes) or 53(15 digits/8 Bytes).
- REAL === FLOAT(24)
- Can represent a larger range of values, but use exact numeric types when precision is needed

### String and Binary data
- BINARY(n|max) and VARBINARY(n|max)
    - n is number of bytes (1-8000).  Max uses 2GB max for storage
- CHAR(n|max) and VARCHAR(n|max)
    - n is number of bytes (1-8000).  Max uses 2GB max for storage
- NCHAR(n) and NVARCHAR(n)
    - Used for Unicode string
    - n is number of bytes (1-4000).  Max uses 2GB max for storage
- Use fixed width when size doesn't change much, and/or where update performance is a priority
- Use variable width when size changes significantly, and/or read performance is a priority

### Date and Time
- DATE - 3 Bytes, accurate to 1 day
- TIME - Max 5 Bytes, accurate to 100ns by default
- DATETIME2 - 6-8 Bytes depending on precision.  Accurate to 100ns
- DATETIMEOFFSET - 10 Bytes, accurate to 100ns.  Time zone aware
- DATETIME - 8 Bytes
- SMALLDATETIME - 4 Bytes, accurate to 1 minute

### Others
- XML
- UNIQUEIDENTIFIER - 16 Bytes.  Generate using NEWID for GUID or NEWSEQUENTIALID for sequential GUIDS

## Data Types for Keys
- Identity column property
- Sequence Object
- Nonsequential GUIDS
- Sequential GUIDS

Size of the key type can have a cascading effect on sizes of indexes

Sequential keys result in less fragmentation, but potential performance problems when loading data from multiple sessions.  When using a high-end storage subsystem and loading data from multiple sessions, nonsequential keys can give better perfomance
