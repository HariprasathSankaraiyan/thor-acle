# Chapter 8: Procedures, Functions & Packages
## ⏱️ ~15 min read

---

## 🧠 Think of it this way

> **Procedure**: A recipe you follow. You cook the meal (do the work). No return value - the action IS the result.
> **Function**: A calculator. You give input, it gives you one answer back.
> **Package**: A recipe book. Organizes related recipes (procedures + functions) together.
>
> "Call HR department" (procedure - they do things)
> "What's the tax on $500?" (function - it gives you $90)
> "HR Handbook" (package - all HR-related things in one place)

---

## 1. Stored Procedures

```sql
-- Create a procedure
CREATE OR REPLACE PROCEDURE update_order_status (
  p_order_id  IN  NUMBER,
  p_status    IN  VARCHAR2,
  p_message   OUT VARCHAR2   -- OUT parameter returns a value
) AS
  v_count NUMBER;
BEGIN
  -- Validate
  SELECT COUNT(*) INTO v_count FROM orders WHERE order_id = p_order_id;
  IF v_count = 0 THEN
    p_message := 'ERROR: Order not found';
    RETURN;
  END IF;

  -- Perform update
  UPDATE orders
  SET status = p_status, last_updated = SYSDATE
  WHERE order_id = p_order_id;

  COMMIT;
  p_message := 'SUCCESS: Order ' || p_order_id || ' updated to ' || p_status;

EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
    p_message := 'ERROR: ' || SQLERRM;
END update_order_status;
/
```

### Calling a Procedure
```sql
DECLARE
  v_msg VARCHAR2(200);
BEGIN
  update_order_status(1001, 'CLOSED', v_msg);
  DBMS_OUTPUT.PUT_LINE(v_msg);
END;
/
```

---

## 2. Parameter Modes

| Mode | Direction | Can read? | Can write? | Use for |
|------|-----------|-----------|------------|---------|
| `IN` (default) | Input | Yes | No | Passing values in |
| `OUT` | Output | No | Yes | Returning values out |
| `IN OUT` | Both | Yes | Yes | Pass in, modify, return |

```sql
PROCEDURE calculate_tax (
  p_amount    IN     NUMBER,     -- Input only
  p_tax       OUT    NUMBER,     -- Output only
  p_net_total IN OUT NUMBER      -- Pass in, modified, returned
)
```

---

## 3. Stored Functions

> Functions MUST return a value. Can be used directly in SQL.

```sql
-- Create a function
CREATE OR REPLACE FUNCTION get_customer_name (
  p_customer_id IN NUMBER
) RETURN VARCHAR2 AS
  v_name VARCHAR2(100);
BEGIN
  SELECT customer_name INTO v_name
  FROM customers
  WHERE customer_id = p_customer_id;
  RETURN v_name;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RETURN 'UNKNOWN';
END get_customer_name;
/

-- Use in SQL directly!
SELECT order_id, get_customer_name(customer_id) AS cust_name
FROM orders WHERE status = 'OPEN';

-- Use in PL/SQL
v_name := get_customer_name(101);
```

---

## 4. Procedure vs Function - Key Differences

| | Procedure | Function |
|--|-----------|----------|
| Returns value? | Not required (uses OUT params) | Mandatory (RETURN) |
| Used in SQL? | No | Yes (if no DML inside) |
| Can have DML? | Yes | Yes (but restricted in SQL) |
| Primary purpose | Perform actions | Compute and return |
| Called with | `EXEC proc_name()` or in BEGIN/END | `v := func_name()` or in SELECT |

---

## 5. Packages - The Most Important PL/SQL Feature

> A package is like a toolbox with a lock.
> - **Package Spec (Specification)**: The label on the toolbox - tells you what tools are inside (public interface)
> - **Package Body**: The actual tools inside the box (implementation)

### Package Specification (Public interface)
```sql
CREATE OR REPLACE PACKAGE pkg_orders AS
  -- Public constant
  C_MAX_HOLD_DAYS CONSTANT NUMBER := 30;

  -- Public type
  TYPE t_order_rec IS RECORD (
    order_id NUMBER,
    amount   NUMBER,
    status   VARCHAR2(20)
  );

  -- Public procedure declarations
  PROCEDURE approve_order(p_order_id IN NUMBER);
  PROCEDURE hold_order(p_order_id IN NUMBER, p_reason IN VARCHAR2);

  -- Public function declaration
  FUNCTION get_order_total(p_customer_id IN NUMBER) RETURN NUMBER;

END pkg_orders;
/
```

### Package Body (Implementation)
```sql
CREATE OR REPLACE PACKAGE BODY pkg_orders AS

  -- Private variable (only accessible within this package)
  g_last_processed_id NUMBER;

  -- Private helper function (not in spec = private)
  FUNCTION validate_order(p_id NUMBER) RETURN BOOLEAN AS
    v_cnt NUMBER;
  BEGIN
    SELECT COUNT(*) INTO v_cnt FROM orders WHERE order_id = p_id;
    RETURN v_cnt > 0;
  END validate_order;

  -- Public procedure implementation
  PROCEDURE approve_order(p_order_id IN NUMBER) AS
  BEGIN
    IF NOT validate_order(p_order_id) THEN
      RAISE_APPLICATION_ERROR(-20001, 'Order not found: ' || p_order_id);
    END IF;
    UPDATE orders SET status = 'APPROVED' WHERE order_id = p_order_id;
    g_last_processed_id := p_order_id;
    COMMIT;
  END approve_order;

  PROCEDURE hold_order(p_order_id IN NUMBER, p_reason IN VARCHAR2) AS
  BEGIN
    UPDATE orders SET status = 'HOLD', hold_reason = p_reason
    WHERE order_id = p_order_id;
    COMMIT;
  END hold_order;

  FUNCTION get_order_total(p_customer_id IN NUMBER) RETURN NUMBER AS
    v_total NUMBER;
  BEGIN
    SELECT NVL(SUM(amount), 0) INTO v_total
    FROM orders WHERE customer_id = p_customer_id AND status != 'CANCELLED';
    RETURN v_total;
  END get_order_total;

END pkg_orders;
/
```

### Calling Package Members
```sql
-- Call procedure
EXEC pkg_orders.approve_order(1001);

-- Call function
v_total := pkg_orders.get_order_total(500);

-- Use constant
v_days := pkg_orders.C_MAX_HOLD_DAYS;
```

---

## 6. Package Benefits - Why Packages Are Preferred

1. **Encapsulation**: Hide private logic, expose only public interface
2. **Performance**: Package is loaded into shared pool at first call, subsequent calls reuse in-memory version
3. **Overloading**: Multiple procedures/functions with same name but different parameters
4. **Global state**: Package-level variables persist for the session
5. **Modular design**: Related code grouped together
6. **Reduce recompilation**: Changing body doesn't invalidate callers (only spec changes do)

---

## 7. Overloading in Packages

```sql
CREATE OR REPLACE PACKAGE pkg_utils AS
  -- Same name, different parameter types
  FUNCTION format_amount(p_amount NUMBER)  RETURN VARCHAR2;
  FUNCTION format_amount(p_amount NUMBER, p_currency VARCHAR2) RETURN VARCHAR2;
  FUNCTION format_amount(p_amount NUMBER, p_decimals NUMBER) RETURN VARCHAR2;
END pkg_utils;
```

---

## 8. Package Variables (Session-Level State)

```sql
CREATE OR REPLACE PACKAGE pkg_context AS
  g_user_id   NUMBER;
  g_user_name VARCHAR2(100);

  PROCEDURE set_context(p_user_id NUMBER);
  FUNCTION  get_user_name RETURN VARCHAR2;
END pkg_context;
/

-- Set once at login
pkg_context.set_context(12345);

-- Read anywhere in the session
v_user := pkg_context.get_user_name();
```

---

## 9. Useful Built-in Packages

| Package | Purpose |
|---------|---------|
| `DBMS_OUTPUT` | Print to screen (debugging) |
| `DBMS_SQL` | Dynamic SQL execution |
| `DBMS_SCHEDULER` | Job scheduling |
| `DBMS_STATS` | Gather statistics |
| `DBMS_LOB` | Handle LOB (Large Object) data |
| `DBMS_CRYPTO` | Encryption/decryption |
| `DBMS_ALERT` | Database alerts |
| `DBMS_PIPE` | Inter-session communication |
| `UTL_FILE` | Read/write OS files |
| `UTL_HTTP` | HTTP requests from PL/SQL |
| `UTL_MAIL` | Send emails |

---

## 10. Compiling & Managing Code

```sql
-- Check for invalid objects
SELECT object_name, object_type, status
FROM user_objects WHERE status = 'INVALID';

-- Recompile a procedure
ALTER PROCEDURE update_order_status COMPILE;

-- Recompile a package body
ALTER PACKAGE pkg_orders COMPILE BODY;

-- Recompile all invalid objects
EXEC UTL_RECOMP.RECOMP_SERIAL();

-- Check errors after compilation
SHOW ERRORS;
SELECT * FROM user_errors WHERE name = 'PKG_ORDERS';

-- Drop
DROP PROCEDURE update_order_status;
DROP PACKAGE pkg_orders;
DROP FUNCTION get_customer_name;
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between a procedure and a function?**
> A function must return a value and can be used in SQL statements. A procedure performs actions, doesn't have to return a value (uses OUT params). Both can have IN/OUT parameters.

**Q2: What is a package? Why use it?**
> A package groups related procedures and functions. Benefits: encapsulation (private/public), performance (loaded once in shared pool), overloading, session-level state, modular code.

**Q3: What is the difference between package spec and body?**
> Spec is the public interface - declares what's available. Body is the implementation. Callers only need the spec compiled. Changing body alone doesn't invalidate callers.

**Q4: Can a function have DML (INSERT/UPDATE/DELETE)?**
> Yes, but functions with DML cannot be called from SQL statements (only from PL/SQL). This is to maintain SQL's read-consistency guarantee.

**Q5: What is overloading in PL/SQL?**
> Multiple procedures/functions with the same name but different parameter types or counts. Only supported within packages. Oracle resolves which one to call based on argument types.

**Q6: What happens when a package is first accessed?**
> The entire package is loaded into the Shared Pool (Library Cache). All subsequent calls in the session reuse the cached version. This is why packages improve performance over standalone procedures.

---

## 📝 Quick Revision Summary

```
Procedure: action, no mandatory return, cannot use in SQL
Function:  must RETURN, can use in SQL (if no DML)

Parameter modes:
  IN     = input (default)
  OUT    = output (written by proc)
  IN OUT = both

Package = Spec (public interface) + Body (implementation)
  Private members = in body only, not in spec
  Package vars = session-level state
  Overloading = same name, different params
  Load-once performance = shared pool benefit

Built-ins:
  DBMS_OUTPUT -> debug
  UTL_FILE    -> file I/O
  DBMS_STATS  -> gather stats
  DBMS_SQL    -> dynamic SQL

SHOW ERRORS / user_errors -> compile errors
```

---

*Next: [Chapter 9 - Exception Handling & Triggers](09_plsql_exceptions_triggers.md)*
