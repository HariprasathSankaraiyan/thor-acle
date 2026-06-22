# Chapter 2: Oracle SQL Core
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> SQL is like **asking questions to a filing cabinet**.
> - `SELECT` = "Give me these files"
> - `WHERE` = "Only files matching this condition"
> - `JOIN` = "Combine information from two filing cabinets"
> - `GROUP BY` = "Sort and summarize by category"

---

## 1. SQL Statement Types

| Type | Commands | Purpose |
|------|----------|---------|
| **DML** - Data Manipulation | SELECT, INSERT, UPDATE, DELETE, MERGE | Work with data rows |
| **DDL** - Data Definition | CREATE, ALTER, DROP, TRUNCATE, RENAME | Work with structure |
| **DCL** - Data Control | GRANT, REVOKE | Permissions |
| **TCL** - Transaction Control | COMMIT, ROLLBACK, SAVEPOINT | Manage transactions |

> **Interview trap**: TRUNCATE is DDL (not DML). It auto-commits and cannot be rolled back!

---

## 2. JOINs - The Most Important SQL Topic

### 2a. INNER JOIN
Returns rows that **exist in BOTH** tables.
```sql
-- Only show me orders that have a customer record
SELECT o.order_id, c.customer_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;
```
> Only matching rows. Non-matching are excluded.

### 2b. LEFT OUTER JOIN
Returns **ALL rows from left** table + matching rows from right. Non-matching right = NULL.
```sql
-- Show ALL orders, even if customer record is missing
SELECT o.order_id, c.customer_name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id;
```

### 2c. RIGHT OUTER JOIN
Returns **ALL rows from right** table + matching from left.
```sql
-- Show ALL customers, even if they have no orders
SELECT o.order_id, c.customer_name
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.customer_id;
```

### 2d. FULL OUTER JOIN
Returns **ALL rows from BOTH** tables. Non-matching = NULL on either side.
```sql
SELECT o.order_id, c.customer_name
FROM orders o
FULL OUTER JOIN customers c ON o.customer_id = c.customer_id;
```

### 2e. CROSS JOIN
Returns **every combination** of rows (Cartesian product).
```sql
-- 3 shirts × 4 colors = 12 combinations
SELECT shirt, color FROM shirts CROSS JOIN colors;
```

### 2f. SELF JOIN
Table joins **itself** - useful for hierarchies.
```sql
-- Find each employee and their manager (both in same table)
SELECT e.name AS employee, m.name AS manager
FROM employees e
JOIN employees m ON e.manager_id = m.employee_id;
```

---

## 3. Subqueries

### 3a. Single-Row Subquery
```sql
-- Who has the highest salary?
SELECT name FROM employees
WHERE salary = (SELECT MAX(salary) FROM employees);
```

### 3b. Multi-Row Subquery (IN, ANY, ALL)
```sql
-- Orders in 'New York' or 'Chicago'
SELECT * FROM orders
WHERE city_id IN (SELECT city_id FROM cities WHERE city_name IN ('New York','Chicago'));
```

### 3c. Correlated Subquery
> The inner query refers back to the outer query - runs once per outer row.
```sql
-- Employees earning more than their dept average
SELECT name, salary, dept_id
FROM employees e
WHERE salary > (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id);
```

### 3d. EXISTS vs IN
```sql
-- EXISTS: faster when subquery returns many rows
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM order_lines ol WHERE ol.order_id = o.order_id);

-- IN: fine for small subquery result sets
SELECT * FROM orders WHERE status IN ('OPEN','PENDING');
```
> **Interview**: Use EXISTS for large subqueries (stops at first match). IN loads all values first.

---

## 4. Aggregate Functions + GROUP BY + HAVING

```sql
-- Count orders per customer, only show customers with 5+ orders
SELECT customer_id, COUNT(*) AS order_count, SUM(total_amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING COUNT(*) >= 5
ORDER BY total_spent DESC;
```

| Function | What it does |
|----------|-------------|
| `COUNT(*)` | Count all rows |
| `COUNT(col)` | Count non-NULL values |
| `SUM(col)` | Total |
| `AVG(col)` | Average |
| `MAX(col)` | Highest value |
| `MIN(col)` | Lowest value |

> **Key rule**: `WHERE` filters rows BEFORE grouping. `HAVING` filters AFTER grouping.

---

## 5. Analytic (Window) Functions - 9-Year Level Must Know

> Like GROUP BY but you keep all rows - each row gets a "window" calculation.

### ROW_NUMBER, RANK, DENSE_RANK
```sql
-- Rank employees by salary within each department
SELECT name, dept_id, salary,
  ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS row_num,
  RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS d_rnk
FROM employees;
```

| Function | Ties behavior | Example result |
|----------|--------------|---------------|
| ROW_NUMBER | No ties, always unique | 1,2,3,4 |
| RANK | Ties get same rank, next rank skipped | 1,1,3,4 |
| DENSE_RANK | Ties get same rank, NO skip | 1,1,2,3 |

### LAG and LEAD
```sql
-- Compare this month's sales to last month
SELECT month, sales,
  LAG(sales, 1)  OVER (ORDER BY month) AS prev_month_sales,
  LEAD(sales, 1) OVER (ORDER BY month) AS next_month_sales
FROM monthly_sales;
```

### Running Total with SUM OVER
```sql
SELECT order_id, amount,
  SUM(amount) OVER (ORDER BY order_id) AS running_total
FROM orders;
```

---

## 6. DELETE vs TRUNCATE vs DROP

| | DELETE | TRUNCATE | DROP |
|--|--------|----------|------|
| Type | DML | DDL | DDL |
| WHERE clause | Yes | No | No |
| Rollback | Yes | No (auto-commit) | No |
| Triggers fired | Yes | No | No |
| Speed | Slow (row by row) | Fast (deallocates) | Instant |
| Recovers space | No (HWM stays) | Yes | Yes (removes table) |

> DELETE = erase rows one by one with eraser. TRUNCATE = tear the page out. DROP = throw the whole notebook away.

---

## 7. MERGE Statement

> "Upsert" - if the row exists, update it; if not, insert it.

```sql
MERGE INTO target_orders t
USING source_orders s ON (t.order_id = s.order_id)
WHEN MATCHED THEN
  UPDATE SET t.status = s.status, t.amount = s.amount
WHEN NOT MATCHED THEN
  INSERT (order_id, status, amount)
  VALUES (s.order_id, s.status, s.amount);
```

---

## 8. DECODE vs CASE

```sql
-- DECODE (Oracle specific)
SELECT DECODE(status, 'O','Open', 'C','Closed', 'Unknown') FROM orders;

-- CASE (ANSI standard, more powerful)
SELECT CASE
  WHEN status = 'O' THEN 'Open'
  WHEN status = 'C' THEN 'Closed'
  WHEN amount > 10000 THEN 'High Value'
  ELSE 'Other'
END AS status_label
FROM orders;
```

---

## 9. NVL, NVL2, NULLIF, COALESCE

```sql
NVL(val, default)          -- if val is NULL, use default
NVL2(val, not_null, null)  -- if val NOT null -> not_null_val, else null_val
NULLIF(a, b)               -- returns NULL if a = b, else returns a
COALESCE(a, b, c)          -- first non-NULL value from list
```

> NVL = "use a backup value if empty". COALESCE = "pick the first non-empty option from a list".

---

## 10. String Functions (Quick Reference)

```sql
UPPER('hello')           -> 'HELLO'
LOWER('HELLO')           -> 'hello'
SUBSTR('Oracle', 2, 3)   -> 'rac'       (start=2, length=3)
INSTR('Oracle', 'a')     -> 4           (position of 'a')
LENGTH('Oracle')         -> 6
TRIM('  hello  ')        -> 'hello'
LPAD('5', 4, '0')        -> '0005'      (pad left with zeros)
REPLACE('hello', 'l', 'r') -> 'herro'
CONCAT('Ora','cle')      -> 'Oracle'
```

---

## 11. Date Functions

```sql
SYSDATE                         -- current date+time
SYSTIMESTAMP                    -- current date+time+timezone
ADD_MONTHS(SYSDATE, 3)          -- add 3 months
MONTHS_BETWEEN(date1, date2)    -- number of months difference
TRUNC(SYSDATE, 'MM')            -- first day of current month
TO_DATE('2023-01-15','YYYY-MM-DD') -- string to date
TO_CHAR(SYSDATE, 'DD-MON-YYYY') -- date to string
LAST_DAY(SYSDATE)               -- last day of current month
NEXT_DAY(SYSDATE, 'MONDAY')     -- next Monday
```

---

## 12. WITH Clause (CTE - Common Table Expression)

```sql
-- Like creating a temporary named query to reuse
WITH high_value_orders AS (
  SELECT * FROM orders WHERE amount > 5000
),
customer_totals AS (
  SELECT customer_id, SUM(amount) total
  FROM high_value_orders
  GROUP BY customer_id
)
SELECT c.customer_name, ct.total
FROM customer_totals ct
JOIN customers c ON c.customer_id = ct.customer_id;
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between WHERE and HAVING?**
> WHERE filters rows before grouping. HAVING filters groups after GROUP BY. You cannot use aggregate functions in WHERE.

**Q2: Difference between RANK and DENSE_RANK?**
> RANK skips numbers after a tie (1,1,3). DENSE_RANK doesn't skip (1,1,2). ROW_NUMBER always gives unique numbers.

**Q3: What is a correlated subquery?**
> A subquery that references columns from the outer query. It re-executes for every row of the outer query. Example: find employees earning above their department average.

**Q4: When would you use EXISTS over IN?**
> EXISTS stops at the first match (short-circuits). IN evaluates the full subquery result. For large subqueries, EXISTS is faster.

**Q5: What does MERGE do?**
> MERGE performs "upsert" - UPDATE if the row exists, INSERT if it doesn't. Single statement replacing separate INSERT + UPDATE logic.

**Q6: Explain TRUNCATE vs DELETE performance difference.**
> TRUNCATE is faster because it deallocates the data pages directly rather than deleting row by row. It also resets the High Water Mark, allowing Oracle to reclaim space immediately.

---

## 📝 Quick Revision Summary

```
JOINs: INNER (matching only), LEFT (all from left), FULL (all from both)
Self-JOIN: table joins itself (hierarchy/employee-manager)

WHERE  -> filters before GROUP BY
HAVING -> filters after GROUP BY

ROW_NUMBER: unique always
RANK: gaps after ties
DENSE_RANK: no gaps after ties

DELETE=DML (rollback OK), TRUNCATE=DDL (no rollback), DROP=removes table

NVL(val, default) = NULL safety
COALESCE = first non-NULL from list

WITH clause = named temporary query (CTE)
```

---

*Next: [Chapter 3 - Indexing & Performance](03_oracle_indexing_performance.md)*
