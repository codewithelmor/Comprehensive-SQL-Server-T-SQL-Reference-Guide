# Comprehensive SQL Server (T-SQL) Reference Guide
**Audience:** Intermediate–senior .NET developer working with Azure SQL
**Target version:** SQL Server 2022 / Azure SQL Database

Throughout this guide we use one consistent sample schema, a small **Sales** database:

```
Customers(CustomerID PK, FirstName, LastName, Email, CreatedAt)
Employees(EmployeeID PK, FirstName, LastName, ManagerID FK->Employees, HireDate)
Products(ProductID PK, ProductName, UnitPrice, CategoryName)
Orders(OrderID PK, CustomerID FK, EmployeeID FK, OrderDate, Status)
OrderItems(OrderItemID PK, OrderID FK, ProductID FK, Quantity, UnitPrice)
```

---

## 1. Database & Server Basics

- **Instance**: a running copy of the SQL Server engine (a Windows service / Linux process) that manages one or more databases. A single machine can host multiple named instances.
- **Database**: a container of schemas, tables, and other objects, backed by at least one data file (`.mdf`) and one log file (`.ldf`).
- **Schema**: a namespace inside a database (e.g. `dbo`, `sales`) used to group and secure objects; `Schema.Table` is the fully qualified object name.

### Creating a database

```sql
CREATE DATABASE Sales
ON PRIMARY (
    NAME = Sales_Data,
    FILENAME = '/var/opt/mssql/data/Sales.mdf',
    SIZE = 256MB,
    FILEGROWTH = 64MB
)
LOG ON (
    NAME = Sales_Log,
    FILENAME = '/var/opt/mssql/data/Sales_log.ldf',
    SIZE = 64MB,
    FILEGROWTH = 64MB
);
GO

USE Sales;
GO
CREATE SCHEMA sales AUTHORIZATION dbo;
```

> On Azure SQL Database, file placement (`ON PRIMARY`, `FILENAME`) is managed by the service — you only run `CREATE DATABASE Sales;` and choose a service tier.

### Inspecting objects with catalog views

```sql
SELECT name, database_id, create_date FROM sys.databases;

SELECT t.name AS table_name, s.name AS schema_name
FROM sys.tables t
JOIN sys.schemas s ON s.schema_id = t.schema_id;

SELECT c.name AS column_name, ty.name AS data_type, c.max_length, c.is_nullable
FROM sys.columns c
JOIN sys.types ty ON ty.user_type_id = c.user_type_id
WHERE c.object_id = OBJECT_ID('dbo.Customers');
```

**Summary:** An instance hosts databases; a database is organized into schemas; `sys.*` catalog views let you introspect structure without touching SSMS's GUI.

---

## 2. SQL Server Data Types

| Category | Type | Size / Range | Notes |
|---|---|---|---|
| Exact numeric | `bit` | 1 bit (stored packed) | Boolean-like: 0, 1, NULL |
| | `tinyint` | 0–255, 1 byte | Unsigned only |
| | `smallint` | ±32,767, 2 bytes | |
| | `int` | ±2.1 billion, 4 bytes | Default integer choice |
| | `bigint` | ±9.2×10¹⁸, 8 bytes | Use for IDs that may exceed int range |
| | `decimal(p,s)` / `numeric(p,s)` | up to 38 digits | Exact — use for money math |
| | `money` | ±922 trillion, 8 bytes | 4 decimal places; avoid, prefer `decimal` |
| | `smallmoney` | ±214,748, 4 bytes | Rarely used |
| Approximate numeric | `float(n)` | 4 or 8 bytes | Binary floating point, rounding error possible |
| | `real` | 4 bytes | `float(24)` alias |
| Date/time | `date` | 3 bytes | Date only |
| | `time` | 3–5 bytes | Time only, up to 100ns precision |
| | `datetime` | 8 bytes | Legacy; ~3.33ms precision, 1753–9999 |
| | `datetime2(n)` | 6–8 bytes | Preferred; wider range, higher precision |
| | `smalldatetime` | 4 bytes | Minute precision |
| | `datetimeoffset` | 8–10 bytes | `datetime2` + UTC offset — use for multi-timezone apps |
| Character | `char(n)` | fixed, n bytes | Padded with spaces |
| | `varchar(n)` | variable, up to n bytes | ASCII/extended |
| | `varchar(max)` | up to 2GB | Large text |
| | `text` | up to 2GB | **Deprecated** — use `varchar(max)` |
| Unicode | `nchar(n)` / `nvarchar(n)` | 2 bytes/char | Use for multi-language text |
| | `nvarchar(max)` | up to 2GB | Large Unicode text |
| | `ntext` | — | **Deprecated** — use `nvarchar(max)` |
| Binary | `binary(n)` / `varbinary(n)` | n bytes | Fixed/variable raw bytes |
| | `varbinary(max)` | up to 2GB | Blobs, files |
| | `image` | — | **Deprecated** — use `varbinary(max)` |
| Other | `uniqueidentifier` | 16 bytes | GUID; good for distributed key generation, poor for clustering-index locality |
| | `xml` | up to 2GB | Native XML type with XQuery support |
| | `json` (2022+) *(no native type; use `nvarchar` + `JSON_VALUE`/`ISJSON`)* | — | SQL Server stores JSON as text but offers native JSON functions |
| | `sql_variant` | up to 8016 bytes | Stores mixed types — use sparingly |
| | `table` | — | Table-valued variable / return type for TVFs |
| | `cursor` | — | Row-by-row iteration handle — avoid in set-based code |
| | `hierarchyid` | up to 892 bytes | Encodes tree position (org charts, categories) |
| | `geometry` / `geography` | variable | Planar / round-earth spatial data |
| | `rowversion` / `timestamp` | 8 bytes | Auto-incrementing binary counter for optimistic concurrency |

**Notes on deprecated types:** `text`, `ntext`, and `image` are deprecated and restricted in newer engine features (they can't be indexed with modern full-text options and have limited function support). Always use the `(max)` variants instead. In SQL Server 2022, `json` is not a first-class column type — you store it as `nvarchar(max)` and use `JSON_VALUE`, `JSON_QUERY`, `OPENJSON`, and `ISJSON` to work with it, with a `CHECK (ISJSON(col)=1)` constraint if you want validation.

**Summary:** Prefer `int`/`bigint` for numeric keys, `decimal` for money, `datetime2`/`datetimeoffset` for temporal data, and the `nvarchar`/`varbinary` `(max)` family over deprecated LOB types.

---

## 3. DDL — Data Definition Language

```sql
-- DATABASE / SCHEMA
CREATE DATABASE Sales;
ALTER DATABASE Sales SET RECOVERY SIMPLE;
DROP DATABASE Sales;

CREATE SCHEMA sales;
DROP SCHEMA sales;

-- TABLE
CREATE TABLE dbo.Customers (
    CustomerID  INT IDENTITY(1,1) PRIMARY KEY,
    FirstName   NVARCHAR(50)  NOT NULL,
    LastName    NVARCHAR(50)  NOT NULL,
    Email       NVARCHAR(256) NOT NULL UNIQUE,
    CreatedAt   DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME()
);

ALTER TABLE dbo.Customers ADD Phone VARCHAR(20) NULL;
ALTER TABLE dbo.Customers DROP COLUMN Phone;
DROP TABLE dbo.Customers;
TRUNCATE TABLE dbo.Customers; -- fast delete-all, resets IDENTITY, can't be filtered/rolled back mid-trigger

-- VIEW
CREATE VIEW sales.vw_CustomerOrders AS
SELECT c.CustomerID, c.FirstName, c.LastName, o.OrderID, o.OrderDate
FROM dbo.Customers c
JOIN dbo.Orders o ON o.CustomerID = c.CustomerID;
GO
ALTER VIEW sales.vw_CustomerOrders AS
SELECT c.CustomerID, c.FirstName, c.LastName, o.OrderID, o.OrderDate, o.Status
FROM dbo.Customers c
JOIN dbo.Orders o ON o.CustomerID = c.CustomerID;
GO
DROP VIEW sales.vw_CustomerOrders;

-- INDEX
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
    ON dbo.Orders (CustomerID) INCLUDE (OrderDate, Status);
ALTER INDEX IX_Orders_CustomerID ON dbo.Orders REBUILD;
DROP INDEX IX_Orders_CustomerID ON dbo.Orders;
```

### Constraints

```sql
CREATE TABLE dbo.Orders (
    OrderID     INT IDENTITY(1,1) NOT NULL,
    CustomerID  INT NOT NULL,
    EmployeeID  INT NOT NULL,
    OrderDate   DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME(),
    Status      VARCHAR(20)  NOT NULL DEFAULT 'Pending',
    CONSTRAINT PK_Orders PRIMARY KEY (OrderID),
    CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerID)
        REFERENCES dbo.Customers(CustomerID),
    CONSTRAINT FK_Orders_Employees FOREIGN KEY (EmployeeID)
        REFERENCES dbo.Employees(EmployeeID),
    CONSTRAINT CK_Orders_Status CHECK (Status IN ('Pending','Shipped','Cancelled'))
);
```

### Identity columns and sequences

```sql
-- Identity: simple auto-increment tied to one table/column
CREATE TABLE dbo.Log (LogID INT IDENTITY(1,1) PRIMARY KEY, Msg NVARCHAR(200));

-- Sequence: independent, reusable across tables, supports CYCLE/CACHE
CREATE SEQUENCE dbo.OrderNumberSeq
    START WITH 100000 INCREMENT BY 1 NO CYCLE CACHE 50;

INSERT INTO dbo.Orders (OrderID, CustomerID, EmployeeID)
VALUES (NEXT VALUE FOR dbo.OrderNumberSeq, 1, 1);
```

**Summary:** DDL statements define and evolve schema objects; constraints enforce data integrity at the engine level rather than in application code, and `SEQUENCE` gives you more control than `IDENTITY` when numbers must be shared across tables or generated before insert.

---

## 4. DML — Data Manipulation Language

### SELECT

```sql
-- WHERE / ORDER BY
SELECT CustomerID, FirstName, LastName
FROM dbo.Customers
WHERE LastName LIKE 'S%'
ORDER BY LastName, FirstName;

-- GROUP BY / HAVING
SELECT CustomerID, COUNT(*) AS OrderCount, SUM(oi.Quantity * oi.UnitPrice) AS TotalSpend
FROM dbo.Orders o
JOIN dbo.OrderItems oi ON oi.OrderID = o.OrderID
GROUP BY o.CustomerID
HAVING SUM(oi.Quantity * oi.UnitPrice) > 500;

-- INNER JOIN
SELECT o.OrderID, c.FirstName, c.LastName
FROM dbo.Orders o
INNER JOIN dbo.Customers c ON c.CustomerID = o.CustomerID;

-- LEFT JOIN (keep customers with zero orders)
SELECT c.CustomerID, o.OrderID
FROM dbo.Customers c
LEFT JOIN dbo.Orders o ON o.CustomerID = c.CustomerID;

-- RIGHT JOIN (equivalent to swapping table order in a LEFT JOIN)
SELECT o.OrderID, c.CustomerID
FROM dbo.Orders o
RIGHT JOIN dbo.Customers c ON c.CustomerID = o.CustomerID;

-- FULL JOIN
SELECT c.CustomerID, o.OrderID
FROM dbo.Customers c
FULL JOIN dbo.Orders o ON o.CustomerID = c.CustomerID;

-- CROSS JOIN (Cartesian product — every product x every category tag, e.g.)
SELECT p.ProductName, x.TagName
FROM dbo.Products p
CROSS JOIN (VALUES ('New'), ('Sale')) AS x(TagName);

-- SELF JOIN (employee -> manager)
SELECT e.FirstName AS Employee, m.FirstName AS Manager
FROM dbo.Employees e
LEFT JOIN dbo.Employees m ON m.EmployeeID = e.ManagerID;
```

### INSERT

```sql
-- Single row
INSERT INTO dbo.Customers (FirstName, LastName, Email)
VALUES ('Ana', 'Reyes', 'ana.reyes@example.com');

-- Multi-row
INSERT INTO dbo.Customers (FirstName, LastName, Email) VALUES
    ('Ben', 'Cruz', 'ben.cruz@example.com'),
    ('Cel', 'Santos', 'cel.santos@example.com');

-- INSERT ... SELECT
INSERT INTO dbo.OrderItems (OrderID, ProductID, Quantity, UnitPrice)
SELECT o.OrderID, p.ProductID, 1, p.UnitPrice
FROM dbo.Orders o
CROSS JOIN dbo.Products p
WHERE o.OrderID = 1001 AND p.ProductID = 5;
```

### UPDATE (with JOIN)

```sql
UPDATE oi
SET oi.UnitPrice = p.UnitPrice
FROM dbo.OrderItems oi
JOIN dbo.Products p ON p.ProductID = oi.ProductID
WHERE oi.UnitPrice <> p.UnitPrice; -- re-sync stale line-item prices
```

### DELETE vs TRUNCATE

```sql
DELETE FROM dbo.Orders WHERE Status = 'Cancelled' AND OrderDate < '2024-01-01';
-- Row-by-row, logged, fires triggers, can be filtered, rolls back individually.

TRUNCATE TABLE dbo.Log;
-- Deallocates pages, minimally logged, resets IDENTITY, no WHERE clause,
-- fails if the table is referenced by an FK with existing rows.
```

### MERGE (upsert)

```sql
MERGE dbo.Products AS target
USING (VALUES (5, 'Wireless Mouse', 19.99)) AS src (ProductID, ProductName, UnitPrice)
    ON target.ProductID = src.ProductID
WHEN MATCHED THEN
    UPDATE SET ProductName = src.ProductName, UnitPrice = src.UnitPrice
WHEN NOT MATCHED THEN
    INSERT (ProductID, ProductName, UnitPrice)
    VALUES (src.ProductID, src.ProductName, src.UnitPrice);
```

**Summary:** `SELECT` reads and shapes data; `INSERT`/`UPDATE`/`DELETE` mutate it; `MERGE` combines insert-or-update logic in one atomic statement — useful for sync jobs, but test its concurrency behavior carefully.

---

## 5. DCL — Data Control Language

Permissions apply at three scopes: **server** (logins, server roles, endpoints), **database** (database roles, schema-level grants), and **object** (table/view/column/procedure-level grants).

```sql
-- Object-level grants to a database role
GRANT SELECT, INSERT, UPDATE ON dbo.Orders TO app_role;
GRANT SELECT ON SCHEMA::sales TO reporting_role;

-- REVOKE removes a previously granted (or denied) permission — back to "not specified"
REVOKE INSERT ON dbo.Orders FROM app_role;

-- DENY explicitly blocks, and overrides any GRANT from another role membership
DENY DELETE ON dbo.Orders TO app_role;
```

**Summary:** `GRANT` allows, `REVOKE` clears a prior grant/deny back to unset, and `DENY` is an explicit block that wins over any conflicting `GRANT` — reserve `DENY` for cases where you must guarantee an action is blocked regardless of other role memberships.

---

## 6. TCL — Transaction Control Language

```sql
BEGIN TRANSACTION;
    UPDATE dbo.Accounts SET Balance = Balance - 100 WHERE AccountID = 1;
    UPDATE dbo.Accounts SET Balance = Balance + 100 WHERE AccountID = 2;
COMMIT TRANSACTION;

-- Partial rollback with a savepoint
BEGIN TRANSACTION;
    UPDATE dbo.Accounts SET Balance = Balance - 100 WHERE AccountID = 1;
    SAVE TRANSACTION BeforeCredit;
    UPDATE dbo.Accounts SET Balance = Balance + 100 WHERE AccountID = 2;
    IF @@ERROR <> 0
        ROLLBACK TRANSACTION BeforeCredit;
COMMIT TRANSACTION;
```

### Isolation levels

| Level | Dirty reads | Non-repeatable reads | Phantom reads |
|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED (default) | Prevented | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible |
| SERIALIZABLE | Prevented | Prevented | Prevented |
| SNAPSHOT | Prevented | Prevented | Prevented (via row versioning, not locks) |

- **Dirty read**: reading a row another transaction has changed but not committed.
- **Non-repeatable read**: re-reading the same row within a transaction returns different data because another transaction committed a change in between.
- **Phantom read**: re-running the same range query returns new/missing rows because another transaction inserted/deleted matching rows.

### TRY...CATCH with rollback

```sql
BEGIN TRY
    BEGIN TRANSACTION;

    UPDATE dbo.Accounts SET Balance = Balance - 100 WHERE AccountID = 1;
    UPDATE dbo.Accounts SET Balance = Balance + 100 WHERE AccountID = 2;

    IF (SELECT Balance FROM dbo.Accounts WHERE AccountID = 1) < 0
        THROW 50001, 'Insufficient funds.', 1;

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0
        ROLLBACK TRANSACTION;

    SELECT ERROR_NUMBER() AS ErrorNumber, ERROR_MESSAGE() AS ErrorMessage;
END CATCH;
```

**Summary:** Wrap multi-statement writes in explicit transactions, choose an isolation level deliberately (READ COMMITTED or SNAPSHOT for most OLTP work), and always pair `BEGIN TRY` with a `CATCH` block that checks `XACT_STATE()` before rolling back.

---

## 7. Creating a Login and Granting Database Permissions

Full sequence for a brand-new SQL login on a new database:

```sql
-- 1. CREATE LOGIN at the server level
-- SQL authentication:
CREATE LOGIN reporting_login WITH PASSWORD = 'StrongP@ssw0rd!', CHECK_POLICY = ON;
-- Windows authentication:
CREATE LOGIN [DOMAIN\svc_reporting] FROM WINDOWS;

-- 2. CREATE DATABASE (if not already created)
CREATE DATABASE Sales;
GO

-- 3. Switch context
USE Sales;
GO

-- 4. CREATE USER FOR LOGIN — maps the server login into this database
CREATE USER reporting_user FOR LOGIN reporting_login;

-- 5a. Add to a role...
CREATE ROLE reporting_role AUTHORIZATION dbo;
ALTER ROLE reporting_role ADD MEMBER reporting_user;

-- ...or 5b. grant object-level permissions directly
GRANT SELECT ON dbo.Customers TO reporting_user;
```

### Example: read-only "reporting" login via a custom role

```sql
CREATE LOGIN reporting_login WITH PASSWORD = 'StrongP@ssw0rd!';
USE Sales;
CREATE USER reporting_user FOR LOGIN reporting_login;
CREATE ROLE reporting_role;
GRANT SELECT ON SCHEMA::dbo TO reporting_role; -- SELECT on every dbo table/view
ALTER ROLE reporting_role ADD MEMBER reporting_user;
```

### Example: "app" login limited to specific tables and verbs

```sql
CREATE LOGIN app_login WITH PASSWORD = 'AnotherStr0ngP@ss!';
USE Sales;
CREATE USER app_user FOR LOGIN app_login;
GRANT SELECT, INSERT, UPDATE, DELETE ON dbo.Orders     TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON dbo.OrderItems TO app_user;
GRANT SELECT                          ON dbo.Customers TO app_user;
-- No GRANT on other tables => access denied by default
```

### Verifying effective permissions

```sql
-- What can the CURRENT session do?
SELECT * FROM sys.fn_my_permissions('dbo.Orders', 'OBJECT');

-- Impersonate another user to check *their* effective permissions
EXECUTE AS USER = 'app_user';
SELECT * FROM sys.fn_my_permissions('dbo.Customers', 'OBJECT');
REVERT;
```

**Summary:** A login authenticates to the server; a user maps that login into a specific database; permissions are then granted to the user directly or, preferably, to a role the user belongs to — `sys.fn_my_permissions` plus `EXECUTE AS` is the fastest way to verify the result without guessing.

---

## 8. Common Table Expressions (CTEs)

### Basic CTE

```sql
WITH HighValueOrders AS (
    SELECT o.OrderID, o.CustomerID, SUM(oi.Quantity * oi.UnitPrice) AS OrderTotal
    FROM dbo.Orders o
    JOIN dbo.OrderItems oi ON oi.OrderID = o.OrderID
    GROUP BY o.OrderID, o.CustomerID
)
SELECT c.LastName, h.OrderID, h.OrderTotal
FROM HighValueOrders h
JOIN dbo.Customers c ON c.CustomerID = h.CustomerID
WHERE h.OrderTotal > 1000;
```

### Recursive CTE — employee/manager hierarchy

```sql
WITH OrgChart AS (
    -- Anchor: top-level employees (no manager)
    SELECT EmployeeID, FirstName, ManagerID, 0 AS Level
    FROM dbo.Employees
    WHERE ManagerID IS NULL

    UNION ALL

    -- Recursive member: join back to OrgChart on the next level down
    SELECT e.EmployeeID, e.FirstName, e.ManagerID, o.Level + 1
    FROM dbo.Employees e
    JOIN OrgChart o ON e.ManagerID = o.EmployeeID
)
SELECT * FROM OrgChart
ORDER BY Level, FirstName
OPTION (MAXRECURSION 100); -- guard against runaway recursion, 0 = unlimited
```

### CTE vs subquery vs temp table vs view

| Approach | Scope | Materialized? | Reusable elsewhere? | Best for |
|---|---|---|---|---|
| CTE | Single statement | No — inlined by optimizer | No | Readability, recursive queries, breaking a query into named steps |
| Subquery | Single expression | No | No | Small, one-off filters/lookups |
| Temp table (`#t`) | Session/connection | Yes — real rows on disk/tempdb | Within the session | Large intermediate result sets, reused multiple times, need an index on it |
| View | Database object | No (unless indexed view) | Yes, by anyone with permission | Stable, reusable business definition of a query |

### Multiple chained CTEs

```sql
WITH OrderTotals AS (
    SELECT OrderID, SUM(Quantity * UnitPrice) AS Total
    FROM dbo.OrderItems
    GROUP BY OrderID
),
CustomerTotals AS (
    SELECT o.CustomerID, SUM(ot.Total) AS LifetimeSpend
    FROM dbo.Orders o
    JOIN OrderTotals ot ON ot.OrderID = o.OrderID
    GROUP BY o.CustomerID
)
SELECT c.LastName, ct.LifetimeSpend
FROM CustomerTotals ct
JOIN dbo.Customers c ON c.CustomerID = ct.CustomerID
ORDER BY ct.LifetimeSpend DESC;
```

**Summary:** CTEs improve readability and enable recursion but aren't materialized or indexable on their own — reach for a temp table when a large intermediate result needs to be reused or indexed, and a view when the query itself is a reusable business definition.

---

## 9. Built-in SQL Functions

### String functions

| Function | Example | Result |
|---|---|---|
| `LEN` | `LEN('Manila')` | `6` |
| `SUBSTRING` | `SUBSTRING('Manila',1,3)` | `'Man'` |
| `CONCAT` | `CONCAT(FirstName,' ',LastName)` | `'Ana Reyes'` |
| `TRIM` | `TRIM('  hi  ')` | `'hi'` |
| `LTRIM`/`RTRIM` | `LTRIM('  hi')` | `'hi'` |
| `REPLACE` | `REPLACE('a-b-c','-','_')` | `'a_b_c'` |
| `UPPER`/`LOWER` | `UPPER('sale')` | `'SALE'` |
| `CHARINDEX` | `CHARINDEX('@','ana@x.com')` | `4` |
| `STRING_AGG` | `STRING_AGG(ProductName, ', ')` | `'Mouse, Keyboard'` |
| `FORMAT` | `FORMAT(1234.5,'N2')` | `'1,234.50'` |

```sql
SELECT STRING_AGG(p.ProductName, ', ') WITHIN GROUP (ORDER BY p.ProductName) AS Products
FROM dbo.OrderItems oi
JOIN dbo.Products p ON p.ProductID = oi.ProductID
WHERE oi.OrderID = 1001;
```

### Date/time functions

| Function | Example |
|---|---|
| `GETDATE()` | current local `datetime` |
| `SYSDATETIME()` | current local `datetime2`, higher precision |
| `DATEADD` | `DATEADD(day, 7, OrderDate)` |
| `DATEDIFF` | `DATEDIFF(day, OrderDate, GETDATE())` |
| `DATEPART` | `DATEPART(month, OrderDate)` |
| `EOMONTH` | `EOMONTH(OrderDate)` — last day of that month |
| `FORMAT` | `FORMAT(OrderDate,'yyyy-MM-dd')` |

```sql
SELECT OrderID, OrderDate, DATEDIFF(day, OrderDate, SYSDATETIME()) AS AgeDays
FROM dbo.Orders
WHERE OrderDate >= DATEADD(month, -1, SYSDATETIME());
```

### Aggregate functions

```sql
SELECT COUNT(*) AS OrderCount, SUM(oi.Quantity*oi.UnitPrice) AS Revenue,
       AVG(oi.UnitPrice) AS AvgPrice, MIN(o.OrderDate) AS First, MAX(o.OrderDate) AS Last
FROM dbo.Orders o
JOIN dbo.OrderItems oi ON oi.OrderID = o.OrderID;

-- GROUPING SETS: multiple grouping levels in one result set
SELECT c.CustomerID, p.CategoryName, SUM(oi.Quantity*oi.UnitPrice) AS Revenue
FROM dbo.Orders o
JOIN dbo.OrderItems oi ON oi.OrderID = o.OrderID
JOIN dbo.Products p ON p.ProductID = oi.ProductID
JOIN dbo.Customers c ON c.CustomerID = o.CustomerID
GROUP BY GROUPING SETS ((c.CustomerID), (p.CategoryName), ());
```

### Window / analytic functions

```sql
SELECT
    o.OrderID, o.CustomerID, o.OrderDate,
    ROW_NUMBER() OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate) AS OrderSeq,
    RANK()       OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate) AS OrderRank,
    DENSE_RANK() OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate) AS OrderDenseRank,
    NTILE(4)     OVER (ORDER BY o.OrderDate) AS Quartile,
    LAG(o.OrderDate)  OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate) AS PrevOrderDate,
    LEAD(o.OrderDate) OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate) AS NextOrderDate,
    SUM(oi_total.Total) OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate
                               ROWS UNBOUNDED PRECEDING) AS RunningTotal
FROM dbo.Orders o
CROSS APPLY (
    SELECT SUM(Quantity*UnitPrice) AS Total FROM dbo.OrderItems WHERE OrderID = o.OrderID
) oi_total;
```

### Conversion functions

```sql
SELECT CAST('2026-09-26' AS date), CONVERT(varchar(10), GETDATE(), 120);
SELECT TRY_CAST('abc' AS int);      -- NULL, no error
SELECT TRY_CONVERT(decimal(10,2), '19.99');
SELECT PARSE('September 26, 2026' AS date USING 'en-US');
```

### Logical / conditional functions

```sql
SELECT
    CASE WHEN Status = 'Shipped' THEN 'Done' ELSE 'In progress' END AS StatusLabel,
    IIF(Status = 'Cancelled', 1, 0) AS IsCancelled,
    COALESCE(NULLIF(Status, ''), 'Unknown') AS SafeStatus,
    ISNULL(Status, 'Unknown') AS StatusOrDefault
FROM dbo.Orders;
```

### System functions

```sql
INSERT INTO dbo.Customers (FirstName, LastName, Email) VALUES ('Di', 'Torres', 'di@example.com');
SELECT SCOPE_IDENTITY() AS NewCustomerID; -- last identity value inserted by THIS session/scope
SELECT @@ROWCOUNT AS RowsAffected;        -- rows affected by the last statement

BEGIN TRY
    -- something that may fail
END TRY
BEGIN CATCH
    SELECT ERROR_MESSAGE() AS Msg, ERROR_NUMBER() AS Num;
END CATCH;
```

> Prefer `SCOPE_IDENTITY()` over `@@IDENTITY` — the latter also picks up IDENTITY values generated by triggers on other tables, which is rarely what you want.

**Summary:** Reach for `STRING_AGG`/`FORMAT` for presentation, `TRY_CAST`/`TRY_CONVERT` over `CAST`/`CONVERT` when input is untrusted, and window functions instead of correlated subqueries for running totals, rankings, and period-over-period comparisons.

---

## 10. Best Practices & Gotchas

**Indexing**
- Put a clustered index (usually the PK) on a narrow, ever-increasing key to avoid page splits — `IDENTITY int/bigint` is usually better than `uniqueidentifier` for this.
- Add nonclustered indexes on foreign key columns and frequent `WHERE`/`JOIN` predicates; use `INCLUDE` to make an index covering for a specific query.
- Rebuild/reorganize indexes periodically as fragmentation grows; check `sys.dm_db_index_physical_stats`.

**Common pitfalls**
- `NULL` comparisons: `WHERE Status = NULL` never matches — use `IS NULL` / `IS NOT NULL`.
- Implicit conversions (e.g. comparing an `nvarchar` column to a `varchar` literal) can silently disable index usage — match types explicitly.
- `SELECT *` couples callers to schema changes and defeats covering indexes — list columns.
- Cursor overuse: most row-by-row logic can be rewritten as a set-based `UPDATE ... FROM` or window function, which is dramatically faster.
- Missing transaction handling around multi-statement writes leaves data in an inconsistent state on partial failure.

**Security**
- Follow least privilege: grant only the verbs and objects a login actually needs (see Section 7), not `db_owner`.
- Never run application workloads as `sa` or `dbo`-equivalent accounts — create a dedicated login/user per application.
- Store connection secrets (passwords, connection strings) outside source control — e.g. Azure Key Vault for an Azure SQL-backed .NET app.

**Summary:** Most production SQL problems trace back to one of three things — a missing or wrong index, an implicit type mismatch, or a write that wasn't wrapped in a transaction — so make all three a deliberate part of code review, not an afterthought.
