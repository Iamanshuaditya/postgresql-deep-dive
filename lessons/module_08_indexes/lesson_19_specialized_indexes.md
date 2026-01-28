# Lesson 19: GIN, GiST, and Specialized Indexes — Beyond B-tree

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 8 — Index Internals |
| **Lesson** | 19 of 66 |
| **Estimated Time** | 90-120 minutes |
| **Prerequisites** | Lesson 9: B-tree Index Structure |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain when to use GIN, GiST, BRIN, and Hash indexes
2. Understand the internal structure of each index type
3. Choose the right index type for your data and queries
4. Create and maintain specialized indexes effectively
5. Recognize performance trade-offs of each index type

### Key Terms

| Term | Definition |
|------|------------|
| **GIN** | Generalized Inverted Index—for multi-valued data |
| **GiST** | Generalized Search Tree—for complex data types |
| **BRIN** | Block Range INdex—for naturally ordered data |
| **Hash** | Hash-based index—for equality lookups |
| **Operator Class** | Defines operators an index can support |

---

## Introduction

B-tree indexes are PostgreSQL's default, but they're not optimal for all use cases:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Index Type Selection Guide                                                   │
│                                                                               │
│  B-TREE (default)                        GIN                                  │
│  ─────────────────                       ───                                  │
│  - Scalar comparisons (<, =, >)          - Full-text search                  │
│  - Range queries                         - Arrays (contains, overlaps)       │
│  - Equality                              - JSONB containment                 │
│  - Pattern prefix (LIKE 'foo%')          - Multi-valued columns              │
│                                                                               │
│  GiST                                    BRIN                                │
│  ────                                    ────                                │
│  - Geometric data (points, boxes)        - Large tables with natural order   │
│  - Range types (overlap, contains)       - Time-series data                  │
│  - Full-text (alternative to GIN)        - Append-only tables                │
│  - Nearest neighbor (KNN)                - Tiny index, sequential scan fast  │
│                                                                               │
│  HASH                                    SP-GiST                             │
│  ────                                    ──────                              │
│  - Equality only (no range)              - Partitioned search trees          │
│  - Very large values                     - IP addresses, phone numbers       │
│  - Smaller than B-tree for =             - Quadtrees, k-d trees              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## GIN (Generalized Inverted Index)

### When to Use GIN

- **Full-text search**: `tsvector @@ tsquery`
- **Array queries**: `array_column @> ARRAY[1,2]`
- **JSONB containment**: `jsonb_column @> '{"key": "value"}'`
- **Any multi-valued column**

### How GIN Works

```
┌─────────────────────────────────────────────────────────────────┐
│  GIN Index Structure (Inverted Index)                            │
│                                                                  │
│  Document 1: "PostgreSQL is a database"                         │
│  Document 2: "PostgreSQL is powerful"                           │
│  Document 3: "A database can be powerful"                       │
│                                                                  │
│  GIN Index:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Key (Word)     →  Posting List (Document IDs)              │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │  "a"            →  [1, 3]                                   │ │
│  │  "be"           →  [3]                                      │ │
│  │  "can"          →  [3]                                      │ │
│  │  "database"     →  [1, 3]                                   │ │
│  │  "is"           →  [1, 2]                                   │ │
│  │  "postgresql"   →  [1, 2]                                   │ │
│  │  "powerful"     →  [2, 3]                                   │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Query: "postgresql & database"                                 │
│  → Find "postgresql" → [1, 2]                                   │
│  → Find "database"   → [1, 3]                                   │
│  → Intersect         → [1]                                      │
└─────────────────────────────────────────────────────────────────┘
```

### GIN Example: Full-Text Search

```sql
-- Create table with tsvector column
CREATE TABLE articles (
    id serial PRIMARY KEY,
    title text,
    body text,
    search_vector tsvector GENERATED ALWAYS AS (
        to_tsvector('english', title || ' ' || body)
    ) STORED
);

-- Create GIN index on search vector
CREATE INDEX articles_search_idx ON articles USING gin(search_vector);

-- Query using index
SELECT * FROM articles 
WHERE search_vector @@ to_tsquery('english', 'postgresql & performance');
```

### GIN Example: JSONB

```sql
-- JSONB containment queries
CREATE TABLE events (
    id serial PRIMARY KEY,
    data jsonb
);

-- GIN index for JSONB
CREATE INDEX events_data_idx ON events USING gin(data);

-- Queries that use the index
SELECT * FROM events WHERE data @> '{"type": "click"}';
SELECT * FROM events WHERE data ? 'user_id';  -- Key exists
```

### GIN Example: Arrays

```sql
CREATE TABLE products (
    id serial PRIMARY KEY,
    name text,
    tags text[]
);

CREATE INDEX products_tags_idx ON products USING gin(tags);

-- Find products with specific tags
SELECT * FROM products WHERE tags @> ARRAY['electronics', 'sale'];
SELECT * FROM products WHERE tags && ARRAY['electronics', 'fashion'];  -- Overlaps
```

### GIN Pending List

GIN has a "fastupdate" mode where new entries go to a pending list:

```sql
-- Check pending list size
SELECT * FROM pg_stat_all_indexes WHERE indexrelname LIKE '%gin%';

-- Disable for write-heavy (batch updates)
CREATE INDEX ... WITH (fastupdate = off);

-- Force cleanup of pending list
VACUUM table_name;
```

---

## GiST (Generalized Search Tree)

### When to Use GiST

- **Geometric queries**: Points, boxes, polygons
- **Range overlap**: `int4range`, `tsrange`
- **Full-text search**: Alternative to GIN (smaller, slower search)
- **Nearest neighbor**: `ORDER BY distance LIMIT 1`
- **Exclusion constraints**: Prevent overlapping ranges

### How GiST Works

```
┌─────────────────────────────────────────────────────────────────┐
│  GiST Index Structure (Bounding Boxes)                          │
│                                                                  │
│  Geometric data: points in 2D space                            │
│                                                                  │
│          ┌────────────────────────────────┐                     │
│          │         ROOT NODE              │                     │
│          │  [  entire bounding box  ]     │                     │
│          └────────────┬───────────────────┘                     │
│          ┌────────────┴───────────────────┐                     │
│          ▼                                ▼                     │
│  ┌───────────────────┐      ┌───────────────────┐              │
│  │   LEFT CHILD      │      │   RIGHT CHILD     │              │
│  │ [bbox for points] │      │ [bbox for points] │              │
│  └─────────┬─────────┘      └─────────┬─────────┘              │
│            ▼                          ▼                         │
│     ┌──────┴───────┐           ┌──────┴───────┐                │
│     │  Leaf: points│           │  Leaf: points│                │
│     │  (1,2) (3,4) │           │  (10,12)     │                │
│     └──────────────┘           └──────────────┘                │
│                                                                  │
│  Query: Find points near (5,5)                                  │
│  → Check bounding boxes at each level                          │
│  → Prune branches that can't contain result                    │
└─────────────────────────────────────────────────────────────────┘
```

### GiST Example: Geometric Data

```sql
-- PostGIS points (requires PostGIS extension)
CREATE TABLE locations (
    id serial PRIMARY KEY,
    name text,
    coords geometry(Point, 4326)
);

CREATE INDEX locations_coords_idx ON locations USING gist(coords);

-- Find nearby locations
SELECT * FROM locations 
WHERE ST_DWithin(coords, ST_MakePoint(-122.4, 37.8)::geography, 1000);

-- Nearest neighbor (KNN)
SELECT * FROM locations
ORDER BY coords <-> ST_MakePoint(-122.4, 37.8)::geometry
LIMIT 5;
```

### GiST Example: Range Types

```sql
CREATE TABLE reservations (
    id serial PRIMARY KEY,
    room_id int,
    during tsrange
);

CREATE INDEX reservations_during_idx ON reservations USING gist(during);

-- Find overlapping reservations
SELECT * FROM reservations 
WHERE during && tsrange('2024-01-15 14:00', '2024-01-15 16:00');

-- Exclusion constraint: No overlapping reservations for same room
ALTER TABLE reservations ADD CONSTRAINT no_overlap 
    EXCLUDE USING gist (room_id WITH =, during WITH &&);
```

### GiST vs GIN for Full-Text

| Aspect | GIN | GiST |
|--------|-----|------|
| Index size | Larger | Smaller |
| Build time | Slower | Faster |
| Search speed | Faster | Slower |
| Updates | Slower (pending list) | Faster |
| Best for | Read-heavy | Write-heavy |

---

## BRIN (Block Range Index)

### When to Use BRIN

- **Naturally ordered data**: Created date, ID, timestamp
- **Very large tables**: Billions of rows
- **Append-only patterns**: Log tables, time-series
- **When small index size matters**: BRIN is TINY

### How BRIN Works

```
┌─────────────────────────────────────────────────────────────────┐
│  BRIN Index Structure                                            │
│                                                                  │
│  Table: events (sorted by timestamp)                            │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Block 0-127:  2024-01-01 to 2024-01-03                     │ │
│  │ Block 128-255: 2024-01-03 to 2024-01-05                    │ │
│  │ Block 256-383: 2024-01-05 to 2024-01-07                    │ │
│  │ ...                                                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  BRIN Index (very small!):                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Block Range 0-127:   min=2024-01-01, max=2024-01-03       │ │
│  │ Block Range 128-255: min=2024-01-03, max=2024-01-05       │ │
│  │ Block Range 256-383: min=2024-01-05, max=2024-01-07       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Query: WHERE timestamp BETWEEN '2024-01-04' AND '2024-01-04'  │
│  → Check BRIN: Only blocks 128-383 might contain data          │
│  → Skip blocks 0-127 entirely                                  │
│  → Read and filter blocks 128-383                              │
└─────────────────────────────────────────────────────────────────┘
```

### BRIN Example

```sql
-- Time-series table
CREATE TABLE sensor_data (
    id bigserial,
    sensor_id int,
    reading float,
    recorded_at timestamptz DEFAULT now()
);

-- BRIN index (128 pages per range by default)
CREATE INDEX sensor_data_time_idx ON sensor_data 
    USING brin(recorded_at);

-- Smaller ranges for more precision (more index space)
CREATE INDEX sensor_data_time_idx ON sensor_data 
    USING brin(recorded_at) WITH (pages_per_range = 32);

-- Query uses BRIN to skip irrelevant blocks
EXPLAIN ANALYZE
SELECT * FROM sensor_data 
WHERE recorded_at BETWEEN '2024-01-15' AND '2024-01-16';
```

### BRIN Size Comparison

```sql
-- Comparison on 100 million row table
SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass))
FROM pg_indexes WHERE tablename = 'sensor_data';

-- Results might show:
-- B-tree: 2 GB
-- BRIN:   128 KB  (15,000x smaller!)
```

### BRIN Limitations

- **Works only for correlated data**: If data isn't ordered, BRIN won't help
- **Doesn't prevent reading false positives**: Still scans candidate blocks
- **No unique constraint support**: Can't enforce uniqueness

---

## Hash Index

### When to Use Hash

- **Equality-only queries**: `WHERE column = 'value'`
- **Very large values**: Long strings, UUIDs
- **No ordering needed**: Can't use for `<`, `>`, `ORDER BY`

### How Hash Works

```
┌─────────────────────────────────────────────────────────────────┐
│  Hash Index Structure                                            │
│                                                                  │
│  Values:                                                         │
│  'alpha' → hash → 1234 → bucket 4 → heap TID (0,1)             │
│  'beta'  → hash → 5678 → bucket 8 → heap TID (0,2)             │
│  'gamma' → hash → 1234 → bucket 4 → heap TID (0,3) (collision) │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Bucket 0: [empty]                                           │ │
│  │ Bucket 1: [empty]                                           │ │
│  │ ...                                                         │ │
│  │ Bucket 4: [(0,1), (0,3)]  ← Hash collision, both stored    │ │
│  │ ...                                                         │ │
│  │ Bucket 8: [(0,2)]                                           │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Query: WHERE column = 'alpha'                                  │
│  → hash('alpha') = 1234 → bucket 4                              │
│  → Read all TIDs in bucket 4: (0,1), (0,3)                     │
│  → Fetch heap tuples and verify (handle collision)             │
└─────────────────────────────────────────────────────────────────┘
```

### Hash Example

```sql
CREATE TABLE users (
    id serial PRIMARY KEY,
    email text UNIQUE,
    token text
);

-- Hash index for token lookup (equality only)
CREATE INDEX users_token_idx ON users USING hash(token);

-- This query uses the hash index
EXPLAIN SELECT * FROM users WHERE token = 'abc123xyz';

-- This query CANNOT use hash index (range)
EXPLAIN SELECT * FROM users WHERE token > 'abc';  -- Seq Scan!
```

### Hash vs B-tree

| Aspect | Hash | B-tree |
|--------|------|--------|
| Equality (=) | Yes | Yes |
| Range (<, >, BETWEEN) | No | Yes |
| ORDER BY | No | Yes |
| Size | Sometimes smaller | Standard |
| WAL logged | Yes (since PG 10) | Yes |
| Use case | Only = on large values | General purpose |

---

## SP-GiST (Space-Partitioned GiST)

### When to Use SP-GiST

- **Hierarchical data**: IP addresses, phone numbers
- **Quadtrees**: 2D spatial indexes
- **Radix trees**: String prefixes

### SP-GiST Example

```sql
-- IP address range queries
CREATE TABLE ip_blocks (
    id serial,
    network inet
);

CREATE INDEX ip_blocks_idx ON ip_blocks USING spgist(network);

SELECT * FROM ip_blocks WHERE network >>= '192.168.1.100';
```

---

## Choosing the Right Index

### Decision Matrix

| Query Type | Best Index |
|------------|------------|
| `column = value` | B-tree or Hash |
| `column < value` | B-tree |
| `column BETWEEN a AND b` | B-tree or BRIN |
| `column LIKE 'prefix%'` | B-tree |
| `column LIKE '%anywhere%'` | GIN with pg_trgm |
| `tsvector @@ tsquery` | GIN (or GiST) |
| `array @> ARRAY[...]` | GIN |
| `jsonb @> '{...}'` | GIN |
| `geometry && geometry` | GiST |
| `point <-> point ORDER BY LIMIT` | GiST |
| `range && range` | GiST |
| `timestamp > x` (ordered data) | BRIN |

### Index Size/Speed Trade-offs

```
                 Index Size
                     │
     Small ──────────┼─────────── Large
                     │
        BRIN         │         GIN
         │           │          │
         │           │          │
    Scan │           │          │ Fast
   Fast  │     B-tree│          │ Lookups
         │       GiST│          │
         │    Hash   │          │
                     │
```

---

## Hands-On Exercises

### Exercise 1: GIN for Full-Text (Basic)

```sql
CREATE TABLE documents (
    id serial PRIMARY KEY,
    content text
);

INSERT INTO documents (content) VALUES
    ('PostgreSQL is a powerful database'),
    ('MySQL is another database'),
    ('Oracle database is enterprise-grade');

-- Add GIN index
ALTER TABLE documents ADD COLUMN content_vector tsvector;
UPDATE documents SET content_vector = to_tsvector('english', content);
CREATE INDEX doc_search_idx ON documents USING gin(content_vector);

-- Search
EXPLAIN ANALYZE SELECT * FROM documents 
WHERE content_vector @@ to_tsquery('database & powerful');
```

### Exercise 2: BRIN for Time-Series (Intermediate)

```sql
-- Create large time-series table
CREATE TABLE metrics (
    id bigserial,
    ts timestamptz,
    value float
);

-- Insert ordered data
INSERT INTO metrics (ts, value)
SELECT 
    '2024-01-01'::timestamptz + (n || ' seconds')::interval,
    random()
FROM generate_series(1, 1000000) n;

-- Compare B-tree vs BRIN
CREATE INDEX metrics_btree ON metrics (ts);
CREATE INDEX metrics_brin ON metrics USING brin(ts);

SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass))
FROM pg_indexes WHERE tablename = 'metrics';

EXPLAIN ANALYZE SELECT * FROM metrics 
WHERE ts BETWEEN '2024-01-05' AND '2024-01-06';
```

### Exercise 3: GiST for Ranges (Intermediate)

```sql
CREATE TABLE events (
    id serial PRIMARY KEY,
    room text,
    during tsrange
);

CREATE INDEX events_during_idx ON events USING gist(during);

INSERT INTO events (room, during) VALUES
    ('A', '[2024-01-15 09:00, 2024-01-15 10:00)'),
    ('A', '[2024-01-15 10:00, 2024-01-15 11:00)'),
    ('B', '[2024-01-15 09:00, 2024-01-15 12:00)');

-- Find overlapping events
EXPLAIN ANALYZE SELECT * FROM events 
WHERE during && '[2024-01-15 09:30, 2024-01-15 10:30)';

-- Add exclusion constraint
ALTER TABLE events ADD CONSTRAINT no_double_book
    EXCLUDE USING gist (room WITH =, during WITH &&);
```

---

## Key Takeaways

1. **GIN for multi-valued**: Full-text, arrays, JSONB containment.

2. **GiST for complex types**: Geometry, ranges, nearest-neighbor.

3. **BRIN for ordered data**: Tiny index, huge tables, time-series.

4. **Hash for equality only**: When you never need range queries.

5. **Default to B-tree**: Unless you have specific needs above.

6. **Size matters**: BRIN can be 1000x smaller than B-tree.

7. **Operator classes define capabilities**: Check what operators each index supports.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/access/gin/` | GIN implementation |
| `src/backend/access/gist/` | GiST implementation |
| `src/backend/access/brin/` | BRIN implementation |
| `src/backend/access/hash/` | Hash implementation |

### Documentation

- [PostgreSQL Docs: Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [PostgreSQL Docs: GIN Indexes](https://www.postgresql.org/docs/current/gin.html)
- [PostgreSQL Docs: GiST Indexes](https://www.postgresql.org/docs/current/gist.html)
- [PostgreSQL Docs: BRIN Indexes](https://www.postgresql.org/docs/current/brin.html)

---

## References

### Index Method Comparison

| Index | Operators | Best For |
|-------|-----------|----------|
| B-tree | <, <=, =, >=, > | General purpose |
| Hash | = | Equality only |
| GiST | <<, >>, @>, <@, && | Geometric, ranges |
| GIN | @>, @@, ? | Arrays, FTS, JSONB |
| BRIN | <, <=, =, >=, > | Large ordered tables |
| SP-GiST | <<, >>, @> | IP, text patterns |
