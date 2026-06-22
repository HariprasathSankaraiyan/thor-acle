# Chapter 13: Shell Scripting Basics
## ⏱️ ~25 min read

---

## 🧠 Think of it this way

> A shell script is a **to-do list for your computer**.
> Instead of you typing 20 commands manually every morning,
> you write them all in a file, run it once, and go have coffee.
>
> "Every night at 11 PM, check disk space. If > 80%, email the team. Then clean old logs. Then back up Oracle."
> That's a shell script.

---

## 1. Script Basics - Shebang, Comments, Execution

```bash
#!/bin/bash
# This is the shebang - tells OS "use bash to run this file"
# Lines starting with # are comments (except shebang)

# Script to demonstrate basics
echo "Hello World"
echo "Today is: $(date)"

# To make script executable and run:
# chmod +x script.sh
# ./script.sh
# OR: bash script.sh
```

---

## 2. Variables

```bash
#!/bin/bash

# Assign (NO SPACES around =)
name="Oracle DBA"
count=10
today=$(date +%Y-%m-%d)      # Capture command output
today=`date +%Y-%m-%d`       # Older syntax (backticks)

# Use variable (use $variable or ${variable})
echo "Name: $name"
echo "Count: ${count}"
echo "Today: $today"

# Read-only (constant)
readonly MAX_RETRIES=5

# Unset
unset name

# Check if variable is set
echo ${name:-"default_value"}    # Use default if name is unset
echo ${name:="default_value"}    # Set default if name is unset

# String length
echo ${#name}                    # Length of string

# Uppercase/lowercase
echo ${name^^}                   # ALL CAPS
echo ${name,,}                   # all lowercase
```

---

## 3. Command Line Arguments

```bash
#!/bin/bash
# Usage: ./script.sh arg1 arg2 arg3

echo "Script name: $0"
echo "First arg:   $1"
echo "Second arg:  $2"
echo "All args:    $@"          # All arguments as separate words
echo "All args:    $*"          # All arguments as single string
echo "Arg count:   $#"          # Number of arguments

# Check if required arguments provided
if [ $# -lt 2 ]; then
  echo "Usage: $0 <start_date> <end_date>"
  exit 1
fi

START_DATE=$1
END_DATE=$2
echo "Exporting from $START_DATE to $END_DATE"
```

---

## 4. Conditions - if/elif/else

```bash
#!/bin/bash

# String comparison
status="OPEN"
if [ "$status" = "OPEN" ]; then
  echo "Order is open"
elif [ "$status" = "CLOSED" ]; then
  echo "Order is closed"
else
  echo "Unknown status: $status"
fi

# Numeric comparison
count=15
if [ $count -gt 10 ]; then          # gt = greater than
  echo "More than 10"
fi

# Numeric comparison operators:
# -eq  = equal
# -ne  = not equal
# -lt  = less than
# -le  = less than or equal
# -gt  = greater than
# -ge  = greater than or equal

# File tests
if [ -f "/etc/oracle/listener.ora" ]; then
  echo "File exists"
fi
if [ -d "/app/oracle" ]; then
  echo "Directory exists"
fi
if [ -r "data.csv" ]; then
  echo "File is readable"
fi
if [ -x "script.sh" ]; then
  echo "Script is executable"
fi
if [ -s "logfile.log" ]; then
  echo "File is non-empty"
fi

# String tests
if [ -z "$var" ]; then              # -z = empty string
  echo "Variable is empty"
fi
if [ -n "$var" ]; then              # -n = non-empty string
  echo "Variable has value"
fi

# Multiple conditions
if [ $count -gt 5 ] && [ $count -lt 20 ]; then
  echo "Between 5 and 20"
fi
if [ "$status" = "OPEN" ] || [ "$status" = "PENDING" ]; then
  echo "Active order"
fi

# Double bracket [[ ]] - more powerful (bash only)
if [[ "$name" == *"Oracle"* ]]; then   # Pattern matching with *
  echo "Contains Oracle"
fi
if [[ "$var" =~ ^[0-9]+$ ]]; then      # Regex matching
  echo "Numeric value"
fi
```

---

## 5. Loops

### For Loop
```bash
# Numeric range
for i in {1..10}; do
  echo "Iteration: $i"
done

# C-style for loop
for ((i=1; i<=10; i++)); do
  echo "Count: $i"
done

# Loop over list
for color in red green blue; do
  echo "Color: $color"
done

# Loop over files
for file in /var/log/oracle/*.log; do
  echo "Processing: $file"
  wc -l "$file"
done

# Loop over command output
for user in $(cat /etc/passwd | cut -d: -f1); do
  echo "User: $user"
done
```

### While Loop
```bash
# Basic while
count=1
while [ $count -le 5 ]; do
  echo "Count: $count"
  ((count++))
done

# Read file line by line
while IFS= read -r line; do
  echo "Processing: $line"
done < input_file.txt

# Infinite loop with break
while true; do
  echo "Checking status..."
  status=$(check_job_status)
  if [ "$status" = "COMPLETE" ]; then
    break
  fi
  sleep 30
done
```

### Until Loop (opposite of while)
```bash
until [ $count -gt 10 ]; do
  echo $count
  ((count++))
done
```

---

## 6. Functions

```bash
#!/bin/bash

# Define function
log_message() {
  local level="$1"      # 'local' = scope limited to this function
  local message="$2"
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message"
}

# Function with return value (return only returns exit code 0-255!)
get_row_count() {
  local table=$1
  local count=$(sqlplus -s / as sysdba <<EOF
SET HEADING OFF FEEDBACK OFF
SELECT COUNT(*) FROM $table;
EXIT;
EOF
)
  echo "$count"    # Return via echo, capture with $()
}

# Function that checks exit status
check_disk_space() {
  local threshold=${1:-80}   # Default 80% if not provided
  local usage=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

  if [ "$usage" -ge "$threshold" ]; then
    log_message "WARNING" "Disk usage is ${usage}% (threshold: ${threshold}%)"
    return 1    # Non-zero = failure
  fi
  log_message "INFO" "Disk usage OK: ${usage}%"
  return 0    # 0 = success
}

# Call functions
log_message "INFO" "Script started"

row_count=$(get_row_count "ORDERS")
echo "Orders count: $row_count"

if ! check_disk_space 85; then
  echo "ALERT: Disk space critical!"
fi
```

---

## 7. Exit Codes and Error Handling

```bash
#!/bin/bash

# Exit code of last command: $?
ls /nonexistent_dir
echo "Exit code: $?"    # -> 2 (non-zero = failure)

ls /tmp
echo "Exit code: $?"    # -> 0 (success)

# Exit script with specific code
exit 0    # Success
exit 1    # General error
exit 2    # Misuse of shell command

# set -e: exit immediately on any error
set -e

# set -u: treat unset variables as errors
set -u

# set -o pipefail: fail if any command in pipe fails
set -o pipefail

# Best practice at top of scripts:
set -euo pipefail

# Trap errors
trap 'echo "Error on line $LINENO"; exit 1' ERR

# Trap on exit (always runs)
trap 'echo "Script finished"; cleanup' EXIT
```

---

## 8. Reading Input

```bash
# Read from user (interactive)
read -p "Enter start date (YYYY-MM-DD): " start_date
echo "You entered: $start_date"

# Read with timeout
read -t 10 -p "Proceed? [y/N]: " answer

# Read password (hidden input)
read -s -p "Enter DB password: " db_pass
echo ""   # Newline after hidden input

# Read from file
while IFS=',' read -r id name amount; do
  echo "Order $id: $name, Amount: $amount"
done < orders.csv
```

---

## 9. Strings and Arithmetic

```bash
# String operations
str="Hello World"
echo ${#str}                   # Length: 11
echo ${str:0:5}                # Substring: "Hello"
echo ${str:6}                  # From pos 6: "World"
echo ${str/World/Oracle}       # Replace first: "Hello Oracle"
echo ${str//l/L}               # Replace all: "HeLLo WorLd"
echo ${str#Hello }             # Remove prefix: "World"
echo ${str%World}              # Remove suffix: "Hello "

# Arithmetic
a=10
b=3
echo $((a + b))        # 13
echo $((a - b))        # 7
echo $((a * b))        # 30
echo $((a / b))        # 3 (integer division)
echo $((a % b))        # 1 (modulo)
echo $((a ** b))       # 1000 (power)

((a++))                # Increment
((a+=5))               # Add 5

# bc for floating point
echo "scale=2; 10/3" | bc      # -> 3.33
```

---

## 10. Oracle DBA Script Example (Real-World)

```bash
#!/bin/bash
# Oracle tablespace check script
set -euo pipefail

ORACLE_USER="scott"
ORACLE_PASS="tiger"
ORACLE_SID="PROD"
THRESHOLD=85
LOG_FILE="/var/log/oracle_checks/$(date +%Y%m%d).log"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

check_tablespace() {
  log "Checking tablespace usage..."

  sqlplus -s "$ORACLE_USER/$ORACLE_PASS@$ORACLE_SID" <<EOF 2>&1 | tee -a "$LOG_FILE"
SET LINESIZE 100 PAGESIZE 50 FEEDBACK OFF HEADING ON
SELECT tablespace_name,
       ROUND(used_percent, 1) used_pct
FROM dba_tablespace_usage_metrics
WHERE used_percent > $THRESHOLD
ORDER BY used_percent DESC;
EXIT;
EOF

  if [ $? -ne 0 ]; then
    log "ERROR: Failed to query Oracle"
    exit 1
  fi
}

log "=== Oracle Health Check Started ==="
check_tablespace
log "=== Check Complete ==="
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What does `#!/bin/bash` mean?**
> The shebang line. Tells the OS which interpreter to use. `/bin/bash` means use bash. Can also be `/bin/sh`, `/usr/bin/python3`, etc.

**Q2: What is the difference between `$@` and `$*`?**
> Both contain all arguments. `$@` treats each argument as separate (preserves quotes/spaces). `$*` treats all as one string. Use `"$@"` in loops to safely pass arguments.

**Q3: Why use `local` inside functions?**
> `local` limits a variable's scope to the function. Without it, the variable is global and can accidentally overwrite outer variables with the same name.

**Q4: What does `set -euo pipefail` do?**
> `-e` = exit on error. `-u` = error on unset variables. `-o pipefail` = fail if any command in a pipe fails. Best practice for robust scripts.

**Q5: How do you return a value from a function?**
> `return` only returns exit code (0-255). To return a string/number, use `echo` inside the function and capture with `$()`: `result=$(my_function)`

**Q6: How do you read a file line by line in bash?**
> `while IFS= read -r line; do ... done < file.txt`. `IFS=` preserves leading spaces. `-r` prevents backslash interpretation.

---

## 📝 Quick Revision Summary

```
Shebang: #!/bin/bash (first line)

Variables:
  name="value"   -> NO spaces around =
  $name or ${name}
  $1, $2, $# (args), $@ (all args), $0 (script name)

Conditions:
  [ "$var" = "value" ]        -> string equal
  [ $num -gt 5 ]              -> numeric: -eq -ne -lt -le -gt -ge
  [ -f file ] [ -d dir ] [ -z "$var" ]

Loops:
  for i in {1..10}; do ... done
  while [ condition ]; do ... done
  while IFS= read -r line; do ... done < file

Functions:
  func_name() { local var=val; echo "result"; }
  capture: result=$(func_name)

Error handling:
  set -euo pipefail  (top of script)
  $? = exit code of last command
  exit 0 = success, exit 1 = failure

Arithmetic: $((a + b))
String ops: ${#str} ${str:0:5} ${str/old/new}
```

---

*Next: [Chapter 14 - Advanced Shell Scripting](14_shell_scripting_advanced.md)*
