# Lesson 29: Partitioning — Divide and Conquer Large Tables

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 14 — Advanced Features |
| **Lesson** | 29 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 8: Heap Storage, Lesson 23: System Catalogs |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain how partitioning works internally
2. Choose appropriate partitioning strategies
3. Implement range, list, and hash partitioning
4. Understand partition pruning and its impact on performance
5. Manage partitions (add, detach, maintain)

### Key Terms

| Term | Definition |
|------|------------|
| **Partitioned Table** | Virtual table divided into physical partitions |
| **Partition** | Physical table storing subset of data |
| **Partition Key** | Column(s) determining which partition holds row |
| **Partition Pruning** | Excluding irrelevant partitions from queries |
| **Constraint Exclusion** | Similar to pruning for inheritance partitioning |
| **Partition Bound** | Range/list values defining partition scope |

---

## Introduction

### Why Partition?

```
┌─────────────────────────────────────────────────────────────────┐
│  Large Table Problems                                            │
│                                                                  │
│  Without Partitioning:                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ orders table: 10 billion rows, 2TB                          ││
│  │                                                              ││
│  │ Problems:                                                    ││
│  │ • VACUUM takes days                                         ││
│  │ • Index maintenance is slow                                 ││
│  │ • Full table scans are painful                              ││
│  │ • Dropping old data requires DELETE (slow, bloat)          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  With Partitioning:                                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ orders (partitioned by month)                               ││
│  │   ├── orders_2024_01 (100M rows)                           ││
│  │   ├── orders_2024_02 (100M rows)                           ││
│  │   ├── orders_2024_03 (100M rows)                           ││
│  │   └── ...                                                   ││
│  │                                                              ││
│  │ Benefits:                                                    ││
│  │ • VACUUM each partition independently                       ││
│  │ • Smaller indexes per partition                             ││
│  │ • Query only relevant partitions (pruning)                  ││
│  │ • Drop old data: DROP TABLE orders_2023_01 (instant!)      ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## Partitioning Strategies

### Range Partitioning

Best for: Time-series data, ordered values

```sql
-- Create partitioned table
CREATE TABLE orders (
    id bigserial,
    customer_id int,
    order_date date,
    total numeric
) PARTITION BY RANGE (order_date);

-- Create partitions
CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
    
CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
    
CREATE TABLE orders_2024_q3 PARTITION OF orders
    FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');
    
CREATE TABLE orders_2024_q4 PARTITION OF orders
    FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');
```

### List Partitioning

Best for: Categorical data with known values

```sql
CREATE TABLE customers (
    id serial,
    name text,
    region text,
    email text
) PARTITION BY LIST (region);

CREATE TABLE customers_americas PARTITION OF customers
    FOR VALUES IN ('NA', 'SA', 'CA');
    
CREATE TABLE customers_europe PARTITION OF customers
    FOR VALUES IN ('EU', 'UK');
    
CREATE TABLE customers_asia PARTITION OF customers
    FOR VALUES IN ('ASIA', 'OCEANIA');
    
-- Default partition for unexpected values
CREATE TABLE customers_other PARTITION OF customers
    DEFAULT;
```

### Hash Partitioning

Best for: Even distribution when no natural range/list

```sql
CREATE TABLE events (
    id bigserial,
    user_id int,
    event_type text,
    created_at timestamp
) PARTITION BY HASH (user_id);

-- Create partitions (must cover all modulus values)
CREATE TABLE events_0 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE events_2 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE events_3 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

## Partition Pruning

### How Pruning Works

```
┌─────────────────────────────────────────────────────────────────┐
│  Partition Pruning                                               │
│                                                                  │
│  Query: SELECT * FROM orders WHERE order_date = '2024-06-15'   │
│                                                                  │
│  Without pruning:                                               │
│  Scan ALL partitions:                                           │
│  orders_2024_q1 ✗ (unnecessary)                                 │
│  orders_2024_q2 ✓ (contains data)                               │
│  orders_2024_q3 ✗ (unnecessary)                                 │
│  orders_2024_q4 ✗ (unnecessary)                                 │
│                                                                  │
│  With pruning:                                                  │
│  Planner checks: '2024-06-15' matches Q2 bounds                 │
│  Scan ONLY orders_2024_q2                                       │
│                                                                  │
│  Result: 4x less I/O!                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Verifying Pruning in EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE order_date = '2024-06-15';

-- Output shows only one partition scanned:
Append  (cost=0.00..100.00 rows=1000 width=100)
  ->  Seq Scan on orders_2024_q2 orders_1  (cost=0.00..100.00 rows=1000 width=100)
        Filter: (order_date = '2024-06-15'::date)

-- Compare with query on non-partition key:
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- All partitions must be scanned!
```

### Runtime Pruning

```sql
-- Prepared statement with parameter
PREPARE find_orders(date) AS 
SELECT * FROM orders WHERE order_date = $1;

EXPLAIN ANALYZE EXECUTE find_orders('2024-06-15');

-- Pruning happens at execution time (runtime pruning)
```

---

## Indexes on Partitioned Tables

### Partition-Local Indexes

```sql
-- Create index on partitioned table
CREATE INDEX orders_date_idx ON orders (order_date);

-- This creates an index on EACH partition:
-- orders_2024_q1_order_date_idx
-- orders_2024_q2_order_date_idx
-- etc.
```

### Unique Constraints

```sql
-- Unique constraint MUST include partition key!
CREATE TABLE orders (
    id bigserial,
    order_date date,
    ...
    PRIMARY KEY (id, order_date)  -- Includes partition key!
) PARTITION BY RANGE (order_date);

-- This won't work:
-- PRIMARY KEY (id)  -- Error: unique constraint must include partition key
```

### Why Include Partition Key?

```
Each partition enforces uniqueness independently.
Without partition key in constraint, same id could exist in multiple partitions!

orders_q1: id=1, order_date='2024-01-15'
orders_q2: id=1, order_date='2024-05-15'  -- Duplicate id allowed!
```

---

## Managing Partitions

### Adding Partitions

```sql
-- Add new partition for future data
CREATE TABLE orders_2025_q1 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

-- Best practice: Create partitions ahead of time
-- Use a cron job or scheduler
```

### Detaching Partitions

```sql
-- Detach (fast, keeps data)
ALTER TABLE orders DETACH PARTITION orders_2023_q1;

-- Now orders_2023_q1 is a standalone table
-- Can query directly, archive, or drop

-- Reattach if needed
ALTER TABLE orders ATTACH PARTITION orders_2023_q1
    FOR VALUES FROM ('2023-01-01') TO ('2023-04-01');
```

### Dropping Old Data

```sql
-- Old way (slow, causes bloat):
DELETE FROM orders WHERE order_date < '2023-01-01';

-- Partitioned way (instant!):
ALTER TABLE orders DETACH PARTITION orders_2022_q4;
DROP TABLE orders_2022_q4;
```

### Splitting Partitions (PG 17+)

```sql
-- Split existing partition into smaller ones
ALTER TABLE orders SPLIT PARTITION orders_2024_h1 INTO
    (PARTITION orders_2024_q1 FOR VALUES FROM ('2024-01-01') TO ('2024-04-01'),
     PARTITION orders_2024_q2 FOR VALUES FROM ('2024-04-01') TO ('2024-07-01'));
```

---

## Multi-Level Partitioning

### Sub-Partitioning

```sql
-- First level: by region
CREATE TABLE sales (
    id bigserial,
    region text,
    sale_date date,
    amount numeric
) PARTITION BY LIST (region);

-- Create sub-partitioned partition
CREATE TABLE sales_americas PARTITION OF sales
    FOR VALUES IN ('NA', 'SA')
    PARTITION BY RANGE (sale_date);

-- Create sub-partitions
CREATE TABLE sales_americas_2024_q1 PARTITION OF sales_americas
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
    
CREATE TABLE sales_americas_2024_q2 PARTITION OF sales_americas
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

### Structure

```
sales (partitioned by region)
├── sales_americas (sub-partitioned by date)
│   ├── sales_americas_2024_q1
│   ├── sales_americas_2024_q2
│   └── ...
├── sales_europe (sub-partitioned by date)
│   ├── sales_europe_2024_q1
│   └── ...
└── sales_asia
    └── ...
```

---

## Partition Maintenance

### VACUUM and ANALYZE

```sql
-- VACUUM individual partition (faster than whole table)
VACUUM orders_2024_q1;
ANALYZE orders_2024_q1;

-- Or the whole partitioned table
VACUUM orders;  -- Vacuums all partitions
```

### Monitoring Partition Sizes

```sql
SELECT 
    parent.relname AS parent_table,
    child.relname AS partition_name,
    pg_size_pretty(pg_relation_size(child.oid)) AS size,
    pg_stat_get_live_tuples(child.oid) AS live_tuples
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relname = 'orders'
ORDER BY child.relname;
```

### Finding Partition Bounds

```sql
-- View partition structure
SELECT 
    c.relname AS partition,
    pg_get_expr(c.relpartbound, c.oid) AS partition_bound
FROM pg_class c
JOIN pg_inherits i ON c.oid = i.inhrelid
WHERE i.inhparent = 'orders'::regclass
ORDER BY c.relname;
```

---

## Performance Considerations

### When to Partition

| Criteria | Partitioning Helps |
|----------|-------------------|
| Table > 100GB | ✓ Yes |
| Time-series queries | ✓ Yes |
| Regular data archival | ✓ Yes |
| Random access by PK | ✗ Less benefit |
| Small tables | ✗ Overhead |

### Partition Size Guidelines

- **Too few partitions**: Little benefit
- **Too many partitions**: Planning overhead
- **Sweet spot**: 100-10,000 partitions
- **Rule of thumb**: 10M-100M rows per partition

### Common Pitfalls

```sql
-- 1. Queries without partition key in WHERE
SELECT * FROM orders WHERE customer_id = 42;
-- Scans ALL partitions!

-- 2. JOINs with partitioned tables
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;
-- Each partition joins separately

-- 3. Foreign keys TO partitioned tables (limitations)
-- Pre-PG 12: not supported
-- PG 12+: Supported but with caveats
```

---

## Hands-On Exercises

### Exercise 1: Create Range Partitioned Table (Basic)

```sql
-- Create partitioned table
CREATE TABLE logs (
    id bigserial,
    log_time timestamp,
    message text
) PARTITION BY RANGE (log_time);

-- Create partitions
CREATE TABLE logs_2024_01 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE logs_2024_02 PARTITION OF logs
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Insert data
INSERT INTO logs (log_time, message)
SELECT 
    '2024-01-15'::timestamp + (random() * 30) * interval '1 day',
    'Log message ' || i
FROM generate_series(1, 10000) i;

-- Check distribution
SELECT tableoid::regclass, count(*) FROM logs GROUP BY 1;
```

### Exercise 2: Verify Partition Pruning (Intermediate)

```sql
-- Query with partition key
EXPLAIN ANALYZE SELECT * FROM logs WHERE log_time = '2024-01-20';
-- Note: Only one partition scanned

-- Query without partition key
EXPLAIN ANALYZE SELECT * FROM logs WHERE message LIKE '%5000%';
-- Note: All partitions scanned
```

### Exercise 3: Partition Management (Advanced)

```sql
-- Add new partition
CREATE TABLE logs_2024_03 PARTITION OF logs
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Detach old partition
ALTER TABLE logs DETACH PARTITION logs_2024_01;

-- Query detached partition directly
SELECT count(*) FROM logs_2024_01;

-- Reattach
ALTER TABLE logs ATTACH PARTITION logs_2024_01
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

---

## Key Takeaways

1. **Partitioning divides tables physically**: Each partition is a separate table.

2. **Three strategies**: Range, List, Hash—choose based on data and queries.

3. **Partition pruning is crucial**: Queries must include partition key.

4. **Unique constraints include partition key**: Required for enforcement.

5. **Dropping partitions is instant**: Unlike DELETE.

6. **Maintenance is per-partition**: VACUUM, index rebuilds are smaller.

7. **Don't over-partition**: 100-10,000 partitions is typical.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/partitioning/` | Partitioning code |
| `src/backend/optimizer/path/partprune.c` | Partition pruning |
| `src/backend/catalog/partition.c` | Partition catalog |

### Documentation

- [PostgreSQL Docs: Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [PostgreSQL Docs: Partition Pruning](https://www.postgresql.org/docs/current/ddl-partitioning.html#DDL-PARTITION-PRUNING)

---

## References

### Partitioning Types

| Type | Use Case | Partition Key |
|------|----------|---------------|
| RANGE | Time-series, ordered | Continuous values |
| LIST | Categorical | Discrete values |
| HASH | Even distribution | Any values |

### Catalog Tables

| Table | Purpose |
|-------|---------|
| pg_partitioned_table | Partition strategy info |
| pg_inherits | Parent-child relationships |
| pg_class.relpartbound | Partition bounds |
