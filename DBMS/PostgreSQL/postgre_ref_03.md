# PostgreSQL Detailed Quick Reference — Topics 11 and 12

This reference covers **Topic 11: Transactions and ACID** and **Topic 12: Isolation Levels, Locks, Concurrent Updates, and Deadlocks** from the Full-Stack SDE Interview Preparation roadmap.

The examples use a vehicle-rental application. SQL examples target PostgreSQL.

---

# Topic 11 — Transactions and ACID

## 1. What is a transaction?

A transaction is a group of database operations treated as one logical unit of work.

For example, transferring ₹1,000 from Account A to Account B requires two updates:

1. Subtract ₹1,000 from Account A.
2. Add ₹1,000 to Account B.

Both operations must succeed together. If only the debit succeeds, the database is left in an incorrect business state.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE id = 1;

UPDATE accounts
SET balance = balance + 1000
WHERE id = 2;

COMMIT;
```

If an operation fails and the transaction should not continue, roll it back:

```sql
ROLLBACK;
```

- `BEGIN` starts a transaction.
- `COMMIT` makes its changes committed.
- `ROLLBACK` cancels its uncommitted changes.

A transaction should cover the database operations that must succeed or fail together. Avoid keeping transactions open longer than necessary.

## 2. ACID properties

ACID describes four important transaction properties.

### Atomicity

**All operations in the transaction succeed, or none of their changes are committed.**

In the bank-transfer example, the debit and credit must not be committed separately.

If the credit operation fails, the debit must be rolled back too.

### Consistency

**A transaction must preserve the database's defined rules and constraints.**

For example:

```sql
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    balance NUMERIC(12, 2) NOT NULL
        CHECK (balance >= 0)
);
```

The `CHECK` constraint prevents a committed row from having a negative balance.

Consistency does not mean PostgreSQL automatically understands every business rule. The application and schema designer must express the rules using appropriate constraints, transactions, and logic.

### Isolation

**Concurrent transactions should not interfere in ways that violate the guarantees of the selected isolation level.**

For example, two customers may attempt to book the same vehicle at nearly the same time. The database must coordinate their operations so the application does not accidentally confirm two bookings for one vehicle.

Isolation levels determine what a transaction can observe about concurrent transactions.

### Durability

**Once a transaction is successfully committed, its changes should survive a database crash, subject to the configured durability guarantees.**

PostgreSQL uses Write-Ahead Logging (WAL):

1. Changes generate WAL records.
2. Before PostgreSQL confirms a durable commit, the relevant WAL records are flushed according to the configured commit/durability settings.
3. The actual modified table pages may be written to their data files later.
4. During recovery, PostgreSQL can replay WAL records to restore committed changes.

The important distinction: PostgreSQL does not need to write every modified table page to its final location before acknowledging the commit. WAL supports crash recovery.

## 3. Why `BEGIN` and `COMMIT` alone do not solve every race condition

A transaction groups operations atomically, but that alone does not guarantee that two concurrent transactions cannot make the same decision from the same earlier state.

Imagine two requests both do this:

1. Read vehicle 25 and see `available`.
2. Decide that the vehicle can be booked.
3. Create a booking.
4. Change the vehicle status to `rented`.

If both requests read `available` before either one commits, both may attempt to create a booking unless the operation is protected by a suitable concurrency strategy.

Use a conditional update, an explicit row lock, an appropriate isolation level, or a database constraint depending on the business rule.

## 4. Database commit succeeded, but the application did not receive the response

A network failure can occur after PostgreSQL commits but before the application receives the confirmation.

The application then has an **uncertain outcome**:

- The transaction may have committed successfully.
- The response may simply have been lost.
- Retrying the operation blindly could create a duplicate business action, such as a second payment record.

This is one reason production APIs use **idempotency keys** for operations that may be retried. The same logical request can be recognized and prevented from producing duplicate effects.

Idempotency is a broader backend-engineering topic; it will be covered in more depth in the backend phase.

## 5. What PostgreSQL handles vs what the developer decides

PostgreSQL provides mechanisms such as:

- MVCC (Multi-Version Concurrency Control).
- Row and table locks.
- Transaction execution and isolation guarantees.
- Deadlock detection.
- WAL and crash recovery.

The developer still needs to choose and implement:

- Correct transaction boundaries.
- Suitable isolation levels.
- Constraints that express business rules.
- Conditional updates or explicit locks where needed.
- Correct handling of transaction failures and retries.

**Interview takeaway:** PostgreSQL provides concurrency-control mechanisms, but the application developer must use them to protect the actual business invariants.

---

# Topic 12 — Isolation Levels, Locks, Concurrent Updates, and Deadlocks

## 1. Why isolation levels matter

Multiple users can read and modify data at the same time. Isolation levels define which effects of concurrent transactions are visible to one another.

Common SQL isolation-level names:

1. `READ UNCOMMITTED`
2. `READ COMMITTED`
3. `REPEATABLE READ`
4. `SERIALIZABLE`

PostgreSQL-specific point:

- PostgreSQL treats `READ UNCOMMITTED` as `READ COMMITTED`.
- Dirty reads are therefore not allowed in PostgreSQL, even if a transaction requests `READ UNCOMMITTED`.
- PostgreSQL's default isolation level is `READ COMMITTED`.

## 2. Common concurrency anomalies

### Dirty read

A transaction reads another transaction's uncommitted change.

Example:

- Transaction A changes a balance from 5,000 to 3,000 but has not committed.
- Transaction B reads 3,000.
- Transaction A rolls back.

B has read a value that was never committed.

PostgreSQL prevents dirty reads at all its supported isolation levels.

### Non-repeatable read

A transaction reads the same row twice and gets different committed values because another transaction updated and committed that row between the reads.

Example under `READ COMMITTED`:

| Step | Transaction B | Transaction A |
|---|---|---|
| 1 | Reads balance: 5,000 | |
| 2 | | Updates balance to 3,000 |
| 3 | | Commits |
| 4 | Reads balance again: 3,000 | |

The two reads in B return different values.

### Phantom read

A transaction repeats a query that searches for rows matching a condition and sees a different set of rows because another transaction inserted or deleted matching rows and committed.

For example, a transaction counts available vehicles in a branch twice. Another transaction inserts a new available vehicle between those queries.

PostgreSQL's `REPEATABLE READ` snapshot behavior prevents phantom reads in the ordinary SQL sense: later queries in the same transaction continue to use the same snapshot.

### Write skew

Two transactions read overlapping data, make decisions based on that shared state, and then update different rows. Both changes may commit even though the combined result violates a business rule.

Example: a clinic requires at least one doctor to remain on call.

Initial state:

| Doctor | On call |
|---|---|
| A | true |
| B | true |

Two transactions run concurrently under PostgreSQL `REPEATABLE READ`:

- Transaction 1 checks that two doctors are on call, then sets Doctor A to `false`.
- Transaction 2 checks that two doctors are on call, then sets Doctor B to `false`.

They update different rows. Both may commit, leaving no doctor on call.

This is why a stable snapshot by itself does not guarantee that every multi-row business invariant is protected.

## 3. `READ COMMITTED`

`READ COMMITTED` is PostgreSQL's default isolation level.

Key behavior:

- Each statement gets a new snapshot.
- A statement sees rows committed before that statement began, plus the transaction's own earlier changes.
- A later statement in the same transaction can see commits that happened after an earlier statement.

Example:

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

SELECT balance
FROM accounts
WHERE id = 1;

-- Another transaction updates and commits the balance.

SELECT balance
FROM accounts
WHERE id = 1;

COMMIT;
```

The two `SELECT` statements may return different values.

**Use when:** the normal per-statement view of committed data is sufficient. This is a common default for application workloads.

**Limitation:** repeated reads in the same transaction are not guaranteed to return the same committed version.

## 4. `REPEATABLE READ`

In PostgreSQL, `REPEATABLE READ` gives the transaction a consistent snapshot established by its first non-transaction-control statement. Subsequent queries use that same snapshot.

Example:

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT balance
FROM accounts
WHERE id = 1;

-- Another transaction updates and commits the balance.

SELECT balance
FROM accounts
WHERE id = 1;

COMMIT;
```

The second query continues to see the value from the transaction's original snapshot.

Important PostgreSQL behavior:

- It prevents non-repeatable reads.
- It also prevents phantom reads through its snapshot-based behavior.
- It can still allow write skew when transactions read overlapping data but update different rows.
- Conflicting concurrent updates can cause a transaction to fail with a serialization-related error; the application may need to retry the entire transaction.

**Use when:** a transaction needs a stable view of data across multiple queries.

**Limitation:** a stable snapshot does not automatically protect every business rule involving multiple rows.

## 5. `SERIALIZABLE`

`SERIALIZABLE` provides the strongest standard isolation guarantee.

The successful committed result must be equivalent to some serial ordering of the transactions—as if the transactions had run one after another in some order.

PostgreSQL implements this using **Serializable Snapshot Isolation (SSI)**:

- Transactions can still execute concurrently.
- PostgreSQL tracks read/write dependencies between transactions.
- If it detects a dangerous dependency pattern that could produce a non-serializable result, it aborts a transaction.
- The application receives a serialization failure and should retry the complete transaction when appropriate.

A retry must start the transaction again from the beginning so it gets a fresh snapshot and repeats all relevant reads and decisions. Retrying only the final `UPDATE` is not sufficient.

Example concept: if two transactions each see one seat remaining and both attempt to reserve it, a serializable execution cannot allow both to successfully reserve that one seat based on the same last-seat state. PostgreSQL may abort one transaction.

**Use when:** correctness depends on multi-row or predicate-based rules that are difficult to protect with simpler atomic operations or explicit locks.

**Trade-off:** serialization failures and retries add application complexity. `SERIALIZABLE` is not automatically the best choice for every query.

## 6. Isolation-level comparison

| Isolation level | Snapshot behavior in PostgreSQL | Important behavior |
|---|---|---|
| `READ UNCOMMITTED` | Behaves as `READ COMMITTED` | Dirty reads are prevented |
| `READ COMMITTED` | New snapshot per statement | Non-repeatable reads can occur |
| `REPEATABLE READ` | Stable transaction snapshot | Prevents non-repeatable and phantom reads; write skew can still occur |
| `SERIALIZABLE` | Snapshot plus serializability checks | Prevents non-serializable committed outcomes; may abort a transaction |

Choose an isolation level based on the business invariant and the cost of handling retries—not simply because a higher level sounds safer.

## 7. Concurrent updates to the same row

PostgreSQL coordinates conflicting updates to the same row using row-level locks.

Suppose two transactions execute:

```sql
UPDATE vehicles
SET status = 'rented'
WHERE id = 25;
```

If Transaction A updates the row first, Transaction B generally waits for A's conflicting row lock to be released. After A commits, B may continue and update the row too.

This means a row lock coordinates access, but it does not by itself enforce the business rule that a vehicle may only transition from `available` to `rented`.

## 8. Conditional atomic update — a strong choice for simple state transitions

For a simple booking transition, put the condition and the update in the same SQL statement:

```sql
UPDATE vehicles
SET status = 'rented'
WHERE id = 25
  AND status = 'available'
RETURNING id;
```

How it behaves when two customers race:

1. Customer A's request updates the available vehicle and receives its `id`.
2. Customer B's request waits if it conflicts with A's row update.
3. After A commits, PostgreSQL rechecks whether B's row still satisfies the `WHERE` condition.
4. Since the status is now `rented`, B updates zero rows.
5. B's application must not continue with booking creation or payment.

In Node.js with the `pg` library, the application can inspect `rowCount`:

```js
const result = await client.query(
  `UPDATE vehicles
   SET status = $1
   WHERE id = $2
     AND status = $3
   RETURNING id`,
  ["rented", 25, "available"]
);

if (result.rowCount === 0) {
  // Vehicle does not exist, or it is no longer available.
  // Stop the booking flow.
}
```

The query returning zero rows can mean:

- The vehicle ID does not exist.
- The vehicle exists but its status is not `available`.

The application may perform a separate read if it needs to distinguish those outcomes for its response.

**Why this is useful:** the condition and state change are atomic in one statement. It avoids a separate application-level read-then-update decision and is often the simplest choice for a straightforward state transition.

**Important:** PostgreSQL still uses its internal locking/concurrency mechanisms. The benefit is not “no locks”; it is that the application does not need to manage a separate lock query for this simple operation.

## 9. `SELECT ... FOR UPDATE` — explicit row locking for multi-step workflows

Sometimes the application needs to read a row, make a decision, and perform several related database operations.

In that case, use a transaction and lock the relevant row before making the decision:

```sql
BEGIN;

SELECT status
FROM vehicles
WHERE id = 25
FOR UPDATE;

-- Application checks that status is 'available'.

UPDATE vehicles
SET status = 'rented'
WHERE id = 25;

-- Insert the booking and any other related database records.

COMMIT;
```

`FOR UPDATE` acquires a row-level lock on the selected row. A conflicting update or row-locking query must wait until the transaction ends.

Why use it?

Without the lock, another transaction could change the row after the application reads it but before it writes based on that earlier value. The lock protects the row while the transaction performs its multi-step database workflow.

Important rules:

- Use it inside a transaction.
- Check the row's current business state after acquiring the lock.
- Keep the transaction short.
- Commit or roll back promptly so locks are released.
- Lock all relevant rows when the business rule depends on more than one row, using a consistent lock order.

## 10. What happens when a transaction holding a lock fails?

A lock is normally held until the transaction ends.

- On `COMMIT`, the transaction ends and its locks are released.
- On `ROLLBACK`, its uncommitted changes are discarded and its locks are released.
- If the database session disconnects or crashes with an open transaction, PostgreSQL normally rolls back that transaction and releases its locks during cleanup/recovery.

A lock wait can still last a long time if a session remains connected but leaves a transaction open or idle.

Useful PostgreSQL timeout settings include:

```sql
SET lock_timeout = '5s';
```

This limits how long a statement waits to acquire a lock before PostgreSQL raises an error.

Other relevant settings:

- `statement_timeout`: limits how long a statement may run.
- `idle_in_transaction_session_timeout`: terminates sessions left idle while a transaction remains open.

The application should handle such errors and ensure the transaction is rolled back or the connection is safely recovered before reuse.

## 11. Deadlocks

A deadlock occurs when transactions form a circular wait: each transaction is waiting for a lock held by another transaction in the cycle.

Example:

- Transaction A locks Vehicle 25.
- Transaction B locks Vehicle 30.
- Transaction A now requests Vehicle 30 and waits for B.
- Transaction B now requests Vehicle 25 and waits for A.

Neither can proceed.

PostgreSQL detects deadlocks and aborts one transaction as the victim. That transaction is rolled back, its locks are released, and the other transaction can continue.

### How to reduce deadlocks

1. **Use a consistent lock order.** For example, when locking multiple vehicle rows, always lock the lower ID first.
2. **Keep transactions short.** Do not perform slow network calls or wait for user input while holding database locks.
3. **Lock only the rows needed.**
4. **Handle deadlock errors.** A retry may be appropriate for a transaction that is safe to retry.

Example of a consistent lock order:

```sql
BEGIN;

SELECT *
FROM vehicles
WHERE id IN (25, 30)
ORDER BY id
FOR UPDATE;

-- Perform the related database changes.

COMMIT;
```

Consistent ordering reduces the chance that two transactions acquire the same set of locks in opposite orders.

## 12. `NOWAIT` and `SKIP LOCKED`

These options change what happens when a row is already locked.

### Default behavior — wait

```sql
SELECT *
FROM vehicles
WHERE id = 25
FOR UPDATE;
```

If another transaction holds a conflicting lock, PostgreSQL waits for it to be released.

### `NOWAIT` — fail immediately

```sql
SELECT *
FROM vehicles
WHERE id = 25
FOR UPDATE NOWAIT;
```

If the row is already locked, PostgreSQL returns an error immediately instead of waiting.

Useful when the application prefers to fail quickly or ask the user to retry rather than wait for a lock.

### `SKIP LOCKED` — skip locked rows

```sql
SELECT *
FROM jobs
WHERE status = 'pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

If another worker has locked a pending job, this query skips that row and tries to return another unlocked pending job.

Useful for multiple background workers claiming jobs from a shared queue.

Example:

- Worker A locks Job 1.
- Worker B runs the query with `SKIP LOCKED`.
- Worker B skips Job 1 and can claim another unlocked pending job.
- If Job 1 is the only pending job, Worker B gets zero rows. Its application can wait, poll again, or use another queue-notification mechanism.

**Caution:** `SKIP LOCKED` intentionally gives an inconsistent view of the rows because locked rows are omitted. It is useful for queue-like work distribution, not as a general-purpose way to read all matching records. It also does not guarantee fairness: a frequently locked row may be skipped repeatedly.

## 13. Choosing the right concurrency strategy

| Situation | Typical approach |
|---|---|
| Simple state transition, such as `available` → `rented` | Conditional `UPDATE ... WHERE status = 'available' RETURNING ...` |
| Read a row, make a decision, then perform multiple related DB operations | Transaction with `SELECT ... FOR UPDATE` |
| Multiple workers claim different queue jobs | `FOR UPDATE SKIP LOCKED` |
| Application should fail quickly if a row is busy | `FOR UPDATE NOWAIT` |
| Multi-row business invariant that needs serializable behavior | Consider `SERIALIZABLE` and implement whole-transaction retries |
| Several rows must be locked | Acquire locks in a consistent order |

These are starting points, not universal rules. Constraints and the exact business invariant still matter.

## 14. Interview-ready explanations

### What is the difference between `READ COMMITTED` and `REPEATABLE READ` in PostgreSQL?

`READ COMMITTED` takes a new snapshot for each statement, so two reads in one transaction can see different committed values. `REPEATABLE READ` uses a stable transaction snapshot, so subsequent reads continue to see the same snapshot. PostgreSQL's `REPEATABLE READ` also prevents phantom reads, but it can still allow write skew.

### Why might `REPEATABLE READ` not be enough?

A stable snapshot does not prevent every multi-row anomaly. Two transactions can read the same set of rows and update different rows based on that shared state. Both can commit and violate a business invariant. This is write skew. `SERIALIZABLE` prevents non-serializable committed outcomes by detecting dangerous dependencies and may abort a transaction.

### What is the difference between a conditional update and `SELECT ... FOR UPDATE`?

A conditional update is ideal for a simple atomic state transition because the condition and modification happen in one statement. `SELECT ... FOR UPDATE` is useful when a workflow must read and inspect a row, then perform multiple related database operations while protecting that row from conflicting changes.

### What is a deadlock, and how does PostgreSQL handle it?

A deadlock is a circular wait among transactions for locks held by one another. PostgreSQL detects the cycle and aborts one transaction so the other can proceed. Applications should keep transactions short, acquire locks in a consistent order, and handle retryable failures.

### Why use `SKIP LOCKED`?

It allows workers to skip rows already claimed by other workers and claim other available work without waiting. It is useful for database-backed job queues, but it does not guarantee FIFO processing or fairness.

---

# Final revision checklist

You should be able to explain these without relying on memorized definitions:

- [ ] Why related database operations belong in a transaction.
- [ ] Atomicity, consistency, isolation, and durability.
- [ ] Why `BEGIN`/`COMMIT` alone does not automatically prevent race conditions.
- [ ] Why a commit response can be lost and why retries need idempotency.
- [ ] PostgreSQL's default isolation level and its treatment of `READ UNCOMMITTED`.
- [ ] Dirty reads, non-repeatable reads, phantom reads, and write skew.
- [ ] The difference between `READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE`.
- [ ] Why PostgreSQL may abort a serializable transaction and why the whole transaction must be retried.
- [ ] How row locks coordinate concurrent updates.
- [ ] When to use a conditional update versus `SELECT ... FOR UPDATE`.
- [ ] What `NOWAIT` and `SKIP LOCKED` do.
- [ ] What causes deadlocks and how consistent lock ordering helps.
- [ ] Why a booking flow must stop when its conditional vehicle update returns zero rows.
- [ ] Why booking creation and the vehicle status update should be in the same transaction.

**Practical Rentee rule:** secure the vehicle's availability with an atomic conditional update or an appropriate row lock, create the booking in the same database transaction, and do not proceed to payment when the vehicle could not be claimed. Payment-provider calls are external operations and need a carefully designed workflow; do not hold a database transaction open while waiting on a slow network call.
