# Database Transactions — Foundations

---

## What is a Transaction

A transaction is a boundary around a group of database operations. Everything inside that boundary either all succeeds or all fails. There is no partial success.

The classic example: transfer ₹1000 from account A to account B.

Under the hood, that's two separate operations:
1. Deduct ₹1000 from A
2. Add ₹1000 to B

If the system crashes between step 1 and step 2 — A is debited, B is never credited. Money disappears into an inconsistent state. A transaction wraps both operations so either both happen or neither does.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 1000 WHERE id = 'B';
COMMIT;
```

Two exits from every transaction:

**COMMIT** — all operations succeeded. Changes are permanent and visible to everyone.

**ROLLBACK** — something went wrong. Undo everything. State is exactly as if the transaction never ran.

From the outside, a transaction either happened or it didn't. That's the entire concept.

---

## Why Transactions Exist

Real operations are almost never a single step.

- Order placement: create order + deduct inventory + create payment + trigger notification
- User registration: create user + create profile + send verification email
- Fund transfer: debit source + credit destination + log audit trail

Without transactions, any crash, bug, or error mid-operation leaves the database in a broken in-between state. There's no automatic way to detect it or recover cleanly.

Transactions give you a hard boundary: either the entire operation happened, or it never happened at all.

---

## ACID

ACID describes the four guarantees a database makes about every transaction.

### Atomicity

All or nothing. If any single operation inside the transaction fails, every operation is rolled back. You cannot commit a partial transaction.

### Consistency

The database moves from one valid state to another valid state. Every constraint you've defined — unique keys, foreign keys, not-null, check constraints — is enforced at commit time. A transaction that would violate any constraint is rejected and rolled back.

One important caveat: consistency only covers what the database knows about. Business rules the DB doesn't know — "a user cannot have more than 5 active subscriptions" — are the application's responsibility, not the database's.

### Isolation

Concurrent transactions don't interfere with each other. While your transaction is in progress, other transactions don't see your intermediate state. You also don't see their incomplete work.

How strictly this is enforced is configurable — that's what isolation levels control.

### Durability

Once committed, data survives. Crashes, power cuts, restarts — a committed transaction is permanent. The database achieves this through disk-based logging (covered in the advanced guide).

---

## Isolation Levels

Isolation is not binary. It's a dial. Weaker isolation means faster performance but more risk of seeing inconsistent data. Stronger isolation means safer data but more locking overhead.

The SQL standard defines 4 levels:

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | possible | possible | possible |
| Read Committed | prevented | possible | possible |
| Repeatable Read | prevented | prevented | possible |
| Serializable | prevented | prevented | prevented |

Each level prevents one more class of anomaly than the one above it.

### The three anomalies (short version)

**Dirty read:** you read data another transaction wrote but hasn't committed yet. If that transaction rolls back, you acted on data that officially never existed.

**Non-repeatable read:** you read the same row twice in one transaction and get different values — another transaction committed a change between your two reads.

**Phantom read:** you run the same query twice and get different rows — another transaction inserted or deleted rows matching your filter between your two queries.

### Which level to use

**Read Committed** is the default in most systems (Postgres, Oracle). Prevents dirty reads, which are the most dangerous. Good enough for most application workloads.

**Repeatable Read** when your transaction logic reads a value, makes a decision based on it, and you need that value to stay stable throughout the transaction.

**Serializable** when correctness is non-negotiable — financial calculations, ticket booking, any place where two concurrent transactions making decisions based on shared data would cause a business problem.

---

## What Happens When a Transaction Fails

### Scenario 1: application detects an error

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';
  -- your application code detects something wrong
ROLLBACK;
-- the debit is reversed. account A is back to original balance.
```

The database undoes every change made inside the transaction. State is exactly as it was before BEGIN.

### Scenario 2: a constraint is violated

A constraint violation — unique key conflict, foreign key violation, null in a required field — is detected at the point of the violation or at commit time. The DB automatically rejects the commit and rolls back the entire transaction. You get an error. Your application must catch it and decide what to do.

### Scenario 3: a deadlock is detected

Two transactions each waiting for the other's lock. The DB detects this cycle, picks one transaction as the victim, rolls it back automatically, and lets the other proceed.

The rolled-back client receives an error like `ERROR: deadlock detected`. Your application must catch this and retry the transaction from scratch.

Deadlock retries are a normal part of production code that uses transactions under write contention.

### Scenario 4: the DB crashes mid-transaction

Power cut. Process killed. Hardware failure. The DB was in the middle of a transaction.

On restart, the DB reads its internal write-ahead log and reverses every transaction that was in-progress at crash time. No partial state leaks out. Committed transactions are preserved. The DB comes back in a clean, consistent state automatically.

If your application crashes without sending COMMIT, the DB treats the dropped connection as an implicit ROLLBACK. All changes from that session are reversed.

### Savepoints — partial rollback inside a transaction

You can place named markers inside a transaction and roll back only to that marker without aborting everything.

```sql
BEGIN;
  INSERT INTO orders (id, user_id) VALUES (1, 42);

  SAVEPOINT before_payment;
  INSERT INTO payments (order_id, amount) VALUES (1, 500);  -- this fails

  ROLLBACK TO SAVEPOINT before_payment;  -- only the payment insert is undone
  -- order insert is still in place

  -- continue with other logic...
COMMIT;
```

Useful when you want to attempt a risky sub-operation but don't want its failure to kill the entire transaction.

---

## Transactions Across Entities

### Multi-entity (same DB instance)

Multiple tables, multiple documents, within one database. This is the easy case — wrap everything in a local transaction. The DB handles all of it.

```sql
BEGIN;
  INSERT INTO orders (id, user_id, total) VALUES (1, 42, 999);
  UPDATE inventory SET stock = stock - 1 WHERE product_id = 5;
  INSERT INTO payments (order_id, amount) VALUES (1, 999);
COMMIT;
```

If any of these fail, all three are rolled back together. This is exactly what transactions were built for. No extra complexity.

### Cross-cluster (same DB type, multiple nodes or shards)

Your data is partitioned across multiple independent DB nodes. User A is on shard 1, user B is on shard 2. A transfer between them crosses shard boundaries — now two independent DB instances need to agree on a commit.

The problem: you can't just run a local transaction. There's no shared engine controlling both shards. If shard 1 commits and shard 2 fails — you have an inconsistency with no automatic fix.

This is where distributed commit protocols like 2PC (Two-Phase Commit) come in — covered in the advanced guide. Some databases have built-in cross-shard transaction support (CockroachDB, Google Spanner). With traditional DBs, you manage it yourself.

### Cross-DB (different systems entirely)

Writing to Postgres and publishing an event to Kafka. Writing to MongoDB and updating a Redis cache. Calling a downstream HTTP service as part of your operation.

These systems have no shared transaction manager. There is no native atomic guarantee across them. This is the hardest category — and the one where most production systems quietly have bugs.

The core danger: **dual write**. You write to system A, then write to system B. If the first succeeds and the second fails — both systems are permanently out of sync, often with no immediate error and no automatic recovery.

Patterns like the Outbox pattern and Sagas exist specifically to handle this — covered in the advanced guide.

The minimum thing to know right now: never write to two independent systems in sequence without a coordination mechanism. Silent divergence is worse than a visible error.

---

## How Different Databases Handle Transactions

| Database | Default Isolation | Has MVCC | Transaction Scope | Key Behavior to Know |
|---|---|---|---|---|
| PostgreSQL | Read Committed | Yes | Full ACID, row-level locking | Read Uncommitted behaves identically to Read Committed — Postgres never exposes dirty reads regardless of setting |
| MySQL (InnoDB) | Repeatable Read | Yes | Full ACID, row-level locking | Uses gap locks at Repeatable Read — prevents most phantom reads even though the SQL standard says it shouldn't. Stronger in practice than the standard defines. |
| MySQL (MyISAM) | None | No | No transactions | Table-level locks only. Legacy engine. Avoid for any transactional work. |
| Oracle | Read Committed | Yes | Full ACID, row-level locking | No Read Uncommitted level at all. Doesn't expose dirty reads. |
| SQL Server | Read Committed | Only if enabled | Full ACID | Two modes for Read Committed: default uses locking (readers block writers); with RCSI enabled, uses row versioning (readers don't block writers). Must explicitly enable. |
| SQLite | Serializable | Partial | Full ACID, single writer at a time | WAL mode allows concurrent reads, but only one writer at a time. Good for embedded use, not for high-concurrency writes. |
| MongoDB | Read Committed (per document) | Yes (since 4.0) | Single-document always atomic; multi-document transactions since 4.0 | Single-document operations were atomic even before 4.0 — MongoDB's document model is designed so that what would be multiple rows in SQL is one document. Multi-doc transactions have more overhead. |
| CockroachDB | Serializable | Yes | Full ACID natively distributed | Built for distribution. Cross-node transactions work out of the box. Uses Raft consensus under the hood. |
| Google Spanner | Serializable | Yes | Full ACID globally distributed | Uses TrueTime (GPS + atomic clocks) for global transaction ordering. The only system with external consistency at global scale. |

### The practical summary

Most applications use Postgres or MySQL InnoDB and run at Read Committed or Repeatable Read. That covers the majority of transactional use cases.

MongoDB is fine for single-document atomicity (which covers a lot of NoSQL use cases). Its multi-document transaction support exists but has overhead — design your document schema to avoid needing it where possible.

Cross-node distributed ACID (CockroachDB, Spanner) is available if you need it, but comes with latency cost. These systems wait for consensus across nodes before acknowledging a commit.
