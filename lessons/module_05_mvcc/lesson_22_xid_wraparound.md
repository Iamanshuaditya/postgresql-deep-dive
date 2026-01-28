# Lesson 22: Transaction ID Wraparound — Preventing a Silent Disaster

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 5 — MVCC and Visibility |
| **Lesson** | 22 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 13: MVCC Fundamentals, Lesson 15: VACUUM |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain why transaction ID wraparound is dangerous
2. Understand how PostgreSQL prevents wraparound through freezing
3. Monitor wraparound risk in your databases
4. Configure anti-wraparound autovacuum properly
5. Respond to wraparound emergencies

### Key Terms

| Term | Definition |
|------|------------|
| **XID** | Transaction ID—32-bit counter (0 to ~4 billion) |
| **Wraparound** | When XID counter resets, old data becomes "future" |
| **Freezing** | Replacing XID with FrozenTransactionId |
| **relfrozenxid** | Oldest unfrozen XID in a table |
| **datfrozenxid** | Oldest unfrozen XID in a database |
| **Anti-wraparound** | Aggressive VACUUM to prevent wraparound |

---

## Introduction

PostgreSQL uses 32-bit transaction IDs (XIDs). The problem:

```
┌─────────────────────────────────────────────────────────────────┐
│  The Wraparound Problem                                          │
│                                                                  │
│  32-bit XID: ~4.3 billion possible values                       │
│                                                                  │
│  XID Space (circular):                                           │
│                    past ← | → future                            │
│                           │                                      │
│         ┌─────────────────┼─────────────────┐                   │
│         │                 │                 │                   │
│         │   2 billion     │   2 billion     │                   │
│         │   "in past"     │   "in future"   │                   │
│         │                 │                 │                   │
│         └─────────────────┼─────────────────┘                   │
│                           │                                      │
│                     Current XID                                  │
│                                                                  │
│  If we keep consuming XIDs without cleanup:                     │
│                                                                  │
│  Time 1: Current XID = 100                                      │
│          Row created with xmin=50 is in past (visible)          │
│                                                                  │
│  Time 2: Current XID = 2,000,000,100                            │
│          Row with xmin=50 is STILL in past (visible)            │
│                                                                  │
│  Time 3: Current XID = 2,100,000,000 (halfway around!)         │
│          Row with xmin=50 appears IN THE FUTURE!                │
│          ⚠️ DATA LOSS: Row becomes invisible!                   │
└─────────────────────────────────────────────────────────────────┘
```

> **Critical Warning**: Without prevention, wraparound causes committed data to vanish. This is a silent, catastrophic data loss scenario.

---

## Conceptual Foundation

### How XIDs Work

```c
/* Transaction IDs are 32-bit unsigned integers */
typedef uint32 TransactionId;

/* Special values */
#define InvalidTransactionId        0
#define BootstrapTransactionId      1  
#define FrozenTransactionId         2   /* Permanently in the past */
#define FirstNormalTransactionId    3

/* XIDs wrap around after ~4.3 billion */
/* 2^32 = 4,294,967,296 */
```

### The Modular Arithmetic Problem

XID comparison uses modular arithmetic:

```
┌─────────────────────────────────────────────────────────────────┐
│  XID Comparison (Modular)                                        │
│                                                                  │
│  Is XID A "before" XID B?                                       │
│                                                                  │
│  If (A - B) > 2^31:   A is in the PAST relative to B           │
│  If (A - B) < 2^31:   A is in the FUTURE relative to B         │
│                                                                  │
│  Example:                                                        │
│  Current XID = 100                                              │
│  XID 50:    100 - 50 = 50 < 2^31     → 50 is in past ✓         │
│  XID 2^31+100: This would be in future                         │
│                                                                  │
│  AFTER 2 billion more transactions:                             │
│  Current XID = 2,100,000,100                                    │
│  XID 50:    2100000100 - 50 = 2100000050 < 2^31 → past ✓       │
│                                                                  │
│  AFTER 2+ billion MORE:                                         │
│  Current XID = 4,200,000,100 (wrapped to ~100,000,100)         │
│  XID 50:    100000100 - 50 = 100000050 < 2^31 → PAST?          │
│  Wait, 50 was created 4B transactions ago, but looks recent!  │
└─────────────────────────────────────────────────────────────────┘
```

### The Solution: Freezing

Instead of keeping the original XID forever, we replace it with a special "frozen" value:

```
Before Freezing:
┌─────────────────────────────────────────────────────────────────┐
│ Tuple: xmin = 12345678 (some old transaction)                   │
│        This XID could become "future" if we wrap around        │
└─────────────────────────────────────────────────────────────────┘

After Freezing:
┌─────────────────────────────────────────────────────────────────┐
│ Tuple: xmin = FrozenTransactionId (2)                          │
│        OR: t_infomask has HEAP_XMIN_FROZEN bit set             │
│        This tuple is ALWAYS in the past, visible to all        │
└─────────────────────────────────────────────────────────────────┘
```

---

## How Freezing Works

### VACUUM Performs Freezing

VACUUM checks each tuple's age and freezes old ones:

```c
/* Simplified freezing logic */
void heap_freeze_tuple(HeapTupleHeader tuple)
{
    TransactionId xid = tuple->t_xmin;
    
    /* If old enough, freeze it */
    if (TransactionIdPrecedes(xid, FreezeLimit))
    {
        /* Option 1: Set to FrozenTransactionId */
        tuple->t_xmin = FrozenTransactionId;
        
        /* Option 2 (modern): Set frozen bit in infomask */
        tuple->t_infomask |= HEAP_XMIN_FROZEN;
    }
}
```

### Freeze Thresholds

| Parameter | Default | Description |
|-----------|---------|-------------|
| `vacuum_freeze_min_age` | 50 million | Minimum age to freeze |
| `vacuum_freeze_table_age` | 150 million | Trigger aggressive vacuum |
| `autovacuum_freeze_max_age` | 200 million | Force anti-wraparound |

```
┌─────────────────────────────────────────────────────────────────┐
│  Freeze Timeline                                                 │
│                                                                  │
│  XID Age:                                                        │
│  0           50M          150M         200M         2B           │
│  │───────────│────────────│────────────│────────────│──────→    │
│  │           │            │            │            │           │
│  │           │            │            │            Danger zone │
│  │           │            │            │                        │
│  │           │            │            Anti-wraparound          │
│  │           │            │            VACUUM forced            │
│  │           │            │                                     │
│  │           │            Aggressive whole-table                │
│  │           │            VACUUM triggered                      │
│  │           │                                                  │
│  │           Tuples eligible for                                │
│  │           freezing during normal VACUUM                      │
│  │                                                              │
│  New tuples                                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tracking Freeze Progress

### Per-Table: relfrozenxid

Each table tracks its oldest unfrozen XID:

```sql
SELECT 
    relname,
    age(relfrozenxid) AS xid_age,
    relfrozenxid
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 10;
```

### Per-Database: datfrozenxid

```sql
SELECT 
    datname,
    age(datfrozenxid) AS xid_age,
    datfrozenxid
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

### Current XID

```sql
SELECT txid_current();
-- Returns current transaction's XID

SELECT age(datfrozenxid) FROM pg_database WHERE datname = current_database();
-- How close to danger zone?
```

---

## Anti-Wraparound Autovacuum

### How It Triggers

```
┌─────────────────────────────────────────────────────────────────┐
│  Anti-Wraparound Trigger Conditions                              │
│                                                                  │
│  Normal autovacuum:                                              │
│    Triggers based on dead tuple count                           │
│    Respects cost limits, can be slow                            │
│                                                                  │
│  Anti-wraparound autovacuum:                                    │
│    Triggers when: age(relfrozenxid) > autovacuum_freeze_max_age │
│    IGNORES cost limits!                                         │
│    Cannot be canceled (except by administrator)                 │
│    MUST complete to prevent data loss                           │
└─────────────────────────────────────────────────────────────────┘
```

### Identifying Anti-Wraparound VACUUMs

```sql
-- Check for forced vacuum
SELECT 
    pid,
    datname,
    relid::regclass AS table_name,
    phase,
    heap_blks_scanned,
    heap_blks_total
FROM pg_stat_progress_vacuum
WHERE relid IN (
    SELECT oid FROM pg_class 
    WHERE age(relfrozenxid) > current_setting('autovacuum_freeze_max_age')::int
);

-- Or check pg_stat_activity
SELECT * FROM pg_stat_activity 
WHERE query LIKE '%wraparound%' 
   OR query LIKE '%to prevent wraparound%';
```

### Emergency: Near Wraparound

If XID age approaches 2 billion:

```sql
-- Check emergency status
SELECT 
    datname,
    age(datfrozenxid) as xid_age,
    2000000000 - age(datfrozenxid) as transactions_left
FROM pg_database
ORDER BY xid_age DESC;
```

At ~1 billion XIDs remaining, PostgreSQL issues warnings. At shutdown threshold (~40 million remaining), it refuses to start new transactions!

```
WARNING: database "mydb" must be vacuumed within 177009986 transactions
HINT: To avoid a database shutdown, execute a database-wide VACUUM in that database.
```

---

## Preventing Wraparound Issues

### Proper Configuration

```sql
-- Aggressive settings for high-transaction systems
ALTER SYSTEM SET vacuum_freeze_min_age = 10000000;  -- 10M
ALTER SYSTEM SET vacuum_freeze_table_age = 50000000;  -- 50M
ALTER SYSTEM SET autovacuum_freeze_max_age = 100000000;  -- 100M

-- More frequent autovacuum checks
ALTER SYSTEM SET autovacuum_naptime = '30s';

-- Don't let autovacuum be too gentle
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '0ms';  -- For anti-wraparound

SELECT pg_reload_conf();
```

### Per-Table Settings

```sql
-- For high-churn table
ALTER TABLE frequently_updated SET (
    autovacuum_freeze_min_age = 5000000,
    autovacuum_freeze_table_age = 20000000,
    autovacuum_freeze_max_age = 50000000
);
```

### Monitoring Script

```sql
-- Tables approaching freeze danger
SELECT 
    c.relname,
    c.relnamespace::regnamespace AS schema,
    age(c.relfrozenxid) AS xid_age,
    pg_size_pretty(pg_relation_size(c.oid)) AS size,
    CASE 
        WHEN age(c.relfrozenxid) > 1500000000 THEN 'CRITICAL'
        WHEN age(c.relfrozenxid) > 1000000000 THEN 'WARNING'
        WHEN age(c.relfrozenxid) > 500000000 THEN 'ELEVATED'
        ELSE 'OK'
    END AS status
FROM pg_class c
WHERE c.relkind = 'r'
  AND c.relnamespace NOT IN (
      SELECT oid FROM pg_namespace WHERE nspname ~ '^pg_'
  )
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;
```

---

## Practical Examples

### Example 1: Checking XID Age

```sql
-- Current transaction ID
SELECT txid_current();

-- Database XID ages
SELECT datname, age(datfrozenxid), datfrozenxid
FROM pg_database;

-- Table XID ages
SELECT 
    n.nspname || '.' || c.relname AS table_name,
    age(c.relfrozenxid) AS xid_age,
    c.relfrozenxid
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
ORDER BY age(c.relfrozenxid) DESC;
```

### Example 2: Manually Freezing

```sql
-- Aggressive vacuum with low freeze threshold
VACUUM (FREEZE, VERBOSE) my_table;

-- This will:
-- 1. Scan entire table
-- 2. Freeze all tuples old enough
-- 3. Update relfrozenxid
```

### Example 3: Monitoring Freeze Progress

```sql
-- Before vacuum
SELECT relname, age(relfrozenxid) FROM pg_class WHERE relname = 'my_table';

-- Run vacuum
VACUUM (FREEZE) my_table;

-- After vacuum
SELECT relname, age(relfrozenxid) FROM pg_class WHERE relname = 'my_table';
-- Age should be close to 0!
```

---

## Emergency Procedures

### When XIDs Are Running Low

```bash
# Step 1: Stop non-essential activity
# Reduce XID consumption

# Step 2: Run aggressive vacuum on worst tables
psql -c "VACUUM (FREEZE) schema.problem_table;"

# Step 3: If in single-user mode needed
pg_ctl stop
postgres --single -D $PGDATA mydb
# Then run VACUUM FREEZE

# Step 4: Monitor progress
psql -c "SELECT datname, age(datfrozenxid) FROM pg_database;"
```

### Preventing Cascade Failures

```sql
-- Priority order for freezing
WITH table_ages AS (
    SELECT 
        c.oid,
        c.relname,
        n.nspname,
        age(c.relfrozenxid) AS xid_age,
        pg_relation_size(c.oid) AS size
    FROM pg_class c
    JOIN pg_namespace n ON c.relnamespace = n.oid
    WHERE c.relkind = 'r'
)
SELECT 
    nspname || '.' || relname AS table_name,
    xid_age,
    pg_size_pretty(size) AS size,
    -- Smaller tables first for quick wins
    xid_age::float / nullif(size, 0) AS urgency_score
FROM table_ages
WHERE nspname NOT LIKE 'pg_%'
ORDER BY urgency_score DESC NULLS LAST
LIMIT 20;
```

---

## Hands-On Exercises

### Exercise 1: XID Age Inspection (Basic)

```sql
-- View database ages
SELECT datname, age(datfrozenxid) AS xid_age 
FROM pg_database ORDER BY xid_age DESC;

-- View table ages
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relkind = 'r'
ORDER BY xid_age DESC LIMIT 10;
```

### Exercise 2: Watching Freeze Progress (Intermediate)

```sql
-- Create test table
CREATE TABLE freeze_test AS SELECT generate_series(1, 100000);

-- Check initial age
SELECT relname, age(relfrozenxid) FROM pg_class 
WHERE relname = 'freeze_test';

-- Normal vacuum
VACUUM freeze_test;
SELECT relname, age(relfrozenxid) FROM pg_class 
WHERE relname = 'freeze_test';

-- Freeze vacuum
VACUUM (FREEZE) freeze_test;
SELECT relname, age(relfrozenxid) FROM pg_class 
WHERE relname = 'freeze_test';
-- Age should now be very low
```

### Exercise 3: Simulating Age Advancement (Advanced)

```sql
-- Generate transactions to age the table
-- (Run this in a loop or use pgbench)
DO $$
BEGIN
    FOR i IN 1..1000 LOOP
        INSERT INTO freeze_test VALUES (i + 100000);
        COMMIT;
    END LOOP;
END $$;

-- Check age progression
SELECT relname, age(relfrozenxid) FROM pg_class 
WHERE relname = 'freeze_test';
```

### Exercise 4: Monitoring Dashboard Query (Advanced)

```sql
-- Comprehensive wraparound monitoring
SELECT 
    'Database' AS level,
    datname AS name,
    age(datfrozenxid) AS xid_age,
    round(100.0 * age(datfrozenxid) / 2000000000, 2) AS pct_to_wraparound,
    NULL AS size
FROM pg_database

UNION ALL

SELECT 
    'Table' AS level,
    n.nspname || '.' || c.relname AS name,
    age(c.relfrozenxid) AS xid_age,
    round(100.0 * age(c.relfrozenxid) / 2000000000, 2) AS pct_to_wraparound,
    pg_size_pretty(pg_relation_size(c.oid)) AS size
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
  AND n.nspname NOT LIKE 'pg_%'
  AND age(c.relfrozenxid) > 100000000  -- Only show aged tables

ORDER BY xid_age DESC;
```

---

## Key Takeaways

1. **XIDs are finite**: 32-bit means ~4 billion, wraparound after ~2 billion.

2. **Wraparound = data loss**: Old data appears "in future," becomes invisible.

3. **Freezing prevents wraparound**: Replaces XIDs with permanent "frozen" marker.

4. **VACUUM performs freezing**: Regular VACUUM when old enough, aggressive when forced.

5. **Monitor XID ages**: Track relfrozenxid and datfrozenxid regularly.

6. **Anti-wraparound can't be stopped**: It ignores cost limits to prevent disaster.

7. **Early intervention is key**: Don't let tables get close to the threshold.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/access/heap/heapam.c` | Freeze tuple logic |
| `src/backend/commands/vacuum.c` | VACUUM freeze handling |
| `src/backend/access/transam/varsup.c` | XID allocation |
| `src/include/access/transam.h` | XID definitions |

### Documentation

- [PostgreSQL Docs: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND)
- [PostgreSQL Docs: Preventing Transaction ID Wraparound Failures](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND)

---

## References

### Critical Thresholds

| Threshold | XIDs Remaining | Action |
|-----------|----------------|--------|
| autovacuum_freeze_max_age | ~1.8B | Anti-wraparound starts |
| Warning threshold | ~1B | LOG warnings issued |
| Shutdown threshold | ~40M | New transactions blocked |
| Wraparound | 0 | Data loss (never reach!) |

### Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `vacuum_freeze_min_age` | 50M | Min age to freeze |
| `vacuum_freeze_table_age` | 150M | Trigger aggressive |
| `autovacuum_freeze_max_age` | 200M | Force anti-wraparound |
| `vacuum_multixact_freeze_min_age` | 5M | MultiXact freeze age |
| `autovacuum_multixact_freeze_max_age` | 400M | MultiXact force |
