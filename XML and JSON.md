## XML
### Concepts
- XML documents are widely used for data exchange
- XML is case-sensitive Unicode text
- XML mixes metadata(elements and attributes) with data
- Tags are used to name part of the XML document.  These parts are called *elements* 
    - Every begin tag must have a corresponding end tag
    - Elements can have *attributes*
- XML documents are ordered
- Namespaces can be used to distinguish elements with same name that are used in different areas
- XML Schema Description(XSD) is most commonly used standard for describing the metadata of an XML document

### Producing XML in queries
- FOR XML
    - RAW
    - AUTO
    - ELEMENTS
    - ROOT
- WITH XMLNAMESPACES
- A proper ORDER BY clause is important
- Order of columns in SELECT influences the XML returned 

Example FOR XML RAW
```sql
SELECT Customer.custid, Customer.companyname, 
  [Order].orderid, [Order].orderdate
FROM Sales.Customers AS Customer
  INNER JOIN Sales.Orders AS [Order]
    ON Customer.custid = [Order].custid
WHERE Customer.custid <= 2
  AND [Order].orderid %2 = 0
ORDER BY Customer.custid, [Order].orderid
FOR XML RAW;
```
Produces
```xml
<row custid="1" companyname="Customer NRZBB" orderid="10692" orderdate="2015-10-03" />
<row custid="1" companyname="Customer NRZBB" orderid="10702" orderdate="2015-10-13" />
<row custid="1" companyname="Customer NRZBB" orderid="10952" orderdate="2016-03-16" />
<row custid="2" companyname="Customer MLTDN" orderid="10308" orderdate="2014-09-18" />
<row custid="2" companyname="Customer MLTDN" orderid="10926" orderdate="2016-03-04" />
```

Example FOR XML AUTO
```sql
WITH XMLNAMESPACES('ER70761-CustomersOrders' AS co)
SELECT [co:Customer].custid AS [co:custid], 
  [co:Customer].companyname AS [co:companyname], 
  [co:Order].orderid AS [co:orderid], 
  [co:Order].orderdate AS [co:orderdate]
FROM Sales.Customers AS [co:Customer]
  INNER JOIN Sales.Orders AS [co:Order]
    ON [co:Customer].custid = [co:Order].custid
WHERE [co:Customer].custid <= 2
  AND [co:Order].orderid %2 = 0
ORDER BY [co:Customer].custid, [co:Order].orderid
FOR XML AUTO, ELEMENTS, ROOT('CustomersOrders');
```
Produces
```xml
<CustomersOrders xmlns:co="ER70761-CustomersOrders">
  <co:Customer>
    <co:custid>1</co:custid>
    <co:companyname>Customer NRZBB</co:companyname>
    <co:Order>
      <co:orderid>10692</co:orderid>
      <co:orderdate>2015-10-03</co:orderdate>
    </co:Order>
    <co:Order>
      <co:orderid>10702</co:orderid>
      <co:orderdate>2015-10-13</co:orderdate>
    </co:Order>
    <co:Order>
      <co:orderid>10952</co:orderid>
      <co:orderdate>2016-03-16</co:orderdate>
    </co:Order>
  </co:Customer>
  <co:Customer>
    <co:custid>2</co:custid>
    <co:companyname>Customer MLTDN</co:companyname>
    <co:Order>
      <co:orderid>10308</co:orderid>
      <co:orderdate>2014-09-18</co:orderdate>
    </co:Order>
    <co:Order>
      <co:orderid>10926</co:orderid>
      <co:orderdate>2016-03-04</co:orderdate>
    </co:Order>
  </co:Customer>
</CustomersOrders>
```

### Shredding XML
Process of converting XML to relational tables.

OPENXML rowset function
- XML DOM handle returned by `sp_xml_preparedocument`
- XPath expression to find the nodes you want
- Rowset description(schema)
- Mapping between XML nodes and rowset columns

```sql
DECLARE @DocHandle AS INT;
DECLARE @XmlDocument AS NVARCHAR(1000);
SET @XmlDocument = N'
<CustomersOrders>
  <Customer custid="1">
    <companyname>Customer NRZBB</companyname>
    <Order orderid="10692">
      <orderdate>2015-10-03T00:00:00</orderdate>
    </Order>
    <Order orderid="10702">
      <orderdate>2015-10-13T00:00:00</orderdate>
    </Order>
    <Order orderid="10952">
      <orderdate>2016-03-16T00:00:00</orderdate>
    </Order>
  </Customer>
  <Customer custid="2">
    <companyname>Customer MLTDN</companyname>
    <Order orderid="10308">
      <orderdate>2014-09-18T00:00:00</orderdate>
    </Order>
    <Order orderid="10926">
      <orderdate>2016-03-04T00:00:00</orderdate>
    </Order>
  </Customer>
</CustomersOrders>';
-- Create an internal representation
EXEC sys.sp_xml_preparedocument @DocHandle OUTPUT, @XmlDocument;
-- Attribute- and element-centric mapping
-- Combining flag 8 with flags 1 and 2
SELECT *
FROM OPENXML (@DocHandle, '/CustomersOrders/Customer',11)
     WITH (custid INT,
           companyname NVARCHAR(40));
-- Remove the DOM
EXEC sys.sp_xml_removedocument @DocHandle;
GO
```
### XQuery
Standard language for browsing XML instances and returing XML output. 

FLWOR expressions - used to iterate through a sequence returned by an XPath expression
- For - bind iterator variables to input sequence
- Let - Assign a value to a variable for a specific iteration
- Where - Filter the iteration
- Order by - Control the order in which the input sequence is processed
- Return - Format the resulting XML

Reference: https://docs.microsoft.com/en-us/sql/xquery/xquery-language-reference-sql-server?view=sql-server-2017

### XML Data type
The XML data type is a large object type(up to 2GB).  It supports the following methods
- `value()` - Returns a scalar value based on an XQuery expression
- `query()` - Returns XML based on XQuery expression
- `exists()` - Returns a bit if the xQuery expression returns a nonempty result
- `modify()` - Supports modification of XML data using SQL Server DML extensions to xQuery
- `nodes()` - Used to shred an XML value into relational data

## JSON
### Producing JSON from queries
FOR JSON <AUTO | PATH>
- AUTO - SQL Server formats the JSON automatically based on order of columns in SLELCT and order of tables in FROM
- PATH - 
- ROOT
- INCLUDE_NULL_VALUES
- WITHOUT_ARRAY_WRAPPER

```sql
SELECT c.custid AS [Customer.Id], 
  c.companyname AS [Customer.Name], 
  o.orderid AS [Customer.Order.Id], 
  o.orderdate AS [Customer.Order.Date]
FROM Sales.Customers AS c
  INNER JOIN Sales.Orders AS o
    ON c.custid = o.custid
WHERE c.custid = 1
  AND o.orderid = 10692
ORDER BY c.custid, o.orderid
FOR JSON PATH,
    ROOT('Customer 1');
```

```json
{
   "Customer":{
      "Id":1,
      "Name":"Customer NRZBB",
      "Order":{
         "Id":10692,
         "Date":"2015-10-03",
         "Delivery":null
      }
   }
}
```

```sql
SELECT c.custid AS [Customer.Id], 
  c.companyname AS [Customer.Name], 
  o.orderid AS [Customer.Order.Id], 
  o.orderdate AS [Customer.Order.Date],
  NULL AS [Customer.Order.Delivery]
FROM Sales.Customers AS c
  INNER JOIN Sales.Orders AS o
    ON c.custid = o.custid
WHERE c.custid = 1
  AND o.orderid = 10692
ORDER BY c.custid, o.orderid
FOR JSON PATH,
    WITHOUT_ARRAY_WRAPPER,
    INCLUDE_NULL_VALUES;
```

```json
{
   "Customer":{
      "Id":1,
      "Name":"Customer NRZBB",
      "Order":{
         "Id":10692,
         "Date":"2015-10-03",
         "Delivery":null
      }
   }
}
```

### Converting JSON to tabular format
`OPENJSON(value, [path])`

```sql
DECLARE @json AS NVARCHAR(MAX) = N'
{ 
   "Customer":{ 
      "Id":1, 
      "Name":"Customer NRZBB",
      "Order":{ 
         "Id":10692, 
         "Date":"2015-10-03",
         "Delivery":null
      }
   }
}';
SELECT *
FROM OPENJSON(@json,'$.Customer');
```
key | value | type
--- | ----- | ----
Id	| 1|	2
Name	| Customer NRZBB |	1
Order |	{            "Id":10692,            "Date":"2015-10-03",           "Delivery":null        }	| 5

WITH Clause allows explicit schema mapping
```sql
DECLARE @json AS NVARCHAR(MAX) = N'
{ 
   "Customer":{ 
      "Id":1, 
      "Name":"Customer NRZBB",
      "Order":{ 
         "Id":10692, 
         "Date":"2015-10-03",
         "Delivery":null
      }
   }
}';
SELECT *
FROM OPENJSON(@json)
WITH
(
 CustomerId   INT           '$.Customer.Id',
 CustomerName NVARCHAR(20)  '$.Customer.Name',
 Orders       NVARCHAR(MAX) '$.Customer.Order' AS JSON
);
```

CustomerId	| CustomerName	| Orders
----------  | ------------ | -------
1	| Customer NRZBB	|{            "Id":10692,            "Date":"2015-10-03",           "Delivery":null        }

### Other JSON functions
- JSON_VALUE(val, path) - Returns scalar JSON value
- JSON_QUERY(val, path) - Return JSON object 
- JSON_MODIFY(val, path, newval) - Modify JSON
- ISJSON(val) - Returns 1 if val is valid json, 0 otherwise

```sql
DECLARE @json AS NVARCHAR(MAX) = N'
{ 
   "Customer":{ 
      "Id":1, 
      "Name":"Customer NRZBB",
      "Order":{ 
         "Id":10692, 
         "Date":"2015-10-03",
         "Delivery":null
      }
   }
}';
SELECT JSON_VALUE(@json, '$.Customer.Id') AS CustomerId,
 JSON_VALUE(@json, '$.Customer.Name') AS CustomerName,
 JSON_QUERY(@json, '$.Customer.Order') AS Orders;
```

```sql
DECLARE @json AS NVARCHAR(MAX) = N'
{ 
   "Customer":{ 
      "Id":1, 
      "Name":"Customer NRZBB",
      "Order":{ 
         "Id":10692, 
         "Date":"2015-10-03",
         "Delivery":null
      }
   }
}'; 
-- Update name  
SET @json = JSON_MODIFY(@json, '$.Customer.Name', 'Modified first name'); 
-- Delete Id  
SET @json = JSON_MODIFY(@json, '$.Customer.Id', NULL)  
-- Insert last name  
SET @json = JSON_MODIFY(@json, '$.Customer.LastName', 'Added last name')  
PRINT @json;
```
```json
{ 
   "Customer":{ 
       
      "Name":"Modified first name",
      "Order":{ 
         "Id":10692, 
         "Date":"2015-10-03",
         "Delivery":null
      }
   ,"LastName":"Added last name"}
}
```