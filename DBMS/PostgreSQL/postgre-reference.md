# PostgreSQL Complete Reference Guide

> A structured, in-depth reference covering data types, query categories, operators, joins, set operations, constraints, indexes, and more — for production use and interview prep.

---

## 1. SQL Command Categories Overview

| Category | Full Form | Purpose | Common Commands |
|---|---|---|---|
| **DDL** | Data Definition Language | Define/modify schema structure | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `COMMENT`, `RENAME` |
| **DML** | Data Manipulation Language | Manipulate data inside tables | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE` (PG 15+) |
| **DCL** | Data Control Language | Control access/permissions | `GRANT`, `REVOKE` |
| **TCL** | Transaction Control Language | Manage transactions | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `SET TRANSACTION` |
| **DQL** | Data Query Language (sometimes split from DML) | Retrieve data | `SELECT` |

**Key interview point:** `TRUNCATE` is DDL (not DML) in PostgreSQL because it cannot be rolled back in older systems and resets identity columns — but in PostgreSQL specifically, `TRUNCATE` *is* transactional (can be rolled back inside a transaction block), which is a PostgreSQL-specific nuance worth mentioning.

---

## 2. PostgreSQL Data Types (Full Table)

### 2.1 Numeric Types

| Type | Storage | Range / Precision | Notes |
|---|---|---|---|
| `smallint` | 2 bytes | -32768 to 32767 | Small range integer |
| `integer` / `int` / `int4` | 4 bytes | -2147483648 to 2147483647 | Default integer type |
| `bigint` / `int8` | 8 bytes | -9223372036854775808 to 9223372036854775807 | Use for IDs in large systems |
| `decimal` / `numeric` | Variable | Up to 131072 digits before decimal, 16383 after | Exact, use for money/finance |
| `real` / `float4` | 4 bytes | 6 decimal digits precision | Inexact, fast |
| `double precision` / `float8` | 8 bytes | 15 decimal digits precision | Inexact, fast |
| `smallserial` | 2 bytes | 1 to 32767 | Auto-increment small int |
| `serial` | 4 bytes | 1 to 2147483647 | Auto-increment int (creates sequence) |
| `bigserial` | 8 bytes | 1 to large range | Auto-increment bigint |
| `money` | 8 bytes | Locale-dependent | Rarely used in production; prefer `numeric` |

### 2.2 Character Types

| Type | Description | Notes |
|---|---|---|
| `char(n)` / `character(n)` | Fixed-length, blank padded | Rarely used; wastes space |
| `varchar(n)` / `character varying(n)` | Variable-length with limit | Most common for bounded strings |
| `text` | Variable, unlimited length | Preferred in PostgreSQL — no real performance difference vs varchar |

### 2.3 Date/Time Types

| Type | Storage | Range | Notes |
|---|---|---|---|
| `date` | 4 bytes | 4713 BC – 5874897 AD | Date only |
| `time [without tz]` | 8 bytes | 00:00:00 – 24:00:00 | Time only |
| `time with time zone` | 12 bytes | Same + TZ offset | Rarely recommended |
| `timestamp [without tz]` | 8 bytes | 4713 BC – 294276 AD | Date + time, no TZ |
| `timestamp with time zone` (`timestamptz`) | 8 bytes | Same | **Recommended for production** — stores in UTC internally |
| `interval` | 16 bytes | -178000000 to 178000000 years | Duration of time |

### 2.4 Boolean

| Type | Storage | Values |
|---|---|---|
| `boolean` / `bool` | 1 byte | `TRUE`, `FALSE`, `NULL` (also accepts `'t'/'f'/'yes'/'no'/'1'/'0'`) |

### 2.5 Binary Data

| Type | Description |
|---|---|
| `bytea` | Binary string ("byte array") for raw binary data |

### 2.6 JSON Types

| Type | Description | Notes |
|---|---|---|
| `json` | Stores exact text copy of JSON input | Slower; preserves formatting/whitespace/key order |
| `jsonb` | Stores decomposed binary JSON | **Preferred** — supports indexing (GIN), faster querying, no duplicate keys |

### 2.7 Arrays

| Syntax | Description |
|---|---|
| `integer[]`, `text[]`, etc. | PostgreSQL supports arrays of any data type |
| `ARRAY[1,2,3]` | Array constructor |

### 2.8 UUID

| Type | Description |
|---|---|
| `uuid` | Universally Unique Identifier, 128-bit, often used as PK (`gen_random_uuid()` from `pgcrypto` or built-in `uuidv7()` in PG18) |

### 2.9 Geometric Types

| Type | Description |
|---|---|
| `point`, `line`, `lseg`, `box`, `path`, `polygon`, `circle` | For 2D geometric data |

### 2.10 Network Address Types

| Type | Description |
|---|---|
| `cidr` | IPv4/IPv6 network |
| `inet` | IPv4/IPv6 host address |
| `macaddr` | MAC address |
| `macaddr8` | EUI-64 MAC address |

### 2.11 Range Types

| Type | Underlying | Description |
|---|---|---|
| `int4range`, `int8range` | integer/bigint | Range of integers |
| `numrange` | numeric | Range of numerics |
| `tsrange`, `tstzrange` | timestamp | Range of timestamps |
| `daterange` | date | Range of dates |

### 2.12 Other Special Types

| Type | Description |
|---|---|
| `enum` | User-defined enumerated type (`CREATE TYPE mood AS ENUM ('sad','ok','happy')`) |
| `xml` | XML data |
| `tsvector` / `tsquery` | Full text search types |
| `bit(n)` / `bit varying(n)` | Fixed/variable bit strings |
| `composite types` | User-defined row types |
| `domain` | User-defined constrained type alias |

---

## 3. DDL — Data Definition Language

### 3.1 CREATE

```sql
-- Database
CREATE DATABASE company_db;

-- Schema
CREATE SCHEMA hr;

-- Table with constraints
CREATE TABLE employees (
    emp_id      BIGSERIAL PRIMARY KEY,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    email       VARCHAR(100) UNIQUE NOT NULL,
    salary      NUMERIC(10,2) CHECK (salary > 0),
    dept_id     INTEGER REFERENCES departments(dept_id) ON DELETE SET NULL,
    hired_at    TIMESTAMPTZ DEFAULT now(),
    is_active   BOOLEAN DEFAULT TRUE
);

-- Index
CREATE INDEX idx_emp_dept ON employees(dept_id);
CREATE UNIQUE INDEX idx_emp_email ON employees(email);

-- View
CREATE VIEW active_employees AS
SELECT * FROM employees WHERE is_active = TRUE;

-- Materialized View (stores data physically, must REFRESH)
CREATE MATERIALIZED VIEW dept_salary_summary AS
SELECT dept_id, SUM(salary) AS total_salary FROM employees GROUP BY dept_id;

-- Sequence
CREATE SEQUENCE order_seq START 1000 INCREMENT 1;

-- Custom Type
CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');
```

### 3.2 ALTER

```sql
ALTER TABLE employees ADD COLUMN phone VARCHAR(15);
ALTER TABLE employees DROP COLUMN phone;
ALTER TABLE employees ALTER COLUMN salary SET DEFAULT 30000;
ALTER TABLE employees ALTER COLUMN salary TYPE NUMERIC(12,2);
ALTER TABLE employees RENAME COLUMN first_name TO fname;
ALTER TABLE employees RENAME TO staff;
ALTER TABLE employees ADD CONSTRAINT chk_salary CHECK (salary >= 0);
ALTER TABLE employees DROP CONSTRAINT chk_salary;
```

### 3.3 DROP / TRUNCATE

```sql
DROP TABLE IF EXISTS employees CASCADE;   -- CASCADE drops dependent objects
DROP VIEW active_employees;
DROP INDEX idx_emp_dept;
TRUNCATE TABLE employees RESTART IDENTITY CASCADE;  -- Fast delete-all, resets serial
```

| TRUNCATE vs DELETE vs DROP | Speed | Rollback (in txn) | Resets identity | Removes structure | Fires triggers |
|---|---|---|---|---|---|
| `DELETE` | Slow (row-by-row, logged) | Yes | No | No | Yes (row-level) |
| `TRUNCATE` | Fast (deallocates pages) | Yes (PG-specific) | Yes (with RESTART IDENTITY) | No | Only statement-level triggers |
| `DROP` | Fast | Yes | N/A | Yes — removes table entirely | N/A |

### 3.4 COMMENT & RENAME

```sql
COMMENT ON TABLE employees IS 'Stores employee master data';
ALTER TABLE employees RENAME TO emp_master;
```

---

## 4. DML — Data Manipulation Language

### 4.1 INSERT

```sql
INSERT INTO employees (first_name, last_name, email, salary, dept_id)
VALUES ('John', 'Doe', 'john@co.com', 55000, 2);

-- Multi-row insert
INSERT INTO employees (first_name, last_name, email, salary)
VALUES 
  ('Jane', 'Smith', 'jane@co.com', 60000),
  ('Bob', 'Lee', 'bob@co.com', 45000);

-- Insert from SELECT
INSERT INTO archived_employees SELECT * FROM employees WHERE is_active = FALSE;

-- UPSERT (PostgreSQL-specific)
INSERT INTO employees (emp_id, email, salary)
VALUES (1, 'john@co.com', 70000)
ON CONFLICT (emp_id)
DO UPDATE SET salary = EXCLUDED.salary;

-- RETURNING clause (PostgreSQL feature)
INSERT INTO employees (first_name, email) VALUES ('Amy','amy@co.com')
RETURNING emp_id, first_name;
```

### 4.2 UPDATE

```sql
UPDATE employees
SET salary = salary * 1.10
WHERE dept_id = 3;

-- Update with JOIN-like subquery
UPDATE employees e
SET salary = salary + 5000
FROM departments d
WHERE e.dept_id = d.dept_id AND d.dept_name = 'Engineering';

UPDATE employees SET salary = salary + 1000
WHERE emp_id = 5 RETURNING *;
```

### 4.3 DELETE

```sql
DELETE FROM employees WHERE is_active = FALSE;

-- Delete using join condition
DELETE FROM employees e
USING departments d
WHERE e.dept_id = d.dept_id AND d.dept_name = 'Closed Dept';

DELETE FROM employees WHERE salary < 20000 RETURNING emp_id;
```

### 4.4 MERGE (PostgreSQL 15+)

```sql
MERGE INTO employees AS tgt
USING staging_employees AS src
ON tgt.emp_id = src.emp_id
WHEN MATCHED THEN
    UPDATE SET salary = src.salary
WHEN NOT MATCHED THEN
    INSERT (emp_id, first_name, salary) VALUES (src.emp_id, src.first_name, src.salary);
```

### 4.5 SELECT — Full Anatomy

```sql
SELECT [DISTINCT] column_list
FROM table_name
[JOIN ...]
[WHERE condition]
[GROUP BY columns]
[HAVING condition]
[ORDER BY columns [ASC|DESC]]
[LIMIT n]
[OFFSET n];
```

**Logical order of execution** (critical interview question):
```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT (with window funcs) → DISTINCT → ORDER BY → LIMIT/OFFSET
```
> Note: This differs from the *written* order, which is `SELECT...FROM...WHERE...GROUP BY...HAVING...ORDER BY...LIMIT`.

---

## 5. Constraints

| Constraint | Purpose | Example |
|---|---|---|
| `PRIMARY KEY` | Unique + Not Null identifier | `id SERIAL PRIMARY KEY` |
| `FOREIGN KEY` | Referential integrity | `dept_id INT REFERENCES departments(id)` |
| `UNIQUE` | No duplicate values | `email VARCHAR UNIQUE` |
| `NOT NULL` | Disallow null | `name VARCHAR NOT NULL` |
| `CHECK` | Custom validation rule | `CHECK (age >= 18)` |
| `DEFAULT` | Default value if none given | `status VARCHAR DEFAULT 'active'` |
| `EXCLUDE` | Excludes overlapping values (e.g., ranges) | `EXCLUDE USING gist (room WITH =, during WITH &&)` |

**FK referential actions:**

| Action | Behavior |
|---|---|
| `ON DELETE CASCADE` | Deletes child rows when parent deleted |
| `ON DELETE SET NULL` | Sets FK to NULL |
| `ON DELETE SET DEFAULT` | Sets FK to default value |
| `ON DELETE RESTRICT` | Prevents delete if children exist (default-like) |
| `ON DELETE NO ACTION` | Similar to RESTRICT but checked at end of statement/deferred |

---

## 6. Operators (Categorized)

### 6.1 Arithmetic

| Operator | Meaning |
|---|---|
| `+ - * /` | Add, subtract, multiply, divide |
| `%` | Modulo |
| `^` | Exponent |
| `\|/` | Square root |
| `\|\|/` | Cube root |

### 6.2 Comparison

| Operator | Meaning |
|---|---|
| `= <> (or !=)` | Equal, Not equal |
| `< > <= >=` | Less/greater than (or equal) |
| `BETWEEN x AND y` | Range check (inclusive) |
| `IN (...)` | Match in a list |
| `IS NULL` / `IS NOT NULL` | Null checks (never use `= NULL`) |
| `IS DISTINCT FROM` | Null-safe inequality comparison |

### 6.3 Logical

| Operator | Meaning |
|---|---|
| `AND` | Both true |
| `OR` | Either true |
| `NOT` | Negation |

### 6.4 Pattern Matching

| Operator | Meaning |
|---|---|
| `LIKE` | Pattern match (`%` = any chars, `_` = single char) |
| `ILIKE` | Case-insensitive LIKE (PostgreSQL-specific) |
| `~` | Regex match (case-sensitive) |
| `~*` | Regex match (case-insensitive) |
| `!~ / !~*` | Negated regex match |
| `SIMILAR TO` | SQL standard regex-like matching |

### 6.5 Set / Subquery Operators

| Operator | Meaning |
|---|---|
| `IN` / `NOT IN` | Match within subquery/list |
| `EXISTS` / `NOT EXISTS` | True if subquery returns rows |
| `ANY` / `SOME` | True if any comparison holds |
| `ALL` | True if all comparisons hold |

### 6.6 String Concatenation & JSON Operators

| Operator | Meaning |
|---|---|
| `\|\|` | String concatenation (also array/json concat) |
| `->` | Get JSON object field (returns json) |
| `->>` | Get JSON object field as text |
| `#>` | Get JSON object at path |
| `#>>` | Get JSON object at path as text |
| `@>` / `<@` | Contains / contained by (jsonb, arrays, ranges) |
| `?` | Key exists (jsonb) |

### 6.7 Array Operators

| Operator | Meaning |
|---|---|
| `@>` | Contains |
| `<@` | Is contained by |
| `&&` | Overlap |
| `ANY(array)` | Match any element |

---

## 7. JOINS (All Types with Visual Logic)

| Join Type | Description | Syntax |
|---|---|---|
| **INNER JOIN** | Only matching rows from both tables | `SELECT * FROM A JOIN B ON A.id=B.id` |
| **LEFT JOIN (OUTER)** | All rows from left + matched right (NULL if none) | `LEFT JOIN B ON ...` |
| **RIGHT JOIN (OUTER)** | All rows from right + matched left | `RIGHT JOIN B ON ...` |
| **FULL OUTER JOIN** | All rows from both, NULLs where no match | `FULL JOIN B ON ...` |
| **CROSS JOIN** | Cartesian product (every row × every row) | `CROSS JOIN B` |
| **SELF JOIN** | Table joined to itself | `FROM employees e1 JOIN employees e2 ON e1.mgr_id = e2.emp_id` |
| **NATURAL JOIN** | Auto-joins on same-named columns (risky, avoid in production) | `NATURAL JOIN B` |
| **LATERAL JOIN** | Right side subquery can reference columns from left side (per-row) | `JOIN LATERAL (subquery) ON true` |

```sql
-- INNER JOIN
SELECT e.first_name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- LEFT JOIN to find unmatched (anti-join pattern)
SELECT e.* FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_id IS NULL;

-- FULL OUTER JOIN
SELECT e.first_name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;

-- LATERAL JOIN — top 2 highest paid per dept
SELECT d.dept_name, e2.first_name, e2.salary
FROM departments d
JOIN LATERAL (
    SELECT first_name, salary FROM employees e
    WHERE e.dept_id = d.dept_id
    ORDER BY salary DESC LIMIT 2
) e2 ON true;
```

**Interview tip:** "Anti-join" (find rows in A with no match in B) and "Semi-join" (find rows in A that have at least one match in B, without duplicating) are achieved via `LEFT JOIN ... WHERE B.id IS NULL` and `EXISTS`, not via a dedicated keyword.

---

## 8. Set Operations

| Operator | Description | Duplicate handling |
|---|---|---|
| `UNION` | Combines result sets | Removes duplicates |
| `UNION ALL` | Combines result sets | Keeps duplicates (faster) |
| `INTERSECT` | Common rows in both queries | Removes duplicates |
| `EXCEPT` | Rows in first query not in second (like MINUS in Oracle) | Removes duplicates |

```sql
SELECT name FROM current_employees
UNION
SELECT name FROM contractors;

SELECT emp_id FROM employees
INTERSECT
SELECT emp_id FROM payroll;

SELECT emp_id FROM employees
EXCEPT
SELECT emp_id FROM terminated_employees;
```

**Rules:** Same number of columns, compatible data types, column names taken from first query.

---

## 9. Aggregate Functions & GROUP BY / HAVING

| Function | Description |
|---|---|
| `COUNT()` | Number of rows |
| `SUM()` | Total |
| `AVG()` | Average |
| `MIN() / MAX()` | Minimum/Maximum |
| `STRING_AGG(col, ', ')` | Concatenate strings with delimiter |
| `ARRAY_AGG(col)` | Aggregate into array |
| `JSON_AGG(col)` | Aggregate into JSON array |
| `STDDEV() / VARIANCE()` | Statistical functions |
| `PERCENTILE_CONT() / PERCENTILE_DISC()` | Percentile calculations |

```sql
SELECT dept_id, COUNT(*) AS emp_count, AVG(salary) AS avg_sal
FROM employees
GROUP BY dept_id
HAVING COUNT(*) > 5
ORDER BY avg_sal DESC;
```

**WHERE vs HAVING:** `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation.

---

## 10. Window Functions (Critical for Interviews)

```sql
SELECT 
    emp_id, dept_id, salary,
    ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn,
    RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS drnk,
    LAG(salary, 1) OVER (PARTITION BY dept_id ORDER BY salary) AS prev_salary,
    LEAD(salary, 1) OVER (PARTITION BY dept_id ORDER BY salary) AS next_salary,
    SUM(salary) OVER (PARTITION BY dept_id) AS dept_total,
    AVG(salary) OVER (PARTITION BY dept_id ORDER BY salary 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
FROM employees;
```

| Function | Behavior |
|---|---|
| `ROW_NUMBER()` | Unique sequential number, no ties |
| `RANK()` | Same rank for ties, gaps after ties |
| `DENSE_RANK()` | Same rank for ties, no gaps |
| `NTILE(n)` | Splits rows into n buckets |
| `LAG()/LEAD()` | Access previous/next row's value |
| `FIRST_VALUE()/LAST_VALUE()` | First/last value in window frame |
| `CUME_DIST()` | Cumulative distribution |
| Frame clause | `ROWS/RANGE BETWEEN ... AND ...` defines window boundary |

**Classic interview question:** Find the 2nd highest salary per department → use `DENSE_RANK()` and filter `WHERE rnk = 2` in an outer query (window functions can't be used directly in `WHERE`).

---

## 11. Subqueries & CTEs

```sql
-- Scalar subquery
SELECT first_name FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);

-- Correlated subquery
SELECT e.first_name FROM employees e
WHERE EXISTS (SELECT 1 FROM payroll p WHERE p.emp_id = e.emp_id);

-- CTE (Common Table Expression)
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id
)
SELECT e.first_name, e.salary, d.avg_sal
FROM employees e JOIN dept_avg d ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_sal;

-- Recursive CTE (e.g., org chart / hierarchy traversal)
WITH RECURSIVE org_chart AS (
    SELECT emp_id, first_name, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.first_name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.emp_id
)
SELECT * FROM org_chart ORDER BY level;
```

---

## 12. Indexes (Production-Critical)

| Index Type | Best For | Notes |
|---|---|---|
| `B-tree` (default) | Equality & range queries (`=`, `<`, `>`, `BETWEEN`) | Default for most columns |
| `Hash` | Equality only (`=`) | Rarely better than B-tree now |
| `GIN` | Full-text search, `jsonb`, arrays | Good for `@>`, `?`, full text |
| `GiST` | Geometric data, ranges, full-text | Supports nearest-neighbor |
| `SP-GiST` | Non-balanced data structures (e.g., quad-trees) | Specialized use cases |
| `BRIN` | Very large, naturally ordered tables (e.g., timestamp logs) | Tiny size, fast on huge sequential data |

```sql
CREATE INDEX idx_emp_salary ON employees(salary);
CREATE INDEX idx_emp_name_lower ON employees (LOWER(first_name));   -- functional index
CREATE INDEX idx_emp_partial ON employees(dept_id) WHERE is_active = TRUE; -- partial index
CREATE INDEX idx_emp_multi ON employees(dept_id, salary);            -- composite index
CREATE INDEX idx_emp_jsonb ON employees USING GIN (metadata);        -- GIN for jsonb
```

**Interview tip:** Composite index column order matters — leftmost prefix rule: index `(a,b)` helps queries filtering on `a` or `a AND b`, but not `b` alone.

---

## 13. Transactions (TCL) & Isolation Levels

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;        -- or ROLLBACK;

SAVEPOINT sp1;
-- ... some statements
ROLLBACK TO sp1;
```

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| `READ UNCOMMITTED` (treated as READ COMMITTED in PG) | Prevented | Possible | Possible |
| `READ COMMITTED` (PostgreSQL default) | Prevented | Possible | Possible |
| `REPEATABLE READ` | Prevented | Prevented | Prevented* (PG uses snapshot, prevents phantoms too) |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**ACID:** Atomicity, Consistency, Isolation, Durability — PostgreSQL is fully ACID-compliant using MVCC (Multi-Version Concurrency Control).

---

## 14. DCL — Permissions

```sql
GRANT SELECT, INSERT ON employees TO hr_user;
REVOKE INSERT ON employees FROM hr_user;
GRANT ALL PRIVILEGES ON DATABASE company_db TO admin_role;
ALTER ROLE hr_user WITH PASSWORD 'newpass';
CREATE ROLE analyst LOGIN PASSWORD 'pass123';
```

---

## 15. String, Date & Conversion Functions (Quick Reference)

| Category | Function | Example |
|---|---|---|
| String | `CONCAT`, `\|\|`, `SUBSTRING`, `LENGTH`, `TRIM`, `UPPER/LOWER`, `REPLACE`, `SPLIT_PART` | `SPLIT_PART('a,b,c', ',', 2)` → 'b' |
| Date | `NOW()`, `CURRENT_DATE`, `AGE()`, `DATE_TRUNC()`, `EXTRACT()`, `TO_CHAR()` | `DATE_TRUNC('month', now())` |
| Conversion | `CAST(x AS type)` or `x::type`, `TO_NUMBER()`, `TO_DATE()` | `'123'::int` |
| Null handling | `COALESCE(a,b,c)`, `NULLIF(a,b)` | `COALESCE(phone,'N/A')` |
| Conditional | `CASE WHEN ... THEN ... ELSE ... END` | Standard conditional logic |

```sql
SELECT 
  CASE 
    WHEN salary > 80000 THEN 'High'
    WHEN salary > 40000 THEN 'Mid'
    ELSE 'Low'
  END AS salary_band
FROM employees;
```

---

## 16. EXPLAIN / Performance Tuning

```sql
EXPLAIN ANALYZE SELECT * FROM employees WHERE dept_id = 3;
```

| Term | Meaning |
|---|---|
| `Seq Scan` | Full table scan (no usable index) |
| `Index Scan` | Uses index to find rows |
| `Index Only Scan` | Reads only the index, no table lookup needed |
| `Bitmap Heap Scan` | Combines multiple index results |
| `Nested Loop / Hash Join / Merge Join` | Join algorithms chosen by planner |
| `cost=` | Estimated startup..total cost (planner units, not time) |
| `actual time=` | Real execution time during ANALYZE |

---

## 17. Common Interview-Style Query Patterns

```sql
-- Nth highest salary
SELECT DISTINCT salary FROM employees
ORDER BY salary DESC LIMIT 1 OFFSET (N-1);

-- Duplicate rows detection
SELECT email, COUNT(*) FROM employees GROUP BY email HAVING COUNT(*) > 1;

-- Delete duplicates keeping one
DELETE FROM employees a USING employees b
WHERE a.emp_id > b.emp_id AND a.email = b.email;

-- Running total
SELECT emp_id, salary, SUM(salary) OVER (ORDER BY emp_id) AS running_total
FROM employees;

-- Pivot-like aggregation
SELECT dept_id,
  SUM(CASE WHEN gender='M' THEN 1 ELSE 0 END) AS male_count,
  SUM(CASE WHEN gender='F' THEN 1 ELSE 0 END) AS female_count
FROM employees GROUP BY dept_id;
```

---

## 18. Quick-Glance Cheat Sheet Summary

| Concept | Key Command(s) |
|---|---|
| Schema definition | `CREATE/ALTER/DROP TABLE` |
| Data changes | `INSERT/UPDATE/DELETE/MERGE` |
| Read data | `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY` |
| Combine tables | `JOIN` (INNER/LEFT/RIGHT/FULL/CROSS/LATERAL) |
| Combine queries | `UNION/UNION ALL/INTERSECT/EXCEPT` |
| Ranking/analytics | Window functions (`ROW_NUMBER`, `RANK`, `LAG/LEAD`) |
| Reusable logic | `CTE` / `RECURSIVE CTE` / `VIEW` |
| Performance | Indexes (B-tree/GIN/GiST/BRIN), `EXPLAIN ANALYZE` |
| Safety | Transactions (`BEGIN/COMMIT/ROLLBACK`), constraints |
| Access control | `GRANT/REVOKE` |

---

*This document covers PostgreSQL 13–17 syntax. Always verify version-specific features (e.g., `MERGE` requires PG 15+) against the [official PostgreSQL documentation](https://www.postgresql.org/docs/).*
