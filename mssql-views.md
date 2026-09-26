# SQL Views in Microsoft SQL Server (T-SQL)

## What is a View?

A view is a virtual table defined by a stored `SELECT` query. It doesn't store data itself (except for indexed views) — it's a saved query that you can reference like a table. Views simplify complex queries, restrict column/row access for security, and provide a stable interface even if underlying tables change.

Syntax basics:
```sql
CREATE VIEW dbo.ViewName
AS
SELECT column1, column2
FROM dbo.SomeTable
WHERE condition;
GO
```

---

## Example 1: Basic View for Simplifying a Join

Combines orders with customer names so callers don't need to repeat the join every time.

```sql
CREATE VIEW dbo.vw_OrderDetails
AS
SELECT
    o.OrderId,
    o.OrderDate,
    c.CustomerName,
    o.TotalAmount
FROM dbo.Orders AS o
INNER JOIN dbo.Customers AS c
    ON o.CustomerId = c.CustomerId;
GO
```

**Query it:**
```sql
SELECT * FROM dbo.vw_OrderDetails
WHERE OrderDate >= '2026-01-01'
ORDER BY OrderDate DESC;
```

**Explanation:**
- A view is queried exactly like a table with `SELECT ... FROM ViewName`.
- SQL Server stores only the view's definition (the `SELECT` text), not its data; the query is re-executed (and can be further filtered/optimized) each time the view is referenced.
- `CREATE VIEW` must be the first statement in a batch, which is why `GO` typically precedes it if other statements come before.

---

## Example 2: View with Aggregation and `WITH SCHEMABINDING`

Produces a per-customer summary and locks the view's dependency on the underlying table schema, which is required if you later want to index the view.

```sql
CREATE VIEW dbo.vw_CustomerSalesSummary
WITH SCHEMABINDING
AS
SELECT
    o.CustomerId,
    COUNT_BIG(*)        AS OrderCount,
    SUM(o.TotalAmount)  AS TotalSpent
FROM dbo.Orders AS o
GROUP BY o.CustomerId;
GO

-- Optional: turn it into an indexed (materialized) view
CREATE UNIQUE CLUSTERED INDEX IX_vw_CustomerSalesSummary
ON dbo.vw_CustomerSalesSummary (CustomerId);
GO
```

**Query it:**
```sql
SELECT CustomerId, OrderCount, TotalSpent
FROM dbo.vw_CustomerSalesSummary
WHERE TotalSpent > 1000;
```

**Explanation:**
- `WITH SCHEMABINDING` prevents underlying tables from being altered/dropped in ways that would break the view, and is a prerequisite for creating an index on the view.
- `COUNT_BIG(*)` (rather than `COUNT(*)`) is required by SQL Server when the result will back an indexed view.
- Once a **unique clustered index** is created on the view, SQL Server physically materializes and maintains the aggregated data on disk — this is what makes it an "indexed view," Microsoft's version of a materialized view. Ordinary views have no physical storage; indexed views do.

---

## Example 3: Updatable View with `WITH CHECK OPTION`

Exposes only active customers in a specific region, and prevents updates through the view from creating rows that wouldn't show up in it.

```sql
CREATE VIEW dbo.vw_ActiveWestCustomers
AS
SELECT CustomerId, CustomerName, Region, IsActive
FROM dbo.Customers
WHERE Region = 'West' AND IsActive = 1
WITH CHECK OPTION;
GO
```

**Use it:**
```sql
-- Reads work like a normal table
SELECT * FROM dbo.vw_ActiveWestCustomers;

-- Updates are allowed because the view is based on a single table
UPDATE dbo.vw_ActiveWestCustomers
SET CustomerName = 'Acme West Corp'
WHERE CustomerId = 55;

-- This INSERT will FAIL because Region = 'East' violates the view's WHERE clause
INSERT INTO dbo.vw_ActiveWestCustomers (CustomerId, CustomerName, Region, IsActive)
VALUES (999, 'Test Co', 'East', 1);
```

**Explanation:**
- A view built on a **single base table** without aggregates, `DISTINCT`, or `GROUP BY` is generally **updatable** — `INSERT`/`UPDATE`/`DELETE` against the view pass through to the base table.
- `WITH CHECK OPTION` enforces that any row inserted or updated through the view must still satisfy the view's `WHERE` clause; otherwise SQL Server rejects it. Without this option, you could silently insert/update rows that then "disappear" from the view.
- Views spanning multiple tables (joins) are generally **not** updatable directly unless you add an `INSTEAD OF` trigger.

---

## Key T-SQL View Concepts Recap

| Concept | Purpose |
|---|---|
| `CREATE VIEW ... AS SELECT ...` | Defines a virtual table from a query |
| `WITH SCHEMABINDING` | Locks the view to its underlying schema; required for indexed views |
| Indexed (materialized) view | A view with a unique clustered index — physically stores data |
| `WITH CHECK OPTION` | Rejects inserts/updates that would violate the view's `WHERE` clause |
| `INSTEAD OF` trigger | Enables updates on views that aren't natively updatable (e.g. joins) |

## Managing Views

```sql
ALTER VIEW dbo.vw_OrderDetails AS SELECT ...;  -- modify a view's definition
DROP VIEW dbo.vw_OrderDetails;                  -- delete a view
sp_helptext 'dbo.vw_OrderDetails';              -- view the definition text
SELECT * FROM sys.views WHERE name = 'vw_OrderDetails'; -- metadata lookup
```
