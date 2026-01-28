# Lesson 24: Statistics and the Query Planner — How PostgreSQL Estimates Costs

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 9 — System Catalogs |
| **Lesson** | 24 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 5: Introduction to the Query Planner, Lesson 23: System Catalogs |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Understand how PostgreSQL collects and stores statistics
2. Read and interpret pg_stats for any column
3. Explain how statistics influence query plans
4. Diagnose and fix bad estimates
5. Configure statistics collection for optimal planning

### Key Terms

| Term | Definition |
|------|------------|
| **ANALYZE** | Command that collects statistics |
| **pg_statistic** | Internal catalog storing raw statistics |
| **pg_stats** | Human-readable view of statistics |
| **n_distinct** | Estimated number of distinct values |
| **most_common_vals** | Most frequent values and their frequencies |
| **histogram_bounds** | Boundaries for value distribution |
| **correlation** | Physical vs logical ordering correlation |

---

## Introduction

The query planner doesn't know the actual data—it uses **statistics** to estimate:

```
┌─────────────────────────────────────────────────────────────────┐
│  Why Statistics Matter                                           │
│                                                                  │
│  Query: SELECT * FROM orders WHERE status = 'pending'           │
│                                                                  │
│  Without statistics:                                             │
│  → Planner has no idea how many rows match                      │
│  → Might choose seq scan when index scan is better               │
│  → Or vice versa                                                 │
│                                                                  │
│  With statistics:                                                │
│  → Knows 'pending' appears in 5% of rows                        │
│  → Estimates ~5000 rows out of 100,000                          │
│  → Chooses index scan (more efficient for selective queries)    │
│                                                                  │
│  Bad statistics → Bad plans → Slow queries!                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## How Statistics Are Collected

### The ANALYZE Command

```sql
-- Analyze a single table
ANALYZE users;

-- Analyze specific columns
ANALYZE users (email, created_at);

-- Analyze entire database
ANALYZE;

-- Verbose output shows what's happening
ANALYZE VERBOSE users;
```

### Autovacuum Runs ANALYZE

```sql
-- Check when tables were last analyzed
SELECT 
    relname,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze
FROM pg_stat_user_tables
ORDER BY last_analyze NULLS FIRST;
```

### What ANALYZE Collects

For each column, ANALYZE samples rows and computes:

1. **Null fraction**: % of NULL values
2. **Average width**: Bytes per value
3. **n_distinct**: Number of unique values
4. **Most common values (MCV)**: Top N values and frequencies
5. **Histogram**: Distribution of remaining values
6. **Correlation**: Physical vs logical order alignment

---

## Reading pg_stats

### The pg_stats View

```sql
SELECT 
    schemaname,
    tablename,
    attname,
    null_frac,
    avg_width,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds,
    correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

### Understanding Each Statistic

#### null_frac — NULL Percentage

```sql
-- null_frac = 0.05 means 5% of values are NULL
SELECT null_frac FROM pg_stats 
WHERE tablename = 'users' AND attname = 'phone';
```

#### n_distinct — Unique Value Count

```sql
-- Positive: Exact count (e.g., 100 distinct values)
-- Negative: Fraction of table size (e.g., -0.5 = 50% unique)
SELECT n_distinct FROM pg_stats 
WHERE tablename = 'orders' AND attname = 'customer_id';

-- n_distinct = -1.0 means all values are unique (like a primary key)
-- n_distinct = 5 means exactly 5 distinct values
-- n_distinct = -0.1 means ~10% of rows have unique values
```

#### most_common_vals / most_common_freqs — MCV List

```sql
-- Top values and their frequencies
SELECT 
    most_common_vals,   -- {'pending','shipped','delivered','cancelled'}
    most_common_freqs   -- {0.35, 0.30, 0.25, 0.10}
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';

-- Interpretation:
-- 'pending' appears in 35% of rows
-- 'shipped' appears in 30% of rows
-- etc.
```

#### histogram_bounds — Value Distribution

```sql
-- For values NOT in MCV list
SELECT histogram_bounds
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'total';

-- Returns: {0, 50, 100, 150, 200, 300, 500, 1000, 5000, 10000}
-- Divides remaining values into equal-frequency buckets
-- Each bucket contains ~10% of non-MCV values
```

#### correlation — Physical Ordering

```sql
SELECT correlation
FROM pg_stats
WHERE tablename = 'events' AND attname = 'created_at';

-- correlation = 1.0: Perfectly ordered (sequential inserts)
-- correlation = 0.0: Random order
-- correlation = -1.0: Reverse ordered

-- High correlation → Index range scans are efficient
-- Low correlation → Many random I/O operations
```

---

## How the Planner Uses Statistics

### Selectivity Estimation

```sql
-- Query: WHERE status = 'pending'
-- Planner calculates selectivity:

-- If 'pending' is in MCV list:
--   selectivity = MCV frequency = 0.35

-- If not in MCV list:
--   remaining_frac = 1 - sum(MCVfreqs) - null_frac
--   remaining_distinct = n_distinct - length(MCV)
--   selectivity = remaining_frac / remaining_distinct
```

### Row Estimation

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'pending';

-- Output: Seq Scan on orders  (rows=35000)
-- Calculation: 100000 (total rows) × 0.35 (selectivity) = 35000
```

### Range Queries and Histograms

```sql
EXPLAIN SELECT * FROM orders WHERE total BETWEEN 100 AND 500;

-- Planner uses histogram_bounds to estimate:
-- {0, 50, 100, 150, 200, 300, 500, 1000, 5000, 10000}
-- Each bucket = 10% of non-MCV values
-- Buckets 3,4,5,6 cover 100-500 → ~40% of non-MCV values
```

---

## Inspecting Bad Estimates

### Compare Estimated vs Actual

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';

--   Seq Scan on orders (rows=35000) (actual rows=42000)
--                       ^^^^^^^^^^   ^^^^^^^^^^^^^^^^
--                       Estimated    Actual

-- Large differences indicate stale or insufficient statistics
```

### Common Causes of Bad Estimates

```
┌─────────────────────────────────────────────────────────────────┐
│  Causes of Bad Row Estimates                                     │
│                                                                  │
│  1. Stale statistics                                            │
│     → Run ANALYZE                                                │
│                                                                  │
│  2. Correlated columns                                          │
│     → WHERE city = 'NYC' AND state = 'NY'                       │
│     → Planner assumes independence, underestimates              │
│     → Use extended statistics                                   │
│                                                                  │
│  3. Skewed data not captured                                    │
│     → Increase statistics target                                │
│                                                                  │
│  4. Functions hide selectivity                                  │
│     → WHERE lower(email) = 'test@example.com'                   │
│     → Planner can't see through function                        │
│                                                                  │
│  5. Complex expressions                                         │
│     → WHERE a + b > 100                                         │
│     → No statistics for expressions                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tuning Statistics Collection

### Statistics Target

Controls how many values are sampled:

```sql
-- Default: 100 (100 MCV values + 100 histogram buckets)
SHOW default_statistics_target;

-- Increase for skewed columns
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;

-- Then re-analyze
ANALYZE orders (status);

-- Check new statistics
SELECT array_length(most_common_vals, 1), array_length(histogram_bounds, 1)
FROM pg_stats WHERE tablename = 'orders' AND attname = 'status';
```

### When to Increase Statistics Target

- Column has many distinct values
- Data distribution is skewed
- Column is frequently used in WHERE clauses
- Query plans show bad estimates

### Global Default

```sql
-- Increase for analytics workloads
ALTER SYSTEM SET default_statistics_target = 200;
SELECT pg_reload_conf();
```

---

## Extended Statistics

### The Problem: Correlated Columns

```sql
-- City and state are correlated!
SELECT * FROM addresses 
WHERE city = 'New York' AND state = 'NY';

-- Planner assumes independence:
-- P(city='NYC') × P(state='NY') = 0.05 × 0.10 = 0.005 (0.5%)
-- Reality: P(city='NYC' AND state='NY') = 0.05 (5%)
-- 10× underestimate!
```

### Creating Extended Statistics

```sql
-- Track dependencies between columns
CREATE STATISTICS city_state_stats (dependencies) 
ON city, state FROM addresses;

ANALYZE addresses;

-- Now planner understands the correlation
EXPLAIN SELECT * FROM addresses 
WHERE city = 'New York' AND state = 'NY';
```

### Types of Extended Statistics

| Type | Purpose |
|------|---------|
| `dependencies` | Track functional dependencies |
| `ndistinct` | Track distinct combinations |
| `mcv` | Track most common value combinations |

```sql
-- Full extended statistics
CREATE STATISTICS full_stats (dependencies, ndistinct, mcv)
ON city, state, zip FROM addresses;
```

---

## Practical Examples

### Example 1: Fix Underestimate

```sql
-- Bad estimate
EXPLAIN ANALYZE SELECT * FROM users WHERE country = 'US';
-- Seq Scan (rows=1000) (actual rows=50000)

-- Check statistics
SELECT most_common_vals, most_common_freqs 
FROM pg_stats WHERE tablename = 'users' AND attname = 'country';
-- Nothing shown or stale

-- Fix: Re-analyze
ANALYZE users;

-- Now accurate
EXPLAIN ANALYZE SELECT * FROM users WHERE country = 'US';
-- Index Scan (rows=48000) (actual rows=50000)
```

### Example 2: Increase Target for Skewed Column

```sql
-- Column with many distinct low-frequency values
SELECT n_distinct, array_length(most_common_vals, 1)
FROM pg_stats WHERE tablename = 'products' AND attname = 'category';
-- n_distinct=500, but only 100 MCVs tracked

-- Increase statistics target
ALTER TABLE products ALTER COLUMN category SET STATISTICS 500;
ANALYZE products;

-- Now more values tracked
SELECT array_length(most_common_vals, 1) FROM pg_stats 
WHERE tablename = 'products' AND attname = 'category';
-- Returns 500
```

### Example 3: Expression Statistics

```sql
-- Statistics on expressions (PG 14+)
CREATE STATISTICS expr_stats ON (lower(email)) FROM users;
ANALYZE users;

-- Now this can use statistics:
EXPLAIN SELECT * FROM users WHERE lower(email) = 'test@example.com';
```

---

## Hands-On Exercises

### Exercise 1: Read Statistics (Basic)

```sql
-- View all statistics for a table
SELECT 
    attname,
    null_frac,
    n_distinct,
    array_length(most_common_vals, 1) AS mcv_count
FROM pg_stats
WHERE tablename = 'orders';
```

### Exercise 2: Observe Estimate Changes (Intermediate)

```sql
-- Create test table
CREATE TABLE stats_test AS
SELECT 
    generate_series(1, 100000) AS id,
    CASE WHEN random() < 0.8 THEN 'common' ELSE 'rare' END AS category;

-- Before ANALYZE
EXPLAIN SELECT * FROM stats_test WHERE category = 'common';
-- Estimate will be wrong (no statistics)

-- After ANALYZE
ANALYZE stats_test;
EXPLAIN SELECT * FROM stats_test WHERE category = 'common';
-- Estimate should be ~80000
```

### Exercise 3: Extended Statistics (Advanced)

```sql
-- Create correlated data
CREATE TABLE corr_test AS
SELECT 
    generate_series(1, 100000) AS id,
    CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END AS group_code,
    '' AS subcode;

UPDATE corr_test SET subcode = 
    CASE WHEN group_code = 'A' THEN 'A1' ELSE 'B1' END;

ANALYZE corr_test;

-- Query with correlated predicates
EXPLAIN ANALYZE SELECT * FROM corr_test 
WHERE group_code = 'A' AND subcode = 'A1';
-- Note the estimate vs actual

-- Add extended statistics
CREATE STATISTICS corr_stats (dependencies) ON group_code, subcode FROM corr_test;
ANALYZE corr_test;

-- Check improved estimate
EXPLAIN ANALYZE SELECT * FROM corr_test 
WHERE group_code = 'A' AND subcode = 'A1';
```

---

## Key Takeaways

1. **Statistics drive planning**: Bad stats → bad plans.

2. **ANALYZE collects statistics**: Runs automatically via autovacuum.

3. **pg_stats is readable**: Use it to inspect column statistics.

4. **MCV captures frequent values**: Histogram handles the rest.

5. **Correlation affects index efficiency**: High correlation = faster range scans.

6. **Increase target for skewed data**: More MCVs = better estimates.

7. **Extended statistics help correlated columns**: Track multi-column dependencies.

8. **EXPLAIN ANALYZE reveals truth**: Compare estimated vs actual rows.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/commands/analyze.c` | ANALYZE implementation |
| `src/backend/utils/adt/selfuncs.c` | Selectivity estimation |
| `src/backend/statistics/` | Extended statistics |
| `src/include/catalog/pg_statistic.h` | Stats catalog definition |

### Documentation

- [PostgreSQL Docs: ANALYZE](https://www.postgresql.org/docs/current/sql-analyze.html)
- [PostgreSQL Docs: pg_stats](https://www.postgresql.org/docs/current/view-pg-stats.html)
- [PostgreSQL Docs: Extended Statistics](https://www.postgresql.org/docs/current/planner-stats.html#PLANNER-STATS-EXTENDED)

---

## References

### pg_stats Columns

| Column | Description |
|--------|-------------|
| null_frac | Fraction of NULL values |
| avg_width | Average byte width |
| n_distinct | Distinct value estimate |
| most_common_vals | Array of MCV values |
| most_common_freqs | Array of MCV frequencies |
| histogram_bounds | Bucket boundaries |
| correlation | Physical/logical order correlation |
| most_common_elems | MCV for array elements |
| elem_count_histogram | Array element frequency |

### Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `default_statistics_target` | 100 | Default MCV/histogram size |
| `autovacuum_analyze_threshold` | 50 | Rows before auto-analyze |
| `autovacuum_analyze_scale_factor` | 0.1 | Fraction for auto-analyze |
