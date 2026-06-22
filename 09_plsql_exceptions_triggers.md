# Chapter 9: Exception Handling & Triggers
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> **Exceptions** = Emergency plan at a restaurant:
> If the chef burns the food (error), you have a plan: apologize to customer, offer free dessert, don't close the restaurant (WHEN OTHERS handle it, don't crash).
>
> **Triggers** = Security camera with auto-alert:
> Every time someone enters the vault (table is modified), the camera automatically records it and sends an alert - without you doing anything manually.

---

## PART A: EXCEPTION HANDLING

## 1. Exception Handling Structure

```sql
BEGIN
  -- Normal code here
  SELECT ...
  UPDATE ...
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    -- Handle "not found" specifically
    DBMS_OUTPUT.PUT_LINE('Record not found');
  WHEN TOO_MANY_ROWS THEN
    -- Handle "too many rows" for SELECT INTO
    DBMS_OUTPUT.PUT_LINE('Multiple records found, expected one');
  WHEN DUP_VAL_ON_INDEX THEN
    -- Duplicate primary key / unique key violation
    DBMS_OUTPUT.PUT_LINE('Duplicate record!');
  WHEN OTHERS THEN
    -- Catch-all for anything else
    DBMS_OUTPUT.PUT_LINE('Unexpected error: ' || SQLERRM);
    RAISE;  -- Re-raise to propagate to caller
END;
```

---

## 2. Pre-Defined (Named) Exceptions

| Exception | Oracle Error | When it happens |
|-----------|-------------|----------------|
| `NO_DATA_FOUND` | ORA-01403 | SELECT INTO returns 0 rows |
| `TOO_MANY_ROWS` | ORA-01422 | SELECT INTO returns >1 rows |
| `DUP_VAL_ON_INDEX` | ORA-00001 | Unique constraint violated |
| `ZERO_DIVIDE` | ORA-01476 | Division by zero |
| `INVALID_NUMBER` | ORA-01722 | String -> Number conversion fails |
| `VALUE_ERROR` | ORA-06502 | Numeric or value error (overflow etc.) |
| `CURSOR_ALREADY_OPEN` | ORA-06511 | Opening an already-open cursor |
| `LOGIN_DENIED` | ORA-01017 | Bad username/password |
| `NOT_LOGGED_ON` | ORA-01012 | Not connected to Oracle |
| `PROGRAM_ERROR` | ORA-06501 | Internal PL/SQL error |
| `STORAGE_ERROR` | ORA-06500 | Out of memory |

---

## 3. SQLERRM and SQLCODE

```sql
EXCEPTION
  WHEN OTHERS THEN
    -- SQLCODE: error number (negative for Oracle errors)
    -- SQLERRM: error message text
    DBMS_OUTPUT.PUT_LINE('Error Code: ' || SQLCODE);
    DBMS_OUTPUT.PUT_LINE('Error Message: ' || SQLERRM);
    -- Example output:
    -- Error Code: -1403
    -- Error Message: ORA-01403: no data found
```

---

## 4. User-Defined Exceptions

```sql
DECLARE
  e_invalid_amount EXCEPTION;          -- Declare custom exception
  v_amount NUMBER := -500;
BEGIN
  IF v_amount < 0 THEN
    RAISE e_invalid_amount;            -- Manually raise it
  END IF;
  DBMS_OUTPUT.PUT_LINE('Processing: ' || v_amount);
EXCEPTION
  WHEN e_invalid_amount THEN
    DBMS_OUTPUT.PUT_LINE('Amount cannot be negative!');
END;
```

---

## 5. RAISE_APPLICATION_ERROR - Custom ORA- Errors

> Like a fire alarm with a custom message. You trigger it AND specify what it says.

```sql
-- Error numbers: -20000 to -20999 are reserved for user applications
RAISE_APPLICATION_ERROR(-20001, 'Order amount cannot exceed credit limit');
RAISE_APPLICATION_ERROR(-20002, 'Customer ' || v_id || ' is inactive');
```

This creates an actual ORA error that propagates to the caller:
```
ORA-20001: Order amount cannot exceed credit limit
```

> **Best practice**: Use in procedures/packages when you want meaningful business error messages.

---

## 6. PRAGMA EXCEPTION_INIT - Link ORA Error to Named Exception

```sql
DECLARE
  e_parent_key_not_found EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_parent_key_not_found, -2291);  -- ORA-02291 = FK violation
BEGIN
  INSERT INTO order_lines (order_id, ...) VALUES (99999, ...);
EXCEPTION
  WHEN e_parent_key_not_found THEN
    DBMS_OUTPUT.PUT_LINE('Order 99999 does not exist in orders table!');
END;
```

---

## 7. Exception Propagation

```sql
PROCEDURE inner_proc AS
BEGIN
  RAISE NO_DATA_FOUND;  -- Exception raised here
  -- No handler here -> propagates UP to caller
END;

PROCEDURE outer_proc AS
BEGIN
  inner_proc;           -- Exception propagates from here
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('Caught in outer_proc!');
END;
```

> If no handler anywhere in the call stack -> error returns to the user/application.

---

## PART B: TRIGGERS

## 8. What is a Trigger?

> An automatic "if-this-then-that" rule on a table.
> "If ANYONE inserts a new order, AUTOMATICALLY log it to audit_log."
> You don't call it - it fires on its own.

---

## 9. Types of Triggers

| Type | When it fires |
|------|-------------|
| **BEFORE** | Before the DML action happens |
| **AFTER** | After the DML action completes |
| **INSTEAD OF** | Instead of the DML (on views) |
| **Row-level** | Once per affected row |
| **Statement-level** | Once per DML statement |

---

## 10. Basic Trigger Syntax

```sql
CREATE OR REPLACE TRIGGER trg_orders_audit
  AFTER INSERT OR UPDATE OR DELETE
  ON orders
  FOR EACH ROW  -- Row-level trigger
DECLARE
  v_action VARCHAR2(10);
BEGIN
  IF INSERTING THEN
    v_action := 'INSERT';
  ELSIF UPDATING THEN
    v_action := 'UPDATE';
  ELSIF DELETING THEN
    v_action := 'DELETE';
  END IF;

  INSERT INTO orders_audit_log (
    action, order_id, old_status, new_status, changed_by, changed_on
  ) VALUES (
    v_action,
    :NEW.order_id,
    :OLD.status,
    :NEW.status,
    USER,
    SYSDATE
  );
END trg_orders_audit;
/
```

### :OLD and :NEW Pseudorecords

| Operation | :OLD | :NEW |
|-----------|------|------|
| INSERT | NULL | new values |
| UPDATE | old values | new values |
| DELETE | old values | NULL |

---

## 11. BEFORE Trigger - Modify Data Before Save

```sql
-- Auto-populate audit columns before insert
CREATE OR REPLACE TRIGGER trg_orders_before_insert
  BEFORE INSERT ON orders
  FOR EACH ROW
BEGIN
  :NEW.created_by   := NVL(:NEW.created_by, USER);
  :NEW.created_date := NVL(:NEW.created_date, SYSDATE);
  :NEW.order_id     := NVL(:NEW.order_id, seq_order_id.NEXTVAL);
END trg_orders_before_insert;
/
```

> **BEFORE trigger can modify :NEW** - use to auto-populate fields, validate, transform.
> **AFTER trigger cannot modify :NEW** - row already written.

---

## 12. INSTEAD OF Trigger (on Views)

```sql
-- Views don't support DML normally if they're complex
-- INSTEAD OF trigger intercepts and handles it
CREATE OR REPLACE TRIGGER trg_v_orders_insert
  INSTEAD OF INSERT ON v_order_summary
  FOR EACH ROW
BEGIN
  INSERT INTO orders (order_id, customer_id, amount)
  VALUES (:NEW.order_id, :NEW.customer_id, :NEW.amount);
END;
/
```

---

## 13. Statement vs Row Level

```sql
-- Statement-level: fires ONCE per DML statement (no FOR EACH ROW)
CREATE OR REPLACE TRIGGER trg_orders_stmt
  AFTER UPDATE ON orders
BEGIN
  DBMS_OUTPUT.PUT_LINE('Orders table was updated at ' || SYSDATE);
END;
/
-- If you update 1000 rows, this fires ONCE

-- Row-level: fires FOR EACH ROW affected (with FOR EACH ROW)
CREATE OR REPLACE TRIGGER trg_orders_row
  AFTER UPDATE ON orders
  FOR EACH ROW
BEGIN
  NULL; -- fires 1000 times if 1000 rows updated
END;
/
```

---

## 14. Mutating Table Error (ORA-04091)

> You're changing a room while reading a list of rooms. Oracle won't let you.

```sql
-- PROBLEM: Row-level trigger on orders trying to query orders
CREATE TRIGGER trg_check_limit
  BEFORE INSERT ON orders
  FOR EACH ROW
DECLARE
  v_count NUMBER;
BEGIN
  -- ORA-04091: orders table is mutating!
  SELECT COUNT(*) INTO v_count FROM orders WHERE customer_id = :NEW.customer_id;
END;
/
```

**Solutions**:
1. Use a compound trigger (AFTER STATEMENT section)
2. Use a package variable to collect values, query after statement
3. Restructure logic (use constraint instead of trigger)

---

## 15. Enable/Disable Triggers

```sql
-- Disable (useful for bulk loads)
ALTER TRIGGER trg_orders_audit DISABLE;
ALTER TABLE orders DISABLE ALL TRIGGERS;

-- Enable
ALTER TRIGGER trg_orders_audit ENABLE;
ALTER TABLE orders ENABLE ALL TRIGGERS;

-- Drop
DROP TRIGGER trg_orders_audit;
```

---

## 16. Compound Trigger (Oracle 11g+)

> Combines statement and row events in one trigger to avoid mutating table issue.

```sql
CREATE OR REPLACE TRIGGER trg_compound_orders
  FOR INSERT OR UPDATE OR DELETE ON orders
  COMPOUND TRIGGER

  -- Package-level variable (shared across all phases)
  TYPE t_ids IS TABLE OF NUMBER INDEX BY PLS_INTEGER;
  g_changed_ids t_ids;
  g_idx PLS_INTEGER := 0;

  AFTER EACH ROW IS
  BEGIN
    g_idx := g_idx + 1;
    g_changed_ids(g_idx) := :NEW.order_id;
  END AFTER EACH ROW;

  AFTER STATEMENT IS
  BEGIN
    -- Now safe to query orders table
    FOR i IN 1..g_idx LOOP
      -- process g_changed_ids(i)
      NULL;
    END LOOP;
  END AFTER STATEMENT;

END trg_compound_orders;
/
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between RAISE and RAISE_APPLICATION_ERROR?**
> RAISE re-raises an existing named exception or the current exception. RAISE_APPLICATION_ERROR creates a brand new error with a custom ORA number (-20000 to -20999) and message, visible to the caller as an ORA error.

**Q2: When would you use PRAGMA EXCEPTION_INIT?**
> To give a user-friendly name to a specific Oracle error code. For example, naming ORA-02291 as `e_parent_not_found` for better readability in exception handlers.

**Q3: What is a BEFORE trigger vs AFTER trigger?**
> BEFORE fires before the DML and can modify :NEW values (set defaults, validate, sequence generation). AFTER fires after and is used for auditing, cascades - cannot modify :NEW.

**Q4: What is the mutating table error?**
> ORA-04091. A row-level trigger on a table cannot query or modify that same table (it's "mutating" - mid-change). Solution: use compound trigger, move query to AFTER STATEMENT section.

**Q5: What is :NEW and :OLD?**
> Pseudorecords in row-level triggers. :NEW has the new row values (INSERT/UPDATE). :OLD has the old row values (UPDATE/DELETE). For INSERT: :OLD is NULL. For DELETE: :NEW is NULL.

**Q6: When would you use an INSTEAD OF trigger?**
> On views that can't directly support DML (complex views with joins, GROUP BY). The trigger intercepts the DML and redirects it to the underlying tables.

---

## 📝 Quick Revision Summary

```
Exception types:
  Pre-defined: NO_DATA_FOUND, TOO_MANY_ROWS, DUP_VAL_ON_INDEX, ZERO_DIVIDE
  User-defined: EXCEPTION; + RAISE e_name;
  Named Oracle errors: PRAGMA EXCEPTION_INIT(e_name, -ORA_NUM)

SQLERRM = error message text
SQLCODE = error number (negative)
RAISE_APPLICATION_ERROR(-20001, 'message') = custom ORA error

Trigger timing:
  BEFORE = fires before DML, can modify :NEW
  AFTER  = fires after DML, used for auditing
  INSTEAD OF = on views, intercepts DML

:NEW = new row values (INSERT/UPDATE)
:OLD = old row values (UPDATE/DELETE)

ORA-04091 = mutating table
Fix = compound trigger (AFTER STATEMENT section)

Disable trigger: ALTER TRIGGER ... DISABLE
```

---

*Next: [Chapter 10 - Advanced PL/SQL](10_plsql_advanced.md)*
