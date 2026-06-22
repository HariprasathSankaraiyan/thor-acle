# Chapter 1: Oracle Architecture
## ⏱️ ~15 min read

---

## 🧠 Think of it this way

> Think of Oracle Database like a **busy restaurant kitchen**:
> - The **kitchen** itself = the Database (files on disk)
> - The **open orders board** everyone looks at = SGA (shared memory)
> - The **chef's personal notepad** = PGA (per-session memory)
> - The **waiters/runners** = Background Processes
> - The **receipt book** = Redo Logs (tracks every change)

---

## 1. Oracle Database vs Instance

| Concept | What it is | Analogy |
|---------|-----------|---------|
| **Instance** | Processes + Memory (running in RAM) | The kitchen *in operation* |
| **Database** | Files on disk (data files, logs) | The kitchen's physical building & pantry |

**Key point**: You can have **1 instance + 1 database** (standard) or **multiple instances + 1 database** (RAC - Real Application Clusters).

---

## 2. SGA - System Global Area (Shared Memory)

> SGA is the whiteboard everyone in the kitchen can see and write on. Shared by ALL sessions.

### Major SGA Components

### 2a. Buffer Cache
- Stores **copies of data blocks** read from disk
- When Oracle reads data, it first checks the buffer cache ("Is it already in memory?")
- **LRU algorithm** - least recently used blocks get kicked out when memory is full
- **Analogy**: A chef checks the counter top first before going to the pantry (disk)

### 2b. Shared Pool
Two parts inside:
- **Library Cache**: Stores **parsed SQL statements** - so Oracle doesn't reparse the same SQL repeatedly
- **Data Dictionary Cache**: Stores metadata - table definitions, user privileges, column info
- **Analogy**: Library Cache = recipe cards pinned on the wall so you don't rewrite recipes. Dictionary Cache = the restaurant's employee handbook.

> **Interview gold**: "Soft parse" = SQL found in Library Cache (reuse). "Hard parse" = SQL not found, Oracle parses from scratch (expensive).

### 2c. Redo Log Buffer
- Small circular buffer in SGA
- Captures every **change made to data** before writing to redo log files on disk
- **Analogy**: Waiter writes the order on a notepad before putting it on the kitchen counter

### 2d. Large Pool (Optional)
- Used for **parallel queries**, RMAN backups, shared server connections
- Analogy: Extra prep table used for big catering events

### 2e. Java Pool / Streams Pool
- For Java stored procedures and Oracle Streams replication
- Less commonly asked

---

## 3. PGA - Program Global Area (Private Memory)

> PGA is each chef's own personal notepad. Private to each session.

- Allocated per **server process** (each connection gets its own PGA)
- Contains: sort area, hash join area, session variables, cursor state
- When a session disconnects, PGA memory is released
- **Larger PGA** = better performance for sorts/joins (happens in memory instead of disk temp)

---

## 4. Background Processes (The Kitchen Staff)

| Process | Full Name | Job | Analogy |
|---------|-----------|-----|---------|
| **DBWR** | Database Writer | Writes dirty blocks from buffer cache to data files | Chef who restocks the pantry |
| **LGWR** | Log Writer | Writes redo log buffer to redo log files on disk | Person who files all the order receipts |
| **SMON** | System Monitor | Instance recovery on startup, cleans up temp segments | Manager who cleans up after a fire drill |
| **PMON** | Process Monitor | Cleans up failed user processes, releases locks | Manager who handles when a waiter quits mid-shift |
| **CKPT** | Checkpoint | Triggers DBWR, updates control file with checkpoint info | Shift supervisor who says "save your work now" |
| **ARCn** | Archiver | Copies filled redo logs to archive log destination | Photocopies old receipt books for storage |

> **Interview question**: "What is SMON?" -> Instance recovery (rolls forward committed, rolls back uncommitted). "What is PMON?" -> Cleans up failed sessions/processes.

---

## 5. Physical Database Files

| File Type | Purpose |
|-----------|---------|
| **Data Files (.dbf)** | Actual table/index data |
| **Redo Log Files** | Record every change for recovery |
| **Control Files** | Database "header" - names, SCN, file locations |
| **Archive Log Files** | Archived redo logs (when ARCHIVELOG mode is on) |
| **Parameter File (spfile/pfile)** | Database startup parameters |
| **Temp Files** | Temporary space for sorts, hash joins |

---

## 6. Memory Architecture Flow (Step by Step)

When you run: `SELECT * FROM orders WHERE order_id = 1001`

1. **Session** sends SQL to Oracle server process
2. Oracle checks **Library Cache** - already parsed? (soft parse if yes)
3. Oracle checks **Buffer Cache** - is block in memory?
4. If not in cache -> reads from **Data File** -> puts block in **Buffer Cache**
5. Returns data to your session via **PGA**

When you run: `UPDATE orders SET status = 'CLOSED' WHERE order_id = 1001`

1. Oracle reads block into **Buffer Cache** (marks it "dirty")
2. Change written to **Redo Log Buffer** immediately
3. **LGWR** writes redo buffer to **Redo Log File** on COMMIT
4. **DBWR** writes dirty block to **Data File** lazily (at checkpoint/buffer pressure)

---

## 7. Oracle RAC (Real Application Clusters)

> Multiple kitchens (instances) sharing the same pantry (database files).

- Multiple instances on different servers share ONE database on shared storage
- **Cache Fusion**: If one node has a block in its buffer cache that another node needs, it's transferred over the interconnect (not disk) - very fast
- **Use case**: High availability + load balancing
- **Interview**: "How does RAC maintain data consistency?" -> Cache Fusion protocol, global lock manager (GCS/GES)

---

## 8. ARCHIVELOG vs NOARCHIVELOG Mode

| Mode | Archive Logs Kept? | Recovery Options |
|------|-------------------|-----------------|
| ARCHIVELOG | Yes | Full point-in-time recovery possible |
| NOARCHIVELOG | No | Can only restore to last full backup |

> **Production databases MUST be in ARCHIVELOG mode.**

---

## 9. SCN - System Change Number

> SCN is Oracle's internal clock/counter. Every time data changes, SCN increments.

- Used for: MVCC (consistent reads), recovery, flashback queries
- `SELECT current_scn FROM v$database;`

---

## 10. Data Dictionary Views - Know These

| View | What it shows |
|------|--------------|
| `DBA_TABLES` | All tables in DB |
| `DBA_INDEXES` | All indexes |
| `DBA_USERS` | All users |
| `V$SESSION` | Current active sessions |
| `V$SQL` | SQL in shared pool |
| `V$PROCESS` | Server processes |
| `V$DATAFILE` | Data file info |
| `V$INSTANCE` | Instance name, status |
| `V$DATABASE` | DB name, SCN, mode |

> Prefix **DBA_** = all objects in DB. **ALL_** = objects you can access. **USER_** = objects you own.

---

## 🎯 Top Interview Questions & Answers

**Q1: What is the difference between SGA and PGA?**
> SGA is shared memory - used by all sessions simultaneously. PGA is private memory - each session/process gets its own PGA. SGA has buffer cache, shared pool, redo buffer. PGA has sort area and session info.

**Q2: What happens when LGWR writes?**
> LGWR writes on: COMMIT, redo log buffer 1/3 full, every 3 seconds, before DBWR writes. This is the "write-ahead logging" principle - redo is always written before data.

**Q3: What is a checkpoint?**
> A signal from CKPT process that tells DBWR to write dirty blocks to disk, and updates the control file and data file headers with the current SCN. Reduces recovery time on restart.

**Q4: What is the purpose of undo segments?**
> Undo (formerly rollback segments) store the "before image" of changed data. Used for: rollback operations, consistent reads (MVCC), and flashback queries.

**Q5: How many instances can connect to one database?**
> In standard (non-RAC): 1 instance, 1 database. In RAC: multiple instances, one database.

**Q6: What is cache fusion in RAC?**
> When a node needs a data block that another node has in its buffer cache, it transfers the block directly via high-speed interconnect (not going to disk). This is Cache Fusion.

---

## 📝 Quick Revision Summary

```
Oracle = Instance (Memory + Processes) + Database (Files on Disk)

SGA (Shared):
  Buffer Cache -> data blocks
  Shared Pool  -> parsed SQL + metadata
  Redo Buffer  -> change records (pre-write)

PGA (Private per session):
  Sort area, hash join area, session info

Key Processes:
  DBWR  -> dirty blocks to disk
  LGWR  -> redo buffer to redo logs (on COMMIT)
  SMON  -> instance recovery
  PMON  -> failed process cleanup
  CKPT  -> checkpoint signal
  ARCn  -> archive redo logs

Soft Parse = SQL in Library Cache (good, fast)
Hard Parse = SQL not in cache (bad, expensive)
```

---

*Next: [Chapter 2 - Oracle SQL Core](02_oracle_sql_core.md)*
