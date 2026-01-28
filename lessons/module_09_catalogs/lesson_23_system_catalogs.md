# Lesson 23: System Catalogs — PostgreSQL's Metadata Store

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 9 — System Catalogs |
| **Lesson** | 23 of 66 |
| **Estimated Time** | 60-75 minutes |
| **Prerequisites** | Lesson 1: How PostgreSQL Processes a Query |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Understand how PostgreSQL stores metadata about database objects
2. Query key system catalogs effectively
3. Explain the relationship between catalogs and information_schema
4. Use catalog queries for database introspection
5. Understand how OIDs connect different catalog entries

### Key Terms

| Term | Definition |
|------|------------|
| **System Catalog** | Internal tables storing database metadata |
| **OID** | Object Identifier—unique ID for database objects |
| **pg_class** | Catalog of all relations (tables, indexes, views, etc.) |
| **pg_attribute** | Catalog of all columns |
| **pg_type** | Catalog of all data types |
| **pg_namespace** | Catalog of schemas |
| **information_schema** | SQL-standard views over catalogs |

---

## Introduction

Everything in PostgreSQL is stored in tables—including information ABOUT tables! These special tables are called **system catalogs**.

```
┌─────────────────────────────────────────────────────────────────┐
│  System Catalogs: PostgreSQL's Self-Description                 │
│                                                                  │
│  When you CREATE TABLE users (id int, name text):               │
│                                                                  │
│  PostgreSQL updates:                                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ pg_class    → New row for 'users' table                     ││
│  │ pg_attribute → New rows for 'id' and 'name' columns         ││
│  │ pg_type     → References existing int4 and text types       ││
│  │ pg_namespace → References the schema containing users       ││
│  │ pg_depend   → Tracks dependencies                           ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  This metadata is used by:                                       │
│  • Parser (is 'users' a valid table name?)                      │
│  • Analyzer (what type is the 'id' column?)                     │
│  • Planner (what indexes exist on this table?)                  │
│  • Executor (where is the data stored?)                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Core Catalogs

### pg_class — All Relations

Every table, index, view, sequence, and other relation has a row in `pg_class`:

```sql
SELECT 
    oid,           -- Object ID
    relname,       -- Relation name
    relnamespace,  -- Schema OID (references pg_namespace)
    reltype,       -- Row type OID (references pg_type)
    relkind,       -- r=table, i=index, v=view, S=sequence, etc.
    reltuples,     -- Estimated row count
    relpages,      -- Number of pages (8KB blocks)
    relfilenode,   -- Physical file name
    reltablespace  -- Tablespace OID
FROM pg_class
WHERE relname = 'users';
```

**Common relkind values:**

| relkind | Meaning |
|---------|---------|
| `r` | Ordinary table |
| `i` | Index |
| `S` | Sequence |
| `v` | View |
| `m` | Materialized view |
| `c` | Composite type |
| `t` | TOAST table |
| `f` | Foreign table |
| `p` | Partitioned table |
| `I` | Partitioned index |

### pg_attribute — All Columns

Every column of every table has a row in `pg_attribute`:

```sql
SELECT 
    attrelid,      -- Table OID (references pg_class.oid)
    attname,       -- Column name
    atttypid,      -- Type OID (references pg_type.oid)
    attnum,        -- Column number (1, 2, 3...)
    attlen,        -- Fixed size or -1 for variable
    attnotnull,    -- NOT NULL constraint?
    atthasdef      -- Has default value?
FROM pg_attribute
WHERE attrelid = 'users'::regclass
  AND attnum > 0   -- Exclude system columns
  AND NOT attisdropped;
```

**System columns** (attnum < 0):

| attnum | Name | Purpose |
|--------|------|---------|
| -1 | tableoid | OID of containing table |
| -2 | cmax | Command ID for deletion |
| -3 | xmax | Transaction ID for deletion |
| -4 | cmin | Command ID for insertion |
| -5 | xmin | Transaction ID for insertion |
| -6 | ctid | Physical location (page, offset) |

### pg_type — All Data Types

Every data type has a row in `pg_type`:

```sql
SELECT 
    oid,
    typname,       -- Type name
    typnamespace,  -- Schema OID
    typlen,        -- Fixed size or -1 for varlena
    typtype,       -- b=base, c=composite, d=domain, e=enum
    typarray,      -- Array type OID
    typinput,      -- Input function
    typoutput      -- Output function
FROM pg_type
WHERE typname IN ('int4', 'text', 'varchar', 'timestamp');
```

### pg_namespace — Schemas

```sql
SELECT 
    oid,
    nspname,       -- Schema name
    nspowner       -- Owner user OID
FROM pg_namespace;

-- Common schemas:
-- pg_catalog: System catalogs
-- public: Default user schema
-- information_schema: SQL-standard views
-- pg_toast: TOAST tables
```

---

## OID: The Glue Between Catalogs

OIDs (Object Identifiers) connect catalog entries:

```
┌─────────────────────────────────────────────────────────────────┐
│  OID Relationships                                               │
│                                                                  │
│  pg_class (oid=16384)                                           │
│  ┌────────────────────┐                                         │
│  │ relname: 'users'   │                                         │
│  │ relnamespace: 2200 │───────▶ pg_namespace (oid=2200)        │
│  │ reltype: 16386     │         ┌──────────────────┐            │
│  └────────────────────┘         │ nspname: 'public'│            │
│           │                     └──────────────────┘            │
│           │                                                     │
│           ▼                                                     │
│  pg_attribute                                                   │
│  ┌────────────────────┐                                         │
│  │ attrelid: 16384    │◀─── References users table             │
│  │ attname: 'id'      │                                         │
│  │ atttypid: 23       │───────▶ pg_type (oid=23)               │
│  └────────────────────┘         ┌──────────────────┐            │
│                                 │ typname: 'int4'  │            │
│                                 └──────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### Casting to regclass/regtype

PostgreSQL provides convenient casts:

```sql
-- Table name to OID
SELECT 'users'::regclass::oid;  -- Returns 16384

-- OID to table name
SELECT 16384::regclass;  -- Returns 'users'

-- Type name to OID
SELECT 'integer'::regtype::oid;  -- Returns 23

-- OID to type name
SELECT 23::regtype;  -- Returns 'integer'
```

---

## Important Catalogs Reference

### Tables and Columns

| Catalog | Purpose |
|---------|---------|
| `pg_class` | All relations |
| `pg_attribute` | All columns |
| `pg_attrdef` | Column defaults |
| `pg_constraint` | Constraints (PK, FK, CHECK, UNIQUE) |
| `pg_index` | Index metadata |

### Types and Domains

| Catalog | Purpose |
|---------|---------|
| `pg_type` | All data types |
| `pg_enum` | Enum values |
| `pg_range` | Range type info |

### Functions and Operators

| Catalog | Purpose |
|---------|---------|
| `pg_proc` | Functions/procedures |
| `pg_operator` | Operators (+, -, etc.) |
| `pg_aggregate` | Aggregate functions |
| `pg_cast` | Type casts |

### Access Control

| Catalog | Purpose |
|---------|---------|
| `pg_authid` | Roles/users |
| `pg_auth_members` | Role membership |
| `pg_default_acl` | Default privileges |

### Statistics

| Catalog | Purpose |
|---------|---------|
| `pg_statistic` | Column statistics |
| `pg_stats` | Human-readable stats view |
| `pg_statistic_ext` | Extended statistics |

### Dependencies

| Catalog | Purpose |
|---------|---------|
| `pg_depend` | Object dependencies |
| `pg_shdepend` | Shared dependencies |

---

## Practical Queries

### List All Tables with Sizes

```sql
SELECT 
    n.nspname AS schema,
    c.relname AS table_name,
    c.reltuples::bigint AS row_estimate,
    pg_size_pretty(pg_relation_size(c.oid)) AS size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(c.oid) DESC;
```

### List All Columns of a Table

```sql
SELECT 
    a.attname AS column_name,
    t.typname AS data_type,
    a.attnotnull AS not_null,
    pg_get_expr(d.adbin, d.adrelid) AS default_value
FROM pg_attribute a
JOIN pg_type t ON t.oid = a.atttypid
LEFT JOIN pg_attrdef d ON d.adrelid = a.attrelid AND d.adnum = a.attnum
WHERE a.attrelid = 'users'::regclass
  AND a.attnum > 0
  AND NOT a.attisdropped
ORDER BY a.attnum;
```

### Find All Indexes on a Table

```sql
SELECT 
    i.relname AS index_name,
    a.amname AS index_type,
    pg_get_indexdef(i.oid) AS index_definition
FROM pg_class t
JOIN pg_index ix ON t.oid = ix.indrelid
JOIN pg_class i ON i.oid = ix.indexrelid
JOIN pg_am a ON a.oid = i.relam
WHERE t.relname = 'users';
```

### List All Foreign Keys

```sql
SELECT
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table,
    ccu.column_name AS foreign_column
FROM pg_constraint c
JOIN pg_class t ON c.conrelid = t.oid
JOIN pg_namespace n ON t.relnamespace = n.oid
JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(c.conkey)
JOIN pg_class ft ON c.confrelid = ft.oid
JOIN pg_attribute fa ON fa.attrelid = ft.oid AND fa.attnum = ANY(c.confkey)
WHERE c.contype = 'f';

-- Or use information_schema (simpler):
SELECT 
    tc.table_name, 
    kcu.column_name,
    ccu.table_name AS foreign_table,
    ccu.column_name AS foreign_column
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu 
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu 
    ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY';
```

### Find Unused Indexes

```sql
SELECT 
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## information_schema vs pg_catalog

### information_schema

- **SQL Standard**: Portable across databases
- **Simpler**: Easier to use
- **Slower**: Built on views over catalogs
- **Limited**: Not all PostgreSQL features exposed

```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'public';
```

### pg_catalog

- **PostgreSQL Specific**: Full access to internals
- **Faster**: Direct table access
- **Complete**: All PostgreSQL features
- **Complex**: Requires understanding OIDs

```sql
SELECT c.relname, a.attname, t.typname
FROM pg_class c
JOIN pg_attribute a ON a.attrelid = c.oid
JOIN pg_type t ON t.oid = a.atttypid
WHERE c.relnamespace = 'public'::regnamespace
  AND a.attnum > 0;
```

---

## How Catalogs Are Used Internally

### During Query Parsing

```c
/* Parser looks up table name in pg_class */
RangeVar *relation = makeRangeVar(schemaname, tablename, location);

/* Translates to: */
SELECT oid FROM pg_class 
WHERE relname = 'tablename' 
  AND relnamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'schemaname');
```

### During Query Planning

```c
/* Planner checks for indexes */
List *indexlist = RelationGetIndexList(relation);

/* Translates to: */
SELECT indexrelid FROM pg_index WHERE indrelid = relation_oid;

/* Planner reads statistics */
HeapTuple statsTuple = SearchSysCache(STATRELATTINH, ...);

/* Translates to: */
SELECT * FROM pg_statistic WHERE starelid = rel_oid AND staattnum = col_num;
```

---

## Hands-On Exercises

### Exercise 1: Explore pg_class (Basic)

```sql
-- View all user tables
SELECT relname, relkind, reltuples, relpages
FROM pg_class
WHERE relnamespace = 'public'::regnamespace
  AND relkind = 'r';

-- Compare with \dt output
\dt public.*
```

### Exercise 2: Trace a Column's Metadata (Intermediate)

```sql
-- Create a test table
CREATE TABLE catalog_test (
    id serial PRIMARY KEY,
    name varchar(100) NOT NULL,
    created_at timestamp DEFAULT now()
);

-- Find the table in pg_class
SELECT oid, relname, relnamespace, relfilenode
FROM pg_class WHERE relname = 'catalog_test';

-- Find columns in pg_attribute
SELECT attname, atttypid::regtype, attnotnull, atthasdef
FROM pg_attribute
WHERE attrelid = 'catalog_test'::regclass
  AND attnum > 0;

-- Find default values
SELECT a.attname, pg_get_expr(d.adbin, d.adrelid) AS default_expr
FROM pg_attribute a
JOIN pg_attrdef d ON d.adrelid = a.attrelid AND d.adnum = a.attnum
WHERE a.attrelid = 'catalog_test'::regclass;
```

### Exercise 3: Catalog Dependencies (Advanced)

```sql
-- What objects depend on a table?
SELECT 
    classid::regclass AS dependent_type,
    objid::regclass AS dependent_object,
    deptype
FROM pg_depend
WHERE refobjid = 'catalog_test'::regclass;

-- deptype meanings:
-- n = normal dependency
-- a = auto dependency (like index on table)
-- i = internal dependency
-- e = extension dependency
```

### Exercise 4: Build Your Own \d Command (Advanced)

```sql
-- Replicate psql's \d command output
SELECT 
    a.attname AS "Column",
    pg_catalog.format_type(a.atttypid, a.atttypmod) AS "Type",
    CASE WHEN a.attnotnull THEN 'not null' ELSE '' END AS "Nullable",
    pg_get_expr(d.adbin, d.adrelid) AS "Default"
FROM pg_attribute a
LEFT JOIN pg_attrdef d ON d.adrelid = a.attrelid AND d.adnum = a.attnum
WHERE a.attrelid = 'catalog_test'::regclass
  AND a.attnum > 0
  AND NOT a.attisdropped
ORDER BY a.attnum;
```

---

## Key Takeaways

1. **Everything is a table**: Metadata stored in system catalogs.

2. **pg_class is central**: Contains all tables, indexes, views, sequences.

3. **OIDs connect catalogs**: Use ::regclass for easy conversion.

4. **information_schema is portable**: pg_catalog is powerful.

5. **Catalogs drive query processing**: Parser, planner use them extensively.

6. **Statistics live in catalogs**: pg_statistic powers the planner.

7. **Dependencies are tracked**: pg_depend prevents orphaned objects.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/include/catalog/pg_class.h` | pg_class definition |
| `src/include/catalog/pg_attribute.h` | pg_attribute definition |
| `src/include/catalog/pg_type.h` | pg_type definition |
| `src/backend/catalog/` | Catalog management code |

### Documentation

- [PostgreSQL Docs: System Catalogs](https://www.postgresql.org/docs/current/catalogs.html)
- [PostgreSQL Docs: information_schema](https://www.postgresql.org/docs/current/information-schema.html)

---

## References

### Key Catalog Tables

| Catalog | Description | Key Columns |
|---------|-------------|-------------|
| pg_class | Relations | oid, relname, relkind |
| pg_attribute | Columns | attrelid, attname, atttypid |
| pg_type | Types | oid, typname, typlen |
| pg_namespace | Schemas | oid, nspname |
| pg_index | Indexes | indexrelid, indrelid, indkey |
| pg_constraint | Constraints | conname, contype, conrelid |
| pg_proc | Functions | oid, proname, proargtypes |
| pg_statistic | Statistics | starelid, staattnum |
