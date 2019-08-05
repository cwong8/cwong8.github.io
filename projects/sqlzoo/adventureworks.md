---
layout: page
title: Adventure Works
permalink: /projects/sqlzoo/adventureworks
---

# [Adventure Works](https://sqlzoo.net/wiki/AdventureWorks)

### AdventureWorks Easy Questions

1. Show the first name and the email address of customer with CompanyName 'Bike World'

```sql
SELECT FirstName, EmailAddress
FROM Customer
WHERE CompanyName = 'Bike World';
```

2\. Show the CompanyName for all customers with an address in City 'Dallas'.

```sql
SELECT Customer.CompanyName
FROM Customer JOIN CustomerAddress ON Customer.CustomerID = CustomerAddress.CustomerID
  JOIN Address ON CustomerAddress.AddressID = Address.AddressID
WHERE Address.City = 'Dallas';
```

3\. How many items with ListPrice more than $1000 have been sold? 

```sql
SELECT COUNT(*)
FROM Product JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
WHERE Product.ListPrice > 1000;
```

4\. Give the CompanyName of those customers with orders over $100000. Include the subtotal plus tax plus freight.

```sql
SELECT Customer.CompanyName,
       SalesOrderHeader.SubTotal,
       SalesOrderHeader.TaxAmt,
       SalesOrderHeader.Freight
FROM Customer JOIN SalesOrderHeader ON Customer.CustomerID = SalesOrderHeader.CustomerID
WHERE SalesOrderHeader.SubTotal + SalesOrderHeader.TaxAmt + SalesOrderHeader.Freight > 100000;
```

5\. Find the number of left racing socks ('Racing Socks, L') ordered by CompanyName 'Riding Cycles'

```sql
SELECT SalesOrderDetail.OrderQty
FROM Product JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
  JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
  JOIN Customer ON SalesOrderHeader.CustomerID = Customer.CustomerID
WHERE Product.Name = 'Racing Socks, L' AND Customer.CompanyName = 'Riding Cycles'
```

### AdventureWorks Medium Questions

6\. A "Single Item Order" is a customer order where only one item is ordered. Show the SalesOrderID and the UnitPrice for every Single Item Order.

```sql
SELECT SalesOrderHeader.CustomerID, SUM(SalesOrderDetail.OrderQty) AS amount_ordered
FROM SalesOrderDetail JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
GROUP BY SalesOrderHeader.CustomerID, SalesOrderHeader.SalesOrderID
HAVING SUM(SalesOrderDetail.OrderQty) = 1;
```

7\. Where did the racing socks go? List the product name and the CompanyName for all Customers who ordered ProductModel 'Racing Socks'.

```sql
SELECT Product.ProductModelID, Product.Name, Customer.CompanyName, SalesOrderDetail.OrderQty
FROM Product JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
  JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
  JOIN Customer ON SalesOrderHeader.CustomerID = Customer.CustomerID
WHERE Product.ProductModelID = 
  (SELECT ProductModelID
  FROM ProductModel
  WHERE Name = 'Racing Socks')
```

8\. Show the product description for culture 'fr' for product with ProductID 736. 

```sql
SELECT Product.ProductID, ProductModelProductDescription.Culture, ProductDescription.Description
FROM ProductDescription JOIN ProductModelProductDescription ON ProductDescription.ProductDescriptionID = ProductModelProductDescription.ProductDescriptionID
  JOIN ProductModel ON ProductModelProductDescription.ProductModelID = ProductModel.ProductModelID
  JOIN Product ON ProductModel.ProductModelID = Product.ProductModelID
WHERE ProductModelProductDescription.Culture = 'fr' AND Product.ProductID = 736
```

9\. Use the SubTotal value in SaleOrderHeader to list orders from the largest to the smallest. For each order show the CompanyName and the SubTotal and the total weight of the order.

```sql
SELECT Customer.CompanyName, SalesOrderHeader.SubTotal, a.total_weight
FROM SalesOrderHeader JOIN Customer ON SalesOrderHeader.CustomerID = Customer.CustomerID
  JOIN
(SELECT SalesOrderDetail.SalesOrderID,
  SUM(COALESCE(Product.Weight, 0) * SalesOrderDetail.OrderQty) AS total_weight
FROM Product JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
GROUP BY SalesOrderID) AS a ON SalesOrderHeader.SalesOrderID = a.SalesOrderID
ORDER BY SalesOrderHeader.SubTotal DESC;
```

10\. How many products in ProductCategory 'Cranksets' have been sold to an address in 'London'?

```sql
SELECT Address.AddressID, ProductCategory.Name, SalesOrderDetail.OrderQty
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
  JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
  JOIN Address ON SalesOrderHeader.BillToAddressID = Address.AddressID AND SalesOrderHeader.ShipToAddressID = Address.AddressID
WHERE ProductCategory.Name = 'Cranksets' AND Address.City = 'London'
```

### Adventure Works Hard Questions

11\. For every customer with a 'Main Office' in Dallas show AddressLine1 of the 'Main Office' and AddressLine1 of the 'Shipping' address - if there is no shipping address leave it blank. Use one row per customer.

```sql
SELECT main.CustomerID, main.MainOffice, shipping.Shipping
FROM
(SELECT Customer.CustomerID, Address.AddressLine1 AS MainOffice
FROM Customer JOIN CustomerAddress ON Customer.CustomerID = CustomerAddress.CustomerID
  JOIN Address ON CustomerAddress.AddressID = Address.AddressID
WHERE CustomerAddress.AddressType = 'Main Office' AND Address.City = 'Dallas') AS main
LEFT JOIN
(SELECT Customer.CustomerID, Address.AddressLine1 AS Shipping
FROM Customer JOIN CustomerAddress ON Customer.CustomerID = CustomerAddress.CustomerID
  JOIN Address ON CustomerAddress.AddressID = Address.AddressID
WHERE CustomerAddress.AddressType = 'Shipping') AS shipping
ON main.CustomerID = shipping.CustomerID
```

12\. For each order show the SalesOrderID and SubTotal calculated three ways:
  A) From the SalesOrderHeader
  B) Sum of OrderQty*UnitPrice
  C) Sum of OrderQty*ListPrice 

```sql
SELECT SalesOrderDetail.SalesOrderID, SalesOrderHeader.SubTotal,
  SUM(SalesOrderDetail.OrderQty * SalesOrderDetail.UnitPrice),
  SUM(SalesOrderDetail.OrderQty * Product.ListPrice)
FROM SalesOrderHeader JOIN SalesOrderDetail ON SalesOrderHeader.SalesOrderID = SalesOrderDetail.SalesOrderID
  JOIN Product ON SalesOrderDetail.ProductID = Product.ProductID
GROUP BY SalesOrderDetail.SalesOrderID, SalesOrderHeader.SubTotal
```

13\. Show the best selling item by value. 

I don't think we sort by SalesOrderHeader.SubTotal because a sales order can contain multiple items.

```sql
SELECT Product.ProductID,
  SUM(SalesOrderDetail.OrderQty * (SalesOrderDetail.UnitPrice - SalesOrderDetail.UnitPriceDiscount)) AS total_sales
FROM SalesOrderHeader JOIN SalesOrderDetail ON SalesOrderHeader.SalesOrderID = SalesOrderDetail.SalesOrderID
  JOIN Product ON SalesOrderDetail.ProductID = Product.ProductID
GROUP BY ProductID
ORDER BY total_sales DESC
```

14\. Show how many orders are in the following ranges (in $):

    RANGE      Num Orders      Total Value
    0-  99
  100- 999
 1000-9999
10000-

Pretty sure there's an easier way to do this, but I don't know how to using MySQL.

```sql
SELECT *
FROM
(SELECT COUNT(*) AS count0_99, SUM(SalesOrderHeader.SubTotal) AS value0_99
FROM SalesOrderHeader
WHERE SalesOrderHeader.SubTotal BETWEEN 0 AND 99) AS a,
(SELECT COUNT(*) AS count100_999, SUM(SalesOrderHeader.SubTotal) AS value100_999
FROM SalesOrderHeader
WHERE SalesOrderHeader.SubTotal BETWEEN 100 AND 9999) AS b,
(SELECT COUNT(*) AS count1000_9999, SUM(SalesOrderHeader.SubTotal) AS value1000_9999
FROM SalesOrderHeader
WHERE SalesOrderHeader.SubTotal BETWEEN 1000 AND 9999) AS c,
(SELECT COUNT(*) AS count10000, SUM(SalesOrderHeader.SubTotal) AS value10000
FROM SalesOrderHeader
WHERE SalesOrderHeader.SubTotal > 10000) AS d
```

15\. Identify the three most important cities. Show the break down of top level product category against city.

Something like this for quantity of items sold to each city.

```sql
SELECT Address.City,
  SUM(SalesOrderDetail.OrderQty) AS total_sales_quantity
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
  JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
  JOIN Address ON SalesOrderHeader.ShipToAddressID = Address.AddressID
GROUP BY Address.City
ORDER BY total_sales DESC
```

Here it is by dollar amount sold (quantity * unit price).

```sql
SELECT Address.City,
  SUM(SalesOrderDetail.OrderQty * (SalesOrderDetail.UnitPrice - SalesOrderDetail.UnitPriceDiscount)) AS total_sales
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
  JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
  JOIN Address ON SalesOrderHeader.ShipToAddressID = Address.AddressID
GROUP BY Address.City
ORDER BY total_sales DESC
```

Here is a table containing percentage of quantity sold to each city represented as a percentage.

```sql
SELECT a.City, cat1.bike_percent, cat2.component_percent, cat3.clothing_percent, cat4.accessory_percent
FROM
(SELECT DISTINCT(City)
FROM Address) AS a
LEFT JOIN
(SELECT Address.City,
  (COALESCE(SUM(SalesOrderDetail.OrderQty), 0) / one.total_1) * 100 AS bike_percent
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
  JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
  JOIN Address ON SalesOrderHeader.ShipToAddressID = Address.AddressID,
(SELECT SUM(SalesOrderDetail.OrderQty) AS total_1
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
WHERE ProductCategory.ParentProductCategoryID = 1) AS one
WHERE ProductCategory.ParentProductCategoryID = 1
GROUP BY Address.City, one.total_1) AS cat1
ON a.City = cat1.City
LEFT JOIN
(SELECT Address.City,
  (COALESCE(SUM(SalesOrderDetail.OrderQty), 0) / two.total_2) * 100 AS component_percent
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
  JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
  JOIN Address ON SalesOrderHeader.ShipToAddressID = Address.AddressID,
(SELECT SUM(SalesOrderDetail.OrderQty) AS total_2
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
WHERE ProductCategory.ParentProductCategoryID = 2) AS two
WHERE ProductCategory.ParentProductCategoryID = 2
GROUP BY Address.City, two.total_2) AS cat2
ON a.City = cat2.City
LEFT JOIN
(SELECT Address.City,
  (COALESCE(SUM(SalesOrderDetail.OrderQty), 0) / three.total_3) * 100 AS clothing_percent
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
  JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
  JOIN Address ON SalesOrderHeader.ShipToAddressID = Address.AddressID,
(SELECT SUM(SalesOrderDetail.OrderQty) AS total_3
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
WHERE ProductCategory.ParentProductCategoryID = 2) AS three
WHERE ProductCategory.ParentProductCategoryID = 3
GROUP BY Address.City, three.total_3) AS cat3
ON a.City = cat3.City
LEFT JOIN
(SELECT Address.City,
  (COALESCE(SUM(SalesOrderDetail.OrderQty), 0) / four.total_4) * 100 AS accessory_percent
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
  JOIN SalesOrderHeader ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID
  JOIN Address ON SalesOrderHeader.ShipToAddressID = Address.AddressID,
(SELECT SUM(SalesOrderDetail.OrderQty) AS total_4
FROM ProductCategory JOIN Product ON ProductCategory.ProductCategoryID = Product.ProductCategoryID
  JOIN SalesOrderDetail ON Product.ProductID = SalesOrderDetail.ProductID
WHERE ProductCategory.ParentProductCategoryID = 4) AS four
WHERE ProductCategory.ParentProductCategoryID = 4
GROUP BY Address.City, four.total_4) AS cat4
ON a.City = cat4.City
ORDER BY a.City
```
