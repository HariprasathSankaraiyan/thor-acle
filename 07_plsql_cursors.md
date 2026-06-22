# Chapter 7: PL/SQL Cursors
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> A cursor is like a **reading finger** moving through a list of records.
> SELECT INTO gives you one row - like grabbing one card from a stack.
> A cursor lets you **go through an entire stack** of cards, one by one.
>
> Think of a **checkout conveyor belt** at a supermarket:
> - You place ALL your items on the belt (query result = result set)
> - The cashier scans ONE item at a time (FETCH = one row at a time)
> - When the belt is empty, loop ends (NO_DATA_FOUND / %NOTFOUND)

---

## 1. Implicit Cursors

> Oracle automatically creates an implicit cursor for **every SQL statement** in PL/SQL (SELECT INTO, INSERT, UPDATE, DELETE).

```sql
BEGIN
  UPDATE orders SET status = 'CLOSED'
  WHERE order_date < SYSDATE - 365;

  -- Check what the implicit cursor tells us
  IF SQL%FOUND THEN
    DBMS_OUTPUT.PUT_LINE('Updated: ' || SQL%ROWCOUNT || ' rows');
  END IF;
END;
```

### Implicit Cursor Attributes
| Attribute | Meaning |
|-----------|---------|
| `SQL%FOUND` | TRUE if last SQL affected at least 1 row |
| `SQL%NOTFOUND` | TRUE if last SQL affected 0 rows |
| `SQL%ROWCOUNT` | Number of rows affected by last SQL |
| `SQL%ISOPEN` | Always FALSE for implicit (auto-managed) |

---

## 2. Explicit Cursors - Full Syntax

> Use when you need to process **multiple rows** one at a time.

```sql
DECLARE
  -- Step 1: Declare the cursor (define the query)
  CURSOR c_open_orders IS
    SELECT order_id, customer_id, amount
    FROM orders
    WHERE status = 'OPEN';

  -- Declare variables to receive the row
  v_order_id  orders.order_id%TYPE;
  v_cust_id   orders.customer_id%TYPE;
  v_amount    orders.amount%TYPE;

BEGIN
  -- Step 2: Open the cursor (execute the query)
  OPEN c_open_orders;

  -- Step 3: Loop and FETCH rows
  LOOP
    FETCH c_open_orders INTO v_order_id, v_cust_id, v_amount;
    EXIT WHEN c_open_orders%NOTFOUND;  -- Stop when no more rows

    DBMS_OUTPUT.PUT_LINE('Order: ' || v_order_id || ', Amount: ' || v_amount);
  END LOOP;

  -- Step 4: Close the cursor
  CLOSE c_open_orders;
END;
```

### Explicit Cursor Attributes
| Attribute | Meaning |
|-----------|---------|
| `c_name%FOUND` | TRUE if last FETCH returned a row |
| `c_name%NOTFOUND` | TRUE if last FETCH returned nothing |
| `c_name%ROWCOUNT` | How many rows fetched so far |
| `c_name%ISOPEN` | TRUE if cursor is currently open |

---

## 3. Cursor FOR LOOP (Simplest & Most Preferred)

> Oracle handles open/fetch/close automatically. You just write the loop.

```sql
-- Most elegant way - Oracle manages OPEN, FETCH, CLOSE
FOR order_rec IN (SELECT order_id, customer_id, amount FROM orders WHERE status = 'OPEN')
LOOP
  DBMS_OUTPUT.PUT_LINE('Processing Order: ' || order_rec.order_id);
  -- order_rec.amount, order_rec.customer_id etc.
END LOOP;
```

### Using a named cursor in FOR LOOP
```sql
DECLARE
  CURSOR c_orders IS
    SELECT order_id, amount FROM orders WHERE status = 'OPEN';
BEGIN
  FOR order_rec IN c_orders LOOP
    DBMS_OUTPUT.PUT_LINE(order_rec.order_id || ' - ' || order_rec.amount);
  END LOOP;
END;
```

> **Interview**: Cursor FOR LOOP is preferred for readability and safety (can't forget to close). Uses `%ROWTYPE` implicitly.

---

## 4. Parameterized Cursors

> Like a coffee machine with different buttons - same machine, different output based on what you press.

```sql
DECLARE
  CURSOR c_orders(p_status VARCHAR2, p_min_amount NUMBER) IS
    SELECT order_id, amount FROM orders
    WHERE status = p_status AND amount >= p_min_amount;
BEGIN
  -- Call with different parameters
  FOR r IN c_orders('OPEN', 1000) LOOP
    DBMS_OUTPUT.PUT_LINE(r.order_id);
  END LOOP;

  FOR r IN c_orders('HOLD', 500) LOOP
    DBMS_OUTPUT.PUT_LINE(r.order_id);
  END LOOP;
END;
```

---

## 5. Cursor Variables (REF CURSOR)

> A regular cursor is like a hardcoded route. A REF CURSOR is like a GPS - the route can change at runtime.

```sql
-- Two types of REF CURSOR:
-- 1. Strong (typed) - fixed return type
TYPE t_order_cursor IS REF CURSOR RETURN orders%ROWTYPE;

-- 2. Weak (untyped) - flexible, can return any result set
TYPE t_ref_cursor IS REF CURSOR;  -- or use SYS_REFCURSOR (built-in)
```

### Using SYS_REFCURSOR (Most Common)
```sql
DECLARE
  v_cursor SYS_REFCURSOR;
  v_order_id NUMBER;
  v_amount   NUMBER;
BEGIN
  -- Open with dynamic SQL
  IF TRUE THEN
    OPEN v_cursor FOR SELECT order_id, amount FROM orders WHERE status = 'OPEN';
  ELSE
    OPEN v_cursor FOR SELECT order_id, amount FROM archived_orders;
  END IF;

  LOOP
    FETCH v_cursor INTO v_order_id, v_amount;
    EXIT WHEN v_cursor%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(v_order_id || ': ' || v_amount);
  END LOOP;
  CLOSE v_cursor;
END;
```

**Common use**: Return result sets from stored procedures to calling applications (Java/.NET uses SYS_REFCURSOR to receive query results).

---

## 6. BULK COLLECT - Process Multiple Rows at Once

> Instead of picking apples one by one from a tree (row-by-row), you shake the tree and collect ALL apples in a basket at once. Then process the basket.

### Without BULK COLLECT (slow - context switching)
```sql
FOR r IN (SELECT * FROM orders WHERE status = 'OPEN') LOOP
  -- Each iteration = context switch between PL/SQL and SQL engine
  process_order(r.order_id);
END LOOP;
-- 10,000 rows = 10,000 context switches = slow
```

### With BULK COLLECT (fast)
```sql
DECLARE
  TYPE t_order_ids IS TABLE OF orders.order_id%TYPE;
  TYPE t_amounts   IS TABLE OF orders.amount%TYPE;

  v_ids     t_order_ids;
  v_amounts t_amounts;
BEGIN
  -- Fetch ALL rows at once into collections
  SELECT order_id, amount
  BULK COLLECT INTO v_ids, v_amounts
  FROM orders
  WHERE status = 'OPEN';

  -- Process in memory
  FOR i IN 1..v_ids.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE(v_ids(i) || ': ' || v_amounts(i));
  END LOOP;
END;
```

### BULK COLLECT with LIMIT (memory management)
> For very large tables, LIMIT prevents memory exhaustion:
```sql
DECLARE
  CURSOR c_orders IS SELECT order_id, amount FROM orders WHERE status = 'OPEN';
  TYPE t_tab IS TABLE OF c_orders%ROWTYPE;
  v_batch t_tab;
BEGIN
  OPEN c_orders;
  LOOP
    FETCH c_orders BULK COLLECT INTO v_batch LIMIT 1000;  -- 1000 rows at a time
    EXIT WHEN v_batch.COUNT = 0;

    FOR i IN 1..v_batch.COUNT LOOP
      -- process v_batch(i)
      NULL;
    END LOOP;
  END LOOP;
  CLOSE c_orders;
END;
```

---

## 7. FORALL - Bulk DML

> Instead of INSERT/UPDATE one row at a time in a loop, send them ALL in one batch.

```sql
DECLARE
  TYPE t_ids IS TABLE OF NUMBER;
  v_ids t_ids := t_ids(1001, 1002, 1003, 1004);
BEGIN
  -- Update ALL rows in one trip to SQL engine
  FORALL i IN 1..v_ids.COUNT
    UPDATE orders SET status = 'PROCESSED' WHERE order_id = v_ids(i);

  DBMS_OUTPUT.PUT_LINE('Updated: ' || SQL%ROWCOUNT);
  COMMIT;
END;
```

### BULK COLLECT + FORALL (Best Practice for ETL)
```sql
DECLARE
  TYPE t_tab IS TABLE OF source_orders%ROWTYPE;
  v_data t_tab;
BEGIN
  -- Step 1: Bulk collect source data
  SELECT * BULK COLLECT INTO v_data FROM source_orders WHERE processed = 'N';

  -- Step 2: Bulk insert into target
  FORALL i IN 1..v_data.COUNT
    INSERT INTO target_orders VALUES v_data(i);

  COMMIT;
END;
```

> **Performance**: FORALL is 10-100x faster than row-by-row DML in a loop.

---

## 8. Cursor Exceptions

```sql
DECLARE
  CURSOR c IS SELECT * FROM orders WHERE order_id = 9999999;
  v_rec orders%ROWTYPE;
BEGIN
  OPEN c;
  FETCH c INTO v_rec;
  IF c%NOTFOUND THEN
    DBMS_OUTPUT.PUT_LINE('No rows found');
  END IF;
  CLOSE c;

EXCEPTION
  WHEN CURSOR_ALREADY_OPEN THEN
    DBMS_OUTPUT.PUT_LINE('Cursor already open!');
  WHEN INVALID_CURSOR THEN
    DBMS_OUTPUT.PUT_LINE('Invalid cursor operation!');
END;
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between implicit and explicit cursors?**
> Implicit cursors are automatically created by Oracle for every SQL statement (SELECT INTO, UPDATE, etc.) - managed by Oracle. Explicit cursors are manually declared for multi-row queries - you control OPEN, FETCH, CLOSE.

**Q2: What is a Cursor FOR LOOP and why is it preferred?**
> A cursor FOR LOOP automatically handles OPEN, FETCH per iteration, and CLOSE. It's the most readable and least error-prone form. Oracle uses bulk read optimization internally.

**Q3: What is BULK COLLECT and when do you use it?**
> BULK COLLECT fetches multiple rows into a PL/SQL collection in one SQL-to-PL/SQL context switch. Use when processing large result sets row-by-row is too slow. Use LIMIT clause to control memory.

**Q4: What is FORALL?**
> FORALL performs DML (INSERT/UPDATE/DELETE) on entire collection in one batch operation. Dramatically faster than DML inside a FOR loop. Paired with BULK COLLECT for ETL performance.

**Q5: What is a REF CURSOR / SYS_REFCURSOR?**
> A cursor variable that can be assigned different queries at runtime. SYS_REFCURSOR is the built-in weak type. Commonly used to return result sets from stored procedures to Java/.NET applications.

**Q6: How do you handle large result sets without running out of memory?**
> Use BULK COLLECT with LIMIT - fetch in chunks (e.g., 1000 rows at a time). Process each chunk and loop until no more rows.

---

## 📝 Quick Revision Summary

```
Implicit cursor: auto for every SQL (SQL%FOUND, SQL%ROWCOUNT)

Explicit cursor: manual, for multi-row
  OPEN -> FETCH -> loop with EXIT WHEN %NOTFOUND -> CLOSE

Cursor FOR LOOP: simplest, automatic open/fetch/close
  FOR rec IN cursor_name LOOP ... END LOOP;

Parameterized cursor: CURSOR c(param) IS SELECT ... WHERE col = param;

REF CURSOR: flexible, runtime query. SYS_REFCURSOR = built-in type.
  Used to return result sets from procedures to apps.

BULK COLLECT: fetch many rows at once (into collection)
  WITH LIMIT 1000: process in chunks for huge tables

FORALL: bulk DML - one trip to SQL engine
  FORALL i IN 1..v_ids.COUNT UPDATE ... WHERE id = v_ids(i)

BULK COLLECT + FORALL = maximum ETL performance pattern
```

---

*Next: [Chapter 8 - Procedures, Functions & Packages](08_plsql_procedures_functions.md)*
