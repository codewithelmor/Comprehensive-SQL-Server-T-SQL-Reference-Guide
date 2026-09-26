# SQL Server JSON Column — Comprehensive Guide

## 1. Overview

SQL Server does **not** have a native `JSON` data type. Instead, JSON is stored as
`NVARCHAR(MAX)` text, and SQL Server ships a set of built-in functions to parse,
query, modify, and validate that text as JSON. This design (introduced in SQL
Server 2016) means JSON data behaves like any other string column for storage,
but gains JSON-aware behavior through functions and optional constraints/indexes.

Key building blocks:

| Function | Purpose |
|---|---|
| `ISJSON()` | Validates that a string contains valid JSON |
| `JSON_VALUE()` | Extracts a scalar value from JSON |
| `JSON_QUERY()` | Extracts an object or array from JSON |
| `JSON_MODIFY()` | Updates a value inside JSON text |
| `OPENJSON()` | Converts JSON into a relational rowset (table-valued function) |
| `FOR JSON` | Converts relational query results into JSON |
| `JSON_PATH_EXISTS()` | Tests if a path exists in JSON |
| `JSON_ARRAY()` / `JSON_OBJECT()` (2022+) | Construct JSON inline |

---

## 2. Creating a Table with a JSON Column

```sql
CREATE TABLE Products (
    ProductId   INT IDENTITY PRIMARY KEY,
    Name        NVARCHAR(200) NOT NULL,
    Attributes  NVARCHAR(MAX) NULL,       -- holds JSON text
    CONSTRAINT CHK_Attributes_IsJson CHECK (ISJSON(Attributes) = 1)
);
```

Best practice: always add an `ISJSON()` CHECK constraint on any column intended
to store JSON — SQL Server won't stop you from inserting invalid JSON otherwise.

---

## 3. Inserting JSON Data

```sql
INSERT INTO Products (Name, Attributes)
VALUES (
    'Wireless Mouse',
    N'{
        "color": "black",
        "wireless": true,
        "price": 24.99,
        "dimensions": { "width": 6.5, "height": 3.2 },
        "tags": ["electronics", "peripheral", "office"]
    }'
);
```

You can also build JSON from relational data using `FOR JSON` (see §6) or, on
SQL Server 2022+, `JSON_OBJECT()` / `JSON_ARRAY()`:

```sql
-- SQL Server 2022+
SELECT JSON_OBJECT('id': ProductId, 'name': Name) AS Json
FROM Products;
```

---

## 4. Querying JSON — Extracting Values

### 4.1 `JSON_VALUE()` — scalar values

```sql
SELECT
    ProductId,
    Name,
    JSON_VALUE(Attributes, '$.color')            AS Color,
    JSON_VALUE(Attributes, '$.dimensions.width')  AS Width,
    CAST(JSON_VALUE(Attributes, '$.price') AS DECIMAL(10,2)) AS Price
FROM Products;
```

`JSON_VALUE` only returns scalar (string/number/bool) values, truncated to
NVARCHAR(4000). It returns `NULL` if the path doesn't exist or points to an
object/array.

### 4.2 `JSON_QUERY()` — objects/arrays

```sql
SELECT
    ProductId,
    JSON_QUERY(Attributes, '$.dimensions') AS Dimensions,  -- returns JSON object
    JSON_QUERY(Attributes, '$.tags')       AS Tags          -- returns JSON array
FROM Products;
```

### 4.3 Filtering with `WHERE`

```sql
-- filter on a scalar
SELECT * FROM Products
WHERE JSON_VALUE(Attributes, '$.color') = 'black';

-- filter on existence of a path
SELECT * FROM Products
WHERE JSON_PATH_EXISTS(Attributes, '$.dimensions') = 1;

-- filter using LIKE against JSON_QUERY (array contains a value)
SELECT * FROM Products
WHERE JSON_QUERY(Attributes, '$.tags') LIKE '%"office"%';
```

For real "does array contain X" logic, prefer `OPENJSON` (below) over `LIKE`.

---

## 5. `OPENJSON()` — Shredding JSON into Rows

`OPENJSON` is the workhorse for turning JSON into relational rows, especially
for arrays and for joining nested JSON to other tables.

### 5.1 Default schema (key/value/type)

```sql
SELECT * FROM OPENJSON('{"a":1,"b":"x","c":[1,2,3]}');
```

Returns columns: `key`, `value`, `type` (1=string, 2=number, 3=bool, 4=array,
5=object, 0=null).

### 5.2 Explicit schema with `WITH`

```sql
SELECT p.ProductId, j.Color, j.Width, j.Height
FROM Products p
CROSS APPLY OPENJSON(p.Attributes)
WITH (
    Color  NVARCHAR(50)   '$.color',
    Width  DECIMAL(10,2)  '$.dimensions.width',
    Height DECIMAL(10,2)  '$.dimensions.height'
) AS j;
```

### 5.3 Expanding a nested array

```sql
SELECT p.ProductId, t.value AS Tag
FROM Products p
CROSS APPLY OPENJSON(p.Attributes, '$.tags') AS t;
```

### 5.4 Using OPENJSON with a variable / parameter (table-valued parameter alternative)

```sql
DECLARE @json NVARCHAR(MAX) = N'[{"id":1,"qty":5},{"id":2,"qty":3}]';

SELECT *
FROM OPENJSON(@json)
WITH (
    Id  INT '$.id',
    Qty INT '$.qty'
);
```

This pattern is commonly used to pass a JSON array from an application as a
single parameter and shred it server-side instead of using a TVP.

---

## 6. `FOR JSON` — Producing JSON from Relational Data

### 6.1 `FOR JSON AUTO`

```sql
SELECT ProductId, Name
FROM Products
FOR JSON AUTO;
```

Automatically nests based on table/join structure.

### 6.2 `FOR JSON PATH` (more control)

```sql
SELECT
    ProductId AS 'id',
    Name AS 'name',
    JSON_VALUE(Attributes, '$.color') AS 'attributes.color'
FROM Products
FOR JSON PATH, ROOT('products');
```

`FOR JSON PATH` lets you use dotted aliases (`'attributes.color'`) to build
nested output objects.

### 6.3 Options

- `INCLUDE_NULL_VALUES` — keep NULL properties (omitted by default)
- `WITHOUT_ARRAY_WRAPPER` — return a single JSON object instead of an array
- `ROOT('name')` — wrap output under a named root property

```sql
SELECT ProductId, Name
FROM Products
FOR JSON PATH, INCLUDE_NULL_VALUES, WITHOUT_ARRAY_WRAPPER;
```

---

## 7. Modifying JSON — `JSON_MODIFY()`

```sql
-- update an existing scalar property
UPDATE Products
SET Attributes = JSON_MODIFY(Attributes, '$.price', 19.99)
WHERE ProductId = 1;

-- add a new property (created if it doesn't exist)
UPDATE Products
SET Attributes = JSON_MODIFY(Attributes, '$.discontinued', CAST(0 AS BIT))
WHERE ProductId = 1;

-- remove a property (set to NULL removes the key)
UPDATE Products
SET Attributes = JSON_MODIFY(Attributes, '$.discontinued', NULL)
WHERE ProductId = 1;

-- append to an array
UPDATE Products
SET Attributes = JSON_MODIFY(
    Attributes,
    'append $.tags',
    'sale'
)
WHERE ProductId = 1;

-- chained updates in a single statement
UPDATE Products
SET Attributes = JSON_MODIFY(
                    JSON_MODIFY(Attributes, '$.color', 'red'),
                    '$.price', 29.99)
WHERE ProductId = 1;
```

To insert a nested value from a variable, wrap it with `JSON_QUERY()` so it's
treated as JSON rather than an escaped string:

```sql
DECLARE @dims NVARCHAR(100) = N'{"width":10,"height":5}';
UPDATE Products
SET Attributes = JSON_MODIFY(Attributes, '$.dimensions', JSON_QUERY(@dims))
WHERE ProductId = 1;
```

---

## 8. Indexing JSON Data

SQL Server can't index a JSON string directly, but two techniques give
index-level performance:

### 8.1 Computed columns + index (most common)

```sql
ALTER TABLE Products
ADD Color AS JSON_VALUE(Attributes, '$.color');

CREATE INDEX IX_Products_Color ON Products(Color);
```

Queries filtering on `JSON_VALUE(Attributes, '$.color')` will now use
`IX_Products_Color` as long as the predicate matches the computed column
expression (SQL Server matches expressions automatically for simple cases).

### 8.2 Full-text / JSON path search

For deep or full-document search, combine computed columns for the most
selective fields with full-text indexing, or consider a hybrid model where
frequently filtered attributes are promoted to real relational columns and
only variable/optional attributes stay in the JSON blob.

---

## 9. Validation & Constraints

```sql
ALTER TABLE Products
ADD CONSTRAINT CHK_Attributes_Json CHECK (Attributes IS NULL OR ISJSON(Attributes) = 1);
```

Combine with `JSON_PATH_EXISTS` for required-field checks:

```sql
ALTER TABLE Products
ADD CONSTRAINT CHK_Attributes_HasColor
CHECK (JSON_PATH_EXISTS(Attributes, '$.color') = 1);
```

---

## 10. Common Patterns

### 10.1 Merge/upsert with JSON payload from application

```sql
DECLARE @json NVARCHAR(MAX) = N'{"id":1,"name":"Mouse","price":24.99}';

MERGE Products AS target
USING (
    SELECT * FROM OPENJSON(@json)
    WITH (Id INT '$.id', Name NVARCHAR(200) '$.name', Price DECIMAL(10,2) '$.price')
) AS source
ON target.ProductId = source.Id
WHEN MATCHED THEN UPDATE SET Name = source.Name
WHEN NOT MATCHED THEN INSERT (Name) VALUES (source.Name);
```

### 10.2 Aggregating JSON arrays

```sql
SELECT p.ProductId, COUNT(t.value) AS TagCount
FROM Products p
CROSS APPLY OPENJSON(p.Attributes, '$.tags') t
GROUP BY p.ProductId;
```

---

## 11. Performance & Best Practices

1. **Always validate** JSON on write with an `ISJSON()` CHECK constraint.
2. **Promote hot fields** to computed columns + indexes rather than repeatedly
   parsing JSON at query time.
3. **Prefer `JSON_VALUE` over `JSON_QUERY` + `LIKE`** for scalar filters — it's
   far more efficient and index-friendly via computed columns.
4. **Watch the 4000-character limit** of `JSON_VALUE`'s return — large text
   values need `JSON_QUERY` or a redesign.
5. **Avoid storing highly relational data as JSON** just for convenience;
   JSON columns are best for sparse, variable, or semi-structured attributes
   (e.g. product specs, user preferences, event payloads), not for data you
   need to join/aggregate heavily.
6. **Use `OPENJSON` with an explicit `WITH` schema** in hot paths — it's
   faster than the default key/value/type shredding because SQL Server can
   push type conversion down more efficiently.
7. **Batch JSON-heavy `UPDATE`s** — `JSON_MODIFY` rewrites the whole string
   each call; chain modifications in one statement rather than one `UPDATE`
   per property.
8. **Consider `SQL_VARIANT` vs JSON** — for simple key/value pairs, a proper
   EAV table or sparse columns may outperform JSON depending on query patterns.

---

## 12. Quick Reference Cheat Sheet

```sql
ISJSON(json_text)                              -- 1/0 validity check
JSON_VALUE(json_col, '$.path')                 -- scalar extraction
JSON_QUERY(json_col, '$.path')                 -- object/array extraction
JSON_MODIFY(json_col, '$.path', new_value)     -- update/insert/delete a property
JSON_PATH_EXISTS(json_col, '$.path')           -- 1/0 path exists
OPENJSON(json_col [, '$.path'])                -- shred JSON into rows
... FOR JSON AUTO | PATH [, ROOT('name')]      -- produce JSON from a query
JSON_OBJECT('key': value, ...)                 -- build JSON object (2022+)
JSON_ARRAY(value, ...)                         -- build JSON array (2022+)
```
