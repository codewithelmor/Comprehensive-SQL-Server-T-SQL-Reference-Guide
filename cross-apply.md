# SQL Server: CROSS APPLY

`CROSS APPLY` lets you join a table to a table-valued expression (often a function, subquery, or a computed set) that references columns from the left-hand table — something a regular `JOIN` can't do, since a `JOIN`'s condition can't reference the outer table inside the joined expression itself.

## How it works

- It's like an `INNER JOIN`: rows from the left table that produce **no** matching rows from the right-hand expression are dropped entirely.
- The right-hand side is typically a table-valued function, a derived table, or a subquery — and it *can* reference columns from the left table.

## Basic syntax

```sql
SELECT a.Id, a.Name, b.*
FROM TableA a
CROSS APPLY (
    SELECT TOP 3 *
    FROM TableB b
    WHERE b.AId = a.Id
    ORDER BY b.CreatedDate DESC
) b;
```

Here, for each row in `TableA`, you get the top 3 related rows from `TableB` — something you can't express with a plain `JOIN` because the `TOP 3 ... WHERE b.AId = a.Id` logic depends on the outer row.

## CROSS APPLY vs OUTER APPLY

- **CROSS APPLY** → acts like `INNER JOIN`: if the apply expression returns zero rows for a given outer row, that outer row disappears from the result.
- **OUTER APPLY** → acts like `LEFT JOIN`: outer row is kept even if the apply expression returns nothing (columns from the right side come back as `NULL`).

## Common use cases

### 1. Top-N per group

```sql
SELECT c.CustomerId, o.OrderId, o.OrderDate
FROM Customers c
CROSS APPLY (
    SELECT TOP 1 OrderId, OrderDate
    FROM Orders o
    WHERE o.CustomerId = c.CustomerId
    ORDER BY o.OrderDate DESC
) o;
```

Gets each customer's most recent order.

### 2. Calling a table-valued function per row

```sql
SELECT p.ProductId, s.SplitValue
FROM Products p
CROSS APPLY dbo.SplitString(p.TagList, ',') s;
```

### 3. Unpivoting/splitting JSON or CSV columns per row

```sql
SELECT o.OrderId, j.[key], j.[value]
FROM Orders o
CROSS APPLY OPENJSON(o.MetadataJson) j;
```

## Quick mental model

Think of `CROSS APPLY` as: *"for each row on the left, run this correlated subquery/function and join the results — dropping left rows that get nothing back."* `OUTER APPLY` is the same but keeps the left row anyway (like a `LEFT JOIN`).
