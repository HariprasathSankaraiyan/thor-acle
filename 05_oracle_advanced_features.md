# Chapter 5: Advanced Oracle Features
## ⏱️ ~20 min read

---

## 🧠 Topics Covered
Materialized Views, Sequences, Synonyms, DBLinks, Temporary Tables, Constraints, Tablespaces, and Scheduler.

---

## 1. Materialized Views (MVIEWs)

> A regular view is like a window - you look through it and see live data every time.
> A Materialized View is like a **photograph** - pre-computed snapshot of data stored physically.
> The photo is fast to look at but may be a bit old.

### Regular View vs Materialized View
| | View | Materialized View |
|--|------|------------------|
| Stores data? | No (live query) | Yes (physical snapshot) |
| Query speed | Depends on base tables | Fast (pre-computed) |
| Refresh needed? | Never (always live) | Yes (scheduled or on-demand) |
| Use case | Security, abstraction | Reporting, summaries, DW |

```sql
-- Create Materialized View refreshed daily
CREATE MATERIALIZED VIEW mv_order_summary
REFRESH COMPLETE START WITH SYSDATE NEXT SYSDATE + 1
AS
SELECT customer_id, COUNT(*) order_count, SUM(amount) total_amount
FROM orders
GROUP BY customer_id;

-- Manual refresh
EXEC DBMS_MVIEW.REFRESH('MV_ORDER_SUMMARY');

-- Query it like a table
SELECT * FROM mv_order_summary WHERE customer_id = 100;
```

### Refresh Types
| Type | What it does |
|------|-------------|
| **COMPLETE** | Truncate and rebuild from scratch |
| **FAST** | Only apply changes since last refresh (needs mview log) |
| **FORCE** | Fast if possible, else Complete |

```sql
-- Mview log needed for FAST refresh
CREATE MATERIALIZED VIEW LOG ON orders
WITH ROWID, SEQUENCE (customer_id, amount) INCLUDING NEW VALUES;
```

### Query Rewrite
Oracle can **automatically use** the MVIEW instead of base tables if the query matches:
```sql
-- You write this:
SELECT customer_id, SUM(amount) FROM orders GROUP BY customer_id;
-- Oracle internally rewrites to use mv_order_summary (faster!)
```

---

## 2. Sequences

> A sequence is like a **ticket dispenser** at a deli counter. Pull a number, get the next unique number.

```sql
-- Create
CREATE SEQUENCE seq_order_id
  START WITH 1000
  INCREMENT BY 1
  NOCACHE        -- Don't pre-allocate (safe but slower)
  NOCYCLE;       -- Don't restart when max reached

-- Use in INSERT
INSERT INTO orders (order_id, ...) VALUES (seq_order_id.NEXTVAL, ...);

-- Peek at current without incrementing
SELECT seq_order_id.CURRVAL FROM DUAL;

-- See next value (increments the sequence)
SELECT seq_order_id.NEXTVAL FROM DUAL;
```

> **Important**: Sequence gaps can occur (if a transaction rollbacks, the number is gone). Sequences are NOT transactional - NEXTVAL is permanent even on rollback.

---

## 3. Synonyms

> A synonym is a **nickname** for a database object. Like calling your friend "Mike" instead of "Michael David Johnson".

```sql
-- Create synonym (so you don't need to type SCHEMA.TABLE)
CREATE SYNONYM orders FOR scott.orders;

-- Private synonym (only for you)
CREATE SYNONYM my_orders FOR scott.orders;

-- Public synonym (for everyone in the DB)
CREATE PUBLIC SYNONYM orders FOR scott.orders;

-- Drop synonym
DROP SYNONYM orders;
```

**Use case**: Applications use synonym - when you move a table to a different schema, just update the synonym. Application code doesn't change.

---

## 4. Database Links (DBLink)

> A DBLink is like a **wormhole** between two databases. You can query a table in another database as if it's local.

```sql
-- Create link to remote database
CREATE DATABASE LINK remote_prod
CONNECT TO username IDENTIFIED BY password
USING 'REMOTE_DB_TNSNAME';

-- Query remote table
SELECT * FROM orders@remote_prod WHERE status = 'OPEN';

-- Insert from remote
INSERT INTO orders SELECT * FROM orders@remote_prod WHERE order_date > SYSDATE - 30;

-- Drop link
DROP DATABASE LINK remote_prod;
```

> Security note: Avoid storing passwords in DBLinks - use Oracle Wallets.

---

## 5. Global Temporary Tables (GTT)

> A whiteboard that gets automatically erased - either at COMMIT or when the session ends.

```sql
-- Data deleted at COMMIT
CREATE GLOBAL TEMPORARY TABLE temp_orders (
  order_id NUMBER,
  amount NUMBER
) ON COMMIT DELETE ROWS;

-- Data deleted at session end
CREATE GLOBAL TEMPORARY TABLE temp_session (
  order_id NUMBER
) ON COMMIT PRESERVE ROWS;
```

**Characteristics**:
- Each session sees ONLY its own data
- No redo logging (fast)
- No undo logging (fast)
- Structure is permanent, data is temporary
- Used in: ETL staging, complex batch processing, temp calculations

---

## 6. Constraints

| Constraint | Purpose | Example |
|-----------|---------|---------|
| **PRIMARY KEY** | Unique + Not Null identifier | `order_id NUMBER PRIMARY KEY` |
| **UNIQUE** | No duplicates, NULLs allowed | `UNIQUE (email)` |
| **NOT NULL** | Value required | `status VARCHAR2(20) NOT NULL` |
| **FOREIGN KEY** | Referential integrity | `REFERENCES customers(customer_id)` |
| **CHECK** | Custom rule | `CHECK (amount > 0)` |

```sql
-- Disable constraint (for bulk loads)
ALTER TABLE orders DISABLE CONSTRAINT fk_customer_id;

-- Enable (validates all existing rows)
ALTER TABLE orders ENABLE CONSTRAINT fk_customer_id;

-- Enable NOVALIDATE (only check new rows, don't validate existing)
ALTER TABLE orders ENABLE NOVALIDATE CONSTRAINT fk_customer_id;

-- Deferrable constraints (check at COMMIT not at statement)
ALTER TABLE orders ADD CONSTRAINT fk_cust
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
  DEFERRABLE INITIALLY DEFERRED;
```

---

## 7. Tablespaces

> A tablespace is a **storage container** in Oracle - like a folder that holds data files. You assign tables/indexes to tablespaces.

```sql
-- Common tablespaces
SYSTEM     -- Oracle core data dictionary
SYSAUX     -- Auxiliary to SYSTEM (AWR, XML DB, etc.)
USERS      -- Default for user objects
TEMP       -- Temporary sort/hash join space
UNDO       -- Undo segments
```

```sql
-- Create tablespace
CREATE TABLESPACE app_data
DATAFILE '/u01/oradata/app_data01.dbf' SIZE 1G AUTOEXTEND ON NEXT 256M;

-- Create table in specific tablespace
CREATE TABLE orders (...) TABLESPACE app_data;

-- Check tablespace usage
SELECT tablespace_name,
  ROUND((1 - free_space/total_space) * 100, 1) pct_used
FROM dba_tablespace_usage_metrics;
```

---

## 8. Oracle Scheduler (DBMS_SCHEDULER)

> Like setting an alarm on your phone - "run this task every night at 2 AM".

```sql
-- Create a job
BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name        => 'REFRESH_SUMMARIES',
    job_type        => 'STORED_PROCEDURE',
    job_action      => 'pkg_reports.refresh_all',
    start_date      => SYSTIMESTAMP,
    repeat_interval => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=0',
    enabled         => TRUE,
    comments        => 'Nightly summary refresh'
  );
END;
/

-- Check job runs
SELECT job_name, status, actual_start_date, run_duration
FROM dba_scheduler_job_run_details
ORDER BY actual_start_date DESC;

-- Disable/Enable
EXEC DBMS_SCHEDULER.DISABLE('REFRESH_SUMMARIES');
EXEC DBMS_SCHEDULER.ENABLE('REFRESH_SUMMARIES');

-- Run immediately
EXEC DBMS_SCHEDULER.RUN_JOB('REFRESH_SUMMARIES');
```

---

## 9. Oracle AWR & ADDM (Performance Tools)

> AWR is like a **health report** Oracle automatically generates for your database. ADDM reads it and says "Hey, you have 3 problems and here's how to fix them."

### AWR - Automatic Workload Repository
- Oracle takes snapshots of performance data every hour (default)
- Stores in SYSAUX tablespace
- `DBA_HIST_*` views

```sql
-- Generate AWR report (from sqlplus)
@?/rdbms/admin/awrrpt.sql

-- Manual snapshot
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
```

### ADDM - Automatic Database Diagnostic Monitor
- Runs automatically after each AWR snapshot
- Identifies top problems and recommends fixes
- Example findings: "Top SQL consuming 45% of DB time - consider adding index"

### ASH - Active Session History
- Samples active sessions every second
- Great for diagnosing what was happening during a specific time window
```sql
SELECT * FROM v$active_session_history WHERE sample_time > SYSDATE - 1/24;
```

---

## 10. Important Oracle Parameters to Know

| Parameter | Purpose | Typical Value |
|-----------|---------|--------------|
| `SGA_TARGET` | Auto-tune SGA size | 8G |
| `PGA_AGGREGATE_TARGET` | Total PGA limit | 2G |
| `UNDO_RETENTION` | Keep undo for N seconds | 900 |
| `DB_RECOVERY_FILE_DEST` | FRA location | /u01/fra |
| `PROCESSES` | Max OS processes | 300 |
| `SESSIONS` | Max sessions | ~1.1 × PROCESSES |
| `AUDIT_TRAIL` | Enable auditing | DB or OS |
| `LOG_ARCHIVE_DEST_1` | Archive log destination | /u01/arch |

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between a view and a materialized view?**
> A view is a stored query - it runs against base tables every time you query it. A materialized view stores the result physically. It's faster to query but needs refreshing to stay current.

**Q2: When would you use a materialized view?**
> For complex aggregations queried frequently in reporting. Also when base tables are slow or in a remote database. Query rewrite can transparently use MVIEWs.

**Q3: What is a Global Temporary Table?**
> A table where structure is permanent but data is session-private and automatically deleted at commit or session end. No redo/undo overhead. Used for staging data in complex procedures.

**Q4: What happens to a sequence on ROLLBACK?**
> The sequence number is consumed permanently - it doesn't roll back. This is intentional to avoid contention. Gaps in sequences are expected and normal.

**Q5: What is ENABLE NOVALIDATE?**
> Enables a constraint so new rows must comply, but doesn't validate existing rows. Used after bulk loading historical data that may have violations.

**Q6: What is AWR used for?**
> AWR captures snapshots of performance statistics. Used to generate reports showing top SQL, wait events, resource usage between two time periods. Basis for ADDM recommendations.

---

## 📝 Quick Revision Summary

```
Materialized View:
  Physical snapshot (like a photo)
  Refresh: COMPLETE (rebuild), FAST (incremental), FORCE (auto)
  Query Rewrite: Oracle uses MVIEW transparently

Sequence: ticket dispenser. NEXTVAL never rolls back. Gaps are normal.

Synonym: alias/nickname for object. UPDATE synonym = no app changes.

DBLink: query remote DB. CREATE DATABASE LINK … USING 'TNS_NAME'

GTT: session-private temp table. ON COMMIT DELETE ROWS or PRESERVE ROWS.

Constraints: PK, UK, FK, CHECK, NOT NULL
  ENABLE NOVALIDATE = skip existing rows
  DEFERRABLE = check at COMMIT

Scheduler: DBMS_SCHEDULER.CREATE_JOB with repeat_interval = 'FREQ=DAILY'

AWR = performance snapshots (hourly)
ADDM = reads AWR, gives recommendations
ASH = per-second active session samples
```

---

*Next: [Chapter 6 - PL/SQL Basics](06_plsql_basics.md)*
