# Lesson 32: The Cost Model — How PostgreSQL Estimates Query Costs

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 11 — Query Optimization |
| **Lesson** | 32 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 5: Introduction to the Query Planner, Lesson 24: Statistics |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Understand how PostgreSQL calculates query costs
2. Interpret cost numbers in EXPLAIN output
3. Explain the cost constants and their meanings
4. Diagnose when cost estimates are wrong
5. Tune cost parameters for specific hardware

### Key Terms

| Term | Definition |
|------|------------|
| **Startup Cost** | Cost before first row can be returned |
| **Total Cost** | Cost to return all rows |
| **Cost Unit** | Abstract unit based on sequential page read |
| **Selectivity** | Fraction of rows matching a predicate |
| **Cost Constant** | Configuration value for operation costs |

---

## Introduction

### What is Cost?

```
┌─────────────────────────────────────────────────────────────────┐
│  PostgreSQL Cost Model                                           │
│                                                                  │
│  EXPLAIN SELECT * FROM users WHERE email = 'test@example.com'; │
│                                                                  │
│  Index Scan on users_email_idx  (cost=0.42..8.44 rows=1)       │
│                                       ↑       ↑     ↑            │
│                              startup   total   rows              │
│                                                                  │
│  What do these numbers mean?                                    │
│                                                                  │
│  Cost is:                                                        │
│  • An ESTIMATE (not actual time)                                │
│  • Relative (lower is better)                                   │
│  • In arbitrary units (based on seq_page_cost = 1.0)           │
│                                                                  │
│  Planner chooses plan with LOWEST total cost                    │
└─────────────────────────────────────────────────────────────────┘
```

### Startup vs Total Cost

```sql
EXPLAIN SELECT * FROM orders ORDER BY created_at LIMIT 10;

-- Sort (cost=0.15..100.00 rows=10 ...)
--            ↑        ↑
--       startup    total

-- Startup cost: 0.15
--   Cost before ANY rows returned
--   For Sort: just index lookup setup

-- Total cost: 100.00
--   Cost to return ALL rows
--   For Sort with LIMIT: still low (stops early)
```

### When Startup Cost Matters

```sql
-- EXISTS uses startup cost (stops at first row)
SELECT EXISTS(SELECT 1 FROM large_table WHERE condition);

-- Nested Loop inner: executed many times, startup matters
EXPLAIN SELECT * FROM a JOIN b ON a.id = b.a_id;
```

---

## The Cost Formula

### Basic Components

```
Total Cost = 
    (pages_read × page_cost) +
    (tuples_scanned × cpu_tuple_cost) +
    (operators_applied × cpu_operator_cost)
```

### Cost Constants

```sql
-- View current cost settings
SHOW seq_page_cost;         -- 1.0 (baseline)
SHOW random_page_cost;      -- 4.0 (random I/O more expensive)
SHOW cpu_tuple_cost;        -- 0.01 (per-row processing)
SHOW cpu_index_tuple_cost;  -- 0.005 (index entry processing)
SHOW cpu_operator_cost;     -- 0.0025 (per-operator evaluation)
SHOW parallel_setup_cost;   -- 1000 (starting parallel workers)
SHOW parallel_tuple_cost;   -- 0.1 (passing tuple to leader)
```

### Example: Sequential Scan Cost

```sql
-- Table: 1000 pages, 100,000 rows
-- Predicate: WHERE age > 25 (selectivity 0.5)

-- Cost calculation:
seq_scan_cost = 
    (1000 pages × 1.0 seq_page_cost) +           -- 1000
    (100000 tuples × 0.01 cpu_tuple_cost) +       -- 1000
    (100000 tuples × 0.0025 cpu_operator_cost)    -- 250
    = 2250

-- With 50% selectivity, output rows = 50000
-- But scan still reads all pages!
```

### Example: Index Scan Cost

```sql
-- Same table, using btree index on age
-- Selectivity 0.5 = 50000 matching rows

-- Cost calculation:
index_scan_cost =
    (tree_height pages × random_page_cost) +       -- Index traverse
    (50000 index_entries × cpu_index_tuple_cost) + -- Read index
    (50000 heap_pages × random_page_cost) +        -- Fetch rows (random!)
    (50000 tuples × cpu_tuple_cost)                -- Process rows

-- For 50% selectivity, seq scan usually wins!
-- Index scan only wins for low selectivity (<10-20%)
```

---

## Cost Constants Explained

### seq_page_cost (Default: 1.0)

```
Baseline for all costs.
1 sequential page read = 1.0 cost units.

All other costs are relative to this.
```

### random_page_cost (Default: 4.0)

```
Random I/O is slower than sequential.
Default 4.0 means: 1 random read ≈ 4 sequential reads.

For SSDs, lower this:
  SET random_page_cost = 1.1;  -- SSD
  SET random_page_cost = 4.0;  -- HDD

Why it matters:
  Index scans do random I/O
  Lower value = more likely to use indexes
```

### cpu_tuple_cost (Default: 0.01)

```
Cost to process one row.
Includes: checking visibility, forming result tuple.

1000 rows × 0.01 = 10 cost units
```

### cpu_operator_cost (Default: 0.0025)

```
Cost to apply one operator per row.
Examples: comparisons, function calls.

WHERE a = 1 AND b > 5
= 2 operators × 0.0025 × rows
```

### effective_cache_size (Default: 4GB)

```
How much of the OS cache + shared_buffers is available.
Affects index scan cost estimates.

Larger value = planner assumes more pages cached
             = lower estimated I/O cost
             = more likely to use indexes
```

---

## Understanding EXPLAIN Cost

### Reading EXPLAIN Output

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Output:
Index Scan using orders_customer_idx on orders
  (cost=0.43..12.87 rows=10 width=100)
  Index Cond: (customer_id = 42)

-- Interpretation:
-- cost=0.43..12.87
--       ↑      ↑
--   startup   total

-- rows=10
--   Estimated rows returned

-- width=100
--   Estimated bytes per row
```

### Nested Plans

```sql
EXPLAIN SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;

-- Output:
Hash Join (cost=25.00..150.00 rows=1000 width=200)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o (cost=0.00..100.00 rows=5000 width=100)
  ->  Hash  (cost=15.00..15.00 rows=1000 width=100)
        ->  Seq Scan on customers c (cost=0.00..15.00 rows=1000 width=100)

-- Cost breakdown:
-- Seq Scan orders: 0..100
-- Seq Scan customers: 0..15
-- Hash build: 15..15 (startup cost includes scan)
-- Hash Join: 25..150 (includes probing all orders)
```

### Aggregation Costs

```sql
EXPLAIN SELECT count(*) FROM orders;

-- Aggregate (cost=100.00..100.01 rows=1 width=8)
--   ->  Seq Scan on orders (cost=0.00..75.00 rows=5000 width=0)

-- Note: Aggregate adds cost on top of scan
-- Startup cost high because must process all rows first
```

---

## When Cost Estimates Go Wrong

### Common Causes

```
┌─────────────────────────────────────────────────────────────────┐
│  Why Estimates Fail                                              │
│                                                                  │
│  1. Stale Statistics                                            │
│     → Run ANALYZE                                                │
│                                                                  │
│  2. Correlated Columns                                          │
│     → Use extended statistics                                   │
│                                                                  │
│  3. Incorrect Cost Constants                                    │
│     → Tune for your hardware (SSD vs HDD)                      │
│                                                                  │
│  4. Functions Hide Selectivity                                  │
│     → WHERE function(col) = value                               │
│     → Planner assumes 0.5% selectivity                         │
│                                                                  │
│  5. Parameterized Queries                                       │
│     → Planner uses generic estimates                            │
└─────────────────────────────────────────────────────────────────┘
```

### Diagnosing with EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';

-- Seq Scan on orders (cost=0.00..2500.00 rows=100 width=100)
--                                         ↑ estimated
--   (actual time=0.02..25.00 rows=50000 loops=1)
--                                 ↑ actual!

-- 100 vs 50000 = 500× underestimate!
-- Solution: ANALYZE orders; or check statistics
```

---

## Tuning Cost Constants

### For SSD Storage

```sql
-- SSDs have similar random and sequential speed
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;
SELECT pg_reload_conf();
```

### For Well-Cached Databases

```sql
-- If most data fits in RAM
ALTER SYSTEM SET effective_cache_size = '32GB';  -- Set to RAM size
ALTER SYSTEM SET random_page_cost = 1.5;         -- Random reads often cached
SELECT pg_reload_conf();
```

### Per-Tablespace Settings

```sql
-- Different settings for different storage
ALTER TABLESPACE fast_ssd SET (random_page_cost = 1.1);
ALTER TABLESPACE slow_hdd SET (random_page_cost = 4.0);
```

### Per-Session Override

```sql
-- Test different settings
SET random_page_cost = 1.1;
EXPLAIN SELECT ...;  -- See if plan changes
```

---

## Selectivity Estimation

### How Planner Estimates Rows

```sql
-- Table has 100,000 rows

-- Simple equality (uses MCV or histogram)
WHERE status = 'pending'
-- Selectivity from pg_stats

-- Range query (uses histogram)
WHERE created_at > '2024-01-01'
-- Estimates fraction from histogram bounds

-- Unknown function
WHERE custom_func(col) = true
-- Default selectivity: 0.5% (0.005)

-- NULL check
WHERE col IS NULL
-- Uses null_frac from pg_stats
```

### Compound Predicates

```sql
-- AND: multiply selectivities (assumes independence!)
WHERE a = 1 AND b = 2
-- selectivity = sel(a=1) × sel(b=2)

-- OR: add selectivities
WHERE a = 1 OR b = 2
-- selectivity = sel(a=1) + sel(b=2) - sel(a=1)×sel(b=2)

-- Problem: Correlated columns
WHERE city = 'NYC' AND state = 'NY'
-- Assumes independence, underestimates!
-- Fix: CREATE STATISTICS
```

---

## Practical Examples

### Example 1: Compare Scan Methods

```sql
-- Force different scan methods and compare costs
SET enable_seqscan = off;
EXPLAIN SELECT * FROM orders WHERE id = 100;

SET enable_indexscan = off;
SET enable_seqscan = on;
EXPLAIN SELECT * FROM orders WHERE id = 100;

-- Reset
RESET enable_seqscan;
RESET enable_indexscan;
```

### Example 2: Tune for SSD

```sql
-- Current settings
SHOW random_page_cost;  -- Probably 4.0

-- See current plan
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Try SSD settings
SET random_page_cost = 1.1;
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- Plan may change to prefer indexes!
```

### Example 3: Find Misestimates

```sql
-- Look for large estimate errors
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table WHERE complex_condition;

-- Check: estimated rows vs actual rows
-- If difference > 10×, statistics need work
```

---

## Hands-On Exercises

### Exercise 1: Calculate Cost Manually (Basic)

```sql
-- Get table statistics
SELECT 
    relpages,
    reltuples
FROM pg_class
WHERE relname = 'orders';

-- Get cost parameters
SHOW seq_page_cost;
SHOW cpu_tuple_cost;

-- Calculate expected seq scan cost:
-- cost = relpages × seq_page_cost + reltuples × cpu_tuple_cost

-- Compare with EXPLAIN
EXPLAIN SELECT * FROM orders;
```

### Exercise 2: Observe Cost Constant Impact (Intermediate)

```sql
-- Create test table
CREATE TABLE cost_test AS
SELECT generate_series AS id, 
       random() AS val
FROM generate_series(1, 100000);
CREATE INDEX cost_test_id ON cost_test(id);

-- Default settings
EXPLAIN SELECT * FROM cost_test WHERE id BETWEEN 1 AND 1000;

-- Lower random_page_cost (simulate SSD)
SET random_page_cost = 1.1;
EXPLAIN SELECT * FROM cost_test WHERE id BETWEEN 1 AND 1000;

-- Higher random_page_cost (simulate slow disk)
SET random_page_cost = 10;
EXPLAIN SELECT * FROM cost_test WHERE id BETWEEN 1 AND 1000;

RESET random_page_cost;
```

### Exercise 3: Selectivity Investigation (Advanced)

```sql
-- Check how selectivity affects plans
EXPLAIN ANALYZE SELECT * FROM cost_test WHERE id = 500;
EXPLAIN ANALYZE SELECT * FROM cost_test WHERE id BETWEEN 1 AND 10;
EXPLAIN ANALYZE SELECT * FROM cost_test WHERE id BETWEEN 1 AND 1000;
EXPLAIN ANALYZE SELECT * FROM cost_test WHERE id BETWEEN 1 AND 10000;
EXPLAIN ANALYZE SELECT * FROM cost_test WHERE id BETWEEN 1 AND 50000;

-- At what selectivity does plan switch from index to seq scan?
```

---

## Key Takeaways

1. **Cost is relative, not time**: Lower cost = better plan.

2. **Sequential I/O is baseline**: seq_page_cost = 1.0.

3. **Random I/O is expensive**: Default 4× sequential.

4. **Tune for your hardware**: SSD needs lower random_page_cost.

5. **Startup vs total cost**: Both matter for different queries.

6. **Estimates can be wrong**: Use EXPLAIN ANALYZE to verify.

7. **Selectivity drives decisions**: Low selectivity favors indexes.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/optimizer/path/costsize.c` | Cost calculation |
| `src/backend/optimizer/util/plancat.c` | Statistics retrieval |
| `src/backend/utils/adt/selfuncs.c` | Selectivity functions |

### Documentation

- [PostgreSQL Docs: Cost Constants](https://www.postgresql.org/docs/current/runtime-config-query.html#RUNTIME-CONFIG-QUERY-CONSTANTS)
- [PostgreSQL Docs: EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)

---

## References

### Cost Constants

| Parameter | Default | Description |
|-----------|---------|-------------|
| seq_page_cost | 1.0 | Sequential page read |
| random_page_cost | 4.0 | Random page read |
| cpu_tuple_cost | 0.01 | Per-row processing |
| cpu_index_tuple_cost | 0.005 | Index entry processing |
| cpu_operator_cost | 0.0025 | Per-operator evaluation |
| parallel_setup_cost | 1000 | Start parallel worker |
| parallel_tuple_cost | 0.1 | Pass tuple to leader |

### Recommended Settings

| Hardware | random_page_cost |
|----------|------------------|
| HDD | 4.0 (default) |
| SSD | 1.1 - 1.5 |
| NVMe | 1.0 - 1.1 |
| Cloud block storage | 1.1 - 2.0 |
