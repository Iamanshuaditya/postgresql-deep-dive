# Lesson 16: Transaction Isolation Levels — Preventing Anomalies

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 6 — Transactions and Locking |
| **Lesson** | 16 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 13: MVCC Fundamentals |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain the four SQL standard isolation levels
2. Identify concurrency anomalies (dirty reads, phantoms, etc.)
3. Understand how PostgreSQL implements each isolation level
4. Choose the appropriate isolation level for your application
5. Handle serialization failures in application code

### Key Terms

| Term | Definition |
|------|------------|
| **Dirty Read** | Reading uncommitted data from another transaction |
| **Non-Repeatable Read** | Same query returns different results within transaction |
| **Phantom Read** | New rows appear in repeated range queries |
| **Serialization Anomaly** | Results inconsistent with any serial execution |
| **Snapshot** | Frozen view of database state at a point in time |
| **SSI** | Serializable Snapshot Isolation (PostgreSQL's method) |

---

## Introduction

Transaction isolation determines what changes a transaction can see from other concurrent transactions. The SQL standard defines four levels:

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
|-----------------|------------|---------------------|--------------|----------------------|
| Read Uncommitted | Possible | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible (SQL) | Possible |
| Serializable | Prevented | Prevented | Prevented | Prevented |

**PostgreSQL's implementation:**
- Read Uncommitted = Read Committed (no dirty reads ever)
- Read Committed uses per-statement snapshots
- Repeatable Read uses per-transaction snapshot (also prevents phantoms!)
- Serializable uses SSI (Serializable Snapshot Isolation)

> **Key Insight**: PostgreSQL's MVCC prevents dirty reads at ALL isolation levels. You can never read uncommitted data.

---

## Conceptual Foundation

### The Anomalies Explained

#### Dirty Read (Prevented in PostgreSQL)

```
Transaction A                    Transaction B
-----------                      -----------
BEGIN;                           
UPDATE t SET x = 100 WHERE id=1;
                                 BEGIN;
                                 SELECT x FROM t WHERE id=1;
                                 -- Would see x=100 (uncommitted!)
ROLLBACK;
                                 -- But x=100 never existed!
```

PostgreSQL prevents this at all levels—you only see committed data.

#### Non-Repeatable Read

```
Transaction A                    Transaction B
-----------                      -----------
BEGIN;                           
SELECT x FROM t WHERE id=1;      
-- Returns x=50                  
                                 BEGIN;
                                 UPDATE t SET x=100 WHERE id=1;
                                 COMMIT;

SELECT x FROM t WHERE id=1;      
-- Returns x=100 (changed!)      
COMMIT;
```

- **Read Committed**: This happens (new snapshot per statement)
- **Repeatable Read**: Prevented (same snapshot for whole transaction)

#### Phantom Read

```
Transaction A                    Transaction B
-----------                      -----------
BEGIN;                           
SELECT * FROM t WHERE category='A';
-- Returns 5 rows                
                                 BEGIN;
                                 INSERT INTO t VALUES (..., 'A');
                                 COMMIT;

SELECT * FROM t WHERE category='A';
-- Returns 6 rows (phantom!)     
COMMIT;
```

- **Read Committed**: This happens
- **Repeatable Read**: Prevented in PostgreSQL (not in SQL standard!)
- **Serializable**: Always prevented

#### Serialization Anomaly (Write Skew)

```
-- Constraint: At least one doctor must be on call
-- Currently: Doctor A and Doctor B are both on call

Transaction A                    Transaction B
-----------                      -----------
BEGIN ISOLATION LEVEL            BEGIN ISOLATION LEVEL
REPEATABLE READ;                 REPEATABLE READ;

SELECT count(*) FROM oncall;     SELECT count(*) FROM oncall;
-- Returns 2                     -- Returns 2

-- Both see 2 doctors, think it's safe to remove one

UPDATE oncall SET status='off'   UPDATE oncall SET status='off'
WHERE doctor='A';                WHERE doctor='B';

COMMIT;                          COMMIT;
-- RESULT: Both doctors are off call! Constraint violated!
```

- **Repeatable Read**: This happens
- **Serializable**: One transaction aborts with serialization_failure

---

## PostgreSQL Isolation Levels

### Read Committed (Default)

```sql
BEGIN;  -- Or: BEGIN ISOLATION LEVEL READ COMMITTED;

SELECT * FROM accounts WHERE id = 1;  -- Snapshot 1
-- Some time passes, other transactions commit...
SELECT * FROM accounts WHERE id = 1;  -- Snapshot 2 (may differ!)

COMMIT;
```

**Behavior:**
- New snapshot at the start of each statement
- Sees all changes committed before statement started
- UPDATE/DELETE rechecks conditions after locking

```sql
-- Read Committed UPDATE behavior
UPDATE accounts SET balance = balance + 100 WHERE balance > 500;

-- Steps:
-- 1. Find rows with balance > 500 (using current snapshot)
-- 2. Lock each row
-- 3. Re-read row (might have changed!)
-- 4. Re-check condition: If balance still > 500, update
-- 5. If not, skip this row
```

### Repeatable Read

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT * FROM accounts WHERE id = 1;  -- Snapshot taken here!
-- Other transactions commit changes...
SELECT * FROM accounts WHERE id = 1;  -- Same snapshot, same results!

COMMIT;
```

**Behavior:**
- Single snapshot for entire transaction
- Same query always returns same results
- Cannot see commits that happened after transaction started
- May get serialization_failure errors

```sql
-- Repeatable Read serialization failure
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- ERROR: could not serialize access due to concurrent update
```

If another transaction modified the row after your snapshot, you get an error.

### Serializable

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- Complex operations that must be atomic
SELECT sum(balance) FROM accounts WHERE branch = 'NYC';
INSERT INTO audit_log VALUES ('Total checked');
-- etc.

COMMIT;
```

**Behavior:**
- Same snapshot as Repeatable Read
- Plus: Guarantees result equivalent to some serial execution
- Detects write skew and other anomalies
- More serialization failures possible (application must retry)

---

## Deep Dive: Implementation Details

### Snapshot Structure

```c
/* Simplified from src/include/utils/snapshot.h */
typedef struct SnapshotData
{
    TransactionId xmin;      /* Oldest active XID when snapshot taken */
    TransactionId xmax;      /* First unassigned XID */
    TransactionId *xip;      /* In-progress XIDs at snapshot time */
    uint32        xcnt;      /* Count of in-progress XIDs */
    
    CommandId     curcid;    /* Current command ID (for same-transaction visibility) */
    uint32        active_count; /* Refcount */
    
    /* For Serializable */
    bool          takenDuringRecovery;
    
} SnapshotData;
```

### Visibility Check

```c
/* Is this XID visible in this snapshot? */
bool
XidInMVCCSnapshot(TransactionId xid, Snapshot snapshot)
{
    /* Before xmin: definitely committed (or aborted) */
    if (TransactionIdPrecedes(xid, snapshot->xmin))
        return false;  /* Not in snapshot = visible if committed */
    
    /* >= xmax: started after snapshot */
    if (TransactionIdFollowsOrEquals(xid, snapshot->xmax))
        return true;   /* In snapshot = not visible */
    
    /* Between xmin and xmax: check if in running list */
    for (i = 0; i < snapshot->xcnt; i++)
    {
        if (TransactionIdEquals(xid, snapshot->xip[i]))
            return true;  /* Was running = not visible */
    }
    
    return false;  /* Not in list = was committed = visible */
}
```

### Serializable Snapshot Isolation (SSI)

PostgreSQL's Serializable level uses SSI, which:

1. **Tracks read/write dependencies** between transactions
2. **Detects dangerous structures** (cycles in the dependency graph)
3. **Aborts one transaction** to prevent anomaly

```
┌─────────────────────────────────────────────────────────────────┐
│  SSI Conflict Detection                                          │
│                                                                  │
│  Transaction A reads X, Transaction B writes X                  │
│  Transaction B reads Y, Transaction A writes Y                  │
│                                                                  │
│  Dependency graph:                                               │
│       A ──rw──> B                                               │
│       B ──rw──> A                                               │
│                                                                  │
│  CYCLE DETECTED! One must abort.                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Practical Examples

### Example 1: Read Committed Behavior

```sql
-- Session 1                      -- Session 2
BEGIN;                            
SELECT balance FROM accounts      
WHERE id = 1;  -- Returns 1000   
                                  BEGIN;
                                  UPDATE accounts SET balance = 500 
                                  WHERE id = 1;
                                  COMMIT;

SELECT balance FROM accounts      
WHERE id = 1;  -- Returns 500!   
COMMIT;
```

### Example 2: Repeatable Read Consistency

```sql
-- Session 1                      -- Session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts      
WHERE id = 1;  -- Returns 1000   
                                  BEGIN;
                                  UPDATE accounts SET balance = 500 
                                  WHERE id = 1;
                                  COMMIT;

SELECT balance FROM accounts      
WHERE id = 1;  -- Still 1000!    
COMMIT;
-- Now shows 500
```

### Example 3: Repeatable Read Conflict

```sql
-- Session 1                      -- Session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE accounts SET balance = balance - 100 
WHERE id = 1;
                                  BEGIN ISOLATION LEVEL REPEATABLE READ;
                                  UPDATE accounts SET balance = balance - 50 
                                  WHERE id = 1;
                                  -- WAITING for Session 1...

COMMIT;
                                  -- ERROR: could not serialize access 
                                  -- due to concurrent update
```

### Example 4: Serializable Write Skew Prevention

```sql
-- Constraint: x + y must be positive
-- Currently: x = 50, y = 50

-- Session 1                      -- Session 2
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM t;  -- x=50, y=50  
                                  BEGIN ISOLATION LEVEL SERIALIZABLE;
                                  SELECT * FROM t;  -- x=50, y=50

UPDATE t SET x = -40;            UPDATE t SET y = -40;
-- Both think: "other value is 50, 
-- so -40 + 50 = 10 > 0, OK!"

COMMIT;  -- Succeeds             COMMIT;
                                  -- ERROR: could not serialize access 
                                  -- due to read/write dependencies
```

### Example 5: Handling Serialization Failures

```python
import psycopg2
from psycopg2 import errors

def transfer_funds(from_id, to_id, amount):
    max_retries = 5
    for attempt in range(max_retries):
        try:
            conn = psycopg2.connect(...)
            conn.set_isolation_level(
                psycopg2.extensions.ISOLATION_LEVEL_SERIALIZABLE
            )
            cur = conn.cursor()
            
            cur.execute(
                "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                (amount, from_id)
            )
            cur.execute(
                "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                (amount, to_id)
            )
            
            conn.commit()
            return True  # Success!
            
        except errors.SerializationFailure:
            conn.rollback()
            if attempt == max_retries - 1:
                raise
            # Retry with exponential backoff
            time.sleep(0.1 * (2 ** attempt))
            
        finally:
            conn.close()
```

---

## Choosing Isolation Levels

### Decision Guide

| Use Case | Recommended Level |
|----------|-------------------|
| Most web applications | Read Committed (default) |
| Reports that must be consistent | Repeatable Read |
| Financial transactions | Serializable |
| Analytics queries | Repeatable Read |
| Batch processing | Read Committed with explicit locks |
| High-contention workloads | Read Committed (fewer retries) |

### Performance Considerations

| Level | Overhead | Conflicts |
|-------|----------|-----------|
| Read Committed | Lowest | Rare (row-level only) |
| Repeatable Read | Low | Some (concurrent updates) |
| Serializable | Higher | More frequent |

### When NOT to Use Serializable

1. **Very high contention**: Too many retries
2. **Long-running transactions**: Hold predicate locks too long
3. **Simple CRUD operations**: Overkill, Read Committed is fine

---

## Hands-On Exercises

### Exercise 1: Observing Non-Repeatable Reads (Basic)

```sql
-- Terminal 1
BEGIN;
SELECT * FROM test_table;
-- Note the results

-- Terminal 2
UPDATE test_table SET value = value + 100;
COMMIT;

-- Terminal 1
SELECT * FROM test_table;
-- Results changed!
COMMIT;
```

### Exercise 2: Repeatable Read Isolation (Intermediate)

```sql
-- Setup
CREATE TABLE isolation_test (id int PRIMARY KEY, val int);
INSERT INTO isolation_test VALUES (1, 100);

-- Terminal 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM isolation_test;

-- Terminal 2
UPDATE isolation_test SET val = 200 WHERE id = 1;
COMMIT;

-- Terminal 1
SELECT * FROM isolation_test;
-- Still shows 100!
COMMIT;
SELECT * FROM isolation_test;
-- Now shows 200
```

### Exercise 3: Serialization Failure (Intermediate)

```sql
-- Setup
CREATE TABLE serial_test (id int PRIMARY KEY, counter int);
INSERT INTO serial_test VALUES (1, 0);

-- Terminal 1
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT counter FROM serial_test WHERE id = 1;
UPDATE serial_test SET counter = counter + 1 WHERE id = 1;
-- Don't commit yet!

-- Terminal 2
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT counter FROM serial_test WHERE id = 1;
UPDATE serial_test SET counter = counter + 1 WHERE id = 1;
-- Waits...

-- Terminal 1
COMMIT;

-- Terminal 2
-- Should get serialization_failure error
```

### Exercise 4: Write Skew Detection (Advanced)

```sql
-- Setup: On-call doctors constraint
CREATE TABLE oncall (
    doctor text PRIMARY KEY,
    on_duty boolean
);
INSERT INTO oncall VALUES ('Alice', true), ('Bob', true);

-- Terminal 1
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM oncall WHERE on_duty;  -- 2
UPDATE oncall SET on_duty = false WHERE doctor = 'Alice';

-- Terminal 2
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM oncall WHERE on_duty;  -- 2
UPDATE oncall SET on_duty = false WHERE doctor = 'Bob';

-- Terminal 1
COMMIT;

-- Terminal 2
COMMIT;  -- Should fail!

-- Verify constraint maintained
SELECT * FROM oncall;
```

---

## Key Takeaways

1. **No dirty reads ever**: PostgreSQL's MVCC prevents reading uncommitted data.

2. **Read Committed is the default**: Good balance of consistency and performance.

3. **Repeatable Read gives snapshot consistency**: Same query, same results.

4. **PostgreSQL's Repeatable Read prevents phantoms**: Stronger than SQL standard.

5. **Serializable guarantees correctness**: But requires retry logic.

6. **Applications must handle serialization failures**: Always retry on these errors.

7. **Choose based on requirements**: Correctness needs vs. performance/retries.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/utils/time/snapmgr.c` | Snapshot management |
| `src/backend/storage/lmgr/predicate.c` | SSI predicate locks |
| `src/backend/access/transam/xact.c` | Transaction handling |

### Documentation

- [PostgreSQL Docs: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL Docs: Serializable Isolation](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE)
- [PostgreSQL Wiki: SSI](https://wiki.postgresql.org/wiki/Serializable)

---

## References

### Isolation Level Commands

```sql
-- Set for transaction
BEGIN ISOLATION LEVEL READ COMMITTED;
BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- Set for session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Check current level
SHOW transaction_isolation;
```

### Error Codes

| SQLSTATE | Name | Meaning |
|----------|------|---------|
| 40001 | serialization_failure | Retry transaction |
| 40P01 | deadlock_detected | Retry transaction |
