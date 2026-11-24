# PostgreSQL from Node.js — Detailed Quick Reference (Topic 13)

This document consolidates the concepts covered in **Phase 1, Topic 13: PostgreSQL from Node.js** for the Rentee vehicle-rental examples.

The focus is on using PostgreSQL safely and reliably from a Node.js application with the `pg` package.

---

## 1. Connecting Node.js to PostgreSQL

A Node.js application commonly uses the `pg` package to communicate with PostgreSQL.

Install it:

```bash
npm install pg
```

There are two important objects:

- **`Pool`**: manages a collection of reusable database connections.
- **`Client`**: represents one specific database connection.

For most application code, a shared pool is the normal starting point.

---

## 2. Connection pooling

### 2.1 Why use a connection pool?

Opening a brand-new database connection for every API request creates unnecessary overhead. Connections consume database resources, and too many simultaneous connections can exhaust the database's connection capacity.

A connection pool manages reusable connections:

1. An API request needs to query PostgreSQL.
2. The pool provides an available connection.
3. The query runs on that connection.
4. For a normal one-off query, the pool releases the connection back for reuse.
5. If every connection is busy, a request may have to wait for one.

Think of the pool as a limited set of reusable service desks. Requests use a desk, finish their work, and free it for the next request.

### 2.2 Creating a pool

```js
// db.js
const { Pool } = require("pg");

const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,

  max: 10,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

module.exports = pool;
```

Important settings:

| Setting | Meaning |
|---|---|
| `max` | Maximum number of connections this pool manages |
| `idleTimeoutMillis` | How long an idle connection can remain in the pool before being closed |
| `connectionTimeoutMillis` | How long the pool waits while trying to establish/acquire a connection before timing out |

**Important:** `max: 10` means this particular pool manages at most 10 connections. If the application runs multiple server instances, each instance may have its own pool. The total possible connections can therefore be greater than 10.

### 2.3 Running a normal query with `pool.query()`

```js
const pool = require("./db");

async function getVehicles() {
  const result = await pool.query(
    "SELECT * FROM vehicles"
  );

  return result.rows;
}
```

For an independent query, `pool.query()` is convenient because it acquires a connection and releases it after the query completes.

### 2.4 What happens when the pool is full?

If all pool connections are busy, another request may wait for a connection. If it cannot obtain one within the configured connection timeout, the operation fails with a connection/acquisition timeout error.

Distinguish these settings:

- `connectionTimeoutMillis`: limits waiting to obtain/establish a connection.
- `idleTimeoutMillis`: controls how long an unused connection remains open.

### 2.5 When to use `pool.query()` and when to use a client

Use `pool.query()` for independent queries.

Use a checked-out client when multiple statements must run on the **same database connection**, especially when using a transaction. This is covered in Section 6.

---

## 3. Parameterized queries

### 3.1 The problem: SQL injection

Never build SQL by directly concatenating untrusted user input into the query string.

Unsafe example:

```js
const email = req.body.email;

const result = await pool.query(
  `SELECT * FROM users WHERE email = '${email}'`
);
```

The value of `email` is inserted directly into SQL text. A malicious input could change the meaning of the SQL statement instead of being treated as an ordinary email value.

This vulnerability is called **SQL injection**.

### 3.2 Use placeholders and pass values separately

With PostgreSQL and `pg`, use `$1`, `$2`, and so on:

```js
const email = req.body.email;

const result = await pool.query(
  "SELECT * FROM users WHERE email = $1",
  [email]
);
```

Here:

- `$1` is the first value placeholder.
- `[email]` supplies the value for `$1`.
- The driver sends the parameter separately from the SQL structure, so the input is treated as data rather than executable SQL.

With two parameters:

```js
const result = await pool.query(
  "SELECT * FROM users WHERE name = $1 AND age = $2",
  [name, age]
);
```

The mapping is:

- `$1` → `name`
- `$2` → `age`

**Rule:** Do not concatenate user-provided values into SQL. Use parameterized queries.

### 3.3 Parameterized INSERT

```js
const result = await pool.query(
  `INSERT INTO vehicles (registration_number, brand, daily_rent)
   VALUES ($1, $2, $3)
   RETURNING *`,
  [registrationNumber, brand, dailyRent]
);
```

This protects the values while keeping the SQL statement readable.

### 3.4 Important limitation: placeholders are for values, not identifiers

This does not work for a table name:

```js
// Incorrect
pool.query(
  "SELECT * FROM $1",
  ["users"]
);
```

A parameter placeholder represents a value. It cannot stand for SQL structure such as:

- a table name
- a column name
- a SQL keyword

The table or column identifier must be part of the SQL statement when PostgreSQL parses it.

If an application genuinely needs a dynamic table name, validate it against a strict allowlist before inserting it into the SQL string:

```js
const allowedTables = ["users", "vehicles"];

if (!allowedTables.includes(tableName)) {
  throw new Error("Invalid table name");
}

const result = await pool.query(
  `SELECT * FROM ${tableName} WHERE id = $1`,
  [id]
);
```

In this example:

- `tableName` is checked against known, permitted identifiers.
- `id` remains a parameterized value.
- Never interpolate an unchecked identifier supplied by a user.

---

## 4. `RETURNING` in PostgreSQL

An INSERT, UPDATE, or DELETE can return affected row values using `RETURNING`.

Example:

```js
const result = await pool.query(
  `INSERT INTO users (name, email)
   VALUES ($1, $2)
   RETURNING id, name, email`,
  [name, email]
);

const newUser = result.rows[0];
```

### Why use `RETURNING`?

The database can return the inserted row, including values generated by the database, such as an automatically generated ID.

This is useful when:

- the application needs the new record's ID immediately;
- the next step needs values from the inserted record;
- you want to avoid a separate SELECT query just to retrieve the inserted row.

`RETURNING` does not itself decide whether the operation succeeded. If the INSERT succeeds, PostgreSQL returns the requested row values. If it fails, PostgreSQL reports an error. If no row is affected by a statement, there may be no returned row.

For UPDATE, `RETURNING` can also help verify which row was changed:

```js
const result = await pool.query(
  `UPDATE vehicles
   SET status = 'rented'
   WHERE id = $1 AND status = 'available'
   RETURNING id`,
  [vehicleId]
);
```

---

## 5. Transactions from Node.js

### 5.1 Why transactions matter

Consider a Rentee booking operation. Two database operations belong to one business flow:

1. Change the vehicle's status from `available` to `rented`.
2. Insert a booking record.

Suppose the vehicle update succeeds but the booking insert fails. Without a transaction, the database could be left with a vehicle marked as rented but no corresponding booking.

A transaction groups related database statements into one unit:

- If all required operations succeed, commit them.
- If an operation fails, roll back the transaction so its changes are not committed.

### 5.2 `BEGIN`, `COMMIT`, and `ROLLBACK`

- `BEGIN`: starts a transaction.
- `COMMIT`: makes the transaction's changes permanent.
- `ROLLBACK`: aborts the transaction and discards its uncommitted changes.

### 5.3 Why all transaction statements must use the same client

A PostgreSQL transaction belongs to one database connection.

`pool.query()` may acquire a different connection for each call. Therefore, using it separately for `BEGIN`, `UPDATE`, `INSERT`, and `COMMIT` does not reliably put those statements in one transaction.

For a transaction, check out one client and use that same client for every statement.

### 5.4 Basic transaction pattern

```js
const client = await pool.connect();

try {
  await client.query("BEGIN");

  await client.query(
    "UPDATE vehicles SET status = 'rented' WHERE id = $1",
    [vehicleId]
  );

  await client.query(
    "INSERT INTO bookings (vehicle_id) VALUES ($1)",
    [vehicleId]
  );

  await client.query("COMMIT");
} catch (error) {
  await client.query("ROLLBACK");
  throw error;
} finally {
  client.release();
}
```

Key points:

- `pool.connect()` checks out one specific connection.
- All transaction statements use `client.query()`.
- `COMMIT` is used only after the required operations succeed.
- On failure, `ROLLBACK` aborts the transaction.
- `client.release()` returns the checked-out client to the pool. Put cleanup in `finally` so it runs on both success and failure.

The example shows the basic pattern. In production code, rollback itself can fail if the connection has already been lost; error handling should account for that and still ensure the client is released appropriately.

---

## 6. Conditional updates and checking `rowCount`

A transaction alone does not know the business rule that a booking is allowed only when a vehicle is available. The application must enforce that rule, while the database operation must safely handle concurrent attempts.

### 6.1 Use a conditional UPDATE

```js
const updateResult = await client.query(
  `UPDATE vehicles
   SET status = 'rented'
   WHERE id = $1 AND status = 'available'
   RETURNING id`,
  [vehicleId]
);
```

The update changes the vehicle only if:

- the requested vehicle exists; and
- its current status is `available`.

### 6.2 Check the affected row count

```js
if (updateResult.rowCount === 0) {
  throw new Error("Vehicle is unavailable");
}
```

If `rowCount` is zero, no vehicle row was updated. The application must not continue with booking insertion.

If the update succeeds, the application can proceed:

```js
await client.query(
  `INSERT INTO bookings (vehicle_id, customer_id)
   VALUES ($1, $2)`,
  [vehicleId, customerId]
);
```

### 6.3 Putting it together

```js
async function createBooking(vehicleId, customerId) {
  const client = await pool.connect();

  try {
    await client.query("BEGIN");

    const updateResult = await client.query(
      `UPDATE vehicles
       SET status = 'rented'
       WHERE id = $1 AND status = 'available'
       RETURNING id`,
      [vehicleId]
    );

    if (updateResult.rowCount === 0) {
      throw new Error("Vehicle is unavailable");
    }

    const bookingResult = await client.query(
      `INSERT INTO bookings (vehicle_id, customer_id)
       VALUES ($1, $2)
       RETURNING id, vehicle_id, customer_id`,
      [vehicleId, customerId]
    );

    await client.query("COMMIT");

    return bookingResult.rows[0];
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  } finally {
    client.release();
  }
}
```

This combines three protections:

1. **Conditional UPDATE:** only an available vehicle can be changed to rented.
2. **`rowCount` check:** prevents booking insertion if no vehicle was updated.
3. **Transaction:** makes the vehicle update and booking insertion succeed or fail together.

Isolation levels and locks help coordinate concurrent transactions. They do not replace application checks for business rules.

---

## 7. Handling PostgreSQL errors in Node.js

Database operations can fail for different reasons. The application should recognize expected failures and avoid exposing raw database details to API clients.

### 7.1 Example: duplicate email

Suppose the `users.email` column has a UNIQUE constraint. Inserting an email that already exists causes PostgreSQL error code `23505` (`unique_violation`).

```js
try {
  const result = await pool.query(
    `INSERT INTO users (name, email)
     VALUES ($1, $2)
     RETURNING id`,
    [name, email]
  );

  return res.status(201).json(result.rows[0]);
} catch (error) {
  if (error.code === "23505") {
    return res.status(409).json({
      message: "An account with this email already exists."
    });
  }

  console.error(error);

  return res.status(500).json({
    message: "Internal server error"
  });
}
```

Why `409 Conflict`?

The request conflicts with existing database state: an account already uses that email. `400 Bad Request` can be appropriate for other invalid request inputs, but `409` is a more specific choice for a uniqueness conflict.

Do not return the raw PostgreSQL error message to the client. Map known failures to safe, useful API messages.

### 7.2 Foreign-key violation

PostgreSQL error code `23503` means `foreign_key_violation`.

For example, a booking references a `vehicle_id` that does not exist. The API may return `404 Not Found` with a message such as `"Vehicle not found."`

Be careful: a foreign-key violation can arise from different relationships and operations. Identify which relationship failed before mapping it to an HTTP response. Do not blindly map every `23503` to the same response.

### 7.3 Useful PostgreSQL error codes

| Code | Meaning | Typical application consideration |
|---|---|---|
| `23505` | Unique violation | Often map a known duplicate-resource conflict to HTTP 409 |
| `23503` | Foreign-key violation | Identify the missing or invalid relationship; a missing referenced resource may map to 404 |
| `23514` | Check constraint violation | Usually indicates a value violated a database rule; map to a safe validation/business response where appropriate |
| `40001` | Serialization failure | Retry the entire transaction when safe |
| `40P01` | Deadlock detected | A bounded retry of the entire transaction may be appropriate |

These codes identify database-level causes. The API response should reflect the actual business meaning, not just mechanically expose the database code.

---

## 8. Serialization failures and retries

### 8.1 What is a serialization failure?

With PostgreSQL's SERIALIZABLE isolation level, concurrent transactions are monitored to ensure their committed results are consistent with some serial execution order.

Sometimes PostgreSQL determines that concurrent operations cannot safely commit together under that rule. It aborts one transaction with error code `40001` (`serialization_failure`).

A serialization failure is not necessarily just a lock failure. It is a concurrency-control outcome.

### 8.2 Retry the entire transaction

When PostgreSQL aborts a transaction due to `40001`, do not retry only the failed SQL statement.

Why?

- The transaction has been aborted.
- Earlier reads and decisions may have been made using a snapshot that is no longer suitable.
- The whole business operation needs to be evaluated again against a fresh transaction and snapshot.

For example, a booking transaction may:

1. Check vehicle availability.
2. Update vehicle status.
3. Insert a booking.
4. Update a related record.

If serialization fails during the operation, start a new transaction and repeat the complete logical operation. Do not continue from the failed statement using assumptions from the previous transaction.

### 8.3 Retry carefully

A robust retry strategy should:

- retry only errors that are appropriate to retry, such as `40001` and sometimes `40P01`;
- retry the complete transaction from the beginning;
- limit the number of attempts;
- use a small delay/backoff between attempts;
- stop and return an appropriate failure if retries continue to fail.

Do not retry every database error blindly. A duplicate-email error (`23505`), for example, usually needs a meaningful client response rather than repeated attempts.

Also be careful with external side effects. A transaction retry should not accidentally send duplicate emails, charge a payment provider twice, or repeat another external action. Database transaction retries and external-system idempotency need deliberate coordination.

---

## 9. Quick comparison: the main Node.js `pg` patterns

| Need | Recommended pattern |
|---|---|
| One independent query | `pool.query(...)` |
| User-provided values in SQL | `$1`, `$2` placeholders with a values array |
| Retrieve inserted or updated row values | SQL `RETURNING` |
| Multiple statements in one transaction | `const client = await pool.connect()` |
| Ensure statements share a transaction | Use the same `client` for `BEGIN`, all statements, and `COMMIT`/`ROLLBACK` |
| Release a checked-out client | `client.release()` in `finally` |
| Prevent a booking when a vehicle is unavailable | Conditional `UPDATE` and check `rowCount` |
| Handle known database errors | Inspect `error.code` and map to safe application/API behavior |
| Handle serialization failure | Retry the entire transaction with a bounded retry strategy |

---

## 10. Interview-ready summary

A concise explanation of how to access PostgreSQL safely from Node.js:

> I use the `pg` connection pool so database connections can be reused rather than opening a new connection for every request. For independent queries, I use `pool.query()`. I use parameterized queries with `$1`, `$2`, and a values array to prevent SQL injection. When I need multiple operations to succeed atomically, I check out one client with `pool.connect()` and use that same client for `BEGIN`, all transaction statements, and `COMMIT` or `ROLLBACK`. I release the client in a `finally` block. For business rules such as vehicle availability, I use conditional updates and inspect `rowCount` before continuing. Finally, I handle known PostgreSQL error codes deliberately, return safe API errors, and retry the entire transaction for retryable concurrency failures such as serialization errors.

---

## 11. Self-check questions

Try answering these without looking at the explanations:

1. Why is a connection pool preferable to creating a new database connection for every API request?
2. What is the difference between `connectionTimeoutMillis` and `idleTimeoutMillis`?
3. Why is direct string interpolation of user input dangerous in SQL?
4. Why can `$1` represent an email value but not a table name?
5. What benefit does `RETURNING` provide after an INSERT?
6. Why must all statements in one transaction use the same checked-out client?
7. Why is a transaction not enough by itself to enforce the rule that a vehicle must be available before booking?
8. What does `rowCount === 0` tell you after the conditional vehicle update?
9. How would you handle PostgreSQL error `23505` in an API?
10. Why should a transaction be retried from the beginning after a `40001` serialization failure?

---

**Topic 13 scope:** connection pools, parameterized queries, SQL injection, identifier limitations, `RETURNING`, transactions from Node.js, same-client transaction handling, conditional updates, `rowCount`, PostgreSQL error handling, and safe transaction retries.
