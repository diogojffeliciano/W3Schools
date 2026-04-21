
# 📊 SQL Additional Queries – W3Schools Dataset
This section builds on the initial set of queries by exploring additional business-focused questions using the same dataset. The goal is to dive deeper into areas like customer behavior, product performance, and revenue patterns, while applying more advanced SQL techniques.

Each query is designed to answer a practical question and demonstrate a structured approach to analysis, from simple aggregations to more advanced concepts like ranking, segmentation, and distribution.

## 🗂️ Data Sources

Brief overview of the dataset and tables used.

* Tables
    * `Customers` – customer information
    * `Orders` – order transactions
    * `OrderDetails` – line-level order data
    * `Products` – product catalog
    * `Employees` – sales representatives
    * `Shippers` – shipper information
    * `Categories` – product categories
    * `Suppliers` – supplier information

* Relationships
    * OrderDetails → Orders → Customers
    * Orders → Employees
    * Orders → Shippers
    * OrderDetails → Products → Categories
    * Products → Suppliers

## 🔍 Analytical Questions
### 1. Which customers have placed orders in multiple distinct years?

#### Customer Retention

```sql
SELECT c.CustomerID,
    c.CustomerName,
    COUNT(DISTINCT YEAR(o.OrderDate)) AS NumYrs
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerID, c.CustomerName
Order By NumYrs DESC
;
```

#### Improvements
1. Retention Segment
1. First and Last years

### 2. Which countries have the highest average order value?

#### Average Order Value by Country

```sql
SELECT COALESCE(c.Country, 'Unknown') AS Country,
    SUM(od.Quantity * p.Price) TotalValue,
    COUNT(DISTINCT o.OrderID) TotalOrders,
    SUM(od.Quantity * p.Price) * 1.0 
        / NULLIF(COUNT(DISTINCT o.OrderID), 0) AS AvgOrderValue
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
LEFT JOIN OrderDetails od ON o.OrderID = od.OrderID
LEFT JOIN Products p ON od.ProductID = p.ProductID
GROUP BY COALESCE(c.Country, 'Unknown')
Order BY AvgOrderValue DESC
;
```

#### Improvements
1. Rank by Average
1. Revenue Percentage

### 3. Which product categories generate the most revenue and how do they compare in terms of quantity sold?

#### Product Category Performance

```sql
SELECT ca.CategoryID,
	ca.CategoryName,
	COALESCE(SUM(od.Quantity * p.Price), 0) AS Revenue,
	COALESCE(SUM(od.Quantity), 0) AS QttSold,
	RANK() OVER(ORDER BY COALESCE(SUM(od.Quantity * p.Price), 0) DESC) RevenueRnk,
	RANK() OVER(ORDER BY SUM(od.Quantity) DESC) QttSoldRnk
FROM Categories ca
	LEFT JOIN Products p ON ca.CategoryID = p.CategoryID
	LEFT JOIN OrderDetails od ON p.ProductID = od.ProductID
GROUP BY ca.CategoryID, ca.CategoryName
ORDER BY Revenue DESC
;
```

#### Improvements
1. Revenue Percentage


### 4. How are orders distributed by size (small, medium, large) based on total value?

#### Order Size Distribution

```sql
WITH OrderValues AS (
    SELECT 
        o.OrderID,
        COALESCE(SUM(od.Quantity * p.Price), 0) AS TotalValue
    FROM Orders o
    LEFT JOIN OrderDetails od 
        ON o.OrderID = od.OrderID
    LEFT JOIN Products p 
        ON od.ProductID = p.ProductID
    GROUP BY o.OrderID
),
Segmented AS (
    SELECT 
        OrderID,
        TotalValue,
        CASE
            WHEN TotalValue >= 10000 THEN 'large'
            WHEN TotalValue >= 3000 THEN 'medium'
            ELSE 'small'
        END AS OrderSize
    FROM OrderValues
)

SELECT 
    OrderSize,
    COUNT(*) AS NumOrders,
    ROUND(
        COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 
        2
    ) AS PctOrders,
    SUM(TotalValue) AS SegmentRevenue,
    SUM(TotalValue) * 100.0 / SUM(SUM(TotalValue)) OVER () RevenuePct
FROM Segmented
GROUP BY OrderSize
ORDER BY NumOrders DESC
;
```

### 5. Which customers place orders most frequently (highest number of orders)?

#### Customer Purchase Frequency

```sql
SELECT
    RANK() OVER (
        ORDER BY COUNT(DISTINCT o.OrderID) DESC
    ) AS FrequencyRank,
    c.CustomerID,
    c.CustomerName,
    COUNT(DISTINCT o.OrderID) AS TotalOrders,
    CASE 
        WHEN COUNT(DISTINCT o.OrderID) >= 20 THEN 'Very Frequent'
        WHEN COUNT(DISTINCT o.OrderID) >= 10 THEN 'Frequent'
        WHEN COUNT(DISTINCT o.OrderID) >= 3 THEN 'Occasional'
        WHEN COUNT(DISTINCT o.OrderID) >= 1 THEN 'Rare'
        ELSE 'Inactive'
    END AS FrequencySegment
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
GROUP BY 
    c.CustomerID, 
    c.CustomerName
ORDER BY TotalOrders DESC
;
```

### 6. How does each employee’s revenue contribution evolve over time?

#### Revenue per Employee Over Time

```sql
SELECT 
    e.EmployeeID,
    e.FirstName + ' ' + e.LastName AS EmployeeName,
    YEAR(o.OrderDate) AS Year,
    MONTH(o.OrderDate) AS Month,
    SUM(od.Quantity * p.Price) AS TotalRevenue,
    LAG(SUM(od.Quantity * p.Price)) OVER (
        PARTITION BY e.EmployeeID 
        ORDER BY YEAR(o.OrderDate), MONTH(o.OrderDate)
    ) AS PrevRevenue,
    SUM(od.Quantity * p.Price) 
        - LAG(SUM(od.Quantity * p.Price)) OVER (
            PARTITION BY e.EmployeeID 
            ORDER BY YEAR(o.OrderDate), MONTH(o.OrderDate)
        ) AS RevenueChange
FROM Employees e
LEFT JOIN Orders o ON e.EmployeeID = o.EmployeeID
LEFT JOIN OrderDetails od ON o.OrderID = od.OrderID
LEFT JOIN Products p ON od.ProductID = p.ProductID
WHERE o.OrderDate IS NOT NULL
GROUP BY 
    e.EmployeeID, 
    e.FirstName, 
    e.LastName,
    YEAR(o.OrderDate),
    MONTH(o.OrderDate)
ORDER BY 
    e.EmployeeID, 
    Year, 
    Month
;
```

### 7. What are the top 3 best-selling products within each category?

#### Top 3 Products per Category

```sql
WITH ProductSales AS (
    SELECT 
        p.ProductID,
        p.ProductName,
        p.CategoryID,
        COALESCE(SUM(od.Quantity), 0) AS TotalQtt
    FROM OrderDetails od
    JOIN Products p ON od.ProductID = p.ProductID
    GROUP BY 
        p.ProductID, p.ProductName, p.CategoryID
),
Ranked AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY CategoryID 
            ORDER BY TotalQtt DESC
        ) AS Rank,
        SUM(TotalQtt) OVER (PARTITION BY CategoryID) AS CategoryTotal
    FROM ProductSales
)

SELECT
    r.CategoryID,
    c.CategoryName,
    r.ProductID,
    r.ProductName,
    r.Rank,
    r.TotalQtt,
    r.TotalQtt * 100.0 / NULLIF(r.CategoryTotal, 0) AS Pct
FROM Ranked r
JOIN Categories c ON r.CategoryID = c.CategoryID
WHERE r.Rank <= 3
ORDER BY r.CategoryID, r.Rank
;
```

### 8. What percentage of each country’s revenue is generated by its top 3 customers?

#### Customer Concentration per Country

```sql
WITH CustomerRevenue AS (
    SELECT 
        COALESCE(c.Country, 'Unknown') AS Country,
        c.CustomerID,
        c.CustomerName,
        SUM(od.Quantity * p.Price) AS TotalRevenue
    FROM Customers c
    LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
    LEFT JOIN OrderDetails od ON o.OrderID = od.OrderID
    LEFT JOIN Products p ON od.ProductID = p.ProductID
    GROUP BY 
        c.CustomerID, c.CustomerName, COALESCE(c.Country, 'Unknown')
),
CountryTotals AS (
    SELECT 
        Country,
        SUM(TotalRevenue) AS CountryTotal
    FROM CustomerRevenue
    GROUP BY Country
),
Ranked AS (
    SELECT 
        cr.*,
        ROW_NUMBER() OVER (PARTITION BY cr.Country ORDER BY cr.TotalRevenue DESC) AS rank
    FROM CustomerRevenue cr
)

SELECT
    r.Country,
    SUM(r.TotalRevenue) AS Top3Revenue,
    ct.CountryTotal,
    SUM(r.TotalRevenue) * 100.0 / NULLIF(ct.CountryTotal, 0) AS pct
FROM Ranked r
JOIN CountryTotals ct ON r.Country = ct.Country
WHERE r.rank <= 3
GROUP BY r.Country, ct.CountryTotal
ORDER BY r.Country
;
```

### 9. Which shippers handle the highest number of orders and how does that compare to the revenue they carry?

#### Shipping Efficiency (Proxy)

```sql
WITH ShippersTotals AS (
    SELECT
        s.ShipperID,
        s.ShipperName,
        COUNT(DISTINCT o.OrderID) AS TotalOrders,
        COALESCE(SUM(od.Quantity * p.Price), 0) AS TotalRevenue
    FROM Shippers s
    LEFT JOIN Orders o ON s.ShipperID = o.ShipperID
    LEFT JOIN OrderDetails od ON o.OrderID = od.OrderID
    LEFT JOIN Products p ON od.ProductID = p.ProductID
    GROUP BY s.ShipperID, s.ShipperName
),
Ranked AS (
    SELECT 
        *,
        RANK() OVER (ORDER BY TotalOrders DESC) AS TotalOrderRank,
        RANK() OVER (ORDER BY TotalRevenue DESC) AS TotalRevRank
    FROM ShippersTotals
)

SELECT
    r.ShipperID,
    r.ShipperName,
    r.TotalOrders,
    r.TotalOrderRank,
    r.TotalRevenue,
    r.TotalRevRank,
    r.TotalOrderRank - r.TotalRevRank AS RankDiff
FROM Ranked r
ORDER BY r.TotalRevRank;
```

#### Improvements
1. Total revenue Percentage

### 10. Which customers have not placed any orders?

#### Inactive Customers

```sql
SELECT
	c.CustomerID,
    c.CustomerName,
    c.ContactName,
    c.Address,
    c.City,
    c.PostalCode,
    c.Country
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM Orders o
    WHERE o.CustomerID = c.CustomerID
    )
;
```