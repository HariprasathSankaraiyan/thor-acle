# Chapter 17: Oracle EBS - Concurrent Programs & Request Sets
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> A **Concurrent Program** is like a **background task** you submit to a call center:
> "Please generate my monthly statement and email it to me."
> You don't wait at the desk - you get a **Request ID**, and come back later to pick up the output.
>
> The **Concurrent Manager** is the **office manager** who distributes tasks to available workers.
>
> A **Request Set** is like a **workflow package**: "First run Report A, then (only if A succeeds) run Report B, then send email."

---

## 1. What is a Concurrent Program?

- A background process that runs independently of the user's session
- Can be: PL/SQL stored procedure, Shell script, SQL*Plus script, Java program, Report (XML/RDF)
- Identified by: **Short Name** (internal) and **User Name** (display name)
- Output saved to: **Log file** + **Output file**
- Status: Pending -> Running -> Completed (Normal/Error/Warning)

---

## 2. Concurrent Manager Architecture

```
User submits request
       │
       ▼
FND_CONCURRENT_REQUESTS table (queue)
       │
       ▼
Internal Concurrent Manager (ICM)
       │
       ├─► Standard Manager         -> generic programs
       ├─► Output Post Processor    -> handles output printing
       ├─► Conflict Resolution Mgr  -> handles incompatibilities
       └─► Custom Managers          -> created by DBA for specific programs
```

### Key Tables
```sql
FND_CONCURRENT_PROGRAMS       -- Program definitions
FND_CONCURRENT_REQUESTS       -- All submitted requests (queue + history)
FND_CONCURRENT_MANAGERS       -- Manager definitions
FND_REQUEST_SET_PROGRAMS      -- Programs in a request set
FND_CONCURRENT_QUEUES         -- Manager queues
```

---

## 3. Submitting a Concurrent Request

### Via UI
```
Navigator -> View -> Requests -> Submit New Request
Select program name -> Enter parameters -> Submit
Note the Request ID
```

### Via API (from PL/SQL)
```sql
DECLARE
  l_request_id NUMBER;
BEGIN
  -- Submit a concurrent request programmatically
  l_request_id := FND_REQUEST.SUBMIT_REQUEST(
    application => 'ONT',                    -- Application short name
    program     => 'OEXOEORD',               -- Program short name
    description => 'Sales Order Report',
    start_time  => NULL,                     -- NULL = run now
    sub_request => FALSE,
    argument1   => '101',                    -- Org ID
    argument2   => SYSDATE,                  -- Start date
    argument3   => SYSDATE + 30             -- End date
  );

  IF l_request_id = 0 THEN
    -- Failed to submit
    RAISE_APPLICATION_ERROR(-20001, 'Failed to submit: ' || FND_MESSAGE.GET);
  ELSE
    DBMS_OUTPUT.PUT_LINE('Submitted. Request ID: ' || l_request_id);
    COMMIT;  -- MUST commit after submitting!
  END IF;
END;
```

> **Critical**: Always COMMIT after `FND_REQUEST.SUBMIT_REQUEST`. Without commit, the request won't be picked up.

### Wait for Completion (in PL/SQL)
```sql
DECLARE
  l_phase      VARCHAR2(30);
  l_status     VARCHAR2(30);
  l_dev_phase  VARCHAR2(30);
  l_dev_status VARCHAR2(30);
  l_message    VARCHAR2(2000);
  l_complete   BOOLEAN;
BEGIN
  -- Wait for request to complete (max 300 seconds, check every 5 sec)
  l_complete := FND_CONCURRENT.WAIT_FOR_REQUEST(
    request_id => l_request_id,
    interval   => 5,       -- Check every 5 seconds
    max_wait   => 300,     -- Give up after 5 minutes
    phase      => l_phase,
    status     => l_status,
    dev_phase  => l_dev_phase,
    dev_status => l_dev_status,
    message    => l_message
  );

  IF l_dev_status = 'NORMAL' THEN
    DBMS_OUTPUT.PUT_LINE('Request completed successfully');
  ELSE
    DBMS_OUTPUT.PUT_LINE('Request failed: ' || l_message);
  END IF;
END;
```

---

## 4. Monitor Concurrent Requests

### Via UI
```
Navigator -> View -> Requests
Filter by: Your requests / All requests / Pending / Running
Click Details -> View Log / View Output
```

### Via SQL
```sql
-- Check request status
SELECT fcr.request_id,
       fcr.requested_start_date,
       fcr.actual_start_date,
       fcr.actual_completion_date,
       fcr.phase_code,    -- P=Pending, R=Running, C=Completed, I=Inactive
       fcr.status_code,   -- N=Normal, E=Error, W=Warning, H=Hold, C=Cancelled
       fcr.logfile_name,
       fcr.outfile_name,
       fcp.user_concurrent_program_name,
       fu.user_name
FROM fnd_concurrent_requests fcr
JOIN fnd_concurrent_programs_tl fcp ON fcr.concurrent_program_id = fcp.concurrent_program_id
JOIN fnd_user fu ON fcr.requested_by = fu.user_id
WHERE fcr.request_id = 12345;

-- Find stuck/long-running requests
SELECT request_id, concurrent_program_id,
       actual_start_date,
       ROUND((SYSDATE - actual_start_date) * 24 * 60, 1) AS running_minutes
FROM fnd_concurrent_requests
WHERE phase_code = 'R'      -- Running
AND status_code = 'R'       -- Running status
ORDER BY actual_start_date;

-- Find errored requests from last 24 hours
SELECT request_id, logfile_name, completion_text
FROM fnd_concurrent_requests
WHERE status_code = 'E'     -- Error
AND actual_completion_date > SYSDATE - 1
ORDER BY actual_completion_date DESC;
```

---

## 5. Creating a Concurrent Program

### Step 1: Create the PL/SQL Procedure
```sql
CREATE OR REPLACE PROCEDURE process_open_orders (
  errbuf   OUT VARCHAR2,    -- REQUIRED: error message output
  retcode  OUT NUMBER,      -- REQUIRED: 0=success, 1=warning, 2=error
  p_org_id IN  NUMBER,
  p_from_date IN VARCHAR2
) AS
  l_from_date DATE;
BEGIN
  l_from_date := TO_DATE(p_from_date, 'YYYY/MM/DD HH24:MI:SS');

  -- Your logic here
  UPDATE oe_order_headers_all
  SET flow_status_code = 'PROCESSED'
  WHERE org_id = p_org_id
  AND creation_date >= l_from_date;

  COMMIT;
  retcode := 0;                          -- 0 = success
  errbuf := 'Processed successfully';

EXCEPTION
  WHEN OTHERS THEN
    retcode := 2;                         -- 2 = error
    errbuf := 'Error: ' || SQLERRM;
    ROLLBACK;
END process_open_orders;
```

> **Critical**: First two parameters MUST be `errbuf OUT VARCHAR2` and `retcode OUT NUMBER`. EBS populates these in the log file.

### Step 2: Register in EBS UI
```
System Administrator -> Concurrent -> Program -> Define
  Program: CUSTOM_PROCESS_ORDERS
  User Name: Process Open Orders
  Executable: Type=PL/SQL Stored Procedure, Name=PROCESS_OPEN_ORDERS
  Parameters: Define p_org_id and p_from_date
```

### Step 3: Add to Request Group
```
System Administrator -> Security -> Responsibility -> Request
Add program to appropriate responsibility's request group
```

---

## 6. Concurrent Program Executable Types

| Type | Use for |
|------|---------|
| **PL/SQL Stored Procedure** | Most common - database logic |
| **SQL*Plus** | Simple SQL scripts |
| **Shell Script** | Unix/Linux shell scripts |
| **Java** | Java programs |
| **Oracle Reports (RDF)** | Classic Oracle Reports |
| **XML Publisher/BI Publisher** | Modern report output |
| **Request Set Stage Function** | Conditional logic in sets |

---

## 7. Request Sets

> Like a playlist of songs - run programs in sequence, optionally with conditions.

### Types of Stage Linking
- **Serial**: Run stage B only after stage A completes
- **Parallel**: Run stages A and B simultaneously
- **Conditional**: Run stage B only if stage A succeeded (`On Success`), failed (`On Error`), or always

### Creating a Request Set
```
System Administrator -> Concurrent -> Sets -> Define
  Set Name: MONTH_END_CLOSE
  Stage 1: GENERATE_INVOICES
  Stage 2: POST_TO_GL (On Success of Stage 1)
  Stage 3: SEND_REPORT (Always)
```

---

## 8. Concurrent Manager Troubleshooting

### Manager Not Running
```sql
-- Check manager status
SELECT concurrent_queue_name, running_processes, max_processes,
       manager_type
FROM fnd_concurrent_queues_v
WHERE manager_type IN ('C', 'A')   -- Concurrent, Advanced
ORDER BY concurrent_queue_name;
```

```bash
# Restart concurrent manager from OS level
cd $ADMIN_SCRIPTS_HOME
./adcmctl.sh stop apps/apps_password
./adcmctl.sh start apps/apps_password
```

### Request Stuck in PENDING
- Manager not running -> restart CM
- Incompatibility -> check `fnd_concurrent_program_serial`
- Worker busy -> check active workers count
- No available worker -> increase workers in CM definition

### Common Status Codes
| Phase | Status | Meaning |
|-------|--------|---------|
| C (Complete) | N | Normal - success |
| C | E | Error |
| C | W | Warning |
| R (Running) | R | Running |
| P (Pending) | I | Scheduled for future |
| P | S | Waiting for another request |
| P | H | On Hold |
| I (Inactive) | D | Disabled |

---

## 9. Log and Output Files

```bash
# Location varies: check FND_CONCURRENT_REQUESTS.LOGFILE_NAME
# Common path pattern:
$APPLCSF/log/                    # Log files
$APPLCSF/out/                    # Output files

# From DB: get the path
SELECT logfile_node_name || ':' || logfile_name,
       outfile_node_name || ':' || outfile_name
FROM fnd_concurrent_requests
WHERE request_id = 12345;
```

---

## 10. Using FND_FILE for Logging in Concurrent Programs

```sql
CREATE OR REPLACE PROCEDURE my_program (
  errbuf  OUT VARCHAR2,
  retcode OUT NUMBER,
  p_org_id IN NUMBER
) AS
BEGIN
  -- Write to log file (goes to .req log file)
  FND_FILE.PUT_LINE(FND_FILE.LOG, 'Starting processing for Org: ' || p_org_id);

  -- Write to output file (printable output)
  FND_FILE.PUT_LINE(FND_FILE.OUTPUT, 'Order ID, Status, Amount');

  FOR r IN (SELECT order_id, status, amount FROM oe_order_headers_all WHERE org_id = p_org_id)
  LOOP
    FND_FILE.PUT_LINE(FND_FILE.OUTPUT, r.order_id || ',' || r.status || ',' || r.amount);
    FND_FILE.PUT_LINE(FND_FILE.LOG, 'Processed: ' || r.order_id);
  END LOOP;

  retcode := 0;
  errbuf := 'Complete';
EXCEPTION
  WHEN OTHERS THEN
    FND_FILE.PUT_LINE(FND_FILE.LOG, 'ERROR: ' || SQLERRM);
    retcode := 2;
    errbuf := SQLERRM;
END;
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What are the two mandatory parameters for a PL/SQL concurrent program?**
> `errbuf OUT VARCHAR2` and `retcode OUT NUMBER`. errbuf receives error/success message. retcode: 0=success, 1=warning, 2=error.

**Q2: What happens if you don't COMMIT after FND_REQUEST.SUBMIT_REQUEST?**
> The request is inserted into FND_CONCURRENT_REQUESTS but the Concurrent Manager won't pick it up because the row isn't committed. The request stays pending forever.

**Q3: How do you check if a concurrent program errored and see the log?**
> Query FND_CONCURRENT_REQUESTS where status_code = 'E'. Get logfile_name from that query. Or in UI: View > Requests > select request > View Log.

**Q4: What is the difference between errbuf and FND_FILE.LOG?**
> errbuf is a one-line summary that appears in the request status. FND_FILE.LOG writes detailed lines to the log file (viewable via View Log). For detailed tracing, always use FND_FILE.PUT_LINE.

**Q5: How do you run concurrent programs from PL/SQL?**
> Use `FND_REQUEST.SUBMIT_REQUEST(application, program, ...)`. Capture the returned request_id. Commit immediately. Optionally use `FND_CONCURRENT.WAIT_FOR_REQUEST` to wait for completion.

**Q6: What is a Request Set?**
> A collection of concurrent programs run in sequence or parallel with conditional logic. Stage B can run "On Success", "On Error", or "Always" relative to Stage A.

---

## 📝 Quick Revision Summary

```
Concurrent Program = background job (PL/SQL, Shell, Report)
  First 2 params: errbuf OUT VARCHAR2, retcode OUT NUMBER
  retcode: 0=success, 1=warning, 2=error
  FND_FILE.PUT_LINE(FND_FILE.LOG, msg)   -> log file
  FND_FILE.PUT_LINE(FND_FILE.OUTPUT, msg) -> output file

Submit: FND_REQUEST.SUBMIT_REQUEST(...) + COMMIT (must commit!)
Wait: FND_CONCURRENT.WAIT_FOR_REQUEST(request_id, ...)

Status codes:
  Phase: P=Pending, R=Running, C=Complete, I=Inactive
  Status: N=Normal, E=Error, W=Warning, H=Hold

Monitor:
  FND_CONCURRENT_REQUESTS -> all submissions
  FND_CONCURRENT_QUEUES   -> manager status

Request Set = staged programs (serial/parallel/conditional)
```

---

*Next: [Chapter 18 - Flexfields (KFF & DFF)](18_oracle_ebs_flex_dff.md)*
