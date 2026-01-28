# Lesson 8: Heap Storage and Page Layout — How PostgreSQL Stores Table Data

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 2 — Storage Architecture |
| **Lesson** | 8 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Module 1 completion recommended |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain how PostgreSQL organizes table data in heap files
2. Describe the internal structure of an 8KB page
3. Understand how tuples are stored within pages
4. Use `pageinspect` to examine page internals
5. Explain how free space is managed within pages

### Key Terms

| Term | Definition |
|------|------------|
| **Heap** | The primary storage structure for table data (unordered) |
| **Page** | A fixed-size (8KB) block of storage |
| **Tuple** | A single row stored in a page |
| **Line Pointer** | An array entry pointing to a tuple within a page |
| **TID (ctid)** | Tuple ID: (block number, line pointer offset) |
| **Free Space Map (FSM)** | Tracks available space in pages |
| **Visibility Map (VM)** | Tracks which pages are all-visible |

---

## Introduction

Every SQL query ultimately reads from or writes to disk storage. Understanding how PostgreSQL physically stores data helps you:

1. **Optimize table design**: Row size affects tuples per page affects I/O
2. **Understand I/O patterns**: Sequential vs. random access makes huge performance differences
3. **Diagnose fragmentation**: Dead tuples accumulate in pages over time
4. **Tune storage settings**: Page size, fill factor, TOAST thresholds

PostgreSQL uses a **heap** storage model for tables. Unlike a "heap" data structure (priority queue), a database heap is simply an unordered collection of pages. Rows can be anywhere in the heap, and new rows go wherever there's space.

This contrasts with **clustered indexes** (like SQL Server or MySQL InnoDB) where table data is stored sorted by primary key. PostgreSQL's heap approach is simpler and works well with MVCC.

> **Why 8KB Pages?** The 8KB page size (configurable at compile time) balances several factors:
> - Small enough for efficient caching
> - Large enough to hold typical rows
> - Matches common OS and disk block sizes
> - Powers of 2 for efficient addressing

---

## Conceptual Foundation

### The Storage Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                      Database Cluster                            │
│  └── PGDATA directory                                           │
│      └── base/                                                   │
│          └── <database OID>/                                     │
│              ├── 16384           (table: orders)                │
│              ├── 16384_fsm       (free space map)               │
│              ├── 16384_vm        (visibility map)               │
│              ├── 16385           (index: orders_pkey)           │
│              └── ...                                             │
└─────────────────────────────────────────────────────────────────┘
```

Each table is stored as one or more files:
- **Main file**: The heap data (named by relfilenode OID)
- **FSM file**: Tracks free space in pages
- **VM file**: Tracks visibility status

Large tables span multiple 1GB segments: `16384`, `16384.1`, `16384.2`, etc.

### Page Structure Overview

Every PostgreSQL page has the same basic structure:

```
┌─────────────────────────────────────────────────────────────────┐
│                         8192 bytes (8KB)                         │
├─────────────────────────────────────────────────────────────────┤
│  PageHeaderData (24 bytes)                                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ pd_lsn (8)  │ pd_checksum (2) │ pd_flags (2) │ pd_lower (2) │ │
│  │ pd_upper (2) │ pd_special (2) │ pd_pagesize_version (2)     │ │
│  │ pd_prune_xid (4)                                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  Line Pointers (ItemIdData array)                                │
│  ┌────┬────┬────┬────┬────┬────┬────────────────────────────┐   │
│  │ LP1│ LP2│ LP3│ LP4│ LP5│ LP6│  ...grows downward →       │   │
│  └────┴────┴────┴────┴────┴────┴────────────────────────────┘   │
│       (4 bytes each)                pd_lower points here ↑       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    FREE SPACE                                    │
│                                                                  │
│         pd_lower ↓ grows down    pd_upper ↑ grows up            │
├─────────────────────────────────────────────────────────────────┤
│  Tuples (grow upward from bottom)                               │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 Tuple 6 (newest)                           │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │                 Tuple 5                                     │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │                 Tuple 4                                     │ │
│  ├────────────────────────────────────────────────────────────────┤
│  │                 Tuple 3                                     │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │                 Tuple 2                                     │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │                 Tuple 1 (oldest)            ← pd_upper here │ │
│  └────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  Special Space (for indexes, empty for heap pages)              │
│                                              ← pd_special here   │
└─────────────────────────────────────────────────────────────────┘
```

### Tuple Identifier (TID / ctid)

Every tuple has a physical address: `(block_number, offset_number)`

```sql
SELECT ctid, * FROM users LIMIT 5;
```

```
 ctid  | id | name
-------+----+-------
 (0,1) |  1 | Alice
 (0,2) |  2 | Bob
 (0,3) |  3 | Carol
 (1,1) |  4 | Dave    -- First tuple on block 1
 (1,2) |  5 | Eve
```

- `(0,1)` = Block 0, line pointer 1
- `(1,1)` = Block 1, line pointer 1

**Important**: TIDs are physical locations, not logical identifiers. They change when:
- Rows are updated (new tuple created)
- Table is VACUUMed (tuples may be compacted)
- Table is REINDEXed or CLUSTERed

---

## Deep Dive: Implementation Details

### PageHeaderData Structure

```c
/* From src/include/storage/bufpage.h */
typedef struct PageHeaderData
{
    /* LSN of last change (for WAL) */
    PageXLogRecPtr  pd_lsn;         /* 8 bytes */
    
    /* Checksum for data integrity */
    uint16          pd_checksum;    /* 2 bytes */
    
    /* Flag bits (see below) */
    uint16          pd_flags;       /* 2 bytes */
    
    /* Offset to start of free space */
    LocationIndex   pd_lower;       /* 2 bytes */
    
    /* Offset to end of free space */
    LocationIndex   pd_upper;       /* 2 bytes */
    
    /* Offset to start of special space */
    LocationIndex   pd_special;     /* 2 bytes */
    
    /* Page size and version */
    uint16          pd_pagesize_version;  /* 2 bytes */
    
    /* Oldest XID for pruning */
    TransactionId   pd_prune_xid;   /* 4 bytes */
    
    /* Total: 24 bytes */
} PageHeaderData;

/* Flag bits */
#define PD_HAS_FREE_LINES   0x0001  /* Has unused line pointers */
#define PD_PAGE_FULL        0x0002  /* Not enough free space */
#define PD_ALL_VISIBLE      0x0004  /* All tuples are visible */
```

### Line Pointers (ItemIdData)

```c
/* From src/include/storage/itemid.h */
typedef struct ItemIdData
{
    unsigned    lp_off:15,      /* Offset to tuple (from page start) */
                lp_flags:2,     /* State of the line pointer */
                lp_len:15;      /* Byte length of tuple */
} ItemIdData;                   /* 4 bytes */

/* lp_flags values */
#define LP_UNUSED       0   /* Unused (available for use) */
#define LP_NORMAL       1   /* Used, points to a tuple */
#define LP_REDIRECT     2   /* HOT redirect to another offset */
#define LP_DEAD         3   /* Dead, can be reused */
```

### HeapTupleHeaderData

Each tuple has a header before its data:

```c
/* From src/include/access/htup_details.h (simplified) */
typedef struct HeapTupleHeaderData
{
    union {
        HeapTupleFields t_heap;
        DatumTupleFields t_datum;
    } t_choice;
    
    ItemPointerData t_ctid;     /* Current TID (or pointer to newer version) */
    
    uint16      t_infomask2;    /* Number of attributes + flags */
    uint16      t_infomask;     /* Various flags */
    uint8       t_hoff;         /* Offset to user data */
    
    /* Followed by null bitmap (if any nulls) */
    /* Then user data */
} HeapTupleHeaderData;

/* Key fields in t_heap */
typedef struct HeapTupleFields
{
    TransactionId t_xmin;       /* Inserting transaction XID */
    TransactionId t_xmax;       /* Deleting/updating transaction XID */
    union {
        CommandId   t_cid;      /* Command ID */
        TransactionId t_xvac;   /* VACUUM transaction ID */
    } t_field3;
} HeapTupleFields;
```

**Tuple layout:**

```
┌─────────────────────────────────────────────────────────────────┐
│ HeapTupleHeaderData (23 bytes minimum)                          │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ t_xmin (4) │ t_xmax (4) │ t_cid (4) │ t_ctid (6)           │ │
│ │ t_infomask2 (2) │ t_infomask (2) │ t_hoff (1)              │ │
│ └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│ Null Bitmap (if t_infomask has HEAP_HASNULL)                    │
│ 1 bit per column: 0 = NULL, 1 = not NULL                        │
├─────────────────────────────────────────────────────────────────┤
│ Padding (to MAXALIGN boundary)                                  │
├─────────────────────────────────────────────────────────────────┤
│ User Data (column values)                                       │
│ ┌─────────┬─────────┬─────────┬─────────┬─────────────────────┐ │
│ │ Column1 │ Column2 │ Column3 │   ...   │ (var-length last)   │ │
│ └─────────┴─────────┴─────────┴─────────┴─────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Free Space Tracking

Pages track free space with `pd_lower` and `pd_upper`:

```
Free space = pd_upper - pd_lower
```

When free space is too small for a new tuple + line pointer:
1. PostgreSQL checks the Free Space Map (FSM)
2. FSM tracks approximate free space per page
3. If no page has enough space, extend the file (add new page)

### Fill Factor

The `fillfactor` setting reserves space in pages for updates:

```sql
CREATE TABLE orders (
    ...
) WITH (fillfactor = 70);
```

With `fillfactor = 70`:
- Only fill pages to 70%
- Leave 30% for HOT updates (updates that can stay on same page)
- Reduces table bloat from update-heavy workloads

---

## Practical Exploration with pageinspect

### Installing pageinspect

```sql
CREATE EXTENSION pageinspect;
```

### Examining Page Headers

```sql
-- Create test table
CREATE TABLE page_demo (
    id serial PRIMARY KEY,
    name text,
    value int
);

INSERT INTO page_demo (name, value) 
SELECT 'item' || i, i FROM generate_series(1, 100) i;

-- View page header
SELECT * FROM page_header(get_raw_page('page_demo', 0));
```

```
    lsn     | checksum | flags | lower | upper | special | pagesize | version | prune_xid
------------+----------+-------+-------+-------+---------+----------+---------+----------
 0/164F3A0  |        0 |     0 |   224 |  4704 |    8192 |     8192 |       4 |         0
```

**Interpreting:**
- `lower = 224`: Line pointers end at byte 224
- `upper = 4704`: Tuples start at byte 4704
- Free space: 4704 - 224 = 4480 bytes
- `special = 8192`: No special space (heap page)

### Viewing Line Pointers

```sql
SELECT * FROM heap_page_items(get_raw_page('page_demo', 0)) LIMIT 10;
```

```
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_ctid | t_attrs
----+--------+----------+--------+--------+--------+--------+---------
  1 |   8152 |        1 |     36 | 100001 |      0 | (0,1)  | 3
  2 |   8112 |        1 |     36 | 100001 |      0 | (0,2)  | 3
  3 |   8072 |        1 |     36 | 100001 |      0 | (0,3)  | 3
  ...
```

- `lp = 1`: First line pointer
- `lp_off = 8152`: Tuple starts at byte 8152
- `lp_flags = 1`: LP_NORMAL (valid tuple)
- `lp_len = 36`: Tuple is 36 bytes
- `t_xmin`: Transaction that inserted this tuple
- `t_xmax = 0`: Not deleted/updated

### Examining Tuple Data

```sql
SELECT t_data FROM heap_page_items(get_raw_page('page_demo', 0)) LIMIT 3;
```

```
                  t_data
------------------------------------------
 \x010000001169746530310000000001000000
 \x020000001169746530320000000002000000
 \x030000001169746530330000000003000000
```

The raw bytes include:
- 4-byte integer (id)
- Variable-length text (name)
- 4-byte integer (value)

### Free Space Calculation

```sql
SELECT 
    pagesize, 
    upper - lower AS free_space,
    round(100.0 * (upper - lower) / pagesize, 1) AS free_pct
FROM page_header(get_raw_page('page_demo', 0));
```

### Viewing Free Space Map

```sql
SELECT * FROM pg_freespace('page_demo');
```

```
 blkno | avail
-------+-------
     0 |  4480
     1 |  4512
     2 |  8160   -- Nearly empty page
```

---

## Performance Implications

### Row Size and Tuples Per Page

Smaller rows = more tuples per page = fewer I/O operations:

```sql
-- Calculate average row size and tuples per page
SELECT 
    pg_relation_size('orders') / 8192 AS total_pages,
    reltuples::bigint AS total_rows,
    round(reltuples / (pg_relation_size('orders') / 8192.0), 1) AS rows_per_page
FROM pg_class
WHERE relname = 'orders';
```

**Optimization tips:**
- Use appropriate data types (int2 vs int4 vs int8)
- Avoid overly wide VARCHAR
- Consider normalizing wide tables

### Sequential vs. Random I/O

```
Sequential read:  Page 0 → Page 1 → Page 2 → Page 3 → ...
Random read:      Page 0 → Page 47 → Page 12 → Page 88 → ...
```

Sequential is ~25-100x faster on HDDs (seek time matters).
Less difference on SSDs, but sequential still wins.

**EXPLAIN shows this:**
- `Seq Scan`: Sequential pages
- `Index Scan`: Random pages (for heap fetches)
- `Bitmap Heap Scan`: Batched random → more sequential

### Table Bloat

When rows are updated/deleted:
1. Old tuple is marked dead (not immediately removed)
2. Dead tuples waste space
3. Table becomes "bloated"

```sql
-- Check bloat estimate
SELECT 
    schemaname, tablename,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public';

-- After VACUUM FULL (rewrites table)
VACUUM FULL tablename;
```

### Page Splits and Fragmentation

For heap tables, there are no "splits" like B-tree indexes. But:
- Dead tuples fragment pages
- Updates may scatter related rows across pages
- CLUSTER can reorder based on an index

```sql
-- Reorder orders table by order_date
CLUSTER orders USING orders_date_idx;
```

---

## TOAST: The Oversized-Attribute Storage Technique

Large values (>2KB typically) don't fit inline:

```
┌─────────────────────────────────────────────────────────────────┐
│  Main Table Page                                                 │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Tuple with TOAST pointer:                                   │ │
│  │ [id][name][large_text_ptr → TOAST table]                   │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  TOAST Table (pg_toast.pg_toast_<oid>)                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Chunk 1: [chunk_id][chunk_seq=0][chunk_data (2000 bytes)]  │ │
│  │ Chunk 2: [chunk_id][chunk_seq=1][chunk_data (2000 bytes)]  │ │
│  │ Chunk 3: [chunk_id][chunk_seq=2][chunk_data (remaining)]   │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**TOAST strategies:**

| Strategy | Behavior |
|----------|----------|
| PLAIN | No TOAST (fixed-length only) |
| EXTENDED | Compress, then out-of-line if needed |
| EXTERNAL | Out-of-line without compression |
| MAIN | Compression, avoid out-of-line if possible |

```sql
-- Change TOAST strategy
ALTER TABLE documents ALTER COLUMN content SET STORAGE EXTERNAL;
```

### Viewing TOAST Tables

```sql
SELECT 
    relname AS table_name,
    reltoastrelid::regclass AS toast_table
FROM pg_class
WHERE reltoastrelid != 0 AND relname = 'documents';
```

---

## Hands-On Exercises

### Exercise 1: Page Inspection (Basic)

```sql
-- Install extension
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- Create and populate table
CREATE TABLE inspect_me (id int, data text);
INSERT INTO inspect_me SELECT i, 'Row ' || i FROM generate_series(1, 50) i;

-- Examine the page
SELECT * FROM page_header(get_raw_page('inspect_me', 0));
SELECT lp, lp_off, lp_len, t_xmin, t_ctid 
FROM heap_page_items(get_raw_page('inspect_me', 0));
```

### Exercise 2: Free Space Tracking (Intermediate)

```sql
-- Create table with small fill factor
CREATE TABLE low_fill (id serial, data text) WITH (fillfactor = 50);
INSERT INTO low_fill (data) SELECT 'x' FROM generate_series(1, 1000);

-- Check free space
SELECT blkno, avail FROM pg_freespace('low_fill') LIMIT 10;

-- Compare to default fillfactor
CREATE TABLE normal_fill (id serial, data text);
INSERT INTO normal_fill (data) SELECT 'x' FROM generate_series(1, 1000);
SELECT blkno, avail FROM pg_freespace('normal_fill') LIMIT 10;
```

### Exercise 3: Observing Updates (Intermediate)

```sql
-- Create table and check page
CREATE TABLE update_test (id int PRIMARY KEY, value int);
INSERT INTO update_test VALUES (1, 100);

SELECT lp, lp_off, lp_flags, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('update_test', 0));

-- Update the row
UPDATE update_test SET value = 200 WHERE id = 1;

-- Check page again - see both versions!
SELECT lp, lp_off, lp_flags, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('update_test', 0));

-- VACUUM removes the old version
VACUUM update_test;
SELECT lp, lp_off, lp_flags, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('update_test', 0));
```

### Exercise 4: TOAST Exploration (Advanced)

```sql
-- Create table with large text
CREATE TABLE toast_test (id serial, content text);

-- Insert large content
INSERT INTO toast_test (content) 
SELECT repeat('x', 10000) FROM generate_series(1, 10);

-- Find TOAST table
SELECT reltoastrelid::regclass FROM pg_class WHERE relname = 'toast_test';

-- Examine TOAST chunks
SELECT chunk_id, chunk_seq, length(chunk_data) 
FROM pg_toast.pg_toast_<oid>  -- Use actual OID
ORDER BY chunk_id, chunk_seq;
```

### Exercise 5: Row Size Analysis (Advanced)

```sql
-- Create table with various column types
CREATE TABLE size_test (
    a smallint,    -- 2 bytes
    b integer,     -- 4 bytes
    c bigint,      -- 8 bytes
    d text,        -- variable
    e timestamp    -- 8 bytes
);

INSERT INTO size_test 
SELECT i, i, i, 'test' || i, now() 
FROM generate_series(1, 100) i;

-- Analyze row sizes using heap_page_items
SELECT 
    avg(lp_len) AS avg_tuple_size,
    min(lp_len) AS min_tuple_size,
    max(lp_len) AS max_tuple_size
FROM heap_page_items(get_raw_page('size_test', 0));
```

---

## Key Takeaways

1. **Tables are stored as heap files**: Unordered collections of pages.

2. **Pages are 8KB fixed-size blocks**: Header + line pointers + free space + tuples.

3. **Line pointers provide indirection**: TID → line pointer → tuple offset. This allows tuples to move within a page.

4. **Tuples have headers**: xmin, xmax, ctid, null bitmap—essential for MVCC.

5. **Free space is tracked**: FSM (Free Space Map) helps find pages with room.

6. **Large values are TOASTed**: Compressed and/or stored out-of-line in chunks.

7. **Fill factor reserves headroom**: Lower fillfactor leaves room for HOT updates.

8. **Row size affects performance**: Fewer bytes per row = more rows per page = less I/O.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/include/storage/bufpage.h` | Page layout definitions |
| `src/include/storage/itemid.h` | Line pointer definitions |
| `src/include/access/htup_details.h` | Tuple header structure |
| `src/backend/access/heap/heapam.c` | Heap access methods |
| `src/backend/storage/freespace/` | Free space map code |

### Documentation

- [PostgreSQL Docs: Database Physical Storage](https://www.postgresql.org/docs/current/storage.html)
- [PostgreSQL Docs: Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html)
- [PostgreSQL Docs: TOAST](https://www.postgresql.org/docs/current/storage-toast.html)

### Next Lessons

- **Lesson 9**: Index Structures (B-tree, GIN, GiST)
- **Lesson 10**: Relation Files and Tablespaces
- **Module 5**: MVCC and how xmin/xmax enable concurrency

---

## References

### Key Structures

| Structure | File | Purpose |
|-----------|------|---------|
| `PageHeaderData` | bufpage.h | Page header |
| `ItemIdData` | itemid.h | Line pointer |
| `HeapTupleHeaderData` | htup_details.h | Tuple header |
| `HeapTupleData` | htup.h | Complete tuple reference |

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `heap_insert()` | heapam.c | Insert a tuple |
| `heap_delete()` | heapam.c | Mark tuple deleted |
| `heap_update()` | heapam.c | Update a tuple |
| `heap_getnext()` | heapam.c | Get next tuple in scan |
| `PageAddItem()` | bufpage.c | Add item to page |
| `PageGetFreeSpace()` | bufpage.c | Calculate free space |
