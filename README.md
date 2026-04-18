# SQL Keys & Constraints — A Complete Guide

> The rules that keep your data honest — explained step by step with examples, violations, and real syntax.

---

## 📌 What are Keys & Constraints?

In SQL, **keys** identify rows uniquely and link tables together. **Constraints** are rules that enforce the integrity of your data — they prevent invalid, duplicate, or missing values from ever entering the database.

> **Why this matters:** A database without constraints is like a spreadsheet with no validation. Anyone can enter duplicate IDs, blank required fields, or references to rows that don't exist. Constraints make the database itself enforce correctness — not just your application code.

---

## 🗂️ Types of Keys & Constraints

| # | Type | Description |
|---|------|-------------|
| 1 | **PRIMARY KEY** | Uniquely identifies each row |
| 2 | **FOREIGN KEY** | Links rows across tables |
| 3 | **UNIQUE** | No duplicate values allowed |
| 4 | **NOT NULL** | Value must always be provided |
| 5 | **CHECK** | Value must pass a custom condition |
| 6 | **DEFAULT** | Fills in a value automatically |
| 7 | **INDEX** | Speeds up search queries |

---

## 01 — PRIMARY KEY *(Unique + Not Null)*

A PRIMARY KEY **uniquely identifies every row** in a table. No two rows can share the same primary key value, and it can never be NULL.

Every table should have exactly one primary key. It can be a single column (simple key) or a combination of columns (composite key). The database automatically creates an index on the primary key.

### Syntax

```sql
-- Single-column primary key
CREATE TABLE students (
    student_id  INT          PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(150)
);

-- Composite primary key (defined at table level)
CREATE TABLE enrollments (
    student_id  INT,
    course_id   INT,
    enrolled_on DATE,
    PRIMARY KEY (student_id, course_id)
);
```

### How it Works — Step by Step

1. You declare one column (or a combination) as the PRIMARY KEY.
2. On every INSERT or UPDATE, the database checks: is this value already used? Is it NULL?
3. If either check fails → the operation is rejected with an error.
4. If both pass → the row is accepted and stored.

### Worked Trace

**Existing Data:**

| student_id | name  |
|-----------|-------|
| 1         | Alice |
| 2         | Bob   |

**New INSERT attempts:**

| Value | Result |
|-------|--------|
| `student_id = 3` | ✅ Unique & not null → **ACCEPTED** |
| `student_id = 1` | ❌ Duplicate value → **REJECTED** |
| `student_id = NULL` | ❌ NULL not allowed → **REJECTED** |

> **How to read this:** `student_id = 3` is brand new and not NULL → inserted successfully. `student_id = 1` already exists → database rejects it with a *duplicate key* error. `student_id = NULL` violates the not-null rule → rejected immediately.

**Errors you'll see:**
```
ERROR: duplicate key value violates unique constraint "students_pkey"
ERROR: null value in column "student_id" violates not-null constraint
```

---

## 02 — FOREIGN KEY *(Referential Integrity)*

A FOREIGN KEY **links a column in one table to the PRIMARY KEY of another**. It ensures you can't reference a row that doesn't exist.

Foreign keys enforce *referential integrity* — the guarantee that every relationship in your database points to something real. You can't add an order for a customer who doesn't exist, and you can't delete a customer who still has orders (unless you use CASCADE).

### Syntax

```sql
CREATE TABLE orders (
    order_id    INT  PRIMARY KEY,
    customer_id INT,
    amount      DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- With cascade options
FOREIGN KEY (customer_id)
    REFERENCES customers(customer_id)
    ON DELETE CASCADE    -- delete orders when customer is deleted
    ON UPDATE CASCADE;   -- update orders if customer_id changes
```

### How it Works — Step by Step

1. You declare a column as a FOREIGN KEY referencing another table's primary key.
2. On every INSERT or UPDATE in the child table, the database checks: does this value exist in the parent table?
3. If the value doesn't exist → the operation is rejected.
4. On DELETE/UPDATE of the parent row, the ON DELETE/ON UPDATE rule determines what happens to children.

### Worked Trace

**customers (parent):**

| customer_id | name  |
|------------|-------|
| 101        | Alice |
| 102        | Bob   |

**New INSERT attempts into orders:**

| Value | Result |
|-------|--------|
| `customer_id = 102` | ✅ Exists in customers → **ACCEPTED** |
| `customer_id = 999` | ❌ Not in customers → **REJECTED** |
| `customer_id = NULL` | ⚠️ Allowed (unless NOT NULL also set) |

> **How to read this:** `customer_id = 102` exists in the customers table → order inserted successfully. `customer_id = 999` has no matching row in customers → rejected. This prevents "orphan" orders that point to nobody. NULL is allowed by default unless you also add NOT NULL.

### CASCADE Options

| Option | Effect on child rows when parent is deleted |
|--------|---------------------------------------------|
| `ON DELETE CASCADE` | Child rows are automatically deleted too |
| `ON DELETE SET NULL` | FK column set to NULL in child rows |
| `ON DELETE RESTRICT` | Parent delete is blocked if children exist *(default)* |
| `ON DELETE NO ACTION` | Like RESTRICT but checked at end of transaction |

---

## 03 — UNIQUE *(No Duplicates)*

A UNIQUE constraint ensures **no two rows have the same value** in the specified column(s). Unlike PRIMARY KEY, a UNIQUE column can hold NULL.

Use UNIQUE for natural identifiers that aren't the primary key — like email addresses, usernames, or phone numbers. A table can have multiple UNIQUE constraints but only one PRIMARY KEY.

### Syntax

```sql
CREATE TABLE users (
    user_id  INT          PRIMARY KEY,
    email    VARCHAR(200) UNIQUE,        -- inline
    username VARCHAR(50),
    phone    VARCHAR(20),
    UNIQUE (username),                   -- table level
    UNIQUE (phone)
);
```

### How it Works — Step by Step

1. On INSERT or UPDATE, the database scans existing values in the UNIQUE column.
2. If the new value already exists → operation is rejected.
3. If the value is NULL → it passes (most databases allow multiple NULLs in a UNIQUE column).
4. If the value is brand new → row is inserted/updated successfully.

### Worked Trace

**Existing Data:**

| user_id | email | username |
|---------|-------|----------|
| 1 | alice@x.com | alice |
| 2 | bob@x.com | bob |

**New INSERT attempts:**

| Value | Result |
|-------|--------|
| email = `carol@x.com` | ✅ Not duplicate → **ACCEPTED** |
| email = `alice@x.com` | ❌ Duplicate → **REJECTED** |
| email = `NULL` | ✅ NULL allowed in UNIQUE → **ACCEPTED** |
| username = `alice` | ❌ Duplicate username → **REJECTED** |

> **How to read this:** New unique values pass freely. Duplicate non-null values are always rejected. NULL is special — it represents "unknown", so most databases treat multiple NULLs as *not equal* to each other, allowing several rows to have NULL in a UNIQUE column.
>
> **Difference from PRIMARY KEY:** UNIQUE allows NULL; PRIMARY KEY does not. A table can have many UNIQUE constraints, but only one PRIMARY KEY.

---

## 04 — NOT NULL *(Value Required)*

NOT NULL ensures a column **always has a value** — it can never be left empty (NULL). It is the simplest and most common constraint.

By default, every column in SQL accepts NULL unless you explicitly add NOT NULL. Use it for fields that must always be present: names, dates, status codes, prices, and so on.

### Syntax

```sql
CREATE TABLE products (
    product_id   INT           PRIMARY KEY,
    product_name VARCHAR(200)  NOT NULL,   -- must be provided
    price        DECIMAL(8,2)  NOT NULL,   -- must be provided
    description  TEXT                      -- nullable (optional)
);
```

### How it Works — Step by Step

1. When a row is inserted or updated, the database checks every NOT NULL column.
2. If a NOT NULL column has no value supplied (or explicitly NULL) → rejected immediately.
3. If all NOT NULL columns have values → row proceeds to other constraint checks.

### Worked Trace

**NOT NULL columns: product_name, price**

| INSERT attempt | Result |
|----------------|--------|
| `(1, "Laptop", 999.99, "Fast laptop")` | ✅ All NOT NULL fields present → **ACCEPTED** |
| `(2, "Mouse", 29.99, NULL)` | ✅ description is nullable → **ACCEPTED** |
| `(3, NULL, 49.99, "Keyboard")` | ❌ product_name is NULL → **REJECTED** |
| `(4, "Monitor", NULL, NULL)` | ❌ price is NULL → **REJECTED** |

> **How to read this:** Row 2 is valid — `description` has no NOT NULL constraint so NULL is fine. Row 3 fails because a product with no name makes no sense. Row 4 fails because a product must always have a price. The `description` column is intentionally optional.

---

## 05 — CHECK *(Custom Condition)*

A CHECK constraint lets you define a **custom rule that every row must satisfy**. If the condition evaluates to false, the row is rejected.

CHECK is the most flexible constraint — you can write any valid SQL expression. Common uses: age range, positive salary, or a fixed set of allowed status values.

### Syntax

```sql
CREATE TABLE employees (
    emp_id   INT          PRIMARY KEY,
    name     VARCHAR(100) NOT NULL,
    age      INT          CHECK (age >= 18 AND age <= 65),
    salary   DECIMAL(10,2) CHECK (salary > 0),
    status   VARCHAR(20)  CHECK (status IN ('active', 'inactive', 'on_leave'))
);
```

### How it Works — Step by Step

1. On INSERT or UPDATE, the database evaluates the CHECK expression with the new column value.
2. If the expression returns TRUE → row is accepted.
3. If the expression returns FALSE → row is rejected with an error.
4. If the expression returns NULL (e.g., column is NULL) → row is accepted (NULL is not FALSE).

### Worked Trace

**CHECK constraints: age (18–65), salary (>0), status (active/inactive/on_leave)**

| INSERT attempt | Result |
|----------------|--------|
| `age=30, salary=50000, status='active'` | ✅ All checks pass → **ACCEPTED** |
| `age = 15` | ❌ `15 >= 18` is FALSE → **REJECTED** |
| `salary = -500` | ❌ `-500 > 0` is FALSE → **REJECTED** |
| `status = 'fired'` | ❌ Not in allowed list → **REJECTED** |
| `age = NULL` | ⚠️ NULL skips CHECK (not FALSE) → **ACCEPTED** |

> **How to read this:** All three conditions must evaluate TRUE. The NULL edge case: when the column is NULL, CHECK evaluates to NULL (not FALSE), so the row passes. To also block NULLs, combine `CHECK` with `NOT NULL`.

---

## 06 — DEFAULT *(Auto-fill Value)*

DEFAULT **automatically fills in a value** when no value is provided during INSERT. It doesn't restrict — it helps.

DEFAULT is the only constraint that adds data rather than rejecting it. Common uses: timestamps (`created_at`), boolean flags (`is_active`), or any field with a sensible starting value.

### Syntax

```sql
CREATE TABLE orders (
    order_id    INT          PRIMARY KEY,
    customer_id INT          NOT NULL,
    status      VARCHAR(20)  DEFAULT 'pending',
    created_at  TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    quantity    INT          DEFAULT 1,
    is_paid     BOOLEAN      DEFAULT FALSE
);
```

### How it Works — Step by Step

1. On INSERT, the database checks each DEFAULT column: was a value provided?
2. If yes → use the provided value.
3. If no (column omitted from INSERT) → substitute the DEFAULT value automatically.
4. If NULL is explicitly inserted → NULL is used (overrides DEFAULT).

### Worked Trace

**DEFAULT columns: status='pending', quantity=1, is_paid=FALSE**

| INSERT attempt | Result |
|----------------|--------|
| `INSERT (1, 101, 'shipped', NOW(), 3, TRUE)` | ✅ All values provided → used as-is |
| `INSERT (2, 101)` — omit status, qty, is_paid | ✅ Defaults fill in: `'pending'`, `1`, `FALSE` |
| `INSERT (3, 101, NULL, ...)` | ⚠️ Explicit NULL overrides DEFAULT → `status = NULL` |

**Resulting stored data:**

| order_id | customer_id | status | quantity | is_paid |
|----------|------------|--------|----------|---------|
| 1 | 101 | shipped | 3 | TRUE |
| 2 | 101 | pending | 1 | FALSE |
| 3 | 101 | NULL | 1 | FALSE |

> **How to read this:** Row 2 had only `order_id` and `customer_id` provided → `status`, `quantity`, `is_paid` were auto-filled from DEFAULT. Row 3 had `status = NULL` explicitly written — DEFAULT is *not* applied when NULL is explicit. To prevent this, also add NOT NULL to the column.

---

## 07 — INDEX *(Performance Booster)*

An INDEX is a **lookup structure that speeds up queries** on a column. It doesn't restrict data — it makes finding it faster.

Without an index, the database must scan every row (a *full table scan*) to find matching values. An index works like a book's index — you jump straight to the right page instead of reading every page.

### Syntax

```sql
-- Create a regular index
CREATE INDEX idx_orders_customer
ON orders (customer_id);

-- Create a unique index (same effect as UNIQUE constraint)
CREATE UNIQUE INDEX idx_users_email
ON users (email);

-- Composite index (for queries filtering on both columns)
CREATE INDEX idx_orders_customer_date
ON orders (customer_id, order_date);

-- Drop an index
DROP INDEX idx_orders_customer;
```

### How it Works — Step by Step

1. When you create an index, the database builds a sorted B-tree structure of the column's values, each pointing to the actual row location.
2. When a query filters or sorts by that column, the database uses the index to locate matching rows instantly.
3. On INSERT/UPDATE/DELETE, the index is automatically updated to stay in sync.

### Worked Trace

**Table: orders (1,000,000 rows) — query: `WHERE customer_id = 102`**

| Scenario | Rows Read | Time |
|----------|-----------|------|
| Without index — full table scan | 1,000,000 rows | ~800ms |
| With index on customer_id — B-tree lookup | ~20 rows | ~1ms |

> **How to read this:** Without an index, finding all orders for `customer_id = 102` requires checking every single row. With an index, the database jumps directly to the matching rows.
>
> **When to add an index:** columns used frequently in `WHERE`, `JOIN ON`, or `ORDER BY` clauses.  
> **When NOT to:** small tables (full scan is fast anyway), columns with very few distinct values, or tables with very frequent INSERT/UPDATE.

> **Note:** PRIMARY KEY and UNIQUE constraints automatically create an index. You don't need to create one manually for those columns.

---

## 📋 Quick Reference Summary

| Constraint | Allows NULL? | Allows Duplicates? | Rejects Data? | Per Table |
|------------|-------------|-------------------|---------------|-----------|
| **PRIMARY KEY** | No | No | Yes | Only 1 |
| **FOREIGN KEY** | Yes* | Yes | Yes | Many |
| **UNIQUE** | Yes | No | Yes | Many |
| **NOT NULL** | No | Yes | Yes | Many |
| **CHECK** | Yes | Yes | Yes | Many |
| **DEFAULT** | Yes | Yes | No | Many |
| **INDEX** | Yes | Yes* | No | Many |

*FOREIGN KEY allows NULL unless combined with NOT NULL. UNIQUE INDEX does not allow duplicates.*

---

## 🔑 Key Takeaways

- **PRIMARY KEY** — Every table needs one. It's the unique, non-null identity of every row.
- **FOREIGN KEY** — Use to link tables. Prevents orphan records and broken references.
- **UNIQUE** — Like PRIMARY KEY for alternate identifiers (email, username). Allows NULL.
- **NOT NULL** — The simplest constraint. Use it for every field that must always have a value.
- **CHECK** — Custom validation logic. Watch out for the NULL edge case.
- **DEFAULT** — Reduces boilerplate in INSERT statements. Explicit NULL overrides it.
- **INDEX** — Not a constraint but critical for performance. Add to frequently queried columns.

---

*SQL Keys & Constraints Reference Guide · Based on standard SQL (ANSI)*
