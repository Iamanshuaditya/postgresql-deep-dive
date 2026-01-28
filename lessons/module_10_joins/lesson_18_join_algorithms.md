# Lesson 18: Join Algorithms — Nested Loop, Hash, and Merge Joins

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 10 — Join Optimization |
| **Lesson** | 18 of 66 |
| **Estimated Time** | 90-120 minutes |
| **Prerequisites** | Lesson 5: Introduction to the Query Planner |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain how each join algorithm works internally
2. Understand when the planner chooses each algorithm
3. Identify join algorithms in EXPLAIN output
4. Optimize queries by influencing join selection
5. Recognize and fix inefficient join plans

### Key Terms

| Term | Definition |
|------|------------|
| **Nested Loop Join** | For each outer row, scan inner relation |
| **Hash Join** | Build hash table on inner, probe with outer |
| **Merge Join** | Merge two sorted inputs |
| **Outer Relation** | First (outer) input to a join |
| **Inner Relation** | Second (inner) input, scanned/probed |
| **Join Condition** | Predicate determining which rows match |

---

## Introduction

When you write a query that joins tables:

```sql
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;
```

PostgreSQL must decide HOW to physically combine rows from both tables. The three main algorithms are:

```
┌────────────────────────────────────────────────────────────────────────────┐
│  Join Algorithms Comparison                                                 │
│                                                                             │
│  NESTED LOOP                    HASH JOIN                  MERGE JOIN       │
│  ───────────                    ─────────                  ──────────       │
│  For each outer row:           1. Build hash table:       1. Sort both:    │
│    Scan inner rows              on inner relation          (or use index)  │
│    Match? → Output             2. Probe hash table:       2. Merge:        │
│                                  with outer rows            Walk both in   │
│  Best for:                     3. Match? → Output           sorted order   │
│  - Small outer                                                              │
│  - Indexed inner              Best for:                   Best for:        │
│  - Non-equality joins         - Large tables              - Pre-sorted     │
│                               - Equality joins only       - Range joins    │
│                                                           - Merge outputs  │
└────────────────────────────────────────────────────────────────────────────┘
```

The planner estimates costs for each algorithm and picks the cheapest.

---

## Nested Loop Join

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│  Nested Loop Algorithm                                           │
│                                                                  │
│  OUTER TABLE           INNER TABLE                              │
│  (customers)           (orders)                                 │
│  ┌─────────┐          ┌─────────────────┐                       │
│  │ Alice   │─────────▶│ Scan for Alice  │──▶ Match? Output     │
│  │ Bob     │─────────▶│ Scan for Bob    │──▶ Match? Output     │
│  │ Carol   │─────────▶│ Scan for Carol  │──▶ Match? Output     │
│  └─────────┘          └─────────────────┘                       │
│                                                                  │
│  Cost: O(N × M) with sequential scan on inner                   │
│  Cost: O(N × log M) with index scan on inner                    │
│                                                                  │
│  With index on orders.customer_id:                              │
│  ┌─────────┐          ┌──────────────────────┐                  │
│  │ Alice   │─────────▶│ Index lookup: Alice  │──▶ 3 rows       │
│  │ Bob     │─────────▶│ Index lookup: Bob    │──▶ 1 row        │
│  │ Carol   │─────────▶│ Index lookup: Carol  │──▶ 5 rows       │
│  └─────────┘          └──────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

### When Nested Loop is Chosen

- Small outer relation (few rows)
- Good index on inner relation's join column
- Non-equality joins (`<`, `>`, `LIKE`, functions)
- Limit clause (early termination possible)

### EXPLAIN Example

```sql
EXPLAIN ANALYZE 
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id
WHERE c.id = 42;
```

```
 Nested Loop  (cost=0.29..100.00 rows=10 width=200)
              (actual time=0.030..0.150 rows=10 loops=1)
   ->  Index Scan using customers_pkey on customers c
                (cost=0.29..8.30 rows=1 width=100)
                (actual time=0.015..0.020 rows=1 loops=1)
         Index Cond: (id = 42)
   ->  Index Scan using orders_customer_idx on orders o
                (cost=0.29..91.50 rows=10 width=100)
                (actual time=0.010..0.120 rows=10 loops=1)
         Index Cond: (customer_id = c.id)
```

**Key observations:**
- Outer: customers (1 row from index)
- Inner: orders (indexed lookup, runs once per outer row = 1 loop)
- Total: Very fast because index on inner

### Nested Loop Variants

| Variant | Description |
|---------|-------------|
| Nested Loop | Basic nested loop |
| Nested Loop Left Join | Preserves outer rows with no match |
| Materialize Inner | Cache inner results in memory |

---

## Hash Join

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│  Hash Join Algorithm                                             │
│                                                                  │
│  PHASE 1: BUILD (on inner/smaller relation)                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Scan customers table                                        │ │
│  │  For each row: hash(id) → bucket                            │ │
│  │                                                              │ │
│  │  Hash Table:                                                 │ │
│  │  Bucket 0: [Alice (id=100)]                                 │ │
│  │  Bucket 1: [Bob (id=42), Dave (id=101)]                     │ │
│  │  Bucket 2: [Carol (id=55)]                                  │ │
│  │  ...                                                         │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  PHASE 2: PROBE (with outer/larger relation)                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Scan orders table                                           │ │
│  │  For each row:                                               │ │
│  │    hash(customer_id) → bucket                               │ │
│  │    Search bucket for match                                  │ │
│  │    Match? → Output joined row                               │ │
│  │                                                              │ │
│  │  Order (customer_id=42) → Bucket 1 → Find Bob → Match!      │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Cost: O(N + M) - Linear!                                       │
│  Memory: Proportional to inner relation size                    │
└─────────────────────────────────────────────────────────────────┘
```

### When Hash Join is Chosen

- Large tables to join
- Equality join conditions (`=`)
- Sufficient work_mem for hash table
- No useful indexes on join columns

### EXPLAIN Example

```sql
EXPLAIN ANALYZE 
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;
```

```
 Hash Join  (cost=300.00..2500.00 rows=100000 width=200)
            (actual time=5.000..100.000 rows=100000 loops=1)
   Hash Cond: (o.customer_id = c.id)
   ->  Seq Scan on orders o  (cost=0.00..1500.00 rows=100000 width=100)
                              (actual time=0.010..40.000 rows=100000 loops=1)
   ->  Hash  (cost=200.00..200.00 rows=10000 width=100)
              (actual time=4.500..4.500 rows=10000 loops=1)
         Buckets: 16384  Batches: 1  Memory Usage: 800kB
         ->  Seq Scan on customers c  (cost=0.00..200.00 rows=10000 width=100)
                                       (actual time=0.010..2.000 rows=10000 loops=1)
```

**Key observations:**
- Build phase: Hash on customers (smaller table)
- Probe phase: Scan orders, probe hash table
- Buckets: 16384, Batches: 1 (fits in memory)
- Fast: Linear time complexity

### Hash Join Batching

When hash table exceeds work_mem:

```sql
SET work_mem = '64kB';
EXPLAIN ANALYZE SELECT * FROM orders o JOIN big_customers c ON ...;
```

```
   ->  Hash  (actual time=100.000..100.000 rows=100000 loops=1)
         Buckets: 1024  Batches: 64  Memory Usage: 65kB
```

**Batches: 64** means data was partitioned into 64 batches:
- First batch processed entirely in memory
- Remaining batches written to temp files
- Multiple passes through data (MUCH slower)

**Solution**: Increase work_mem for this query.

---

## Merge Join

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│  Merge Join Algorithm                                            │
│                                                                  │
│  PREREQUISITE: Both inputs must be sorted on join key          │
│                                                                  │
│  Customers (sorted by id):    Orders (sorted by customer_id):  │
│  ┌──────────────┐             ┌──────────────────────┐          │
│  │ id=1: Alice  │             │ customer_id=1: #101  │          │
│  │ id=2: Bob    │             │ customer_id=1: #102  │          │
│  │ id=3: Carol  │             │ customer_id=2: #103  │          │
│  │ id=4: Dave   │             │ customer_id=3: #104  │          │
│  └──────────────┘             │ customer_id=4: #105  │          │
│        │                      │ customer_id=4: #106  │          │
│        │                      └──────────────────────┘          │
│        │                              │                         │
│        └──────────┬───────────────────┘                         │
│                   ▼                                             │
│  MERGE: Walk both in sorted order                               │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Compare id=1 with customer_id=1 → Match! Output #101, #102 │ │
│  │ Advance orders (still customer_id=1? No, now 2)            │ │
│  │ Advance customers to id=2                                   │ │
│  │ Compare id=2 with customer_id=2 → Match! Output #103       │ │
│  │ ... continue ...                                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Cost: O(N + M) if already sorted, O(N log N + M log M) if not │
│  Memory: Minimal (just current row from each side)             │
└─────────────────────────────────────────────────────────────────┘
```

### When Merge Join is Chosen

- Both inputs already sorted (from index or previous sort)
- Very large tables (too big for hash table)
- Range join conditions (can use sorted order)
- Many-to-many joins (efficient for duplicates)

### EXPLAIN Example

```sql
-- Create indexes for sorted access
CREATE INDEX ON orders(customer_id);
CREATE INDEX ON customers(id);  -- Usually exists as PK

EXPLAIN ANALYZE 
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;
```

```
 Merge Join  (cost=0.00..3000.00 rows=100000 width=200)
             (actual time=0.050..150.000 rows=100000 loops=1)
   Merge Cond: (o.customer_id = c.id)
   ->  Index Scan using orders_customer_idx on orders o
                (cost=0.00..2000.00 rows=100000 width=100)
                (actual time=0.010..60.000 rows=100000 loops=1)
   ->  Index Scan using customers_pkey on customers c
                (cost=0.00..500.00 rows=10000 width=100)
                (actual time=0.010..20.000 rows=10000 loops=1)
```

**Key observations:**
- No Sort node: Using indexes for pre-sorted access
- Linear scan through both sorted inputs
- Memory efficient: No hash table needed

### Merge Join with Explicit Sort

```
 Merge Join  (cost=1500.00..3500.00 rows=100000 width=200)
   Merge Cond: (o.customer_id = c.id)
   ->  Sort  (cost=1000.00..1200.00 rows=100000 width=100)
         Sort Key: o.customer_id
         Sort Method: external merge  Disk: 10000kB
         ->  Seq Scan on orders o  (...)
   ->  Sort  (cost=500.00..550.00 rows=10000 width=100)
         Sort Key: c.id
         Sort Method: quicksort  Memory: 1000kB
         ->  Seq Scan on customers c  (...)
```

Sort is required if no index provides sorted order.

---

## Comparing the Algorithms

### Performance Characteristics

| Algorithm | Best Case | Worst Case | Memory | Requirements |
|-----------|-----------|------------|--------|--------------|
| Nested Loop | O(N) | O(N × M) | Minimal | Index on inner |
| Hash Join | O(N + M) | O(N × M) | O(M) | Equality, fits memory |
| Merge Join | O(N + M) | O(N log N + M log M) | Minimal | Sorted input |

### Selection Heuristics

```
┌─────────────────────────────────────────────────────────────────┐
│  Which algorithm is best?                                        │
│                                                                  │
│  Outer is small AND inner has index → NESTED LOOP               │
│  Tables large AND equality join AND fits work_mem → HASH JOIN   │
│  Tables sorted (indexes) AND any join type → MERGE JOIN         │
│  Non-equality join (<, >, LIKE) → NESTED LOOP or MERGE          │
│  Very large AND can't hash → MERGE JOIN with sorts              │
└─────────────────────────────────────────────────────────────────┘
```

### Forcing Join Methods

```sql
-- Disable hash join
SET enable_hashjoin = off;
EXPLAIN SELECT ...;

-- Disable merge join
SET enable_mergejoin = off;

-- Disable nested loop (be careful!)
SET enable_nestloop = off;

-- Reset all
RESET enable_hashjoin;
RESET enable_mergejoin;
RESET enable_nestloop;
```

---

## Multiple Table Joins

### Join Order Matters!

```sql
SELECT * FROM a 
JOIN b ON a.id = b.a_id 
JOIN c ON b.id = c.b_id;
```

Three possible orders:
1. (a ⋈ b) ⋈ c
2. (a ⋈ c) ⋈ b — if there's a join condition
3. (b ⋈ c) ⋈ a

PostgreSQL's planner evaluates all orderings (up to a limit) and picks cheapest.

### Join Collapse Limit

```sql
-- For more than 8 tables, stop evaluating all orderings
SHOW join_collapse_limit;  -- Default: 8

-- For EXPLAIN, show more join orderings
SET geqo_threshold = 12;  -- Use genetic algorithm above this
```

---

## Hands-On Exercises

### Exercise 1: Identifying Join Types (Basic)

```sql
-- Create test tables
CREATE TABLE left_table (id int PRIMARY KEY, val text);
CREATE TABLE right_table (id int, left_id int, data text);
INSERT INTO left_table SELECT i, 'left_' || i FROM generate_series(1, 1000) i;
INSERT INTO right_table SELECT i, random() * 1000, 'right_' || i 
FROM generate_series(1, 100000) i;

-- Compare plans
EXPLAIN ANALYZE SELECT * FROM left_table l JOIN right_table r ON l.id = r.left_id;

CREATE INDEX ON right_table(left_id);
EXPLAIN ANALYZE SELECT * FROM left_table l JOIN right_table r ON l.id = r.left_id;
```

### Exercise 2: Hash Join Memory (Intermediate)

```sql
-- Create larger tables
CREATE TABLE hash_outer AS SELECT generate_series(1, 100000) AS id;
CREATE TABLE hash_inner AS SELECT generate_series(1, 50000) AS id;

-- With default work_mem
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM hash_outer o JOIN hash_inner i ON o.id = i.id;

-- With reduced work_mem
SET work_mem = '64kB';
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM hash_outer o JOIN hash_inner i ON o.id = i.id;
-- Note: Batches > 1?
```

### Exercise 3: Merge Join with Sorts (Intermediate)

```sql
-- Force merge join
SET enable_hashjoin = off;
SET enable_nestloop = off;

EXPLAIN ANALYZE 
SELECT * FROM hash_outer o JOIN hash_inner i ON o.id = i.id;

-- Add index for sorted access
CREATE INDEX ON hash_outer(id);
CREATE INDEX ON hash_inner(id);

EXPLAIN ANALYZE 
SELECT * FROM hash_outer o JOIN hash_inner i ON o.id = i.id;
-- Compare: Sort nodes present?
```

### Exercise 4: Analyzing Multi-Table Joins (Advanced)

```sql
CREATE TABLE t1 (id int PRIMARY KEY);
CREATE TABLE t2 (id int, t1_id int);
CREATE TABLE t3 (id int, t2_id int);

INSERT INTO t1 SELECT generate_series(1, 1000);
INSERT INTO t2 SELECT i, random() * 1000 FROM generate_series(1, 10000) i;
INSERT INTO t3 SELECT i, random() * 10000 FROM generate_series(1, 100000) i;

ANALYZE t1; ANALYZE t2; ANALYZE t3;

EXPLAIN ANALYZE 
SELECT * FROM t1 
JOIN t2 ON t1.id = t2.t1_id 
JOIN t3 ON t2.id = t3.t2_id;

-- What order did the planner choose?
-- What join methods were used?
```

---

## Key Takeaways

1. **Three join algorithms**: Nested loop (small outer), hash (large equality), merge (sorted).

2. **Nested loop with index is fast**: For selective outer + indexed inner.

3. **Hash join is O(N+M)**: But needs work_mem for hash table.

4. **Merge join needs sorted input**: Free with indexes, expensive otherwise.

5. **Batches > 1 = performance problem**: Increase work_mem for hash joins.

6. **Join order affects performance**: Planner evaluates multiple orderings.

7. **Use EXPLAIN to verify**: Check which algorithm was chosen and why.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/executor/nodeNestloop.c` | Nested loop executor |
| `src/backend/executor/nodeHashjoin.c` | Hash join executor |
| `src/backend/executor/nodeMergejoin.c` | Merge join executor |
| `src/backend/optimizer/path/joinpath.c` | Join path generation |

### Documentation

- [PostgreSQL Docs: Planner Method Configuration](https://www.postgresql.org/docs/current/runtime-config-query.html#RUNTIME-CONFIG-QUERY-ENABLE)
- [PostgreSQL Docs: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

---

## References

### EXPLAIN Output Indicators

| Node | Algorithm |
|------|-----------|
| Nested Loop | Basic nested loop |
| Hash Join, Hash | Hash table join |
| Merge Join | Sorted merge |
| Sort | Sort before merge |
| Materialize | Cache results |

### Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `enable_hashjoin` | on | Enable hash joins |
| `enable_mergejoin` | on | Enable merge joins |
| `enable_nestloop` | on | Enable nested loops |
| `hash_mem_multiplier` | 2.0 | Extra memory for hash |
| `work_mem` | 4MB | Memory per operation |
