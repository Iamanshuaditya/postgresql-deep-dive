# Lesson 30: Extensions — Extending PostgreSQL's Capabilities

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 14 — Advanced Features |
| **Lesson** | 30 of 66 |
| **Estimated Time** | 60-75 minutes |
| **Prerequisites** | Lesson 1: How PostgreSQL Processes a Query |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Understand how PostgreSQL's extension system works
2. Install and manage common extensions
3. Explore extension internals and control files
4. Know when and how to use popular extensions
5. Understand the basics of creating custom extensions

### Key Terms

| Term | Definition |
|------|------------|
| **Extension** | Packaged bundle of SQL objects and optional C code |
| **Control File** | Metadata file (.control) defining extension |
| **SQL Script** | Installation script creating objects |
| **PGXS** | PostgreSQL Extension Building Infrastructure |
| **contrib** | Extensions bundled with PostgreSQL source |

---

## Introduction

Extensions are PostgreSQL's modular way to add functionality:

```
┌─────────────────────────────────────────────────────────────────┐
│  PostgreSQL Extension System                                     │
│                                                                  │
│  Core PostgreSQL                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ • SQL Engine                                                 ││
│  │ • Storage Engine                                            ││
│  │ • Query Planner                                             ││
│  │ • Built-in types and functions                              ││
│  └─────────────────────────────────────────────────────────────┘│
│                        ↑ Extensions plug in                     │
│  ┌─────────────────────┼───────────────────────────────────────┐│
│  │        ┌────────────┴───────────┐                          ││
│  │        ↓            ↓           ↓                          ││
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐                     ││
│  │ │ PostGIS  │ │ pg_stat  │ │pageinspect│                    ││
│  │ │ (geo)    │ │ statements│ │ (debug)  │                    ││
│  │ └──────────┘ └──────────┘ └──────────┘                     ││
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐                     ││
│  │ │ pg_trgm  │ │  uuid-   │ │ pgcrypto │                     ││
│  │ │ (fuzzy)  │ │  ossp    │ │ (crypto) │                     ││
│  │ └──────────┘ └──────────┘ └──────────┘                     ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## Managing Extensions

### List Available Extensions

```sql
-- Extensions installed on system (available to create)
SELECT name, default_version, comment 
FROM pg_available_extensions 
ORDER BY name;

-- Extensions currently installed in this database
SELECT extname, extversion FROM pg_extension;
```

### Install an Extension

```sql
-- Basic installation
CREATE EXTENSION pg_trgm;

-- Specific version
CREATE EXTENSION pg_trgm VERSION '1.6';

-- In specific schema
CREATE EXTENSION pg_trgm SCHEMA public;

-- With dependencies
CREATE EXTENSION postgis CASCADE;  -- Installs required extensions
```

### Update an Extension

```sql
-- Upgrade to latest version
ALTER EXTENSION pg_trgm UPDATE;

-- Upgrade to specific version
ALTER EXTENSION pg_trgm UPDATE TO '1.6';
```

### Remove an Extension

```sql
-- Drop extension and dependent objects
DROP EXTENSION pg_trgm CASCADE;

-- Check what would be dropped first
SELECT * FROM pg_depend WHERE refobjid = 'pg_trgm'::regextension;
```

---

## Extension File Structure

### Where Extensions Live

```bash
# Find extension directory
pg_config --sharedir
# Usually: /usr/share/postgresql/16/

# Extension files:
/usr/share/postgresql/16/extension/
├── pg_trgm.control           # Metadata
├── pg_trgm--1.6.sql          # Installation script
├── pg_trgm--1.5--1.6.sql     # Upgrade script
└── ...

# Shared libraries (for C extensions):
/usr/lib/postgresql/16/lib/
├── pg_trgm.so
└── ...
```

### Control File Example

```ini
# pg_trgm.control
comment = 'text similarity measurement and index searching based on trigrams'
default_version = '1.6'
module_pathname = '$libdir/pg_trgm'
relocatable = true
```

### SQL Script Example

```sql
-- pg_trgm--1.6.sql (simplified)
CREATE FUNCTION similarity(text, text)
RETURNS float4
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT IMMUTABLE PARALLEL SAFE;

CREATE OPERATOR % (
    LEFTARG = text,
    RIGHTARG = text,
    PROCEDURE = similarity_op
);

CREATE OPERATOR CLASS gin_trgm_ops
    FOR TYPE text USING gin AS ...;
```

---

## Popular Contrib Extensions

### pg_stat_statements — Query Statistics

```sql
-- Install
CREATE EXTENSION pg_stat_statements;

-- Configure in postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
-- Requires restart!

-- Usage: Find slow queries
SELECT 
    query,
    calls,
    total_exec_time / 1000 AS total_seconds,
    mean_exec_time AS avg_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

### pageinspect — Low-Level Page Examination

```sql
CREATE EXTENSION pageinspect;

-- View heap page header
SELECT * FROM page_header(get_raw_page('users', 0));

-- View tuple details
SELECT * FROM heap_page_items(get_raw_page('users', 0));

-- View B-tree page
SELECT * FROM bt_page_items('users_pkey', 1);
```

### pg_trgm — Trigram Text Matching

```sql
CREATE EXTENSION pg_trgm;

-- Similarity search
SELECT similarity('hello', 'hallo');  -- 0.5

-- Fuzzy matching with index
CREATE INDEX users_name_trgm ON users USING gin (name gin_trgm_ops);

-- Query (uses index)
SELECT * FROM users WHERE name % 'Jhon';  -- Finds "John"
SELECT * FROM users WHERE name ILIKE '%smith%';  -- Also uses index!
```

### pgcrypto — Cryptographic Functions

```sql
CREATE EXTENSION pgcrypto;

-- Password hashing
INSERT INTO users (email, password_hash)
VALUES ('user@example.com', crypt('secret123', gen_salt('bf')));

-- Verify password
SELECT * FROM users 
WHERE email = 'user@example.com'
  AND password_hash = crypt('secret123', password_hash);

-- Generate UUID
SELECT gen_random_uuid();

-- Encryption
SELECT pgp_sym_encrypt('secret data', 'mypassword');
```

### uuid-ossp — UUID Generation

```sql
CREATE EXTENSION "uuid-ossp";

-- Generate various UUID versions
SELECT uuid_generate_v1();   -- Time-based
SELECT uuid_generate_v4();   -- Random
SELECT uuid_generate_v5(uuid_ns_url(), 'https://example.com');  -- Name-based

-- As primary key
CREATE TABLE items (
    id uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
    name text
);
```

### btree_gist / btree_gin — Additional Index Support

```sql
CREATE EXTENSION btree_gist;

-- Now can use GiST for scalar types (for exclusion constraints)
CREATE TABLE reservations (
    room int,
    during tsrange,
    EXCLUDE USING gist (room WITH =, during WITH &&)
);
```

---

## Third-Party Extensions

### PostGIS — Geospatial

```sql
CREATE EXTENSION postgis;

-- Store locations
CREATE TABLE places (
    id serial PRIMARY KEY,
    name text,
    location geography(Point, 4326)
);

INSERT INTO places (name, location) VALUES
    ('NYC', ST_Point(-74.006, 40.7128)),
    ('LA', ST_Point(-118.2437, 34.0522));

-- Find nearby places
SELECT name FROM places
WHERE ST_DWithin(location, ST_Point(-73.9, 40.7)::geography, 50000);
```

### TimescaleDB — Time-Series

```sql
CREATE EXTENSION timescaledb;

-- Create hypertable (partitioned by time)
CREATE TABLE metrics (
    time timestamptz NOT NULL,
    device_id int,
    value float
);
SELECT create_hypertable('metrics', 'time');

-- Automatic time-based partitioning and compression
```

### Citus — Distributed PostgreSQL

```sql
CREATE EXTENSION citus;

-- Distribute table across nodes
SELECT create_distributed_table('orders', 'customer_id');
```

---

## Extension Dependencies

### Viewing Dependencies

```sql
-- What extensions does this extension require?
SELECT 
    e.extname,
    d.refobjid::regextension AS depends_on
FROM pg_extension e
JOIN pg_depend d ON d.objid = e.oid
WHERE d.deptype = 'e'
  AND d.classid = 'pg_extension'::regclass;

-- Objects created by an extension
SELECT 
    pg_describe_object(classid, objid, objsubid) AS object
FROM pg_depend
WHERE refobjid = 'pg_trgm'::regextension
  AND deptype = 'e';
```

---

## Creating a Simple Extension

### Step 1: Control File

```ini
# my_extension.control
comment = 'My custom extension'
default_version = '1.0'
relocatable = true
```

### Step 2: SQL Script

```sql
-- my_extension--1.0.sql
CREATE FUNCTION my_hello(name text)
RETURNS text
AS $$
    SELECT 'Hello, ' || name || '!';
$$ LANGUAGE SQL IMMUTABLE;

CREATE FUNCTION my_add(a int, b int)
RETURNS int
AS $$
    SELECT a + b;
$$ LANGUAGE SQL IMMUTABLE;
```

### Step 3: Install

```bash
# Copy files to extension directory
cp my_extension.control /usr/share/postgresql/16/extension/
cp my_extension--1.0.sql /usr/share/postgresql/16/extension/
```

### Step 4: Use

```sql
CREATE EXTENSION my_extension;
SELECT my_hello('World');  -- 'Hello, World!'
```

---

## Extension Best Practices

### Schema Placement

```sql
-- Create in dedicated schema
CREATE SCHEMA extensions;
CREATE EXTENSION pg_trgm SCHEMA extensions;

-- Set search path to include it
SET search_path = public, extensions;
```

### Version Tracking

```sql
-- Check installed versions
SELECT extname, extversion FROM pg_extension;

-- Update all extensions
DO $$
DECLARE
    ext record;
BEGIN
    FOR ext IN SELECT extname FROM pg_extension LOOP
        EXECUTE format('ALTER EXTENSION %I UPDATE', ext.extname);
    END LOOP;
END $$;
```

### Documentation

```sql
-- View extension documentation
\dx+ pg_trgm
-- Shows objects created by extension
```

---

## Hands-On Exercises

### Exercise 1: Install and Use pageinspect (Basic)

```sql
CREATE EXTENSION pageinspect;

-- Examine a page
CREATE TABLE inspect_test (id int, data text);
INSERT INTO inspect_test VALUES (1, 'hello'), (2, 'world');

-- View page header
SELECT * FROM page_header(get_raw_page('inspect_test', 0));

-- View tuples
SELECT lp, t_xmin, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('inspect_test', 0));
```

### Exercise 2: Fuzzy Search with pg_trgm (Intermediate)

```sql
CREATE EXTENSION pg_trgm;

CREATE TABLE products (id serial, name text);
INSERT INTO products (name) VALUES 
    ('iPhone 15 Pro'), ('Samsung Galaxy'), ('Google Pixel'),
    ('Apple MacBook'), ('Dell XPS'), ('HP Spectre');

CREATE INDEX products_name_trgm ON products USING gin (name gin_trgm_ops);

-- Fuzzy search
SELECT * FROM products WHERE name % 'iphone';
SELECT * FROM products WHERE name % 'galxy';  -- Typo still works!
SELECT *, similarity(name, 'mackbook') AS sim 
FROM products ORDER BY sim DESC;
```

### Exercise 3: Query Analysis with pg_stat_statements (Advanced)

```sql
-- Requires: shared_preload_libraries = 'pg_stat_statements'
-- and restart

CREATE EXTENSION pg_stat_statements;

-- Run some queries
SELECT * FROM products WHERE name % 'test';
SELECT count(*) FROM products;

-- Analyze
SELECT query, calls, mean_exec_time, rows
FROM pg_stat_statements
WHERE query LIKE '%products%'
ORDER BY total_exec_time DESC;
```

---

## Key Takeaways

1. **Extensions package functionality**: SQL objects + optional C code.

2. **CREATE EXTENSION installs**: Runs version-specific SQL script.

3. **Control file defines metadata**: Version, dependencies, relocatable.

4. **Contrib extensions are bundled**: pg_stat_statements, pageinspect, etc.

5. **Third-party extensions extend capabilities**: PostGIS, TimescaleDB, Citus.

6. **Manage versions carefully**: ALTER EXTENSION UPDATE.

7. **Extensions create trackable dependencies**: Easy to see what they install.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/commands/extension.c` | Extension commands |
| `contrib/` | Bundled extension source |
| `src/include/catalog/pg_extension.h` | Extension catalog |

### Documentation

- [PostgreSQL Docs: Extensions](https://www.postgresql.org/docs/current/extend-extensions.html)
- [PostgreSQL Docs: Contrib Modules](https://www.postgresql.org/docs/current/contrib.html)
- [PGXN - Extension Network](https://pgxn.org/)

---

## References

### Popular Extensions

| Extension | Purpose |
|-----------|---------|
| pg_stat_statements | Query statistics |
| pageinspect | Page examination |
| pg_trgm | Trigram matching |
| pgcrypto | Cryptography |
| uuid-ossp | UUID generation |
| PostGIS | Geospatial |
| TimescaleDB | Time-series |
| pg_partman | Partition management |
| pg_repack | Online table reorganization |

### Extension Catalog Tables

| Table | Purpose |
|-------|---------|
| pg_extension | Installed extensions |
| pg_available_extensions | Available extensions |
| pg_available_extension_versions | All versions |
| pg_depend (deptype='e') | Extension dependencies |
