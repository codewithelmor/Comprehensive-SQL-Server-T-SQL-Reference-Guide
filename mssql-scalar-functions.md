# Scalar Functions in Microsoft SQL Server (T-SQL)

## What is a Scalar Function?

A **user-defined scalar function** (UDF) takes zero or more input parameters and returns a **single value** (e.g. an `INT`, `VARCHAR`, `DECIMAL`). Unlike stored procedures, scalar functions can be used directly inside `SELECT` lists, `WHERE` clauses, computed columns, and `CHECK` constraints — anywhere a scalar expression is valid.

Syntax basics:
```sql
CREATE FUNCTION dbo.FunctionName (@Param1 INT, @Param2 VARCHAR(50))
RETURNS INT
AS
BEGIN
    DECLARE @Result INT;
    -- logic
    RETURN @Result;
END;
GO
```

---

## Example 1: Basic Scalar Function

Calculates a customer's age in years from their birth date.

```sql
CREATE FUNCTION dbo.CalculateAge (@BirthDate DATE)
RETURNS INT
AS
BEGIN
    DECLARE @Age INT;

    SET @Age = DATEDIFF(YEAR, @BirthDate, GETDATE())
        - CASE
            WHEN (MONTH(@BirthDate) > MONTH(GETDATE()))
              OR (MONTH(@BirthDate) = MONTH(GETDATE()) AND DAY(@BirthDate) > DAY(GETDATE()))
            THEN 1
            ELSE 0
          END;

    RETURN @Age;
END;
GO
```

**Use it:**
```sql
SELECT CustomerName, dbo.CalculateAge(BirthDate) AS Age
FROM dbo.Customers;

SELECT * FROM dbo.Customers
WHERE dbo.CalculateAge(BirthDate) >= 18;
```

**Explanation:**
- `RETURNS INT` declares the function's single output type; `RETURN @Age` sends that value back.
- Scalar functions must always be schema-qualified when called (`dbo.CalculateAge`, not just `CalculateAge`) — this is a distinctive T-SQL requirement.
- `DATEDIFF(YEAR, ...)` alone would just subtract calendar years, so the `CASE` expression adjusts for whether the birthday has occurred yet this year.
- **Performance caution:** calling a scalar UDF per-row in a `WHERE` clause (as in the second query) can prevent the optimizer from using indexes efficiently and force row-by-row execution — a well-known T-SQL performance pitfall, improved somewhat by "scalar UDF inlining" introduced in SQL Server 2019 for eligible functions.

---

## Example 2: Function with Conditional Logic and NULL Handling

Categorizes an order's size into a label based on its total amount, safely handling `NULL` input.

```sql
CREATE FUNCTION dbo.GetOrderSizeLabel (@TotalAmount DECIMAL(10,2))
RETURNS VARCHAR(20)
AS
BEGIN
    IF @TotalAmount IS NULL
        RETURN 'Unknown';

    RETURN CASE
        WHEN @TotalAmount < 50    THEN 'Small'
        WHEN @TotalAmount < 500   THEN 'Medium'
        WHEN @TotalAmount < 5000  THEN 'Large'
        ELSE 'Enterprise'
    END;
END;
GO
```

**Use it:**
```sql
SELECT
    OrderId,
    TotalAmount,
    dbo.GetOrderSizeLabel(TotalAmount) AS SizeLabel
FROM dbo.Orders;
```

**Explanation:**
- A function body can contain multiple `RETURN` statements as early exits (here, for the `NULL` case); execution stops at the first `RETURN` reached.
- `CASE` expressions inside a function are a clean, readable way to bucket values, and they themselves can be returned directly.
- Scalar functions in T-SQL are deterministic or non-deterministic based on the operations used; deterministic functions (same input always yields same output, e.g. no `GETDATE()`) can be indexed if the function is schema-bound and used in a persisted computed column.

---

## Example 3: Inline Scalar Expression Function Used in a Computed Column and Constraint

Calculates a discounted price and demonstrates reusing a scalar function in a table's computed column and a check constraint.

```sql
CREATE FUNCTION dbo.ApplyDiscount (@Price DECIMAL(10,2), @DiscountPercent DECIMAL(5,2))
RETURNS DECIMAL(10,2)
WITH SCHEMABINDING
AS
BEGIN
    IF @DiscountPercent < 0 OR @DiscountPercent > 100
        RETURN @Price; -- ignore invalid discount, return original price

    RETURN @Price - (@Price * @DiscountPercent / 100.0);
END;
GO

-- Used inside a computed column
ALTER TABLE dbo.Products
ADD DiscountedPrice AS dbo.ApplyDiscount(Price, 15.0) PERSISTED;
```

**Use it:**
```sql
SELECT Name, Price, DiscountedPrice
FROM dbo.Products;

-- Or call it directly, anywhere a scalar expression is valid
SELECT dbo.ApplyDiscount(199.99, 20) AS SalePrice;
```

**Explanation:**
- `WITH SCHEMABINDING` is required here because the function is referenced in a `PERSISTED` computed column — SQL Server needs to guarantee the underlying objects (none, in this simple case) won't change unexpectedly.
- `PERSISTED` tells SQL Server to physically store the computed column's value on disk (recalculated only when source columns change) rather than recomputing it on every read.
- Guarding against invalid input (`@DiscountPercent < 0 OR > 100`) inside the function centralizes validation logic in one reusable place instead of repeating it in every query.

---

## Key T-SQL Scalar Function Concepts Recap

| Concept | Purpose |
|---|---|
| `RETURNS <type>` | Declares the single scalar type returned |
| `RETURN` | Exits the function with a value; multiple `RETURN`s allowed as early exits |
| Schema-qualified calls (`dbo.Func(...)`) | Required syntax for calling scalar UDFs |
| `WITH SCHEMABINDING` | Locks the function to its referenced objects; needed for persisted computed columns |
| Performance caveat | Row-by-row UDF calls in `WHERE`/`JOIN` can hurt query plans; SQL Server 2019+ can inline simple ones |

## Managing Functions

```sql
ALTER FUNCTION dbo.CalculateAge (@BirthDate DATE) RETURNS INT AS BEGIN ... END; -- modify
DROP FUNCTION dbo.CalculateAge;                -- delete a function
sp_helptext 'dbo.CalculateAge';                 -- view definition
SELECT * FROM sys.objects WHERE type = 'FN';    -- list scalar functions
```
