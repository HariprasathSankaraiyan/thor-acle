# Chapter 14: Advanced Shell Scripting - Cron, Process Management, Traps, Logging
## ⏱️ ~25 min read

---

## 🧠 Think of it this way

> **Cron** = Your housekeeper's schedule: "Every Monday 8 AM, clean the house. Every Friday 6 PM, take out trash."
>
> **Traps** = Smoke detector: "If anything goes wrong (fire/error/Ctrl+C), do THIS first before exiting."
>
> **Process management** = A restaurant manager tracking which chef is doing what and being able to stop/restart them.

---

## 1. Cron - Scheduled Jobs

### Crontab Syntax
```
┌───── Minute (0-59)
│ ┌────── Hour (0-23)
│ │ ┌──────── Day of month (1-31)
│ │ │ ┌──────────── Month (1-12)
│ │ │ │ ┌──────────────── Day of week (0-7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command_to_run
```

### Crontab Examples
```bash
# Edit crontab for current user
crontab -e

# List current crontab
crontab -l

# Remove crontab
crontab -r

# Common schedules:
# Every minute
* * * * * /script/check.sh

# Every hour at minute 0
0 * * * * /script/hourly.sh

# Every day at 2:30 AM
30 2 * * * /script/daily_backup.sh

# Every Monday at 8 AM
0 8 * * 1 /script/weekly_report.sh

# Every weekday (Mon-Fri) at 6 PM
0 18 * * 1-5 /script/end_of_day.sh

# First day of every month at midnight
0 0 1 * * /script/monthly_cleanup.sh

# Every 15 minutes
*/15 * * * * /script/health_check.sh

# Every 6 hours
0 */6 * * * /script/refresh.sh

# Multiple times: 9 AM and 5 PM daily
0 9,17 * * * /script/twice_daily.sh
```

### Cron Best Practices
```bash
# Always use full paths in cron (cron has minimal PATH)
0 2 * * * /usr/local/bin/python3 /home/oracle/scripts/export.py

# Redirect output to a log file
30 2 * * * /scripts/backup.sh >> /var/log/backup.log 2>&1

# Suppress all output (sends to /dev/null)
0 * * * * /scripts/check.sh > /dev/null 2>&1

# Set environment in crontab (top of file)
ORACLE_HOME=/u01/app/oracle/product/19c
PATH=/usr/local/bin:/usr/bin:/bin:$ORACLE_HOME/bin
MAIL=dba@company.com

# For Oracle DBA: switch to oracle user
0 2 * * * su - oracle -c "/home/oracle/scripts/backup.sh"
# OR with crontab of oracle user: sudo crontab -u oracle -e
```

---

## 2. Process Management

```bash
# View processes
ps -ef                          # All processes (full format)
ps aux                          # All processes (BSD format)
ps -ef | grep oracle | grep -v grep
pgrep oracle                    # PIDs matching name
pgrep -a oracle                 # PIDs + command line

# Process tree
pstree -p oracle                # Tree view

# Real-time monitoring
top                             # Live CPU/memory per process
top -u oracle                   # Only oracle user's processes
top -p 12345                    # Specific PID

# Send signals
kill 12345                      # SIGTERM (15) - graceful shutdown
kill -9 12345                   # SIGKILL - force kill (no cleanup)
kill -HUP 12345                 # SIGHUP - reload config
kill -TERM 12345                # Same as kill (graceful)
killall oracle                  # Kill all processes named 'oracle'
pkill -u oracle                 # Kill all processes by user 'oracle'

# Background jobs
./long_script.sh &              # Start in background
nohup ./long_script.sh &        # Continue even after logout
nohup ./script.sh > output.log 2>&1 &

# Job control
jobs                            # List background jobs
fg %1                           # Bring job 1 to foreground
bg %1                           # Resume stopped job in background
Ctrl+Z                          # Pause current process
Ctrl+C                          # Terminate current process

# Wait for process
wait 12345                      # Wait for specific PID to complete
wait                            # Wait for all background jobs
```

---

## 3. Traps - Cleanup on Exit/Error

> "No matter what happens - whether the script finishes normally, gets Ctrl+C'd, or crashes - make sure you clean up temp files and send a log."

```bash
#!/bin/bash
set -euo pipefail

TEMP_FILE="/tmp/export_$$"           # $$ = current process ID
LOG_FILE="/var/log/export.log"

# Cleanup function
cleanup() {
  local exit_code=$?
  echo "Cleaning up temp file: $TEMP_FILE"
  rm -f "$TEMP_FILE"

  if [ $exit_code -ne 0 ]; then
    echo "Script FAILED with exit code: $exit_code" >> "$LOG_FILE"
    # Send alert email
    echo "Export failed!" | mail -s "ALERT: Export failed" dba@company.com
  else
    echo "Script completed successfully" >> "$LOG_FILE"
  fi
}

# Register the cleanup function
trap cleanup EXIT            # Run cleanup on any exit

# Handle specific signals
trap 'echo "Interrupted! Cleaning up..."; cleanup; exit 130' INT    # Ctrl+C
trap 'echo "Terminated! Cleaning up..."; cleanup; exit 143' TERM   # kill

# Your actual script logic
echo "Starting export..."
cp large_data.csv "$TEMP_FILE"
process_data "$TEMP_FILE"
# If anything fails here, cleanup() will still run
echo "Done!"
```

---

## 4. Logging Best Practices

```bash
#!/bin/bash

LOG_FILE="/var/log/app/export_$(date +%Y%m%d).log"
LOCK_FILE="/tmp/export.lock"

# Logging function
log() {
  local level="$1"
  local message="$2"
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$$] [$level] $message" | tee -a "$LOG_FILE"
}

# Prevent duplicate runs (lock file)
acquire_lock() {
  if [ -f "$LOCK_FILE" ]; then
    local pid=$(cat "$LOCK_FILE")
    if kill -0 "$pid" 2>/dev/null; then
      log "ERROR" "Another instance is running (PID: $pid)"
      exit 1
    fi
    log "WARNING" "Stale lock file found. Removing."
    rm -f "$LOCK_FILE"
  fi
  echo $$ > "$LOCK_FILE"
  log "INFO" "Lock acquired (PID: $$)"
}

release_lock() {
  rm -f "$LOCK_FILE"
  log "INFO" "Lock released"
}

trap release_lock EXIT

# Log rotation
find /var/log/app -name "*.log" -mtime +30 -delete    # Delete logs older than 30 days

# Log levels
log "INFO"    "Script started"
log "WARNING" "Disk at 78%, approaching threshold"
log "ERROR"   "Database connection failed"
log "DEBUG"   "Processing order ID: 1001"
```

---

## 5. Here Documents (Heredoc) - Oracle Scripts

```bash
#!/bin/bash
# Run SQL*Plus commands from shell script

ORACLE_HOME=/u01/app/oracle/product/19c
ORACLE_SID=PROD
export ORACLE_HOME ORACLE_SID

# Method 1: SQL in heredoc
sqlplus -s / as sysdba <<EOF
SET HEADING OFF FEEDBACK OFF PAGESIZE 0
SELECT COUNT(*) FROM dba_objects WHERE status = 'INVALID';
EXIT;
EOF

# Method 2: Capture SQL output
invalid_count=$(sqlplus -s / as sysdba <<EOF
SET HEADING OFF FEEDBACK OFF PAGESIZE 0
SELECT COUNT(*) FROM dba_objects WHERE status = 'INVALID';
EXIT;
EOF
)
echo "Invalid objects: $invalid_count"

# Method 3: Variable in SQL (be careful with SQL injection in scripts)
table_name="ORDERS"
row_count=$(sqlplus -s "$DB_USER/$DB_PASS@$DB_SID" <<EOF
SET HEADING OFF FEEDBACK OFF PAGESIZE 0
SELECT COUNT(*) FROM $table_name;
EXIT;
EOF
)
echo "Rows in $table_name: $row_count"

# Method 4: Spool to file
sqlplus -s "$DB_USER/$DB_PASS@$DB_SID" <<EOF
SET LINESIZE 200 PAGESIZE 1000
SPOOL /tmp/report.csv
SET COLSEP ','
SELECT order_id, customer_id, amount FROM orders WHERE status='OPEN';
SPOOL OFF
EXIT;
EOF
```

---

## 6. Parallel Execution in Scripts

```bash
#!/bin/bash

# Run multiple jobs in parallel
process_chunk() {
  local start=$1
  local end=$2
  echo "Processing $start to $end..."
  sleep 5  # Simulate work
  echo "Done: $start to $end"
}

# Launch 4 jobs in parallel
process_chunk 1 100 &
process_chunk 101 200 &
process_chunk 201 300 &
process_chunk 301 400 &

# Wait for all to complete
wait
echo "All chunks complete!"

# With xargs for parallel execution
echo -e "2023-01\n2023-02\n2023-03\n2023-04" | \
  xargs -P 4 -I{} bash -c 'echo "Processing month: {}"; sleep 2'
```

---

## 7. Arrays in Bash

```bash
# Declare and initialize
servers=("db01" "db02" "app01" "app02")
declare -a months=("Jan" "Feb" "Mar" "Apr")

# Access
echo ${servers[0]}              # First element: db01
echo ${servers[-1]}             # Last element: app02
echo ${#servers[@]}             # Length: 4
echo ${servers[@]}              # All elements

# Loop over array
for server in "${servers[@]}"; do
  echo "Checking server: $server"
  ping -c 1 "$server" > /dev/null && echo "$server: UP" || echo "$server: DOWN"
done

# Add element
servers+=("db03")

# Associative array (key-value)
declare -A config
config["db_host"]="dr.rds.example.com"
config["db_port"]="5432"
config["db_name"]="mydb"

echo ${config["db_host"]}
for key in "${!config[@]}"; do
  echo "$key = ${config[$key]}"
done
```

---

## 8. Script Template (Production-Ready)

```bash
#!/bin/bash
# =============================================================================
# Script: oracle_health_check.sh
# Purpose: Check Oracle DB health and send alerts
# Usage: ./oracle_health_check.sh [--threshold 80] [--db PROD]
# =============================================================================
set -euo pipefail

# --- Configuration ---
SCRIPT_NAME=$(basename "$0")
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_DIR="/var/log/oracle_checks"
LOG_FILE="${LOG_DIR}/${SCRIPT_NAME%.sh}_$(date +%Y%m%d).log"
LOCK_FILE="/tmp/${SCRIPT_NAME%.sh}.lock"
THRESHOLD=80
DB_SID="PROD"

# --- Parse Arguments ---
while [[ $# -gt 0 ]]; do
  case "$1" in
    --threshold) THRESHOLD="$2"; shift 2 ;;
    --db)        DB_SID="$2"; shift 2 ;;
    --help)      echo "Usage: $0 [--threshold N] [--db SID]"; exit 0 ;;
    *) echo "Unknown option: $1"; exit 1 ;;
  esac
done

# --- Logging ---
mkdir -p "$LOG_DIR"
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$1] $2" | tee -a "$LOG_FILE"; }

# --- Lock ---
[[ -f "$LOCK_FILE" ]] && { log "ERROR" "Already running"; exit 1; }
echo $$ > "$LOCK_FILE"
trap 'rm -f "$LOCK_FILE"; log "INFO" "Script ended"' EXIT

# --- Main ---
log "INFO" "Starting health check for $DB_SID (threshold: ${THRESHOLD}%)"

# Check tablespace
ts_usage=$(sqlplus -s "/ as sysdba" <<SQL 2>&1
SET HEADING OFF FEEDBACK OFF PAGESIZE 0
SELECT MAX(used_percent) FROM dba_tablespace_usage_metrics;
EXIT;
SQL
)

if (( $(echo "$ts_usage > $THRESHOLD" | bc -l) )); then
  log "WARNING" "Tablespace usage ${ts_usage}% exceeds threshold ${THRESHOLD}%"
fi

log "INFO" "Health check complete"
```

---

## 🎯 Top Interview Questions & Answers

**Q1: Write a cron job that runs a script every day at 2:30 AM.**
> `30 2 * * * /home/oracle/scripts/backup.sh >> /var/log/backup.log 2>&1`

**Q2: What is a trap in shell scripting?**
> `trap` registers commands to run when the script receives a signal or exits. `trap cleanup EXIT` ensures cleanup runs no matter how the script ends (success, error, Ctrl+C).

**Q3: What is `nohup` used for?**
> `nohup` makes a process immune to the HUP signal (sent when terminal closes). Use `nohup ./script.sh &` to keep a background job running after SSH logout.

**Q4: How do you prevent a script from running multiple instances at the same time?**
> Use a lock file. Create a file at startup with the PID, check if it exists before running, remove it on exit with `trap`. If lock exists but PID is dead, it's stale - remove it.

**Q5: How do you pass variables to a SQL*Plus heredoc?**
> Shell variables expand inside heredoc. Use `$VAR_NAME` directly in the SQL. Example: `SELECT COUNT(*) FROM $TABLE_NAME;` where `TABLE_NAME` is set in bash.

**Q6: What is `$$` in shell scripting?**
> `$$` is the PID (Process ID) of the current shell/script. Used to create unique temp files: `TEMP_FILE="/tmp/data_$$"` ensures uniqueness even if multiple instances run.

---

## 📝 Quick Revision Summary

```
Cron syntax: MIN HOUR DOM MON DOW command
  0 2 * * *  -> daily at 2 AM
  */15 * * * * -> every 15 minutes
  0 9,17 * * 1-5 -> 9 AM and 5 PM weekdays

Always use full paths in cron
Redirect: >> log 2>&1

trap 'cleanup' EXIT  -> always runs on exit
trap 'exit 130' INT  -> handle Ctrl+C
$$ = current PID -> unique temp files

Lock file pattern:
  Check if lock exists -> quit if running
  Create lock with PID -> do work -> remove on EXIT trap

Heredoc SQL*Plus:
  sqlplus -s user/pass <<EOF ... EXIT; EOF
  Capture: result=$(sqlplus -s ... <<EOF ... EOF)

nohup ./script.sh & -> survive logout
wait -> wait for all background jobs
xargs -P 4 -> parallel execution
```

---

*Next: [Chapter 15 - Linux Networking & Troubleshooting](15_linux_networking_troubleshoot.md)*
