# ⚡ thor-acle
## Oracle | PL/SQL | Linux/Shell | Oracle EBS | Order Management System

> **Study Plan**: Each chapter is a focused read of 15-25 minutes.
> Total: ~480 min across all 25 chapters if read end to end.

---

## 📚 How to Use This Guide

- Each file is a focused read - check the time estimate at the top of each chapter
- Chapters go from **fundamentals -> advanced -> interview traps**
- Every concept has a **plain-English analogy** to explain in plain English
- End of each chapter has **top interview questions with answers**

---

## 🗂️ Study Index

### Section 1: Oracle Database (~100 min)
| File | Topic | Priority |
|------|-------|----------|
| [01_oracle_architecture.md](01_oracle_architecture.md) | Architecture, SGA, PGA, Processes | ⭐⭐⭐ |
| [02_oracle_sql_core.md](02_oracle_sql_core.md) | SQL - Joins, Subqueries, Aggregates | ⭐⭐⭐ |
| [03_oracle_indexing_performance.md](03_oracle_indexing_performance.md) | Indexes, Explain Plan, Tuning | ⭐⭐⭐ |
| [04_oracle_transactions_locks.md](04_oracle_transactions_locks.md) | Transactions, Locks, MVCC, Undo | ⭐⭐⭐ |
| [05_oracle_advanced_features.md](05_oracle_advanced_features.md) | Partitioning, Materialized Views, Hints | ⭐⭐ |

### Section 2: PL/SQL (~90 min)
| File | Topic | Priority |
|------|-------|----------|
| [06_plsql_basics.md](06_plsql_basics.md) | Blocks, Variables, Control Flow | ⭐⭐⭐ |
| [07_plsql_cursors.md](07_plsql_cursors.md) | Explicit/Implicit Cursors, Bulk Collect | ⭐⭐⭐ |
| [08_plsql_procedures_functions.md](08_plsql_procedures_functions.md) | Procedures, Functions, Packages | ⭐⭐⭐ |
| [09_plsql_exceptions_triggers.md](09_plsql_exceptions_triggers.md) | Exception Handling, Triggers | ⭐⭐⭐ |
| [10_plsql_advanced.md](10_plsql_advanced.md) | Dynamic SQL, Collections, Pipelining | ⭐⭐ |

### Section 3: Linux & Unix Shell Scripting (~115 min)
| File | Topic | Priority |
|------|-------|----------|
| [11_linux_basics.md](11_linux_basics.md) | File System, Permissions, Navigation | ⭐⭐⭐ |
| [12_linux_text_processing.md](12_linux_text_processing.md) | grep, awk, sed, cut, sort, uniq | ⭐⭐⭐ |
| [13_shell_scripting_basics.md](13_shell_scripting_basics.md) | Variables, Loops, Conditions, Functions | ⭐⭐⭐ |
| [14_shell_scripting_advanced.md](14_shell_scripting_advanced.md) | Cron, Process Management, Traps, Logging | ⭐⭐⭐ |
| [15_linux_networking_troubleshoot.md](15_linux_networking_troubleshoot.md) | Networking, Debugging, Performance | ⭐⭐ |

### Section 4: Oracle EBS (~90 min)
| File | Topic | Priority |
|------|-------|----------|
| [16_oracle_ebs_architecture.md](16_oracle_ebs_architecture.md) | EBS Architecture, Tiers, Navigation | ⭐⭐⭐ |
| [17_oracle_ebs_concurrent.md](17_oracle_ebs_concurrent.md) | Concurrent Programs, Request Sets | ⭐⭐⭐ |
| [18_oracle_ebs_flex_dff.md](18_oracle_ebs_flex_dff.md) | Flexfields - KFF, DFF | ⭐⭐⭐ |
| [19_oracle_ebs_workflow_alerts.md](19_oracle_ebs_workflow_alerts.md) | Oracle Workflow & Alerts | ⭐⭐ |
| [20_oracle_ebs_apis_interfaces.md](20_oracle_ebs_apis_interfaces.md) | APIs, Open Interfaces, RICE Components | ⭐⭐⭐ |

### Section 5: Order Management System (~85 min)
| File | Topic | Priority |
|------|-------|----------|
| [21_oms_order_cycle.md](21_oms_order_cycle.md) | Order Life Cycle, Order Types, Status | ⭐⭐⭐ |
| [22_oms_pricing_tax.md](22_oms_pricing_tax.md) | Pricing, Price Lists, Tax | ⭐⭐⭐ |
| [23_oms_shipping_fulfillment.md](23_oms_shipping_fulfillment.md) | Shipping, Pick/Pack/Ship, WSH | ⭐⭐⭐ |
| [24_oms_holds_returns.md](24_oms_holds_returns.md) | Order Holds, Returns (RMA), Credits | ⭐⭐⭐ |
| [25_oms_integration_troubleshoot.md](25_oms_integration_troubleshoot.md) | Integration with AR/INV, Booking Issues | ⭐⭐⭐ |

---

## ⚡ Quick Revision (Day Before Interview)

### Oracle Cheatsheet
- SGA = shared memory (buffer cache, shared pool, redo log buffer)
- MVCC = readers don't block writers (undo segments)
- Index types: B-Tree (default), Bitmap (low cardinality), Function-based

### PL/SQL Cheatsheet
- Cursor FOR LOOP = simplest cursor (auto open/fetch/close)
- BULK COLLECT + FORALL = fastest DML on large sets
- Packages = related procedures/functions grouped together

### Linux Cheatsheet
- `grep -rin "word" file` - recursive, case-insensitive, line number
- `awk '{print $2}'` - print 2nd column
- `sed 's/old/new/g'` - find and replace globally

### OMS Cheatsheet
- Order -> Booked -> Awaiting Shipping -> Shipped -> Invoiced -> Closed
- Hold = stop order from progressing (credit hold, manual hold)
- RMA = Return Merchandise Authorization

---

## 🔥 Most Asked Interview Questions

1. What is the difference between `DELETE`, `TRUNCATE`, and `DROP`?
2. Explain Oracle MVCC - how does Oracle avoid read locks?
3. What is a cursor? When do you use BULK COLLECT?
4. Difference between a procedure and a function?
5. How do you debug a slow query in Oracle?
6. What is the OMS order life cycle?
7. What happens when you book an order in Oracle OMS?
8. What are the types of flexfields?
9. How do concurrent programs work in Oracle EBS?
10. Write a shell script to find files older than 7 days and delete them.

---

*Wield it like Mjolnir. Only the prepared are worthy.* 🔨
