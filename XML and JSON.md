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