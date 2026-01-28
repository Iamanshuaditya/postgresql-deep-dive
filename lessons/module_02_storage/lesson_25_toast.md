# Lesson 25: TOAST — The Oversized-Attribute Storage Technique

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 2 — Storage Architecture |
| **Lesson** | 25 of 66 |
| **Estimated Time** | 45-60 minutes |
| **Prerequisites** | Lesson 8: Heap Storage and Page Layout |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain why TOAST exists and how it works
2. Understand the different TOAST storage strategies
3. Identify when and how data gets TOASTed
4. Query TOAST tables and measure TOAST overhead
5. Configure TOAST behavior for optimal performance

### Key Terms

| Term | Definition |
|------|------------|
| **TOAST** | The Oversized-Attribute Storage Technique |
| **TOAST table** | Separate table storing oversized values |
| **Varlena** | Variable-length data type (text, bytea, jsonb) |
| **Inline storage** | Value stored directly in heap tuple |
| **External storage** | Value stored in TOAST table |
| **Compression** | LZ compression applied before storage |

---

## Introduction

PostgreSQL pages are 8KB, but columns can store much larger values. How?

```
┌─────────────────────────────────────────────────────────────────┐
│  The TOAST Problem and Solution                                  │
│                                                                  │
│  Problem:                                                        │
│  • Pages are 8KB                                                 │
│  • A tuple must fit in a page                                   │
│  • But text column could be 1GB!                                │
│                                                                  │
│  Solution: TOAST                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Main Table                    TOAST Table                   │ │
│  │ ┌──────────────────────┐     ┌──────────────────────────┐ │ │
│  │ │ id: 1                │     │ chunk_id: 12345          │ │ │
│  │ │ name: "Alice"        │     │ chunk_seq: 0             │ │ │
│  │ │ document: [TOAST PTR]│────▶│ chunk_data: [compressed] │ │ │
│  │ │           ↑          │     │                          │ │ │
│  │ │      18-byte pointer │     │ chunk_seq: 1             │ │ │
│  │ └──────────────────────┘     │ chunk_data: [compressed] │ │ │
│  │                              │                          │ │ │
│  │ Large values stored          │ chunk_seq: 2             │ │ │
│  │ out-of-line, compressed      │ chunk_data: [compressed] │ │ │
│  │                              └──────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

TOAST = **T**he **O**versized-**A**ttribute **S**torage **T**echnique

---

## When TOAST Kicks In

### The TOAST Threshold

```
┌─────────────────────────────────────────────────────────────────┐
│  TOAST Decision Process                                          │
│                                                                  │
│  Tuple size after header: ~1960 bytes (TOAST_TUPLE_THRESHOLD)   │
│                                                                  │
│  If tuple > threshold:                                          │
│    1. Find largest TOASTable column                             │
│    2. Try compression first                                     │
│    3. If still too big, move to TOAST table                    │
│    4. Repeat until tuple fits                                   │
│                                                                  │
│  Compression target: ~2KB (TOAST_TUPLE_TARGET)                  │
│                                                                  │
│  Note: Individual values > ~1GB require TOAST                   │
└─────────────────────────────────────────────────────────────────┘
```

### Which Data Types Use TOAST?

| Category | Types | TOASTable? |
|----------|-------|------------|
| Fixed-width | integer, bigint, boolean | No |
| Variable-width | text, varchar, bytea | Yes |
| Composite | jsonb, arrays, hstore | Yes |
| Large objects | text, bytea | Yes |

---

## TOAST Storage Strategies

### The Four Strategies

```sql
-- View column storage strategies
SELECT 
    attname,
    attstorage
FROM pg_attribute
WHERE attrelid = 'my_table'::regclass
  AND attnum > 0;
```

| Strategy | Code | Behavior |
|----------|------|----------|
| PLAIN | p | No TOAST (fixed-width types) |
| EXTENDED | x | Compress, then externalize if still big (default) |
| EXTERNAL | e | Externalize without compression |
| MAIN | m | Compress, externalize only as last resort |

### Changing Storage Strategy

```sql
-- Store without compression (faster writes, bigger storage)
ALTER TABLE documents ALTER COLUMN content SET STORAGE EXTERNAL;

-- Prefer inline storage (faster reads for small values)
ALTER TABLE logs ALTER COLUMN message SET STORAGE MAIN;

-- Reset to default
ALTER TABLE documents ALTER COLUMN content SET STORAGE EXTENDED;
```

### When to Use Each Strategy

| Strategy | Use Case |
|----------|----------|
| EXTENDED | Default. Best overall balance |
| EXTERNAL | Pre-compressed data (images, already zipped) |
| MAIN | Small values that compress well, frequent access |
| PLAIN | Not available for varlena types |

---

## Inside the TOAST Table

### TOAST Table Structure

Every TOASTable table gets a companion TOAST table:

```sql
-- Find TOAST table for a relation
SELECT 
    c.relname AS main_table,
    t.relname AS toast_table,
    pg_size_pretty(pg_relation_size(c.oid)) AS main_size,
    pg_size_pretty(pg_relation_size(t.oid)) AS toast_size
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
WHERE c.relname = 'documents';

-- TOAST table naming: pg_toast.pg_toast_<oid>
```

### TOAST Table Columns

```sql
-- Structure of a TOAST table
-- (You can't query pg_toast tables directly without superuser)

-- chunk_id:   OID of the original value
-- chunk_seq:  Sequence number (0, 1, 2...)
-- chunk_data: Actual data chunk (~2000 bytes each)
```

### Chunk Size

```
┌─────────────────────────────────────────────────────────────────┐
│  TOAST Chunking                                                  │
│                                                                  │
│  Original value: 50KB                                           │
│                                                                  │
│  After compression: 15KB                                        │
│                                                                  │
│  Stored as:                                                     │
│  Chunk 0: ~2000 bytes                                           │
│  Chunk 1: ~2000 bytes                                           │
│  Chunk 2: ~2000 bytes                                           │
│  ...                                                            │
│  Chunk 7: ~1000 bytes (remainder)                              │
│                                                                  │
│  Each chunk = separate row in TOAST table                       │
│  Fetching value = reading all chunks in sequence                │
└─────────────────────────────────────────────────────────────────┘
```

---

## TOAST and Query Performance

### The Detoasting Process

```sql
-- This query must detoast document:
SELECT id, document FROM documents WHERE id = 1;

-- This query does NOT detoast (column not used):
SELECT id, title FROM documents WHERE id = 1;

-- Even in WHERE clause, no detoast:
SELECT id FROM documents WHERE id = 1;
-- The document column is never read!
```

### Partial Detoasting

```sql
-- Only detoasts first 100 chars:
SELECT id, left(document, 100) FROM documents;

-- PostgreSQL is smart: reads only needed chunks
```

### Performance Implications

```
┌─────────────────────────────────────────────────────────────────┐
│  TOAST Performance Trade-offs                                    │
│                                                                  │
│  Advantages:                                                     │
│  ✓ Main table rows stay small → faster scans                   │
│  ✓ Compression saves disk space                                 │
│  ✓ Only detoast when actually needed                           │
│                                                                  │
│  Disadvantages:                                                  │
│  ✗ Extra I/O to read TOAST table                               │
│  ✗ Decompression CPU overhead                                  │
│  ✗ Random I/O pattern for TOAST chunks                        │
│                                                                  │
│  Best practices:                                                 │
│  • Don't SELECT * on tables with large columns                 │
│  • Only select columns you need                                 │
│  • Consider EXTERNAL storage for pre-compressed data           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Measuring TOAST Usage

### Table and TOAST Sizes

```sql
-- Size breakdown
SELECT
    pg_size_pretty(pg_relation_size('documents')) AS main_table,
    pg_size_pretty(pg_relation_size(
        (SELECT reltoastrelid FROM pg_class WHERE relname = 'documents')
    )) AS toast_table,
    pg_size_pretty(pg_total_relation_size('documents')) AS total;
```

### Check if Values are TOASTed

```sql
-- Using pg_column_size
SELECT 
    id,
    pg_column_size(document) AS on_disk_size,
    octet_length(document) AS uncompressed_size,
    CASE 
        WHEN pg_column_size(document) < 2000 THEN 'inline'
        ELSE 'toasted'
    END AS storage
FROM documents
LIMIT 10;
```

### Compression Ratio

```sql
-- Estimate compression ratio
SELECT 
    AVG(pg_column_size(document)::float / 
        NULLIF(octet_length(document), 0)) AS compression_ratio
FROM documents
WHERE octet_length(document) > 2000;

-- < 1.0 means compression is working
-- ~0.3 means 70% space savings
```

---

## TOAST and Indexes

### You Can't Index TOAST Values Directly

```sql
-- This doesn't work for large text:
CREATE INDEX ON documents (content);
-- ERROR: index row size exceeds maximum

-- Solutions:
-- 1. Index a prefix
CREATE INDEX ON documents (left(content, 100));

-- 2. Index a hash
CREATE INDEX ON documents (md5(content));

-- 3. Use full-text search
CREATE INDEX ON documents USING gin(to_tsvector('english', content));
```

### Expression Indexes

```sql
-- Index computed values from TOAST columns
CREATE INDEX ON documents (length(content));
CREATE INDEX ON documents ((content LIKE '%error%'));
```

---

## Practical Examples

### Example 1: View TOAST Behavior

```sql
-- Create table with potentially TOASTed column
CREATE TABLE toast_test (
    id serial PRIMARY KEY,
    small_text text,
    large_text text
);

-- Insert data
INSERT INTO toast_test (small_text, large_text)
SELECT 
    'small_' || i,
    repeat('x', 10000)  -- 10KB text
FROM generate_series(1, 100) i;

-- Check sizes
SELECT 
    pg_size_pretty(pg_relation_size('toast_test')) AS main_size,
    pg_size_pretty(pg_relation_size(
        (SELECT reltoastrelid FROM pg_class WHERE relname = 'toast_test')
    )) AS toast_size;
```

### Example 2: Compare Storage Strategies

```sql
-- Create tables with different strategies
CREATE TABLE toast_extended (data text);
CREATE TABLE toast_external (data text);

ALTER TABLE toast_external ALTER COLUMN data SET STORAGE EXTERNAL;

-- Insert same data
INSERT INTO toast_extended SELECT repeat('hello world ', 1000);
INSERT INTO toast_external SELECT repeat('hello world ', 1000);

-- Compare sizes
SELECT 'extended', pg_column_size(data) FROM toast_extended
UNION ALL
SELECT 'external', pg_column_size(data) FROM toast_external;
-- Extended will be smaller (compressed)
```

### Example 3: Query Without Detoasting

```sql
-- This is fast (no detoast):
SELECT id, small_text FROM toast_test WHERE id < 10;

-- This is slower (must detoast):
SELECT id, large_text FROM toast_test WHERE id < 10;

-- Measure with EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS) 
SELECT id, small_text FROM toast_test;

EXPLAIN (ANALYZE, BUFFERS) 
SELECT id, large_text FROM toast_test;
-- Note: "Buffers: ... read" will be higher for large_text
```

---

## TOAST and VACUUM

### Cleaning TOAST Tables

```sql
-- VACUUM on main table also vacuums TOAST
VACUUM documents;

-- Check TOAST table stats
SELECT 
    schemaname, relname, n_dead_tup
FROM pg_stat_all_tables
WHERE relname LIKE 'pg_toast_%';
```

### TOAST and Bloat

TOAST tables can bloat just like regular tables:

```sql
-- Check TOAST table bloat
SELECT 
    t.relname AS toast_table,
    pg_size_pretty(pg_relation_size(t.oid)) AS size,
    t.n_dead_tup AS dead_tuples
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
JOIN pg_stat_all_tables s ON s.relid = t.oid
WHERE c.relnamespace = 'public'::regnamespace;
```

---

## Hands-On Exercises

### Exercise 1: Identify TOASTed Columns (Basic)

```sql
-- Check which columns can be TOASTed
SELECT 
    a.attname,
    t.typname,
    a.attstorage,
    CASE a.attstorage
        WHEN 'p' THEN 'PLAIN'
        WHEN 'e' THEN 'EXTERNAL'
        WHEN 'm' THEN 'MAIN'
        WHEN 'x' THEN 'EXTENDED'
    END AS strategy
FROM pg_attribute a
JOIN pg_type t ON t.oid = a.atttypid
WHERE a.attrelid = 'toast_test'::regclass
  AND a.attnum > 0;
```

### Exercise 2: Measure Compression (Intermediate)

```sql
-- Insert compressible vs incompressible data
INSERT INTO toast_test (small_text, large_text)
VALUES ('small', repeat('a', 10000));  -- Very compressible

INSERT INTO toast_test (small_text, large_text)
VALUES ('small', (SELECT string_agg(chr((random()*25)::int + 65), '') 
                  FROM generate_series(1, 10000)));  -- Random = less compressible

-- Compare compression
SELECT 
    id,
    octet_length(large_text) AS original,
    pg_column_size(large_text) AS on_disk,
    round(pg_column_size(large_text)::numeric / 
          octet_length(large_text) * 100, 1) AS percent
FROM toast_test
ORDER BY id DESC LIMIT 2;
```

### Exercise 3: TOAST Table Exploration (Advanced)

```sql
-- Get TOAST table OID
SELECT reltoastrelid::regclass FROM pg_class WHERE relname = 'toast_test';

-- Examine TOAST index (superuser required)
SELECT 
    c.relname AS toast_table,
    i.relname AS toast_index
FROM pg_class c
JOIN pg_index ix ON c.oid = ix.indrelid
JOIN pg_class i ON i.oid = ix.indexrelid
WHERE c.relname LIKE 'pg_toast_%';
```

---

## Key Takeaways

1. **TOAST handles oversized values**: Automatic, transparent storage.

2. **Compression happens first**: Then out-of-line storage if needed.

3. **Different strategies available**: EXTENDED, EXTERNAL, MAIN, PLAIN.

4. **Detoasting is lazy**: Only happens when you access the column.

5. **Don't SELECT ***: Avoid detoasting columns you don't need.

6. **TOAST tables need maintenance**: VACUUM cleans them too.

7. **Measure with pg_column_size**: See actual on-disk storage.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/access/heap/tuptoaster.c` | TOAST implementation |
| `src/backend/access/common/toast_internals.c` | TOAST internals |
| `src/include/access/toast_internals.h` | TOAST definitions |

### Documentation

- [PostgreSQL Docs: TOAST](https://www.postgresql.org/docs/current/storage-toast.html)
- [PostgreSQL Docs: Column Storage](https://www.postgresql.org/docs/current/sql-altertable.html)

---

## References

### TOAST Thresholds

| Threshold | Value | Purpose |
|-----------|-------|---------|
| TOAST_TUPLE_THRESHOLD | ~2KB | Start considering TOAST |
| TOAST_TUPLE_TARGET | ~2KB | Target after compression |
| TOAST_MAX_CHUNK_SIZE | ~2000 bytes | Size of each chunk |

### Storage Strategy Codes

| Code | Name | Behavior |
|------|------|----------|
| p | PLAIN | Never TOAST |
| x | EXTENDED | Compress + out-of-line |
| e | EXTERNAL | Out-of-line only |
| m | MAIN | Compress, avoid out-of-line |
