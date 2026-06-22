# Chapter 4: Oracle Transactions, Locks & MVCC
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> **Transaction** = going to a restaurant:
> You order, eat, and pay - all as one complete event.
> If your card declines, the entire meal is "undone" (rolled back).
> You don't pay for half a meal.
>
> **Locking** = toilet cubicle door:
> When you're inside, you lock it. Others wait outside.
> Oracle does this for data - but smartly!
>
> **MVCC** = Google Docs:
> Two people can read/edit the same doc simultaneously.
> You see a "snapshot" version while someone else edits live.
> Readers don't block writers; writers don't block readers.

---

## 1. ACID Properties - The Foundation

| Property | Meaning | Example |
|----------|---------|---------|
| **Atomicity** | All or nothing | Transfer $100: both debit AND credit happen, or neither |
| **Consistency** | Data stays valid | Constraints, rules always satisfied after transaction |
| **Isolation** | Transactions don't interfere | Your draft order doesn't affect someone else's query |
| **Durability** | Committed = permanent | Even if server crashes, COMMIT survives |

---

## 2. Transaction Basics

```sql
-- Start (implicit - happens when you run first DML)
UPDATE accounts SET balance = balance - 100 WHERE acc_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE acc_id = 2;

COMMIT;    -- Make permanent
-- OR
ROLLBACK;  -- Undo everything back to last COMMIT

-- Partial rollback using SAVEPOINT
SAVEPOINT sp1;
UPDATE orders SET status = 'HOLD' WHERE customer_id = 500;
SAVEPOINT sp2;
DELETE FROM order_lines WHERE order_id = 9999; -- Oops, wrong!
ROLLBACK TO sp2;  -- Undo the delete only, keep the UPDATE
COMMIT;
```

> **Key**: Oracle starts a transaction automatically on first DML. DDL (CREATE, ALTER, TRUNCATE) auto-commits.

---

## 3. Oracle MVCC - Multi-Version Concurrency Control

> **This is the most important Oracle concurrency concept. Know it well.**

### The Problem Without MVCC
Session A reads 1000 rows from orders - takes 30 seconds.
Session B updates 500 of those rows while A is reading.
Without MVCC: A either blocks B or gets inconsistent data.

### Oracle's Solution: Read Consistency via Undo
1. When Session A starts a query, Oracle notes the **SCN (System Change Number)** at that moment
2. As A reads data, if a block has been modified AFTER A's SCN, Oracle reconstructs the **original version** from **undo segments**
3. A always sees a **consistent snapshot** of data as of when the query started

```
Session A starts query (SCN = 5000)
Session B updates row X (SCN = 5010)
Session A reaches row X -> Oracle reads undo -> shows row X as it was at SCN 5000
Result: A gets consistent read, B's update is not blocked
```

> **Golden rule**: In Oracle, **readers don't block writers, writers don't block readers**.

### Undo Segments (Rollback Segments)
- Store the "before image" of every changed row
- Used for: 1) ROLLBACK, 2) Read consistency (MVCC), 3) Flashback queries
- Managed by UNDO tablespace
- `ORA-01555: snapshot too old` -> undo expired before query finished (increase undo retention)

---

## 4. Types of Locks in Oracle

### 4a. DML Locks (Row-Level)
- When you `UPDATE` or `DELETE` a row, Oracle locks **that specific row**
- Other sessions can update OTHER rows in the same table simultaneously
- **TX Lock** (transaction lock) - held until COMMIT or ROLLBACK

```
Session A: UPDATE orders SET status='HOLD' WHERE order_id=100;  -- locks row 100
Session B: UPDATE orders SET status='HOLD' WHERE order_id=200;  -- fine! different row
Session C: UPDATE orders SET status='HOLD' WHERE order_id=100;  -- WAITS (same row)
```

### 4b. Table-Level Locks (DDL Locks)
- **RS** (Row Share) - SELECT FOR UPDATE
- **RX** (Row Exclusive) - DML (UPDATE/DELETE/INSERT)
- **S** (Share) - prevents other DML while allowing reads
- **SRX** (Share Row Exclusive)
- **X** (Exclusive) - DDL on table, prevents everything
- RX lock = "I'm editing this table's rows, don't change the structure."

### 4c. SELECT FOR UPDATE
```sql
-- Lock rows you intend to update - prevents others from modifying
SELECT order_id, status FROM orders
WHERE customer_id = 100
FOR UPDATE;

-- With NOWAIT: don't wait, fail immediately if locked
SELECT * FROM orders WHERE order_id = 1001 FOR UPDATE NOWAIT;

-- With WAIT: wait up to N seconds
SELECT * FROM orders WHERE order_id = 1001 FOR UPDATE WAIT 5;
```

---

## 5. Deadlocks

> Two cars in a narrow alley, head-to-head, both waiting for the other to reverse.

```
Session A: Locks row 100, then tries to lock row 200 -> waits for B
Session B: Locks row 200, then tries to lock row 100 -> waits for A
-> DEADLOCK!
```

Oracle detects deadlocks automatically:
- One session gets `ORA-00060: deadlock detected`
- That session's last statement is rolled back (not the whole transaction)
- The other session proceeds

**Prevention**:
- Always lock resources in the same order in all transactions
- Keep transactions short

---

## 6. V$LOCK and V$SESSION - Diagnosing Locks

```sql
-- Find who is blocking whom
SELECT l.sid blocking_sid, s.username blocker,
       l2.sid waiting_sid, s2.username waiter
FROM v$lock l
JOIN v$lock l2 ON l.id1 = l2.id1 AND l.id2 = l2.id2
JOIN v$session s ON l.sid = s.sid
JOIN v$session s2 ON l2.sid = s2.sid
WHERE l.block = 1 AND l2.request > 0;

-- Kill a blocking session
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;
```

---

## 7. Transaction Isolation Levels

Oracle supports these isolation levels (SQL standard):

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|--------------------|----|
| **Read Committed** (Oracle default) | No | Possible | Possible |
| **Serializable** | No | No | No |

```sql
-- Set serializable for entire transaction
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Read-only transaction (no DML allowed, consistent snapshot)
SET TRANSACTION READ ONLY;
```

> Oracle's Read Committed = statement-level consistency (each statement sees its own snapshot).
> Serializable = transaction-level consistency (entire transaction sees same snapshot).

---

## 8. Commit and Redo Log Writing

> When you COMMIT, Oracle's "receipt" (redo log) is written to disk first before saying "done". That's the guarantee of durability.

**Commit flow**:
1. LGWR writes redo buffer to redo log files on disk
2. Oracle returns "commit complete" to user
3. DBWR writes dirty data blocks to data files lazily (checkpoint)

This is **write-ahead logging (WAL)** - redo is always written before data.

---

## 9. Flashback - Time Travel Using Undo

```sql
-- See what a table looked like 1 hour ago
SELECT * FROM orders AS OF TIMESTAMP (SYSDATE - 1/24);

-- See what it looked like at a specific SCN
SELECT * FROM orders AS OF SCN 5000000;

-- Flashback entire table (requires FLASHBACK privilege)
FLASHBACK TABLE orders TO TIMESTAMP (SYSDATE - 1/24);

-- Query flashback versions of a row
SELECT versions_starttime, versions_endtime, status
FROM orders VERSIONS BETWEEN TIMESTAMP MINVALUE AND MAXVALUE
WHERE order_id = 1001;
```

> Requires sufficient undo retention. Setting: `UNDO_RETENTION` parameter.

---

## 10. Autonomous Transactions

> A sub-task that commits independently of the parent task.
> Like logging "I attempted this" even if the main operation fails.

```sql
CREATE OR REPLACE PROCEDURE log_audit(p_action VARCHAR2) AS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO audit_log (action, log_time) VALUES (p_action, SYSDATE);
  COMMIT;  -- This commits only the INSERT, not the parent transaction
END;
/
```

Used for: audit logging, error logging that must persist even if the main transaction rolls back.

---

## 11. Two-Phase Commit (Distributed Transactions)

> Used when a single transaction spans **multiple databases**.
> Oracle ensures all databases commit or all rollback together.

**Phase 1 - Prepare**: All participants confirm they are ready
**Phase 2 - Commit**: Coordinator tells all to commit

`In-doubt transactions`: If coordinator fails between phases -> `DBA_2PC_PENDING`

---

## 🎯 Top Interview Questions & Answers

**Q1: How does Oracle ensure read consistency without blocking?**
> MVCC - Oracle uses undo segments to reconstruct old versions of data. When a query starts, it notes the SCN. If a block has been changed after that SCN, Oracle reads the original version from undo. Readers never block writers.

**Q2: What is ORA-01555?**
> "Snapshot too old." Undo data has been overwritten before a long-running query could use it for read consistency. Fix: increase UNDO_RETENTION, increase undo tablespace size, or optimize slow queries.

**Q3: What happens when a deadlock occurs in Oracle?**
> Oracle's SMON detects the deadlock, one of the sessions gets ORA-00060, and that session's last statement is rolled back (not the entire transaction). The other session continues.

**Q4: What is the difference between COMMIT and SAVEPOINT?**
> COMMIT makes all changes permanent and releases locks. SAVEPOINT marks a point you can ROLLBACK TO (partial rollback) without undoing the entire transaction.

**Q5: What is an autonomous transaction? When would you use it?**
> A transaction that executes independently of its parent. Used for audit/error logging that must persist even if the calling transaction rolls back. Declared with PRAGMA AUTONOMOUS_TRANSACTION.

**Q6: What is SELECT FOR UPDATE NOWAIT?**
> Locks selected rows for update. NOWAIT means fail immediately with ORA-00054 if rows are already locked (instead of waiting). Useful for real-time UIs where you don't want the user to wait.

---

## 📝 Quick Revision Summary

```
ACID: Atomicity, Consistency, Isolation, Durability

MVCC: Readers don't block writers! Undo stores old versions.
ORA-01555 = undo expired = increase UNDO_RETENTION

Locks:
  Row-level TX lock on DML (UPDATE/DELETE)
  SELECT FOR UPDATE = explicit row lock
  NOWAIT = fail fast, WAIT n = wait n seconds

Deadlock: ORA-00060, Oracle resolves automatically
  Prevention: lock in consistent order, short transactions

Isolation:
  Read Committed (default) = statement-level snapshot
  Serializable = transaction-level snapshot

Flashback: AS OF TIMESTAMP / AS OF SCN = time travel query
Autonomous Transaction: PRAGMA AUTONOMOUS_TRANSACTION = independent commit
```

---

*Next: [Chapter 5 - Advanced Oracle Features](05_oracle_advanced_features.md)*
