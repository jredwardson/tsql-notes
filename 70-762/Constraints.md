Constraints

# Constraints
## Types
- **Primary Key** - A uniqueness constraint that defines the primary key for a table
    - Cannot be null
    - Creates a unique clustered index by default
- **Foreign Key** - Used to enforce referential integrity.
    - Foreign key column values in the referencing table must correspond to column values in the referenced table
    - Key columns in a table that are referenced by a foreign key cannot be changed without an `ON UPDATE` cascade rule
    - Rows in a table that are referenced by a foreign key cannot be deleted with an `ON DELETE` cascade rule
    - Can be used to relate a table to itself to form a heirarchy
    - Can point to a Unique column as well as a primary key
- **Default** - Provides a value for a column when the user does not provide one
    - Can be used to suggest values
    - Can be used for fields like lastUpdateTime or LastUpdateUser
    - Useful for adding new `NOT NULL` columns to a table that already has data in it
- **Unique** - Implements uniqueness criteria on a column (these would be alternate/candidate keys)
    - Useful when using surrogate keys to enforce uniqueness on candidate keys
    - Nulls are allowed(but only one row can have a `NULL`, i.e. it uses distinctness based comparison)
    - Enforced using a unique index (non-clustered by default)
- **Check** - Apply a predicate to values in an insert or update statement
    - Validate format of input data, 
    - Enforce a specific range of values
    - Limit the data to a domain stricter than the data type
    - Coordinate multiple values together 

## Defining constraints
Create constraints at table creation time

```sql
----Write Transact-SQL statements to add constraints to tables
CREATE TABLE Examples.CreateTableExample
(
    --Uniqueness constraint referencing single column
    SingleColumnKey int NOT NULL CONSTRAINT PKCreateTableExample PRIMARY KEY,

    --Uniqueness constraint in separate line
    TwoColumnKey1 int NOT NULL,
    TwoColumnKey2 int NOT NULL,
    CONSTRAINT AKCreateTableExample UNIQUE (TwoColumnKey1, TwoColumnKey2),

    --CHECK constraint declare as column constraint
    PositiveInteger int NOT NULL 
         CONSTRAINT CHKCreateTableExample_PostiveInteger CHECK (PositiveInteger > 0),
    --CHECK constraint that could reference multiple columns
    NegativeInteger int NOT NULL,
    CONSTRAINT CHKCreateTableExample_NegativeInteger CHECK (NegativeInteger > 0),
    --FOREIGN KEY constraint inline with column
    FKColumn1 int NULL CONSTRAINT FKColumn1_ref_Table REFERENCES Tbl (TblId),
    --FOREIGN KEY constraint… Could reference more than one columns
    FKColumn2 int NULL,
    CONSTRAINT FKColumn2_ref_Table FOREIGN KEY (FKColumn2) REFERENCES Tbl (TblId)
);
GO
```

Create constraints after table definition.  This will fail if a new unique or primary key constraints is violated by existing data.
```sql
ALTER TABLE Examples.CreateTableExample
    DROP PKCreateTableExample;
GO

ALTER TABLE Examples.CreateTableExample
    ADD CONSTRAINT PKCreateTableExample PRIMARY KEY (SingleColumnKey);
GO
```

Can use `NOCHECK` option to create `CHECK` constraints that are violated by existing data.  However, these are treated as 'not-trusted' by the engine, and are thus not used in query optimization
```sql
ALTER TABLE Examples.BadData
   ADD CONSTRAINT CHKBadData_PostiveValue CHECK(PositiveValue > 0);
GO

ALTER TABLE Examples.BadData WITH NOCHECK
   ADD CONSTRAINT CHKBadData_PostiveValue CHECK(PositiveValue > 0);
GO
```

## Identify effects of DML given constraints
See the example following, and be aware that constraint violations are statement terminating errors.  In this case, one row would be inserted into the ScenarioTestType table, and two rows in the ScenarioTest table.  The first and fourth inserts will fail due to constraint violations.  The last insert will fail due to a binding error(typo in table name).
```sql
CREATE TABLE Examples.ScenarioTestType
(
    ScenarioTestType varchar(10) NOT NULL CONSTRAINT PKScenarioTestType PRIMARY KEY
);
GO

CREATE TABLE Examples.ScenarioTest
(
    ScenarioTestId int NOT NULL PRIMARY KEY,
    ScenarioTestType varchar(10) NULL CONSTRAINT CHKScenarioTest_ScenarioTestType 
                                       CHECK (ScenarioTestType IN ('Type1','Type2'))
);
GO

ALTER TABLE Examples.ScenarioTest
   ADD CONSTRAINT FKScenarioTest_Ref_ExamplesScenarioTestType
       FOREIGN KEY (ScenarioTestType) REFERENCES Examples.ScenarioTestType;
GO

SET XACT_ABORT OFF 

INSERT INTO Examples.ScenarioTest(ScenarioTestId, ScenarioTestType)
VALUES (1,'Type1');
INSERT INTO Examples.ScenarioTestType(ScenarioTestType)
VALUES ('Type1');
INSERT INTO Examples.ScenarioTest(ScenarioTestId, ScenarioTestType)
VALUES (1,'Type1');
INSERT INTO Examples.ScenarioTest(ScenarioTestId, ScenarioTestType)
VALUES (1,'Type2');
INSERT INTO Examples.ScenarioTest(ScenarioTestId, ScenarioTestType)
VALUES (2,'Type1');
INSERT INTO Examples.ScenarioTests(ScenarioTestId, ScenarioTestType)
VALUES (3,'Type1');
```

## Primary keys
In many cases, a surrogate key is a good choice for primary keys, as the natural key may have too many fields, or be a large data type.  Options are
- Identity columns
- Sequence objects
- Guid