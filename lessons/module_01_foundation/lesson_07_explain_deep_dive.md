# Lesson 7: Understanding EXPLAIN in Depth — Mastering Query Plan Analysis

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 1 — Foundation: From SQL to Execution |
| **Lesson** | 7 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 6: The Executor |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Use all EXPLAIN options (ANALYZE, BUFFERS, TIMING, etc.) effectively
2. Interpret output in different formats (TEXT, JSON, YAML)
3. Identify the most expensive operations in a query plan
4. Diagnose common issues: bad estimates, inefficient scans, memory spills
5. Use EXPLAIN to guide optimization decisions

### Key Terms

| Term | Definition |
|------|------------|
| **EXPLAIN** | Command that shows a query's execution plan without running it |
| **EXPLAIN ANALYZE** | Runs the query and shows actual execution statistics |
| **Startup Cost** | Estimated effort to produce the first row |
| **Total Cost** | Estimated effort to produce all rows |
| **Actual Time** | Real execution time in milliseconds |
| **Loops** | Number of times a node was executed |
| **Buffers** | Statistics about page access (hits, reads, writes) |
| **Rows Removed** | Rows filtered out by a condition |

---

## Introduction

You've learned how PostgreSQL processes queries—parsing, planning, and executing. But how do you see what PostgreSQL is actually doing? How do you diagnose slow queries or verify that your indexes are being used?

The answer is **EXPLAIN**.

`EXPLAIN` is your window into PostgreSQL's query processing. It reveals:
- Which access methods are chosen (sequential scan, index scan, etc.)
- How tables are joined (nested loop, hash join, merge join)
- What the planner estimates (rows, costs)
- What actually happened during execution (with ANALYZE)
- Where memory and I/O are consumed (with BUFFERS)

Mastering `EXPLAIN` is perhaps the single most valuable skill for PostgreSQL performance tuning. This lesson goes deep into every aspect of EXPLAIN output.

> **The Golden Rule of EXPLAIN ANALYZE**: Always use `EXPLAIN ANALYZE` for diagnosing real performance issues. Plain `EXPLAIN` shows estimates; `ANALYZE` shows reality. Comparing them reveals planning problems.

---

## Conceptual Foundation

### EXPLAIN Options Overview

```sql
EXPLAIN [options] statement

-- Options:
ANALYZE [ boolean ]    -- Execute the query and show actual stats
VERBOSE [ boolean ]    -- Show more detail (output columns, etc.)
COSTS [ boolean ]      -- Show cost estimates (default: on)
SETTINGS [ boolean ]   -- Show non-default settings affecting plan
BUFFERS [ boolean ]    -- Show buffer usage (requires ANALYZE)
WAL [ boolean ]        -- Show WAL usage (requires ANALYZE, for writes)
TIMING [ boolean ]     -- Show actual timing (default: on with ANALYZE)
SUMMARY [ boolean ]    -- Show total planning/execution time
FORMAT { TEXT | XML | JSON | YAML }
```

### Common Combinations

```sql
-- Basic plan check (no execution)
EXPLAIN SELECT ...;

-- Full analysis with timing
EXPLAIN ANALYZE SELECT ...;

-- Analysis with I/O details
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- Full diagnostic output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS) SELECT ...;

-- Machine-readable format
EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...;

-- Safe for modifying queries (use transaction)
BEGIN;
EXPLAIN ANALYZE UPDATE ...;
ROLLBACK;
```

### Plan Tree Structure

EXPLAIN output is a tree, displayed with indentation:

```
                         QUERY PLAN
──────────────────────────────────────────────────────────────────
 Sort                          ← Root node (final output)
   Sort Key: order_date
   ->  Hash Join               ← Intermediate node (child of Sort)
         Hash Cond: (...)
         ->  Seq Scan on orders    ← Leaf node (child of Hash Join)
               Filter: (...)
         ->  Hash
               ->  Seq Scan on customers  ← Leaf node

Execution order: Bottom-up (leaves execute first)
Data flow: Bottom-up (results flow to parents)
```

---

## Deep Dive: Understanding Every Component

### Reading a Basic EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
```

```
                                    QUERY PLAN
-------------------------------------------------------------------------------
 Index Scan using orders_customer_idx on orders  (cost=0.29..50.25 rows=15 width=120)
   Index Cond: (customer_id = 42)
```

**Breakdown:**

| Component | Value | Meaning |
|-----------|-------|---------|
| Node Type | `Index Scan` | Using an index to find rows |
| Index Name | `orders_customer_idx` | Which index |
| Table | `orders` | Being scanned |
| cost | `0.29..50.25` | Startup..Total estimated cost |
| rows | `15` | Estimated output rows |
| width | `120` | Average row size in bytes |
| Index Cond | `customer_id = 42` | Condition checked via index |

### Cost Numbers Explained

```
(cost=startup..total rows=N width=W)
```

- **Startup cost**: Work before first row can be returned
  - For SeqScan: 0 (start immediately)
  - For Sort: High (must read all input first)
  - For IndexScan: Low (one index lookup)

- **Total cost**: Work to return all rows
  - Includes disk I/O + CPU time
  - Unitless (not milliseconds!)
  - Based on `seq_page_cost`, `random_page_cost`, etc.

- **Rows**: Estimated output rows (can be very wrong!)

- **Width**: Average bytes per output row

### EXPLAIN ANALYZE: Actual vs. Estimated

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

```
                                                QUERY PLAN
-----------------------------------------------------------------------------------------------------------
 Index Scan using orders_customer_idx on orders  (cost=0.29..50.25 rows=15 width=120)
                                                 (actual time=0.025..0.142 rows=12 loops=1)
   Index Cond: (customer_id = 42)
 Planning Time: 0.150 ms
 Execution Time: 0.180 ms
```

**New Components:**

| Component | Value | Meaning |
|-----------|-------|---------|
| actual time | `0.025..0.142` | Real ms (startup..total) |
| rows | `12` | Actual rows returned |
| loops | `1` | Times this node executed |
| Planning Time | `0.150 ms` | Time to create the plan |
| Execution Time | `0.180 ms` | Time to run the plan |

**Comparing Estimates to Actuals:**
- Estimated rows: 15 → Actual rows: 12 ✓ (close enough)
- If they differ by 10x or more, statistics may be stale

### The Loops Multiplier

For nested loops and correlated subqueries, `loops > 1`:

```sql
EXPLAIN ANALYZE
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

```
 Nested Loop  (actual time=0.030..150.000 rows=100000 loops=1)
   ->  Seq Scan on orders o  (actual time=0.010..10.000 rows=100000 loops=1)
   ->  Index Scan using customers_pkey on customers c  
                              (actual time=0.001..0.001 rows=1 loops=100000)
         Index Cond: (id = o.customer_id)
```

**Critical insight**: The index scan runs 100,000 times!
- Reported time: 0.001 ms
- Total time: 0.001 × 100,000 = 100 ms

### Buffer Statistics (BUFFERS)

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE total > 1000;
```

```
 Seq Scan on orders  (cost=0.00..2000.00 rows=5000 width=120)
                     (actual time=0.015..25.000 rows=5213 loops=1)
   Filter: (total > 1000)
   Rows Removed by Filter: 94787
   Buffers: shared hit=1200 read=300
 Planning Time: 0.100 ms
 Execution Time: 30.000 ms
```

**Buffer breakdown:**

| Metric | Value | Meaning |
|--------|-------|---------|
| shared hit | 1200 | Pages found in buffer cache |
| shared read | 300 | Pages read from disk |
| shared dirtied | (not shown) | Pages modified |
| shared written | (not shown) | Pages flushed to disk |
| temp read | (for sorts) | Temp pages read |
| temp written | (for sorts) | Temp pages written |

**Interpreting:**
- High `read` = cold cache or large data
- `temp` stats = sorting/hashing spilled to disk
- Aim for high `hit` ratio

### Rows Removed by Filter

```
   Filter: (total > 1000)
   Rows Removed by Filter: 94787
```

This means:
- The node read 100,000 rows (5213 kept + 94787 removed)
- The filter couldn't be pushed to an index
- This is AFTER row fetching—wasted work!

**Optimization hint**: If many rows are removed, consider an index on `total`.

### Sort Method and Memory

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders ORDER BY order_date;
```

```
 Sort  (cost=15000.00..15200.00 rows=100000 width=120)
       (actual time=150.000..200.000 rows=100000 loops=1)
   Sort Key: order_date
   Sort Method: quicksort  Memory: 25000kB
   Buffers: shared hit=1500
   ->  Seq Scan on orders  (...)
```

**Sort Methods:**

| Method | Meaning |
|--------|---------|
| `quicksort` | In-memory sort |
| `top-N heapsort` | In-memory, for LIMIT |
| `external merge` | Spilled to disk (slow!) |
| `external sort` | Disk-based sort |

**Memory:**
- Shows actual memory used
- Controlled by `work_mem`
- If you see "external", increase `work_mem`

### Hash Join Details

```sql
EXPLAIN ANALYZE 
SELECT o.*, c.name FROM orders o JOIN customers c ON o.customer_id = c.id;
```

```
 Hash Join  (cost=300.00..2500.00 rows=100000 width=140)
            (actual time=5.000..100.000 rows=100000 loops=1)
   Hash Cond: (o.customer_id = c.id)
   ->  Seq Scan on orders o  (...)
   ->  Hash  (actual time=4.500..4.500 rows=10000 loops=1)
         Buckets: 16384  Batches: 1  Memory Usage: 800kB
         ->  Seq Scan on customers c  (...)
```

**Hash Details:**

| Field | Value | Meaning |
|-------|-------|---------|
| Buckets | 16384 | Hash table size |
| Batches | 1 | 1 = all in memory |
| Memory Usage | 800kB | RAM used |

**If Batches > 1**: Hash table didn't fit in `work_mem`, spilled to disk.

### Bitmap Scan Details

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id IN (1, 2, 3, 4, 5);
```

```
 Bitmap Heap Scan on orders  (cost=25.00..500.00 rows=100 width=120)
                             (actual time=0.500..5.000 rows=87 loops=1)
   Recheck Cond: (customer_id = ANY ('{1,2,3,4,5}'::integer[]))
   Heap Blocks: exact=45
   ->  Bitmap Index Scan on orders_customer_idx  (cost=0.00..24.97 rows=100 width=0)
                                                  (actual time=0.450..0.450 rows=87 loops=1)
         Index Cond: (customer_id = ANY ('{1,2,3,4,5}'::integer[]))
```

**Components:**
- `Bitmap Index Scan`: Scans index, builds bitmap of matching TIDs
- `Bitmap Heap Scan`: Uses bitmap to fetch heap pages
- `Heap Blocks: exact=45`: 45 heap pages were accessed
- `Recheck Cond`: Condition re-checked after heap fetch (for lossy bitmaps)

---

## Output Formats

### TEXT (Default)

Human-readable, good for manual analysis.

### JSON Format

```sql
EXPLAIN (FORMAT JSON) SELECT * FROM orders WHERE customer_id = 42;
```

```json
[
  {
    "Plan": {
      "Node Type": "Index Scan",
      "Scan Direction": "Forward",
      "Index Name": "orders_customer_idx",
      "Relation Name": "orders",
      "Startup Cost": 0.29,
      "Total Cost": 50.25,
      "Plan Rows": 15,
      "Plan Width": 120,
      "Index Cond": "(customer_id = 42)"
    }
  }
]
```

Useful for:
- Programmatic parsing
- Visualization tools (pgAdmin, explain.depesz.com)
- Storing in logs for later analysis

### YAML Format

```sql
EXPLAIN (FORMAT YAML) SELECT * FROM orders WHERE customer_id = 42;
```

```yaml
- Plan:
    Node Type: "Index Scan"
    Scan Direction: "Forward"
    Index Name: "orders_customer_idx"
    Relation Name: "orders"
    Startup Cost: 0.29
    Total Cost: 50.25
    Plan Rows: 15
    Plan Width: 120
    Index Cond: "(customer_id = 42)"
```

More readable than JSON for humans.

### VERBOSE Output

```sql
EXPLAIN (VERBOSE) SELECT name FROM users WHERE id = 1;
```

```
                                    QUERY PLAN
-------------------------------------------------------------------------------
 Index Scan using users_pkey on public.users  (cost=0.29..8.30 rows=1 width=32)
   Output: name
   Index Cond: (users.id = 1)
```

Adds:
- Schema names (`public.users`)
- Output columns list
- Full expression details

---

## Common Patterns and Anti-Patterns

### Pattern 1: Sequential Scan on Large Table

```
 Seq Scan on huge_table  (cost=0.00..500000.00 rows=10000000 width=100)
   Filter: (status = 'active')
   Rows Removed by Filter: 9000000
```

**Problem**: Reading 10M rows to find 1M.

**Solution**: Add an index on `status` or a partial index:
```sql
CREATE INDEX ON huge_table(status) WHERE status = 'active';
```

### Pattern 2: Nested Loop with Large Inner

```
 Nested Loop  (actual time=0.050..5000.000 rows=1000000 loops=1)
   ->  Seq Scan on orders  (rows=1000000 loops=1)
   ->  Index Scan using customers_pkey on customers
         (actual time=0.003..0.003 rows=1 loops=1000000)
```

**Analysis**: 1M index lookups! Total inner time: 3,000ms

**May be OK** if:
- Each lookup is fast (0.003ms)
- No better alternative

**Consider**: Hash Join if customers fits in memory.

### Pattern 3: Hash Join with Multiple Batches

```
 Hash Join  (actual time=1000.00..5000.00 rows=1000000 loops=1)
   ->  ...
   ->  Hash  (actual time=900.00..900.00 rows=5000000 loops=1)
         Buckets: 65536  Batches: 64  Memory Usage: 4097kB
```

**Problem**: `Batches: 64` means 64 passes through temp files.

**Solution**: Increase `work_mem`:
```sql
SET work_mem = '256MB';
```

### Pattern 4: Sort Spilling to Disk

```
 Sort  (actual time=30000.00..35000.00 rows=10000000)
   Sort Key: created_at
   Sort Method: external merge  Disk: 500000kB
```

**Problem**: 500MB of data sorted on disk—very slow.

**Solutions**:
1. Increase `work_mem` (per-operation)
2. Add an index for pre-sorted access
3. Use LIMIT if possible

### Pattern 5: Bad Row Estimates

```
 Nested Loop  (cost=0.29..100.00 rows=1 width=100)
              (actual time=0.050..5000.00 rows=100000 loops=1)
              ^^^^^^^ Expected 1, got 100,000!
```

**Problem**: Planner expected 1 row, got 100K. Led to bad join choice.

**Solutions**:
1. Run `ANALYZE` on the table
2. Increase statistics target:
   ```sql
   ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000;
   ANALYZE t;
   ```
3. Check for correlation between columns

---

## Practical Diagnostic Workflow

### Step 1: Get the Full Picture

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS) 
SELECT ...;
```

### Step 2: Find the Bottleneck

Look for:
- Highest `actual time` (exclusive of children)
- Large `Rows Removed by Filter`
- `external merge` or `Batches > 1`
- High `read` vs `hit` ratio

### Step 3: Calculate Exclusive Time

For any node:
```
Exclusive time = node total time - sum(children total times)
```

Example:
```
 Sort: actual time=0.5..150.0
   ->  Hash Join: actual time=0.3..120.0

Sort exclusive time = 150.0 - 120.0 = 30ms
Hash Join exclusive time = 120.0 - sum(children) = ...
```

### Step 4: Check Estimate Accuracy

Compare `rows` (estimate) vs `rows` (actual):
- Within 2x: Good
- 10x off: May affect plan choice
- 100x+ off: Serious statistics problem

### Step 5: Test Alternatives

```sql
-- Force different index
SET enable_seqscan = off;
EXPLAIN ANALYZE ...;

-- Force different join
SET enable_hashjoin = off;
EXPLAIN ANALYZE ...;

-- Test with more memory
SET work_mem = '256MB';
EXPLAIN ANALYZE ...;
```

---

## Hands-On Exercises

### Exercise 1: Reading EXPLAIN Output (Basic)

```sql
CREATE TABLE explain_test (
    id serial PRIMARY KEY,
    category text,
    value int,
    data text
);
INSERT INTO explain_test (category, value, data)
SELECT 
    'cat' || (random() * 10)::int,
    random() * 1000,
    repeat('x', 100)
FROM generate_series(1, 100000);
CREATE INDEX ON explain_test(category);
CREATE INDEX ON explain_test(value);
ANALYZE explain_test;

-- Analyze these plans:
EXPLAIN SELECT * FROM explain_test WHERE id = 500;
EXPLAIN SELECT * FROM explain_test WHERE category = 'cat5';
EXPLAIN SELECT * FROM explain_test WHERE value BETWEEN 0 AND 10;
EXPLAIN SELECT * FROM explain_test ORDER BY category;
```

### Exercise 2: Comparing Estimated vs Actual (Intermediate)

```sql
-- Create skewed data
CREATE TABLE skewed (
    id serial PRIMARY KEY,
    status text
);
INSERT INTO skewed (status)
SELECT CASE 
    WHEN random() < 0.99 THEN 'common'
    ELSE 'rare'
END
FROM generate_series(1, 100000);
CREATE INDEX ON skewed(status);
ANALYZE skewed;

-- Compare estimates for common vs rare
EXPLAIN ANALYZE SELECT * FROM skewed WHERE status = 'common';
EXPLAIN ANALYZE SELECT * FROM skewed WHERE status = 'rare';
```

**Questions**:
- Are row estimates accurate?
- What access methods are used?

### Exercise 3: Buffer Analysis (Intermediate)

```sql
-- Run twice to see cache effects
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM explain_test;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM explain_test;

-- Compare hit vs read
```

### Exercise 4: Finding Sort Memory Spills (Intermediate)

```sql
SET work_mem = '64kB';
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM explain_test ORDER BY data;

SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM explain_test ORDER BY data;
```

**Compare**: Sort Method and Disk usage.

### Exercise 5: Debugging a Bad Plan (Advanced)

```sql
-- Create tables with bad statistics
CREATE TABLE orders_bad AS 
SELECT id, (random() * 1000)::int as customer_id, random() * 1000 as total
FROM generate_series(1, 100000) id;

-- Don't analyze - intentionally bad stats
-- Compare plans before and after ANALYZE
EXPLAIN ANALYZE SELECT * FROM orders_bad WHERE total > 900;
ANALYZE orders_bad;
EXPLAIN ANALYZE SELECT * FROM orders_bad WHERE total > 900;
```

---

## Key Takeaways

1. **Always use EXPLAIN ANALYZE for real diagnosis**: Plain EXPLAIN shows guesses; ANALYZE shows reality.

2. **Costs are relative, not absolute**: Don't compare cost numbers to time. Compare actual times.

3. **Multiply by loops**: For nested operations, time = reported time × loops.

4. **Check estimate accuracy**: Big misestimates → bad plans → run ANALYZE.

5. **Watch for disk spills**: `external merge` and `Batches > 1` indicate memory pressure.

6. **Buffers tell the I/O story**: High `read` vs `hit` means cold cache or data too large.

7. **Rows Removed by Filter = wasted work**: Consider indexes to reduce filtering.

8. **Use JSON format for tools**: Many visualization tools expect JSON output.

---

## Further Reading

### Tools

- [explain.depesz.com](https://explain.depesz.com) — Paste EXPLAIN output for visual analysis
- [explain.dalibo.com](https://explain.dalibo.com) — Another visualization tool
- [pgAdmin](https://www.pgadmin.org/) — Has built-in EXPLAIN visualization

### Documentation

- [PostgreSQL Docs: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL Docs: EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
- [PostgreSQL Wiki: Using EXPLAIN](https://wiki.postgresql.org/wiki/Using_EXPLAIN)

### Next Lessons

- **Module 2**: Storage Architecture (understand what's being scanned)
- **Module 9**: Query Optimization (use EXPLAIN insights to optimize)

---

## References

### EXPLAIN Options Reference

| Option | Default | Description |
|--------|---------|-------------|
| ANALYZE | off | Execute query, show actual stats |
| VERBOSE | off | Show additional detail |
| COSTS | on | Show cost estimates |
| SETTINGS | off | Show non-default settings |
| BUFFERS | off | Show buffer usage |
| WAL | off | Show WAL usage (writes only) |
| TIMING | on (with ANALYZE) | Show actual timing |
| SUMMARY | on (with ANALYZE) | Show totals |
| FORMAT | TEXT | Output format |

### Common Node Types Quick Reference

| Node | Description | Key Stats |
|------|-------------|-----------|
| Seq Scan | Full table scan | Filter, Rows Removed |
| Index Scan | B-tree lookup + heap | Index Cond |
| Index Only Scan | B-tree only | Heap Fetches |
| Bitmap Heap Scan | Bitmap-driven heap | Heap Blocks, Recheck |
| Nested Loop | For-each join | loops count |
| Hash Join | Hash table join | Buckets, Batches, Memory |
| Merge Join | Sorted merge | Sort nodes below |
| Sort | In-memory or disk | Sort Method, Memory/Disk |
| Aggregate | SUM, COUNT, etc. | Strategy |
| Limit | Row limiting | Actual rows |
| Gather | Parallel collection | Workers |
