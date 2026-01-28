# Lesson 14: Write-Ahead Logging (WAL) — Durability and Recovery

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 4 — Write-Ahead Logging |
| **Lesson** | 14 of 66 |
| **Estimated Time** | 90-120 minutes |
| **Prerequisites** | Lesson 8: Heap Storage and Page Layout |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain why WAL is essential for durability and recovery
2. Describe the structure of WAL records and segments
3. Understand how LSN (Log Sequence Number) works
4. Trace the crash recovery process step by step
5. Configure WAL for performance and reliability

### Key Terms

| Term | Definition |
|------|------------|
| **WAL** | Write-Ahead Logging: log changes before applying to data files |
| **LSN** | Log Sequence Number: byte offset into the WAL stream |
| **WAL Segment** | Fixed-size file (16MB default) containing WAL records |
| **Checkpoint** | Point where all data is flushed, enabling WAL truncation |
| **REDO Point** | LSN from which recovery must replay |
| **Full Page Write** | Writing entire page to WAL after checkpoint |
| **WAL Buffer** | In-memory buffer for WAL records before disk write |

---

## Introduction

PostgreSQL guarantees that committed transactions survive system crashes. This durability comes from **Write-Ahead Logging (WAL)**—the principle that:

> **Changes must be logged to WAL before they're applied to data files.**

If the system crashes:
1. Data files may contain uncommitted changes (dirty pages flushed pre-commit)
2. Data files may be missing committed changes (dirty pages not yet flushed)
3. WAL contains the authoritative record of what happened

Recovery replays WAL from the last checkpoint, bringing the database to a consistent state.

### Why Not Just Write Data Directly?

Consider what happens without WAL during a crash:

```
UPDATE accounts SET balance = 500 WHERE id = 1;
COMMIT;

Timeline:
1. Modify page in memory (shared_buffers)     ✓
2. Write commit record                         ✓
3. ...time passes...
4. Background writer flushes to disk           ← CRASH HERE!

Result: Committed change never made it to disk!
```

With WAL:

```
UPDATE accounts SET balance = 500 WHERE id = 1;
COMMIT;

Timeline:
1. Modify page in memory (shared_buffers)     ✓
2. Write WAL record of the change             ✓
3. Flush WAL to disk                          ✓
4. Write commit record to WAL                 ✓
5. Flush commit record to disk                ✓
6. Return success to client                   ✓
7. ...later...
8. Background writer flushes data to disk     ← CRASH HERE is OK!

Recovery: Replay WAL, change is reapplied, nothing lost!
```

---

## Conceptual Foundation

### The WAL Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  SHARED MEMORY                                                   │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  WAL Buffer                                                 │ │
│  │  ┌──────┬──────┬──────┬──────┬──────┬──────────────────┐   │ │
│  │  │Rec 1 │Rec 2 │Rec 3 │Rec 4 │Rec 5 │  free space...   │   │ │
│  │  └──────┴──────┴──────┴──────┴──────┴──────────────────┘   │ │
│  │                           │                                 │ │
│  │                           ▼ (WAL writer / commit flush)    │ │
│  └────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  pg_wal/ DIRECTORY (on disk)                                     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 000000010000000000000001  (16 MB segment)                   │ │
│  │ ┌──────────────────────────────────────────────────────┐    │ │
│  │ │ Page 0: [Rec1][Rec2][Rec3]...                        │    │ │
│  │ │ Page 1: [Rec4][Rec5][Rec6]...                        │    │ │
│  │ │ Page 2: ...                                           │    │ │
│  │ └──────────────────────────────────────────────────────┘    │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ 000000010000000000000002  (next segment)                    │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ 000000010000000000000003  ...                               │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Log Sequence Number (LSN)

An LSN is a 64-bit pointer into the WAL stream:

```
LSN: 0/16B4E60

Format: XXXXXXXX/YYZZZZZZ
        ────────  ────────
        High 32   Low 32 bits
        bits      

Segment: 0/16000000 → 000000010000000000000016
Offset within segment: 0xB4E60 = 740,960 bytes
```

LSNs increase monotonically—every new WAL record gets a higher LSN.

### WAL Segment Naming

```
000000010000000000000016
────────────────────────
TTTTTTTTSSSSSSSSSSSSSSSS

T = Timeline (8 hex digits)
S = Segment number (16 hex digits)

Example: Timeline 1, Segment 22 (hex 16)
```

### What Gets Logged

| Operation | WAL Record Type |
|-----------|-----------------|
| INSERT | HEAP_INSERT |
| UPDATE | HEAP_UPDATE or HOT_UPDATE |
| DELETE | HEAP_DELETE |
| Index operations | BTREE_INSERT, etc. |
| Commit | XACT_COMMIT |
| Abort | XACT_ABORT |
| Checkpoint | CHECKPOINT_SHUTDOWN/ONLINE |
| Table creation | CREATE_TABLE |

---

## Deep Dive: WAL Record Structure

### WAL Record Components

```c
/* Simplified from src/include/access/xlogrecord.h */
typedef struct XLogRecord
{
    uint32      xl_tot_len;     /* Total record length */
    TransactionId xl_xid;       /* Transaction ID */
    XLogRecPtr  xl_prev;        /* Ptr to previous record */
    uint8       xl_info;        /* Flag bits */
    RmgrId      xl_rmid;        /* Resource manager ID */
    pg_crc32c   xl_crc;         /* CRC of this record */
    
    /* Followed by XLogRecordBlockHeaders and data */
} XLogRecord;

/* Resource managers (xl_rmid) */
#define RM_XLOG_ID          0   /* WAL control */
#define RM_XACT_ID          1   /* Transaction */
#define RM_HEAP_ID          10  /* Heap operations */
#define RM_HEAP2_ID         11  /* More heap ops */
#define RM_BTREE_ID         12  /* B-tree index */
```

### WAL Page Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  WAL Page (8KB, same as data pages)                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ XLogPageHeaderData (24 bytes)                                │ │
│  │ ┌─────────────────────────────────────────────────────────┐  │ │
│  │ │ xlp_magic    │ xlp_info    │ xlp_tli                   │  │ │
│  │ │ xlp_pageaddr │ xlp_rem_len (for continued records)     │  │ │
│  │ └─────────────────────────────────────────────────────────┘  │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ WAL Records (variable length, back to back)                  │ │
│  │ ┌──────────┬──────────┬──────────┬──────────────────────┐   │ │
│  │ │ Record 1 │ Record 2 │ Record 3 │ Record 4 (partial→   │   │ │
│  │ └──────────┴──────────┴──────────┴──────────────────────┘   │ │
│  │                                               → next page)   │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Full Page Writes

To handle partial page writes (OS writes part of a page during crash), PostgreSQL logs the full page image after each checkpoint:

```
┌─────────────────────────────────────────────────────────────────┐
│  Full Page Write (FPW) Protection                                │
│                                                                  │
│  CHECKPOINT at LSN 0/1000000                                    │
│                                                                  │
│  First modification to page after checkpoint:                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ WAL Record:                                                  │ │
│  │   - Operation: HEAP_INSERT                                   │ │
│  │   - Full Page Image: [8KB of page data]  ← BACKUP BLOCK    │ │
│  │   - Change delta: (what changed)                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Subsequent modifications (same page, before next checkpoint):  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ WAL Record:                                                  │ │
│  │   - Operation: HEAP_INSERT                                   │ │
│  │   - NO full page image (already logged this checkpoint)      │ │
│  │   - Change delta only                                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Checkpoints

### What is a Checkpoint?

A checkpoint is a point where:
1. All dirty buffers are flushed to disk
2. A checkpoint record is written to WAL
3. pg_control is updated with checkpoint info
4. Old WAL segments can be recycled

```
┌─────────────────────────────────────────────────────────────────┐
│  Checkpoint Timeline                                             │
│                                                                  │
│  WAL: ═══════════════════════════════════════════════════════►  │
│       │                    │                    │               │
│       ▼                    ▼                    ▼               │
│  Checkpoint 1         Checkpoint 2         Checkpoint 3         │
│  (REDO point 1)       (REDO point 2)       (REDO point 3)      │
│       │                    │                    │               │
│       │◄──────────────────►│                    │               │
│       │   WAL needed for   │◄──────────────────►│               │
│       │   recovery         │   WAL needed for   │               │
│       │                    │   recovery now     │               │
│       │                    │                    │               │
│  Old WAL                Old WAL can         Current             │
│  can be                 be recycled         recovery            │
│  deleted                now                 window              │
└─────────────────────────────────────────────────────────────────┘
```

### Checkpoint Process

```c
/* Simplified checkpoint process */
void CreateCheckPoint(int flags)
{
    XLogRecPtr  redo;
    
    /* 1. Mark REDO point (where recovery should start) */
    redo = GetRedoRecPtr();
    
    /* 2. Flush all dirty shared buffers to disk */
    CheckPointBuffers(flags);
    
    /* 3. Flush other caches (clog, multixact, etc.) */
    CheckPointCaches();
    
    /* 4. Write checkpoint WAL record */
    XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_ONLINE);
    
    /* 5. Update pg_control with new checkpoint info */
    UpdateControlFile();
    
    /* 6. Delete/recycle old WAL segments */
    RemoveOldXlogFiles(redo);
}
```

### Checkpoint Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `checkpoint_timeout` | 5min | Maximum time between checkpoints |
| `max_wal_size` | 1GB | Trigger checkpoint if WAL exceeds this |
| `min_wal_size` | 80MB | Keep at least this much WAL |
| `checkpoint_completion_target` | 0.9 | Spread checkpoint I/O over this fraction |

---

## Crash Recovery

### Recovery Process

```
┌─────────────────────────────────────────────────────────────────┐
│  CRASH RECOVERY STEPS                                            │
│                                                                  │
│  1. Read pg_control                                              │
│     ├── State: CRASHED (not clean shutdown)                     │
│     └── Last checkpoint at: 0/16B4E60                           │
│                                                                  │
│  2. Read checkpoint record from WAL                              │
│     ├── REDO point: 0/1680000                                   │
│     └── Timeline: 1                                             │
│                                                                  │
│  3. Start REDO from REDO point                                   │
│     ├── Read WAL record at 0/1680000                            │
│     ├── Apply change to buffer pool                             │
│     ├── Read next WAL record...                                 │
│     ├── Apply...                                                │
│     └── Continue until end of WAL                               │
│                                                                  │
│  4. Check for uncommitted transactions                          │
│     └── Any transaction without COMMIT record → rolled back     │
│                                                                  │
│  5. Write end-of-recovery checkpoint                            │
│     └── pg_control state → IN_PRODUCTION                        │
│                                                                  │
│  6. Accept connections                                           │
└─────────────────────────────────────────────────────────────────┘
```

### REDO Logic

```c
/* Simplified REDO process */
void StartupXLOG(void)
{
    XLogRecPtr  record;
    
    /* Read checkpoint from pg_control */
    checkpoint = ReadControlFile();
    
    /* Begin reading WAL from REDO point */
    record = checkpoint.redo;
    
    while ((xlogrec = XLogReadRecord(&record)) != NULL)
    {
        /* Get resource manager for this record type */
        RmgrId rmid = XLogRecGetRmid(xlogrec);
        
        /* Apply the change */
        RmgrTable[rmid].rm_redo(xlogrec);
        
        /* Move to next record */
        record = XLogRecGetNext(xlogrec);
    }
    
    /* Recovery complete */
    EndOfRecovery();
}
```

---

## WAL Configuration and Performance

### Synchronous Commit Levels

```sql
-- Most durable: wait for WAL to reach disk (default)
SET synchronous_commit = on;

-- Less durable: don't wait for fsync
SET synchronous_commit = off;

-- More durable: wait for standby to receive
SET synchronous_commit = remote_write;

-- Maximum durability: wait for standby to fsync
SET synchronous_commit = remote_apply;
```

### WAL Compression

```sql
-- Compress full page writes (saves space, costs CPU)
SET wal_compression = on;
```

### WAL Level

| Level | Usage | Size |
|-------|-------|------|
| `minimal` | Standalone, no backup | Smallest |
| `replica` | Streaming replication | Medium |
| `logical` | Logical replication | Largest |

```sql
SHOW wal_level;  -- Default: replica
```

---

## Practical Examples

### Example 1: Viewing Current WAL Position

```sql
-- Current WAL write position
SELECT pg_current_wal_lsn();

-- Current WAL flush position
SELECT pg_current_wal_flush_lsn();

-- Current WAL insert position
SELECT pg_current_wal_insert_lsn();

-- Difference in bytes
SELECT pg_wal_lsn_diff(
    pg_current_wal_insert_lsn(),
    pg_current_wal_flush_lsn()
) AS unflushed_bytes;
```

### Example 2: WAL Files

```bash
# List WAL files
ls -la $PGDATA/pg_wal/

# Size of pg_wal directory
du -sh $PGDATA/pg_wal/
```

```sql
-- Current WAL segment
SELECT pg_walfile_name(pg_current_wal_lsn());

-- Which segment contains a specific LSN
SELECT pg_walfile_name('0/16B4E60');
```

### Example 3: Monitoring Checkpoints

```sql
-- Checkpoint statistics
SELECT * FROM pg_stat_bgwriter;

-- Key columns:
-- checkpoints_timed: Scheduled checkpoints
-- checkpoints_req: Requested checkpoints (WAL size exceeded)
-- checkpoint_write_time: Time spent writing
-- checkpoint_sync_time: Time spent syncing
```

### Example 4: Page LSN

```sql
-- Each data page tracks its most recent WAL LSN
SELECT lsn FROM page_header(get_raw_page('mytable', 0));
-- This helps verify if a page needs recovery
```

### Example 5: Force a Checkpoint

```sql
-- Manually trigger checkpoint
CHECKPOINT;

-- Log checkpoint activity
SET log_checkpoints = on;
```

---

## WAL Archiving

### Continuous Archiving

```sql
-- Enable archiving
ALTER SYSTEM SET archive_mode = on;
ALTER SYSTEM SET archive_command = 'cp %p /backup/wal/%f';

-- After restart
-- Every completed WAL segment is archived
```

### Point-in-Time Recovery (PITR)

```bash
# recovery.signal file indicates recovery mode
touch $PGDATA/recovery.signal

# postgresql.conf settings for recovery
restore_command = 'cp /backup/wal/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
```

---

## Hands-On Exercises

### Exercise 1: WAL Position Tracking (Basic)

```sql
-- Note starting position
SELECT pg_current_wal_lsn() AS start_lsn;

-- Generate some WAL
CREATE TABLE wal_test AS SELECT generate_series(1, 10000);

-- Check new position
SELECT pg_current_wal_lsn() AS end_lsn;

-- Calculate WAL generated
SELECT pg_wal_lsn_diff(
    pg_current_wal_lsn(),
    'start_lsn_value'::pg_lsn
) AS bytes_generated;
```

### Exercise 2: Checkpoint Observation (Intermediate)

```sql
-- Reset stats
SELECT pg_stat_reset_shared('bgwriter');

-- Check current stats
SELECT checkpoints_timed, checkpoints_req, 
       checkpoint_write_time, checkpoint_sync_time
FROM pg_stat_bgwriter;

-- Generate WAL to trigger checkpoint
INSERT INTO wal_test SELECT generate_series(1, 1000000);

-- Force checkpoint
CHECKPOINT;

-- Check stats again
SELECT checkpoints_timed, checkpoints_req,
       checkpoint_write_time, checkpoint_sync_time  
FROM pg_stat_bgwriter;
```

### Exercise 3: LSN on Data Pages (Intermediate)

```sql
-- Create table
CREATE TABLE lsn_test (id int);
INSERT INTO lsn_test VALUES (1);

-- Check page LSN
SELECT lsn FROM page_header(get_raw_page('lsn_test', 0));

-- Modify the page
UPDATE lsn_test SET id = 2;

-- Check LSN increased
SELECT lsn FROM page_header(get_raw_page('lsn_test', 0));
```

### Exercise 4: WAL Segment Analysis (Advanced)

```bash
# Use pg_waldump to inspect WAL (if available)
pg_waldump -p $PGDATA/pg_wal -s 0/1000000 -e 0/2000000

# Shows each WAL record with:
# - LSN
# - Transaction ID
# - Record type (HEAP_INSERT, COMMIT, etc.)
# - Size
```

---

## Key Takeaways

1. **WAL guarantees durability**: Changes are logged before data files are modified.

2. **LSN is the universal position marker**: 64-bit offset into the WAL stream.

3. **Checkpoints enable recovery**: They mark points where all data is on disk.

4. **Full page writes protect against partial writes**: First modification after checkpoint logs entire page.

5. **Recovery replays from REDO point**: Only WAL since last checkpoint is needed.

6. **synchronous_commit controls durability/speed tradeoff**: Default is safe but can be relaxed.

7. **wal_level controls features**: Higher levels enable replication but generate more WAL.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/access/transam/xlog.c` | Main WAL code |
| `src/backend/access/transam/xloginsert.c` | WAL record insertion |
| `src/backend/access/transam/xlogreader.c` | WAL reading |
| `src/include/access/xlogrecord.h` | WAL record structure |

### Documentation

- [PostgreSQL Docs: WAL Configuration](https://www.postgresql.org/docs/current/wal-configuration.html)
- [PostgreSQL Docs: WAL Internals](https://www.postgresql.org/docs/current/wal-internals.html)
- [PostgreSQL Docs: Continuous Archiving](https://www.postgresql.org/docs/current/continuous-archiving.html)

### Next Lessons

- **Lesson 15**: VACUUM and Dead Tuple Cleanup
- **Lesson 16**: Streaming Replication
- **Lesson 17**: Logical Replication

---

## References

### Key Structures

| Structure | File | Purpose |
|-----------|------|---------|
| `XLogRecord` | xlogrecord.h | WAL record header |
| `XLogPageHeaderData` | xlog_internal.h | WAL page header |
| `CheckPoint` | pg_control.h | Checkpoint info |

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `XLogInsert()` | xloginsert.c | Insert WAL record |
| `XLogFlush()` | xlog.c | Flush WAL to disk |
| `CreateCheckPoint()` | xlog.c | Perform checkpoint |
| `StartupXLOG()` | xlog.c | Crash recovery |

### Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `wal_level` | replica | WAL detail level |
| `fsync` | on | Force sync to disk |
| `synchronous_commit` | on | Wait for WAL flush |
| `wal_buffers` | -1 (auto) | WAL buffer size |
| `checkpoint_timeout` | 5min | Max checkpoint interval |
| `max_wal_size` | 1GB | Checkpoint trigger size |
