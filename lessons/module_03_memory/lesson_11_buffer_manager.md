# Lesson 11: The Buffer Manager — Caching Data in Memory

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 3 — Memory Management |
| **Lesson** | 11 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 8: Heap Storage and Page Layout |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain the role of the buffer manager in PostgreSQL architecture
2. Describe the three-layer structure: buffer pool, descriptors, buffer table
3. Understand the pin/unpin mechanism for concurrency
4. Explain the clock-sweep replacement algorithm
5. Monitor and tune shared_buffers effectively

### Key Terms

| Term | Definition |
|------|------------|
| **Buffer Manager** | Subsystem that manages shared memory for data pages |
| **Shared Buffers** | The shared memory area holding cached pages |
| **Buffer Pool** | Array of page-sized slots in shared memory |
| **Buffer Descriptor** | Metadata about a buffer slot (pin count, dirty, etc.) |
| **Buffer Table** | Hash table mapping (page tag) → (buffer slot) |
| **Pin** | Mark a buffer as in-use (increment reference count) |
| **Dirty** | A buffer whose contents differ from disk |
| **Clock-Sweep** | Algorithm for selecting buffers to evict |

---

## Introduction

Every time PostgreSQL reads or writes data, it goes through the **buffer manager**. This subsystem:

1. Caches frequently-accessed pages in memory (**shared_buffers**)
2. Tracks which buffers are in use and which can be evicted
3. Manages concurrent access to the same page
4. Decides which pages to evict when memory is full
5. Ensures dirty pages are written back before eviction

The buffer manager is the bridge between the storage layer (disk files) and the executor (query processing). Understanding it helps you:

- **Size shared_buffers correctly**: Too small wastes memory; too large can hurt too
- **Interpret buffer statistics**: EXPLAIN (BUFFERS) output makes more sense
- **Diagnose I/O issues**: High read counts may indicate cache misses
- **Understand write behavior**: Dirty buffers and checkpoints

> **Key Insight**: PostgreSQL uses a **double-caching** strategy. Data is cached both in PostgreSQL's shared_buffers AND in the operating system's filesystem cache. This is why shared_buffers doesn't need to use all available RAM.

---

## Conceptual Foundation

### The Three-Layer Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    SHARED MEMORY                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  BUFFER TABLE (Hash Table)                                │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │ Key: BufferTag (relfilenode, fork, blocknum)       │  │   │
│  │  │ Value: buffer_id (index into descriptor array)     │  │   │
│  │  │                                                     │  │   │
│  │  │ Tag(rel=16384,fork=0,blk=0) → buffer_id=42         │  │   │
│  │  │ Tag(rel=16384,fork=0,blk=1) → buffer_id=17         │  │   │
│  │  │ Tag(rel=16385,fork=0,blk=0) → buffer_id=103        │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  BUFFER DESCRIPTORS (Array)                               │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │ [0] tag | flags | refcount | usage_count | ...     │   │   │
│  │  │ [1] tag | flags | refcount | usage_count | ...     │   │   │
│  │  │ [2] tag | flags | refcount | usage_count | ...     │   │   │
│  │  │ ...                                                 │   │   │
│  │  │ [N] tag | flags | refcount | usage_count | ...     │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼ (1:1 mapping)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  BUFFER POOL (Array of 8KB pages)                         │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │ [0] ████████████  8KB page data  ██████████████    │   │   │
│  │  │ [1] ████████████  8KB page data  ██████████████    │   │   │
│  │  │ [2] ████████████  8KB page data  ██████████████    │   │   │
│  │  │ ...                                                 │   │   │
│  │  │ [N] ████████████  8KB page data  ██████████████    │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Buffer Tag: Page Identity

Each page in the buffer is identified by a **BufferTag**:

```c
typedef struct BufferTag
{
    RelFileNode rnode;      /* Database, tablespace, relfilenode */
    ForkNumber  forkNum;    /* Main=0, FSM=1, VM=2 */
    BlockNumber blockNum;   /* Which block */
} BufferTag;
```

Example: Block 5 of the main fork of table 16384 in database 12345:
```
{ rnode: {dbOid=12345, spcOid=1663, relNode=16384}, 
  forkNum: 0, 
  blockNum: 5 }
```

### Page Lifecycle in the Buffer

```
                    ┌──────────────────┐
                    │  Page not in     │
                    │  buffer          │
                    └────────┬─────────┘
                             │
                   ReadBuffer(rel, blocknum)
                             │
                             ▼
              ┌──────────────────────────────┐
              │ Check buffer table:          │
              │   Tag → buffer_id exists?    │
              └──────┬──────────────┬────────┘
                     │              │
                   Yes            No
                     │              │
              ┌──────▼──────┐   ┌───▼───────────────┐
              │   HIT!      │   │  1. Find victim   │
              │  Pin buffer │   │  2. Evict if dirty│
              └─────────────┘   │  3. Read from disk│
                                │  4. Pin buffer    │
                                └───────────────────┘
                                         │
                     ┌───────────────────┘
                     ▼
              ┌─────────────────┐
              │ Buffer pinned   │
              │ refcount++      │
              │ usage_count++   │
              └────────┬────────┘
                       │
         Process works with page
                       │
              ReleaseBuffer()
                       │
              ┌────────▼────────┐
              │ Buffer unpinned │
              │ refcount--      │
              └────────┬────────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
  refcount > 0               refcount == 0
        │                             │
   Still pinned              Candidate for eviction
   (other users)             (when needed)
```

### Pin/Unpin Mechanism

**Pinning** protects a buffer from eviction:

```c
/* Simplified pin operation */
void PinBuffer(BufferDesc *buf)
{
    /* Increment reference count */
    buf->refcount++;
    
    /* Increment usage count (for clock sweep) */
    buf->usage_count++;
}

void UnpinBuffer(BufferDesc *buf)
{
    /* Decrement reference count */
    buf->refcount--;
}
```

- **refcount > 0**: Buffer is in use, cannot be evicted
- **refcount == 0**: Buffer can be considered for eviction
- **usage_count**: How "popular" is this buffer (for replacement)

### Clock-Sweep Replacement Algorithm

When PostgreSQL needs a buffer slot but all are occupied:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Clock-Sweep Algorithm                        │
│                                                                  │
│    Buffer Array (circular):                                      │
│                                                                  │
│       [0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]  ...              │
│        │    │    │    │    │    │    │    │                     │
│       u=2  u=0  u=3  u=0  u=1  u=0  u=2  u=0                    │
│       p=1  p=0  p=0  p=0  p=0  p=0  p=1  p=0                    │
│             ↑                                                    │
│          victim                                                  │
│             │                                                    │
│    Sweep hand checks each buffer:                               │
│    1. If pinned (p>0): skip                                     │
│    2. If usage_count > 0: decrement, skip                       │
│    3. If usage_count == 0 AND unpinned: EVICT!                  │
│                                                                  │
│    In this example:                                             │
│    - [0]: pinned, skip                                          │
│    - [1]: usage=0, unpinned → EVICT THIS ONE                    │
└─────────────────────────────────────────────────────────────────┘
```

The algorithm sweeps around the buffer array like a clock hand:
1. Check if buffer is pinned → skip if yes
2. Check usage_count → if > 0, decrement and skip
3. If usage_count == 0 and unpinned → this is the victim

**Why it works**: Frequently-accessed buffers have high usage_count. The sweep gradually decrements counts, so "cold" buffers eventually reach 0 and get evicted.

---

## Deep Dive: Implementation Details

### Buffer Descriptor Structure

```c
/* From src/include/storage/buf_internals.h (simplified) */
typedef struct BufferDesc
{
    BufferTag   tag;            /* ID of page stored in buffer */
    int         buf_id;         /* Buffer's index (0-based) */
    
    /* State info */
    pg_atomic_uint32 state;     /* Packed state: refcount, usage, flags */
    
    /* Lock info */
    int         wait_backend_pid;
    LWLock      content_lock;   /* Protects buffer contents */
    
    /* Freelist link */
    int         freeNext;       /* Next free buffer, or -1 */
} BufferDesc;

/* State flags (packed into state field) */
#define BM_LOCKED           (1U << 22)  /* Content lock held */
#define BM_DIRTY            (1U << 23)  /* Page has been modified */
#define BM_VALID            (1U << 24)  /* Page contains valid data */
#define BM_TAG_VALID        (1U << 25)  /* Tag is valid */
#define BM_IO_IN_PROGRESS   (1U << 26)  /* I/O in progress */
#define BM_IO_ERROR         (1U << 27)  /* I/O failed */
#define BM_JUST_DIRTIED     (1U << 28)  /* Dirtied since last write */
#define BM_CHECKPOINT_NEEDED (1U << 29) /* Needs checkpoint write */
```

### ReadBuffer: Getting a Page

```c
/* Simplified from src/backend/storage/buffer/bufmgr.c */
Buffer
ReadBuffer(Relation reln, BlockNumber blockNum)
{
    BufferTag   tag;
    int         buf_id;
    bool        found;
    
    /* Create tag for this page */
    INIT_BUFFERTAG(tag, reln->rd_node, MAIN_FORKNUM, blockNum);
    
    /* Look up in buffer table */
    buf_id = BufTableLookup(&tag);
    
    if (buf_id >= 0)
    {
        /* HIT: Page already in buffer */
        PinBuffer(&BufferDescriptors[buf_id]);
        return BufferDescriptorGetBuffer(&BufferDescriptors[buf_id]);
    }
    
    /* MISS: Need to load from disk */
    
    /* 1. Get a victim buffer */
    buf_id = StrategyGetBuffer();  /* Clock-sweep */
    
    /* 2. If victim is dirty, write it out */
    BufferDesc *buf = &BufferDescriptors[buf_id];
    if (buf->state & BM_DIRTY)
        FlushBuffer(buf);
    
    /* 3. Clear old tag, set new tag */
    BufTableDelete(&buf->tag);
    buf->tag = tag;
    BufTableInsert(&tag, buf_id);
    
    /* 4. Read page from disk */
    smgrread(reln->rd_smgr, MAIN_FORKNUM, blockNum,
             BufferGetPage(buf_id));
    
    /* 5. Pin and return */
    PinBuffer(buf);
    return BufferDescriptorGetBuffer(buf);
}
```

### StrategyGetBuffer: Clock-Sweep

```c
/* Simplified from src/backend/storage/buffer/freelist.c */
int
StrategyGetBuffer(void)
{
    BufferDesc *buf;
    int         trycounter;
    
    /* Maximum attempts = 2 * NBuffers */
    trycounter = NBuffers * 2;
    
    for (;;)
    {
        buf = &BufferDescriptors[StrategyControl->nextVictimBuffer];
        
        /* Advance clock hand */
        StrategyControl->nextVictimBuffer++;
        if (StrategyControl->nextVictimBuffer >= NBuffers)
            StrategyControl->nextVictimBuffer = 0;
        
        /* Skip if pinned */
        if (buf->refcount > 0)
            continue;
        
        /* Skip if usage_count > 0; decrement it */
        if (buf->usage_count > 0)
        {
            buf->usage_count--;
            continue;
        }
        
        /* Found victim: usage=0, unpinned */
        return buf->buf_id;
        
        if (--trycounter <= 0)
            elog(ERROR, "no unpinned buffers available");
    }
}
```

### Dirty Buffer Handling

```c
/* Mark buffer as dirty (modified) */
void
MarkBufferDirty(Buffer buffer)
{
    BufferDesc *buf = GetBufferDescriptor(buffer - 1);
    
    /* Set dirty flag */
    buf->state |= BM_DIRTY | BM_JUST_DIRTIED;
}

/* Flush dirty buffer to disk */
void
FlushBuffer(BufferDesc *buf)
{
    /* Write page to disk */
    smgrwrite(reln->rd_smgr, buf->tag.forkNum, buf->tag.blockNum,
              BufferGetPage(buf->buf_id), false);
    
    /* Clear dirty flag */
    buf->state &= ~(BM_DIRTY | BM_JUST_DIRTIED);
}
```

---

## Monitoring Buffer Usage

### Buffer Cache Hit Ratio

```sql
SELECT 
    sum(blks_hit) AS hits,
    sum(blks_read) AS reads,
    round(100.0 * sum(blks_hit) / nullif(sum(blks_hit) + sum(blks_read), 0), 2) 
        AS hit_ratio
FROM pg_stat_database;
```

A healthy system typically has 95%+ hit ratio.

### Buffer Content Analysis

```sql
-- Install extension
CREATE EXTENSION pg_buffercache;

-- See what's in the buffer cache
SELECT 
    c.relname,
    count(*) AS buffers,
    pg_size_pretty(count(*) * 8192) AS buffered_size,
    round(100.0 * count(*) / (SELECT count(*) FROM pg_buffercache), 2) AS pct
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 20;
```

### Per-Table Buffer Usage

```sql
SELECT 
    c.relname,
    count(*) AS buffers,
    sum(CASE WHEN b.isdirty THEN 1 ELSE 0 END) AS dirty_buffers,
    sum(b.usagecount) AS total_usage
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY buffers DESC;
```

### EXPLAIN BUFFERS Output

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;
```

```
 Index Scan using orders_customer_idx on orders  (...)
   Buffers: shared hit=15 read=3
```

- **shared hit=15**: 15 pages found in buffer cache
- **shared read=3**: 3 pages read from disk (cache miss)

---

## Performance Tuning

### Sizing shared_buffers

**Recommendations:**
- Start with 25% of system RAM
- For dedicated servers, try up to 40%
- More is not always better (OS cache also matters)

```sql
-- Current setting
SHOW shared_buffers;

-- Change (requires restart)
ALTER SYSTEM SET shared_buffers = '4GB';
```

### Monitoring for Right-Sizing

```sql
-- If hit ratio is low, consider increasing shared_buffers
SELECT 
    datname,
    blks_hit,
    blks_read,
    round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) AS hit_pct
FROM pg_stat_database
WHERE datname NOT LIKE 'template%';

-- If most of shared_buffers are unused, you have too much
SELECT 
    count(*) AS total_buffers,
    count(*) FILTER (WHERE relfilenode IS NULL) AS unused_buffers,
    round(100.0 * count(*) FILTER (WHERE relfilenode IS NULL) / count(*), 2) AS unused_pct
FROM pg_buffercache;
```

### Huge Pages

For large shared_buffers (>8GB), huge pages reduce memory management overhead:

```bash
# In postgresql.conf
huge_pages = try  # or 'on'

# Configure OS (Linux)
echo 4096 > /proc/sys/vm/nr_hugepages  # For 8GB at 2MB pages
```

### effective_cache_size

This tells the planner how much total memory is available for caching (shared_buffers + OS cache):

```sql
-- Should be ~50-75% of system RAM
SET effective_cache_size = '12GB';
```

This affects cost estimates, not actual memory allocation.

---

## Hands-On Exercises

### Exercise 1: Buffer Cache Analysis (Basic)

```sql
-- Install pg_buffercache
CREATE EXTENSION pg_buffercache;

-- Create and load a table
CREATE TABLE buffer_test AS SELECT generate_series(1, 100000) AS id;

-- Check buffer cache
SELECT count(*) FROM pg_buffercache 
WHERE relfilenode = (SELECT relfilenode FROM pg_class WHERE relname = 'buffer_test');

-- Create some activity
SELECT count(*) FROM buffer_test WHERE id > 50000;

-- Check again
SELECT count(*) FROM pg_buffercache 
WHERE relfilenode = (SELECT relfilenode FROM pg_class WHERE relname = 'buffer_test');
```

### Exercise 2: Hit Ratio Monitoring (Basic)

```sql
-- Get baseline
SELECT 
    datname,
    blks_hit, blks_read,
    round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) AS hit_pct
FROM pg_stat_database
WHERE datname = current_database();

-- Run some queries
SELECT count(*) FROM pg_class;
SELECT * FROM pg_attribute LIMIT 100;

-- Check again
SELECT 
    datname,
    blks_hit, blks_read,
    round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) AS hit_pct
FROM pg_stat_database
WHERE datname = current_database();
```

### Exercise 3: EXPLAIN BUFFERS (Intermediate)

```sql
-- Create table
CREATE TABLE orders_buf (id serial, customer_id int, total numeric);
INSERT INTO orders_buf (customer_id, total) 
SELECT random() * 1000, random() * 1000 FROM generate_series(1, 100000);
CREATE INDEX ON orders_buf(customer_id);

-- Clear cache (if you can restart PostgreSQL) or just note current state

-- First query (expect reads)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders_buf WHERE customer_id = 500;

-- Same query (expect hits)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders_buf WHERE customer_id = 500;

-- Different query (might have some reads)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders_buf WHERE customer_id = 999;
```

### Exercise 4: Dirty Buffer Analysis (Intermediate)

```sql
-- Check dirty buffers before changes
SELECT count(*) AS total_dirty
FROM pg_buffercache 
WHERE isdirty = true;

-- Make some changes
UPDATE buffer_test SET id = id + 1 WHERE id < 1000;

-- Check dirty buffers after
SELECT count(*) AS total_dirty
FROM pg_buffercache 
WHERE isdirty = true;

-- Checkpoint writes dirty buffers
CHECKPOINT;

-- Check again
SELECT count(*) AS total_dirty
FROM pg_buffercache 
WHERE isdirty = true;
```

### Exercise 5: Usage Count Distribution (Advanced)

```sql
-- See usage count distribution
SELECT 
    usagecount,
    count(*) AS buffers,
    round(100.0 * count(*) / (SELECT count(*) FROM pg_buffercache), 2) AS pct
FROM pg_buffercache
WHERE relfilenode IS NOT NULL
GROUP BY usagecount
ORDER BY usagecount;

-- Higher usage_count = more frequently accessed
-- Lots of 0/1 = cold data or working through large table
-- Lots of 4/5 = hot data
```

---

## Key Takeaways

1. **All I/O goes through buffer manager**: No direct disk access from executor.

2. **Three-layer structure**: Buffer table (lookup) → descriptors (metadata) → pool (data).

3. **Pin/unpin controls eviction**: Pinned buffers can't be evicted; unpinned are candidates.

4. **Clock-sweep approximates LRU**: Usage count determines how long buffers survive.

5. **Dirty buffers must be written before eviction**: This is when writes happen (plus checkpoints).

6. **shared_buffers ~25% of RAM**: OS filesystem cache handles the rest.

7. **Hit ratio over 95% is healthy**: Use pg_stat_database to monitor.

8. **pg_buffercache shows cache contents**: Great for diagnosing what's using memory.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/storage/buffer/bufmgr.c` | Main buffer manager code |
| `src/backend/storage/buffer/freelist.c` | Clock-sweep algorithm |
| `src/include/storage/buf_internals.h` | Buffer structures |
| `src/include/storage/bufmgr.h` | Public buffer interface |

### Documentation

- [PostgreSQL Docs: Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html)
- [PostgreSQL Docs: pg_buffercache](https://www.postgresql.org/docs/current/pgbuffercache.html)

### Next Lessons

- **Lesson 12**: Work Memory and Sorting
- **Lesson 13**: Background Writer and Checkpoints
- **Module 4**: Write-Ahead Logging

---

## References

### Key Structures

| Structure | File | Purpose |
|-----------|------|---------|
| `BufferDesc` | buf_internals.h | Buffer metadata |
| `BufferTag` | buf_internals.h | Page identifier |

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `ReadBuffer()` | bufmgr.c | Get page (load if needed) |
| `ReleaseBuffer()` | bufmgr.c | Unpin a buffer |
| `MarkBufferDirty()` | bufmgr.c | Mark as modified |
| `FlushBuffer()` | bufmgr.c | Write dirty page |
| `StrategyGetBuffer()` | freelist.c | Clock-sweep victim selection |

### Configuration Parameters

| Parameter | Purpose | Typical Value |
|-----------|---------|---------------|
| `shared_buffers` | Size of buffer pool | 25% of RAM |
| `effective_cache_size` | Planner's cache estimate | 50-75% of RAM |
| `huge_pages` | Use Linux huge pages | 'try' or 'on' |
