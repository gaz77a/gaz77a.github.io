---
layout: post
title: "🔒 Enforcing Tenant-Level Security with SQL Server's CREATE SECURITY POLICY 🔒"
image: /assets/images/tenant-level-security/title.png
date: 2025-06-26 20:46:33 +1000
categories: Tenant-Level Security
tags: SQL Server, Security, Multi-Tenant Applications
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
      src="/assets/images/tenant-level-security/title.png"
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

Tenant-level security is critical for multi-tenant applications where a single instance of the application serves multiple tenants. Ensuring that a tenant's data is isolated and inaccessible to other tenants is paramount. SQL Server's `Row-level Security` (RLS) feature; introduced in SQL 2016 and accessed via the `CREATE SECURITY POLICY` statement; provides a robust and efficient way to enforce such data isolation.
In this blog, I will walk through the process of implementing tenant-level security using the `CREATE SECURITY POLICY` feature in SQL Server, from setting up your database and defining security functions to observing the behavior in execution plans.

## 📑 Table of Contents

-   [Example of the Problem](#example-of-the-problem)
    -   [Initial Table Setup](#initial-table-setup)
    -   [Unrestricted Query Access](#unrestricted-query-access)
-   [Step-by-Step Guide to Enforcing Tenant-Level Security](#step-by-step-guide-to-enforcing-tenant-level-security)
    -   [1. Setting Session Context for Tenant Identification](#1-setting-session-context-for-tenant-identification)
    -   [2. Creating the Security Predicate Function](#2-creating-the-security-predicate-function)
    -   [3. Applying the Security Policy to Data Retrieval](#3-applying-the-security-policy-to-data-retrieval)
    -   [4. Testing the Implementation](#4-testing-the-implementation)
    -   [5. Observing the Impact in Execution Plans](#5-observing-the-impact-in-execution-plans)
    -   [6. Blocking Updates, Inserts, and Deletes](#6-blocking-updates-inserts-and-deletes)
    -   [7. Testing the Implementation](#7-testing-the-implementation)
-   [Final SQL Solution](#final-sql-solution)
-   [Conclusion](#conclusion)
-   [STILL TO COME:](#still-to-come)

## 🧩 Example of the Problem

Imagine we have a multi-tenant application that stores data for multiple tenants in a single table without row-level security. Tenants should only see their own data, but without proper security measures, they could potentially query and access data from other tenants.

### 🏗️ Initial Table Setup

Here’s a setup for a simple table without any security policies:

```sql
CREATE TABLE dbo.SomeMultiTenantedTable(
    Id INT PRIMARY KEY,
    TenantId INT,
    SomeData VARCHAR(100)
);

INSERT INTO dbo.SomeMultiTenantedTable(Id, TenantId, SomeData)
VALUES  (1, 1, 'Data for Tenant 1'),
        (2, 2, 'Data for Tenant 2'),
        (3, 1, 'More Data for Tenant 1'),
        (4, 2, 'More Data for Tenant 2');
```

### 🔓 Unrestricted Query Access

Without row-level security in place, any tenant can execute the following query and access all the data, regardless of tenant boundaries:

```sql
SELECT * FROM dbo.SomeMultiTenantedTable;
```

#### 📈 Result:

| Id  | TenantId | SomeData               |
| --- | -------- | ---------------------- |
| 1   | 1        | Data for Tenant 1      |
| 2   | 2        | Data for Tenant 2      |
| 3   | 1        | More Data for Tenant 1 |
| 4   | 2        | More Data for Tenant 2 |

This poses a serious security risk as Tenant 1 can see data belonging to Tenant 2 and vice versa.
This is also an issue if the application has a bug which excludes the Tenant from a CRUD query.

## 👣 Step-by-Step Guide to Enforcing Tenant-Level Security

### 1️⃣ Setting Session Context for Tenant Identification

To ensure each session identifies which tenant it belongs to, SQL Server's session context feature can be used. This allows you to set a case-sensitive key-value pair that will be used in the security predicate.

```sql
EXEC sp_set_session_context @key = N'TenantId', @value = '1';
```

### 2️⃣ Creating the Security Predicate Function

The next step is to create a security predicate function. This function will use the `SESSION_CONTEXT` to filter rows based on the tenant's ID.

```sql
CREATE FUNCTION dbo.fn_securityPredicate (@tenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS fn_securityPredicateResult
    WHERE @tenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);
```

This function checks if the `TenantId` of the row matches the `TenantId` set in the session context.

### 3️⃣ Applying the Security Policy to Data Retrieval

With the predicate function in place, you can now create a security policy that applies this function to `SomeMultiTenantedTable`.

```sql
CREATE SECURITY POLICY dbo.SecurityPolicy
ADD FILTER PREDICATE dbo.fn_securityPredicate(TenantId) ON dbo.SomeMultiTenantedTable
WITH (STATE = ON);
```

This policy will enforce row-level security to select queries based on the tenant ID stored in the session context.

### 4️⃣ Testing the Implementation

To test if tenant-level security is correctly enforced, set different `TenantId` values in the session context and observe the results:

#### 🎛️ Set Session Context to Tenant 1:

```sql
EXEC sp_set_session_context @key = N'TenantId', @value = '1';
SELECT * FROM dbo.SomeMultiTenantedTable;
```

#### 📈 Result:

| Id  | TenantId | SomeData               |
| --- | -------- | ---------------------- |
| 1   | 1        | Data for Tenant 1      |
| 3   | 1        | More Data for Tenant 1 |

#### 🎛️ Set Session Context to Tenant 2:

```sql
EXEC sp*set_session_context @key = N'TenantId', @value = '2';
SELECT * FROM dbo.SomeMultiTenantedTable;
```

#### 📈 Result:

| Id  | TenantId | SomeData               |
| --- | -------- | ---------------------- |
| 2   | 2        | Data for Tenant 2      |
| 4   | 2        | More Data for Tenant 2 |

### 5️⃣ Observing the Impact in Execution Plans

To see how the security policy affects query plans, enable the actual execution plan in SQL Server Management Studio (SSMS) and execute the query `SELECT \* FROM dbo.SomeMultiTenantedTable;`

![](/assets/images/tenant-level-security/image1.png)

You’ll notice additional operators related to the security predicates, reflecting the enforcement of tenant-level security. When compared to querying a table without row level security no noticeable differences where observed even when the number of rows processed was 1 million.

![](/assets/images/tenant-level-security/image2.png)

_10,000 rows queried_<br><br>

![](/assets/images/tenant-level-security/image3.png)

_1 million rows queried_<br><br>

![](/assets/images/tenant-level-security/image4.png)

_Predicate includes the RLS_<br><br>

### 6️⃣ Blocking Updates, Inserts, and Deletes

In addition to filtering query results, you may also want to prevent tenants from inserting, updating, or deleting data that does not belong to them. This can be achieved using block predicates in your security policy.

```sql
ALTER SECURITY POLICY dbo.SecurityPolicy
ADD BLOCK PREDICATE dbo.fn_securityPredicate(TenantId) ON dbo.SomeMultiTenantedTable;
```

This policy will enforce row-level security based on the tenant ID stored in the session context.

### 7️⃣ Testing the Implementation

To test if block predicate is correctly applied, attempt to perform insert, update, and delete operations with different `TenantId` values set in the session context:

#### 🎛️ Set Session Context to Tenant 1:

```sql
EXEC sp_set_session_context @key = N'TenantId', @value = '1';
```

#### 🧪 Testing Queries:

```sql
-- Successful attempt: Inserting a row for a matching TenantId
INSERT INTO dbo.SomeMultiTenantedTable(Id, TenantId, SomeData)
VALUES (5, 1, 'Valid Insert for Tenant 1');

-- Failed attempt: Inserting a row for a different TenantId
INSERT INTO dbo.SomeMultiTenantedTable(Id, TenantId, SomeData)
VALUES (6, 2, 'Invalid Insert for Tenant 1');

-- Failed attempt: Updating a row for a different TenantId
UPDATE dbo.SomeMultiTenantedTable
SET SomeData = 'Invalid Update for Tenant 1'
WHERE Id = 2;

-- Failed attempt: Deleting a row with a different TenantId
DELETE FROM dbo.SomeMultiTenantedTable
WHERE Id = 2;

-- Failed attempt: Changing a TenantId
UPDATE dbo.SomeMultiTenantedTable
SET TenantId = 2
WHERE Id = 1;

```

#### 📜 Listing Security Policies

In order to list the security policies in a data base run:

```sql
SELECT
    [name] AS SecurityPolicyName,
    object_id AS PolicyObjectId,
    principal_id AS PrincipalId,
    schema_id AS SchemaId,
    create_date AS CreateDate,
    modify_date AS ModifyDate
FROM sys.security_policies;
```

To list the security policies on each table run the query:

```sql
SELECT
    sp.name AS SecurityPolicyName,
    st.name AS TableName,
    schema_name(st.schema_id) AS SchemaName,
    spd.predicate_type_desc AS PredicateType
FROM
    sys.security_policies sp
    JOIN sys.security_predicates spd ON sp.object_id = spd.object_id
    JOIN sys.tables st ON spd.target_object_id = st.object_id;
```

## ⚗️Final SQL Solution

Here is the final SQL code required to add Row-level Security to a SQL database:

```sql
CREATE FUNCTION dbo.fn_securityPredicate (@tenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS fn_securitypredicate_result
    WHERE @tenantId = CAST(SESSION_CONTEXT(N'tenantId') AS INT);
GO

CREATE SECURITY POLICY dbo.SecurityPolicy
ADD FILTER PREDICATE dbo.fn_securityPredicate(TenantId) ON dbo.SomeMultiTenantedTable,
ADD BLOCK PREDICATE dbo.fn_securityPredicate(TenantId) ON dbo.SomeMultiTenantedTable
WITH (STATE = ON);
GO

EXEC sp_set_session_context @key = N'tenantId', @value = '1'; -- NOTE: The key is case sensitive
```

## 🎯 Conclusion

SQL Server's `CREATE SECURITY POLICY` feature offers a powerful way to enforce tenant-level security within your multi-tenant applications. By leveraging Row-level security, you can ensure that each tenant's data is isolated and secure, reducing the risk of unauthorized access and data breaches.
With the detailed steps provided in this blog, you can confidently apply tenant-level security in your database, ensuring that your multi-tenant application remains secure, efficient, and compliant with industry standards regardless of the way the data is accessed.

## 🔜 STILL TO COME:

Look out for an upcoming blog where I will detail how to set the session context from within C# and how connection pools are affected?
