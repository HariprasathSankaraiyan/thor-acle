# Chapter 6: PL/SQL Basics
## ⏱️ ~15 min read

---

## 🧠 Think of it this way

> SQL is a **question** you ask. PL/SQL is a **program** that asks many questions, makes decisions, loops through results, and takes actions.
>
> SQL: "Give me all open orders."
> PL/SQL: "Give me all open orders. For each one, check if the customer's credit is OK. If yes, approve it. If no, put it on hold. Log every decision. If anything fails, log the error."

---

## 1. PL/SQL Block Structure

Every PL/SQL program follows this structure:

```
[DECLARE]       ← Optional: declare variables, constants, cursors
BEGIN           ← Required: your code goes here
  [EXCEPTION]   ← Optional: error handling
END;            ← Required: close the block
/               ← Execute the block (in SQL*Plus / SQLDeveloper)
```

### Simple Example
```sql
DECLARE
  v_name    VARCHAR2(100);
  v_salary  NUMBER := 0;
  v_today   DATE := SYSDATE;
BEGIN
  SELECT employee_name, salary
  INTO v_name, v_salary
  FROM employees
  WHERE employee_id = 101;

  DBMS_OUTPUT.PUT_LINE('Name: ' || v_name || ', Salary: ' || v_salary);
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('Employee not found!');
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
```

---

## 2. Variable Declaration & Data Types

```sql
DECLARE
  -- Basic types
  v_id        NUMBER(10);
  v_name      VARCHAR2(100);
  v_flag      BOOLEAN := FALSE;
  v_date      DATE := SYSDATE;
  v_amount    NUMBER(15,2) := 0.00;

  -- Anchored types (inherit type from column - BEST PRACTICE)
  v_emp_name  employees.employee_name%TYPE;
  v_order_rec orders%ROWTYPE;           -- entire row as a record

  -- Constant
  c_tax_rate  CONSTANT NUMBER := 0.18;
```

> **Best practice**: Use `%TYPE` and `%ROWTYPE` - if column type changes in DB, your code adapts automatically.

### %ROWTYPE Usage
```sql
DECLARE
  v_order orders%ROWTYPE;
BEGIN
  SELECT * INTO v_order FROM orders WHERE order_id = 1001;
  DBMS_OUTPUT.PUT_LINE(v_order.order_date || ' - ' || v_order.status);
END;
```

---

## 3. Control Structures

### IF-ELSIF-ELSE
```sql
IF v_salary > 100000 THEN
  v_grade := 'A';
ELSIF v_salary > 50000 THEN
  v_grade := 'B';
ELSIF v_salary > 20000 THEN
  v_grade := 'C';
ELSE
  v_grade := 'D';
END IF;
```

### CASE Expression
```sql
v_label := CASE v_status
  WHEN 'O' THEN 'Open'
  WHEN 'C' THEN 'Closed'
  WHEN 'H' THEN 'On Hold'
  ELSE 'Unknown'
END;
```

### Searched CASE
```sql
v_discount := CASE
  WHEN v_amount > 10000 THEN 0.20
  WHEN v_amount > 5000  THEN 0.10
  WHEN v_amount > 1000  THEN 0.05
  ELSE 0
END;
```

---

## 4. Loops

### Basic LOOP (manual exit)
```sql
DECLARE
  v_i NUMBER := 1;
BEGIN
  LOOP
    DBMS_OUTPUT.PUT_LINE('Count: ' || v_i);
    v_i := v_i + 1;
    EXIT WHEN v_i > 5;  -- Must have EXIT or it runs forever!
  END LOOP;
END;
```

### WHILE LOOP
```sql
WHILE v_i <= 10 LOOP
  v_total := v_total + v_i;
  v_i := v_i + 1;
END LOOP;
```

### FOR LOOP (numeric - most common)
```sql
FOR i IN 1..10 LOOP
  DBMS_OUTPUT.PUT_LINE('Row: ' || i);
END LOOP;

-- Reverse
FOR i IN REVERSE 1..10 LOOP ...
```

### CONTINUE (skip to next iteration)
```sql
FOR i IN 1..10 LOOP
  CONTINUE WHEN MOD(i, 2) = 0;  -- skip even numbers
  DBMS_OUTPUT.PUT_LINE(i);
END LOOP;
```

---

## 5. SELECT INTO

> Fetch EXACTLY ONE row from a query into variables.

```sql
-- Single value
SELECT COUNT(*) INTO v_count FROM orders WHERE status = 'OPEN';

-- Multiple columns
SELECT customer_name, credit_limit
INTO v_cust_name, v_credit
FROM customers
WHERE customer_id = v_id;
```

**Errors to handle**:
- `NO_DATA_FOUND` - query returns 0 rows
- `TOO_MANY_ROWS` - query returns more than 1 row

> For multiple rows, use a CURSOR (next chapter).

---

## 6. DBMS_OUTPUT - Debugging

```sql
-- Enable output (run once in session)
SET SERVEROUTPUT ON;

DBMS_OUTPUT.PUT_LINE('Debug: value = ' || v_amount);
DBMS_OUTPUT.PUT_LINE('Order ID: ' || TO_CHAR(v_id));
```

---

## 7. String & Number Operations

```sql
-- String concatenation
v_full_name := v_first_name || ' ' || v_last_name;

-- String functions
v_upper   := UPPER(v_name);
v_len     := LENGTH(v_name);
v_sub     := SUBSTR(v_name, 1, 10);
v_pos     := INSTR(v_name, 'Smith');

-- Number functions
v_round   := ROUND(v_amount, 2);
v_trunc   := TRUNC(v_amount);
v_mod     := MOD(v_id, 10);
v_abs     := ABS(-100);

-- Type conversion
v_str     := TO_CHAR(SYSDATE, 'YYYY-MM-DD');
v_date    := TO_DATE('2023-01-15', 'YYYY-MM-DD');
v_num     := TO_NUMBER('12345.67');
```

---

## 8. Records (Custom Data Type)

```sql
-- Define a record type
TYPE t_order_rec IS RECORD (
  order_id  NUMBER,
  amount    NUMBER,
  status    VARCHAR2(20)
);

-- Declare variable of that type
v_order t_order_rec;

-- Assign values
v_order.order_id := 1001;
v_order.amount   := 5000;
v_order.status   := 'OPEN';
```

---

## 9. Nested Blocks

```sql
BEGIN
  -- Outer block
  BEGIN
    -- Inner block
    SELECT ... INTO ... FROM ...;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      NULL;  -- Handle inner block error, continue outer block
  END;
  -- Outer block continues even if inner fails
  DBMS_OUTPUT.PUT_LINE('Done');
END;
```

> Like try-catch inside try-catch. Handle inner errors locally.

---

## 10. NULL Handling in PL/SQL

```sql
-- NULL is NOT equal to anything, even NULL
IF v_val = NULL THEN   -- NEVER works!
IF v_val IS NULL THEN  -- Correct

-- NULL in boolean
v_flag := NULL;
IF v_flag THEN ... END IF;   -- skipped (NULL is not TRUE)
IF NOT v_flag THEN ...        -- also skipped!

-- NVL to handle NULL
v_total := NVL(v_amount, 0) + NVL(v_tax, 0);
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What are the sections of a PL/SQL block?**
> DECLARE (optional - variables), BEGIN (mandatory - logic), EXCEPTION (optional - error handling), END (mandatory).

**Q2: What is %TYPE and why use it?**
> `%TYPE` inherits the data type of a database column. Best practice because if the column type changes, your variable type automatically matches. Avoids type mismatch bugs.

**Q3: What errors can SELECT INTO raise?**
> NO_DATA_FOUND (0 rows returned) and TOO_MANY_ROWS (more than 1 row returned). Both must be handled in the EXCEPTION block.

**Q4: What is %ROWTYPE?**
> Declares a variable that represents an entire row of a table or cursor. Contains all columns as fields. More convenient than declaring each column separately.

**Q5: Difference between LOOP, WHILE LOOP, and FOR LOOP?**
> LOOP = infinite loop with manual EXIT. WHILE = condition checked before each iteration. FOR = fixed range (1..N), most readable. Cursor FOR LOOP is also very common.

**Q6: What does CONTINUE do in a loop?**
> Skips the rest of the current iteration and goes to the next. Added in Oracle 11g. `CONTINUE WHEN condition` is the shorthand form.

---

## 📝 Quick Revision Summary

```
Block structure: DECLARE -> BEGIN -> EXCEPTION -> END

Variables:
  v_name VARCHAR2(100);
  v_name employees.emp_name%TYPE;   -- anchored type (best)
  v_rec  orders%ROWTYPE;            -- entire row

Control:
  IF/ELSIF/ELSE/END IF
  CASE ... WHEN ... THEN ... END
  LOOP/EXIT WHEN, WHILE, FOR i IN 1..10

SELECT INTO = exactly 1 row (NO_DATA_FOUND, TOO_MANY_ROWS)

NULL: use IS NULL, never = NULL

DBMS_OUTPUT.PUT_LINE = print/debug
```

---

*Next: [Chapter 7 - PL/SQL Cursors](07_plsql_cursors.md)*
