# Lesson 15: VACUUM — Dead Tuple Cleanup and Space Reclamation

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 7 — VACUUM and Maintenance |
| **Lesson** | 15 of 66 |
| **Estimated Time** | 90-120 minutes |
| **Prerequisites** | Lesson 13: MVCC Fundamentals |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain why VACUUM is essential in PostgreSQL
2. Describe the difference between regular VACUUM and VACUUM FULL
3. Understand how dead tuples are identified and removed
4. Configure autovacuum for optimal performance
5. Monitor and troubleshoot VACUUM operations

### Key Terms

| Term | Definition |
|------|------------|
| **Dead Tuple** | A row version no longer visible to any transaction |
| **VACUUM** | Process that reclaims space from dead tuples |
| **VACUUM FULL** | Rewrites entire table, reclaiming all space |
| **Autovacuum** | Background daemon that runs VACUUM automatically |
| **Free Space Map (FSM)** | Tracks available space in pages for reuse |
| **Visibility Map (VM)** | Tracks pages where all tuples are visible |
| **Freeze** | Mark old tuples as permanently visible |

---

## Introduction

MVCC creates multiple versions of rows to enable concurrent access. But these old versions accumulate:

```
┌─────────────────────────────────────────────────────────────────┐
│  Table: accounts                                                 │
│                                                                  │
│  Page 0:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Live: id=1, balance=500   (xmin=100, xmax=0)               │ │
│  │ Dead: id=1, balance=1000  (xmin=50, xmax=100) ← OLD!       │ │
│  │ Dead: id=1, balance=800   (xmin=75, xmax=100) ← OLD!       │ │
│  │ Live: id=2, balance=200   (xmin=90, xmax=0)                │ │
│  │ Dead: id=2, balance=300   (xmin=60, xmax=90)  ← OLD!       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Without VACUUM: Dead tuples waste space, table grows forever   │
│  With VACUUM: Dead tuples removed, space reused for new rows    │
└─────────────────────────────────────────────────────────────────┘
```

VACUUM solves multiple problems:
1. **Space reclamation**: Remove dead tuples, mark space as reusable
2. **Index cleanup**: Remove index entries pointing to dead tuples
3. **Transaction ID wraparound prevention**: Freeze old XIDs
4. **Statistics updates**: Update visibility map

> **Critical Understanding**: VACUUM doesn't shrink the table file—it marks space as reusable. Only VACUUM FULL (or pg_repack) actually shrinks files.

---

## Conceptual Foundation

### Types of VACUUM

| Command | Locks | Space Reclaim | When to Use |
|---------|-------|---------------|-------------|
| `VACUUM` | ShareUpdateExclusive | Marks space reusable (no shrink) | Regular maintenance |
| `VACUUM FULL` | AccessExclusive (blocks all) | Rewrites table, shrinks | Emergency only |
| `VACUUM ANALYZE` | Same as VACUUM | + Updates statistics | After bulk changes |
| `VACUUM (VERBOSE)` | Same as VACUUM | + Shows detailed output | Debugging |

### The VACUUM Process

```
┌─────────────────────────────────────────────────────────────────┐
│  VACUUM Process Steps                                            │
│                                                                  │
│  1. Determine oldest transaction that might need old versions   │
│     └── OldestXmin = oldest active transaction's XID            │
│                                                                  │
│  2. Scan heap pages                                              │
│     └── For each page:                                          │
│         ├── Skip if all-visible in visibility map               │
│         ├── Identify dead tuples (xmax < OldestXmin, committed) │
│         └── Build list of dead TIDs                             │
│                                                                  │
│  3. Remove index entries pointing to dead tuples                │
│     └── Scan each index, remove entries for dead TIDs           │
│                                                                  │
│  4. Mark dead tuple space as reusable                           │
│     └── Update line pointers to LP_DEAD or LP_UNUSED            │
│     └── Update Free Space Map (FSM)                             │
│                                                                  │
│  5. Freeze old tuples if needed                                  │
│     └── Replace xmin with FrozenTransactionId                   │
│                                                                  │
│  6. Update visibility map                                        │
│     └── Mark pages as all-visible if applicable                 │
│                                                                  │
│  7. Update pg_class statistics (reltuples, relpages, etc.)      │
└─────────────────────────────────────────────────────────────────┘
```

### Dead Tuple Identification

A tuple is dead if:
1. Its `xmax` is set (deleted/updated)
2. The deleting transaction committed
3. No active transaction could possibly need this version

```sql
-- When is a tuple "dead" to VACUUM?

-- Transaction 100 deleted the row
-- All transactions that started before 100 have finished
-- Therefore: No transaction will ever see this row again → DEAD
```

### Free Space Map (FSM)

```
┌─────────────────────────────────────────────────────────────────┐
│  Free Space Map (FSM)                                            │
│                                                                  │
│  Stored in: <relfilenode>_fsm                                   │
│                                                                  │
│  Structure: Tree of pages tracking free space per heap page     │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Page 0: 2000 bytes free                                     │ │
│  │ Page 1: 0 bytes free (full)                                 │ │
│  │ Page 2: 4500 bytes free                                     │ │
│  │ Page 3: 8000 bytes free (nearly empty)                      │ │
│  │ ...                                                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Used by: INSERT to find page with enough space                 │
│  Updated by: VACUUM after reclaiming dead tuple space           │
└─────────────────────────────────────────────────────────────────┘
```

### Visibility Map (VM)

```
┌─────────────────────────────────────────────────────────────────┐
│  Visibility Map (VM)                                             │
│                                                                  │
│  Stored in: <relfilenode>_vm                                    │
│                                                                  │
│  Two bits per heap page:                                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Bit 1: All-Visible (all tuples visible to all transactions)│ │
│  │ Bit 2: All-Frozen (all tuples have frozen XIDs)            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Benefits:                                                       │
│  - VACUUM skips all-visible pages (faster!)                     │
│  - Index-only scans skip heap fetch for all-visible pages       │
│  - Anti-wraparound VACUUM skips all-frozen pages                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Deep Dive: Implementation Details

### Lazy VACUUM Algorithm

```c
/* Simplified from src/backend/access/heap/vacuumlazy.c */
void
lazy_scan_heap(LVRelState *vacrel)
{
    BlockNumber blkno;
    Buffer      buf;
    
    /* Calculate oldest XID that might need old versions */
    OldestXmin = GetOldestNonRemovableTransactionId();
    
    for (blkno = 0; blkno < nblocks; blkno++)
    {
        /* Skip if all-visible and no freezing needed */
        if (VM_ALL_VISIBLE(blkno) && !need_freeze)
            continue;
        
        buf = ReadBuffer(rel, blkno);
        LockBuffer(buf, BUFFER_LOCK_EXCLUSIVE);
        
        /* Scan all tuples on page */
        for (offnum = FirstOffsetNumber; offnum <= maxoff; offnum++)
        {
            HeapTupleHeader tuple = ...;
            
            switch (HeapTupleSatisfiesVacuum(tuple, OldestXmin))
            {
                case HEAPTUPLE_DEAD:
                    /* Mark for removal, add TID to dead list */
                    dead_tuples[ndead++] = ItemPointerMake(blkno, offnum);
                    break;
                    
                case HEAPTUPLE_LIVE:
                    /* Check if needs freezing */
                    if (TransactionIdPrecedes(tuple->t_xmin, FreezeLimit))
                        heap_freeze_tuple(tuple);
                    break;
                    
                case HEAPTUPLE_RECENTLY_DEAD:
                    /* Still visible to some transaction, skip */
                    break;
            }
        }
        
        UnlockReleaseBuffer(buf);
        
        /* Process dead tuples in batches to limit memory */
        if (ndead >= LAZY_ALLOC_TUPLES)
            lazy_vacuum_heap_rel(vacrel);
    }
    
    /* Final cleanup */
    lazy_vacuum_heap_rel(vacrel);
}
```

### Index Vacuuming

```c
/* Remove index entries pointing to dead heap tuples */
void
lazy_vacuum_index(LVRelState *vacrel, Relation indrel)
{
    IndexBulkDeleteResult *result;
    
    /* For each index on the table */
    result = index_bulk_delete(
        indrel,
        vacrel->dead_tuples,    /* List of dead TIDs */
        lazy_tid_reaped,        /* Callback to check if TID is dead */
        vacrel
    );
    
    /* Update index statistics */
    vacrel->num_index_scans++;
}
```

### Freezing

```c
/* Replace xmin with FrozenTransactionId */
void
heap_freeze_tuple(HeapTupleHeader tuple)
{
    /* Before: xmin = 1234567890 (old transaction) */
    /* After:  xmin = 2 (FrozenTransactionId) */
    
    tuple->t_infomask |= HEAP_XMIN_FROZEN;
    
    /* Frozen tuples are visible to ALL transactions forever */
}
```

---

## Autovacuum

### How Autovacuum Works

```
┌─────────────────────────────────────────────────────────────────┐
│  Autovacuum Daemon                                               │
│                                                                  │
│  Launcher Process:                                               │
│  └── Wakes every autovacuum_naptime (1 min default)             │
│      └── Checks all tables for vacuum/analyze thresholds        │
│          └── Launches worker processes as needed                │
│                                                                  │
│  Worker Process (up to autovacuum_max_workers):                 │
│  └── Picks a table needing work                                 │
│      └── Runs VACUUM and/or ANALYZE                             │
│          └── Respects cost limits to avoid I/O storms           │
│                                                                  │
│  Thresholds:                                                     │
│  VACUUM if: dead_tuples > threshold + scale_factor × reltuples │
│  ANALYZE if: changed_tuples > threshold + scale_factor × reltuples│
└─────────────────────────────────────────────────────────────────┘
```

### Autovacuum Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `autovacuum` | on | Enable autovacuum |
| `autovacuum_naptime` | 1min | Time between launcher runs |
| `autovacuum_max_workers` | 3 | Max concurrent workers |
| `autovacuum_vacuum_threshold` | 50 | Min dead tuples before vacuum |
| `autovacuum_vacuum_scale_factor` | 0.2 | Fraction of table triggering vacuum |
| `autovacuum_analyze_threshold` | 50 | Min changes before analyze |
| `autovacuum_analyze_scale_factor` | 0.1 | Fraction of table triggering analyze |
| `autovacuum_vacuum_cost_limit` | -1 | Cost limit per worker (-1 = use vacuum_cost_limit) |
| `autovacuum_vacuum_cost_delay` | 2ms | Delay when cost limit reached |

### Vacuum Trigger Calculation

```sql
-- VACUUM triggered when:
dead_tuples > autovacuum_vacuum_threshold + 
              autovacuum_vacuum_scale_factor × reltuples

-- Example: Table with 1,000,000 rows
-- Threshold: 50 + 0.2 × 1,000,000 = 200,050 dead tuples

-- For huge tables, reduce scale_factor:
ALTER TABLE huge_table SET (autovacuum_vacuum_scale_factor = 0.01);
```

### Per-Table Settings

```sql
-- Aggressive vacuum for frequently-updated table
ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 100,
    autovacuum_vacuum_cost_limit = 1000
);

-- Less frequent vacuum for append-only table
ALTER TABLE log_table SET (
    autovacuum_vacuum_scale_factor = 0.5,
    autovacuum_enabled = false  -- Disable completely
);
```

---

## VACUUM FULL vs Regular VACUUM

### Regular VACUUM (Lazy)

```
Before VACUUM:
┌────────────────────────────────────────────┐
│ Page 0: [Live][Dead][Dead][Live]    50% used │
│ Page 1: [Dead][Dead][Dead][Live]    25% used │
│ Page 2: [Live][Live][Live][Dead]    75% used │
└────────────────────────────────────────────┘
File size: 24KB (3 pages)

After VACUUM:
┌────────────────────────────────────────────┐
│ Page 0: [Live][Free][Free][Live]    50% available │
│ Page 1: [Free][Free][Free][Live]    75% available │
│ Page 2: [Live][Live][Live][Free]    25% available │
└────────────────────────────────────────────┘
File size: STILL 24KB (space marked reusable, file not shrunk)
```

### VACUUM FULL

```
Before VACUUM FULL:
┌────────────────────────────────────────────┐
│ Page 0: [Live][Dead][Dead][Live]           │
│ Page 1: [Dead][Dead][Dead][Live]           │
│ Page 2: [Live][Live][Live][Dead]           │
└────────────────────────────────────────────┘
File size: 24KB (3 pages)

After VACUUM FULL (rewrites entire table):
┌────────────────────────────────────────────┐
│ Page 0: [Live][Live][Live][Live][Live][Live]│
└────────────────────────────────────────────┘
File size: 8KB (1 page) - TABLE SHRUNK!
```

**VACUUM FULL costs:**
- Exclusive lock (blocks ALL reads and writes)
- Requires extra disk space (creates new copy)
- Rebuilds all indexes
- Very slow for large tables

### Alternative: pg_repack

```sql
-- pg_repack: Like VACUUM FULL but doesn't block reads
-- Install extension first
CREATE EXTENSION pg_repack;

-- Run from command line (minimal locking)
pg_repack -t mytable mydb
```

---

## Monitoring VACUUM

### Key System Views

```sql
-- When was table last vacuumed?
SELECT 
    relname,
    last_vacuum,
    last_autovacuum,
    n_dead_tup,
    n_live_tup,
    n_mod_since_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Currently running vacuums
SELECT 
    pid,
    datname,
    relid::regclass,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed,
    index_vacuum_count,
    max_dead_tuples,
    num_dead_tuples
FROM pg_stat_progress_vacuum;

-- Autovacuum activity
SELECT * FROM pg_stat_activity 
WHERE backend_type = 'autovacuum worker';
```

### Identifying Bloated Tables

```sql
-- Estimate table bloat (requires pgstattuple extension)
CREATE EXTENSION pgstattuple;

SELECT 
    table_len,
    tuple_count,
    dead_tuple_count,
    dead_tuple_len,
    round(100.0 * dead_tuple_len / nullif(table_len, 0), 1) AS dead_pct
FROM pgstattuple('mytable');

-- Quick bloat estimate without loading extension
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

### VACUUM VERBOSE Output

```sql
VACUUM (VERBOSE) mytable;
```

```
INFO:  vacuuming "public.mytable"
INFO:  scanned index "mytable_pkey" to remove 15000 row versions
INFO:  "mytable": removed 15000 row versions in 500 pages
INFO:  "mytable": found 15000 removable, 85000 nonremovable row versions 
       in 1000 out of 1000 pages
INFO:  "mytable": removed 0 dead item identifiers in 0 pages
INFO:  "mytable": truncated 1000 to 800 pages
VACUUM
```

---

## Hands-On Exercises

### Exercise 1: Observing Dead Tuples (Basic)

```sql
CREATE TABLE vacuum_test (id serial, data text);
INSERT INTO vacuum_test SELECT i, 'row ' || i FROM generate_series(1, 10000) i;

-- Check live/dead counts
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables 
WHERE relname = 'vacuum_test';

-- Create dead tuples
DELETE FROM vacuum_test WHERE id <= 5000;

-- Wait for stats collector (or force)
SELECT pg_stat_force_stats();

SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables 
WHERE relname = 'vacuum_test';

-- VACUUM cleans up
VACUUM vacuum_test;

SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables 
WHERE relname = 'vacuum_test';
```

### Exercise 2: FSM and Space Reuse (Intermediate)

```sql
-- Check table size before
SELECT pg_size_pretty(pg_relation_size('vacuum_test'));

-- Delete half the rows
DELETE FROM vacuum_test WHERE id % 2 = 0;
VACUUM vacuum_test;

-- Check size after (same!)
SELECT pg_size_pretty(pg_relation_size('vacuum_test'));

-- Insert new rows (reuses freed space)
INSERT INTO vacuum_test SELECT i, 'new row ' || i FROM generate_series(20001, 25000) i;

-- Check size (minimal growth because space was reused)
SELECT pg_size_pretty(pg_relation_size('vacuum_test'));
```

### Exercise 3: Monitoring VACUUM Progress (Intermediate)

```sql
-- In terminal 1: Start a long vacuum
CREATE TABLE big_table AS SELECT generate_series(1, 5000000) AS id;
DELETE FROM big_table WHERE id % 2 = 0;
VACUUM (VERBOSE) big_table;

-- In terminal 2: Monitor progress
SELECT 
    phase,
    heap_blks_scanned,
    heap_blks_vacuumed,
    heap_blks_total,
    round(100.0 * heap_blks_scanned / heap_blks_total, 1) AS pct_complete
FROM pg_stat_progress_vacuum;
```

### Exercise 4: Autovacuum Configuration (Advanced)

```sql
-- Check current autovacuum settings for a table
SELECT 
    relname,
    reloptions
FROM pg_class
WHERE relname = 'vacuum_test';

-- Set aggressive autovacuum
ALTER TABLE vacuum_test SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 10
);

-- Verify settings
SELECT * FROM pg_stat_user_tables WHERE relname = 'vacuum_test';

-- Generate activity and wait for autovacuum
UPDATE vacuum_test SET data = data || ' updated';
-- Watch pg_stat_activity for autovacuum worker
```

### Exercise 5: VACUUM FULL Comparison (Advanced)

```sql
-- Create bloated table
CREATE TABLE bloat_demo AS SELECT generate_series(1, 100000) AS id;
SELECT pg_size_pretty(pg_relation_size('bloat_demo')) AS before_size;

-- Create bloat
DELETE FROM bloat_demo WHERE id > 10000;
VACUUM bloat_demo;
SELECT pg_size_pretty(pg_relation_size('bloat_demo')) AS after_vacuum;
-- Size unchanged!

-- VACUUM FULL shrinks
VACUUM FULL bloat_demo;
SELECT pg_size_pretty(pg_relation_size('bloat_demo')) AS after_full;
-- Size reduced!
```

---

## Key Takeaways

1. **VACUUM is essential**: MVCC creates dead tuples that must be cleaned up.

2. **Regular VACUUM doesn't shrink files**: It marks space as reusable for new rows.

3. **VACUUM FULL shrinks but blocks**: Only use as last resort; consider pg_repack.

4. **Autovacuum handles most cases**: Configure thresholds for your workload.

5. **FSM tracks free space**: Updated by VACUUM, used by INSERT.

6. **VM enables optimizations**: Index-only scans and faster VACUUM.

7. **Monitor dead tuple counts**: High counts indicate VACUUM can't keep up.

8. **Per-table settings matter**: Hot tables may need aggressive autovacuum.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/access/heap/vacuumlazy.c` | Lazy vacuum implementation |
| `src/backend/commands/vacuum.c` | VACUUM command handling |
| `src/backend/postmaster/autovacuum.c` | Autovacuum daemon |
| `src/backend/access/heap/visibilitymap.c` | Visibility map |
| `src/backend/storage/freespace/freespace.c` | Free space map |

### Documentation

- [PostgreSQL Docs: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [PostgreSQL Docs: VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html)
- [PostgreSQL Docs: Autovacuum](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html)

---

## References

### Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `vacuum_cost_delay` | 0 | Delay when cost limit reached |
| `vacuum_cost_page_hit` | 1 | Cost of reading cached page |
| `vacuum_cost_page_miss` | 2 | Cost of reading from disk |
| `vacuum_cost_page_dirty` | 20 | Cost of dirtying a page |
| `vacuum_cost_limit` | 200 | Max cost before sleeping |
| `vacuum_freeze_min_age` | 50M | Min age before freezing |
| `vacuum_freeze_table_age` | 150M | Trigger aggressive freeze |
