# Lesson 13: MVCC Fundamentals — Multi-Version Concurrency Control

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 5 — MVCC and Visibility |
| **Lesson** | 13 of 66 |
| **Estimated Time** | 90-120 minutes |
| **Prerequisites** | Lesson 8: Heap Storage and Page Layout |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain what MVCC is and why PostgreSQL uses it
2. Describe how xmin and xmax track tuple versions
3. Understand how snapshots determine tuple visibility
4. Trace visibility decisions for different transaction scenarios
5. Explain the relationship between MVCC and isolation levels

### Key Terms

| Term | Definition |
|------|------------|
| **MVCC** | Multi-Version Concurrency Control: keeping multiple row versions |
| **xmin** | Transaction ID that created (inserted) this tuple version |
| **xmax** | Transaction ID that deleted/updated this tuple (0 if still valid) |
| **Snapshot** | A "view" of which transactions are considered committed |
| **Visibility** | Whether a tuple is visible to a particular transaction |
| **Transaction ID (XID)** | Unique identifier for each transaction |
| **Commit Log (clog)** | Tracks commit status of every transaction |

---

## Introduction

PostgreSQL uses **Multi-Version Concurrency Control (MVCC)** to enable high concurrency without heavy locking. The key principle:

> **Readers don't block writers, and writers don't block readers.**

Instead of locking rows during reads (which would block writers), PostgreSQL:

1. Keeps **multiple versions** of each row
2. Decides which version each transaction should see
3. Uses **snapshots** to ensure consistency
4. Cleans up old versions eventually (VACUUM)

This approach enables maximum concurrency while maintaining the ACID properties.

### Why MVCC Matters

Consider two concurrent transactions:

```sql
-- Transaction A (starts first)           -- Transaction B
BEGIN;                                     BEGIN;
SELECT balance FROM accounts               
WHERE id = 1;  -- Returns 1000            
                                           UPDATE accounts SET balance = 500 
                                           WHERE id = 1;
                                           COMMIT;

SELECT balance FROM accounts 
WHERE id = 1;  -- Still returns 1000!
COMMIT;
```

Without MVCC:
- Transaction A would need to lock the row, blocking B
- OR Transaction A would see B's uncommitted change (dirty read)
- OR Transaction A's second SELECT would see 500 (non-repeatable read)

With MVCC:
- Both transactions run without blocking each other
- Each transaction sees a consistent view based on its snapshot
- The old row version (1000) is kept until A commits

---

## Conceptual Foundation

### How Updates Really Work

In PostgreSQL, **update is delete + insert**:

```
Before UPDATE:                  After UPDATE "SET balance = 500":

Tuple Version 1:                Tuple Version 1 (now "dead"):
┌─────────────────────────┐     ┌─────────────────────────┐
│ xmin = 100 (committed)  │     │ xmin = 100 (committed)  │
│ xmax = 0                │     │ xmax = 200 (B's xid)    │ ← Marked deleted
│ data: id=1, balance=1000│     │ data: id=1, balance=1000│
└─────────────────────────┘     └─────────────────────────┘

                                Tuple Version 2 (new):
                                ┌─────────────────────────┐
                                │ xmin = 200 (B's xid)    │ ← Created by B
                                │ xmax = 0                │
                                │ data: id=1, balance=500 │
                                └─────────────────────────┘
```

**Key observations:**
- The old tuple isn't physically deleted—it's marked with xmax
- Both versions coexist until VACUUM cleans up
- Different transactions see different versions

### The Tuple Header Fields

```c
/* Simplified from src/include/access/htup_details.h */
typedef struct HeapTupleFields
{
    TransactionId t_xmin;       /* Inserting XID */
    TransactionId t_xmax;       /* Deleting/updating XID, or 0 */
    CommandId     t_cid;        /* Command ID within transaction */
    ItemPointerData t_ctid;     /* Current TID (or link to newer version) */
} HeapTupleFields;
```

### Viewing xmin and xmax

```sql
CREATE TABLE accounts (id int, balance int);
INSERT INTO accounts VALUES (1, 1000);

-- View system columns
SELECT ctid, xmin, xmax, * FROM accounts;
```

```
 ctid  | xmin | xmax | id | balance
-------+------+------+----+---------
 (0,1) | 1234 |    0 |  1 |    1000
```

After an update:

```sql
UPDATE accounts SET balance = 500 WHERE id = 1;
SELECT ctid, xmin, xmax, * FROM accounts;
```

```
 ctid  | xmin | xmax | id | balance
-------+------+------+----+---------
 (0,2) | 1235 |    0 |  1 |     500
```

The ctid changed (new tuple at different location), and xmin is the new transaction.

### Transaction ID Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│  Transaction Lifecycle                                           │
│                                                                  │
│  1. BEGIN → AssignTransactionId() → Get XID (e.g., 12345)       │
│                                                                  │
│  2. Execute statements:                                          │
│     - INSERT: new tuple with xmin = 12345                       │
│     - UPDATE: old tuple gets xmax = 12345, new tuple xmin = 12345│
│     - DELETE: tuple gets xmax = 12345                           │
│                                                                  │
│  3. COMMIT → Write commit record to clog                        │
│     Status: 12345 → COMMITTED                                   │
│                                                                  │
│  OR:                                                            │
│                                                                  │
│  3. ROLLBACK → Write abort record to clog                       │
│     Status: 12345 → ABORTED                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Snapshots and Visibility

### What is a Snapshot?

A snapshot is a "point-in-time view" of which transactions are considered committed. It contains:

```c
/* Simplified from src/include/utils/snapshot.h */
typedef struct SnapshotData
{
    TransactionId xmin;           /* Oldest active (uncommitted) XID */
    TransactionId xmax;           /* First XID not yet assigned */
    TransactionId *xip;           /* Array of active XIDs */
    uint32        xcnt;           /* Count of active XIDs */
    CommandId     curcid;         /* Current command's CID */
} SnapshotData;
```

**Key fields:**
- **xmin**: All XIDs below this are committed (or aborted)
- **xmax**: All XIDs >= this haven't been assigned yet
- **xip[]**: List of active transactions between xmin and xmax

### Visibility Rules

A tuple is visible to a transaction if:

```
┌─────────────────────────────────────────────────────────────────┐
│  VISIBILITY DECISION TREE                                        │
│                                                                  │
│  Is tuple.xmin committed?                                        │
│  ├── NO (in-progress or aborted)                                │
│  │   ├── xmin is my own transaction? → VISIBLE (see own changes)│
│  │   └── Otherwise → INVISIBLE                                  │
│  │                                                               │
│  └── YES (committed)                                            │
│      └── Is tuple.xmax set (deleted)?                           │
│          ├── NO (xmax = 0) → VISIBLE                            │
│          └── YES                                                │
│              └── Is xmax committed?                             │
│                  ├── NO (in-progress/aborted) → VISIBLE         │
│                  └── YES → INVISIBLE (deleted before snapshot)  │
└─────────────────────────────────────────────────────────────────┘
```

### Simplified Visibility Check

```c
/* Simplified from src/backend/access/heap/heapam_visibility.c */
bool
HeapTupleSatisfiesMVCC(HeapTuple tuple, Snapshot snapshot)
{
    TransactionId xmin = tuple->t_xmin;
    TransactionId xmax = tuple->t_xmax;
    
    /* Check xmin (inserting transaction) */
    if (TransactionIdEquals(xmin, GetCurrentTransactionId()))
    {
        /* My own insert - visible if not deleted by me */
        if (xmax == 0)
            return true;
        if (TransactionIdEquals(xmax, GetCurrentTransactionId()))
            return false;  /* I deleted it */
        return true;
    }
    
    if (!XidInMVCCSnapshot(xmin, snapshot))
    {
        /* xmin not in snapshot - check if committed */
        if (!TransactionIdDidCommit(xmin))
            return false;  /* Inserter aborted */
    }
    else
    {
        /* xmin still active when snapshot was taken */
        return false;
    }
    
    /* Inserter committed, check xmax (deleting transaction) */
    if (xmax == 0)
        return true;  /* Not deleted */
    
    if (TransactionIdEquals(xmax, GetCurrentTransactionId()))
        return false;  /* I deleted it */
    
    if (!XidInMVCCSnapshot(xmax, snapshot))
    {
        /* Deleter not in snapshot */
        if (TransactionIdDidCommit(xmax))
            return false;  /* Deleter committed - deleted! */
    }
    
    /* Deleter still active or aborted - still visible */
    return true;
}
```

---

## Isolation Levels and Snapshot Timing

### When Snapshots Are Taken

| Isolation Level | Snapshot Timing |
|-----------------|-----------------|
| Read Committed | At the start of each **statement** |
| Repeatable Read | At the start of the first **statement** in transaction |
| Serializable | At the start of the first **statement** in transaction + conflict detection |

### Read Committed Behavior

```sql
-- Transaction A                    -- Transaction B
BEGIN;                              BEGIN;
SELECT * FROM t WHERE id = 1;       
-- Snapshot taken, sees old value   
                                    UPDATE t SET val = 'new' WHERE id = 1;
                                    COMMIT;

SELECT * FROM t WHERE id = 1;
-- New snapshot, sees 'new'!        
COMMIT;
```

Each SELECT gets a fresh snapshot, so it sees recently committed changes.

### Repeatable Read Behavior

```sql
-- Transaction A                    -- Transaction B
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM t WHERE id = 1;       
-- Snapshot taken, sees old value   
                                    UPDATE t SET val = 'new' WHERE id = 1;
                                    COMMIT;

SELECT * FROM t WHERE id = 1;
-- SAME snapshot, sees old value!   
COMMIT;
```

The snapshot is taken once and reused, ensuring consistent reads within the transaction.

---

## Practical Examples

### Example 1: Observing xmin and xmax

```sql
BEGIN;
SELECT txid_current();  -- Returns, e.g., 12345

CREATE TABLE mvcc_demo (id int, val text);
INSERT INTO mvcc_demo VALUES (1, 'original');

SELECT ctid, xmin, xmax, * FROM mvcc_demo;
-- xmin = 12345, xmax = 0

UPDATE mvcc_demo SET val = 'updated' WHERE id = 1;
SELECT ctid, xmin, xmax, * FROM mvcc_demo;
-- New ctid, xmin still 12345, xmax = 0

COMMIT;
```

### Example 2: Concurrent Transaction Visibility

```sql
-- Terminal 1                       -- Terminal 2
BEGIN;
SELECT txid_current();  -- 100
                                    BEGIN;
                                    SELECT txid_current();  -- 101

INSERT INTO t VALUES (1, 'visible');
                                    SELECT * FROM t;
                                    -- EMPTY! 100 not committed yet

COMMIT;
                                    SELECT * FROM t;
                                    -- Now sees row (new snapshot)
                                    COMMIT;
```

### Example 3: Deleted Row Visibility

```sql
-- Terminal 1                       -- Terminal 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM t;  -- Sees row       
                                    BEGIN;
                                    DELETE FROM t WHERE id = 1;
                                    COMMIT;

SELECT * FROM t;  -- STILL sees row!
COMMIT;
SELECT * FROM t;  -- Now empty
```

### Example 4: Using pageinspect

```sql
CREATE EXTENSION pageinspect;

SELECT 
    lp, 
    t_xmin, 
    t_xmax,
    t_ctid,
    CASE WHEN t_infomask & 256 > 0 THEN 'committed' ELSE '' END AS xmin_status,
    CASE WHEN t_infomask & 512 > 0 THEN 'invalid' ELSE '' END AS xmin_invalid
FROM heap_page_items(get_raw_page('mvcc_demo', 0));
```

---

## Commit Log (clog)

### How Transaction Status is Tracked

The commit log stores the status of every transaction:

```
┌─────────────────────────────────────────────────────────────────┐
│  pg_xact/ (formerly pg_clog)                                    │
│                                                                  │
│  2 bits per transaction:                                        │
│  - 00: IN_PROGRESS                                              │
│  - 01: COMMITTED                                                │
│  - 10: ABORTED                                                  │
│  - 11: SUB_COMMITTED (subtransaction)                          │
│                                                                  │
│  Storage: 4 transactions per byte, chunks of 8KB                │
│  File 0000: XIDs 0 - 32767                                      │
│  File 0001: XIDs 32768 - 65535                                  │
│  ...                                                            │
└─────────────────────────────────────────────────────────────────┘
```

### Hint Bits Optimization

Checking clog for every visibility decision is expensive. PostgreSQL caches the result in the tuple header using **hint bits**:

```c
/* Hint bits in t_infomask */
#define HEAP_XMIN_COMMITTED   0x0100   /* xmin is committed */
#define HEAP_XMIN_INVALID     0x0200   /* xmin is aborted */
#define HEAP_XMAX_COMMITTED   0x0400   /* xmax is committed */
#define HEAP_XMAX_INVALID     0x0800   /* xmax is aborted */
```

First check:
1. Look at hint bits
2. If set, use them (fast!)
3. If not set, check clog, then SET hint bits

Setting hint bits dirties the page, which is why SELECT statements can generate writes.

---

## MVCC and Dead Tuples

### Tuple Lifecycle

```
INSERT ──► Live Tuple (xmin=X, xmax=0)
             │
             ▼
UPDATE/DELETE ──► Dead Tuple (xmax=Y committed)
                    │
                    ▼ (when no transactions need it)
              VACUUM removes it
                    │
                    ▼
              Space reclaimed
```

### Why VACUUM is Essential

Old tuple versions accumulate until:
1. No active transaction might need them (visible in their snapshot)
2. VACUUM runs and marks them as reusable

Without VACUUM:
- Tables grow without bound
- Index bloat increases
- Performance degrades

---

## Hands-On Exercises

### Exercise 1: Observing Tuple Versions (Basic)

```sql
CREATE TABLE mvcc_test (id int PRIMARY KEY, val text);
INSERT INTO mvcc_test VALUES (1, 'v1');

SELECT ctid, xmin, xmax, * FROM mvcc_test;

UPDATE mvcc_test SET val = 'v2' WHERE id = 1;
SELECT ctid, xmin, xmax, * FROM mvcc_test;

-- Check both versions in page
SELECT lp, t_xmin, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('mvcc_test', 0));
```

### Exercise 2: Snapshot Isolation (Intermediate)

```sql
-- Terminal 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM mvcc_test;  -- Remember this

-- Terminal 2 (don't wait for 1)
BEGIN;
UPDATE mvcc_test SET val = 'v3' WHERE id = 1;
COMMIT;

-- Back to Terminal 1
SELECT * FROM mvcc_test;  -- What do you see?
COMMIT;
SELECT * FROM mvcc_test;  -- Now what?
```

### Exercise 3: Seeing Dead Tuples (Intermediate)

```sql
CREATE TABLE dead_test (id int);
INSERT INTO dead_test SELECT generate_series(1, 100);

-- Check tuple count before
SELECT lp_flags, count(*) 
FROM heap_page_items(get_raw_page('dead_test', 0))
GROUP BY lp_flags;

-- Delete some rows
DELETE FROM dead_test WHERE id <= 50;

-- Check tuple count after (before VACUUM)
SELECT lp_flags, count(*) 
FROM heap_page_items(get_raw_page('dead_test', 0))
GROUP BY lp_flags;

-- VACUUM cleans up
VACUUM dead_test;

SELECT lp_flags, count(*) 
FROM heap_page_items(get_raw_page('dead_test', 0))
GROUP BY lp_flags;
```

### Exercise 4: Hint Bits (Advanced)

```sql
-- Create table with data
CREATE TABLE hint_test (id int);
INSERT INTO hint_test SELECT generate_series(1, 100);
CHECKPOINT;  -- Ensure pages are on disk

-- Check infomask before any reads
SELECT lp, t_infomask & (256+512) AS xmin_hints
FROM heap_page_items(get_raw_page('hint_test', 0))
LIMIT 5;

-- Read the data (sets hint bits)
SELECT count(*) FROM hint_test;
CHECKPOINT;

-- Check infomask after reads
SELECT lp, t_infomask & (256+512) AS xmin_hints
FROM heap_page_items(get_raw_page('hint_test', 0))
LIMIT 5;
-- 256 = HEAP_XMIN_COMMITTED set
```

---

## Key Takeaways

1. **MVCC keeps multiple versions**: Updates create new tuples; old ones are marked with xmax.

2. **xmin/xmax track lifecycle**: xmin = inserter, xmax = deleter (or 0 if live).

3. **Snapshots define visibility**: Each transaction has a snapshot determining what it sees.

4. **Isolation levels affect snapshot timing**: Read Committed per-statement; Repeatable Read per-transaction.

5. **Hint bits optimize visibility checks**: Avoid repeated clog lookups.

6. **Dead tuples accumulate**: VACUUM is essential to reclaim space.

7. **Writers don't block readers**: Different transactions see different versions—maximum concurrency.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/access/heap/heapam_visibility.c` | Visibility checking |
| `src/backend/utils/time/snapmgr.c` | Snapshot management |
| `src/backend/access/transam/xact.c` | Transaction management |
| `src/backend/access/transam/clog.c` | Commit log |
| `src/include/access/htup_details.h` | Tuple header |

### Documentation

- [PostgreSQL Docs: Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL Docs: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)

### Next Lessons

- **Lesson 14**: Transaction Isolation Levels Deep Dive
- **Lesson 15**: VACUUM and Dead Tuple Cleanup
- **Lesson 16**: Transaction ID Wraparound

---

## References

### Key Structures

| Structure | File | Purpose |
|-----------|------|---------|
| `HeapTupleFields` | htup_details.h | xmin, xmax, cid, ctid |
| `SnapshotData` | snapshot.h | Snapshot definition |
| `TransactionId` | c.h | 32-bit XID type |

### Visibility Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `HeapTupleSatisfiesMVCC()` | heapam_visibility.c | Main visibility check |
| `XidInMVCCSnapshot()` | snapmgr.c | Is XID in snapshot? |
| `TransactionIdDidCommit()` | transam.c | Check clog for commit |
| `SetHintBits()` | heapam_visibility.c | Set visibility hints |

### System Columns

| Column | Purpose |
|--------|---------|
| `ctid` | Physical tuple location |
| `xmin` | Inserting transaction |
| `xmax` | Deleting transaction (0 if live) |
| `cmin` | Inserting command ID |
| `cmax` | Deleting command ID |
