# PostgreSQL --- Detailed Quick Reference (Topics 8--10)

This document consolidates normalization, indexes, and query-performance
investigation, including the important gotchas discussed during
preparation. Examples use generic patient, doctor, appointment, and
vehicle-booking data.

------------------------------------------------------------------------

# 8. Schema design and normalization

## 8.1 Why normalization matters

Normalization is a way of structuring relational data to reduce
unnecessary duplication and avoid anomalies.

Suppose an `appointments` table repeats patient name and phone number in
every appointment:

  -----------------------------------------------------------------------------------
    appointment_id       patient_id patient_name   patient_phone   appointment_date
  ---------------- ---------------- -------------- --------------- ------------------
               101                1 Rahul          9876543210      2026-10-01

               102                1 Rahul          9876543210      2026-10-10

               103                2 Priya          9123456780      2026-10-12
  -----------------------------------------------------------------------------------

Patient information is duplicated whenever the same patient has multiple
appointments.

A normalized design separates the entities:

``` sql
CREATE TABLE patients (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(30)
);

CREATE TABLE appointments (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL REFERENCES patients(id),
    appointment_date DATE NOT NULL
);
```

Now the patient's name and phone are stored once, and each appointment
references the patient.

## 8.2 The three common anomalies

-   **Update anomaly:** If Rahul changes his phone number, multiple
    appointment rows would need updating. Missing one creates
    inconsistent data.
-   **Insert anomaly:** You may be unable to store a patient until an
    appointment exists if patient and appointment facts are forced into
    one table.
-   **Delete anomaly:** Deleting the last appointment for a patient
    could accidentally remove the only stored copy of the patient's
    information.

Normalization helps separate facts according to what they describe.

## 8.3 First Normal Form (1NF)

Core idea: each field should contain an atomic value for the intended
relational model; avoid list-like repeating groups in a single column.

Problematic structure:

    student_id subjects
  ------------ ------------------------
             1 Math, Science, English

Better design:

``` sql
CREATE TABLE subjects (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE student_subjects (
    student_id BIGINT REFERENCES students(id),
    subject_id BIGINT REFERENCES subjects(id),
    PRIMARY KEY (student_id, subject_id)
);
```

The relationship is represented as rows rather than a comma-separated
list.

The same principle applies to multiple phone numbers for one patient:
use a related table with one phone number per row rather than storing
`"111, 222, 333"` in one field.

### 1NF gotcha

Atomicity depends on the meaning of the data. A full address stored as
one string may be acceptable if the application treats it as one
indivisible value; if the application must query city, postal code, or
country separately, those may need separate columns. Design atomicity
around the operations and meaning required by the application.

## 8.4 Second Normal Form (2NF)

Core idea: - The table is in 1NF. - Every non-key attribute depends on
the whole candidate key, not just part of a composite candidate key.

This addresses **partial dependency**.

Example:

    student_id   course_id student_name   course_name
  ------------ ----------- -------------- -------------
             1          10 Rahul          SQL
             1          11 Rahul          Java
             2          10 Priya          SQL

Suppose the key is `(student_id, course_id)`: - `student_name` depends
only on `student_id`. - `course_name` depends only on `course_id`. -
Neither depends on the complete pair.

Separate the facts:

``` sql
CREATE TABLE students (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE courses (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE student_courses (
    student_id BIGINT REFERENCES students(id),
    course_id BIGINT REFERENCES courses(id),
    PRIMARY KEY (student_id, course_id)
);
```

Another example: if `(appointment_id, service_id)` is the composite key
of `appointment_services`, but `service_name` depends only on
`service_id`, service details belong in a separate `services` table.

### 2NF gotcha

Partial dependency is specifically about a proper subset of a composite
candidate key. If a table has only a single-column candidate key, it
cannot have a partial dependency on part of that key.

## 8.5 Third Normal Form (3NF)

Core idea: - The table is in 2NF. - Remove transitive dependencies of
non-key attributes on a key.

Example dependency chain:

`doctor_id → branch_id → branch_name`

If `doctors` stores both `branch_id` and `branch_name`, the branch name
is determined by the branch ID, not directly by the doctor ID.

Better design:

``` sql
CREATE TABLE branches (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE doctors (
    id BIGSERIAL PRIMARY KEY,
    branch_id BIGINT REFERENCES branches(id)
);
```

This avoids repeating the branch name for every doctor in that branch.

The same pattern occurs if an employee row stores `department_id` and
`department_name`: department details should normally be in a
`departments` table.

### 2NF vs 3NF

-   **2NF:** Is an attribute dependent on only part of a composite key?
-   **3NF:** Is a non-key attribute dependent on another non-key
    attribute rather than directly on a key?

Do not confuse partial dependency with transitive dependency.

## 8.6 Boyce--Codd Normal Form (BCNF)

The rule:

**Every determinant must be a superkey.**

A determinant is an attribute or set of attributes that determines
another attribute. A superkey is a set of columns that uniquely
identifies a row.

BCNF handles certain dependency patterns that can remain even when a
relation satisfies 3NF.

Example concept: - A table records `(student, subject, instructor)`. - A
subject determines its instructor. - But subject alone does not uniquely
identify a row because many students can take the same subject. -
Therefore `subject` is a determinant but not a superkey.

A possible decomposition is: -
`student_subjects(student_id, subject_id)` -
`subject_instructors(subject_id, instructor_id)`

The decomposition should preserve the intended facts and relationships.

## 8.7 Normal forms at a glance

  -----------------------------------------------------------------------
  Normal form             Main rule               Typical issue addressed
  ----------------------- ----------------------- -----------------------
  1NF                     Atomic values; no       List-like values in a
                          repeating lists/groups  column

  2NF                     No partial dependency   Attributes depending on
                          on part of a composite  only one part of a
                          candidate key           composite key

  3NF                     No transitive           One non-key fact
                          dependency of non-key   determining another
                          attributes on a key     

  BCNF                    Every determinant is a  Determinants that are
                          superkey                not candidate/superkeys
  -----------------------------------------------------------------------

## 8.8 Foreign-key deletion behavior and history

A foreign key does not automatically cascade.

``` sql
CREATE TABLE appointments (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL
        REFERENCES patients(id)
        ON DELETE CASCADE
);
```

Deleting a patient deletes referencing appointments. It does not work in
the reverse direction: deleting an appointment does not delete its
patient.

**Design gotcha:** Historical records may need to survive even when a
parent record is no longer active. In that case, hard deletion or
cascading deletion may be inappropriate. Consider the application's
retention and audit requirements before selecting `CASCADE`.

## 8.9 When denormalization may be justified

Normalization is a strong default for correctness and maintainability.
Denormalization can be useful when a measured read/reporting requirement
justifies keeping repeated or precomputed information.

Examples: - Store a historical price on an order line so later price
changes do not alter the historical sale amount. - Maintain a
precomputed summary for a read-heavy report, with a deliberate
refresh/consistency strategy.

Trade-off: - Reads may become simpler or faster. - Writes and
consistency become more complex. - Every duplicated value needs a clear
rule for how it stays correct.

Do not denormalize merely to avoid learning joins or because it might be
faster.

------------------------------------------------------------------------

# 9. Indexes

## 9.1 What an index is

An index is a separate data structure maintained by PostgreSQL that
helps locate table rows more efficiently for suitable queries.

Without a useful index, PostgreSQL may use a Sequential Scan, examining
table pages to find rows that satisfy a condition.

Example:

``` sql
CREATE INDEX idx_patients_phone
ON patients(phone);
```

This creates an index on `patients.phone`.

**Important:** Creating an index does not force PostgreSQL to use it.
The query planner chooses a plan based on estimated costs and available
statistics.

## 9.2 Sequential Scan vs Index Scan

### Sequential Scan

PostgreSQL reads through the table's pages and checks rows against the
query condition.

It can be efficient when: - The table is small. - The query needs a
large portion of the table. - Reading table pages sequentially is
cheaper than jumping around to fetch many individual rows.

### Index Scan

PostgreSQL traverses the index to find matching index entries and then
fetches the corresponding table rows as needed.

It can be efficient when: - The condition matches a small part of a
large table. - The index supports the filter or ordering. - The cost of
fetching matching rows is lower than scanning the table.

An Index Scan is not automatically faster. The best plan depends on the
data and query.

## 9.3 B-tree basics

-   B-tree is PostgreSQL's default index type.
-   It stores indexed values in an ordered structure and maintains
    references to table rows.
-   It is useful for many equality and range conditions and can support
    compatible ordering.
-   The table itself is not necessarily physically sorted in index
    order.

Example:

``` sql
CREATE INDEX idx_appointments_date
ON appointments(appointment_date);
```

This may help queries that filter or order by `appointment_date`,
depending on selectivity, table size, and plan costs.

## 9.4 Index trade-offs

Benefits: - Faster suitable lookups and filters. - Can help joins on
indexed columns. - Can help satisfy `ORDER BY` without a separate sort
in suitable cases.

Costs: - Additional disk space. - Extra work on `INSERT`, `UPDATE`, and
`DELETE` to maintain indexes. - Too many indexes can increase write cost
and maintenance overhead.

A practical rule: create indexes to support real, frequent query
patterns---not automatically on every column.

## 9.5 Foreign-key columns are not automatically indexed

PostgreSQL creates an index for a primary key and for a unique
constraint, but it does **not** automatically create an index on the
referencing side of a foreign key.

If the application frequently retrieves appointments for one patient:

``` sql
CREATE INDEX idx_appointments_patient_id
ON appointments(patient_id);
```

This can also help some parent-delete/update checks involving the
foreign key, because PostgreSQL may need to find referencing rows.

## 9.6 Composite indexes

Example:

``` sql
CREATE INDEX idx_appointments_patient_status
ON appointments(patient_id, status);
```

The index is ordered by: 1. `patient_id` 2. `status` within each
`patient_id`

Typical useful query patterns:

``` sql
-- Leading column only
SELECT *
FROM appointments
WHERE patient_id = 25;
```

``` sql
-- Leading column plus second column
SELECT *
FROM appointments
WHERE patient_id = 25
  AND status = 'completed';
```

A query filtering only on `status` is generally not well served by this
index:

``` sql
SELECT *
FROM appointments
WHERE status = 'completed';
```

### Leftmost-prefix rule

A useful starting mental model is that a composite index `(A, B)` is
most directly useful for: - `A` - `A` plus `B`

It is generally not the ideal index for `B` alone because the entries
are ordered first by `A`.

This is a rule of thumb, not an absolute statement that PostgreSQL can
never use such an index for a second-column-only condition. Planner
choices and PostgreSQL scan strategies can vary. Design index order
around actual query patterns.

## 9.7 Composite index for filtering and sorting

Suppose a common query retrieves one patient's appointments newest
first:

``` sql
SELECT *
FROM appointments
WHERE patient_id = 25
ORDER BY appointment_date DESC;
```

A matching index could be:

``` sql
CREATE INDEX idx_appointments_patient_date
ON appointments(patient_id, appointment_date DESC);
```

Why this order? - First, PostgreSQL can locate entries for
`patient_id = 25`. - Within that patient range, entries are ordered by
appointment date descending. - The index may therefore support both
filtering and ordering, avoiding a separate sort.

The index does not physically reorder the table rows. It stores its own
ordered entries and references to the corresponding table rows.

## 9.8 Selectivity

Selectivity describes how narrowly a filter reduces the candidate rows.

Imagine a table with one million users: - `email = 'person@example.com'`
may match one row: high selectivity. - `gender = 'female'` may match a
large fraction of the table: low selectivity.

A highly selective condition often makes an index more useful because it
narrows the matching set substantially.

### Gotchas

-   Low selectivity does not make an index automatically useless; the
    right choice depends on the query, table size, distribution, and
    other plan requirements.
-   High selectivity does not guarantee an index will be used; a
    sequential scan can still be cheaper for a small table or a query
    needing many columns/rows.
-   An index's usefulness depends on the actual workload, not just the
    column's name or data type.

## 9.9 Why PostgreSQL may ignore an existing index

Suppose a table contains one million appointments and 900,000 have
`status = 'completed'`.

``` sql
SELECT *
FROM appointments
WHERE status = 'completed';
```

Even if `status` has an index, PostgreSQL may choose a Sequential Scan.

Reasoning: 1. The index may identify 900,000 matching entries. 2.
PostgreSQL may then need to fetch a large number of table rows. 3. Those
table-row fetches may be scattered across table pages. 4. Reading the
table sequentially can be cheaper than using the index to fetch almost
the whole table.

So: - A Sequential Scan does not mean the index is missing or broken. -
An index is an option for the planner, not an instruction. - The planner
chooses the plan it estimates to be cheapest.

------------------------------------------------------------------------

# 10. Query performance and execution plans

## 10.1 Start with EXPLAIN

`EXPLAIN` shows the plan PostgreSQL estimates it will use:

``` sql
EXPLAIN
SELECT *
FROM appointments
WHERE patient_id = 25;
```

Look for plan nodes such as: - `Seq Scan` --- sequentially scans table
pages. - `Index Scan` --- uses an index to find row references and
fetches table rows. - Other node types may appear for joins, sorts,
aggregates, and other operations.

`EXPLAIN` does not execute the query itself.

## 10.2 EXPLAIN ANALYZE

``` sql
EXPLAIN ANALYZE
SELECT *
FROM appointments
WHERE patient_id = 25;
```

`EXPLAIN ANALYZE` executes the statement and reports actual execution
information in addition to the plan.

It can show: - Actual rows produced by each plan node. - Actual timing
for plan nodes. - How many times each node ran (`loops`). - Estimated
rows alongside actual rows.

### Critical safety gotcha

`EXPLAIN ANALYZE` really executes the statement. Be careful with
data-changing statements such as `UPDATE` and `DELETE`, because they can
modify data. If testing a write statement, use an appropriate safe test
environment or a transaction that is deliberately rolled back after
inspection.

## 10.3 EXPLAIN vs EXPLAIN ANALYZE

  Command               Executes the statement? Main purpose
  ------------------- ------------------------- ----------------------------------------
  `EXPLAIN`                                  No See the estimated plan
  `EXPLAIN ANALYZE`                         Yes Compare estimates with actual behavior

The execution time reported by PostgreSQL is database execution timing.
It is not necessarily the complete time observed by a browser or API
client: network latency, application processing, serialization, and
other overhead can add time outside the database execution.

## 10.4 Reading a sample plan

``` text
Seq Scan on appointments
  (cost=0.00..25000.00 rows=50000)
  (actual time=0.050..120.000 rows=48000 loops=1)
```

Interpretation:

  -----------------------------------------------------------------------
  Part                                Meaning
  ----------------------------------- -----------------------------------
  `Seq Scan on appointments`          The node performs a Sequential Scan
                                      on this table

  `cost=0.00`                         Estimated startup cost

  `..25000.00`                        Estimated total cost

  `rows=50000`                        Estimated number of rows this node
                                      will return

  `actual time=0.050..120.000`        Approximate time to first row and
                                      completion time, in milliseconds

  `actual rows=48000`                 Actual number of rows returned by
                                      this node

  `loops=1`                           This node executed once
  -----------------------------------------------------------------------

### Important correction: estimated rows are not scanned rows

`rows=50000` is the planner's estimate of how many rows the node will
return. It does not mean the Sequential Scan examined exactly 50,000
rows.

A sequential scan may examine every row in the table but return only the
rows that satisfy the condition. The number scanned and the number
returned are different concepts.

## 10.5 Understanding cost

In:

``` text
(cost=0.00..25000.00 rows=50000)
```

-   First number: estimated startup cost---the work before the node can
    produce its first row.
-   Second number: estimated total cost---the estimated work to complete
    the node and produce all rows.

**Cost is not milliseconds or seconds.** It is an internal, relative
unit used by PostgreSQL's planner to compare candidate execution plans.

For example, if one plan is estimated at 8,000 and another at 25,000,
the first is generally more attractive to the planner. These values are
not elapsed-time measurements and should not be read as 8,000 ms and
25,000 ms.

## 10.6 Why estimated and actual performance can differ

The planner makes estimates using statistics and cost assumptions. Real
execution can differ because of: - Statistics that do not reflect the
latest data. - Data distributions that are skewed or more complex than
the estimates capture. - A much larger or smaller number of matching
rows than estimated. - Disk I/O and cache state. - Many scattered row
fetches during an Index Scan. - Other runtime conditions.

A slow network can make an entire application request slower, but it is
not normally the explanation for why PostgreSQL's estimated plan cost
differs from the execution time reported inside `EXPLAIN ANALYZE`.
Network and application overhead are outside the database's plan-node
execution time.

## 10.7 ANALYZE and planner statistics

`ANALYZE` collects or refreshes table statistics used by the query
planner.

``` sql
ANALYZE appointments;
```

Statistics help PostgreSQL estimate: - How many rows a condition may
match. - How values are distributed. - How selective a filter may be.

Example: - A table originally has 10,000 appointments. - A bulk import
adds 500,000 more. - The distribution of statuses may also change. - Old
statistics may no longer describe the current table well. - Running
`ANALYZE appointments;` refreshes statistics so future planning can use
more representative estimates.

## 10.8 Do we run ANALYZE after every write?

Usually, no.

PostgreSQL has **autovacuum**, which monitors table activity and can
automatically trigger maintenance, including `ANALYZE`, when configured
thresholds are reached.

Manual `ANALYZE` can be useful: - After a large bulk import or major
data change. - During query-performance troubleshooting when estimates
appear inaccurate. - When you need to refresh statistics rather than
wait for automatic maintenance.

Do not assume every insert/update needs a manual `ANALYZE`.

## 10.9 VACUUM vs ANALYZE

  -----------------------------------------------------------------------
  Operation                           Main purpose
  ----------------------------------- -----------------------------------
  `VACUUM`                            Handles dead tuples and routine
                                      space/database maintenance

  `ANALYZE`                           Refreshes statistics used by the
                                      query planner
  -----------------------------------------------------------------------

Autovacuum can perform vacuuming and automatic analysis according to its
thresholds and configuration. They are related maintenance activities,
but they solve different problems.

## 10.10 A practical query-performance troubleshooting workflow

When a query becomes slow:

1.  **Reproduce the query** with representative parameters and data.
2.  **Run `EXPLAIN`** to inspect the estimated execution plan.
3.  **Run `EXPLAIN ANALYZE`** when safe, to see actual row counts and
    timings.
4.  **Compare estimated rows with actual rows.** A large mismatch
    suggests the planner's assumptions may be inaccurate.
5.  **Inspect the plan structure.** Look at scans, joins, sorts,
    aggregates, and loops rather than focusing only on whether an index
    appears.
6.  **Check statistics.** If estimates look stale or inaccurate,
    consider `ANALYZE`.
7.  **Re-run the plan** after refreshing statistics and compare.
8.  **Investigate other causes.** Consider selectivity, index column
    order, large result sets, join conditions, sorting, disk I/O, and
    application/network overhead.
9.  **Only then consider a schema or index change.** An index is not
    automatically the answer to every slow query.

### Example diagnostic reasoning

A query filters by `status = 'completed'` and 90% of the table has that
status.

-   An index exists on `status`.
-   PostgreSQL chooses `Seq Scan`.
-   This may be a sensible choice because the query needs most of the
    table.
-   The correct next step is to inspect the plan and actual behavior,
    not to assume the index is broken.

A different query filters by a patient ID and returns a handful of
appointments from a very large table. An index on `patient_id` may be
useful because it narrows the result substantially.

------------------------------------------------------------------------

# Consolidated gotchas and interview reminders (Topics 8--10)

1.  Normalization reduces duplication and update/insert/delete
    anomalies.
2.  1NF concerns atomic values; 2NF concerns partial dependency on part
    of a composite candidate key; 3NF concerns transitive dependency;
    BCNF requires every determinant to be a superkey.
3.  A foreign key does not automatically cascade deletes. Understand the
    direction of `ON DELETE CASCADE`.
4.  Historical facts may need to be stored separately from an entity's
    current state.
5.  Denormalization can improve selected reads but creates a consistency
    and maintenance responsibility.
6.  PostgreSQL's default index type is B-tree.
7.  PostgreSQL does not automatically create an index on the referencing
    side of a foreign key.
8.  Composite index order matters. `(patient_id, status)` is naturally
    suited to filters beginning with `patient_id`, not generally to
    `status` alone.
9.  The table is not physically sorted just because an index is ordered.
10. Indexes consume storage and add write-maintenance work.
11. High selectivity often makes an index useful; low selectivity can
    make a Sequential Scan cheaper, but neither guarantees a particular
    plan.
12. A Sequential Scan does not mean an index is broken or absent.
13. `EXPLAIN` shows estimated plans; `EXPLAIN ANALYZE` executes and
    reports actual plan-node behavior.
14. `EXPLAIN ANALYZE` can execute data-changing statements---use care.
15. `rows=...` in the estimated plan is the estimated number of rows
    returned by a node, not necessarily the number of rows scanned.
16. `actual time=a..b` is in milliseconds; the first value is time to
    first row and the second is completion time for the node.
17. Planner `cost` is a relative estimate, not wall-clock time.
18. Network latency can affect end-to-end application time, but not the
    database's internal plan-node execution measurement in the same way.
19. `ANALYZE` refreshes planner statistics; it does not create indexes.
20. You generally do not manually run `ANALYZE` after every write;
    autovacuum can trigger it automatically.
21. `VACUUM` and `ANALYZE` are different maintenance operations.
22. Investigate the actual execution plan and estimates before adding
    indexes or changing schema.
