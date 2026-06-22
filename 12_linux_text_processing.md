# Chapter 12: Linux Text Processing - grep, awk, sed, cut, sort, uniq
## ⏱️ ~25 min read

---

## 🧠 Think of it this way

> Think of text processing tools as **different workers** at a document processing desk:
> - **grep** = the searcher: "Find all documents mentioning Oracle"
> - **awk** = the analyst: "For each line, extract column 3 and calculate total"
> - **sed** = the editor: "In every document, replace 'ERROR' with 'ISSUE'"
> - **cut** = the scissor: "Cut out just columns 1-5 from each line"
> - **sort** = the organizer: "Sort these documents alphabetically"
> - **uniq** = the deduplicator: "Remove duplicate entries"

---

## 1. grep - Search Text

```bash
# Basic search
grep "error" logfile.txt               # Case-sensitive search
grep -i "error" logfile.txt            # Case-INsensitive
grep -r "ORA-" /var/log/oracle/        # Recursive (search in directory)
grep -n "error" logfile.txt            # Show line numbers
grep -v "INFO" logfile.txt             # Invert: show lines NOT matching
grep -c "error" logfile.txt            # Count matching lines

# Multiple patterns
grep -E "error|warning|critical" logfile.txt  # Extended regex (OR)
grep -e "error" -e "warning" logfile.txt       # Multiple patterns

# Context lines
grep -A 3 "ORA-01555" logfile.txt      # 3 lines AFTER match
grep -B 3 "ORA-01555" logfile.txt      # 3 lines BEFORE match
grep -C 3 "ORA-01555" logfile.txt      # 3 lines BOTH sides

# Word boundary
grep -w "error" logfile.txt            # Exact word (not "errorlog")

# Show only matching part
grep -o "ORA-[0-9]*" logfile.txt       # Show only the matched text

# Files only / line count
grep -l "ORA-" /var/log/oracle/*.log   # Show filenames that match
grep -rl "ORA-" /var/log/             # Recursive, filenames only

# Combine with other commands
ps -ef | grep oracle | grep -v grep
tail -100 alert.log | grep -i "error"
```

---

## 2. awk - Column Processing & Calculation

> awk reads each line, splits it into columns ($1, $2 ...), and lets you process/calculate.

```bash
# Print specific columns
awk '{print $1}' file.txt              # Print column 1
awk '{print $1, $3}' file.txt         # Print columns 1 and 3
awk '{print NR, $0}' file.txt         # Add line numbers (NR=row number)
awk '{print NF}' file.txt             # Print number of fields per line

# Custom delimiter
awk -F: '{print $1, $3}' /etc/passwd  # Colon-delimited
awk -F, '{print $2}' data.csv         # CSV: print second column
awk -F'|' '{print $1}' file.txt       # Pipe-delimited

# Conditions (like WHERE)
awk '$3 > 1000 {print $0}' file.txt   # Lines where col 3 > 1000
awk '/ERROR/ {print $0}' logfile.txt  # Lines matching regex

# Calculations
awk '{sum += $3} END {print "Total:", sum}' amounts.txt
awk '{sum += $3; count++} END {print "Average:", sum/count}' amounts.txt
awk 'NR > 1 {print}' file.txt         # Skip header (first line)

# BEGIN and END blocks
awk 'BEGIN {print "=== Report ==="} 
     {sum += $2} 
     END {print "Total:", sum}' data.txt

# Real-world examples
# Count lines per log level
awk '{print $5}' app.log | sort | uniq -c | sort -rn

# Find lines with amount > 5000 in CSV
awk -F, '$4 > 5000 {print $1, $4}' orders.csv

# Print filename and line for Oracle errors
awk '/ORA-/{print FILENAME ":" NR ": " $0}' *.log
```

---

## 3. sed - Stream Editor (Find and Replace)

```bash
# Basic substitution
sed 's/old/new/' file.txt             # Replace FIRST occurrence per line
sed 's/old/new/g' file.txt            # Replace ALL occurrences (global)
sed 's/old/new/gi' file.txt           # Case-insensitive global replace
sed 's/ERROR/ISSUE/g' logfile.txt

# Edit in-place (modify the file itself)
sed -i 's/ERROR/ISSUE/g' logfile.txt          # Modify file directly
sed -i.bak 's/ERROR/ISSUE/g' logfile.txt      # Modify + keep backup

# Delete lines
sed '/pattern/d' file.txt             # Delete lines matching pattern
sed '5d' file.txt                     # Delete line 5
sed '5,10d' file.txt                  # Delete lines 5 to 10
sed '/^$/d' file.txt                  # Delete blank lines
sed '/^#/d' config.txt                # Delete comment lines

# Print specific lines
sed -n '5,10p' file.txt               # Print lines 5-10 only
sed -n '/START/,/END/p' file.txt      # Print between START and END

# Multiple commands
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt
sed 's/foo/bar/g; s/baz/qux/g' file.txt

# Add line before/after match
sed '/pattern/i\NEW LINE BEFORE' file.txt
sed '/pattern/a\NEW LINE AFTER' file.txt

# Extract values
echo "order_id=1001" | sed 's/order_id=//'      # -> 1001
echo "2023-01-15" | sed 's/-//g'                # -> 20230115
```

---

## 4. cut - Extract Columns

```bash
# By character position
cut -c1-5 file.txt                    # Characters 1-5 of each line
cut -c1,5,10 file.txt                 # Characters 1, 5, and 10

# By delimiter and field
cut -d: -f1 /etc/passwd               # First field, colon-delimited
cut -d: -f1,3 /etc/passwd             # Fields 1 and 3
cut -d, -f2 orders.csv                # Second column in CSV
cut -d'|' -f1-3 data.txt             # Fields 1 through 3, pipe-delimited

# Practical examples
echo "oracle:x:500:500:" | cut -d: -f1   # -> oracle
date | cut -d' ' -f1                       # Day of week
```

---

## 5. sort - Sort Lines

```bash
sort file.txt                         # Alphabetical ascending
sort -r file.txt                      # Reverse (descending)
sort -n file.txt                      # Numeric sort
sort -rn file.txt                     # Numeric descending
sort -u file.txt                      # Sort and remove duplicates
sort -t: -k3 -n /etc/passwd           # Sort by field 3, colon-delimited
sort -t, -k2 -rn orders.csv           # Sort CSV by col 2, numeric desc
sort -k1,1 -k2,2n file.txt            # Sort by col1 asc, then col2 numeric

# Practical examples
# Top 10 largest files
du -sh /var/log/* | sort -rh | head -10

# Sort processes by memory
ps aux --sort=-%mem | head -10
```

---

## 6. uniq - Remove/Count Duplicates

> **Note**: `uniq` only removes ADJACENT duplicates. Always `sort` first!

```bash
sort file.txt | uniq                  # Remove duplicate lines
sort file.txt | uniq -c               # Count occurrences
sort file.txt | uniq -d               # Show only DUPLICATE lines
sort file.txt | uniq -u               # Show only UNIQUE (non-duplicate) lines

# Practical: count most common values
cut -d, -f3 orders.csv | sort | uniq -c | sort -rn | head -10
# -> "Count of each order status, sorted most common first"

# Real-world: most common errors in log
grep "ORA-" alert.log | awk '{print $NF}' | sort | uniq -c | sort -rn
```

---

## 7. tr - Translate/Replace Characters

```bash
tr 'a-z' 'A-Z' < file.txt            # Lowercase to uppercase
tr -d '\n' < file.txt                  # Remove newlines
tr -s ' ' < file.txt                   # Squeeze multiple spaces to one
echo "hello world" | tr ' ' '_'        # Replace spaces with underscore
echo "A,B,C" | tr ',' '\n'            # Replace commas with newlines
```

---

## 8. xargs - Pass Command Output as Arguments

```bash
find /tmp -name "*.log" | xargs rm          # Delete all .log files in /tmp
find /app -name "*.sh" | xargs chmod +x    # Make all .sh files executable
cat file_list.txt | xargs ls -la           # List files from a file list
echo "1001 1002 1003" | xargs -n1 echo "Processing order:"
# -> Processing order: 1001
# -> Processing order: 1002

# With delimiter
cat order_ids.txt | xargs -I{} echo "SELECT * FROM orders WHERE id={};"
```

---

## 9. Combining Commands - Pipelines (Real-World)

```bash
# Find top 5 largest files in /var/log
find /var/log -type f -exec ls -lh {} \; | sort -k5 -rh | head -5

# Count errors per hour in a log file
grep "ERROR" app.log | awk '{print substr($2,1,2)}' | sort | uniq -c

# Get distinct Oracle user connections in last 100 lines
tail -100 audit.log | grep "CONNECT" | awk '{print $5}' | sort -u

# Find disk hogs
du -sh /app/* 2>/dev/null | sort -rh | head -10

# Replace all passwords in config files (DANGEROUS example - dry run first!)
grep -rl "password=admin123" /etc/ | xargs sed -i 's/password=admin123/password=XXXXX/g'

# Monitor log in real-time for errors
tail -f /var/log/oracle/alert.log | grep -i "error\|ORA-"

# Process CSV and calculate sum
awk -F, 'NR>1 {sum+=$4} END {printf "Total Amount: %.2f\n", sum}' orders.csv
```

---

## 10. Regular Expressions Quick Reference

| Pattern | Matches |
|---------|---------|
| `.` | Any single character |
| `*` | Zero or more of preceding |
| `+` | One or more (use with -E) |
| `?` | Zero or one (use with -E) |
| `^` | Start of line |
| `$` | End of line |
| `[abc]` | Any of a, b, or c |
| `[^abc]` | Any NOT a, b, or c |
| `[0-9]` | Any digit |
| `[a-z]` | Any lowercase letter |
| `\b` | Word boundary |
| `\d` | Digit (Perl regex, use `[0-9]` in grep) |

```bash
# Examples
grep "^ORA-" file.txt          # Lines starting with ORA-
grep "ERROR$" file.txt         # Lines ending with ERROR
grep "ORA-[0-9]*" file.txt     # ORA- followed by any digits
grep -E "^[0-9]{4}-[0-9]{2}"  # Date format YYYY-MM
```

---

## 🎯 Top Interview Questions & Answers

**Q1: How do you search for a string recursively in all files?**
> `grep -r "search_string" /path/to/directory`
> Or: `grep -rl "string" /path/` (just filenames)

**Q2: How do you print the 3rd column of a space-delimited file?**
> `awk '{print $3}' file.txt`
> Or for different delimiter: `awk -F: '{print $3}' file.txt`

**Q3: How do you replace a string in a file in-place?**
> `sed -i 's/old_string/new_string/g' filename.txt`
> Use `sed -i.bak` to keep a backup.

**Q4: How do you find the top 5 most frequent values in a column?**
> `cut -d, -f2 file.csv | sort | uniq -c | sort -rn | head -5`

**Q5: What is the difference between grep and awk?**
> grep searches for patterns and shows matching lines. awk is a full processing language - it can split lines into fields, do arithmetic, apply conditions, and format output.

**Q6: How do you delete blank lines from a file?**
> `sed '/^$/d' file.txt` or `grep -v '^$' file.txt`

---

## 📝 Quick Revision Summary

```
grep "pattern" file     -> search
  -i = case insensitive
  -r = recursive
  -n = line numbers
  -v = invert (NOT matching)
  -A/-B/-C n = context lines

awk '{print $2}' file   -> process columns
  -F: = delimiter
  NR = row number, NF = field count
  BEGIN/END blocks
  $0 = whole line, $1..$n = fields

sed 's/old/new/g' file  -> find/replace
  -i = in-place edit
  /pattern/d = delete lines
  -n 'np' = print specific line

cut -d: -f1 file        -> extract column by delimiter
sort -rn file           -> sort (numeric descending)
sort | uniq -c          -> count occurrences
xargs                   -> pipe to arguments
```

---

*Next: [Chapter 13 - Shell Scripting Basics](13_shell_scripting_basics.md)*
