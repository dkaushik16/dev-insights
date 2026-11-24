# PostgreSQL & SQL --- Detailed Quick Reference (Topics 1--7)

This document consolidates the concepts, practical examples, and
important gotchas covered in the first seven parts of the
relational-database preparation roadmap. Examples use a generic clinic
and booking domain. Names are illustrative.

------------------------------------------------------------------------

# 1. Relational database fundamentals

## 1.1 The basic structure

A relational database organizes data into tables.

-   **Database:** A container for related data and database objects.
-   **Schema:** A namespace that organizes objects such as tables,
    views, and functions.
-   **Table:** Represents an entity or relationship, such as `patients`,
    `doctors`, or `appointments`.
-   **Row:** One record in a table.
-   **Column:** One attribute of each record.
-   **Data type:** Defines what kind of value a column can store.

Example:

    id name    phone              age
  ---- ------- ---------------- -----
     1 Rahul   +91-9876543210      29
     2 Priya   09876543210         34

Each row represents a patient. Each column represents one property of
that patient.

## 1.2 Choosing data types by meaning

Choose a type based on the meaning and valid representation of the
value---not merely whether it looks numeric.

  -----------------------------------------------------------------------
  Data                    Suitable type           Reason
  ----------------------- ----------------------- -----------------------
  Person's name           `VARCHAR` or `TEXT`     Text

  Phone number            `VARCHAR` or `TEXT`     May contain `+`,
                                                  leading zeroes, spaces,
                                                  and formatting

  Age/count               `INTEGER`               Whole-number quantity

  Money/fee               `NUMERIC(10,2)`         Exact decimal
                                                  arithmetic

  True/false flag         `BOOLEAN`               Boolean state

  Calendar date           `DATE`                  Date without time of
                                                  day

  Event timestamp         `TIMESTAMP` /           Date and time; timezone
                          `TIMESTAMPTZ`           semantics matter
  -----------------------------------------------------------------------

### Important gotchas

**Phone numbers should not be stored as integers.** A phone number is an
identifier, not a quantity used in arithmetic. Converting it to an
integer can lose leading zeroes and cannot represent `+` or formatting
characters.

**Use `NUMERIC` for exact monetary values.** Floating-point types such
as `REAL` and `DOUBLE PRECISION` use binary floating-point
representation and can introduce rounding surprises for decimal money
values.

**`NULL` is not the same as an empty string.** - `NULL` means no value /
unknown / absent. - `''` is a present text value containing zero
characters. - A comparison such as `email = NULL` is not the correct way
to test for null; use `email IS NULL`.

------------------------------------------------------------------------

# 2. SQL essentials

## 2.1 Reading data with SELECT

``` sql
SELECT name, age
FROM patients
WHERE age > 25
  AND city = 'Meerut'
ORDER BY age DESC
LIMIT 2;
```

Logical reading: 1. Read from `patients`. 2. Keep rows satisfying the
`WHERE` condition. 3. Sort the remaining rows by age, highest first. 4.
Return at most two rows.

Common clauses:

  Clause       Purpose
  ------------ ---------------------------------------
  `SELECT`     Chooses output columns or expressions
  `FROM`       Chooses the source table(s)
  `WHERE`      Filters individual rows
  `ORDER BY`   Sorts the result
  `LIMIT`      Restricts the number of returned rows
  `OFFSET`     Skips rows before returning results

## 2.2 Sorting and pagination

``` sql
SELECT id, patient_id, appointment_date
FROM appointments
ORDER BY appointment_date DESC, id DESC
LIMIT 20 OFFSET 40;
```

-   `LIMIT 20` returns up to 20 rows.
-   `OFFSET 40` skips the first 40 rows of the ordered result.
-   Together, this returns the third page when page size is 20.

### Gotchas

**Always use a deterministic order for pagination.** If several rows
have the same `appointment_date`, the database may return them in an
arbitrary relative order unless a tie-breaker is added. A unique column
such as `id` is a useful tie-breaker.

**OFFSET pagination can become expensive at large offsets.** The
database may still need to walk past many rows before returning the
requested page. Keyset pagination is an alternative for large,
frequently paginated datasets; it is a separate technique to learn
later.

## 2.3 INSERT

``` sql
INSERT INTO patients (name, phone, age)
VALUES ('Rahul', '+91-9876543210', 29);
```

Name the target columns explicitly. This makes the statement easier to
maintain and avoids depending on the table's physical column order.

## 2.4 UPDATE

``` sql
UPDATE appointments
SET status = 'completed'
WHERE id = 101;
```

This changes the status only for the appointment whose `id` is 101.

**Critical gotcha:** An `UPDATE` without a `WHERE` clause updates every
row:

``` sql
-- Dangerous: changes every appointment.
UPDATE appointments
SET status = 'completed';
```

Before running a consequential update, it is good practice to run a
matching `SELECT` with the same condition to inspect the affected rows.

## 2.5 DELETE

``` sql
DELETE FROM appointments
WHERE id = 101;
```

**Critical gotcha:** A `DELETE` without a `WHERE` clause deletes every
row from the table:

``` sql
-- Dangerous: deletes every appointment row.
DELETE FROM appointments;
```

## 2.6 DELETE vs TRUNCATE

  -----------------------------------------------------------------------
  `DELETE`                            `TRUNCATE`
  ----------------------------------- -----------------------------------
  Removes rows; can target a subset   Empties the table
  with `WHERE`                        

  Row-level delete operation          Table-level operation

  Does not normally reset             Can use `RESTART IDENTITY`
  identity/sequence values            

  Can invoke row-level delete         Has its own truncate-trigger
  triggers                            behavior
  -----------------------------------------------------------------------

PostgreSQL-specific points: - `TRUNCATE` is transactional in PostgreSQL
and can be rolled back if executed inside a transaction that is rolled
back. - `TRUNCATE ... RESTART IDENTITY` resets sequences owned by
identity/serial columns. - A foreign-key dependency can prevent
truncating a referenced table. `TRUNCATE ... CASCADE` can include
dependent tables, so use it carefully. - This is not the same as
`ON DELETE CASCADE`, which is a row-level foreign-key delete behavior.

Example:

``` sql
TRUNCATE TABLE appointments RESTART IDENTITY;
```

This empties `appointments` and restarts its owned identity sequences.

------------------------------------------------------------------------

# 3. Constraints

Constraints make the database enforce data rules. Application validation
is useful, but database constraints protect the data even when writes
come from another application path, script, or concurrent request.

## 3.1 Primary key

A primary key uniquely identifies each row.

``` sql
CREATE TABLE patients (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);
```

A primary key is unique and cannot be `NULL`.

## 3.2 Foreign key

A foreign key requires a referencing value to match a referenced key,
subject to nullability.

``` sql
CREATE TABLE appointments (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL REFERENCES patients(id)
);
```

This prevents an appointment from referring to a patient ID that does
not exist.

A foreign key does **not** automatically mean that deleting a patient
will delete appointments. The delete behavior must be chosen.

## 3.3 UNIQUE

``` sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);
```

This prevents duplicate non-null email values.

**PostgreSQL gotcha:** A regular `UNIQUE` constraint allows multiple
`NULL` values by default, because nulls are treated as distinct for this
purpose. If an email is required, combine `UNIQUE` with `NOT NULL`.

## 3.4 NOT NULL

``` sql
name VARCHAR(100) NOT NULL
```

This requires a value to be provided. `NOT NULL` and `CHECK` solve
different problems: `NOT NULL` requires presence, while `CHECK`
validates a condition.

## 3.5 CHECK

``` sql
age INTEGER CHECK (age > 0)
```

`age > 0` disallows zero; `age >= 0` allows zero. The exact condition
should reflect the business rule.

For an allowed set of status values:

``` sql
status VARCHAR(20) NOT NULL DEFAULT 'available'
    CHECK (status IN ('available', 'rented', 'maintenance'))
```

For a positive rental fee:

``` sql
daily_rent NUMERIC(10,2) NOT NULL
    CHECK (daily_rent > 0)
```

### Important gotcha: CHECK is not concurrency control

A check such as `CHECK (status IN (...))` validates that a status is one
of the allowed values. It does not prevent two concurrent booking
requests from both observing a vehicle as available and attempting to
reserve it. Preventing conflicting concurrent operations requires
appropriate transaction, locking, or constraint design.

## 3.6 DEFAULT

``` sql
notifications_enabled BOOLEAN NOT NULL DEFAULT TRUE
```

The default is used when an insert omits the column or explicitly
requests `DEFAULT`. It does not override a value explicitly supplied by
the insert.

## 3.7 Foreign-key delete actions

Example:

``` sql
CREATE TABLE appointments (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL
        REFERENCES patients(id)
        ON DELETE CASCADE
);
```

Deleting a patient deletes appointments that reference that patient.

Direction matters: - Deleting the **parent patient** can cascade to
child appointments. - Deleting an **appointment** does not delete the
patient.

Choose among restrictive/default behavior, `CASCADE`, `SET NULL`, and
other supported actions based on the actual lifecycle rules. Do not use
cascade casually when child records must be retained for audit or
history.

------------------------------------------------------------------------

# 4. Relationships and cardinality

Cardinality describes how records in one table relate to records in
another.

## 4.1 One-to-one (1:1)

Example: one user has at most one settings record.

``` sql
CREATE TABLE account_settings (
    user_id BIGINT PRIMARY KEY REFERENCES users(id),
    theme VARCHAR(20) DEFAULT 'light',
    notifications_enabled BOOLEAN DEFAULT TRUE
);
```

Because `user_id` is both the primary key and foreign key: - Each
settings row belongs to an existing user. - A user can have at most one
settings row. - It does **not** force every user to have a settings row.

This last point is important: a foreign key in the settings table
guarantees that each settings row has a user; it does not guarantee that
every user has a settings row.

## 4.2 One-to-many (1:N)

Example: one patient can have many appointments; each appointment
belongs to one patient.

``` sql
CREATE TABLE appointments (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL REFERENCES patients(id),
    appointment_date TIMESTAMPTZ NOT NULL
);
```

The foreign key is placed on the **many** side. Each appointment stores
the `patient_id` it belongs to.

A doctor-to-branch relationship can be modeled similarly if each doctor
belongs to one current branch:

``` sql
CREATE TABLE doctors (
    id BIGSERIAL PRIMARY KEY,
    branch_id BIGINT NOT NULL REFERENCES branches(id)
);
```

If a doctor transfers, updating `doctors.branch_id` changes the current
branch. If historical appointments must preserve the branch where they
occurred, store that historical branch reference on the appointment as
well. Current state and historical fact are different requirements.

## 4.3 Many-to-many (M:N)

Example: a patient can visit multiple clinics, and a clinic can have
many patients.

Relational databases represent this with a junction/association table:

``` sql
CREATE TABLE patient_clinics (
    patient_id BIGINT REFERENCES patients(id),
    clinic_id BIGINT REFERENCES clinics(id),
    PRIMARY KEY (patient_id, clinic_id)
);
```

The composite primary key prevents duplicate copies of the same
patient-clinic relationship.

## 4.4 Self-referencing many-to-many

Example: users subscribe to other users.

``` sql
CREATE TABLE subscriptions (
    subscriber_id BIGINT REFERENCES users(id),
    subscribed_to_id BIGINT REFERENCES users(id),
    PRIMARY KEY (subscriber_id, subscribed_to_id),
    CHECK (subscriber_id <> subscribed_to_id)
);
```

Both foreign keys reference the same `users` table, but they represent
different roles: - `subscriber_id`: the user initiating the
subscription. - `subscribed_to_id`: the user being subscribed to.

The `CHECK` prevents a user from subscribing to themselves.

------------------------------------------------------------------------

# 5. Joins

A join combines rows from tables using a related-column condition.

Assume these patients:

    id name
  ---- -------
     1 Rahul
     2 Priya
     3 Amit

And these appointments:

     id   patient_id status
  ----- ------------ -----------
    101            1 completed
    102            1 scheduled
    103            2 completed

A patient may have zero, one, or multiple appointments.

## 5.1 INNER JOIN

``` sql
SELECT p.name, a.id AS appointment_id
FROM patients AS p
INNER JOIN appointments AS a
    ON a.patient_id = p.id;
```

Result conceptually:

  name      appointment_id
  ------- ----------------
  Rahul                101
  Rahul                102
  Priya                103

Only matching rows are returned. Amit is absent because Amit has no
appointment.

A parent with multiple matching children appears multiple times---once
for each matching pair.

## 5.2 LEFT JOIN

``` sql
SELECT p.name, a.id AS appointment_id
FROM patients AS p
LEFT JOIN appointments AS a
    ON a.patient_id = p.id;
```

Result conceptually:

  name      appointment_id
  ------- ----------------
  Rahul                101
  Rahul                102
  Priya                103
  Amit              `NULL`

Every row from the left (`patients`) table is preserved. When no
appointment matches, right-side columns are `NULL`.

Use a left join when the requirement is to show every patient, including
those with no appointments.

## 5.3 Choosing the left table

The table on the left is the set of rows you want to preserve. If the
requirement is "all patients, whether or not they have appointments,"
start from `patients` and left join appointments.

## 5.4 ON vs WHERE with LEFT JOIN --- important gotcha

These queries produce different results.

**Condition in `ON`:**

``` sql
SELECT p.name, a.status
FROM patients AS p
LEFT JOIN appointments AS a
    ON a.patient_id = p.id
   AND a.status = 'completed';
```

This preserves every patient. Only completed appointments are eligible
to match. A patient with no completed appointment receives `NULL` for
the appointment columns.

**Condition in `WHERE`:**

``` sql
SELECT p.name, a.status
FROM patients AS p
LEFT JOIN appointments AS a
    ON a.patient_id = p.id
WHERE a.status = 'completed';
```

This filters the joined result. For patients with no appointment,
`a.status` is `NULL`; `NULL = 'completed'` is not true, so those
patients are removed. The result behaves like an inner join for this
condition.

**Rule of thumb:** - Put a right-table condition in `ON` when it should
restrict which rows match but preserve all left-side rows. - Put it in
`WHERE` when it should filter the final result, including removing
unmatched left-side rows.

------------------------------------------------------------------------

# 6. Aggregation

Aggregation summarizes rows, often by category or entity.

## 6.1 Common aggregate functions

  Function          Purpose
  ----------------- ---------------------------------------------
  `COUNT(*)`        Counts rows
  `COUNT(column)`   Counts rows where that column is not `NULL`
  `SUM(column)`     Adds non-null values
  `AVG(column)`     Calculates average of non-null values
  `MIN(column)`     Smallest value
  `MAX(column)`     Largest value

## 6.2 GROUP BY

Example: total fees collected per doctor.

``` sql
SELECT doctor_id, SUM(fee) AS total_fees
FROM appointments
GROUP BY doctor_id;
```

`GROUP BY doctor_id` creates one group for each doctor ID. `SUM(fee)` is
calculated separately for each group.

### Gotcha: selected columns and grouping

A selected column must generally either: - Be part of the `GROUP BY`,
or - Be wrapped in an aggregate function.

For example, selecting `doctor_id, patient_id, SUM(fee)` while grouping
only by `doctor_id` is not generally valid because there may be many
patient IDs within a doctor's group.

## 6.3 WHERE vs HAVING

-   `WHERE` filters individual rows before grouping.
-   `HAVING` filters the groups after aggregation.

``` sql
SELECT doctor_id, SUM(fee) AS total_fees
FROM appointments
WHERE fee > 500
GROUP BY doctor_id
HAVING SUM(fee) > 1000;
```

Execution idea: 1. Remove appointments with `fee <= 500`. 2. Group the
remaining appointments by doctor. 3. Calculate each doctor's sum. 4.
Keep only groups whose sum exceeds 1000.

### Gotcha: SELECT aliases in HAVING

In PostgreSQL, do not generally rely on a `SELECT` alias being available
in `HAVING`:

``` sql
-- Prefer repeating the aggregate expression:
HAVING SUM(fee) > 1000
```

A CTE or subquery can be used if you want to filter by a named aggregate
result in an outer query.

------------------------------------------------------------------------

# 7. Subqueries, CTEs, and window functions

## 7.1 Subqueries

A subquery is a query nested inside another query. The expected number
of rows and columns depends on the context.

### Scalar subquery

A scalar subquery returns one value:

``` sql
SELECT *
FROM appointments
WHERE fee > (
    SELECT AVG(fee)
    FROM appointments
);
```

The inner query calculates one average fee. The outer query returns
appointments whose fee is above that average.

If a scalar subquery unexpectedly returns more than one row, PostgreSQL
raises an error.

### IN subquery

Use `IN` when comparing a value against a set of values:

``` sql
SELECT *
FROM patients
WHERE id IN (
    SELECT patient_id
    FROM appointments
    WHERE status = 'completed'
);
```

The inner query returns patient IDs; the outer query finds patients
whose IDs are in that set.

### EXISTS subquery

Use `EXISTS` when the question is whether at least one matching row
exists:

``` sql
SELECT p.*
FROM patients AS p
WHERE EXISTS (
    SELECT 1
    FROM appointments AS a
    WHERE a.patient_id = p.id
);
```

The inner query is correlated with the current patient. `EXISTS` tests
for the existence of a match; it does not need to return meaningful
selected values.

### Subquery in FROM

A subquery in `FROM` behaves like a derived table for the outer query:

``` sql
SELECT doctor_id, total_fees
FROM (
    SELECT doctor_id, SUM(fee) AS total_fees
    FROM appointments
    GROUP BY doctor_id
) AS doctor_totals
WHERE total_fees > 1000;
```

### Key distinction

Do not assume all subqueries return one value: - Scalar context: one
value. - `IN`: a set/list of values. - `EXISTS`: an existence test. -
`FROM`: a table-like result.

## 7.2 CTEs (`WITH`)

A Common Table Expression names an intermediate result for one SQL
statement.

``` sql
WITH doctor_fees AS (
    SELECT doctor_id, SUM(fee) AS total_fees
    FROM appointments
    GROUP BY doctor_id
)
SELECT doctor_id, total_fees
FROM doctor_fees
WHERE total_fees > 1000;
```

Benefits: - Makes complex queries easier to read. - Gives a meaningful
name to an intermediate result. - Can help structure multi-step query
logic.

### Gotcha

A CTE is primarily a query-organization tool, not an automatic
performance optimization. PostgreSQL's treatment of CTEs depends on
version and query shape; do not assume that using `WITH` always makes a
query faster.

## 7.3 Window functions

A window function calculates across a related set of rows while keeping
the individual rows in the output.

``` sql
SELECT
    id,
    doctor_id,
    fee,
    SUM(fee) OVER (
        PARTITION BY doctor_id
    ) AS doctor_total
FROM appointments;
```

If a doctor has three appointments, all three appointment rows remain in
the result, and each can display that doctor's total.

### GROUP BY vs window function

  -----------------------------------------------------------------------
  `GROUP BY`                          Window function
  ----------------------------------- -----------------------------------
  Combines rows into groups           Preserves individual rows

  Usually returns one row per group   Returns a result for each input row

  Useful for summary reports          Useful when each row also needs
                                      group-level context
  -----------------------------------------------------------------------

## 7.4 PARTITION BY and ORDER BY

-   `PARTITION BY` divides rows into groups for the window calculation.
-   `ORDER BY` inside `OVER (...)` defines the order used by ranking and
    ordered calculations.

Example ranking doctors by appointment fee:

``` sql
SELECT
    doctor_id,
    fee,
    RANK() OVER (ORDER BY fee DESC) AS fee_rank
FROM appointments;
```

An `ORDER BY` inside the window definition is not the same as the final
query-level `ORDER BY`. The former controls the window calculation; the
latter controls the order in which rows are returned to the client.

## 7.5 Ranking functions and ties

For fee values `900, 900, 700`:

  Function         Result
  ---------------- -----------
  `RANK()`         `1, 1, 3`
  `DENSE_RANK()`   `1, 1, 2`
  `ROW_NUMBER()`   `1, 2, 3`

-   `RANK()` assigns equal ranks to ties and leaves gaps afterward.
-   `DENSE_RANK()` assigns equal ranks to ties but does not leave gaps.
-   `ROW_NUMBER()` assigns a distinct sequence number to each row, even
    when values tie. If deterministic row numbering matters, add a
    tie-breaker to the window `ORDER BY`.

------------------------------------------------------------------------

# Consolidated gotchas and interview reminders (Topics 1--7)

1.  A phone number is an identifier; store it as text, not an integer.
2.  `NULL` and `''` are different. Use `IS NULL` to test for null.
3.  Use exact decimal types such as `NUMERIC` for money where exact
    decimal arithmetic is required.
4.  Always inspect the `WHERE` condition before executing `UPDATE` or
    `DELETE`.
5.  `DELETE` and `TRUNCATE` are different operations; PostgreSQL
    `TRUNCATE` can be rolled back in a transaction.
6.  A PostgreSQL `UNIQUE` constraint allows multiple `NULL`s by default.
7.  A `CHECK` constraint validates a row's allowed values; it does not
    solve a race between concurrent booking requests.
8.  A one-to-one PK/FK in the child table guarantees at most one child
    row per parent; it does not require every parent to have a child.
9.  In one-to-many, the foreign key normally belongs on the many side.
10. Many-to-many relationships need a junction table; a composite key
    can prevent duplicate pairs.
11. In a `LEFT JOIN`, a right-side condition in `WHERE` can remove
    unmatched left rows; putting it in `ON` can preserve them.
12. `WHERE` filters rows before grouping; `HAVING` filters groups after
    aggregation.
13. A scalar subquery must produce one value; `IN`, `EXISTS`, and `FROM`
    subqueries have different roles.
14. A CTE improves structure/readability but is not automatically
    faster.
15. `GROUP BY` collapses rows; window functions preserve the individual
    rows.
16. Window `ORDER BY` controls the window calculation; final `ORDER BY`
    controls returned row order.
