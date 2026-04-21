# 📊 SQL Insights Report – W3Schools Dataset

This report presents a series of analytical explorations based on a sample relational database. The objective is to address a set of relevant data-driven questions that reflect common business analysis scenarios.

Each section focuses on a specific analytical question and demonstrates how SQL can be used to extract, transform, and interpret data to generate meaningful insights. The emphasis is placed not only on query construction, but also on the reasoning behind each approach and the interpretation of results.

This document is intended as a practical showcase of SQL skills applied to realistic data problems, highlighting the ability to derive actionable insights from structured data.

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

### 1. Which customers generate the highest total revenue?

#### 💻 SQL Query

```sql
WITH CustomerRevenue AS (
    SELECT 
        cust.CustomerID,
        cust.CustomerName,
        COALESCE(SUM(det.Quantity * prod.Price), 0) AS TotalRevenue
    FROM Customers AS cust
    LEFT JOIN Orders AS ord ON cust.CustomerID = ord.CustomerID
    LEFT JOIN OrderDetails AS det ON ord.OrderID = det.OrderID
    LEFT JOIN Products AS prod ON det.ProductID = prod.ProductID
    GROUP BY cust.CustomerID, cust.CustomerName
)

SELECT 
    CustomerName,
    TotalRevenue,
    RANK() OVER (ORDER BY TotalRevenue DESC) AS RevenueRank,
    CASE
        WHEN TotalRevenue = 0 THEN 'Potential Customer'
        WHEN TotalRevenue < 2000 THEN 'Occasional Customer'
        WHEN TotalRevenue BETWEEN 2000 AND 10000 THEN 'Regular Customer'
        ELSE 'High-value Customer'
    END AS Segment
FROM CustomerRevenue
ORDER BY TotalRevenue DESC;
```

#### 📊 Insight / Result

* Revenue is highly concentrated among a small group of customers classified as “High-value”
* A significant number of customers generate little to no revenue, indicating potential inactivity or churn
* Mid-tier customers (“Regular”) represent an opportunity for targeted growth strategies

#### 📈 Visualization (upcoming)

*

#### 📝 Remarks

* Segmentation thresholds are arbitrary and for demonstration purposes
    * In real scenarios, more realistic segmentation would be based on:
        * Percentiles (e.g., top 20%)
        * Business rules
        * Revenue distribution
* Data retrieval could be extended by:
    * Adding time filters (e.g.: last year only)
    * Tracking customer growth over time
    * Adding Customer-oriented filters (e.g.: Top N Customers; specific segment)
    * Calculating Customer (%) contribution over the Total Revenue

---

### 2. What are the top-performing products by revenue and quantity sold?

#### 💻 SQL Query

```sql
SELECT TOP 10
    prod.ProductID,
    prod.ProductName,
    RANK() OVER (ORDER BY SUM(det.Quantity * prod.Price) DESC) AS RevenueRank,
    COALESCE(SUM(det.Quantity * prod.Price), 0) AS Revenue,
    COALESCE(SUM(det.Quantity), 0) AS Quantity,
    RANK() OVER (ORDER BY SUM(det.Quantity) DESC) AS QuantityRank
FROM Products AS prod
JOIN OrderDetails AS det ON det.ProductID = prod.ProductID
GROUP BY prod.ProductID, prod.ProductName
ORDER BY Revenue DESC;
```

#### 📊 Insight / Result

* Some products rank highly in quantity but not in revenue, indicating lower-priced, high-volume items
* Other products generate high revenue despite lower sales volume, suggesting premium pricing
* This distinction can inform pricing, promotion, and inventory strategies
* Further steps: Conduct a root cause analysis on the top product's revenue outlier to isolate the driving factor — marketing, pricing, seasonality, or distribution — and develop a strategy to scale these results to other products

#### 📈 Visualization (upcoming)

*

#### 📝 Remarks

* Ranking is computed separately for quantity and revenue
* Top 10 is based on revenue
    * It can be changed depending on business focus (e.g.: quantity)
    * Or both could be obtained so that more details are returned and, possibly, matched (only 4 of the top-10 products by volume are shown)
* Could extend by:
    * Adding category breakdown
    * Calculating average price of product's category
    * Identifying “high-volume, low-revenue” vs “low-volume, high-revenue” products
    * Presenting product revenue as a percentage of total sales

---

### 3. How does monthly revenue evolve over time (trend analysis)?

#### 💻 SQL Query

```sql
SELECT 
    FORMAT(ord.OrderDate, 'yyyy-MM') AS YearMonth,
    SUM(det.Quantity * prod.Price) AS TotalRevenue,
    COUNT(DISTINCT ord.OrderID) AS TotalOrders,
    SUM(det.Quantity) AS TotalUnitsSold
FROM Orders AS ord
LEFT JOIN OrderDetails AS det ON ord.OrderID = det.OrderID
LEFT JOIN Products AS prod ON det.ProductID = prod.ProductID
GROUP BY FORMAT(ord.OrderDate, 'yyyy-MM')
ORDER BY YearMonth;
```

-->

#### 📊 Insight / Result

* Revenue on an upward trajectory, despite some fluctuations, indicating seasonality or demand variation
* Further data collection is required to deeply analyze why certain months experience significant growth or decline compared to previous periods
* Because monthly variations in TotalOrders and TotalUnitsSold fail to align with revenue changes, they are not effective key indicators

#### 📈 Visualization (upcoming)

*

#### 📝 Remarks

* Monthly aggregation enables trend analysis
* Could extend by:
    * Month-over-month comparison
    * Year-over-year comparison
    * Moving averages
    * Seasonal decomposition

---

### 4. Which countries contribute the most to total sales?

#### 💻 SQL Query

```sql
SELECT TOP 10
    COALESCE(cust.Country, 'Unknown') AS Country,
    SUM(det.Quantity * prod.Price) AS TotalRevenue,
    COUNT(DISTINCT ord.OrderID) AS TotalOrders,
    SUM(det.Quantity) AS TotalUnitsSold,
    RANK() OVER (ORDER BY SUM(det.Quantity * prod.Price) DESC) AS RevenueRank,
    ROUND(
        SUM(det.Quantity * prod.Price) * 100.0 
        / SUM(SUM(det.Quantity * prod.Price)) OVER (), 
        2
    ) AS RevenuePct
FROM Customers AS cust
LEFT JOIN Orders AS ord ON cust.CustomerID = ord.CustomerID
LEFT JOIN OrderDetails AS det ON ord.OrderID = det.OrderID
LEFT JOIN Products AS prod ON det.ProductID = prod.ProductID
GROUP BY COALESCE(cust.Country, 'Unknown')
ORDER BY TotalRevenue DESC;
```

#### 📊 Insight / Result

* Revenue is concentrated in a small number of countries
* Top-performing countries contribute a disproportionate share of total sales
* Lower-ranked countries show significantly smaller contributions, suggesting potential market expansion opportunities
* In some countries, premium pricing boosts revenue ranking, even when total order volume is low

#### 📈 Visualization (upcoming)

*

#### 📝 Remarks

* Could extend by:
    * Region grouping (e.g., Europe vs others)
    * Per-customer averages (per country)
    * Time-based country trends

---

### 5. Which employees (sales representatives) are associated with the highest revenue?

#### 💻 SQL Query

```sql
SELECT TOP 5
    RANK() OVER (ORDER BY SUM(det.Quantity * prod.Price) DESC) AS RevenueRank,
    emp.EmployeeID,
    emp.FirstName + ' ' + emp.LastName AS EmployeeName,
    SUM(det.Quantity * prod.Price) AS TotalRevenue,
    COUNT(DISTINCT ord.OrderID) AS TotalOrders,
    SUM(det.Quantity) AS TotalItemsSold,
    SUM(det.Quantity * prod.Price) * 1.0 
        / COUNT(DISTINCT ord.OrderID) AS AvgOrderValue
FROM Employees AS emp
LEFT JOIN Orders AS ord ON emp.EmployeeID = ord.EmployeeID
LEFT JOIN OrderDetails AS det ON ord.OrderID = det.OrderID
LEFT JOIN Products AS prod ON det.ProductID = prod.ProductID
GROUP BY emp.EmployeeID, emp.FirstName, emp.LastName
ORDER BY TotalRevenue DESC;
```

#### 📊 Insight / Result

* Some employees generate higher total revenue due to higher sales volume
* Others achieve higher average order values, suggesting stronger upselling or focus on higher-value products
* Differences between total revenue and average order value highlight varying sales strategies or customer portfolios
    * A deeper analysis of Margaret’s and Robert’s divergent strategies reveals critical opportunities for optimizing pricing, marketing, and market location to scale the business

#### 📈 Visualization (upcoming)

*

#### 📝 Remarks

* Lack of Seniority Adjustment: Older, more experienced employees may have more opportunities to disperse or balance out poor sales periods
* Unknown Business Context: The lack of industry details makes it difficult to assess if this requires a longer, relationship-driven sales cycle
* Could extend by:
    * Time-based performance (monthly/quarterly)
    * Customer portfolio per employee
    * Conversion or efficiency metrics

---

### 6. How dependent is each employee on a single shipper, and where are potential operational risks?

#### 💻 SQL Query

```sql
WITH ShipperRevenue AS (
    SELECT
        emp.EmployeeID,
        emp.FirstName + ' ' + emp.LastName AS EmployeeName,
        ship.ShipperID,
        ship.ShipperName,
        SUM(det.Quantity * prod.Price) AS ShipperTotal
    FROM Employees emp
    LEFT JOIN Orders ord ON emp.EmployeeID = ord.EmployeeID
    LEFT JOIN Shippers ship ON ord.ShipperID = ship.ShipperID
    LEFT JOIN OrderDetails det ON ord.OrderID = det.OrderID
    LEFT JOIN Products prod ON det.ProductID = prod.ProductID
    GROUP BY
        emp.EmployeeID,
        emp.FirstName,
        emp.LastName,
        ship.ShipperID,
        ship.ShipperName
),
Ranked AS (
    SELECT
        *,
        SUM(ShipperTotal) OVER (PARTITION BY EmployeeID) AS EmpTotal,
        ROW_NUMBER() OVER (
            PARTITION BY EmployeeID
            ORDER BY ShipperTotal DESC
        ) AS rnk
    FROM ShipperRevenue
)

SELECT
    EmployeeID,
    EmployeeName,
    ShipperName,
    ShipperTotal,
    EmpTotal,
    ShipperTotal * 100.0 / NULLIF(EmpTotal, 0) AS Pct,
    CASE 
        WHEN ShipperTotal * 100.0 / NULLIF(EmpTotal, 0) >= 80 THEN 'High Dependency'
        WHEN ShipperTotal * 100.0 / NULLIF(EmpTotal, 0) >= 50 THEN 'Moderate Dependency'
        WHEN ShipperTotal * 100.0 / NULLIF(EmpTotal, 0) IS NULL THEN 'N/A'
        ELSE 'Low Dependency'
    END AS DependencyLevel
FROM Ranked
WHERE rnk = 1
ORDER BY Pct DESC;
```

#### 📊 Insight / Result

* Several employees show moderate dependency on a single shipper, indicating potential operational risk
* A disruption in the primary shipping provider could significantly impact their sales performance
* Employees with lower dependency demonstrate more diversified logistics usage, suggesting greater resilience
* Current shipper allocation is skewed, with all employees operating above a balanced average distribution for the preferred partner

#### 📈 Visualization (upcoming)

*

#### 📝 Remarks

* Dependency thresholds are illustrative and could be adjusted
* Could extend by:
    * Tracking dependency over time
    * Including top 2–3 shippers per employee
    * Comparing performance vs dependency
* Separately evaluate sales volume distribution across shippers to identify reliance risks

---

### 7. Which customer has the highest total spend in each country?

#### 💻 SQL Query

```sql
WITH CustomerRevenue AS (
    SELECT 
        c.CustomerID,
        c.CustomerName,
        c.Country,
        SUM(od.Quantity * p.Price) AS TotalSpent
    FROM Customers c
    JOIN Orders o ON c.CustomerID = o.CustomerID
    JOIN OrderDetails od ON o.OrderID = od.OrderID
    JOIN Products p ON od.ProductID = p.ProductID
    GROUP BY 
        c.CustomerID,
        c.CustomerName,
        c.Country
),
Ranked AS (
    SELECT 
        *,
        RANK() OVER (
            PARTITION BY Country 
            ORDER BY TotalSpent DESC
        ) AS rnk
    FROM CustomerRevenue
)

SELECT 
    Country,
    CustomerID,
    CustomerName,
    TotalSpent,
    TotalSpent * 100.0 
        / SUM(TotalSpent) OVER (PARTITION BY Country) AS CountryPct
FROM Ranked
WHERE rnk = 1
ORDER BY TotalSpent DESC;
```

#### 📊 Insight / Result

* In most countries, a single customer represents the largest share of revenue
* The degree of dominance varies, indicating different levels of customer concentration across markets
* High concentration may signal dependency on key accounts within specific regions

#### 📈 Visualization (upcoming)

*

#### 📝 Remarks

* Could extend by:
    * Top 3 customers per country
    * Measuring concentration (e.g., % share)
    * Comparing across time periods

## ✅ Notes



<!--

### #. Template Question?

#### 💻 SQL Query

```sql
    ---- sql statement
```

#### 🧠 Approach / Logic

*

#### 📊 Insight / Result

*

#### 📈 Visualization (upcoming)

*

#### 📝 Remarks

*

-->