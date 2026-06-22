# Chapter 15: Linux Networking & Troubleshooting
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> Networking = postal system.
> - **IP address** = your home address
> - **Port** = apartment number (which app gets the letter)
> - **DNS** = phone book (converts name -> address)
> - **ping** = "Are you home?" (knock on door)
> - **telnet/netcat** = "Can you open the door?" (test the port)

---

## 1. Network Diagnostics Commands

```bash
# Check if a host is reachable
ping -c 4 dr.rds.example.com        # 4 pings then stop
ping -c 4 10.0.1.100

# DNS lookup (name -> IP)
nslookup dr.rds.example.com
dig dr.rds.example.com
host dr.rds.example.com
cat /etc/hosts                       # Local DNS overrides

# Trace network path (like GPS route)
traceroute dr.rds.example.com
tracepath dr.rds.example.com

# Check network interfaces
ifconfig                             # Old way
ip addr show                         # New way
ip a                                 # Short form

# Check routing table
route -n
ip route show

# Active network connections
netstat -an                          # All connections
netstat -an | grep 1521              # Oracle listener port
netstat -tlnp                        # Listening ports + process
ss -tlnp                             # Modern replacement for netstat
```

---

## 2. Port Testing (Critical for Oracle DBA)

```bash
# Test if Oracle listener port is reachable
telnet db-server 1521                # "Can I connect to port 1521?"
# Ctrl+] then 'quit' to exit

# nc (netcat) - more flexible
nc -zv db-server 1521                # -z=scan, -v=verbose
nc -zv db-server 1521-1525           # Test port range

# Check if local port is listening
netstat -an | grep 1521
ss -an | grep 1521

# Test HTTP endpoints (without curl)
(echo "HEAD / HTTP/1.1"; echo "Host: example.com"; echo "") | nc example.com 80

# Oracle tnsping (test TNS connectivity)
tnsping PROD                         # Test TNS alias
tnsping "(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=dbhost)(PORT=1521))(CONNECT_DATA=(SID=PROD)))"
```

---

## 3. File Transfer

```bash
# SCP (Secure Copy over SSH)
scp file.txt user@remote:/tmp/                    # Local -> Remote
scp user@remote:/tmp/file.txt /local/dir/          # Remote -> Local
scp -r /local/dir/ user@remote:/remote/dir/        # Directory copy

# rsync (incremental, smart copy)
rsync -avz /local/dir/ user@remote:/remote/dir/    # Sync with compression
rsync -avz --delete /local/ remote:/remote/        # Mirror (delete extra)
rsync -avz --dry-run /local/ remote:/remote/       # Dry run (preview)
```

---

## 4. System Performance Monitoring

```bash
# CPU and Memory
top                          # Live: CPU, memory per process
top -b -n1                   # One-time snapshot (good for scripting)
free -h                      # Memory usage (human-readable)
vmstat 5                     # Virtual memory stats every 5 seconds
mpstat 1                     # CPU stats per core
uptime                       # Load averages (1m, 5m, 15m)

# Load average interpretation:
# 1-core system: load > 1.0 means overloaded
# 4-core system: load > 4.0 means overloaded

# Disk I/O
iostat -xh 2                 # I/O stats every 2 seconds
iotop                        # Live I/O per process (like top for disk)
df -h                        # Disk space
du -sh /path/*               # Directory sizes

# Network I/O
sar -n DEV 1                 # Network device stats
nethogs                      # Bandwidth per process (like top for network)

# Check who is using a port / file
lsof -i :1521                # Who is using port 1521
lsof -u oracle               # Files opened by oracle user
lsof /u01/oradata/prod01.dbf # Who has this file open
fuser 1521/tcp               # Process using port 1521
```

---

## 5. System Logs - Where to Look

```bash
# Real-time log monitoring
tail -f /var/log/messages
tail -f /var/log/syslog

# Oracle specific logs
tail -f $ORACLE_BASE/diag/rdbms/$ORACLE_SID/$ORACLE_SID/trace/alert_$ORACLE_SID.log

# System journal (modern Linux)
journalctl -f                    # Follow journal
journalctl -u oracle             # Oracle service journal
journalctl --since "2024-01-15 10:00" --until "2024-01-15 11:00"

# Search through logs
grep -i "error" /var/log/messages | tail -50
zgrep "error" /var/log/messages.gz        # Search compressed log

# Log files for troubleshooting
/var/log/messages         # General system messages
/var/log/secure           # Authentication/SSH logs (RHEL)
/var/log/auth.log         # Authentication (Ubuntu)
/var/log/cron             # Cron job execution logs
/var/log/maillog          # Mail server logs
/var/log/boot.log         # System boot messages
```

---

## 6. Disk and Filesystem Troubleshooting

```bash
# Check disk space
df -h                            # Human-readable
df -ih                           # Inode usage (can be full even if space ok!)

# Find largest files/directories
du -sh /var/log/*                # Size of each dir in /var/log
du -sh /var/log/* | sort -rh | head -10  # Top 10 largest

# Find files larger than 100MB
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# Find files older than 30 days
find /tmp -type f -mtime +30 -delete       # Delete old temp files
find /var/log -name "*.log" -mtime +7      # Find old logs

# Check filesystem
df -T                            # Show filesystem types
mount | grep -v tmpfs             # Mounted filesystems

# Inode exhaustion (can't create files even with disk space)
df -i                            # Check inode usage
find / -xdev -type f | cut -d"/" -f2 | sort | uniq -c | sort -rn | head   # Files by dir
```

---

## 7. SSH Troubleshooting

```bash
# Basic SSH
ssh user@hostname
ssh -p 2222 user@hostname        # Non-standard port
ssh -i /path/to/key user@host    # Specific key

# Debug connection issues
ssh -v user@hostname             # Verbose (shows handshake)
ssh -vvv user@hostname           # Very verbose

# SSH config file (~/.ssh/config)
Host db-prod
  HostName dr.rds.example.com
  User oracle
  Port 22
  IdentityFile ~/.ssh/oracle_key

# Then just: ssh db-prod

# Copy SSH key (passwordless login)
ssh-keygen -t rsa -b 4096                  # Generate key pair
ssh-copy-id user@hostname                   # Copy public key to server

# Check SSH service
ss -tlnp | grep :22
```

---

## 8. Firewall Basics (Know for Context)

```bash
# Check iptables rules
iptables -L -n -v

# Check firewalld (RHEL 7+)
firewall-cmd --list-all
firewall-cmd --list-ports

# UFW (Ubuntu)
ufw status
ufw allow 1521/tcp               # Allow Oracle port
```

---

## 9. Troubleshooting Script - Oracle Connectivity Check

```bash
#!/bin/bash
# Comprehensive Oracle connectivity check

DB_HOST="dr.rds.example.com"
DB_PORT="1521"
DB_SID="PROD"

echo "=== Oracle Connectivity Check ==="

# Step 1: DNS resolution
echo -n "1. DNS resolution... "
if nslookup "$DB_HOST" > /dev/null 2>&1; then
  ip=$(dig +short "$DB_HOST")
  echo "OK -> $ip"
else
  echo "FAILED! Check VPN or /etc/hosts"
  exit 1
fi

# Step 2: Network reachability
echo -n "2. Host reachability (ping)... "
if ping -c 2 -W 3 "$DB_HOST" > /dev/null 2>&1; then
  echo "OK"
else
  echo "FAILED! Host unreachable (firewall or network issue)"
fi

# Step 3: Port connectivity
echo -n "3. Port $DB_PORT connectivity... "
if nc -zw3 "$DB_HOST" "$DB_PORT" 2>/dev/null; then
  echo "OK"
else
  echo "FAILED! Port $DB_PORT blocked or listener down"
  exit 1
fi

# Step 4: TNS ping
echo -n "4. TNS ping... "
if tnsping "$DB_SID" > /dev/null 2>&1; then
  echo "OK"
else
  echo "FAILED! TNS configuration issue"
fi

echo "=== Check Complete ==="
```

---

## 10. Performance Troubleshooting Workflow

### When a system is "slow":
```bash
# Step 1: Check load
uptime
# Load avg: 0.15, 0.12, 0.08 -> OK (< # of CPUs)
# Load avg: 16.50, 14.20, 12.10 -> PROBLEM

# Step 2: Check what's consuming CPU
top -bn1 | head -20
ps aux --sort=-%cpu | head -10

# Step 3: Check memory
free -h
# If 'available' is near 0 -> memory pressure
# Check for swap usage

# Step 4: Check disk I/O
iostat -xh 1 3
# %util near 100% -> disk is saturated

# Step 5: Check network
sar -n DEV 1 3

# Step 6: Check Oracle specifically
# (In sqlplus)
SELECT event, count(*) FROM v$session_wait
GROUP BY event ORDER BY 2 DESC;
```

---

## 🎯 Top Interview Questions & Answers

**Q1: How do you check if a port is open on a remote server?**
> `nc -zv hostname 1521` or `telnet hostname 1521`. Also: `nmap -p 1521 hostname` if nmap is available.

**Q2: The Oracle listener is down - how do you troubleshoot?**
> 1) `tnsping SID` - check if TNS resolves. 2) `netstat -an | grep 1521` - is port listening? 3) `lsnrctl status` - listener status. 4) Check `listener.log` for errors. 5) `lsnrctl start` to restart.

**Q3: How do you find which process is using a specific port?**
> `lsof -i :1521` or `ss -tlnp | grep 1521` or `netstat -tlnp | grep 1521`.

**Q4: How do you check disk space and find what's consuming it?**
> `df -h` for overall. `du -sh /path/* | sort -rh | head -10` to find largest directories.

**Q5: What is the load average in Linux?**
> The average number of processes in the "run queue" over 1, 5, and 15 minutes. For a healthy system, load should be less than the number of CPU cores. Load > cores = system under pressure.

**Q6: How do you follow a log file in real-time?**
> `tail -f /path/to/logfile.log`. For multiple files: `tail -f file1.log file2.log`. With grep filter: `tail -f logfile.log | grep -i error`.

---

## 📝 Quick Revision Summary

```
Connectivity:
  ping hostname           -> reachable?
  nc -zv host port        -> port open?
  nslookup hostname       -> DNS resolution
  tnsping SID             -> Oracle TNS connectivity
  telnet host 1521        -> test Oracle port

Performance:
  top                     -> live CPU/memory
  uptime                  -> load averages
  free -h                 -> memory
  iostat -xh              -> disk I/O
  df -h / df -ih          -> disk space / inodes
  du -sh /path/* | sort -rh -> find disk hogs

Logs:
  tail -f logfile         -> follow live
  journalctl -f           -> systemd journal
  grep -i error *.log     -> search for errors

Process / Port:
  lsof -i :1521           -> who uses port 1521
  lsof -u oracle          -> files opened by oracle
  ps -ef | grep oracle    -> oracle processes
  kill -9 PID             -> force kill
```

---

*Next: [Chapter 16 - Oracle EBS Architecture](16_oracle_ebs_architecture.md)*
