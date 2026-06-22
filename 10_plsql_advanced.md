# Chapter 10: Advanced PL/SQL - Dynamic SQL, Collections & Pipelining
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> **Dynamic SQL** = Writing a new recipe on a whiteboard at runtime instead of using a pre-printed recipe card.
>
> **Collections** = Shopping list in your pocket - a variable-length list of items you can loop through.
>
> **Pipelining** = Assembly line in a factory - data flows out as it's produced, you don't wait for the whole batch.

---

## 1. Dynamic SQL - EXECUTE IMMEDIATE

> **When to use**: Table name, column name, or SQL structure unknown until runtime.

```sql
-- Static SQL (compile-time known)
SELECT * FROM orders WHERE status = 'OPEN';  -- Fine

-- Dynamic SQL (table name unknown at compile time)
DECLARE
  v_table_name VARCHAR2(30) := 'ORDERS';
  v_sql        VARCHAR2(500);
  v_count      NUMBER;
BEGIN
  -- Build SQL as string
  v_sql := 'SELECT COUNT(*) FROM ' || v_table_name;

  -- Execute and get result
  EXECUTE IMMEDIATE v_sql INTO v_count;
  DBMS_OUTPUT.PUT_LINE('Count: ' || v_count);
END;
```

### Dynamic SQL with Bind Variables (IMPORTANT - Prevents SQL Injection)
```sql
DECLARE
  v_sql    VARCHAR2(500);
  v_status VARCHAR2(20) := 'OPEN';
  v_count  NUMBER;
BEGIN
  -- Using bind variable :1 (not string concatenation!)
  v_sql := 'SELECT COUNT(*) FROM orders WHERE status = :1';
  EXECUTE IMMEDIATE v_sql INTO v_count USING v_status;
  -- USING clause passes bind variable value
END;
```

### Dynamic DML
```sql
DECLARE
  v_sql    VARCHAR2(500);
  v_rows   NUMBER;
BEGIN
  v_sql := 'UPDATE ' || v_table || ' SET status = :1 WHERE id = :2';
  EXECUTE IMMEDIATE v_sql USING 'CLOSED', 1001;
  v_rows := SQL%ROWCOUNT;
  DBMS_OUTPUT.PUT_LINE('Updated: ' || v_rows || ' rows');
END;
```

### Dynamic DDL
```sql
-- DDL doesn't support bind variables
EXECUTE IMMEDIATE 'CREATE TABLE temp_' || v_suffix || ' (id NUMBER, name VARCHAR2(100))';
EXECUTE IMMEDIATE 'DROP TABLE ' || v_table_name || ' PURGE';
EXECUTE IMMEDIATE 'ALTER TABLE orders ADD (new_col VARCHAR2(50))';
```

### Dynamic SQL with REF CURSOR
```sql
DECLARE
  v_cursor SYS_REFCURSOR;
  v_sql    VARCHAR2(500);
  v_id     NUMBER;
BEGIN
  v_sql := 'SELECT order_id FROM orders WHERE status = :1';
  OPEN v_cursor FOR v_sql USING 'OPEN';
  LOOP
    FETCH v_cursor INTO v_id;
    EXIT WHEN v_cursor%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(v_id);
  END LOOP;
  CLOSE v_cursor;
END;
```

---

## 2. Collections - Three Types

### 2a. Associative Array (INDEX BY TABLE) - Most Flexible
```sql
DECLARE
  -- Key-value map. Key = NUMBER or VARCHAR2
  TYPE t_name_map IS TABLE OF VARCHAR2(100) INDEX BY NUMBER;
  v_names t_name_map;
BEGIN
  v_names(1)   := 'Alice';
  v_names(100) := 'Bob';      -- Keys don't need to be sequential
  v_names(500) := 'Charlie';

  -- Navigate
  DBMS_OUTPUT.PUT_LINE(v_names(100));        -- 'Bob'
  DBMS_OUTPUT.PUT_LINE(v_names.COUNT);       -- 3
  DBMS_OUTPUT.PUT_LINE(v_names.FIRST);       -- 1
  DBMS_OUTPUT.PUT_LINE(v_names.LAST);        -- 500

  -- Loop with FIRST..LAST (non-sequential safe)
  DECLARE v_key NUMBER := v_names.FIRST;
  BEGIN
    WHILE v_key IS NOT NULL LOOP
      DBMS_OUTPUT.PUT_LINE(v_key || ': ' || v_names(v_key));
      v_key := v_names.NEXT(v_key);
    END LOOP;
  END;
END;
```

> **Cannot store in DB** (PL/SQL scope only). Best for in-memory lookup tables.

### 2b. Nested Table - Like a Dynamic Array
```sql
DECLARE
  TYPE t_number_list IS TABLE OF NUMBER;
  v_nums t_number_list := t_number_list();  -- Initialize empty
BEGIN
  v_nums.EXTEND(3);      -- Add 3 slots
  v_nums(1) := 10;
  v_nums(2) := 20;
  v_nums(3) := 30;

  v_nums.EXTEND;         -- Add 1 more slot
  v_nums(4) := 40;

  -- Delete element 2
  v_nums.DELETE(2);      -- Creates "hole" at position 2

  DBMS_OUTPUT.PUT_LINE('Count: ' || v_nums.COUNT);  -- 3 (deleted not counted)
  DBMS_OUTPUT.PUT_LINE('Limit: ' || NVL(TO_CHAR(v_nums.LIMIT), 'Unlimited'));

  FOR i IN 1..v_nums.LAST LOOP
    IF v_nums.EXISTS(i) THEN  -- Check before accessing (holes!)
      DBMS_OUTPUT.PUT_LINE(v_nums(i));
    END IF;
  END LOOP;
END;
```

> **Can be stored in DB** as column type. Supports SET operations (MULTISET UNION, INTERSECT, etc).

### 2c. VARRAY - Fixed Maximum Size
```sql
-- Maximum 5 elements, ordered, no holes
TYPE t_top5 IS VARRAY(5) OF VARCHAR2(50);
v_winners t_top5 := t_top5('Gold','Silver','Bronze');

-- Can store in DB as column
CREATE TABLE race_results (
  race_id NUMBER,
  top5 t_top5
);
```

---

## 3. Collection Methods

| Method | What it does |
|--------|-------------|
| `.COUNT` | Number of elements |
| `.FIRST` | Index of first element |
| `.LAST` | Index of last element |
| `.NEXT(n)` | Index after n |
| `.PRIOR(n)` | Index before n |
| `.EXISTS(n)` | TRUE if element at n exists |
| `.EXTEND` | Add 1 element |
| `.EXTEND(n)` | Add n elements |
| `.TRIM` | Remove last element |
| `.DELETE` | Remove all elements |
| `.DELETE(n)` | Remove element at n |
| `.LIMIT` | Max size (VARRAY only) |

---

## 4. Table Functions & Pipelined Functions

> A factory assembly line. Items come out as they're made, not all at the end.

### Regular Function (returns all data at end)
```sql
-- Problem: generates all rows first, then returns
-- Memory-heavy for large datasets
FUNCTION get_orders RETURN t_order_tab AS ...
```

### Pipelined Function (streams data row by row)
```sql
CREATE TYPE t_order_row AS OBJECT (
  order_id NUMBER,
  amount   NUMBER,
  status   VARCHAR2(20)
);
CREATE TYPE t_order_tab AS TABLE OF t_order_row;

-- Pipelined function
CREATE OR REPLACE FUNCTION stream_open_orders
RETURN t_order_tab PIPELINED AS
BEGIN
  FOR r IN (SELECT order_id, amount, status FROM orders WHERE status = 'OPEN') LOOP
    PIPE ROW(t_order_row(r.order_id, r.amount, r.status));  -- Stream one row
  END LOOP;
  RETURN;  -- No data returned here - all via PIPE ROW
END;
/

-- Query it like a table!
SELECT * FROM TABLE(stream_open_orders());
SELECT * FROM TABLE(stream_open_orders()) WHERE amount > 1000;
```

---

## 5. DBMS_SQL - Advanced Dynamic SQL

> Use when column count/types are unknown at runtime.

```sql
DECLARE
  v_cursor INTEGER;
  v_status INTEGER;
  v_col1   VARCHAR2(200);
BEGIN
  v_cursor := DBMS_SQL.OPEN_CURSOR;
  DBMS_SQL.PARSE(v_cursor, 'SELECT order_id FROM orders WHERE rownum <= 5',
                 DBMS_SQL.NATIVE);
  DBMS_SQL.DEFINE_COLUMN(v_cursor, 1, v_col1, 200);
  v_status := DBMS_SQL.EXECUTE(v_cursor);

  WHILE DBMS_SQL.FETCH_ROWS(v_cursor) > 0 LOOP
    DBMS_SQL.COLUMN_VALUE(v_cursor, 1, v_col1);
    DBMS_OUTPUT.PUT_LINE(v_col1);
  END LOOP;

  DBMS_SQL.CLOSE_CURSOR(v_cursor);
END;
```

> Use `EXECUTE IMMEDIATE` for 95% of dynamic SQL. Use `DBMS_SQL` only when column structure is truly dynamic.

---

## 6. Object Types in Oracle

```sql
-- Define an object type
CREATE OR REPLACE TYPE t_address AS OBJECT (
  street    VARCHAR2(100),
  city      VARCHAR2(50),
  zip_code  VARCHAR2(10),

  -- Member function
  MEMBER FUNCTION format_address RETURN VARCHAR2
);
/

CREATE OR REPLACE TYPE BODY t_address AS
  MEMBER FUNCTION format_address RETURN VARCHAR2 AS
  BEGIN
    RETURN street || ', ' || city || ' ' || zip_code;
  END;
END;
/

-- Use it
DECLARE
  v_addr t_address := t_address('123 Main St', 'Chicago', '60601');
BEGIN
  DBMS_OUTPUT.PUT_LINE(v_addr.format_address());
END;
```

---

## 7. Conditional Compilation

```sql
-- Compile different code for different environments
$IF $$DEBUG_MODE $THEN
  DBMS_OUTPUT.PUT_LINE('Debug: ' || v_value);
$END

-- Set the flag at session level
EXECUTE IMMEDIATE 'ALTER SESSION SET PLSQL_CCFLAGS = ''DEBUG_MODE:TRUE''';
```

---

## 8. SQL Injection Prevention

```sql
-- VULNERABLE (never do this!)
v_sql := 'SELECT * FROM users WHERE name = ''' || v_input || '''';
-- Attacker inputs: ' OR '1'='1 -> returns all rows

-- SAFE: Use bind variables
v_sql := 'SELECT * FROM users WHERE name = :1';
EXECUTE IMMEDIATE v_sql INTO v_result USING v_input;

-- Also safe: DBMS_ASSERT for object names
v_safe_table := DBMS_ASSERT.SIMPLE_SQL_NAME(v_table_input);
v_sql := 'SELECT * FROM ' || v_safe_table;
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is dynamic SQL? When do you use EXECUTE IMMEDIATE?**
> Dynamic SQL builds and executes SQL strings at runtime. Use EXECUTE IMMEDIATE when table/column names or SQL structure is not known until runtime (e.g., archiving to different tables, reporting tool).

**Q2: What are the three collection types in PL/SQL?**
> Associative Array (INDEX BY - in-memory key/value, any key), Nested Table (dynamic size, can have holes, storable in DB), VARRAY (fixed max size, ordered, no holes, storable in DB).

**Q3: What is a pipelined function?**
> A function that returns rows one at a time as they're processed (PIPE ROW). You can query it like a table in SQL. More memory-efficient than returning all rows at once. Used for complex data generation/transformation.

**Q4: What is the difference between EXECUTE IMMEDIATE and DBMS_SQL?**
> EXECUTE IMMEDIATE is simpler for most dynamic SQL. DBMS_SQL is needed when number of columns or their types are unknown at runtime (truly dynamic SELECT *). 95% of cases use EXECUTE IMMEDIATE.

**Q5: How do you prevent SQL injection in PL/SQL?**
> Always use bind variables (USING clause) instead of string concatenation for data values. For object names (tables, columns), use DBMS_ASSERT.SIMPLE_SQL_NAME to validate.

**Q6: What is BULK COLLECT LIMIT and why use it?**
> `FETCH cursor BULK COLLECT INTO collection LIMIT n` fetches n rows at a time. Without LIMIT, all rows load into memory at once - risk of ORA-04031 (out of memory). LIMIT controls memory usage for large datasets.

---

## 📝 Quick Revision Summary

```
EXECUTE IMMEDIATE 'SQL string' [INTO var] [USING bind_vars]
  -> For DML/DDL/SELECT with dynamic structure

Bind variables: USING clause = prevent SQL injection + better performance

Collections:
  Associative Array: INDEX BY, any key, in-memory only
  Nested Table:      dynamic, holes possible, storable in DB
  VARRAY:            fixed max size, ordered, storable in DB

Methods: COUNT, FIRST, LAST, NEXT, PRIOR, EXISTS, EXTEND, DELETE

Pipelined Function:
  PIPE ROW(row_object) -> streams rows to caller
  Query with: SELECT * FROM TABLE(func_name())

DBMS_SQL: Use when column count/types unknown at compile time

SQL Injection: ALWAYS use bind variables for user input
  DBMS_ASSERT.SIMPLE_SQL_NAME() for object names
```

---

*Next: [Chapter 11 - Linux Basics](11_linux_basics.md)*
