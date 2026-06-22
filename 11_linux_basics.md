# Chapter 11: Linux Basics - Filesystem, Permissions & Navigation
## ⏱️ ~20 min read

---

## 🧠 Think of it this way

> Linux filesystem = a **tree of folders** starting from root `/` (the trunk).
> Everything in Linux is a **file** - programs, devices, settings, even directories.
>
> Permissions = **3 locks on a box**:
> One for the owner, one for the group, one for everyone else.
> Each lock can be "read", "write", or "execute".

---

## 1. Linux Filesystem Structure

```
/                   ← Root (everything starts here)
├── /bin            ← Essential user commands (ls, cp, mv, rm, bash)
├── /sbin           ← System admin commands (fdisk, init, ifconfig)
├── /usr            ← User programs and libraries
│   ├── /usr/bin    ← Most user commands live here
│   └── /usr/local  ← Locally installed software
├── /etc            ← Configuration files (system-wide)
├── /home           ← User home directories (/home/oracle, /home/scott)
├── /root           ← Root user's home directory
├── /var            ← Variable data (logs, spool, databases)
│   └── /var/log    ← System logs
├── /tmp            ← Temporary files (cleared on reboot)
├── /opt            ← Optional/third-party software
├── /dev            ← Device files (disks, terminals)
├── /proc           ← Virtual filesystem (running processes, kernel info)
├── /mnt            ← Mount points for filesystems
└── /lib            ← Shared libraries
```

---

## 2. Navigation Commands

```bash
pwd                        # Print working directory (where am I?)
cd /home/oracle            # Go to absolute path
cd ..                      # Go up one level
cd -                       # Go to previous directory
cd ~                       # Go to home directory

ls                         # List files
ls -l                      # Long listing (permissions, size, date)
ls -la                     # Long listing including hidden files
ls -lh                     # Human-readable sizes (KB, MB, GB)
ls -lt                     # Sort by time (newest first)
ls -ltr                    # Sort by time reversed (oldest first)
ls -R                      # Recursive listing
```

---

## 3. File Operations

```bash
# Copy
cp file.txt /tmp/              # Copy file to /tmp
cp -r dir1/ dir2/             # Copy directory recursively
cp -p file.txt backup.txt     # Preserve timestamps/permissions

# Move / Rename
mv file.txt /tmp/              # Move file
mv old_name.txt new_name.txt   # Rename

# Remove
rm file.txt                    # Delete file
rm -f file.txt                 # Force delete (no prompt)
rm -r dir/                     # Remove directory recursively
rm -rf dir/                    # Force remove directory (DANGEROUS!)

# Create
touch filename.txt             # Create empty file / update timestamp
mkdir -p /path/to/new/dir      # Create directory (with parents)

# View file content
cat file.txt                   # Display all content
less file.txt                  # Paginated view (q to quit)
head -20 file.txt              # First 20 lines
tail -20 file.txt              # Last 20 lines
tail -f logfile.log            # Follow file (live log watching)
wc -l file.txt                 # Count lines
wc -c file.txt                 # Count bytes

# File info
file document                  # What type of file is this?
stat file.txt                  # Detailed info (size, permissions, timestamps)
du -sh /opt/oracle             # Disk usage of directory
df -h                          # Disk space of all filesystems
```

---

## 4. File Permissions - The Most Important Linux Concept

```bash
$ ls -l orders.sh
-rwxr-xr-- 1 oracle dba 4096 Jan 15 10:30 orders.sh
 │││└────┘   └───┘ └─┘
 ││└──────── Other permissions (r--)
 │└───────── Group permissions (r-x)
 └────────── Owner permissions (rwx)
             Owner=oracle  Group=dba

First character:
  - = regular file
  d = directory
  l = symbolic link
  c = character device
  b = block device
```

### Permission Bits
| Symbol | Meaning | Numeric |
|--------|---------|---------|
| `r` | Read | 4 |
| `w` | Write | 2 |
| `x` | Execute | 1 |
| `-` | No permission | 0 |

### Numeric (Octal) Notation
```
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0

chmod 755 script.sh  ->  rwxr-xr-x  (owner=7, group=5, others=5)
chmod 644 data.txt   ->  rw-r--r--  (owner=6, group=4, others=4)
chmod 600 secret.key ->  rw-------  (owner only read/write)
chmod 777 shared/    ->  rwxrwxrwx  (everyone full access)
```

### chmod Symbolic Mode
```bash
chmod +x script.sh          # Add execute for all
chmod u+x script.sh         # Add execute for user/owner only
chmod g+w file.txt          # Add write for group
chmod o-r private.txt       # Remove read from others
chmod u=rwx,g=rx,o=        # Explicit permissions
```

### chown and chgrp
```bash
chown oracle file.txt            # Change owner to oracle
chown oracle:dba file.txt        # Change owner and group
chown -R oracle:dba /app/        # Recursive change
chgrp dba file.txt               # Change group only
```

---

## 5. Special Permissions

| Permission | Meaning |
|-----------|---------|
| **SUID (4xxx)** | Execute file as file owner (not as you) |
| **SGID (2xxx)** | New files inherit directory's group |
| **Sticky Bit (1xxx)** | Only owner can delete files in directory |

```bash
chmod 4755 program     # SUID - runs as owner
chmod 2775 shared_dir  # SGID - group inheritance
chmod 1777 /tmp        # Sticky bit on /tmp (everyone can create, only owner can delete)
```

---

## 6. Links - Hard and Symbolic

### Symbolic Link (Symlink = Shortcut)
```bash
ln -s /opt/oracle/product/19c/bin/sqlplus /usr/local/bin/sqlplus
# Now you can type 'sqlplus' instead of full path
ls -la /usr/local/bin/sqlplus
# lrwxrwxrwx -> /opt/oracle/product/19c/bin/sqlplus
```

### Hard Link
```bash
ln original.txt hardlink.txt
# Both point to same inode (same actual file data)
# Deleting one doesn't affect the other
```

> **Difference**: Symlink = shortcut pointing to a path (broken if target deleted). Hard link = another name for same file data (can't span filesystems).

---

## 7. Environment Variables

```bash
# View
env                          # All environment variables
echo $PATH                   # PATH variable
echo $HOME                   # Home directory
echo $USER                   # Current user
echo $SHELL                  # Current shell
echo $ORACLE_HOME            # Oracle home (if set)

# Set (current session only)
export MY_VAR="value"
export ORACLE_SID=PROD
export ORACLE_HOME=/u01/app/oracle/product/19c

# Make permanent (add to ~/.bash_profile or ~/.bashrc)
echo 'export ORACLE_HOME=/u01/app/oracle/product/19c' >> ~/.bash_profile
source ~/.bash_profile        # Reload without logout

# Unset
unset MY_VAR
```

---

## 8. Process Management

```bash
ps -ef                       # All processes, full listing
ps -ef | grep oracle         # Filter oracle processes
ps aux                       # All processes with CPU/memory
top                          # Live process monitor
htop                         # Enhanced top (if installed)

# Process signals
kill 1234                    # Send SIGTERM (graceful)
kill -9 1234                 # Send SIGKILL (force kill)
kill -l                      # List all signals

# Background / Foreground
./long_script.sh &           # Run in background
jobs                         # List background jobs
fg 1                         # Bring job 1 to foreground
bg 1                         # Send stopped job to background
Ctrl+Z                       # Stop (pause) current process
Ctrl+C                       # Interrupt (kill) current process
```

---

## 9. Important Linux Configuration Files

| File | Purpose |
|------|---------|
| `~/.bash_profile` | Login shell startup (Oracle user config here) |
| `~/.bashrc` | Interactive shell startup |
| `~/.bash_history` | Command history |
| `/etc/passwd` | User accounts |
| `/etc/group` | Group definitions |
| `/etc/hosts` | Hostname to IP mapping |
| `/etc/profile` | System-wide shell config |
| `/etc/crontab` | System cron jobs |
| `/var/log/messages` | System messages |
| `/var/log/syslog` | Syslog (Ubuntu) |

---

## 10. Useful Shortcuts & Tricks

```bash
!!                   # Repeat last command
!grep                # Repeat last command starting with 'grep'
Ctrl+R               # Reverse search command history
history | tail -20   # Last 20 commands
history | grep sqlplus

# Tab completion
cd /opt/ora[TAB]     # Autocomplete
ls /etc/pa[TAB][TAB] # Show all matches

# Redirection
command > file.txt    # Redirect stdout to file (overwrite)
command >> file.txt   # Redirect stdout to file (append)
command 2> error.log  # Redirect stderr to file
command 2>&1          # Redirect stderr to stdout
command &> all.log    # Both stdout and stderr to file

# Pipes
ls -la | head -10     # Pass output of ls to head
ps -ef | grep oracle | grep -v grep
```

---

## 🎯 Top Interview Questions & Answers

**Q1: What does chmod 755 mean?**
> Owner: rwx (7=read+write+execute), Group: r-x (5=read+execute), Others: r-x (5=read+execute). Typical for executable scripts.

**Q2: What is the difference between a hard link and a symbolic link?**
> Hard link: another name for the same inode (same file data). Symlink: a pointer/shortcut to another path. Deleting a symlink's target breaks it. Hard links survive deletion of other names.

**Q3: What does the sticky bit do?**
> When set on a directory, only the file owner (or root) can delete files in it, even if others have write permission. Classic example: /tmp directory.

**Q4: How do you make a script executable?**
> `chmod +x script.sh` or `chmod 755 script.sh`

**Q5: What is the difference between ~/.bash_profile and ~/.bashrc?**
> .bash_profile runs for login shells (SSH login, su -). .bashrc runs for interactive non-login shells. Oracle environment variables (ORACLE_HOME, ORACLE_SID) go in .bash_profile.

**Q6: How do you find all files owned by oracle user?**
> `find / -user oracle -type f 2>/dev/null`

---

## 📝 Quick Revision Summary

```
Filesystem: / (root), /etc (config), /var/log (logs), /tmp (temp), /home (users)

Permissions: rwxrwxrwx = owner|group|others
  r=4, w=2, x=1
  chmod 755 = rwxr-xr-x
  chmod 644 = rw-r--r--

chown oracle:dba file    -> change owner and group
chmod +x script.sh       -> add execute permission

Hard link: same inode, survives original deletion
Symlink: ln -s target link, shortcut to path

env / export: environment variables
export ORACLE_HOME=... in ~/.bash_profile

Kill signals: kill 1234 (SIGTERM), kill -9 1234 (SIGKILL)

Redirection: > (overwrite), >> (append), 2>&1 (stderr to stdout)
Pipe: | (chain commands)
```

---

*Next: [Chapter 12 - Linux Text Processing](12_linux_text_processing.md)*
