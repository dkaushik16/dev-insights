# PostgreSQL — Complete Interview Preparation Guide
### From Basics to Advanced (with Queries, Concepts & Common Interview Questions)

---

## TABLE OF CONTENTS
1. Introduction & Architecture
2. Installation & Basic Setup
3. Data Types
4. DDL — Creating & Modifying Structures
5. DML — Inserting, Updating, Deleting
6. Querying — SELECT Deep Dive
7. Joins
8. Aggregations & Grouping
9. Subqueries & CTEs
10. Window Functions
11. Indexes
12. Constraints & Keys
13. Views & Materialized Views
14. Transactions & Concurrency (ACID, Isolation Levels, MVCC)
15. Functions, Stored Procedures, Triggers
16. JSON/JSONB in PostgreSQL
17. Performance Tuning & EXPLAIN
18. Partitioning
19. Backup & Replication (conceptual)
20. Security
21. Common Interview Questions (Theory)
22. Common Interview SQL Query Problems (with Answers)

---

## 1. INTRODUCTION & ARCHITECTURE

**What is PostgreSQL?**
An open-source, object-relational database management system (ORDBMS) known for standards compliance, extensibility, and robustness (ACID-compliant).

**Key architectural concepts:**
- **Process-based architecture**: Each client connection gets a dedicated backend process (unlike MySQL's thread-based model).
- **Postmaster**: The main supervisor process that spawns backend processes.
- **Shared Buffers**: In-memory cache for data pages.
- **WAL (Write-Ahead Log)**: Ensures durability — changes are logged before being written to disk.
- **MVCC (Multi-Version Concurrency Control)**: Allows readers and writers to not block each other.

**Interview tip:** Be ready to explain MVCC and WAL — these are PostgreSQL's signature differentiators.

---

## 2. INSTALLATION & BASIC SETUP

```bash
# Connect to PostgreSQL
psql -U postgres -h localhost -d mydatabase

# Common psql meta-commands
\l        -- list databases
\c dbname -- connect to a database
\dt       -- list tables
\d table  -- describe table structure
\du       -- list users/roles
\q        -- quit
```

---

## 3. DATA TYPES

| Category | Types |
|---|---|
| Numeric | `SMALLINT`, `INTEGER`, `BIGINT`, `DECIMAL`, `NUMERIC`, `REAL`, `DOUBLE PRECISION`, `SERIAL`, `BIGSERIAL` |
| Character | `CHAR(n)`, `VARCHAR(n)`, `TEXT` |
| Date/Time | `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL` |
| Boolean | `BOOLEAN` |
| Binary | `BYTEA` |
| JSON | `JSON`, `JSONB` |
| Arrays | `INTEGER[]`, `TEXT[]` |
| UUID | `UUID` |
| Special | `ENUM`, `RANGE` types (`int4range`, `tsrange`) |

**Interview Q: JSON vs JSONB?**
`JSON` stores an exact text copy (preserves formatting/whitespace, slower to process). `JSONB` stores in decomposed binary format (faster querying, supports indexing, no whitespace preserved). **Always prefer JSONB** unless you need to preserve original text.

**Interview Q: VARCHAR vs TEXT?**
No real performance difference in PostgreSQL. `VARCHAR(n)` enforces a length limit; `TEXT` has none. Internally both use the same storage mechanism.

---

## 4. DDL — DATA DEFINITION LANGUAGE

```sql
-- Create database
CREATE DATABASE company_db;

-- Create table
CREATE TABLE employees (
    emp_id      SERIAL PRIMARY KEY,
    first_name  VARCHAR(50) NOT NULL,
    last_name   VARCHAR(50) NOT NULL,
    email       VARCHAR(100) UNIQUE,
    salary      NUMERIC(10,2) CHECK (salary > 0),
    dept_id     INTEGER REFERENCES departments(dept_id),
    hire_date   DATE DEFAULT CURRENT_DATE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Alter table
ALTER TABLE employees ADD COLUMN phone VARCHAR(15);
ALTER TABLE employees ALTER COLUMN salary SET DEFAULT 30000;
ALTER TABLE employees DROP COLUMN phone;
ALTER TABLE employees RENAME COLUMN first_name TO fname;
ALTER TABLE employees RENAME TO staff;

-- Drop
DROP TABLE employees;
TRUNCATE TABLE employees;  -- removes all rows, faster than DELETE, resets identity
```

**Interview Q: DROP vs TRUNCATE vs DELETE?**
- `DELETE`: Removes rows one at a time, logged, can use WHERE, can be rolled back, triggers fire, slower on large tables.
- `TRUNCATE`: Removes all rows instantly, minimal logging, resets `SERIAL`/identity, cannot use WHERE, triggers normally don't fire (unless `TRUNCATE` triggers defined).
- `DROP`: Removes the entire table structure + data.

---

## 5. DML — DATA MANIPULATION LANGUAGE

```sql
INSERT INTO employees (first_name, last_name, salary, dept_id)
VALUES ('Anil', 'Sharma', 55000, 2);

-- Insert multiple rows
INSERT INTO employees (first_name, last_name, salary, dept_id) VALUES
('Riya', 'Verma', 62000, 1),
('Kabir', 'Singh', 48000, 3);

-- Insert with returning
INSERT INTO employees (first_name, last_name, salary, dept_id)
VALUES ('Meera', 'Joshi', 70000, 2)
RETURNING emp_id;

UPDATE employees SET salary = salary * 1.10 WHERE dept_id = 2;

DELETE FROM employees WHERE emp_id = 5;

-- UPSERT (Insert or Update on conflict)
INSERT INTO employees (emp_id, first_name, salary)
VALUES (1, 'Anil', 60000)
ON CONFLICT (emp_id)
DO UPDATE SET salary = EXCLUDED.salary;
```

**Interview Q: Explain `ON CONFLICT` / UPSERT.**
PostgreSQL's `INSERT ... ON CONFLICT` lets you handle unique constraint violations gracefully — either `DO NOTHING` or `DO UPDATE`, avoiding race conditions in check-then-insert logic.

---

## 6. QUERYING — SELECT DEEP DIVE

```sql
SELECT first_name, salary
FROM employees
WHERE salary > 50000
  AND dept_id IN (1, 2)
ORDER BY salary DESC
LIMIT 5 OFFSET 10;

-- Pattern matching
SELECT * FROM employees WHERE first_name LIKE 'A%';   -- case-sensitive
SELECT * FROM employees WHERE first_name ILIKE 'a%';  -- case-insensitive

-- DISTINCT
SELECT DISTINCT dept_id FROM employees;

-- NULL handling
SELECT * FROM employees WHERE phone IS NULL;
SELECT COALESCE(phone, 'N/A') FROM employees;

-- CASE expression
SELECT first_name,
  CASE
    WHEN salary > 60000 THEN 'High'
    WHEN salary > 40000 THEN 'Medium'
    ELSE 'Low'
  END AS salary_band
FROM employees;
```

**Interview Q: Execution order of a SQL query (logical, not written order)?**
```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET
```
This is a **very common** interview question.

---

## 7. JOINS

```sql
-- INNER JOIN
SELECT e.first_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;

-- LEFT JOIN (all from left, matched from right)
SELECT e.first_name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;

-- RIGHT JOIN
SELECT e.first_name, d.dept_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;

-- FULL OUTER JOIN
SELECT e.first_name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;

-- SELF JOIN (e.g., employees and their managers)
SELECT e.first_name AS employee, m.first_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;

-- CROSS JOIN (Cartesian product)
SELECT * FROM employees CROSS JOIN departments;

-- Find employees with NO matching department (anti-join pattern)
SELECT e.* FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_id IS NULL;
```

**Interview Q: Difference between WHERE and ON in joins?**
`ON` filters rows during the join process (before the join result is materialized) — important for outer joins because it determines which rows get NULL-padded. `WHERE` filters AFTER the join is complete, which can eliminate outer-joined NULL rows unintentionally.

---

## 8. AGGREGATIONS & GROUPING

```sql
SELECT dept_id, COUNT(*), AVG(salary), MAX(salary), MIN(salary), SUM(salary)
FROM employees
GROUP BY dept_id
HAVING COUNT(*) > 3
ORDER BY AVG(salary) DESC;

-- GROUPING SETS / ROLLUP / CUBE
SELECT dept_id, manager_id, SUM(salary)
FROM employees
GROUP BY ROLLUP(dept_id, manager_id);
```

**Interview Q: WHERE vs HAVING?**
`WHERE` filters rows before grouping/aggregation; `HAVING` filters groups after aggregation. You cannot use aggregate functions in `WHERE`.

---

## 9. SUBQUERIES & CTEs

```sql
-- Scalar subquery
SELECT first_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Correlated subquery
SELECT first_name, salary
FROM employees e
WHERE salary > (
  SELECT AVG(salary) FROM employees e2 WHERE e2.dept_id = e.dept_id
);

-- EXISTS
SELECT * FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id);

-- CTE (Common Table Expression)
WITH dept_avg AS (
  SELECT dept_id, AVG(salary) AS avg_sal
  FROM employees
  GROUP BY dept_id
)
SELECT e.first_name, e.salary, d.avg_sal
FROM employees e
JOIN dept_avg d ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_sal;

-- Recursive CTE (e.g., org hierarchy)
WITH RECURSIVE org_chain AS (
  SELECT emp_id, first_name, manager_id, 1 AS level
  FROM employees
  WHERE manager_id IS NULL
  UNION ALL
  SELECT e.emp_id, e.first_name, e.manager_id, oc.level + 1
  FROM employees e
  JOIN org_chain oc ON e.manager_id = oc.emp_id
)
SELECT * FROM org_chain ORDER BY level;
```

**Interview Q: When are CTEs materialized?**
Pre-PG12: CTEs were always materialized (optimization fence). PG12+: the planner can inline non-recursive CTEs unless marked `AS MATERIALIZED`. Recursive CTEs are always materialized.

---

## 10. WINDOW FUNCTIONS (Very common in interviews)

```sql
-- Rank employees by salary within department
SELECT first_name, dept_id, salary,
  RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dense_rnk,
  ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS row_num
FROM employees;

-- Running total
SELECT first_name, salary,
  SUM(salary) OVER (ORDER BY emp_id) AS running_total
FROM employees;

-- LAG / LEAD (compare with previous/next row)
SELECT first_name, salary,
  LAG(salary) OVER (ORDER BY emp_id) AS prev_salary,
  LEAD(salary) OVER (ORDER BY emp_id) AS next_salary
FROM employees;

-- NTILE (bucket into n groups)
SELECT first_name, salary, NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;

-- Nth highest salary (classic interview question)
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;  -- 2nd highest

-- Better way using DENSE_RANK
SELECT salary FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM employees
) t WHERE rnk = 2;
```

**Interview Q: RANK vs DENSE_RANK vs ROW_NUMBER?**
- `ROW_NUMBER()`: Unique sequential number, no ties.
- `RANK()`: Same rank for ties, but skips numbers after a tie (1,1,3).
- `DENSE_RANK()`: Same rank for ties, no gaps (1,1,2).

---

## 11. INDEXES

```sql
CREATE INDEX idx_emp_salary ON employees(salary);
CREATE UNIQUE INDEX idx_emp_email ON employees(email);
CREATE INDEX idx_emp_multi ON employees(dept_id, salary);  -- composite
CREATE INDEX idx_emp_lower_email ON employees(LOWER(email));  -- expression index
CREATE INDEX idx_emp_partial ON employees(salary) WHERE salary > 50000;  -- partial

-- Index types
-- B-Tree (default) – equality & range queries
-- Hash – equality only
-- GIN – full text search, JSONB, arrays
-- GiST – geometric data, full text search
-- BRIN – very large tables with naturally ordered data (e.g., timestamps)
```

**Interview Q: How do indexes work internally?**
B-Tree indexes store a balanced tree structure pointing to row locations (TIDs - tuple IDs), enabling O(log n) lookups instead of full table scans.

**Interview Q: When NOT to use an index?**
- Small tables (sequential scan is faster).
- Columns with low cardinality (few distinct values).
- Tables with heavy write load (indexes slow down INSERT/UPDATE/DELETE).

**Interview Q: Difference between Clustered and Non-Clustered Index?**
PostgreSQL doesn't have true clustered indexes like SQL Server (data isn't physically reordered automatically). `CLUSTER` command can physically reorder a table based on an index once, but it isn't maintained automatically afterward.

---

## 12. CONSTRAINTS & KEYS

```sql
CREATE TABLE departments (
  dept_id    SERIAL PRIMARY KEY,
  dept_name  VARCHAR(50) NOT NULL UNIQUE,
  budget     NUMERIC CHECK (budget >= 0)
);

ALTER TABLE employees
  ADD CONSTRAINT fk_dept FOREIGN KEY (dept_id)
  REFERENCES departments(dept_id) ON DELETE CASCADE ON UPDATE CASCADE;
```

**Interview Q: ON DELETE CASCADE vs SET NULL vs RESTRICT vs NO ACTION?**
- `CASCADE`: deletes child rows too.
- `SET NULL`: sets FK column to NULL.
- `RESTRICT`/`NO ACTION`: prevents deletion if referenced (default behavior, slight timing difference within transactions).

**Interview Q: Primary Key vs Unique Key?**
PK = NOT NULL + UNIQUE, only one per table. Unique key allows NULLs (multiple), and a table can have multiple unique constraints.

---

## 13. VIEWS & MATERIALIZED VIEWS

```sql
CREATE VIEW high_earners AS
SELECT first_name, salary, dept_id FROM employees WHERE salary > 60000;

CREATE MATERIALIZED VIEW dept_summary AS
SELECT dept_id, COUNT(*), AVG(salary) FROM employees GROUP BY dept_id;

REFRESH MATERIALIZED VIEW dept_summary;
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_summary; -- needs unique index, doesn't lock reads
```

**Interview Q: View vs Materialized View?**
A View is a stored query executed live each time (no storage of results). A Materialized View stores the actual result physically, must be refreshed manually (or scheduled), and is faster for expensive aggregations but can serve stale data.

---

## 14. TRANSACTIONS, ACID & CONCURRENCY (Heavy interview area)

```sql
BEGIN;
UPDATE employees SET salary = salary - 5000 WHERE emp_id = 1;
UPDATE employees SET salary = salary + 5000 WHERE emp_id = 2;
COMMIT;
-- or ROLLBACK;

SAVEPOINT sp1;
UPDATE employees SET salary = 0 WHERE emp_id = 3;
ROLLBACK TO sp1;
COMMIT;
```

**ACID:**
- **Atomicity**: All or nothing.
- **Consistency**: DB moves from one valid state to another.
- **Isolation**: Concurrent transactions don't interfere.
- **Durability**: Committed data survives crashes (via WAL).

**Isolation Levels (PostgreSQL supports):**
| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted (treated as Read Committed) | No | Yes | Yes |
| Read Committed (default) | No | Yes | Yes |
| Repeatable Read | No | No | No* |
| Serializable | No | No | No |

*PostgreSQL's Repeatable Read actually prevents phantom reads too (stricter than SQL standard requires), using snapshot isolation.

**Interview Q: What is MVCC?**
Multi-Version Concurrency Control — instead of locking rows for reads, PostgreSQL keeps multiple versions of a row (using hidden `xmin`/`xmax` transaction ID columns). Readers see a consistent snapshot without blocking writers, and vice versa.

**Interview Q: What is a deadlock and how does PostgreSQL handle it?**
Two transactions waiting on each other's locks. PostgreSQL automatically detects deadlocks and aborts one transaction (with an error) to break the cycle.

**Interview Q: VACUUM — what and why?**
Because of MVCC, updated/deleted rows leave "dead tuples" behind. `VACUUM` reclaims this space. `VACUUM FULL` rewrites the table to reclaim space physically (locks the table). `AUTOVACUUM` runs this automatically in the background.

---

## 15. FUNCTIONS, STORED PROCEDURES, TRIGGERS

```sql
-- Function
CREATE OR REPLACE FUNCTION get_total_salary(p_dept_id INT)
RETURNS NUMERIC AS $$
DECLARE
  total NUMERIC;
BEGIN
  SELECT SUM(salary) INTO total FROM employees WHERE dept_id = p_dept_id;
  RETURN total;
END;
$$ LANGUAGE plpgsql;

SELECT get_total_salary(2);

-- Procedure (can manage transactions internally, unlike functions)
CREATE OR REPLACE PROCEDURE give_raise(p_emp_id INT, p_amount NUMERIC)
LANGUAGE plpgsql AS $$
BEGIN
  UPDATE employees SET salary = salary + p_amount WHERE emp_id = p_emp_id;
  COMMIT;
END;
$$;

CALL give_raise(1, 5000);

-- Trigger
CREATE OR REPLACE FUNCTION log_salary_change()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO salary_log(emp_id, old_salary, new_salary, changed_at)
  VALUES (OLD.emp_id, OLD.salary, NEW.salary, NOW());
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_salary_update
AFTER UPDATE OF salary ON employees
FOR EACH ROW
WHEN (OLD.salary IS DISTINCT FROM NEW.salary)
EXECUTE FUNCTION log_salary_change();
```

**Interview Q: Function vs Procedure in PostgreSQL?**
Functions must return a value, can be used in `SELECT`, cannot independently commit/rollback. Procedures (added in PG11) are called with `CALL`, can contain transaction control statements (`COMMIT`/`ROLLBACK`), and don't have to return anything.

---

## 16. JSON / JSONB

```sql
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  details JSONB
);

INSERT INTO orders (details) VALUES
('{"customer": "Anil", "items": [{"name": "pen", "qty": 2}], "total": 100}');

-- Querying JSON
SELECT details->>'customer' AS customer FROM orders;
SELECT details->'items'->0->>'name' AS first_item FROM orders;
SELECT * FROM orders WHERE details @> '{"customer": "Anil"}';
SELECT * FROM orders WHERE details ? 'customer';  -- key exists

CREATE INDEX idx_orders_details ON orders USING GIN(details);
```

**Interview Q: -> vs ->> operator?**
`->` returns JSON/JSONB type, `->>` returns text type.

---

## 17. PERFORMANCE TUNING & EXPLAIN

```sql
EXPLAIN SELECT * FROM employees WHERE salary > 50000;
EXPLAIN ANALYZE SELECT * FROM employees WHERE salary > 50000;
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
```

**Key things to look for in EXPLAIN output:**
- **Seq Scan** vs **Index Scan** vs **Bitmap Index Scan**
- **Cost** (startup cost..total cost)
- **Rows** estimated vs actual (from ANALYZE)
- **Nested Loop** vs **Hash Join** vs **Merge Join**

**Interview Q: How would you optimize a slow query?**
1. Run `EXPLAIN ANALYZE` to find the bottleneck.
2. Check if indexes exist on filter/join columns.
3. Avoid `SELECT *`; select only needed columns.
4. Avoid functions on indexed columns in WHERE (`WHERE LOWER(email)='x'` breaks index unless expression index exists).
5. Check `work_mem`, `shared_buffers` configuration.
6. Consider partitioning very large tables.
7. Update statistics with `ANALYZE`; ensure autovacuum is healthy.

**Interview Q: Nested Loop vs Hash Join vs Merge Join?**
- **Nested Loop**: Good for small datasets / indexed lookups.
- **Hash Join**: Good for large unsorted datasets, builds hash table on smaller side.
- **Merge Join**: Good when both inputs are already sorted on join key.

---

## 18. PARTITIONING

```sql
CREATE TABLE sales (
  id SERIAL,
  sale_date DATE NOT NULL,
  amount NUMERIC
) PARTITION BY RANGE (sale_date);

CREATE TABLE sales_2024 PARTITION OF sales
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE sales_2025 PARTITION OF sales
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

**Interview Q: Why partition tables?**
Improves query performance via "partition pruning" (only scans relevant partitions), eases maintenance (drop old partitions instantly instead of DELETE), and improves vacuum/index efficiency.

**Types:** Range, List, Hash partitioning.

---

## 19. BACKUP & REPLICATION (Conceptual)

- `pg_dump` / `pg_restore`: logical backup of a database.
- `pg_basebackup`: physical backup of the entire cluster.
- **Streaming Replication**: WAL records shipped to standby servers in near real-time (primary-replica setup).
- **Logical Replication**: Replicates specific tables using publish/subscribe model, allows replication across major versions.

**Interview Q: Streaming vs Logical Replication?**
Streaming replication copies the entire cluster byte-for-byte via WAL (used for HA/failover). Logical replication replicates at the table/row level, allowing selective replication and cross-version compatibility.

---

## 20. SECURITY

```sql
CREATE ROLE app_user WITH LOGIN PASSWORD 'secret';
GRANT SELECT, INSERT ON employees TO app_user;
REVOKE INSERT ON employees FROM app_user;

-- Row Level Security
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;
CREATE POLICY emp_policy ON employees
  USING (dept_id = current_setting('app.current_dept')::INT);
```

**Interview Q: What is Row-Level Security (RLS)?**
Allows restricting which rows a particular role/user can see or modify, enforced at the database level regardless of the query, useful for multi-tenant applications.

---

## 21. TOP THEORY INTERVIEW QUESTIONS (Quick-Fire)

1. **What makes PostgreSQL different from MySQL?** → Stronger standards compliance, advanced data types (JSONB, arrays, ranges), better support for complex queries/window functions, MVCC implementation differs (PostgreSQL stores old row versions in the table itself, not in a separate undo log like InnoDB).
2. **What is a sequence?** → An auto-incrementing number generator object, used internally by `SERIAL`.
3. **What's the difference between `SERIAL` and `IDENTITY` columns?** → `SERIAL` is older syntax creating a sequence + default; `GENERATED ALWAYS AS IDENTITY` (SQL standard, PG10+) is the modern recommended approach, prevents manual override issues.
4. **What is a tablespace?** → A location on disk where PostgreSQL stores data files, useful for spreading I/O across disks.
5. **What is connection pooling and why needed?** → Since PostgreSQL is process-per-connection, too many connections is expensive; tools like **PgBouncer** or **Pgpool-II** reuse connections.
6. **Difference between `CHAR`, `VARCHAR`, `TEXT`?** → Covered above.
7. **What's the difference between `UNION` and `UNION ALL`?** → `UNION` removes duplicates (slower, implicit sort/distinct); `UNION ALL` keeps duplicates (faster).
8. **What's a Common Table Expression (CTE) vs subquery?** → CTEs improve readability, support recursion, and (pre-PG12) acted as optimization fences.
9. **What is the difference between a Heap table and Index-Organized table?** → PostgreSQL tables are heap-organized by default (no implicit physical ordering); some other RDBMS support clustered/index-organized tables natively.
10. **Explain Foreign Data Wrappers (FDW).** → Extension mechanism (`postgres_fdw`, `file_fdw`) to query external data sources as if they were local tables.

---

## 22. COMMON SQL QUERY INTERVIEW PROBLEMS (Practice Set)

**Schema assumed:**
```sql
employees(emp_id, first_name, last_name, salary, dept_id, manager_id, hire_date)
departments(dept_id, dept_name)
```

**Q1. Find the 2nd highest salary in each department.**
```sql
SELECT dept_id, salary FROM (
  SELECT dept_id, salary,
         DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) rnk
  FROM employees
) t WHERE rnk = 2;
```

**Q2. Find duplicate emails in a table.**
```sql
SELECT email, COUNT(*) FROM employees GROUP BY email HAVING COUNT(*) > 1;
```

**Q3. Delete duplicate rows keeping only one.**
```sql
DELETE FROM employees a
USING employees b
WHERE a.emp_id > b.emp_id AND a.email = b.email;
```

**Q4. Find employees who earn more than their manager.**
```sql
SELECT e.first_name, e.salary, m.first_name AS manager
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

**Q5. Find departments with no employees.**
```sql
SELECT d.dept_name FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
WHERE e.emp_id IS NULL;
```

**Q6. Find the running total of salaries ordered by hire date.**
```sql
SELECT first_name, hire_date, salary,
  SUM(salary) OVER (ORDER BY hire_date) AS running_total
FROM employees;
```

**Q7. Find consecutive days/streaks (e.g., login streak problems) — classic "gaps and islands."**
```sql
WITH numbered AS (
  SELECT user_id, login_date,
    login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::INT AS grp
  FROM logins
)
SELECT user_id, MIN(login_date) AS streak_start, MAX(login_date) AS streak_end, COUNT(*) AS streak_len
FROM numbered
GROUP BY user_id, grp
ORDER BY streak_len DESC;
```

**Q8. Pivot data (rows to columns) without crosstab extension.**
```sql
SELECT dept_id,
  SUM(CASE WHEN gender='M' THEN 1 ELSE 0 END) AS male_count,
  SUM(CASE WHEN gender='F' THEN 1 ELSE 0 END) AS female_count
FROM employees
GROUP BY dept_id;
```

**Q9. Find the department with the highest average salary.**
```sql
SELECT dept_id, AVG(salary) avg_sal
FROM employees
GROUP BY dept_id
ORDER BY avg_sal DESC
LIMIT 1;
```

**Q10. Find employees hired in the last 30 days.**
```sql
SELECT * FROM employees WHERE hire_date >= CURRENT_DATE - INTERVAL '30 days';
```

**Q11. Find the cumulative count of employees joined per month.**
```sql
SELECT TO_CHAR(hire_date, 'YYYY-MM') AS month,
  COUNT(*) AS monthly_hires,
  SUM(COUNT(*)) OVER (ORDER BY TO_CHAR(hire_date, 'YYYY-MM')) AS cumulative_hires
FROM employees
GROUP BY month
ORDER BY month;
```

**Q12. Swap values of two columns without a temp variable.**
```sql
UPDATE employees SET salary = bonus, bonus = salary;  -- works because RHS evaluated first using old row values
```

**Q13. Find the Nth row without LIMIT/OFFSET (using ROW_NUMBER).**
```sql
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (ORDER BY emp_id) rn FROM employees
) t WHERE rn = 5;
```

**Q14. Find employees who share the same manager and same salary (potential anomaly check).**
```sql
SELECT manager_id, salary, COUNT(*)
FROM employees
GROUP BY manager_id, salary
HAVING COUNT(*) > 1;
```

---

## QUICK CHEAT-SHEET: COMMON GOTCHAS TO MENTION IN INTERVIEWS

- PostgreSQL is **case-sensitive** for identifiers wrapped in double quotes; unquoted identifiers are folded to lowercase.
- `NULL` is never equal to anything, even another `NULL` — use `IS NULL`/`IS DISTINCT FROM`.
- String concatenation uses `||`, not `+`.
- `LIMIT`/`OFFSET` without `ORDER BY` gives non-deterministic results.
- Auto-increment uses **sequences**, not a special column type — gaps can occur after rollbacks (this is normal/expected).
- `COUNT(*)` vs `COUNT(column)`: the latter ignores NULLs.
- Always specify isolation level concerns and lock behavior when discussing concurrency — interviewers love probing MVCC and VACUUM knowledge since it's PostgreSQL-specific.

---

### How to use this guide
- Go topic by topic, actually **run every query** in a local/psql sandbox or an online editor.
- For the "Common SQL Query Problems" section, try solving each blind before peeking at the answer.
- Revise sections 14 (Transactions/MVCC) and 17 (Performance) most — these differentiate strong PostgreSQL candidates from generic SQL candidates.
