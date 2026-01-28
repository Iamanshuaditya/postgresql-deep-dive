# Lesson 5: Introduction to the Query Planner — Choosing the Best Execution Strategy

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 1 — Foundation: From SQL to Execution |
| **Lesson** | 5 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 4: The Rewriter |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain the role of the planner in query processing
2. Describe the difference between paths and plans
3. Understand how costs are estimated and compared
4. Identify the main access methods (sequential scan, index scan, etc.)
5. Describe the three join algorithms: nested loop, hash join, merge join
6. Interpret basic EXPLAIN output

### Key Terms

| Term | Definition |
|------|------------|
| **Planner** | The component that chooses the optimal execution strategy |
| **Path** | A lightweight representation of a possible execution strategy |
| **Plan** | The final, detailed execution tree given to the executor |
| **Cost** | A unitless estimate of execution effort (I/O + CPU) |
| **Startup Cost** | The cost to produce the first row |
| **Total Cost** | The cost to produce all rows |
| **Selectivity** | The fraction of rows that pass a filter condition |
| **Cardinality** | The estimated number of rows at each point in the plan |

---

## Introduction

We've now traced a query through parsing, analysis, and rewriting. The query tree is complete—we know exactly what data we want. But *how* should PostgreSQL retrieve it?

Consider a simple query:

```sql
SELECT * FROM orders WHERE customer_id = 42;
```

There are multiple ways to execute this:

1. **Sequential scan**: Read every row in `orders`, check each one
2. **Index scan**: If there's an index on `customer_id`, use it to find matching rows
3. **Bitmap scan**: Use an index to build a bitmap of matching pages, then read those pages

Which is best? It depends:
- How many rows does `orders` have? (10? 10 million?)
- How many rows have `customer_id = 42`? (1? 10,000?)
- Is there an index? Is it cached in memory?
- How is the data physically organized?

The **planner** (also called the optimizer) makes these decisions using:
1. **Statistics** about data distribution
2. **Cost formulas** that estimate execution time
3. **Exhaustive search** (or heuristics for complex queries) to compare alternatives

The planner is arguably the most complex subsystem in PostgreSQL. This lesson introduces its key concepts; later lessons in Module 9 will explore it in depth.

> **Why This Matters:** When a query is slow, understanding the planner helps you diagnose why. Is it choosing the wrong plan? Are statistics outdated? Is there a missing index? The planner's decisions are visible via `EXPLAIN`, and understanding costs helps you interpret them.

---

## Conceptual Foundation

### The Planner's Goals

The planner aims to find the execution plan with the **lowest estimated total cost**. Cost is a unitless number representing a combination of:

- **Disk I/O**: Reading pages from disk (or buffer cache)
- **CPU processing**: Evaluating expressions, comparing values
- **Memory operations**: Building hash tables, sorting

The planner **does not measure actual time**. It uses formulas based on statistics and configuration parameters. The hope is that lower-cost plans actually run faster—and they usually do, when statistics are accurate.

### Paths vs. Plans

The planner works in two phases:

```
┌─────────────────────────────────────────────────────────────────┐
│  Phase 1: PATH GENERATION                                        │
│                                                                  │
│  For each relation and operation, generate possible "paths"      │
│  Paths are lightweight: just cost estimate and key parameters    │
│                                                                  │
│  Examples for scanning "orders":                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ SeqScan Path: cost = 1000.00                            │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ IndexScan Path (customer_id_idx): cost = 50.25          │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ BitmapScan Path (customer_id_idx): cost = 80.30         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Keep the cheapest paths, prune the rest                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 2: PLAN CREATION                                          │
│                                                                  │
│  Convert the chosen path into a complete Plan tree               │
│  Plans have full detail needed by the executor                   │
│                                                                  │
│  IndexScan Plan:                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ - Index OID, scan direction                             │    │
│  │ - Index conditions (customer_id = 42)                   │    │
│  │ - Filter conditions (if any)                            │    │
│  │ - Target list (columns to return)                       │    │
│  │ - Cost estimates (startup: 0.29, total: 50.25)          │    │
│  │ - Rows estimate: 15                                     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### The Cost Model

PostgreSQL's cost is computed using configurable parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `seq_page_cost` | 1.0 | Cost to read one page sequentially |
| `random_page_cost` | 4.0 | Cost to read one page randomly |
| `cpu_tuple_cost` | 0.01 | Cost to process one tuple |
| `cpu_index_tuple_cost` | 0.005 | Cost per index entry |
| `cpu_operator_cost` | 0.0025 | Cost per operator/function |
| `effective_cache_size` | 4GB | Planner's estimate of available cache |

**Example: Sequential Scan Cost**

```
SeqScan Cost = (pages × seq_page_cost) + (rows × cpu_tuple_cost)

For a table with 10,000 pages and 1,000,000 rows:
Cost = (10,000 × 1.0) + (1,000,000 × 0.01)
     = 10,000 + 10,000
     = 20,000
```

**Example: Index Scan Cost**

```
IndexScan Cost ≈ (index_pages × random_page_cost) 
               + (index_tuples × cpu_index_tuple_cost)
               + (heap_pages × random_page_cost)
               + (rows × cpu_tuple_cost)
```

Random I/O is penalized (4× by default) because seeking to random pages is slower than sequential reads.

### Startup Cost vs. Total Cost

In `EXPLAIN` output, you see costs like `(cost=0.29..50.25)`:

- **0.29**: Startup cost—effort to return the first row
- **50.25**: Total cost—effort to return all rows

Why does startup cost matter? Consider:

```sql
SELECT * FROM orders ORDER BY date LIMIT 10;
```

With a `LIMIT`, we might not need all rows. Plans with low startup cost are preferable here.

### Statistics: The Foundation of Cost Estimation

The planner uses statistics from `pg_statistic` (accessed via `pg_stats`):

```sql
SELECT 
    attname,
    n_distinct,        -- Number of distinct values
    null_frac,         -- Fraction that are NULL
    avg_width,         -- Average column width in bytes
    correlation        -- Physical vs. logical ordering
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'customer_id';
```

Key statistics include:

| Statistic | Purpose |
|-----------|---------|
| `n_distinct` | Estimate selectivity of equality conditions |
| `null_frac` | Account for NULL handling |
| `most_common_vals` | Values that appear frequently |
| `most_common_freqs` | Frequencies of those values |
| `histogram_bounds` | Distribution for range queries |
| `correlation` | How sorted is the physical order? |

**If statistics are wrong, the planner makes wrong decisions.** This is why `ANALYZE` is critical.

---

## Deep Dive: Implementation Details

### Planner Entry Point

The planner is invoked from `pg_plan_queries()`:

```c
/* From src/backend/optimizer/plan/planner.c */
PlannedStmt *
planner(Query *parse, const char *query_string, 
        int cursorOptions, ParamListInfo boundParams)
{
    PlannedStmt *result;
    PlannerGlobal *glob;
    PlannerInfo *root;
    Plan       *top_plan;
    
    /* Set up planning context */
    glob = makeNode(PlannerGlobal);
    root = makeNode(PlannerInfo);
    root->parse = parse;
    root->glob = glob;
    
    /* Main planning */
    top_plan = subquery_planner(glob, parse, NULL, false, 0);
    
    /* Build the PlannedStmt */
    result = makeNode(PlannedStmt);
    result->commandType = parse->commandType;
    result->planTree = top_plan;
    result->rtable = glob->finalrtable;
    /* ... more fields ... */
    
    return result;
}
```

### Path Generation for Base Relations

For each table, `set_rel_pathlist()` generates access paths:

```c
/* From src/backend/optimizer/path/allpaths.c */
static void
set_rel_pathlist(PlannerInfo *root, RelOptInfo *rel, Index rti,
                 RangeTblEntry *rte)
{
    if (rel->rtekind == RTE_RELATION)
    {
        /* Generate sequential scan path */
        add_path(rel, create_seqscan_path(root, rel, NULL, 0));
        
        /* Generate index scan paths if indexes exist */
        create_index_paths(root, rel);
        
        /* Generate bitmap scan paths */
        create_bitmap_heap_paths(root, rel);
        
        /* Consider TID scan if applicable */
        create_tidscan_paths(root, rel);
    }
}
```

### The RelOptInfo Structure

Each relation has a `RelOptInfo` that holds optimization state:

```c
/* From src/include/nodes/pathnodes.h (simplified) */
typedef struct RelOptInfo
{
    NodeTag     type;
    
    /* Size estimates */
    double      rows;           /* Estimated number of rows */
    int         pages;          /* Number of pages */
    double      tuples;         /* Total tuples in relation */
    
    /* Path lists */
    List       *pathlist;       /* All non-dominated paths */
    Path       *cheapest_startup_path;
    Path       *cheapest_total_path;
    
    /* Index information */
    List       *indexlist;      /* Available indexes */
    
    /* Join info (for joined rels) */
    Relids      relids;         /* Set of base relids */
    
    /* ... many more fields ... */
} RelOptInfo;
```

### Path Structure

A `Path` is a lightweight plan representation:

```c
/* From src/include/nodes/pathnodes.h */
typedef struct Path
{
    NodeTag     type;
    
    NodeTag     pathtype;       /* Type of path (T_SeqScan, etc.) */
    RelOptInfo *parent;         /* The relation this path scans */
    
    /* Cost estimates */
    Cost        startup_cost;   /* Cost to get first tuple */
    Cost        total_cost;     /* Total cost to get all tuples */
    
    /* Output size */
    double      rows;           /* Estimated result rows */
    
    /* Ordering properties */
    List       *pathkeys;       /* Sort order of output */
    
    /* Parallel execution info */
    int         parallel_workers;
} Path;
```

### Cost Estimation Functions

Each access method has a costing function:

```c
/* From src/backend/optimizer/path/costsize.c */
void
cost_seqscan(Path *path, PlannerInfo *root, RelOptInfo *baserel,
             ParamPathInfo *param_info)
{
    Cost        startup_cost = 0;
    Cost        run_cost = 0;
    double      spc_seq_page_cost;
    double      spc_random_page_cost;
    double      cpu_per_tuple;
    
    /* Get tablespace-specific costs */
    get_tablespace_page_costs(baserel->reltablespace,
                              &spc_random_page_cost,
                              &spc_seq_page_cost);
    
    /* Disk costs */
    run_cost += spc_seq_page_cost * baserel->pages;
    
    /* CPU costs */
    cpu_per_tuple = cpu_tuple_cost + baserel->baserestrictcost.per_tuple;
    run_cost += cpu_per_tuple * baserel->tuples;
    
    /* Apply selectivity for restriction clauses */
    path->rows = baserel->rows;  /* Already computed with selectivity */
    
    path->startup_cost = startup_cost;
    path->total_cost = startup_cost + run_cost;
}
```

### Join Planning

For joins, the planner considers three algorithms:

```c
/* From src/backend/optimizer/path/joinpath.c */
void
add_paths_to_joinrel(PlannerInfo *root, RelOptInfo *joinrel,
                     RelOptInfo *outerrel, RelOptInfo *innerrel,
                     JoinType jointype, SpecialJoinInfo *sjinfo,
                     List *restrictlist)
{
    /* Try nested loop join */
    match_unsorted_outer(root, joinrel, outerrel, innerrel,
                         sjinfo, semifactors, restrictlist);
    
    /* Try merge join (if paths are sorted or can be sorted) */
    sort_inner_and_outer(root, joinrel, outerrel, innerrel,
                         sjinfo, restrictlist);
    
    /* Try hash join */
    hash_inner_and_outer(root, joinrel, outerrel, innerrel,
                         sjinfo, semifactors, restrictlist);
}
```

---

## Practical Examples and EXPLAIN

### Understanding EXPLAIN Output

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
```

```
                                    QUERY PLAN
-------------------------------------------------------------------------------
 Index Scan using orders_customer_id_idx on orders  (cost=0.29..50.25 rows=15 width=120)
   Index Cond: (customer_id = 42)
```

| Component | Meaning |
|-----------|---------|
| `Index Scan` | Access method chosen |
| `using orders_customer_id_idx` | Which index |
| `on orders` | Target table |
| `cost=0.29..50.25` | Startup..Total cost |
| `rows=15` | Estimated result rows |
| `width=120` | Average row width in bytes |
| `Index Cond:` | Condition applied via index |

### Comparing Plans with Different Methods

```sql
-- Force sequential scan
SET enable_indexscan = off;
SET enable_bitmapscan = off;

EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..22500.00 rows=15 width=120)
   Filter: (customer_id = 42)
```

Note:
- Higher total cost (22500 vs 50.25)  
- Condition is now a "Filter" (checked after reading each row)
- Startup cost is 0 (can start immediately)

### EXPLAIN ANALYZE: Actual vs. Estimated

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

```
                                          QUERY PLAN
---------------------------------------------------------------------------------------------
 Index Scan using orders_customer_id_idx on orders  (cost=0.29..50.25 rows=15 width=120)
                                                    (actual time=0.025..0.142 rows=12 loops=1)
   Index Cond: (customer_id = 42)
 Planning Time: 0.150 ms
 Execution Time: 0.180 ms
```

Now we see:
- `actual time=0.025..0.142`: Real milliseconds (startup..total)
- `rows=12`: Actual rows returned (estimate was 15)
- `loops=1`: How many times this node was executed

**Estimates close to actuals = good statistics!**

### EXPLAIN Formats

```sql
-- JSON format (for programmatic parsing)
EXPLAIN (FORMAT JSON) SELECT * FROM orders WHERE customer_id = 42;

-- YAML format
EXPLAIN (FORMAT YAML) SELECT * FROM orders WHERE customer_id = 42;

-- Verbose (shows more details)
EXPLAIN (VERBOSE) SELECT * FROM orders WHERE customer_id = 42;

-- Include buffer usage
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;
```

### Join Examples

```sql
EXPLAIN SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.total > 1000;
```

```
                                    QUERY PLAN
-------------------------------------------------------------------------------
 Hash Join  (cost=30.50..250.75 rows=150 width=140)
   Hash Cond: (o.customer_id = c.id)
   ->  Seq Scan on orders o  (cost=0.00..200.00 rows=500 width=120)
         Filter: (total > 1000)
   ->  Hash  (cost=20.00..20.00 rows=1000 width=20)
         ->  Seq Scan on customers c  (cost=0.00..20.00 rows=1000 width=20)
```

Tree structure:
- Outer: Sequential scan of `orders` with filter
- Inner: Build hash table from `customers`
- Join: Hash lookup for each order row

---

## Access Methods

### Sequential Scan

The simplest access method: read every page, check every row.

**When used:**
- No suitable index
- Need to read most of the table anyway
- Table is small

**Characteristics:**
- Low startup cost (can start immediately)
- Linear cost with table size
- Predictable I/O pattern (good for HDDs)

### Index Scan

Use a B-tree (or other index) to find matching row locations, then fetch rows from the heap.

**When used:**
- Selective conditions on indexed columns
- Retrieving a small fraction of rows

**Characteristics:**
- Random I/O to the heap (each row may be on different page)
- Efficient for selective queries
- Returns rows in index order

### Index Only Scan

If all needed columns are in the index, skip the heap entirely.

**When used:**
- Covering index (all SELECT columns in index)
- Visibility map shows most pages are all-visible

**Characteristics:**
- No heap access (except for recently-modified rows)
- Very efficient for count queries with indexed filters

### Bitmap Index Scan

Build a bitmap of which pages contain matching rows, then scan those pages.

**When used:**
- Medium selectivity (too many rows for index scan, fewer than seq scan threshold)
- Multiple indexes can be combined (OR conditions)

**Characteristics:**
- Two-phase: bitmap build, then heap scan
- Batches random I/O into sequential page reads
- Can combine indexes with BitmapAnd/BitmapOr

---

## Join Algorithms

### Nested Loop Join

For each row in outer, scan inner for matches.

```
for each row R1 in outer:
    for each row R2 in inner:
        if join_condition(R1, R2):
            output (R1, R2)
```

**When used:**
- Small outer, any size inner
- Inner has efficient lookup (index scan)

**Characteristics:**
- O(N × M) without index
- O(N × log M) with index on inner
- Best when outer is small and inner is indexed

### Hash Join

Build hash table from inner, probe with outer.

```
# Build phase
hash_table = {}
for each row R2 in inner:
    hash_table[hash(R2.key)] = R2

# Probe phase
for each row R1 in outer:
    for each R2 in hash_table[hash(R1.key)]:
        if join_condition(R1, R2):
            output (R1, R2)
```

**When used:**
- Equi-joins (conditions like `a = b`)
- Inner fits in memory (or work_mem)

**Characteristics:**
- O(N + M) time complexity
- Requires memory for hash table
- Falls back to batched disk-based hash if too large

### Merge Join

Both inputs sorted on join key, merge them.

```
while not end of outer and not end of inner:
    if outer.key < inner.key:
        advance outer
    elif outer.key > inner.key:
        advance inner
    else:
        output matching rows, advance both
```

**When used:**
- Both inputs already sorted (or cheap to sort)
- Large equi-joins

**Characteristics:**
- O(N log N + M log M) with sorting, O(N + M) if pre-sorted
- Good for large, pre-sorted datasets

---

## Performance Implications

### When the Planner Gets It Wrong

Common causes of bad plans:

1. **Stale statistics**: Run `ANALYZE` after bulk data changes
2. **Bad cardinality estimates**: Correlations between columns, skewed data
3. **Parameter sniffing**: Generic plans don't account for parameter values
4. **Misconfigured costs**: `random_page_cost` might be too high if data is cached

### Helping the Planner

```sql
-- Update statistics
ANALYZE orders;

-- More detailed statistics for important columns
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
ANALYZE orders;

-- Check if statistics are accurate
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
-- Compare estimated rows to actual rows
```

### Configuration Tuning

```sql
-- If data is mostly in RAM:
SET random_page_cost = 1.1;  -- Default 4.0

-- More memory for sorts and hashes:
SET work_mem = '256MB';  -- Default 4MB

-- Tell planner about available cache:
SET effective_cache_size = '24GB';
```

---

## Hands-On Exercises

### Exercise 1: Basic EXPLAIN (Basic)

**Objective**: Read and interpret EXPLAIN output.

```sql
CREATE TABLE test_plan (id serial PRIMARY KEY, val int);
INSERT INTO test_plan (val) SELECT generate_series(1, 100000);
ANALYZE test_plan;

-- Compare these plans:
EXPLAIN SELECT * FROM test_plan WHERE id = 500;
EXPLAIN SELECT * FROM test_plan WHERE val = 500;
EXPLAIN SELECT * FROM test_plan WHERE id BETWEEN 1 AND 100;
EXPLAIN SELECT * FROM test_plan WHERE id BETWEEN 1 AND 50000;
```

**Questions**: 
- Which use indexes? Why?
- At what point does a seq scan become cheaper?

### Exercise 2: EXPLAIN ANALYZE (Intermediate)

**Objective**: Compare estimates to actuals.

```sql
-- Create skewed data
CREATE TABLE skewed (id serial, category text);
INSERT INTO skewed (category) 
SELECT CASE WHEN random() < 0.95 THEN 'common' ELSE 'rare' END
FROM generate_series(1, 100000);
CREATE INDEX ON skewed(category);
ANALYZE skewed;

EXPLAIN ANALYZE SELECT * FROM skewed WHERE category = 'common';
EXPLAIN ANALYZE SELECT * FROM skewed WHERE category = 'rare';
```

**Questions**:
- Are estimated rows close to actual?
- Which access method is used for each?

### Exercise 3: Join Algorithms (Intermediate)

**Objective**: Observe different join strategies.

```sql
CREATE TABLE t1 (id serial PRIMARY KEY, val int);
CREATE TABLE t2 (id serial PRIMARY KEY, t1_id int, data text);
INSERT INTO t1 SELECT generate_series(1, 10000), random() * 100;
INSERT INTO t2 SELECT generate_series(1, 100000), random() * 10000, 'data';
CREATE INDEX ON t2(t1_id);
ANALYZE t1, t2;

-- What join method is used?
EXPLAIN SELECT t1.val, t2.data FROM t1 JOIN t2 ON t1.id = t2.t1_id;

-- Force different methods
SET enable_hashjoin = off;
EXPLAIN SELECT t1.val, t2.data FROM t1 JOIN t2 ON t1.id = t2.t1_id;

SET enable_hashjoin = on;
SET enable_mergejoin = off;
SET enable_nestloop = off;
EXPLAIN SELECT t1.val, t2.data FROM t1 JOIN t2 ON t1.id = t2.t1_id;
```

### Exercise 4: Statistics Impact (Advanced)

**Objective**: See how statistics affect planning.

```sql
-- Create table but don't analyze
CREATE TABLE no_stats (id serial, val int);
INSERT INTO no_stats SELECT generate_series(1, 100000), random() * 100;

-- Plan without statistics
EXPLAIN ANALYZE SELECT * FROM no_stats WHERE id < 100;

-- Now analyze
ANALYZE no_stats;
EXPLAIN ANALYZE SELECT * FROM no_stats WHERE id < 100;
```

**Compare** the row estimates before and after ANALYZE.

---

## Key Takeaways

1. **The planner estimates cost, not time**: Lower cost paths are chosen, hoping they're faster.

2. **Paths are lightweight, plans are complete**: The planner generates many paths, keeps the best, then builds a full plan.

3. **Statistics are critical**: All cost estimates depend on knowing row counts and data distribution. Run `ANALYZE` regularly.

4. **Access methods trade off I/O patterns**: Sequential scans are good for large fractions; index scans for small fractions; bitmap scans in between.

5. **Three join algorithms, each with sweet spots**: Nested loop for small/indexed; hash join for equi-joins; merge join for pre-sorted.

6. **EXPLAIN reveals the plan**: Use `EXPLAIN ANALYZE` to compare estimates to reality and diagnose issues.

7. **Configuration affects choices**: `random_page_cost`, `work_mem`, and other settings influence what plans the optimizer prefers.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/optimizer/plan/planner.c` | Main planner entry |
| `src/backend/optimizer/path/allpaths.c` | Path generation |
| `src/backend/optimizer/path/costsize.c` | Cost estimation |
| `src/backend/optimizer/path/joinpath.c` | Join planning |
| `src/backend/optimizer/util/relnode.c` | RelOptInfo management |
| `src/include/nodes/pathnodes.h` | Path and rel structures |

### Documentation

- [PostgreSQL Docs: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL Docs: Planner Cost Constants](https://www.postgresql.org/docs/current/runtime-config-query.html#RUNTIME-CONFIG-QUERY-CONSTANTS)
- [PostgreSQL Docs: How the Planner Uses Statistics](https://www.postgresql.org/docs/current/planner-stats.html)

### Next Lessons

- **Lesson 6**: The Executor — Running the Plan
- **Module 9**: Deep dives into scan methods, join strategies, cost estimation

---

## References

### Source Code

- `src/backend/optimizer/` — All optimizer code
- `src/include/nodes/pathnodes.h` — Path and RelOptInfo structures
- `src/include/nodes/plannodes.h` — Plan node structures

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `planner()` | planner.c | Main entry point |
| `subquery_planner()` | planner.c | Plan a single query |
| `set_rel_pathlist()` | allpaths.c | Generate paths for relation |
| `cost_seqscan()` | costsize.c | Cost a sequential scan |
| `cost_index()` | costsize.c | Cost an index scan |
| `add_paths_to_joinrel()` | joinpath.c | Generate join paths |

### Key Structures

| Structure | Location | Purpose |
|-----------|----------|---------|
| `PlannerInfo` | pathnodes.h | Planning state |
| `RelOptInfo` | pathnodes.h | Per-relation state |
| `Path` | pathnodes.h | Light execution strategy |
| `Plan` | plannodes.h | Full execution plan |
| `PlannedStmt` | plannodes.h | Top-level plan output |
