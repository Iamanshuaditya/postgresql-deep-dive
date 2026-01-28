# Lesson 26: HOT Updates — Heap-Only Tuples for Better Performance

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 2 — Storage Architecture |
| **Lesson** | 26 of 66 |
| **Estimated Time** | 45-60 minutes |
| **Prerequisites** | Lesson 8: Heap Storage and Page Layout, Lesson 9: B-tree Index Structure |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain why HOT updates are important for performance
2. Understand when HOT updates can and cannot occur
3. Design tables and workloads to maximize HOT updates
4. Monitor HOT update statistics
5. Troubleshoot when HOT updates aren't happening

### Key Terms

| Term | Definition |
|------|------------|
| **HOT** | Heap-Only Tuple—update without index changes |
| **HOT chain** | Linked list of tuple versions on same page |
| **Pruning** | Removing dead tuples from HOT chain |
| **Fillfactor** | Reserved space in pages for HOT updates |
| **Index-only columns** | Columns that appear in indexes |

---

## Introduction

Updates in PostgreSQL create new tuple versions. Normally, ALL indexes must be updated:

```
┌─────────────────────────────────────────────────────────────────┐
│  Normal Update vs HOT Update                                     │
│                                                                  │
│  NORMAL UPDATE: (Every index entry must change!)                │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ UPDATE users SET email = 'new@example.com' WHERE id = 1;    ││
│  │                                                              ││
│  │ 1. Create new tuple in heap                                 ││
│  │ 2. Update PRIMARY KEY index (id)          ← Even though     ││
│  │ 3. Update email index                        id didn't      ││
│  │ 4. Update name index                         change!        ││
│  │ 5. Update created_at index                                  ││
│  │                                                              ││
│  │ 5 index operations for 1 column change!                     ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  HOT UPDATE: (No index changes needed!)                         │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ UPDATE users SET email = 'new@example.com' WHERE id = 1;    ││
│  │                                                              ││
│  │ 1. Create new tuple in SAME page as old                     ││
│  │ 2. Link old → new with t_ctid pointer                       ││
│  │ 3. NO index updates!                                         ││
│  │                                                              ││
│  │ 1 heap operation, 0 index operations!                       ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## HOT Requirements

### When Can HOT Happen?

HOT updates require ALL of these conditions:

```
┌─────────────────────────────────────────────────────────────────┐
│  HOT Update Conditions                                           │
│                                                                  │
│  ✓ No indexed column is modified                                │
│    (changing non-indexed columns only)                          │
│                                                                  │
│  ✓ New tuple fits on the same page                              │
│    (page has enough free space)                                 │
│                                                                  │
│  ✓ Tuple size doesn't grow beyond page capacity                │
│                                                                  │
│  If ANY condition fails → Normal update (all indexes updated)  │
└─────────────────────────────────────────────────────────────────┘
```

### Example: HOT vs Non-HOT

```sql
-- Table with indexes
CREATE TABLE users (
    id serial PRIMARY KEY,
    email text UNIQUE,
    name text,
    bio text,
    updated_at timestamp
);
CREATE INDEX users_name_idx ON users(name);

-- This CAN be HOT (bio and updated_at not indexed):
UPDATE users SET bio = 'New bio', updated_at = now() WHERE id = 1;

-- This CANNOT be HOT (name is indexed):
UPDATE users SET name = 'New Name' WHERE id = 1;

-- This CANNOT be HOT (email is indexed):
UPDATE users SET email = 'new@example.com' WHERE id = 1;
```

---

## How HOT Chains Work

### HOT Chain Structure

```
┌─────────────────────────────────────────────────────────────────┐
│  HOT Chain in a Heap Page                                        │
│                                                                  │
│  Page 42:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Line Pointer Table (at page start):                         ││
│  │ LP[1] → Tuple v1 (DEAD)                                     ││
│  │ LP[2] → Tuple v2 (DEAD)                                     ││
│  │ LP[3] → Tuple v3 (LIVE)                                     ││
│  │                                                              ││
│  │ Tuples (at page end):                                       ││
│  │ ┌──────────────────────────────────────────────────────┐    ││
│  │ │ v1: xmin=100, xmax=101, t_ctid=(42,2), HEAP_HOT_UPDATED ││
│  │ │ v2: xmin=101, xmax=102, t_ctid=(42,3), HEAP_HOT_UPDATED ││
│  │ │ v3: xmin=102, xmax=0,   t_ctid=(42,3), HEAP_ONLY_TUPLE  ││
│  │ └──────────────────────────────────────────────────────┘    ││
│  │                                                              ││
│  │ v1 → v2 → v3 (HOT chain)                                   ││
│  │ Indexes still point to v1!                                  ││
│  │ Index scan follows chain to find v3                        ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Following a HOT Chain

```
Index lookup for id=1:
1. Index points to (42, 1) — the original tuple
2. Scan page 42, line 1
3. v1 is dead (xmax set), has HEAP_HOT_UPDATED flag
4. Follow t_ctid to (42, 2)
5. v2 is dead, follow to (42, 3)
6. v3 is live, return it
```

---

## Pruning: Cleaning HOT Chains

### What is Pruning?

Dead tuples in HOT chains are cleaned during normal operations:

```sql
-- Pruning happens during:
-- 1. SELECT/UPDATE/DELETE that accesses the page
-- 2. VACUUM
-- 3. Index scans following HOT chains
```

### Pruning Example

```
Before Pruning:          After Pruning:
LP[1] → v1 (DEAD)        LP[1] → v3 (redirected!)
LP[2] → v2 (DEAD)        LP[2] → UNUSED
LP[3] → v3 (LIVE)        LP[3] → v3 (LIVE)

v1 and v2 space reclaimed for reuse!
```

### Key Difference: Pruning vs VACUUM

| Operation | What It Does | When |
|-----------|--------------|------|
| Pruning | Cleans dead tuples within HOT chains | On page access |
| VACUUM | Cleans all dead tuples, updates FSM/VM | Background process |

Pruning is **opportunistic** and **page-local**.

---

## Fillfactor: Reserving Space for HOT

### Why Fillfactor Matters

If pages are 100% full, new tuple versions can't fit → HOT fails!

```sql
-- Default fillfactor is 100 (completely fill pages)
CREATE TABLE normal_table (id int, data text);

-- Set fillfactor to leave room for updates
CREATE TABLE hot_optimized (id int, data text) WITH (fillfactor = 70);

-- 70% means 30% of each page reserved for updates
-- Updates can use that space → More HOT updates!
```

### Choosing Fillfactor

| Workload | Recommended Fillfactor |
|----------|------------------------|
| Read-mostly | 100 (default) |
| Frequent updates, small rows | 70-80 |
| Frequent updates, large rows | 50-60 |
| Append-only (inserts) | 100 |

### Example: Fillfactor Impact

```sql
-- Without fillfactor optimization
CREATE TABLE test1 (id int PRIMARY KEY, status text, data text);
CREATE INDEX ON test1(id);  -- Only index id

-- With fillfactor optimization  
CREATE TABLE test2 (id int PRIMARY KEY, status text, data text) 
WITH (fillfactor = 70);
CREATE INDEX ON test2(id);

-- Insert same data
INSERT INTO test1 SELECT i, 'pending', repeat('x', 100) FROM generate_series(1,10000) i;
INSERT INTO test2 SELECT i, 'pending', repeat('x', 100) FROM generate_series(1,10000) i;

-- Update non-indexed column
UPDATE test1 SET status = 'done';  -- May not be all HOT
UPDATE test2 SET status = 'done';  -- More likely to be HOT

-- Compare HOT statistics
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(n_tup_hot_upd::numeric / NULLIF(n_tup_upd, 0) * 100) AS hot_pct
FROM pg_stat_user_tables WHERE relname LIKE 'test%';
```

---

## Monitoring HOT Updates

### HOT Statistics

```sql
-- Table-level HOT statistics
SELECT 
    relname,
    n_tup_upd AS updates,
    n_tup_hot_upd AS hot_updates,
    CASE WHEN n_tup_upd > 0 
         THEN round(100.0 * n_tup_hot_upd / n_tup_upd, 1)
         ELSE 0 
    END AS hot_percent
FROM pg_stat_user_tables
ORDER BY n_tup_upd DESC;

-- Goal: HOT percentage should be high (>90% for update-heavy tables)
```

### Diagnosing Non-HOT Updates

```sql
-- Check which columns are indexed
SELECT 
    i.relname AS index_name,
    array_agg(a.attname) AS indexed_columns
FROM pg_index ix
JOIN pg_class i ON i.oid = ix.indexrelid
JOIN pg_attribute a ON a.attrelid = ix.indrelid 
    AND a.attnum = ANY(ix.indkey)
WHERE ix.indrelid = 'users'::regclass
GROUP BY i.relname;

-- If you're updating these columns, HOT won't work
```

### Page Inspection

```sql
-- Using pageinspect extension
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- Check for HOT chains
SELECT 
    lp, 
    t_ctid,
    t_infomask::bit(16) AS flags,
    CASE WHEN (t_infomask & 16384) != 0 THEN 'HOT_UPDATED' ELSE '' END AS hot_updated,
    CASE WHEN (t_infomask & 32768) != 0 THEN 'HEAP_ONLY' ELSE '' END AS heap_only
FROM heap_page_items(get_raw_page('users', 0));
```

---

## Design Patterns for HOT

### Pattern 1: Separate Indexed and Updated Columns

```sql
-- Anti-pattern: Index on frequently updated column
CREATE TABLE orders (
    id serial PRIMARY KEY,
    status text,          -- Updated frequently
    total numeric
);
CREATE INDEX ON orders(status);  -- Blocks HOT updates!

-- Better: Don't index update-heavy columns
CREATE TABLE orders_v2 (
    id serial PRIMARY KEY,
    status text,          -- Updated frequently, NOT indexed
    total numeric
);
-- If you need to query by status, consider:
-- 1. Partial indexes
-- 2. Separate status lookup table
-- 3. Accept the trade-off
```

### Pattern 2: Use Partial Indexes

```sql
-- Instead of indexing all statuses:
CREATE INDEX ON orders(status);

-- Index only values you query:
CREATE INDEX ON orders(id) WHERE status = 'pending';
CREATE INDEX ON orders(id) WHERE status = 'processing';

-- Changing to 'completed' doesn't touch these indexes!
```

### Pattern 3: BRIN for Range Queries

```sql
-- BRIN indexes are tiny and don't block HOT
CREATE INDEX ON orders USING brin(created_at);

-- Updates to other columns are still HOT-eligible
UPDATE orders SET status = 'done' WHERE id = 1;  -- HOT!
```

---

## Hands-On Exercises

### Exercise 1: Verify HOT Updates (Basic)

```sql
-- Reset stats
SELECT pg_stat_reset();

-- Create test table
CREATE TABLE hot_test (
    id serial PRIMARY KEY,
    indexed_col text,
    normal_col text
);
CREATE INDEX ON hot_test(indexed_col);

-- Insert data
INSERT INTO hot_test (indexed_col, normal_col)
SELECT 'value_' || i, 'data_' || i FROM generate_series(1, 1000) i;

-- Update non-indexed column (should be HOT)
UPDATE hot_test SET normal_col = 'updated' WHERE id <= 500;

-- Update indexed column (NOT HOT)
UPDATE hot_test SET indexed_col = 'new_value' WHERE id > 500;

-- Check HOT stats
SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables 
WHERE relname = 'hot_test';
-- First 500 updates should be HOT, last 500 should not
```

### Exercise 2: Fillfactor Impact (Intermediate)

```sql
-- Create tables with different fillfactors
CREATE TABLE ff100 (id int, data text) WITH (fillfactor = 100);
CREATE TABLE ff70 (id int, data text) WITH (fillfactor = 70);

-- Populate (fill pages)
INSERT INTO ff100 SELECT i, repeat('x', 200) FROM generate_series(1, 5000) i;
INSERT INTO ff70 SELECT i, repeat('x', 200) FROM generate_series(1, 5000) i;

-- Reset stats
SELECT pg_stat_reset();

-- Update rows (extend data slightly)
UPDATE ff100 SET data = repeat('y', 220);
UPDATE ff70 SET data = repeat('y', 220);

-- Compare HOT rates
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0 * n_tup_hot_upd / n_tup_upd) AS hot_pct
FROM pg_stat_user_tables 
WHERE relname IN ('ff100', 'ff70');
```

### Exercise 3: HOT Chain Inspection (Advanced)

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- Create and populate small table
CREATE TABLE chain_test (id int PRIMARY KEY, val text);
INSERT INTO chain_test VALUES (1, 'original');

-- Perform updates to create HOT chain
UPDATE chain_test SET val = 'version2' WHERE id = 1;
UPDATE chain_test SET val = 'version3' WHERE id = 1;
UPDATE chain_test SET val = 'version4' WHERE id = 1;

-- Examine page
SELECT lp, t_xmin, t_xmax, t_ctid, 
       CASE WHEN (t_infomask & 16384) != 0 THEN 'UPDATED' END AS hot_updated,
       CASE WHEN (t_infomask & 32768) != 0 THEN 'ONLY' END AS heap_only
FROM heap_page_items(get_raw_page('chain_test', 0))
WHERE t_xmin IS NOT NULL;
```

---

## Key Takeaways

1. **HOT avoids index updates**: Dramatic performance improvement.

2. **Requires same page + no indexed column change**: Both conditions must be met.

3. **Fillfactor reserves space**: Lower fillfactor = more room for HOT.

4. **Monitor n_tup_hot_upd**: High ratio indicates good HOT usage.

5. **Design for HOT**: Avoid indexing frequently-updated columns.

6. **Pruning cleans chains**: Dead tuples removed opportunistically.

7. **HOT chains persist until pruned**: Index scans follow the chain.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/access/heap/heapam.c` | Heap access methods including HOT |
| `src/backend/access/heap/pruneheap.c` | HOT chain pruning |
| `src/include/access/htup_details.h` | Tuple flags (HOT_UPDATED, etc.) |

### Documentation

- [PostgreSQL Docs: Heap-Only Tuples](https://www.postgresql.org/docs/current/storage-hot.html)
- [PostgreSQL Docs: Storage Parameters](https://www.postgresql.org/docs/current/sql-createtable.html#SQL-CREATETABLE-STORAGE-PARAMETERS)

---

## References

### HOT-Related Statistics

| Statistic | Location | Description |
|-----------|----------|-------------|
| n_tup_upd | pg_stat_user_tables | Total updates |
| n_tup_hot_upd | pg_stat_user_tables | HOT updates |

### Tuple Flags

| Flag | Meaning |
|------|---------|
| HEAP_HOT_UPDATED | This tuple was HOT-updated (has successor) |
| HEAP_ONLY_TUPLE | This tuple is part of a HOT chain (not in index) |

### Fillfactor Guidelines

| Use Case | Fillfactor |
|----------|------------|
| Append-only | 100 |
| Light updates | 90 |
| Moderate updates | 80 |
| Heavy updates | 70 |
| Very heavy updates | 50-60 |
