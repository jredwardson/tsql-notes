Design and implement a relational database schema

# Resources

<https://docs.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql?view=sql-server-2017>
<https://docs.microsoft.com/en-us/sql/t-sql/data-types/data-types-transact-sql?view=sql-server-2017>
<https://www.studytonight.com/dbms/database-normalization.php>
<https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms187099(v=sql.105)>

# Database Design Process

1.  Collect business requirements.  Example Scenario:
>    We have 1,000 computers, comprised of laptops, workstations, and  tablets. Each computer has items associated with it, which we will list as mouse, keyboard, etc. Each computer has a tag number associated with it, and is tagged on each device with a tag graphic that can be read by tag readers manufactured by “Trey Research” (<http://www.treyresearch.net/>) or “Litware, Inc” (<http://www.litwareinc.com/>). Of course tag numbers are unique across tag readers. We don’t know which employees are assigned which computers, but all computers that cost more than $300 are inventoried for the first three years after purchase using a different software system. Finally, employees need to have their names recorded, along with their employee number in this system
2.  Look for key words and phrases to define an initial set of schemas, tables and attributes
    -   Equipment
        -   Computer
            -   Type
            -   Item list(peripherals)
            -   Tag
            -   Tag Company
            -   Tag Company URL
            -   Cost
            -   Employee assignment
            -   Purchase date (for inventory purposes)
    -   HumanResources
        -   Employee
            -   Name
            -   Employee number
3.  Determine natural keys 
    -   Tag Number
    -   Employee number
4.  Improve design using normalization
    -   Equipment
        -   Computer
            -   Tag (key, ref to Tag)
            -   ComputerType
            -   ComputerCost
            -   PurchaseDate
            -   AssignedEmployee (ref to Employee)
        -   TagCompany
            -   TagCompany (key)
            -   TagCompanyURL
        -   Tag
            -   Tag (key)
            -   TagCompany (ref to TagCompany)
        -   ComputerAssociatedItem
            -   Tag (ref to Computer)
            -   AssociatedItem
            -   (key Tag, AssociatedItem)
    -   HumanResources
        -   Employee
            -   EmployeeNumber [key]
            -   LastName
            -   FirstName
5.  Determine data types
6.  Translate tables to SQL
```sql
        CREATE SCHEMA Equipment;
        GO
        
        CREATE SCHEMA HumanResources;
        GO
        
        CREATE TABLE TagCompany (
          TagCompany VARCHAR(10) NOT NULL
            CONSTRAINT PK_TagCompany PRIMARY KEY,
          TagCompanyURL VARCHAR(50) 
        );
        
        CREATE TABLE Tag (
          Tag VARCHAR(10) NOT NULL
            CONSTRAINT PK_Tag_Tag PRIMARY KEY,
          TagCompany VARCHAR(10) NOT NULL FOREIGN KEY References TagCompany(TagCompany)
        );
```
# Normal Forms

-   1st Normal Form
    -   All column values must be atomic values
    -   Column values must be of the same domain
    -   All columns must have unique names
    -   Order of data does not matter
-   2nd Normal Form
    -   All attributes must be a fact about the entire primary key
-   3rd Normal Form
    -   All attribute must be a fact about the entire primary key, and not any non-primary key attributes
-   Boyce-Codd Normal Form
    -   Every candidate key is identified, all attributes are fully dependent on a key, and all columns must identify a fact about a key and nothing buy a key

# Data Types

-   Precise Numeric
    -   bit - 0/1/NULL
    -   tinyint - 1 byte (
    -   smallint - 2 bytes
    -   int - 4 bytes
    -   bigint - 8 bytes
    -   decimal/numeric(Precision,Scale) - Precision is number of digits, scale is number of decimal places.  5-17 bytes, depending on precision
    -   money - 8 byte monetary values
    -   smallmoney - 4 byte monetary values
-   Approximate numeric
    -   float(N)
    -   real - float(24)
-   Date/Time
    -   date - January 1, 0001 to December 31, 9999 (3 Bytes)
    -   time(N) - time of day with N fractional parts of a second (3-5 Bytes)
    -   datetime2(N) - date and time with N fractional parts of a second (6-8 Bytes)
    -   datetimeoffset - includes timezone offset (8-10 Bytes)
    -   smalldatetime - Jan 1, 1990 - June 6, 2079, accurate to 1 minute (4 bytes)
    -   datetime - Jan 1, 1753 to December 31, 9999, accurate to 3.33 milliseconds (8 bytes)
-   Binary data
    -   binary(N) - Fixed length binary up to 8000 B
    -   varbinary(N) - variable-length binary data up to 8000 B
    -   varbinary(max) - variable-length binary data up to 2GB
-   Character (or string) data
    -   char(N) - fixed-length, up to 8000 B
    -   varchar(N) - variable-length, up to 8000 B
    -   varchar(max) - variable-length, up to 2 GB
    -   nchar, nvarchar, nvarchar(max) - unicode character strings.  Use this type for most flexibilty
-   Other data types
    -   sql<sub>variant</sub>
    -   rowversion
    -   uniqueidentifier - GUID
    -   XML
    -   Spatial types (geometry, geography, circularString, compountCurve, curvePolygon)
    -   heirarchyId

## Computed Columns

Allows you to manifest an expression as a column for usage.  Typical use cases is to maintain values that would not meet normalization rules, e.g. a full name column.  If the expression is deterministic, the computed value can be persisted
```sql
    CREATE TABLE Examples.ComputedColumn
    (
      FirstName nvarchar(50) NULL,
      LastName nvarchar(50) NOT NULL,
      FullName AS CONCAT(LastName, ', ' + FirstName) PERSISTED
    );
```

## Dynamic data masking

Allows you to mask a column from the view of a user.  Types of data mask functions

-   Default - Default mask for the data type
-   Email - Masks email so you only see a few characters
-   Random - Masks any of the numeric types with a random value within a range
-   Partial - Takes values from front and back, replacing center with a fixed string

Database level permission called UNMASK controls who can see masked/unmasked data
```sql
     CREATE TABLE Examples.DataMasking (      
       FirstName    nvarchar(50) MASKED WITH (FUNCTION = 'default()') NULL,      
       LastName    nvarchar(50) NOT NULL,      
       PersonNumber char(10) 
         MASKED WITH (FUNCTION = 'partial(2, "*****", 1)') NOT NULL,      
       Status    varchar(10), 
         MASKED WITH (FUNCTION = 'partial(0, "Unknown", 0)') NOT NULL
       EmailAddress nvarchar(50) 
         MASKED WITH (FUNCTION = 'email()') NULL, 
       BirthDate date MASKED WITH (FUNCTION = 'default()') NOT NULL,
       CarCount   tinyint 
         MASKED WITH (FUNCTION = 'random(1,3)') NOT NULL
    );
    
    INSERT INTO Examples.DataMasking
    
    
    CREATE USER MaskedView WITHOUT LOGIN;
    GRANT SELECT ON Examples.DataMasking TO MaskedView;
    
    EXECUTE AS USER = 'MaskedView';
    SELECT * FROM Examples.DataMasking
    
    REVERT; SELECT USER_NAME();
```    