---
layout: post
title: '⏱️⚡ How Much Faster? Real-World Timings When Raising SQL Server Database Compatibility Levels ⚡⏱️'
date: 2025-07-03 21:52:30 +1000
categories: SQL Compatibility
tags: SQL Server, Performance, Compatibility Level
---

<div style="
  max-width: 1100px;
  width: 100%;
  margin: 42px auto 44px auto;
  border-radius: 28px;
  box-shadow:
    0 20px 80px -4px rgba(44,60,180,0.72),             /* bold shadow */
    0 28px 128px 0 rgba(44,50,100,0.19);               /* halo */
  overflow: visible;        /* shadow spills out, not clipped */
  background: none;
  position: relative;
  display: block;
">
  <div style="
    border-radius: 28px;
    overflow: hidden;       /* ensures image corners are rounded */
    width: 100%;
    height: 136px;          /* Flat banner */
  ">
    <img
      src="/assets/images/sql-compatibility/title.png"
      alt="Title image"
      style="
        display: block;
        width: 100%;
        height: 100%;
        object-fit: cover;
        background: #dde8fc;
        border: 0;
      "
    />
  </div>
</div>

> By Gary Butler | 5 min read

<br><br>

## 🚩 Introduction

Upgrading your SQL Server’s database compatibility level is one of the easiest ways to get the latest performance benefits — **without changing a single line of application code**. But what actual improvements can you expect when raising a database’s compatibility level from SQL Server 2014 (level 120) up through 2016, 2017, 2019, and 2022? Are there measurable performance wins, and do you need to be careful? Let’s dive in with worked examples and lessons learned.

<br><br>

## 📈 Why Does Compatibility Level Matter?

The SQL Server `compatibility level` setting lets a database behave like it’s running on an older version of SQL Server by maintaining legacy query processing, optimization, and T-SQL language rules. When you increase the compatibility level, your queries can take advantage of newer performance enhancements, smarter query processing, and additional security features provided in recent SQL Server versions (stay tuned for a blog on this topic).

When you upgrade SQL Server, the compatibility level for existing databases does not change automatically — it remains set to its previous value unless you update it manually.

<br><br>

## ⏱️ Worked Example: Query Timings Across Compatibility Levels

Let’s set the stage:

-   **Hardware:** Processor 12th Gen Intel(R) Core(TM) i7-12800H, 2400 Mhz, 32GB RAM, NVMe SSD
-   **Database:** [Wide World Importers sample database](https://github.com/microsoft/sql-server-samples/releases/tag/wide-world-importers-v1.0)
-   **SQL Server Version:** 2022
-   **Test Query:** Aggregation of sales by product and date.
    Here’s our base query:

```sql
SET STATISTICS TIME ON;

SELECT
    Sales.Customers.CustomerID,
    Sales.Customers.CustomerName,
    Sales.InvoiceLines.InvoiceID,
    Warehouse.StockItems.StockItemID,
    Warehouse.StockItems.StockItemName,
    Application.Cities.CityName AS CustomerCity,
    Sales.InvoiceLines.Description,
    Sales.InvoiceLines.Quantity,
    Sales.InvoiceLines.ExtendedPrice,
    Application.People.FullName AS SalespersonName,
    Sales.Invoices.InvoiceDate,
    AVG(Sales.InvoiceLines.ExtendedPrice) OVER (PARTITION BY Sales.Customers.CustomerID) AS AvgInvoiceAmountForCustomer
FROM Sales.InvoiceLines
INNER JOIN Sales.Invoices ON Sales.InvoiceLines.InvoiceID = Sales.Invoices.InvoiceID
INNER JOIN Sales.Customers ON Sales.Invoices.CustomerID = Sales.Customers.CustomerID
INNER JOIN Warehouse.StockItems ON Sales.InvoiceLines.StockItemID = Warehouse.StockItems.StockItemID
INNER JOIN Application.Cities ON Sales.Customers.DeliveryCityID = Application.Cities.CityID
INNER JOIN Application.People ON Sales.Invoices.ContactPersonID = Application.People.PersonID
WHERE Sales.Invoices.InvoiceDate >= '2015-01-01'
ORDER BY
	Sales.Customers.CustomerID,
	Sales.Invoices.InvoiceDate DESC,
	Warehouse.StockItems.StockItemName
OPTION (RECOMPILE);
```

<br><br>

### 📋 Query Summary

For each customer, the query calculates the average invoice line’s extended price (`AvgInvoiceAmountForCustomer`). This is done using the window function `AVG(Sales.InvoiceLines.ExtendedPrice) OVER (PARTITION BY Sales.Customers.CustomerID).`

In particular, for every row, the value shown is the average of all invoice line extended prices for that specific customer, regardless of the invoice or item.

<br><br>

### 🛠️ Methodology

-   Time measured in milliseconds (ms) using `SET STATISTICS TIME ON`.
-   Run each test 10 times and calculate the median and standard deviation.
-   Compatibility levels: 120 (2014), 130 (2016), 140 (2017), 150 (2019), 160 (2022).
-   Recompile option used to ensure consistency between runs

<br><br>

### 📊 Results Table

| Compatibility Level | SQL Version Emulated | Median Elapsed Time (ms) | % Improvement over 2014 | Standard Deviation | Notes                                                            |
| ------------------- | -------------------- | ------------------------ | ----------------------- | ------------------ | ---------------------------------------------------------------- |
| 120                 | 2014                 | 2423                     | N/A                     | 234                | Legacy cardinality estimator                                     |
| 130                 | 2016                 | 1629                     | 32.8%                   | 73                 | New cardinality estimator, Enhanced parallelism                  |
| 140                 | 2017                 | 1320                     | 45.5%                   | 98                 | Adaptive query processing                                        |
| 150                 | 2019                 | 1302                     | 46.3%                   | 95                 | IQP enhancements; parameter sensitive plan, adaptive parallelism |

<br><br>

## 🗝️ What Features Unlock These Improvements?

-   **Batch Mode Execution** (compatibility 130+): Improved analytic query speed.
-   **Adaptive Query Processing** (140+): Plans can fix themselves on the fly.
-   **Intelligent Query Processing** (150+): Table variable deferred compilation, batch mode on rowstore, scalar UDF inlining—often double-digit percent gains in real workloads.
-   **Parameter Sensitive Plan optimization** (160): Reduces “parameter sniffing” performance issues.

<br><br>

## ⚠️ Caution: Improvements May Vary

A counterexample: One customer’s complex reporting query actually slowed down under compatibility level 150 — because the new optimizer chose what it thought was a more efficient plan, which wasn’t true for that specific data distribution.

**Tip:**
Always test your actual workload. Use SQL Server Query Store (enabled by default since SQL Server 2016) to compare baseline and post-change performance.

<br><br>

## 🪜 Step-by-Step: Measuring in Your Own Environment

1. Set up Query Store for before/after comparison:

    ```sql
    ALTER DATABASE MyDb SET QUERY_STORE = ON;
    ```

1. Run Baseline Tests at current compatibility level:

    ```sql
    ALTER DATABASE MyDb SET COMPATIBILITY_LEVEL = 120;
    -- Run and record timings
    ```

1. Change Compatibility and Rerun:

    ```sql
    ALTER DATABASE MyDb SET COMPATIBILITY_LEVEL = 150;
    -- Run and record timings
    ```

1. Check Plan Regression with Query Store
   Query Store’s “regressed queries” report highlights any slowdowns.

<br><br>

## 🤔 Final Thoughts — Unlocking Performance, Responsibly

**In the right workload, raising database compatibility level can yield query time reductions from 10% to 60%** — often simply by enabling smarter plan choice without any code changes. However, some queries may regress or behave differently, especially if they relied on quirks of older optimizers or cardinality estimators.

### 🥡 Key Takeaways:

-   **Most workloads see a performance boost**, especially for analytical, reporting, and ad hoc queries as you move from 2014 to 2019 and 2022 compatibility.
-   **The magnitude of improvement varies**. My example query showed a reduction from 2,423 ms to 1,261 ms — a 48% improve ment — but YOUR mileage may differ. Expect anywhere from modest gains to dramatic wins, particularly if your queries were previously bottlenecked by older query optimizations.
-   **Some regression risk exists**. Always test and validate against your own business-critical queries. Use Query Store to monitor for regressions, and be prepared to tune problematic queries or revisit compatibility if needed.
-   **New features become available** as you raise the level: native string splitting, temporal tables, adaptive joins, approximate count distinct, scalar UDF inlining, and more.

<br><br>

## 🧩 Additional Worked Example: Scalar UDF Inlining

Suppose you have a table-valued function in use within a frequently run query:

```sql
-- Scalar function to apply a discount
CREATE OR ALTER FUNCTION dbo.ApplyDiscount(@value DECIMAL(18,2))
RETURNS DECIMAL(18,2) AS
BEGIN
    RETURN @value * 0.97 -- apply small discount
END;
```

And you frequently call this:

```sql
SELECT
    StockItemID,
    SUM(dbo.ApplyDiscount(ExtendedPrice)) AS DiscountedRevenue
FROM Sales.InvoiceLines
GROUP BY StockItemID;
```

**In Compatibility Level 140 and below:**
This function is evaluated row-by-row (RBAR = Row-By-Agonizing-Row), which is slow for large tables.

**In Compatibility Level 150 (SQL 2019) and above:**
The function can be "inlined" directly into the query by the optimizer, transforming it into set-based logic and often resulting in orders-of-magnitude performance improvement.

**Example Timings (228,265 rows):**

| Compatibility Level | SQL Version Emulated | Median Duration (ms) | % Improvement |
| ------------------- | -------------------- | -------------------- | ------------- |
| 140                 | 2017                 | 2,704                | N/A           |
| 150                 | 2019                 | 77                   | 97%           |

<br><br>

## 🛡️ How To Safely Upgrade

1. **Benchmark Before and After:**
   Capture baseline timings and execution plans using SET STATISTICS TIME ON, Query Store or third-party monitoring tools.
1. **Step Up Gradually:**
   Consider moving up one compatibility level at a time, especially if you’re several versions behind. Test at each step.
1. **Leverage Query Store Hints:**
   If a specific query regresses, you can pin its plan or force the use of the previous compatibility behavior for just that query.
1. **Update Statistics:**
   Always ensure statistics are up to date before and after upgrading. Newer compatibility levels can rely even more heavily on accurate statistics.

<br><br>

## 🎯 Conclusion

**Raising the SQL Server compatibility level is a high-impact, low-effort way to unlock substantial performance improvements and new features**, as long as you test thoroughly and monitor for regressions. Time investments spent benchmarking and validating will pay off in smoother upgrades, happier users, and the ability to take advantage of SQL Server's continuous advancements.

<br><br>

```
💬 What’s your experience been with upgrading database compatibility levels?
💬 Did you see similar speedups, or hit any bumps in the road?
```
