# Lesson 9: B-tree Index Structure — PostgreSQL's Workhorse Index

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 2 — Storage Architecture |
| **Lesson** | 9 of 66 |
| **Estimated Time** | 90-120 minutes |
| **Prerequisites** | Lesson 8: Heap Storage and Page Layout |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Describe the structure of a B-tree index (metapage, root, internal, leaf)
2. Explain how the Lehman-Yao algorithm enables concurrent access
3. Understand how high keys and right-links enable non-blocking splits
4. Trace how a key lookup navigates the tree
5. Use `pageinspect` to examine B-tree internals

### Key Terms

| Term | Definition |
|------|------------|
| **B-tree** | Balanced tree structure optimized for disk-based storage |
| **Metapage** | Page 0, contains index metadata (root location, etc.) |
| **Root** | Top of the tree, entry point for all searches |
| **Internal Node** | Non-leaf node containing keys and child pointers |
| **Leaf Node** | Bottom level, contains keys and heap TIDs |
| **High Key** | Upper bound for keys on a page (Lehman-Yao feature) |
| **Right-Link** | Pointer to right sibling (for concurrent operations) |

---

## Introduction

B-trees are the default and most commonly used index type in PostgreSQL. When you create an index without specifying a type, you get a B-tree:

```sql
CREATE INDEX ON users(email);  -- Creates a B-tree index
```

B-trees excel at:
- **Equality lookups**: `WHERE email = 'alice@example.com'`
- **Range queries**: `WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'`
- **Sorting**: `ORDER BY last_name, first_name`
- **Prefix matching**: `WHERE name LIKE 'Al%'` (but not `LIKE '%lice'`)

PostgreSQL's B-tree implementation uses the **Lehman-Yao algorithm** (also called B-link tree), which adds right-sibling links and high keys to enable concurrent reads and writes without heavy locking.

> **Why "B-tree"?** The "B" has been variously interpreted as Balanced, Broad, Bushy, or Boeing (where inventors worked). The key property is balance—all leaf nodes are at the same depth, ensuring consistent O(log N) lookup time.

---

## Conceptual Foundation

### B-tree Structure Overview

```
                          ┌─────────────────┐
                          │   Metapage      │   Block 0
                          │  (root ptr: 3)  │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │    Root Node    │   Block 3
                          │  [7]  [15]  [23]│
                          └─┬─────┬─────┬──┬┘
                ┌───────────┘     │     │  └──────────┐
                ▼                 ▼     ▼             ▼
       ┌────────────────┐ ┌────────────┐ ┌────────────────┐
       │ Internal Node  │ │   ...      │ │ Internal Node  │
       │  [2] [4] [6]   │ │            │ │ [24] [26] [28] │
       └─┬───┬───┬───┬──┘ └────────────┘ └─┬───┬───┬───┬──┘
         ▼   ▼   ▼   ▼                     ▼   ▼   ▼   ▼
        ┌───┐┌───┐┌───┐┌───┐              Leaf   Leaf  ...
        │Lf1││Lf2││Lf3││Lf4│
        └─┬─┘└─┬─┘└─┬─┘└─┬─┘
          │    │    │    │
          ├◄───┼◄───┼◄───┘   Right-links between leaves
          └───►┼───►┼───►    (doubly linked list)
```

**Key components:**

| Component | Description |
|-----------|-------------|
| Metapage | Always block 0; stores root location and tree level |
| Root | Single page at the top; entry for all searches |
| Internal Pages | Middle levels; contain keys and child pointers |
| Leaf Pages | Bottom level; contain keys and heap TIDs |
| Right-Links | Each page points to its right sibling |
| High Keys | Each page stores its upper bound key |

### The Lehman-Yao B-Link Tree

Traditional B-trees require locking from root to leaf during modifications. The Lehman-Yao algorithm reduces locking by:

1. **Right-links**: Each page has a pointer to its right sibling
2. **High keys**: Each page (except rightmost) has an upper bound key

These additions allow:
- Readers to navigate without holding parent locks
- Splits to be detected and handled by following right-links
- Near lock-free reading during concurrent modifications

```
┌─────────────────────────────────────────────────────────────────┐
│  Page Layout (Lehman-Yao Style)                                  │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Page Header                                                  │ │
│  │ btpo_prev (left link) | btpo_next (right link) | btpo_level │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ High Key: [K_max]  ← Upper bound for this page              │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Index Entries:                                               │ │
│  │   [K1, TID1] → heap row                                      │ │
│  │   [K2, TID2] → heap row                                      │ │
│  │   [K3, TID3] → heap row                                      │ │
│  │   ...                                                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  If searching for key > K_max: follow right link!               │
└─────────────────────────────────────────────────────────────────┘
```

### How Key Lookup Works

To find `WHERE user_id = 42`:

```
Step 1: Read metapage → find root at block 3
         ↓
Step 2: Read block 3 (root): [10] [50] [100]
        42 is between 10 and 50 → follow second pointer → block 12
         ↓
Step 3: Read block 12 (internal): [20] [30] [40] [45]
        42 is between 40 and 45 → follow fourth pointer → block 45
         ↓
Step 4: Read block 45 (leaf): [40, tid1] [41, tid2] [42, tid3] [43, tid4]
        Found! 42 → tid3 → (5, 17)
         ↓
Step 5: Read heap page 5, line 17 → return row
```

**Complexity**: O(log N) pages read, where N is the number of indexed rows.

### Index-Only Scans

If all needed columns are in the index, PostgreSQL can skip the heap:

```sql
-- Covering index
CREATE INDEX ON orders(customer_id, total);

-- This can be an index-only scan
SELECT customer_id, total FROM orders WHERE customer_id = 42;
```

The index-only scan checks the **visibility map** to verify rows are visible without accessing the heap.

---

## Deep Dive: Implementation Details

### B-tree Page Structure

```c
/* From src/include/access/nbtree.h */

/* Page types */
#define P_NONE          0   /* Unused */
#define P_LEAF          1   /* Leaf page */
#define P_ROOT          2   /* Root page */
#define P_DELETED       3   /* Deleted page */
#define P_META          4   /* Metapage */
#define P_HALF_DEAD     5   /* Half-dead (being deleted) */

/* Page opaque data (at end of page, in pd_special area) */
typedef struct BTPageOpaqueData
{
    BlockNumber btpo_prev;      /* Left sibling, or P_NONE */
    BlockNumber btpo_next;      /* Right sibling, or P_NONE */
    union {
        uint32      level;      /* Tree level (0 = leaf) */
        TransactionId xact;     /* For deleted pages: deleting XID */
    } btpo;
    uint16      btpo_flags;     /* Flag bits */
    BTCycleId   btpo_cycleid;   /* Vac cycle ID of last defrag */
} BTPageOpaqueData;

/* Page flags */
#define BTP_LEAF        (1 << 0)    /* Leaf page */
#define BTP_ROOT        (1 << 1)    /* Root page */
#define BTP_DELETED     (1 << 2)    /* Page has been deleted */
#define BTP_META        (1 << 3)    /* Metapage */
#define BTP_SPLIT_END   (1 << 4)    /* Right half of split */
#define BTP_HAS_GARBAGE (1 << 5)    /* Has dead tuples */
```

### Index Tuple Structure

```c
/* From src/include/access/itup.h */
typedef struct IndexTupleData
{
    ItemPointerData t_tid;      /* 6 bytes: heap TID pointer */
    
    unsigned short  t_info;     /* 2 bytes: various flags + size */
    
    /* Key data follows */
} IndexTupleData;

/* For multi-column index, keys are stored contiguously */
/* Example: (user_id, created_at) index */
/*   t_tid(6) | t_info(2) | user_id(4) | created_at(8) */
```

### Metapage Structure

```c
/* From src/include/access/nbtree.h */
typedef struct BTMetaPageData
{
    uint32      btm_magic;          /* Magic number */
    uint32      btm_version;        /* Version number */
    BlockNumber btm_root;           /* Block # of root page */
    uint32      btm_level;          /* Tree level of root page */
    BlockNumber btm_fastroot;       /* Fast root (for VACUUM) */
    uint32      btm_fastlevel;      /* Level of fast root */
    /* Statistics */
    uint32      btm_oldest_btpo_xact;
    float8      btm_last_cleanup_num_heap_tuples;
    bool        btm_allequalimage;  /* All columns support = operator */
} BTMetaPageData;
```

### B-tree Search Algorithm

```c
/* Simplified from src/backend/access/nbtree/nbtsearch.c */
Buffer
_bt_search(Relation rel, BTScanInsert key, 
           Buffer *bufP, int access)
{
    BTStack     stack;
    Buffer      buf;
    Page        page;
    BTPageOpaque opaque;
    OffsetNumber offnum;
    
    /* Start at root */
    buf = _bt_getroot(rel, access);
    
    for (;;)
    {
        page = BufferGetPage(buf);
        opaque = (BTPageOpaque) PageGetSpecialPointer(page);
        
        /* If this is a leaf, we're done descending */
        if (P_ISLEAF(opaque))
            break;
        
        /* Find the child pointer to follow */
        offnum = _bt_binsrch(rel, buf, key);
        
        /* Descend to child */
        childblk = _bt_getchildblock(page, offnum);
        buf = _bt_relandgetbuf(rel, buf, childblk, access);
    }
    
    /* Now on leaf page, handle possible concurrent split */
    buf = _bt_moveright(rel, buf, key, ...);
    
    return buf;
}
```

### Handling Concurrent Splits (moveright)

```c
/* Simplified from src/backend/access/nbtree/nbtsearch.c */
Buffer
_bt_moveright(Relation rel, Buffer buf, BTScanInsert key, ...)
{
    Page        page;
    BTPageOpaque opaque;
    
    page = BufferGetPage(buf);
    opaque = (BTPageOpaque) PageGetSpecialPointer(page);
    
    /* Keep moving right while key > high key */
    while (!P_RIGHTMOST(opaque) && 
           _bt_compare(rel, key, page, P_HIKEY) > 0)
    {
        /* Key is beyond this page's high key */
        BlockNumber rblkno = opaque->btpo_next;
        
        /* Move to right sibling */
        buf = _bt_relandgetbuf(rel, buf, rblkno, BT_READ);
        page = BufferGetPage(buf);
        opaque = (BTPageOpaque) PageGetSpecialPointer(page);
    }
    
    return buf;
}
```

### Page Split Algorithm

```c
/* Simplified from src/backend/access/nbtree/nbtinsert.c */
Buffer
_bt_split(Relation rel, Buffer buf, OffsetNumber firstright,
          IndexTuple newitem, bool newitemonleft)
{
    /* 1. Allocate new right page */
    Buffer rbuf = _bt_getbuf(rel, P_NEW, BT_WRITE);
    
    /* 2. Determine split point (firstright) */
    /* Goal: balance free space between pages */
    
    /* 3. Copy items >= firstright to new page */
    for (i = firstright; i <= max; i++)
        _bt_pgaddtup(rightpage, item[i]);
    
    /* 4. Set high key on left page */
    /* High key = first key of right page */
    _bt_truncate(rel, firstright_item, &truncated);
    _bt_insert_hikey(leftpage, truncated);
    
    /* 5. Update sibling links */
    rightopaque->btpo_prev = BufferGetBlockNumber(buf);
    rightopaque->btpo_next = leftopaque->btpo_next;
    leftopaque->btpo_next = BufferGetBlockNumber(rbuf);
    
    /* 6. Insert pointer to right page in parent */
    /* This may cause parent to split recursively */
    _bt_insert_parent(rel, buf, rbuf, ...);
    
    return rbuf;
}
```

---

## B-tree Variants and Options

### Unique Indexes

```sql
CREATE UNIQUE INDEX ON users(email);
```

Unique indexes check for duplicates during insertion. They use the B-tree structure with additional uniqueness validation.

### Partial Indexes

```sql
CREATE INDEX active_orders_idx ON orders(customer_id) 
WHERE status = 'active';
```

Only indexes rows matching the predicate. Smaller index = faster scans and updates.

### Covering Indexes (INCLUDE)

```sql
CREATE INDEX customer_orders_idx ON orders(customer_id) 
INCLUDE (total, order_date);
```

Included columns are in leaf pages but not used for searching. Enables index-only scans for more queries.

### Multi-Column Indexes

```sql
CREATE INDEX ON orders(customer_id, order_date, status);
```

Column order matters! This index is useful for:
- `WHERE customer_id = 42` ✓
- `WHERE customer_id = 42 AND order_date = '2024-01-01'` ✓
- `WHERE customer_id = 42 AND order_date > '2024-01-01'` ✓
- `WHERE order_date = '2024-01-01'` ✗ (no leading column)
- `WHERE status = 'active'` ✗ (no leading columns)

### Expression Indexes

```sql
CREATE INDEX ON users(lower(email));
```

Indexes the result of an expression. Query must use the same expression:

```sql
SELECT * FROM users WHERE lower(email) = 'alice@example.com';  -- Uses index
SELECT * FROM users WHERE email = 'alice@example.com';         -- Won't use it
```

---

## Performance Characteristics

### Space Requirements

B-tree index size depends on:
- Number of indexed rows
- Size of indexed columns
- Fill factor (default 90%)

```sql
-- Estimate index size
SELECT 
    pg_relation_size('users_email_idx') AS index_bytes,
    pg_size_pretty(pg_relation_size('users_email_idx')) AS index_size
FROM pg_indexes WHERE indexname = 'users_email_idx';
```

### Lookup Complexity

| Operation | Complexity | Typical Disk Reads |
|-----------|------------|-------------------|
| Point Lookup | O(log N) | 3-5 (tree depth + heap) |
| Range Scan | O(log N + M) | Tree depth + range pages |
| Full Scan | O(N) | All leaf pages |
| Insert | O(log N) | Tree depth × 2 |
| Delete | O(log N) | Tree depth × 2 |

Tree depth for typical table sizes:

| Rows | Approximate Depth |
|------|-------------------|
| 100 | 1-2 |
| 10,000 | 2-3 |
| 1,000,000 | 3-4 |
| 100,000,000 | 4-5 |

### Index Bloat

Like heap tables, indexes can become bloated:

```sql
-- Check index bloat estimate (requires pgstattuple extension)
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstatindex('users_pkey');
```

```
 version | tree_level | index_size | root_block_no | internal_pages | leaf_pages | empty_pages | deleted_pages | avg_leaf_density | leaf_fragmentation
---------+------------+------------+---------------+----------------+------------+-------------+---------------+------------------+--------------------
       4 |          2 |   10485760 |             3 |              5 |       1270 |           0 |             5 |            89.51 |                 2.3
```

Key metrics:
- `avg_leaf_density`: Should be close to fillfactor (90%)
- `deleted_pages`: Wasted pages
- `leaf_fragmentation`: Pages out of logical order

### REINDEX for Maintenance

```sql
-- Rebuild bloated index (blocks writes!)
REINDEX INDEX users_email_idx;

-- Concurrent rebuild (doesn't block)
REINDEX INDEX CONCURRENTLY users_email_idx;
```

---

## Practical Exploration with pageinspect

### Examining B-tree Structure

```sql
CREATE EXTENSION pageinspect;

-- Create test table and index
CREATE TABLE btree_demo (id serial, value text);
INSERT INTO btree_demo (value) 
SELECT 'value_' || i FROM generate_series(1, 10000) i;
CREATE INDEX btree_demo_idx ON btree_demo(id);

-- View metapage
SELECT * FROM bt_metap('btree_demo_idx');
```

```
 magic  | version |  root  | level | fastroot | fastlevel | oldest_xact | last_cleanup_num_heap_tuples | allequalimage
--------+---------+--------+-------+----------+-----------+-------------+------------------------------+---------------
 340322 |       4 |      3 |     1 |        3 |         1 |           0 |                           -1 | t
```

- `root = 3`: Root page is block 3
- `level = 1`: Tree has 2 levels (root + leaves)

### Viewing Page Statistics

```sql
SELECT * FROM bt_page_stats('btree_demo_idx', 3);  -- Root page
```

```
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
     3 | r    |         27 |          0 |            15 |      8192 |      7732 |         0 |         0 |          1 |          2
```

- `type = 'r'`: Root page
- `btpo_level = 1`: One level above leaves

### Viewing Page Items

```sql
SELECT * FROM bt_page_items('btree_demo_idx', 3) LIMIT 10;
```

```
 itemoffset |   ctid   | itemlen | nulls | vars |          data
------------+----------+---------+-------+------+-------------------------
          1 | (1,0)    |       8 | f     | f    |
          2 | (1,1)    |      16 | f     | f    | 6f 01 00 00 00 00 00 00
          3 | (2,1)    |      16 | f     | f    | dd 02 00 00 00 00 00 00
```

- `itemoffset = 1`: High key (first item on non-rightmost pages)
- `ctid`: For internal pages, points to child page

### Tracing a Key Lookup

```sql
-- Find where id = 5000 would be

-- 1. Check root (page 3) for child pointer
SELECT itemoffset, ctid, data FROM bt_page_items('btree_demo_idx', 3)
WHERE itemoffset > 1;  -- Skip high key

-- 2. Follow pointer to leaf page
SELECT * FROM bt_page_items('btree_demo_idx', <leaf_block>) 
WHERE data LIKE '%88 13 00 00%';  -- 5000 in little-endian hex
```

---

## Hands-On Exercises

### Exercise 1: B-tree Basics (Basic)

```sql
-- Create table and index
CREATE TABLE btree_test (id int PRIMARY KEY, name text);
INSERT INTO btree_test SELECT i, 'name_' || i FROM generate_series(1, 100000) i;

-- Examine the index
SELECT * FROM bt_metap('btree_test_pkey');

-- How many levels? What's the root block?

-- View leaf page items
SELECT * FROM bt_page_stats('btree_test_pkey', 1);
SELECT * FROM bt_page_items('btree_test_pkey', 1) LIMIT 20;
```

### Exercise 2: Multi-Column Index Order (Intermediate)

```sql
CREATE TABLE multi_col (a int, b int, c int);
INSERT INTO multi_col SELECT i % 100, i % 1000, i FROM generate_series(1, 100000) i;

CREATE INDEX ON multi_col(a, b, c);
ANALYZE multi_col;

-- Which queries use the index?
EXPLAIN SELECT * FROM multi_col WHERE a = 50;
EXPLAIN SELECT * FROM multi_col WHERE b = 500;
EXPLAIN SELECT * FROM multi_col WHERE a = 50 AND b = 500;
EXPLAIN SELECT * FROM multi_col WHERE a = 50 AND c = 50000;
EXPLAIN SELECT * FROM multi_col WHERE b = 500 AND c = 50000;
```

### Exercise 3: Observing Index-Only Scan (Intermediate)

```sql
CREATE TABLE orders_test (
    id serial PRIMARY KEY,
    customer_id int,
    total numeric
);
INSERT INTO orders_test (customer_id, total) 
SELECT random() * 1000, random() * 1000 FROM generate_series(1, 100000);

-- Regular index
CREATE INDEX ON orders_test(customer_id);

-- Covering index
CREATE INDEX ON orders_test(customer_id) INCLUDE (total);

ANALYZE orders_test;
VACUUM orders_test;  -- Update visibility map

-- Compare plans
EXPLAIN ANALYZE SELECT customer_id, total FROM orders_test WHERE customer_id = 500;
```

### Exercise 4: Partial Index (Intermediate)

```sql
CREATE TABLE tickets (
    id serial PRIMARY KEY,
    status text,
    created_at timestamp
);
INSERT INTO tickets (status, created_at)
SELECT 
    CASE WHEN random() < 0.05 THEN 'open' ELSE 'closed' END,
    now() - (random() * 365 * interval '1 day')
FROM generate_series(1, 1000000);

-- Full index
CREATE INDEX tickets_status_idx ON tickets(status);

-- Partial index (only open tickets)
CREATE INDEX tickets_open_idx ON tickets(created_at) WHERE status = 'open';

ANALYZE tickets;

-- Compare sizes
SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass))
FROM pg_indexes WHERE tablename = 'tickets';

-- Compare queries
EXPLAIN ANALYZE SELECT * FROM tickets WHERE status = 'open' ORDER BY created_at;
EXPLAIN ANALYZE SELECT * FROM tickets WHERE status = 'closed' ORDER BY created_at;
```

### Exercise 5: Index Bloat Detection (Advanced)

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- Create table with many updates
CREATE TABLE bloat_test (id serial PRIMARY KEY, val int);
INSERT INTO bloat_test (val) SELECT i FROM generate_series(1, 10000) i;

-- Check initial state
SELECT * FROM pgstatindex('bloat_test_pkey');

-- Create bloat through updates
UPDATE bloat_test SET val = val + 1;
UPDATE bloat_test SET val = val + 1;
UPDATE bloat_test SET val = val + 1;

-- Check bloated state
SELECT * FROM pgstatindex('bloat_test_pkey');

-- VACUUM and REINDEX
VACUUM bloat_test;
SELECT * FROM pgstatindex('bloat_test_pkey');

REINDEX INDEX bloat_test_pkey;
SELECT * FROM pgstatindex('bloat_test_pkey');
```

---

## Key Takeaways

1. **B-trees are balanced**: All lookups are O(log N) regardless of key value.

2. **Lehman-Yao enables concurrency**: Right-links and high keys allow non-blocking reads during splits.

3. **Tree depth is shallow**: Even billions of rows only need 4-5 levels.

4. **Column order matters**: Multi-column indexes are only useful when queries include leading columns.

5. **Covering indexes enable index-only scans**: Include all needed columns to skip heap access.

6. **Partial indexes save space**: Only index relevant rows.

7. **Indexes need maintenance**: REINDEX to fix bloat, VACUUM to clean up dead entries.

8. **High keys define page boundaries**: Key > high key means move right.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/include/access/nbtree.h` | B-tree definitions |
| `src/backend/access/nbtree/nbtinsert.c` | Insertion and splitting |
| `src/backend/access/nbtree/nbtsearch.c` | Search algorithm |
| `src/backend/access/nbtree/nbtpage.c` | Page management |
| `src/backend/access/nbtree/nbtutils.c` | Utility functions |

### Documentation

- [PostgreSQL Docs: B-tree Indexes](https://www.postgresql.org/docs/current/btree.html)
- [PostgreSQL Docs: CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html)
- [Lehman & Yao Paper](https://www.csd.uoc.gr/~hy460/pdf/p650-lehman.pdf)

### Next Lessons

- **Lesson 10**: Tablespaces and Relation Files
- **Lesson 11**: Other Index Types (Hash, GIN, GiST, BRIN)
- **Module 9**: Query Optimization with Indexes

---

## References

### Key Structures

| Structure | File | Purpose |
|-----------|------|---------|
| `BTPageOpaqueData` | nbtree.h | Page special area data |
| `BTMetaPageData` | nbtree.h | Metapage contents |
| `IndexTupleData` | itup.h | Index entry |
| `BTScanInsert` | nbtree.h | Search key structure |

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `_bt_search()` | nbtsearch.c | Navigate to leaf |
| `_bt_moveright()` | nbtsearch.c | Follow right links |
| `_bt_split()` | nbtinsert.c | Split a full page |
| `_bt_insert()` | nbtinsert.c | Insert a key |
| `_bt_binsrch()` | nbtsearch.c | Binary search within page |
| `_bt_compare()` | nbtutils.c | Compare keys |
