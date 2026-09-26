# Stored Procedures in Microsoft SQL Server (T-SQL)

## What is a Stored Procedure?

A stored procedure is a precompiled collection of one or more SQL statements stored under a name in the database. It can accept input parameters, return output parameters, and return result sets. Stored procedures improve performance (execution plan caching), enforce security (users can execute a procedure without direct table access), and centralize business logic.

Syntax basics:
```sql
CREATE PROCEDURE dbo.ProcedureName
    @Param1 INT,
    @Param2 NVARCHAR(50) = NULL   -- default value
AS
BEGIN
    SET NOCOUNT ON;
    -- procedure body
END;
```

---

## Example 1: Basic Procedure with Input Parameters

Retrieves all orders for a given customer within a date range.

```sql
CREATE PROCEDURE dbo.GetCustomerOrders
    @CustomerId INT,
    @StartDate  DATE,
    @EndDate    DATE
AS
BEGIN
    SET NOCOUNT ON;

    SELECT OrderId, OrderDate, TotalAmount
    FROM dbo.Orders
    WHERE CustomerId = @CustomerId
      AND OrderDate BETWEEN @StartDate AND @EndDate
    ORDER BY OrderDate DESC;
END;
GO
```

**Execute it:**
```sql
EXEC dbo.GetCustomerOrders @CustomerId = 101, @StartDate = '2026-01-01', @EndDate = '2026-06-30';
```

**Explanation:**
- `SET NOCOUNT ON` suppresses the "rows affected" message, reducing network traffic — a standard best practice in T-SQL procedures.
- Parameters are strongly typed and passed by name (`@Param = value`) or by position.
- The procedure simply wraps a parameterized `SELECT`, making it reusable and reducing ad-hoc query duplication.

---

## Example 2: Output Parameters and Error Handling

Inserts a new product and returns the generated identity value, with `TRY...CATCH` error handling and a transaction.

```sql
CREATE PROCEDURE dbo.AddProduct
    @Name       NVARCHAR(100),
    @Price      DECIMAL(10,2),
    @CategoryId INT,
    @NewProductId INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        INSERT INTO dbo.Products (Name, Price, CategoryId)
        VALUES (@Name, @Price, @CategoryId);

        SET @NewProductId = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;

        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE();
        THROW 50000, @ErrMsg, 1;
    END CATCH
END;
GO
```

**Execute it:**
```sql
DECLARE @NewId INT;
EXEC dbo.AddProduct
    @Name = 'Wireless Mouse',
    @Price = 19.99,
    @CategoryId = 3,
    @NewProductId = @NewId OUTPUT;

SELECT @NewId AS NewProductId;
```

**Explanation:**
- `OUTPUT` parameters let a procedure return scalar values back to the caller in addition to result sets.
- `SCOPE_IDENTITY()` safely retrieves the last identity value inserted in the current scope (safer than `@@IDENTITY`, which can be affected by triggers).
- `TRY...CATCH` combined with `BEGIN TRANSACTION` / `COMMIT` / `ROLLBACK` gives robust, atomic error handling. `XACT_STATE()` checks whether the transaction is still committable before rolling back.
- `THROW` re-raises a custom error, preserving the ability for calling applications to catch it.

---

## Example 3: Conditional Logic, Temp Tables, and Dynamic Result Sets

Generates a sales summary report, optionally filtered by region, using a temp table and control-of-flow statements.

```sql
CREATE PROCEDURE dbo.GetSalesSummary
    @Region NVARCHAR(50) = NULL,
    @Year   INT
AS
BEGIN
    SET NOCOUNT ON;

    CREATE TABLE #SalesTemp
    (
        Region      NVARCHAR(50),
        TotalSales  DECIMAL(18,2),
        OrderCount  INT
    );

    INSERT INTO #SalesTemp (Region, TotalSales, OrderCount)
    SELECT
        s.Region,
        SUM(s.Amount),
        COUNT(*)
    FROM dbo.Sales AS s
    WHERE YEAR(s.SaleDate) = @Year
      AND (@Region IS NULL OR s.Region = @Region)
    GROUP BY s.Region;

    IF EXISTS (SELECT 1 FROM #SalesTemp)
    BEGIN
        SELECT Region, TotalSales, OrderCount
        FROM #SalesTemp
        ORDER BY TotalSales DESC;
    END
    ELSE
    BEGIN
        PRINT 'No sales data found for the given criteria.';
    END

    DROP TABLE #SalesTemp;
END;
GO
```

**Execute it:**
```sql
EXEC dbo.GetSalesSummary @Region = NULL, @Year = 2026;   -- all regions
EXEC dbo.GetSalesSummary @Region = 'West', @Year = 2026; -- one region
```

**Explanation:**
- `#SalesTemp` is a **local temporary table**, session-scoped and automatically dropped when the session/procedure ends (explicit `DROP TABLE` here for clarity).
- `@Region IS NULL OR s.Region = @Region` is a common pattern for optional filters in a single procedure.
- `IF EXISTS (...) ... ELSE ...` demonstrates T-SQL's control-of-flow branching.
- Procedures can return **zero, one, or multiple result sets** depending on logic — useful for reporting procedures.

---

## Key T-SQL Concepts Recap

| Concept | Purpose |
|---|---|
| `@Variable` | Parameters and local variables, always prefixed with `@` |
| `OUTPUT` | Returns a value to the caller besides result sets |
| `SET NOCOUNT ON` | Performance best practice, suppresses row-count messages |
| `TRY...CATCH` | Structured error handling |
| `#TempTable` | Session-scoped temporary storage |
| `SCOPE_IDENTITY()` | Safely retrieve last inserted identity value |
| `EXEC` / `EXECUTE` | Invokes a stored procedure |

## Managing Procedures

```sql
ALTER PROCEDURE dbo.GetCustomerOrders ...   -- modify existing procedure
DROP PROCEDURE dbo.GetCustomerOrders;        -- delete a procedure
EXEC sp_helptext 'dbo.GetCustomerOrders';    -- view procedure definition
```
