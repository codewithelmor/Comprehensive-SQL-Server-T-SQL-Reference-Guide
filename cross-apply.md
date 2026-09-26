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

## Why use CROSS APPLY instead of INNER JOIN?

`INNER JOIN` and `CROSS APPLY` overlap a lot, but there's one thing `INNER JOIN` fundamentally can't do: **reference a column from the left table inside the right-hand side's logic.**

A `JOIN`'s `ON` clause can only compare columns — it can't feed a value from the left table into a subquery, function call, or `TOP N` expression on the right table. `CROSS APPLY` can, because the right side is evaluated **once per row of the left table**, with that row's values already available.

### Where this actually matters

**1. Top-N per group (can't do this with INNER JOIN)**

```sql
-- Get each customer's most recent order
SELECT c.CustomerId, o.OrderId, o.OrderDate
FROM Customers c
CROSS APPLY (
    SELECT TOP 1 OrderId, OrderDate
    FROM Orders o
    WHERE o.CustomerId = c.CustomerId
    ORDER BY o.OrderDate DESC
) o;
```

With `INNER JOIN`, you'd get *all* matching orders — there's no way to say "only the top 1 per customer" inside the join itself. You'd need a window function (`ROW_NUMBER()`) with an extra filtering step instead.

**2. Calling a table-valued function with a per-row parameter**

```sql
SELECT p.ProductId, s.SplitValue
FROM Products p
CROSS APPLY dbo.SplitString(p.TagList, ',') s;
```

`INNER JOIN` can't call a function and pass in `p.TagList` — the function argument needs the outer row's value, which `JOIN` syntax has no mechanism for.

**3. Correlated subqueries that need row-specific logic**

Anything where the "right side" needs to *do something* with the left row's value (parse it, rank it, limit it, transform it) rather than just *match* it, is a job for `APPLY`, not `JOIN`.

### CROSS APPLY vs INNER JOIN — quick comparison

| | `INNER JOIN` | `CROSS APPLY` |
|---|---|---|
| Right side | Table or view | Table, function, subquery — can reference left row |
| Can use `TOP N` per row? | No | Yes |
| Can pass left column into a function? | No | Yes |
| Performance for simple equality joins | Same or better | Same (optimizer often treats them equivalently) |

**Rule of thumb:** if you can express the join purely as "these columns are equal," use `INNER JOIN` — it's more standard and often easier to read. Reach for `CROSS APPLY` only when the right side needs to *depend on* the left row's data beyond a simple equality match.

## Why use OUTER APPLY instead of LEFT JOIN?

Same core reason as above: **`LEFT JOIN` can't feed a left-table column into the logic of the right side** — it can only match on equality. `OUTER APPLY` can, and it keeps unmatched left rows (with NULLs) just like `LEFT JOIN` does.

`LEFT JOIN`'s `ON` clause is comparison-only. You can't ask it to run a `TOP N`, call a function, or execute a correlated subquery that needs the left row's value baked into its logic. `OUTER APPLY` evaluates its right side once per left row, using that row's actual values — and if it returns nothing, the left row survives anyway with NULLs, exactly like `LEFT JOIN`.

### Where this actually matters

**1. Top-N per group, but keep rows with no matches**

```sql
-- Every customer, plus their most recent order (or NULL if they have none)
SELECT c.CustomerId, o.OrderId, o.OrderDate
FROM Customers c
OUTER APPLY (
    SELECT TOP 1 OrderId, OrderDate
    FROM Orders o
    WHERE o.CustomerId = c.CustomerId
    ORDER BY o.OrderDate DESC
) o;
```

A `LEFT JOIN` version would return *every* matching order per customer (or you'd need `ROW_NUMBER()` + filtering). `OUTER APPLY` gives you exactly one row per customer, with NULLs for customers who have no orders at all.

**2. Calling a table-valued function, but keeping rows where it returns nothing**

```sql
SELECT p.ProductId, s.SplitValue
FROM Products p
OUTER APPLY dbo.SplitString(p.TagList, ',') s;
```

If `p.TagList` is NULL or empty and the function returns zero rows, the product still shows up (with `s.SplitValue = NULL`). A `LEFT JOIN` simply has no syntax for "call this function per row."

**3. Correlated subqueries with row-dependent logic, optional match**

Same idea as `CROSS APPLY` vs `INNER JOIN` — but here you want to preserve left rows even when the correlated logic finds nothing.

### OUTER APPLY vs LEFT JOIN — quick comparison

| | `LEFT JOIN` | `OUTER APPLY` |
|---|---|---|
| Right side | Table or view | Table, function, subquery — can reference left row |
| Keeps unmatched left rows? | Yes | Yes |
| Can use `TOP N` per row? | No | Yes |
| Can pass left column into a function? | No | Yes |

**Rule of thumb:** if the "match" is a straightforward equality and you want to keep unmatched left rows, use `LEFT JOIN`. Reach for `OUTER APPLY` when you need that same "keep everything" behavior, but the right side has to *compute* something per-row (top-N, a function call, a correlated expression) rather than just compare columns.
