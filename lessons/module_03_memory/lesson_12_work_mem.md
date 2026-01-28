# Lesson 12: Work Memory — Sorting, Hashing, and Per-Operation Memory

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 3 — Memory Management |
| **Lesson** | 12 of 66 |
| **Estimated Time** | 45-60 minutes |
| **Prerequisites** | Lesson 11: The Buffer Manager |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain the difference between shared_buffers and work_mem
2. Understand when work_mem is used (sorting, hashing, etc.)
3. Identify work_mem spills in EXPLAIN output
4. Size work_mem for different workload types
5. Use maintenance_work_mem for bulk operations

### Key Terms

| Term | Definition |
|------|------------|
| **work_mem** | Memory for per-operation sorting and hashing |
| **maintenance_work_mem** | Memory for maintenance operations (VACUUM, INDEX) |
| **Sort** | Operation that orders rows in memory or on disk |
| **Hash** | Operation that builds hash tables for joins or aggregates |
| **External Sort** | Disk-based merge sort when work_mem is exceeded |
| **Batching** | Disk-based hash join when hash table doesn't fit |

---

## Introduction

While **shared_buffers** is shared memory for all connections, **work_mem** is per-operation memory that each query can use for sorting and hashing. This distinction is critical:

```
┌─────────────────────────────────────────────────────────────────┐
│  SHARED MEMORY                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  shared_buffers (e.g., 4GB)                                 │ │
│  │  - Shared by ALL connections                                │ │
│  │  - Caches data pages                                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  PER-PROCESS MEMORY                                              │
│                                                                  │
│  Connection 1:                 Connection 2:                    │
│  ┌───────────────────┐        ┌───────────────────┐            │
│  │ work_mem: 256MB   │        │ work_mem: 256MB   │            │
│  │ ┌───────────────┐ │        │ ┌───────────────┐ │            │
│  │ │ Sort 1: 256MB │ │        │ │ Hash: 256MB   │ │            │
│  │ └───────────────┘ │        │ └───────────────┘ │            │
│  │ ┌───────────────┐ │        │                   │            │
│  │ │ Hash: 256MB   │ │        │                   │            │
│  │ └───────────────┘ │        │                   │            │
│  └───────────────────┘        └───────────────────┘            │
│                                                                  │
│  DANGER: 100 connections × 2 sorts each × 256MB = 51GB!        │
└─────────────────────────────────────────────────────────────────┘
```

> **Critical Insight**: work_mem is per-operation, not per-query. A single query with 5 sorts and 3 hash operations could use up to 8× work_mem. With many concurrent connections, this adds up fast!

---

## Conceptual Foundation

### Operations That Use work_mem

| Operation | Executor Node | When Used |
|-----------|---------------|-----------|
| **Sorting** | Sort | ORDER BY, DISTINCT, merge joins |
| **Hash tables** | Hash, HashJoin, HashAggregate | Hash joins, GROUP BY, DISTINCT |
| **Bitmap scans** | BitmapIndexScan | Combining multiple index scans |
| **Window functions** | WindowAgg | When partitioning/ordering |
| **Recursive CTEs** | RecursiveUnion | Building result sets |

### In-Memory vs. Disk-Based Operations

**When work_mem is sufficient:**
```sql
EXPLAIN ANALYZE SELECT * FROM orders ORDER BY order_date;

 Sort  (actual time=50.12..75.34 rows=100000 loops=1)
   Sort Key: order_date
   Sort Method: quicksort  Memory: 25000kB
```

**When work_mem is exceeded:**
```sql
SET work_mem = '64kB';
EXPLAIN ANALYZE SELECT * FROM orders ORDER BY order_date;

 Sort  (actual time=500.12..650.34 rows=100000 loops=1)
   Sort Key: order_date
   Sort Method: external merge  Disk: 15000kB
```

The **external merge** sort is dramatically slower due to disk I/O.

### Hash Operations and Batching

**Hash join with sufficient memory:**
```sql
 Hash Join  (actual time=50.00..100.00 rows=100000)
   ->  Hash  (actual time=45.00..45.00 rows=50000)
         Buckets: 65536  Batches: 1  Memory Usage: 3500kB
```

**Hash join with insufficient memory:**
```sql
SET work_mem = '64kB';

 Hash Join  (actual time=500.00..1000.00 rows=100000)
   ->  Hash  (actual time=400.00..400.00 rows=50000)
         Buckets: 1024  Batches: 16  Memory Usage: 65kB
```

**Batches: 16** means the hash table was split into 16 pieces, requiring multiple passes over the data.

---

## Deep Dive: Implementation Details

### Sort Memory Allocation

```c
/* Simplified from src/backend/executor/nodeSort.c */
void
ExecSort(SortState *node)
{
    Tuplesortstate *state = node->tuplesortstate;
    
    if (state == NULL)
    {
        /* Initialize sort with work_mem limit */
        state = tuplesort_begin_heap(
            node->ss.ss_ScanTupleSlot->tts_tupleDescriptor,
            work_mem,          /* Memory limit in KB */
            node->bounded ? node->bound : 0
        );
    }
    
    /* Collect all tuples */
    while ((slot = ExecProcNode(outerPlan)) != NULL)
    {
        tuplesort_puttupleslot(state, slot);
        /* If memory exceeded, spill to disk */
    }
    
    /* Sort (in memory or merge external runs) */
    tuplesort_performsort(state);
}
```

### External Sort (Merge Sort)

When data exceeds work_mem:

```
Phase 1: Create sorted runs
┌─────────────────────────────────────────────────────────────────┐
│ Read work_mem worth of data → Sort in memory → Write to temp   │
│ Read work_mem worth of data → Sort in memory → Write to temp   │
│ Read work_mem worth of data → Sort in memory → Write to temp   │
│ ...                                                             │
│ Result: N sorted "runs" on disk                                 │
└─────────────────────────────────────────────────────────────────┘

Phase 2: Merge runs
┌─────────────────────────────────────────────────────────────────┐
│ Read from each run → Merge → Output sorted result               │
│                                                                  │
│ Run 1: [1, 4, 7, ...]                                           │
│ Run 2: [2, 5, 8, ...]  → Merge → [1, 2, 3, 4, 5, 6, 7, 8, ...]  │
│ Run 3: [3, 6, 9, ...]                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Hash Table Memory Management

```c
/* Simplified from src/backend/executor/nodeHash.c */
Hash *
ExecHash(HashState *node)
{
    HashJoinTable hashtable;
    
    /* Calculate optimal bucket count based on estimated rows */
    nbuckets = ExecChooseHashTableSize(ntuples, tupwidth, work_mem);
    
    /* If estimated size > work_mem, use multiple batches */
    if (estimated_size > work_mem * 1024L)
    {
        nbatch = next_power_of_2(estimated_size / (work_mem * 1024L));
    }
    
    /* Build hash table */
    foreach(slot, inner_plan)
    {
        hashvalue = ComputeHash(slot);
        
        if (nbatch > 1)
        {
            /* Determine which batch this tuple belongs to */
            int batchno = hashvalue % nbatch;
            
            if (batchno != current_batch)
            {
                /* Write to temp file for later batch */
                WriteTupleToTempFile(slot, batchno);
                continue;
            }
        }
        
        /* Insert into in-memory hash table */
        InsertHashBucket(hashtable, slot, hashvalue);
    }
}
```

---

## Practical Examples

### Example 1: Sorting Memory

```sql
-- Large work_mem: in-memory sort
SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM large_table ORDER BY created_at;

 Sort  (actual time=120.00..180.00 rows=1000000)
   Sort Key: created_at
   Sort Method: quicksort  Memory: 125000kB  ← In memory!

-- Small work_mem: disk sort
SET work_mem = '4MB';
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM large_table ORDER BY created_at;

 Sort  (actual time=500.00..800.00 rows=1000000)
   Sort Key: created_at
   Sort Method: external merge  Disk: 100000kB  ← Spilled!
```

### Example 2: Hash Join Batching

```sql
-- Observe batching
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- With low work_mem
SET work_mem = '64kB';
-- Look for "Batches: N" where N > 1
```

### Example 3: Aggregate Hashing

```sql
-- Hash aggregate uses work_mem
EXPLAIN ANALYZE
SELECT customer_id, count(*), sum(total)
FROM orders
GROUP BY customer_id;

 HashAggregate  (actual time=100.00..120.00 rows=10000)
   Group Key: customer_id
   Batches: 1  Memory Usage: 2000kB  ← Good
```

### Example 4: Session-Level Tuning

```sql
-- For a specific expensive query, temporarily increase work_mem
SET work_mem = '1GB';

SELECT ... complex query with sorts and joins ...;

-- Reset to default
RESET work_mem;
```

---

## maintenance_work_mem

For maintenance operations, a separate setting allows more memory:

```sql
SHOW maintenance_work_mem;  -- Default: 64MB

-- Operations that use maintenance_work_mem:
-- - CREATE INDEX
-- - VACUUM
-- - ALTER TABLE ADD FOREIGN KEY
-- - REINDEX
-- - CLUSTER

-- For faster index builds:
SET maintenance_work_mem = '2GB';
CREATE INDEX CONCURRENTLY big_idx ON orders(customer_id, order_date);
```

Since maintenance operations are typically single-threaded and serialize naturally, it's safer to set this higher than work_mem.

---

## Performance Tuning

### Sizing Guidelines

| Workload Type | work_mem Guidance |
|---------------|-------------------|
| OLTP (many connections) | 4MB-64MB |
| OLAP (few complex queries) | 256MB-1GB |
| Data warehousing | 1GB-4GB |
| ETL/batch processing | Set per-session |

### Total Memory Calculation

```
Worst case memory = 
    shared_buffers 
    + (max_connections × work_mem × operations_per_query)
    + maintenance_work_mem × maintenance_operations
    + other_per_connection_memory (~10MB each)
```

### Identifying Spills

```sql
-- Find queries that spilled to disk
SELECT 
    query,
    calls,
    total_exec_time,
    rows,
    (shared_blks_hit + shared_blks_read) AS buffer_access,
    temp_blks_read + temp_blks_written AS temp_blocks
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 20;
```

### Per-User or Per-Database Settings

```sql
-- Set work_mem for a specific user
ALTER USER analyst SET work_mem = '512MB';

-- Set work_mem for a database
ALTER DATABASE warehouse SET work_mem = '1GB';
```

---

## Hands-On Exercises

### Exercise 1: Sort Methods (Basic)

```sql
CREATE TABLE sort_test AS 
SELECT id, random() AS val FROM generate_series(1, 500000) id;

-- Large work_mem
SET work_mem = '256MB';
EXPLAIN (ANALYZE, COSTS OFF) SELECT * FROM sort_test ORDER BY val;
-- Note: Sort Method and Memory

-- Small work_mem
SET work_mem = '1MB';
EXPLAIN (ANALYZE, COSTS OFF) SELECT * FROM sort_test ORDER BY val;
-- Note: Sort Method and Disk
```

### Exercise 2: Hash Batching (Intermediate)

```sql
CREATE TABLE hash_large AS SELECT generate_series(1, 1000000) AS id;
CREATE TABLE hash_small AS SELECT generate_series(1, 100000) AS id;

SET work_mem = '256MB';
EXPLAIN ANALYZE 
SELECT * FROM hash_large l JOIN hash_small s ON l.id = s.id;
-- Note: Batches: 1

SET work_mem = '1MB';
EXPLAIN ANALYZE 
SELECT * FROM hash_large l JOIN hash_small s ON l.id = s.id;
-- Note: Batches: N
```

### Exercise 3: Aggregate Memory (Intermediate)

```sql
-- Create data with many groups
CREATE TABLE agg_test AS
SELECT 
    (random() * 100000)::int AS group_id,
    random() * 1000 AS value
FROM generate_series(1, 1000000);

SET work_mem = '256MB';
EXPLAIN ANALYZE SELECT group_id, sum(value) FROM agg_test GROUP BY group_id;

SET work_mem = '1MB';
EXPLAIN ANALYZE SELECT group_id, sum(value) FROM agg_test GROUP BY group_id;
-- Watch for "Batches" in HashAggregate
```

### Exercise 4: Finding Optimal work_mem (Advanced)

```sql
-- Iteratively find minimum work_mem for in-memory operations
DO $$
DECLARE
    wm int;
    plan_text text;
BEGIN
    FOR wm IN SELECT generate_series(1, 64) LOOP
        EXECUTE format('SET work_mem = ''%sMB''', wm);
        EXECUTE 'EXPLAIN (COSTS OFF) SELECT * FROM sort_test ORDER BY val' 
            INTO plan_text;
        IF plan_text LIKE '%quicksort%' THEN
            RAISE NOTICE 'Minimum work_mem for in-memory sort: %MB', wm;
            EXIT;
        END IF;
    END LOOP;
END $$;
```

---

## Key Takeaways

1. **work_mem is per-operation**: A query with 5 sorts uses up to 5× work_mem.

2. **Shared vs. local memory**: shared_buffers is shared; work_mem is per-process.

3. **Spills hurt performance**: External sorts and batched hash joins are much slower.

4. **Watch for spill indicators**: "external merge", "Disk:", "Batches: >1".

5. **Conservative defaults**: Default 4MB is safe for many connections but slow for analytics.

6. **Set per-session for big queries**: `SET work_mem = '1GB'` before expensive queries.

7. **maintenance_work_mem is separate**: Set higher for index builds and VACUUM.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/utils/sort/tuplesort.c` | Sort implementation |
| `src/backend/executor/nodeSort.c` | Sort executor node |
| `src/backend/executor/nodeHash.c` | Hash table building |
| `src/backend/executor/nodeHashjoin.c` | Hash join execution |

### Documentation

- [PostgreSQL Docs: Memory Configuration](https://www.postgresql.org/docs/current/runtime-config-resource.html#RUNTIME-CONFIG-RESOURCE-MEMORY)
- [PostgreSQL Docs: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

---

## References

### Configuration Parameters

| Parameter | Purpose | Default |
|-----------|---------|---------|
| `work_mem` | Per-operation sort/hash memory | 4MB |
| `maintenance_work_mem` | Maintenance operation memory | 64MB |
| `hash_mem_multiplier` | Multiplier for hash operations | 2.0 |
| `temp_file_limit` | Max temp file usage per process | -1 (unlimited) |

### EXPLAIN Indicators

| Indicator | Meaning |
|-----------|---------|
| `Sort Method: quicksort Memory: NkB` | In-memory sort |
| `Sort Method: top-N heapsort Memory: NkB` | In-memory, for LIMIT |
| `Sort Method: external merge Disk: NkB` | Spilled to disk |
| `Batches: 1` | Hash table fits in memory |
| `Batches: N` (N > 1) | Hash table spilled |
