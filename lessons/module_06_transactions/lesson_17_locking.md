# Lesson 17: Locking and Concurrency Control — Managing Concurrent Access

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 6 — Transactions and Locking |
| **Lesson** | 17 of 66 |
| **Estimated Time** | 90-120 minutes |
| **Prerequisites** | Lesson 16: Transaction Isolation Levels |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain PostgreSQL's multi-granularity locking system
2. Describe the different table-level and row-level lock modes
3. Identify and resolve deadlocks
4. Use advisory locks for application-level locking
5. Monitor and troubleshoot locking issues

### Key Terms

| Term | Definition |
|------|------------|
| **Table Lock** | Lock on entire relation (8 different modes) |
| **Row Lock** | Lock on specific tuple (4 modes) |
| **Heavyweight Lock** | Traditional lock managed by lock manager |
| **Lightweight Lock (LWLock)** | Fast internal locks for shared memory |
| **Spin Lock** | Very short-term CPU-level locks |
| **Advisory Lock** | Application-defined locks |
| **Deadlock** | Circular wait between transactions |

---

## Introduction

While MVCC handles most concurrency through snapshots, PostgreSQL still needs locks for:

1. **DDL operations**: ALTER TABLE must block concurrent access
2. **Explicit locking**: SELECT FOR UPDATE, LOCK TABLE
3. **Foreign keys**: Checking referential integrity
4. **Indexes**: Protecting structure during modifications
5. **Internal structures**: Buffer headers, clog, etc.

PostgreSQL uses a sophisticated locking system with multiple levels:

```
┌─────────────────────────────────────────────────────────────────┐
│  Lock Hierarchy                                                  │
│                                                                  │
│  Spin Locks (fastest, shortest-duration)                        │
│    └── Protect: CPU-level critical sections                    │
│                                                                  │
│  Lightweight Locks (LWLocks)                                    │
│    └── Protect: Shared memory structures                        │
│                                                                  │
│  Heavyweight Locks (Regular Locks)                              │
│    ├── Table Locks: 8 modes, coarse-grained                    │
│    ├── Row Locks: 4 modes, stored in tuple headers             │
│    └── Other: Page, transaction, relation, etc.                │
│                                                                  │
│  Advisory Locks (application-controlled)                        │
│    └── Protect: Whatever your application defines              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Table-Level Locks

### The Eight Lock Modes

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Table Lock Compatibility Matrix                                              │
│                                                                               │
│                    Requested Lock Mode                                        │
│  Held Lock    │ ACCESS │ ROW   │ ROW     │ SHARE │ SHARE │ SHARE │ EXCL  │ ACCESS │
│  Mode         │ SHARE  │ SHARE │ EXCLUS  │ UPDT  │       │ ROW   │       │ EXCLUS │
│               │        │       │         │ EXCLUS│       │ EXCLUS│       │        │
│  ─────────────┼────────┼───────┼─────────┼───────┼───────┼───────┼───────┼────────│
│  ACCESS SHARE │   ✓    │   ✓   │    ✓    │   ✓   │   ✓   │   ✓   │   ✓   │   ✗    │
│  ROW SHARE    │   ✓    │   ✓   │    ✓    │   ✓   │   ✓   │   ✓   │   ✗   │   ✗    │
│  ROW EXCLUS   │   ✓    │   ✓   │    ✓    │   ✓   │   ✗   │   ✗   │   ✗   │   ✗    │
│  SHARE UPDT EX│   ✓    │   ✓   │    ✓    │   ✗   │   ✗   │   ✗   │   ✗   │   ✗    │
│  SHARE        │   ✓    │   ✓   │    ✗    │   ✗   │   ✓   │   ✗   │   ✗   │   ✗    │
│  SHARE ROW EX │   ✓    │   ✓   │    ✗    │   ✗   │   ✗   │   ✗   │   ✗   │   ✗    │
│  EXCLUSIVE    │   ✓    │   ✗   │    ✗    │   ✗   │   ✗   │   ✗   │   ✗   │   ✗    │
│  ACCESS EXCLUS│   ✗    │   ✗   │    ✗    │   ✗   │   ✗   │   ✗   │   ✗   │   ✗    │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Lock Modes and SQL Commands

| Lock Mode | Acquired By | Conflicts With |
|-----------|-------------|----------------|
| ACCESS SHARE | SELECT | ACCESS EXCLUSIVE |
| ROW SHARE | SELECT FOR UPDATE/SHARE | EXCLUSIVE, ACCESS EXCLUSIVE |
| ROW EXCLUSIVE | INSERT, UPDATE, DELETE | SHARE, SHARE ROW EXCL, EXCL, ACCESS EXCL |
| SHARE UPDATE EXCLUSIVE | VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY | SHARE UPDATE EXCL and above |
| SHARE | CREATE INDEX (not concurrent) | ROW EXCLUSIVE and above |
| SHARE ROW EXCLUSIVE | CREATE TRIGGER, ALTER TABLE (some) | ROW EXCLUSIVE and above |
| EXCLUSIVE | REFRESH MATERIALIZED VIEW CONCURRENTLY | ROW SHARE and above |
| ACCESS EXCLUSIVE | DROP, TRUNCATE, ALTER TABLE (most), VACUUM FULL | ALL locks |

### Explicit Table Locking

```sql
-- Lock table for exclusive access
LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;

-- Lock for read-only access
LOCK TABLE accounts IN ACCESS SHARE MODE;

-- Wait up to 5 seconds for lock
SET lock_timeout = '5s';
LOCK TABLE accounts IN EXCLUSIVE MODE;
```

---

## Row-Level Locks

### Row Lock Modes

| Mode | Acquired By | Conflicts With |
|------|-------------|----------------|
| FOR KEY SHARE | Foreign key checks | FOR UPDATE |
| FOR SHARE | SELECT FOR SHARE | FOR UPDATE, FOR NO KEY UPDATE |
| FOR NO KEY UPDATE | UPDATE (non-key columns) | FOR SHARE, FOR NO KEY UPDATE, FOR UPDATE |
| FOR UPDATE | UPDATE (key columns), DELETE | All row locks |

### Row Lock Storage

Row locks are stored in the tuple header (xmax field), not in the lock table:

```
┌─────────────────────────────────────────────────────────────────┐
│  Tuple Header                                                    │
│                                                                  │
│  xmax field when row is locked (not deleted):                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ xmax = Locking transaction's XID                            │ │
│  │ t_infomask flags indicate lock type:                        │ │
│  │   HEAP_XMAX_LOCK_ONLY  – Lock, not delete                   │ │
│  │   HEAP_XMAX_KEYSHR_LOCK – FOR KEY SHARE                     │ │
│  │   HEAP_XMAX_SHR_LOCK   – FOR SHARE                          │ │
│  │   HEAP_XMAX_EXCL_LOCK  – FOR UPDATE/FOR NO KEY UPDATE       │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Transaction Row Locks

When multiple transactions hold compatible locks on the same row:

```sql
-- Transaction 1
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- Transaction 2 (can also get FOR SHARE)  
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- Both hold FOR SHARE lock
-- PostgreSQL uses MultiXact to track both
```

### Using Row Locks

```sql
-- Lock and read for update
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Row is locked exclusively until COMMIT/ROLLBACK
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Skip locked rows (don't wait)
SELECT * FROM jobs WHERE status = 'pending' 
FOR UPDATE SKIP LOCKED
LIMIT 1;

-- Don't wait at all (error if locked)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
```

---

## Deadlock Detection

### What is a Deadlock?

```
┌─────────────────────────────────────────────────────────────────┐
│  Deadlock Example                                                │
│                                                                  │
│  Transaction A                    Transaction B                  │
│  ─────────────                    ─────────────                  │
│  UPDATE t1 WHERE id=1;            UPDATE t2 WHERE id=1;         │
│  (Holds lock on t1.id=1)          (Holds lock on t2.id=1)       │
│                                                                  │
│  UPDATE t2 WHERE id=1;            UPDATE t1 WHERE id=1;         │
│  (Waits for B's lock)              (Waits for A's lock)         │
│           │                               │                      │
│           └───────── DEADLOCK! ───────────┘                     │
│                                                                  │
│  Neither can proceed. PostgreSQL detects this and aborts one.  │
└─────────────────────────────────────────────────────────────────┘
```

### How Deadlock Detection Works

```c
/* PostgreSQL runs deadlock detection when a transaction waits */

void CheckDeadLock(void)
{
    /* Only check after deadlock_timeout (default 1s) */
    
    /* Build wait-for graph:
     * Node = Transaction
     * Edge = "waiting for lock held by"
     */
     
    /* Look for cycles in the graph */
    if (DeadLockCheck())
    {
        /* Found a cycle = deadlock! */
        /* Abort this transaction (the one that triggered check) */
        ereport(ERROR,
            (errcode(ERRCODE_T_R_DEADLOCK_DETECTED),
             errmsg("deadlock detected")));
    }
}
```

### Deadlock Configuration

```sql
-- Time to wait before checking for deadlock
SHOW deadlock_timeout;  -- Default: 1s

-- Log deadlock information
SET log_lock_waits = on;

-- Lock wait timeout (give up waiting)
SET lock_timeout = '10s';

-- Statement timeout (give up entire statement)
SET statement_timeout = '30s';
```

### Avoiding Deadlocks

1. **Consistent lock ordering**: Always lock resources in the same order

```sql
-- BAD: Different order
-- Session 1: lock A, then B
-- Session 2: lock B, then A

-- GOOD: Same order
-- Session 1: lock A, then B
-- Session 2: lock A, then B
```

2. **Short transactions**: Hold locks for less time

3. **Use NOWAIT or lock_timeout**: Fail fast instead of waiting

4. **SKIP LOCKED for queues**: Process available work without waiting

---

## Advisory Locks

### What are Advisory Locks?

Application-defined locks that PostgreSQL tracks but doesn't enforce:

```sql
-- Lock a "resource" identified by a bigint
SELECT pg_advisory_lock(12345);
-- ... do work ...
SELECT pg_advisory_unlock(12345);

-- Lock identified by two integers
SELECT pg_advisory_lock(1, 2);  -- (class_id, object_id)
```

### Advisory Lock Types

| Function | Type | Behavior |
|----------|------|----------|
| `pg_advisory_lock(key)` | Session, Exclusive | Blocks if held |
| `pg_advisory_lock_shared(key)` | Session, Shared | Multiple readers OK |
| `pg_try_advisory_lock(key)` | Session, Exclusive | Returns false if held |
| `pg_advisory_xact_lock(key)` | Transaction, Exclusive | Released at commit |
| `pg_advisory_unlock(key)` | Unlock | Must match lock type |
| `pg_advisory_unlock_all()` | Unlock | Release all session locks |

### Advisory Lock Use Cases

```sql
-- Prevent concurrent execution of a batch job
CREATE FUNCTION run_daily_batch() RETURNS void AS $$
BEGIN
    -- Try to get lock (non-blocking)
    IF NOT pg_try_advisory_lock(hashtext('daily_batch')) THEN
        RAISE NOTICE 'Batch already running';
        RETURN;
    END IF;
    
    -- Do batch work...
    PERFORM process_daily_data();
    
    -- Release lock
    PERFORM pg_advisory_unlock(hashtext('daily_batch'));
END;
$$ LANGUAGE plpgsql;

-- Coordinate access to external resource
SELECT pg_advisory_lock(hashtext('printer_' || printer_id));
-- Send to printer...
SELECT pg_advisory_unlock(hashtext('printer_' || printer_id));
```

---

## Monitoring Locks

### Viewing Current Locks

```sql
-- All current locks
SELECT 
    l.locktype,
    l.relation::regclass,
    l.mode,
    l.granted,
    l.pid,
    a.usename,
    a.query,
    a.state
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation IS NOT NULL
ORDER BY l.relation;

-- Find blocking locks
SELECT 
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocked.query AS blocked_query,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks 
    ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.relation = blocking_locks.relation
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_stat_activity blocking ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.granted
  AND blocking_locks.granted;
```

### Built-in Blocking Query View (PG 9.6+)

```sql
-- pg_blocking_pids function
SELECT 
    pid,
    usename,
    pg_blocking_pids(pid) AS blocked_by,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Lock Wait Monitoring

```sql
-- Enable lock wait logging
SET log_lock_waits = on;

-- Queries waiting for locks
SELECT 
    pid,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

---

## Practical Examples

### Example 1: Table Lock Behavior

```sql
-- Session 1
BEGIN;
LOCK TABLE accounts IN SHARE MODE;
-- Allows: SELECT
-- Blocks: INSERT, UPDATE, DELETE

-- Session 2 (blocked)
INSERT INTO accounts VALUES (...);  -- Waits!

-- Session 1
COMMIT;  -- Session 2 proceeds
```

### Example 2: Row Lock for Updates

```sql
-- Session 1
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Row is locked

-- Session 2
BEGIN;
SELECT * FROM accounts WHERE id = 1;  -- Works (MVCC)
UPDATE accounts SET balance = 500 WHERE id = 1;  -- Waits!

-- Session 1
UPDATE accounts SET balance = 1000 WHERE id = 1;
COMMIT;

-- Session 2 proceeds (may get serialization error in RR/SERIALIZABLE)
```

### Example 3: SKIP LOCKED for Job Queue

```sql
-- Create job queue
CREATE TABLE jobs (
    id serial PRIMARY KEY,
    status text DEFAULT 'pending',
    data jsonb,
    worker_pid int
);

-- Worker claims a job
BEGIN;
UPDATE jobs SET 
    status = 'processing',
    worker_pid = pg_backend_pid()
WHERE id = (
    SELECT id FROM jobs 
    WHERE status = 'pending'
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
RETURNING *;

-- Process job...
UPDATE jobs SET status = 'completed' WHERE id = ...;
COMMIT;
```

### Example 4: Avoiding Deadlock

```sql
-- BAD: Risk of deadlock
-- Session 1: Updates A then B
-- Session 2: Updates B then A

-- GOOD: Always same order (sorted by ID)
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

---

## Hands-On Exercises

### Exercise 1: Observing Table Locks (Basic)

```sql
-- Terminal 1
BEGIN;
LOCK TABLE test_locks IN SHARE MODE;

-- Terminal 2
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks WHERE relation = 'test_locks'::regclass;

-- Try INSERT (should wait)
INSERT INTO test_locks VALUES (1);

-- Terminal 1
COMMIT;
-- Terminal 2's INSERT proceeds
```

### Exercise 2: Row Locking (Intermediate)

```sql
-- Setup
CREATE TABLE row_lock_test (id int PRIMARY KEY, val int);
INSERT INTO row_lock_test VALUES (1, 100);

-- Terminal 1
BEGIN;
SELECT * FROM row_lock_test WHERE id = 1 FOR UPDATE;

-- Terminal 2
SELECT * FROM row_lock_test WHERE id = 1;  -- Works!
UPDATE row_lock_test SET val = 200 WHERE id = 1;  -- Waits!

-- Terminal 1
COMMIT;
-- What happens in Terminal 2?
```

### Exercise 3: Deadlock Detection (Intermediate)

```sql
-- Setup
CREATE TABLE dl_test (id int PRIMARY KEY, val int);
INSERT INTO dl_test VALUES (1, 100), (2, 200);

-- Terminal 1
BEGIN;
UPDATE dl_test SET val = val + 1 WHERE id = 1;

-- Terminal 2
BEGIN;
UPDATE dl_test SET val = val + 1 WHERE id = 2;

-- Terminal 1
UPDATE dl_test SET val = val + 1 WHERE id = 2;  -- Waits

-- Terminal 2
UPDATE dl_test SET val = val + 1 WHERE id = 1;  -- DEADLOCK!

-- One session gets: ERROR: deadlock detected
```

### Exercise 4: Advisory Locks (Advanced)

```sql
-- Terminal 1
SELECT pg_advisory_lock(1);
SELECT 'holding lock';

-- Terminal 2
SELECT pg_try_advisory_lock(1);  -- Returns false
SELECT pg_advisory_lock(1);  -- Blocks!

-- Terminal 1
SELECT pg_advisory_unlock(1);
-- Terminal 2 proceeds
```

---

## Key Takeaways

1. **MVCC reduces locking**: Readers don't block writers, but locks still matter.

2. **Eight table lock modes**: From ACCESS SHARE (SELECT) to ACCESS EXCLUSIVE (DDL).

3. **Row locks are stored in tuples**: Not in the lock table, scales better.

4. **Deadlock detection is automatic**: PostgreSQL aborts one transaction.

5. **SKIP LOCKED enables queues**: Workers grab available work without blocking.

6. **Advisory locks are flexible**: Application-defined, PostgreSQL-tracked.

7. **Monitor with pg_locks**: Identify blocking and waiting processes.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/storage/lmgr/lock.c` | Lock manager |
| `src/backend/storage/lmgr/deadlock.c` | Deadlock detection |
| `src/backend/storage/lmgr/lwlock.c` | Lightweight locks |
| `src/include/storage/lock.h` | Lock definitions |

### Documentation

- [PostgreSQL Docs: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [PostgreSQL Docs: Advisory Locks](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)

---

## References

### Lock Mode Comparison

| Scenario | Lock Mode |
|----------|-----------|
| Normal SELECT | ACCESS SHARE |
| SELECT FOR UPDATE | ROW SHARE + row locks |
| INSERT/UPDATE/DELETE | ROW EXCLUSIVE + row locks |
| CREATE INDEX | SHARE |
| CREATE INDEX CONCURRENTLY | SHARE UPDATE EXCLUSIVE |
| VACUUM | SHARE UPDATE EXCLUSIVE |
| ALTER TABLE (add column) | ACCESS EXCLUSIVE |
| DROP TABLE | ACCESS EXCLUSIVE |

### Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `deadlock_timeout` | 1s | Wait before deadlock check |
| `lock_timeout` | 0 (disabled) | Max wait for lock |
| `log_lock_waits` | off | Log long lock waits |
| `max_locks_per_transaction` | 64 | Lock table sizing |
