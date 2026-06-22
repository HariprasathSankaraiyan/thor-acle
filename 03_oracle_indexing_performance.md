# Chapter 3: Oracle Indexing & Performance Tuning
## ⏱️ ~25 min read

---

## 🧠 Think of it this way

> An index is like the **index at the back of a textbook**.
> Without it: you read every page to find "Oracle" (Full Table Scan = slow).
> With it: you go to the index, find page 247, jump directly there (fast).
> 
> But: if you need 90% of the pages anyway, using the index is SLOWER than just reading the whole book.

---

## 1. Types of Indexes in Oracle

### 1a. B-Tree Index (Default - Most Common)
- **Structure**: Balanced tree - root -> branch -> leaf nodes
- **Best for**: High cardinality columns (many unique values like order_id, email)
- **Operations supported**: `=`, `<`, `>`, `BETWEEN`, `LIKE 'abc%'`

```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

> Like a sorted phone book. Works great when you need one specific entry.

### 1b. Bitmap Index
- **Structure**: Bit array per distinct value
- **Best for**: Low cardinality columns (status='OPEN'/'CLOSED', gender='M'/'F')
- **Great for**: Data warehouses, reporting, WHERE with multiple columns
- **Bad for**: OLTP (high insert/update = bitmap lock contention)

```sql
CREATE BITMAP INDEX idx_orders_status ON orders(status);
```

> Like a true/false checklist - "which orders are OPEN?" is just scanning a bit array.

### 1c. Function-Based Index
- Index on the RESULT of a function
- Use when your WHERE clause uses a function on a column

```sql
-- Without this index, UPPER() prevents index use
CREATE INDEX idx_upper_name ON customers(UPPER(customer_name));

-- Now this query can use the index
SELECT * FROM customers WHERE UPPER(customer_name) = 'JOHN SMITH';
```

### 1d. Composite Index
- Index on **multiple columns**
- Column order matters - leading column must be in WHERE for index to be used

```sql
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date);
-- Can use index for: WHERE customer_id = 100
-- Can use index for: WHERE customer_id = 100 AND order_date > ...
-- CANNOT use index for: WHERE order_date > ...  (skipping leading column)
```

> Like a phone book sorted by Last Name, then First Name. You can find "Smith, John" but you can't find "John" without knowing the last name.

### 1e. Unique Index
```sql
CREATE UNIQUE INDEX idx_email_unique ON customers(email);
```

### 1f. Partial / Invisible Index
```sql
-- Invisible index: optimizer ignores it (for testing)
CREATE INDEX idx_test ON orders(amount) INVISIBLE;
-- Make visible again:
ALTER INDEX idx_test VISIBLE;
```

---

## 2. When Oracle USES vs IGNORES an Index

### Oracle USES an index when:
- Equality: `WHERE order_id = 1001` ✅
- Range on leading column: `WHERE order_date BETWEEN ... AND ...` ✅
- `LIKE 'abc%'` (prefix match) ✅

### Oracle IGNORES (skips) an index when:
- Function on indexed column: `WHERE UPPER(name) = 'JOHN'` ❌ (use function-based index)
- NOT EQUAL: `WHERE status != 'OPEN'` ❌
- `LIKE '%abc'` (leading wildcard) ❌
- NULL check: `WHERE col IS NULL` ❌ (B-tree doesn't store NULLs)
- Implicit conversion: `WHERE order_id = '1001'` (number compared to varchar) ❌
- High-selectivity on bitmap: retrieving >10% of rows ❌

---

## 3. EXPLAIN PLAN - Reading Execution Plans

> Explain plan shows you Oracle's "plan of attack" before actually running the query.

```sql
EXPLAIN PLAN FOR
SELECT * FROM orders WHERE customer_id = 100;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Key Operations to Know

| Operation | Meaning | Good/Bad |
|-----------|---------|----------|
| **TABLE ACCESS FULL** | Full table scan - reads every row | Bad for large tables |
| **INDEX RANGE SCAN** | Uses index, fetches range of rows | Good |
| **INDEX UNIQUE SCAN** | Uses index, fetches exactly 1 row | Best |
| **NESTED LOOPS** | For each row in outer, probe inner | Good for small sets |
| **HASH JOIN** | Hash outer, probe with inner | Good for large sets |
| **SORT MERGE JOIN** | Sort both tables, merge | Good when data is pre-sorted |
| **SORT (ORDER BY)** | Sorting operation | Watch cost |
| **BUFFER SORT** | Sort in memory | OK |

### Reading the Plan (Indentation Rule)
```
Most indented = runs first (inner child)
Read bottom-up for execution order
```

---

## 4. AUTOTRACE - Quick Performance Check

```sql
SET AUTOTRACE ON
SELECT * FROM orders WHERE customer_id = 100;
SET AUTOTRACE OFF
```
Shows execution plan + statistics (logical reads, physical reads).

---

## 5. Key Performance Metrics to Understand

### Logical Reads vs Physical Reads
- **Logical reads**: Data found in Buffer Cache (fast)
- **Physical reads**: Data read from disk (slow)
- Goal: minimize physical reads -> warm up cache or create proper indexes

### High Water Mark (HWM)
> The highest point water ever reached in a tank - Oracle won't scan beyond HWM but won't automatically reduce it.

- After DELETE, HWM doesn't move down -> Oracle still scans "empty" space
- TRUNCATE resets HWM -> smaller scan
- To shrink: `ALTER TABLE orders SHRINK SPACE;`

---

## 6. Statistics & the Cost-Based Optimizer (CBO)

> Oracle's query optimizer is like a GPS - it uses statistics (road conditions = table data distribution) to pick the best route (execution plan).

```sql
-- Gather statistics on a table
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA_NAME', 'ORDERS');

-- Gather stats on whole schema
EXEC DBMS_STATS.GATHER_SCHEMA_STATS('SCHEMA_NAME');
```

**If statistics are stale**: CBO makes bad decisions -> slow queries.

**Key statistics stored**:
- Number of rows
- Number of distinct values per column (NDV)
- Column min/max
- Number of NULLs
- Data distribution (histograms)

### Histograms
> When data is skewed (e.g., 90% of orders have status='CLOSED'), a histogram tells Oracle about the skew so it picks a better plan.

---

## 7. Hints - Force Oracle's Behavior

> Hints are like telling your GPS "ignore the fastest route, take the highway."

```sql
-- Force full table scan
SELECT /*+ FULL(o) */ * FROM orders o WHERE customer_id = 100;

-- Force index use
SELECT /*+ INDEX(o idx_orders_customer) */ * FROM orders o WHERE customer_id = 100;

-- Force a join method
SELECT /*+ USE_NL(o c) */ * FROM orders o JOIN customers c ON o.customer_id = c.customer_id;
SELECT /*+ USE_HASH(o c) */ * ...
SELECT /*+ USE_MERGE(o c) */ * ...

-- Parallel query
SELECT /*+ PARALLEL(o, 4) */ * FROM orders o;
```

> **Interview**: Hints override the optimizer's decision. Use when CBO makes wrong choices due to stale stats or complex queries.

---

## 8. Partitioning (Performance + Manageability)

> Instead of one giant table, partition it like a filing cabinet with monthly drawers. A query for "January 2023" only opens the January drawer.

### Types of Partitioning

| Type | Based On | Example |
|------|---------|---------|
| **Range** | Range of values | Date ranges - Jan, Feb, Mar |
| **List** | Discrete values | Country = 'US', 'UK', 'IN' |
| **Hash** | Hash function | Distributes evenly by key |
| **Composite** | Two types combined | Range-List, Range-Hash |

```sql
CREATE TABLE orders (
  order_id NUMBER,
  order_date DATE,
  status VARCHAR2(20)
)
PARTITION BY RANGE (order_date) (
  PARTITION p_2023q1 VALUES LESS THAN (DATE '2023-04-01'),
  PARTITION p_2023q2 VALUES LESS THAN (DATE '2023-07-01'),
  PARTITION p_2023q3 VALUES LESS THAN (DATE '2023-10-01'),
  PARTITION p_2023q4 VALUES LESS THAN (DATE '2024-01-01')
);
```

**Partition Pruning**: Oracle only scans relevant partitions.
```sql
-- Only scans p_2023q1 partition
SELECT * FROM orders WHERE order_date BETWEEN '01-JAN-2023' AND '31-MAR-2023';
```

---

## 9. Common Performance Problems & Solutions

| Problem | Symptom | Fix |
|---------|---------|-----|
| Full table scan on large table | Slow query | Add/rebuild index |
| Stale statistics | Bad execution plan | Gather stats |
| Too many hard parses | High CPU, shared pool contention | Use bind variables |
| Lock contention | Sessions waiting | Check `V$LOCK`, `V$SESSION` |
| Temp space issues | ORA-01652 | Increase temp tablespace |
| Index not used | Function on column | Create function-based index |

### Bind Variables vs Literals
```sql
-- BAD: hard parse for every order_id (1000 different SQLs)
SELECT * FROM orders WHERE order_id = 1001;
SELECT * FROM orders WHERE order_id = 1002;

-- GOOD: parsed once, reused (1 SQL in library cache)
SELECT * FROM orders WHERE order_id = :order_id;
```

---

## 10. V$ Views for Performance Monitoring

```sql
-- Find top SQL by logical reads
SELECT sql_text, executions, buffer_gets/executions avg_lio
FROM v$sql
ORDER BY buffer_gets DESC;

-- Find currently running sessions
SELECT sid, serial#, username, status, sql_id
FROM v$session WHERE status = 'ACTIVE';

-- Find long running operations
SELECT * FROM v$session_longops WHERE time_remaining > 0;

-- Find locking sessions
SELECT l.sid, s.username, l.type, l.mode_held
FROM v$lock l JOIN v$session s ON l.sid = s.sid
WHERE l.block > 0;
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between B-tree and Bitmap index?**
> B-tree is for high cardinality columns (unique IDs, emails). Bitmap is for low cardinality (status, gender). Bitmap is great for warehouse/reporting but bad for OLTP due to lock contention on updates.

**Q2: Why would Oracle not use an index you've created?**
> Functions on the column, leading wildcard LIKE '%abc', implicit type conversion, retrieving a large % of rows (full scan is cheaper), stale statistics misleading CBO, or NULL checks.

**Q3: What is a Full Table Scan and when is it OK?**
> Oracle reads every block in the table. It's fine for small tables or when you need most rows. It's bad for large tables when you only want a few rows.

**Q4: What are bind variables and why do they matter?**
> Bind variables allow SQL to be parsed once and reused with different values. Literals cause a hard parse for every unique value, wasting CPU and shared pool memory.

**Q5: What is partition pruning?**
> When a query's WHERE clause includes the partition key, Oracle eliminates irrelevant partitions and only scans matching ones. This dramatically reduces I/O.

**Q6: How do you investigate a slow query?**
> 1) Get the execution plan (EXPLAIN PLAN). 2) Check for full table scans. 3) Verify statistics are current. 4) Check for missing indexes. 5) Look for function on indexed columns. 6) Check V$SQL for buffer_gets.

---

## 📝 Quick Revision Summary

```
Index Types:
  B-Tree    -> high cardinality (order_id, email)
  Bitmap    -> low cardinality (status, gender) - DW only
  FBI       -> function-based (UPPER(name))
  Composite -> multi-column, leading column matters

Index NOT used when:
  Function on column, LIKE '%abc', IS NULL, type mismatch

Execution Plan:
  TABLE ACCESS FULL -> bad for large tables
  INDEX RANGE SCAN  -> good
  INDEX UNIQUE SCAN -> best

Partition Pruning = only scan relevant partitions
Bind Variables = parse once, reuse many times
Stale Stats = gather stats with DBMS_STATS
```

---

*Next: [Chapter 4 - Transactions & Locks](04_oracle_transactions_locks.md)*
