# Lesson 20: Parallel Query Execution — Harnessing Multiple CPUs

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 11 — Query Execution Optimization |
| **Lesson** | 20 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 6: The Executor |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain how PostgreSQL parallelizes query execution
2. Identify which operations support parallelism
3. Configure parallel query parameters appropriately
4. Interpret parallel query plans in EXPLAIN
5. Troubleshoot queries that don't parallelize

### Key Terms

| Term | Definition |
|------|------------|
| **Parallel Worker** | Background process executing part of a query |
| **Gather** | Node that collects results from parallel workers |
| **Partial Aggregate** | Per-worker aggregation step |
| **Parallel Safe** | Function/operation safe to run in parallel |
| **Leader** | Main backend process coordinating parallel query |

---

## Introduction

PostgreSQL can split work across multiple CPU cores for expensive queries:

```
┌─────────────────────────────────────────────────────────────────┐
│  Sequential vs Parallel Execution                                │
│                                                                  │
│  SEQUENTIAL (Traditional):                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Single Backend Process                                     │  │
│  │ ┌─────────────────────────────────────────────────────┐   │  │
│  │ │ Scan entire 100GB table                              │   │  │
│  │ │ Process all rows                                     │   │  │
│  │ │ Time: 60 seconds                                     │   │  │
│  │ └─────────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  PARALLEL (With 4 Workers):                                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Leader + 4 Parallel Workers                                │  │
│  │ ┌──────────────┐ ┌──────────────┐                         │  │
│  │ │ Worker 1:    │ │ Worker 2:    │                         │  │
│  │ │ Scan 0-25%   │ │ Scan 25-50%  │                         │  │
│  │ └──────────────┘ └──────────────┘                         │  │
│  │ ┌──────────────┐ ┌──────────────┐                         │  │
│  │ │ Worker 3:    │ │ Worker 4:    │                         │  │
│  │ │ Scan 50-75%  │ │ Scan 75-100% │                         │  │
│  │ └──────────────┘ └──────────────┘                         │  │
│  │         ↓              ↓              ↓              ↓     │  │
│  │ ┌─────────────────────────────────────────────────────┐   │  │
│  │ │ Gather (Leader combines results)                     │   │  │
│  │ └─────────────────────────────────────────────────────┘   │  │
│  │ Time: ~15 seconds (4x faster!)                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### What Can Be Parallelized?

| Operation | Parallel Support |
|-----------|-----------------|
| Sequential Scan | ✓ Parallel Seq Scan |
| Index Scan | ✓ Parallel Index Scan (PG 10+) |
| Bitmap Heap Scan | ✓ Parallel Bitmap Heap Scan |
| Aggregates | ✓ Partial + Finalize (most) |
| Hash Join | ✓ Parallel Hash Join |
| Nested Loop | ✓ (outer side only) |
| Merge Join | ✗ Not parallelized |
| Append (UNION) | ✓ Parallel Append |
| Sort | ✗ Not parallelized (gather before sort) |

---

## Conceptual Foundation

### Parallel Query Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Parallel Query Architecture                                     │
│                                                                  │
│  Client                                                          │
│    │                                                             │
│    ▼                                                             │
│  Leader Backend                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 1. Parse, analyze, plan query                               │ │
│  │ 2. Decide: Should this be parallel?                         │ │
│  │ 3. Launch parallel workers                                  │ │
│  │ 4. Execute above Gather node                                │ │
│  │ 5. Collect results from workers                             │ │
│  │ 6. Return to client                                         │ │
│  └─────────────────────────────────────────────────────────────┘ │
│    │           │           │                                     │
│    │           │           │ Dynamic Shared Memory               │
│    ▼           ▼           ▼                                     │
│  ┌───────┐ ┌───────┐ ┌───────┐                                  │
│  │Worker1│ │Worker2│ │Worker3│                                  │
│  ├───────┤ ├───────┤ ├───────┤                                  │
│  │Scan   │ │Scan   │ │Scan   │                                  │
│  │pages  │ │pages  │ │pages  │                                  │
│  │0-33%  │ │33-66% │ │66-100%│                                  │
│  └───────┘ └───────┘ └───────┘                                  │
│      │          │          │                                     │
│      └──────────┴──────────┘                                     │
│                 │                                                │
│                 ▼                                                │
│            Tuple Queue (shared memory)                           │
│                 │                                                │
│                 ▼                                                │
│           Gather Node (in Leader)                                │
└─────────────────────────────────────────────────────────────────┘
```

### When Does Parallel Execute?

The planner considers parallel execution when:

1. **Table is large enough**: `min_parallel_table_scan_size` (8MB default)
2. **Not in a transaction with writes**: Can't parallelize if modifying data
3. **Query is parallel safe**: No unsafe functions or operations
4. **Workers available**: `max_parallel_workers_per_gather` > 0
5. **Cost is worthwhile**: Parallel plan cheaper than sequential

### The Gather Node

```sql
EXPLAIN SELECT count(*) FROM large_table;
```

```
 Finalize Aggregate (...)
   ->  Gather (...)
         Workers Planned: 4
         ->  Partial Aggregate (...)
               ->  Parallel Seq Scan on large_table (...)
```

- **Gather**: Collects rows from workers
- **Gather Merge**: Collects and maintains sort order
- **Workers Planned**: How many workers will be launched
- **Workers Launched** (ANALYZE): How many actually ran

---

## Configuration Parameters

### Key Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_parallel_workers` | 8 | Total parallel workers for all queries |
| `max_parallel_workers_per_gather` | 2 | Workers per query (0 disables) |
| `min_parallel_table_scan_size` | 8MB | Min table size for parallel scan |
| `min_parallel_index_scan_size` | 512KB | Min index size for parallel scan |
| `parallel_tuple_cost` | 0.1 | Cost per tuple transferred to leader |
| `parallel_setup_cost` | 1000 | Cost to launch a worker |
| `force_parallel_mode` | off | Force parallel (for testing) |

### Tuning for Your Hardware

```sql
-- For a 16-core machine with mostly analytical queries
ALTER SYSTEM SET max_parallel_workers = 16;
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET min_parallel_table_scan_size = '1MB';  -- More aggressive

-- Reload settings
SELECT pg_reload_conf();
```

### Per-Table Settings

```sql
-- Specific table: Use more workers
ALTER TABLE huge_analytics_table SET (parallel_workers = 8);

-- Disable parallelism for a table
ALTER TABLE small_table SET (parallel_workers = 0);
```

---

## Parallel Scans

### Parallel Sequential Scan

```sql
EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM orders WHERE total > 1000;
```

```
 Gather  (cost=1000.00..50000.00 rows=10000 width=100)
         (actual time=0.500..100.000 rows=10000 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on public.orders  
                     (cost=0.00..40000.00 rows=4167 width=100)
                     (actual time=0.100..90.000 rows=3333 loops=3)
         Filter: (total > 1000)
         Rows Removed by Filter: 30000
```

**Understanding loops=3:**
- Leader + 2 workers = 3 processes
- Each scans ~1/3 of the table
- rows=3333 is per loop (total 3333 × 3 = ~10000)

### Parallel Index Scan

```sql
CREATE INDEX orders_total_idx ON orders(total);
EXPLAIN ANALYZE SELECT * FROM orders WHERE total BETWEEN 1000 AND 2000;
```

```
 Gather  (...)
   Workers Planned: 2
   ->  Parallel Index Scan using orders_total_idx on orders
         Index Cond: ((total >= 1000) AND (total <= 2000))
```

Workers divide the index pages among themselves.

### Parallel Bitmap Heap Scan

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id IN (1, 2, 3, 4, 5);
```

```
 Gather  (...)
   ->  Parallel Bitmap Heap Scan on orders
         Recheck Cond: (customer_id = ANY (...))
         ->  Bitmap Index Scan on orders_customer_idx
               Index Cond: (customer_id = ANY (...))
```

Bitmap is built cooperatively, then heap is scanned in parallel.

---

## Parallel Aggregation

### How Partial Aggregates Work

```
┌─────────────────────────────────────────────────────────────────┐
│  Parallel Aggregation: SELECT count(*), sum(total) FROM orders │
│                                                                  │
│  Worker 1:              Worker 2:              Worker 3:        │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐│
│  │Partial Aggregate│   │Partial Aggregate│   │Partial Aggregate││
│  │count=3333       │   │count=3333       │   │count=3334       ││
│  │sum=500000       │   │sum=480000       │   │sum=520000       ││
│  └────────┬────────┘   └────────┬────────┘   └────────┬────────┘│
│           │                     │                     │         │
│           └─────────────────────┼─────────────────────┘         │
│                                 ▼                               │
│                         ┌──────────────┐                        │
│                         │    Gather    │                        │
│                         └──────┬───────┘                        │
│                                │                                 │
│                                ▼                                 │
│                   ┌────────────────────────┐                    │
│                   │   Finalize Aggregate   │                    │
│                   │   count=10000          │                    │
│                   │   sum=1500000          │                    │
│                   └────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

### EXPLAIN Example

```sql
EXPLAIN ANALYZE
SELECT customer_id, count(*), sum(total) 
FROM orders 
GROUP BY customer_id;
```

```
 Finalize HashAggregate  (cost=... rows=1000 ...)
   Group Key: customer_id
   Batches: 1  Memory Usage: 200kB
   ->  Gather  (cost=... rows=3000 ...)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial HashAggregate  (cost=... rows=1000 ...)
               Group Key: customer_id
               Batches: 1  Memory Usage: 200kB
               ->  Parallel Seq Scan on orders  (...)
```

- **Partial HashAggregate**: Each worker aggregates its portion
- **Finalize HashAggregate**: Leader combines partial results

---

## Parallel Joins

### Parallel Hash Join

```sql
EXPLAIN ANALYZE
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

```
 Gather  (...)
   Workers Planned: 2
   ->  Parallel Hash Join  (...)
         Hash Cond: (o.customer_id = c.id)
         ->  Parallel Seq Scan on orders o  (...)
         ->  Parallel Hash  (...)
               Buckets: 16384  Batches: 1
               ->  Parallel Seq Scan on customers c  (...)
```

**Parallel Hash behavior:**
- All workers contribute to building a shared hash table
- Uses shared memory for the hash table
- Very efficient for large joins

### Parallel Nested Loop

```sql
EXPLAIN ANALYZE
SELECT * FROM small_table s
JOIN large_table l ON s.id = l.small_id;
```

```
 Gather  (...)
   ->  Nested Loop  (...)
         ->  Parallel Seq Scan on large_table l  (...)  -- Outer
         ->  Index Scan using small_pkey on small_table s (...)  -- Inner
```

Only the outer side is parallelized in nested loop joins.

---

## Parallel Safety

### What Makes Something Parallel Unsafe?

```sql
-- This function is PARALLEL UNSAFE by default
CREATE FUNCTION my_function() RETURNS int AS $$
BEGIN
    -- Modifies database state
    INSERT INTO log_table VALUES (now(), 'called');
    RETURN 1;
END;
$$ LANGUAGE plpgsql;

-- Query using it won't parallelize
EXPLAIN SELECT my_function() FROM large_table;  -- No parallel!
```

### Marking Functions Parallel Safe

```sql
-- Mark a function as parallel safe
CREATE OR REPLACE FUNCTION pure_calculation(x int) RETURNS int AS $$
BEGIN
    RETURN x * x;
END;
$$ LANGUAGE plpgsql PARALLEL SAFE;

-- Now parallelism is possible
EXPLAIN SELECT pure_calculation(id) FROM large_table;
```

### Parallel Safety Levels

| Level | Meaning |
|-------|---------|
| PARALLEL UNSAFE | Never parallel (writes, volatile, etc.) |
| PARALLEL RESTRICTED | Can run in leader only |
| PARALLEL SAFE | Can run in any worker |

Built-in functions are mostly PARALLEL SAFE. User functions default to UNSAFE.

---

## Troubleshooting

### Query Doesn't Parallelize

```sql
-- Check settings
SHOW max_parallel_workers_per_gather;  -- If 0, parallelism disabled
SHOW parallel_tuple_cost;
SHOW parallel_setup_cost;

-- Check table size (might be too small)
SELECT pg_size_pretty(pg_relation_size('my_table'));

-- Force parallel for testing
SET force_parallel_mode = on;
SET parallel_tuple_cost = 0;
SET parallel_setup_cost = 0;
EXPLAIN SELECT * FROM my_table;
```

### Common Reasons for No Parallelism

| Reason | Solution |
|--------|----------|
| Table too small | Lower `min_parallel_table_scan_size` |
| max_parallel_workers_per_gather = 0 | Increase setting |
| In a transaction with writes | Queries can't parallelize |
| Using CURSOR | Cursors not parallelized |
| Using parallel-unsafe function | Mark function PARALLEL SAFE |
| Planner estimates serial is faster | Check costs, statistics |

### Workers Launched < Planned

```sql
EXPLAIN (ANALYZE) SELECT ...;
-- Workers Planned: 4
-- Workers Launched: 2  -- Why only 2?
```

Possible reasons:
- `max_parallel_workers` global limit reached
- Other queries using workers
- System under memory pressure

---

## Hands-On Exercises

### Exercise 1: Enabling Parallelism (Basic)

```sql
-- Create large table
CREATE TABLE parallel_test AS 
SELECT generate_series(1, 5000000) AS id,
       random() * 1000 AS value,
       md5(random()::text) AS data;

-- Check plan without parallelism
SET max_parallel_workers_per_gather = 0;
EXPLAIN ANALYZE SELECT count(*) FROM parallel_test;

-- Enable parallelism
SET max_parallel_workers_per_gather = 4;
EXPLAIN ANALYZE SELECT count(*) FROM parallel_test;

-- Compare times!
```

### Exercise 2: Parallel Aggregation (Intermediate)

```sql
-- Group by with parallel workers
SET max_parallel_workers_per_gather = 4;

EXPLAIN ANALYZE
SELECT (id % 100) AS bucket, count(*), avg(value)
FROM parallel_test
GROUP BY (id % 100);

-- Notice: Partial HashAggregate + Finalize HashAggregate
```

### Exercise 3: Parallel Hash Join (Intermediate)

```sql
-- Create second table
CREATE TABLE parallel_join AS
SELECT generate_series(1, 100000) AS id,
       'name_' || generate_series(1, 100000) AS name;

-- Parallel hash join
SET max_parallel_workers_per_gather = 4;
EXPLAIN ANALYZE
SELECT p.*, j.name
FROM parallel_test p
JOIN parallel_join j ON (p.id % 100000) = j.id;

-- Look for "Parallel Hash Join" and "Parallel Hash"
```

### Exercise 4: Parallel Safety (Advanced)

```sql
-- Create parallel-unsafe function
CREATE FUNCTION unsafe_func(x int) RETURNS int AS $$
BEGIN
    RAISE NOTICE 'Called with %', x;  -- Side effect
    RETURN x;
END;
$$ LANGUAGE plpgsql;

-- This won't parallelize
SET max_parallel_workers_per_gather = 4;
EXPLAIN ANALYZE SELECT unsafe_func(id) FROM parallel_test LIMIT 1000;

-- Mark safe (only if you're sure!)
DROP FUNCTION unsafe_func;
CREATE FUNCTION safe_func(x int) RETURNS int AS $$
BEGIN
    RETURN x * 2;
END;
$$ LANGUAGE plpgsql PARALLEL SAFE;

EXPLAIN ANALYZE SELECT safe_func(id) FROM parallel_test LIMIT 1000;
```

---

## Key Takeaways

1. **Parallel queries use multiple workers**: Split work across CPUs.

2. **Gather collects results**: Tuples flow from workers to leader.

3. **Not everything parallelizes**: Sort, merge join, cursors don't.

4. **Aggregates use partial/finalize**: Workers do partial, leader combines.

5. **Parallel Hash is efficient**: Workers build shared hash table.

6. **Functions must be marked safe**: User functions default to UNSAFE.

7. **Check Workers Launched**: May be less than planned.

8. **Cost must justify parallelism**: Small tables won't parallelize.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/executor/nodeGather.c` | Gather node |
| `src/backend/executor/nodeGatherMerge.c` | Gather Merge |
| `src/backend/access/heap/heapam.c` | Parallel heap scan |
| `src/backend/executor/nodeHash.c` | Parallel hash |
| `src/backend/optimizer/path/allpaths.c` | Parallel path creation |

### Documentation

- [PostgreSQL Docs: Parallel Query](https://www.postgresql.org/docs/current/parallel-query.html)
- [PostgreSQL Docs: Parallel Safety](https://www.postgresql.org/docs/current/parallel-safety.html)

---

## References

### Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `max_worker_processes` | 8 | Total background workers |
| `max_parallel_workers` | 8 | Parallel query workers |
| `max_parallel_workers_per_gather` | 2 | Per-query limit |
| `min_parallel_table_scan_size` | 8MB | Table size threshold |
| `min_parallel_index_scan_size` | 512KB | Index size threshold |
| `parallel_leader_participation` | on | Leader also does work |

### EXPLAIN Parallel Indicators

| Node | Meaning |
|------|---------|
| Gather | Collect from workers (any order) |
| Gather Merge | Collect preserving sort |
| Parallel Seq Scan | Parallel table scan |
| Parallel Index Scan | Parallel index scan |
| Parallel Hash | Shared hash table build |
| Partial Aggregate | Per-worker aggregation |
| Finalize Aggregate | Combine partial results |
