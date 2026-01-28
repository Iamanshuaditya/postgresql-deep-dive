# Lesson 33: Shared Memory — PostgreSQL's Common Data Structures

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 3 — Memory Management |
| **Lesson** | 33 of 66 |
| **Estimated Time** | 60-75 minutes |
| **Prerequisites** | Lesson 11: The Buffer Manager, Lesson 21: Background Processes |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Understand PostgreSQL's shared memory architecture
2. Identify key shared memory structures and their purposes
3. Monitor shared memory usage
4. Configure shared memory for optimal performance
5. Troubleshoot shared memory issues

### Key Terms

| Term | Definition |
|------|------------|
| **Shared Memory** | Memory accessible by all PostgreSQL processes |
| **shared_buffers** | Buffer cache for data pages |
| **WAL Buffers** | Write-ahead log buffer |
| **CLOG** | Commit status cache |
| **Lock Tables** | Shared lock management |
| **Proc Array** | Array of process information |

---

## Introduction

PostgreSQL uses a multi-process architecture where processes share memory:

```
┌─────────────────────────────────────────────────────────────────┐
│  PostgreSQL Shared Memory Architecture                           │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    SHARED MEMORY                            ││
│  │  ┌────────────────────────────────────────────────────────┐ ││
│  │  │              shared_buffers (25% RAM)                  │ ││
│  │  │              Buffer Cache for Data Pages               │ ││
│  │  └────────────────────────────────────────────────────────┘ ││
│  │  ┌──────────────┐  ┌────────────┐  ┌───────────────────┐   ││
│  │  │ WAL Buffers  │  │   CLOG     │  │   Lock Tables     │   ││
│  │  │  (16MB)      │  │  (Commit   │  │   (Shared Lock    │   ││
│  │  │              │  │   Status)  │  │    Manager)       │   ││
│  │  └──────────────┘  └────────────┘  └───────────────────┘   ││
│  │  ┌──────────────┐  ┌────────────┐  ┌───────────────────┐   ││
│  │  │  Proc Array  │  │ Subtrans   │  │   Other Caches    │   ││
│  │  │  (Process    │  │  Cache     │  │   (SLRU, etc.)    │   ││
│  │  │   Info)      │  │            │  │                   │   ││
│  │  └──────────────┘  └────────────┘  └───────────────────┘   ││
│  └─────────────────────────────────────────────────────────────┘│
│        ↑               ↑               ↑               ↑        │
│        │               │               │               │        │
│  ┌─────┴─────┐  ┌──────┴──────┐  ┌─────┴─────┐  ┌─────┴─────┐  │
│  │ Backend 1 │  │  Backend 2  │  │  BG Writer │  │ Autovacuum│  │
│  │ (Client)  │  │  (Client)   │  │            │  │           │  │
│  └───────────┘  └─────────────┘  └───────────┘  └───────────┘  │
│                                                                  │
│  Each process accesses shared memory directly                   │
│  Synchronization via LWLocks and spinlocks                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Shared Memory Components

### 1. Buffer Pool (shared_buffers)

The largest component—caches data pages:

```sql
-- Check size
SHOW shared_buffers;  -- Default: 128MB (way too low!)

-- Recommended: 25% of RAM
-- For 32GB server: shared_buffers = 8GB

-- View buffer usage
SELECT 
    c.relname,
    count(*) AS buffers,
    pg_size_pretty(count(*) * 8192) AS size
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 10;
```

### 2. WAL Buffers

Buffers for write-ahead log before flushing to disk:

```sql
SHOW wal_buffers;  -- Default: -1 (auto = 3% of shared_buffers)

-- Usually auto-tuned, rarely needs adjustment
-- Max effective: 64MB
```

### 3. CLOG (Commit Log / pg_xact)

Tracks transaction commit status:

```sql
-- Status values:
-- IN_PROGRESS = 0
-- COMMITTED = 1
-- ABORTED = 2
-- SUB_COMMITTED = 3

-- CLOG is memory-cached, backed by files in pg_xact/
-- Each transaction = 2 bits
-- Each page = 32KB = tracks 128K transactions
```

### 4. Lock Tables

Shared lock manager structures:

```sql
-- View lock table usage
SELECT 
    locktype, 
    count(*) 
FROM pg_locks 
GROUP BY locktype;

-- Configuration
SHOW max_locks_per_transaction;  -- Default: 64
SHOW max_pred_locks_per_transaction;  -- For serializable
```

### 5. Proc Array

Information about all PostgreSQL processes:

```sql
-- Visible through pg_stat_activity
SELECT pid, state, query FROM pg_stat_activity;

-- Configuration
SHOW max_connections;  -- Affects proc array size
```

---

## Configuring Shared Memory

### Key Parameters

```sql
-- Buffer cache (biggest impact)
SHOW shared_buffers;  -- 25% of RAM recommended

-- WAL buffers (usually fine on auto)
SHOW wal_buffers;  -- Auto-tuned

-- Connection slots (affects multiple structures)
SHOW max_connections;

-- Prepared transaction tracking
SHOW max_prepared_transactions;

-- Locks per transaction
SHOW max_locks_per_transaction;
```

### Calculating Shared Memory Size

```
Approximate shared memory usage:
  shared_buffers: As configured
  + WAL buffers: 16MB (or configured)
  + max_connections × ~400 bytes (Proc array)
  + max_connections × max_locks_per_transaction × 200 bytes (Lock table)
  + CLOG cache: ~128KB per 256 active transactions
  + Other structures: ~10-50MB

For a typical production server:
  shared_buffers = 8GB
  + other = ~100MB
  = ~8.1GB shared memory
```

### OS Configuration

```bash
# Linux: Check current limits
cat /proc/sys/kernel/shmmax   # Max segment size
cat /proc/sys/kernel/shmall   # Total shared memory

# Set limits (as root)
sysctl -w kernel.shmmax=8589934592  # 8GB
sysctl -w kernel.shmall=2097152

# Permanent: /etc/sysctl.conf
kernel.shmmax = 8589934592
kernel.shmall = 2097152

# Modern Linux with huge pages (recommended for large shared_buffers)
vm.nr_hugepages = 4100  # For 8GB (4100 × 2MB pages)
```

---

## Huge Pages

### Why Huge Pages?

```
Normal pages: 4KB each
  8GB shared_buffers = 2 million pages
  2 million page table entries!
  TLB pressure, slower memory access

Huge pages: 2MB each (Linux)
  8GB shared_buffers = 4096 huge pages
  4096 page table entries
  Much better TLB performance!
```

### Configuring Huge Pages

```bash
# 1. Calculate needed huge pages
# shared_buffers = 8GB = 8192MB
# Huge page size = 2MB
# Pages needed = 8192 / 2 = 4096 + overhead = 4200

# 2. Configure Linux
echo 4200 > /proc/sys/vm/nr_hugepages

# Or permanent in /etc/sysctl.conf
vm.nr_hugepages = 4200

# 3. Configure PostgreSQL
# postgresql.conf
huge_pages = on  # try, on, or off
```

```sql
-- Check if huge pages are being used
SHOW huge_pages;

-- Check allocation (Linux)
-- grep HugePages /proc/meminfo
```

---

## SLRU Caches

### What is SLRU?

**S**imple **L**east **R**ecently **U**sed caches:

```
┌─────────────────────────────────────────────────────────────────┐
│  SLRU Caches in PostgreSQL                                       │
│                                                                  │
│  ┌─────────────────┐  Commit status (pg_xact)                   │
│  │   CLOG Cache    │  2 bits per transaction                    │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐  Subtransaction parent map (pg_subtrans)   │
│  │ Subtrans Cache  │  For savepoints                            │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐  Multixact info (pg_multixact)             │
│  │  Multixact      │  For row locks held by multiple xids      │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐  Prepared transaction data                 │
│  │ Notify/Async    │  LISTEN/NOTIFY queue                        │
│  └─────────────────┘                                            │
│                                                                  │
│  Each cache:                                                    │
│  • Fixed size in shared memory                                  │
│  • LRU eviction                                                 │
│  • Backed by files on disk                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Viewing SLRU Statistics

```sql
-- PostgreSQL 13+
SELECT 
    name,
    blks_zeroed,
    blks_hit,
    blks_read,
    blks_written,
    blks_exists
FROM pg_stat_slru;

-- Low hit rate = cache pressure
-- Consider increasing if possible (few parameters available)
```

---

## Monitoring Shared Memory

### Overall Usage

```sql
-- View shared memory allocation (PG 16+)
SELECT * FROM pg_shmem_allocations 
ORDER BY size DESC 
LIMIT 20;

-- Older versions: Estimate from configuration
SELECT 
    (SELECT setting::bigint * 8192 FROM pg_settings WHERE name = 'shared_buffers') AS buffer_cache,
    (SELECT setting::bigint * 1024 FROM pg_settings WHERE name = 'wal_buffers' WHERE setting != '-1') AS wal_buffers;
```

### Buffer Cache Efficiency

```sql
-- Requires pg_buffercache extension
CREATE EXTENSION pg_buffercache;

-- Buffer usage by relation
SELECT 
    c.relname,
    count(*) AS buffers,
    round(100.0 * count(*) / (SELECT count(*) FROM pg_buffercache), 2) AS pct_of_cache,
    round(100.0 * count(*) FILTER (WHERE b.usagecount > 3) / count(*), 2) AS pct_hot
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 20;
```

### Buffer Cache Hit Rate

```sql
-- Cache hit ratio (should be >99% for OLTP)
SELECT 
    sum(blks_hit) * 100.0 / sum(blks_hit + blks_read) AS hit_ratio
FROM pg_stat_database 
WHERE datname = current_database();
```

---

## Synchronization Primitives

### LWLocks (Lightweight Locks)

```sql
-- View LWLock contention
SELECT 
    wait_event,
    count(*)
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock'
GROUP BY wait_event
ORDER BY count(*) DESC;

-- Common LWLocks:
-- BufferContent: Reading/writing buffer
-- BufferMapping: Finding buffer
-- WALWrite: Writing WAL
-- ProcArray: Transaction snapshot
```

### Spinlocks

Very short-duration locks, not visible in pg_stat_activity.

Used for quick operations like:
- Buffer header access
- Spinlock counters
- Quick flag updates

---

## Troubleshooting

### "could not resize shared memory segment"

```bash
# Increase OS shared memory limits
sysctl -w kernel.shmmax=8589934592
sysctl -w kernel.shmall=2097152
```

### High LWLock Contention

```sql
-- Find what's waiting
SELECT 
    pid,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock';

-- Common causes:
-- BufferMapping: Too many processes reading different pages
-- WALWrite: WAL disk too slow
-- ProcArray: Long-running transactions
```

### Low Buffer Cache Hit Rate

```sql
-- Check hit rate
SELECT * FROM pg_stat_database;

-- Solutions:
-- 1. Increase shared_buffers
-- 2. Reduce working set size
-- 3. Add indexes to reduce scans
```

---

## Hands-On Exercises

### Exercise 1: View Shared Memory Usage (Basic)

```sql
-- Check current configuration
SHOW shared_buffers;
SHOW wal_buffers;
SHOW max_connections;

-- Calculate approximate shared memory
SELECT 
    pg_size_pretty(
        (SELECT setting::bigint * 8192 FROM pg_settings WHERE name = 'shared_buffers')
        + 16 * 1024 * 1024  -- WAL buffers estimate
        + (SELECT setting::bigint * 400 FROM pg_settings WHERE name = 'max_connections')
    ) AS estimated_shared_memory;
```

### Exercise 2: Buffer Cache Analysis (Intermediate)

```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

-- What's in the buffer cache?
SELECT 
    CASE 
        WHEN c.relname IS NULL THEN 'empty/other'
        ELSE c.relname 
    END AS object,
    count(*) AS buffers,
    pg_size_pretty(count(*) * 8192) AS size
FROM pg_buffercache b
LEFT JOIN pg_class c ON b.relfilenode = c.relfilenode 
    AND b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 10;
```

### Exercise 3: LWLock Monitoring (Advanced)

```sql
-- Monitor LWLock waits for 10 seconds
SELECT 
    wait_event,
    count(*),
    array_agg(DISTINCT pid) AS waiting_pids
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock'
GROUP BY wait_event
ORDER BY count(*) DESC;

-- Run this multiple times to see patterns
```

---

## Key Takeaways

1. **Shared memory is central**: All processes access it.

2. **shared_buffers is key**: Set to 25% of RAM.

3. **Huge pages improve performance**: For large buffer caches.

4. **LWLocks coordinate access**: Monitor for contention.

5. **Buffer hit rate matters**: Should be >99% for OLTP.

6. **OS limits must be configured**: shmmax, shmall.

7. **SLRU caches are fixed size**: Back various internal structures.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/storage/ipc/shmem.c` | Shared memory management |
| `src/backend/storage/lmgr/lwlock.c` | Lightweight locks |
| `src/backend/storage/buffer/bufmgr.c` | Buffer manager |
| `src/backend/access/transam/slru.c` | SLRU implementation |

### Documentation

- [PostgreSQL Docs: Shared Memory](https://www.postgresql.org/docs/current/runtime-config-resource.html#RUNTIME-CONFIG-RESOURCE-MEMORY)
- [PostgreSQL Docs: Kernel Resources](https://www.postgresql.org/docs/current/kernel-resources.html)

---

## References

### Key Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| shared_buffers | 128MB | Buffer cache size |
| wal_buffers | -1 (auto) | WAL buffer size |
| huge_pages | try | Use huge pages |
| max_connections | 100 | Affects proc array |
| max_locks_per_transaction | 64 | Lock table size |

### Shared Memory Components

| Component | Purpose | Sizing |
|-----------|---------|--------|
| Buffer Pool | Page cache | shared_buffers |
| WAL Buffers | WAL staging | wal_buffers |
| CLOG | Commit status | Fixed |
| Lock Tables | Lock manager | Based on max_connections |
| Proc Array | Process info | Based on max_connections |
