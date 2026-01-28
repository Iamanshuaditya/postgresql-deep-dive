# Lesson 10: Tablespaces and Relation Files — Physical Organization of Data

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 2 — Storage Architecture |
| **Lesson** | 10 of 66 |
| **Estimated Time** | 45-60 minutes |
| **Prerequisites** | Lesson 8: Heap Storage and Page Layout |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Navigate the PostgreSQL data directory structure
2. Explain how tables and indexes are stored as files
3. Create and manage tablespaces for data placement
4. Map database objects to their physical files
5. Understand file segments and large table handling

### Key Terms

| Term | Definition |
|------|------------|
| **PGDATA** | The root directory containing all PostgreSQL data |
| **Tablespace** | A location on disk where database files can be stored |
| **Relfilenode** | The filename (OID) for a relation's data files |
| **Fork** | Different file types for a relation (main, fsm, vm, init) |
| **Segment** | 1GB file chunk for large tables |

---

## Introduction

Every table, index, and database you create in PostgreSQL becomes a file (or set of files) on disk. Understanding this physical organization helps you:

1. **Plan storage**: Put frequently-accessed data on fast disks
2. **Manage capacity**: Know where space is being used
3. **Recover data**: Navigate file structures for recovery
4. **Monitor I/O**: Correlate database activity with file system activity

PostgreSQL's storage is organized in a hierarchical structure:

```
PGDATA/                           (Database cluster root)
├── base/                         (Default tablespace for databases)
│   ├── 1/                        (Database OID 1: template1)
│   ├── 12345/                    (Database OID 12345: your_database)
│   │   ├── 16384                 (Table relfilenode)
│   │   ├── 16384_fsm             (Free Space Map)
│   │   ├── 16384_vm              (Visibility Map)
│   │   ├── 16385                 (Another table)
│   │   └── ...
│   └── ...
├── global/                       (Cluster-wide tables)
├── pg_tblspc/                    (Symbolic links to tablespaces)
├── pg_wal/                       (Write-Ahead Log files)
└── ...
```

> **Key Insight**: A table's data file is named by its **relfilenode** OID, not its table name. If you TRUNCATE or REINDEX, the relfilenode changes even though the table name stays the same.

---

## Conceptual Foundation

### The PGDATA Directory

The main subdirectories of PGDATA:

| Directory | Contents |
|-----------|----------|
| `base/` | Per-database directories (default tablespace) |
| `global/` | Cluster-wide shared tables (pg_database, pg_authid) |
| `pg_tblspc/` | Symbolic links to custom tablespaces |
| `pg_wal/` | Write-ahead log (WAL) files |
| `pg_xact/` | Transaction commit status (clog) |
| `pg_multixact/` | Multitransaction status |
| `pg_subtrans/` | Subtransaction info |
| `pg_stat/` | Permanent statistics |
| `pg_stat_tmp/` | Transient statistics |
| `pg_snapshots/` | Exported snapshots |
| `pg_logical/` | Logical replication data |
| `pg_commit_ts/` | Commit timestamps (if enabled) |

### Relation Files and Forks

Each relation (table, index, sequence) can have multiple files called **forks**:

```
16384           Main data fork (heap data or index data)
16384_fsm       Free Space Map (tracks available space per page)
16384_vm        Visibility Map (tracks all-visible pages)
16384_init      Init fork (for unlogged tables - empty template)
```

For large tables (>1GB), files are split into **segments**:

```
16384           First segment (0-1GB)
16384.1         Second segment (1-2GB)
16384.2         Third segment (2-3GB)
...
```

### Mapping Tables to Files

```sql
-- Find the physical location of a table
SELECT 
    pg_relation_filepath('users') AS filepath,
    pg_relation_filenode('users') AS filenode;
```

```
         filepath          | filenode
---------------------------+----------
 base/12345/16384          |    16384
```

The file can be found at: `$PGDATA/base/12345/16384`

### OID Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│  Table: public.users                                             │
│                                                                  │
│  pg_class.oid = 16384           (Relation OID)                  │
│  pg_class.relfilenode = 16384   (File OID - may differ!)        │
│  pg_class.reltablespace = 0     (0 = default tablespace)        │
│                                                                  │
│  File path: base/<db_oid>/<relfilenode>                         │
│            base/12345/16384                                      │
│                                                                  │
│  Database OID (from pg_database.datid)                          │
└─────────────────────────────────────────────────────────────────┘
```

**Important**: `oid` and `relfilenode` can differ after:
- TRUNCATE (gets new relfilenode)
- REINDEX (gets new relfilenode)
- CLUSTER (gets new relfilenode)
- ALTER TABLE ... SET TABLESPACE

---

## Deep Dive: Implementation Details

### Finding Physical Files

```sql
-- Get database OID
SELECT oid, datname FROM pg_database WHERE datname = current_database();

-- Get table details
SELECT 
    c.oid AS table_oid,
    c.relfilenode,
    c.relname,
    c.reltablespace,
    pg_relation_filepath(c.oid) AS filepath,
    pg_relation_size(c.oid) AS main_size,
    pg_relation_size(c.oid, 'fsm') AS fsm_size,
    pg_relation_size(c.oid, 'vm') AS vm_size
FROM pg_class c
WHERE c.relname = 'users';
```

### File Structure on Disk

```bash
# List files for a table
ls -la $PGDATA/base/12345/16384*

-rw------- 1 postgres postgres 8192 Jan 15 10:00 16384
-rw------- 1 postgres postgres 24576 Jan 15 10:00 16384_fsm
-rw------- 1 postgres postgres 8192 Jan 15 10:00 16384_vm
```

For a large table (>1GB):

```bash
ls -la $PGDATA/base/12345/24680*

-rw------- 1 postgres postgres 1073741824 Jan 15 10:00 24680
-rw------- 1 postgres postgres 1073741824 Jan 15 10:00 24680.1
-rw------- 1 postgres postgres  536870912 Jan 15 10:00 24680.2
-rw------- 1 postgres postgres     524288 Jan 15 10:00 24680_fsm
-rw------- 1 postgres postgres      32768 Jan 15 10:00 24680_vm
```

### Segment Size

The default segment size is 1GB (configurable at compile time):

```c
/* From src/include/pg_config_manual.h */
#define RELSEG_SIZE (1024 * 1024 * 1024 / BLCKSZ)  /* 131072 blocks = 1GB */
```

### Relfilenode Changes

```sql
-- Check current relfilenode
SELECT relfilenode FROM pg_class WHERE relname = 'users';
-- Returns: 16384

-- Truncate table
TRUNCATE users;

-- New relfilenode!
SELECT relfilenode FROM pg_class WHERE relname = 'users';
-- Returns: 16390

-- Old file deleted, new file created
```

This helps with:
- Instant TRUNCATE (allocate new file, mark old for deletion)
- Rewriting tables (create new, swap, delete old)
- Space reclamation

---

## Tablespaces

### What Are Tablespaces?

Tablespaces allow you to place database objects on specific disk locations:

```sql
-- Create a tablespace on a fast SSD
CREATE TABLESPACE fast_storage LOCATION '/mnt/ssd/pgdata';

-- Create a tablespace on cheap bulk storage
CREATE TABLESPACE archive_storage LOCATION '/mnt/hdd/pgdata';

-- Create table on specific tablespace
CREATE TABLE hot_data (id int, data text) TABLESPACE fast_storage;

-- Move existing table
ALTER TABLE cold_data SET TABLESPACE archive_storage;
```

### Tablespace Structure

```
$PGDATA/pg_tblspc/
├── 16389 -> /mnt/ssd/pgdata     (symlink to fast_storage)
└── 16390 -> /mnt/hdd/pgdata     (symlink to archive_storage)

/mnt/ssd/pgdata/
└── PG_16_202307071/             (version-specific directory)
    └── 12345/                   (database OID)
        ├── 16400                (table file)
        └── 16401                (index file)
```

### Default Tablespaces

```sql
-- Set database default tablespace
ALTER DATABASE mydb SET TABLESPACE fast_storage;

-- Set per-table default
CREATE TABLE logs (...) TABLESPACE archive_storage;

-- Check tablespace of a table
SELECT 
    t.spcname AS tablespace_name,
    pg_tablespace_location(t.oid) AS location
FROM pg_class c
JOIN pg_tablespace t ON c.reltablespace = t.oid
WHERE c.relname = 'users';
```

### Tablespace Use Cases

| Use Case | Strategy |
|----------|----------|
| Performance | Put hot tables/indexes on SSD |
| Capacity | Overflow to additional disks |
| Data tiering | Old data on cheap storage |
| Isolation | Separate databases on different disks |
| Backup | Different backup strategies per tablespace |

### Moving Data Between Tablespaces

```sql
-- Move table (blocks writes during move!)
ALTER TABLE users SET TABLESPACE archive_storage;

-- Move all objects in a tablespace
ALTER TABLE ALL IN TABLESPACE old_space SET TABLESPACE new_space;

-- Move database
ALTER DATABASE mydb SET TABLESPACE fast_storage;
```

**Warning**: Moving data copies files—can take a long time for large tables!

---

## Practical Examples

### Example 1: Exploring Data Directory

```bash
# Connect and find data directory
psql -c "SHOW data_directory;"
# /var/lib/postgresql/16/main

# List database directories
ls -la /var/lib/postgresql/16/main/base/

# Find your database's OID
psql -c "SELECT oid, datname FROM pg_database;"
```

### Example 2: Mapping Table to File

```sql
-- Create test table
CREATE TABLE file_test (id serial, data text);
INSERT INTO file_test SELECT i, 'data' FROM generate_series(1, 1000) i;

-- Find the file
SELECT pg_relation_filepath('file_test');
-- base/12345/16400

-- Check file size
SELECT 
    pg_size_pretty(pg_relation_size('file_test')) AS data_size,
    pg_size_pretty(pg_relation_size('file_test', 'fsm')) AS fsm_size,
    pg_size_pretty(pg_relation_size('file_test', 'vm')) AS vm_size;
```

### Example 3: Observing Relfilenode Changes

```sql
SELECT relname, relfilenode FROM pg_class WHERE relname = 'file_test';
-- file_test | 16400

TRUNCATE file_test;

SELECT relname, relfilenode FROM pg_class WHERE relname = 'file_test';
-- file_test | 16405  (changed!)
```

### Example 4: Creating and Using Tablespaces

```sql
-- Create tablespace (requires superuser and directory must exist)
CREATE TABLESPACE ssd_space LOCATION '/mnt/ssd/pg_data';

-- Create table on tablespace
CREATE TABLE fast_table (id int) TABLESPACE ssd_space;

-- Verify location
SELECT pg_relation_filepath('fast_table');
-- pg_tblspc/16390/PG_16_202307071/12345/16405

-- List all tablespaces
SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace;
```

### Example 5: Size Analysis

```sql
-- Total size of all forks
SELECT 
    relname,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
    pg_size_pretty(pg_relation_size(oid)) AS table_size,
    pg_size_pretty(pg_indexes_size(oid)) AS indexes_size
FROM pg_class
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace
ORDER BY pg_total_relation_size(oid) DESC
LIMIT 10;
```

---

## Performance Implications

### I/O Distribution

Spreading data across multiple disks can improve performance:

```sql
-- Hot tables on fast storage
CREATE TABLE active_orders (...) TABLESPACE fast_ssd;

-- Indexes on fast storage  
CREATE INDEX ON active_orders(customer_id) TABLESPACE fast_ssd;

-- Archive tables on slow storage
CREATE TABLE old_orders (...) TABLESPACE archive_hdd;
```

### File System Considerations

| Factor | Impact |
|--------|--------|
| File system type | XFS/ext4 recommended; avoid NFS for data |
| Block size | Match PostgreSQL page size (8KB) if possible |
| I/O scheduler | deadline/noop for SSD; cfq for HDD |
| Journaling | Data=ordered or data=writeback for ext4 |
| Mount options | noatime, barrier (or nobarrier for battery-backed RAID) |

### Large File Handling

The 1GB segment limit exists for:
- Compatibility with older file systems
- Easier backup/restore of individual segments
- Better parallelization of I/O

For very large tables, consider:
- Partitioning (each partition is a separate relfilenode)
- Table-level tablespace placement

---

## Hands-On Exercises

### Exercise 1: Exploring the Data Directory (Basic)

```sql
-- Find your data directory
SHOW data_directory;

-- Find database OID
SELECT oid, datname FROM pg_database WHERE datname = current_database();

-- List tables and their files
SELECT 
    relname,
    pg_relation_filepath(oid) AS path,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace;
```

Then verify in the shell:
```bash
ls -la $PGDATA/base/<database_oid>/
```

### Exercise 2: Tracking Relfilenode Changes (Intermediate)

```sql
CREATE TABLE filenode_test (id int);

SELECT relname, relfilenode, pg_relation_filepath(oid) 
FROM pg_class WHERE relname = 'filenode_test';

-- Operations that change relfilenode:
TRUNCATE filenode_test;
SELECT relfilenode FROM pg_class WHERE relname = 'filenode_test';

VACUUM FULL filenode_test;
SELECT relfilenode FROM pg_class WHERE relname = 'filenode_test';

CLUSTER filenode_test USING filenode_test_pkey;  -- (create index first)
SELECT relfilenode FROM pg_class WHERE relname = 'filenode_test';
```

### Exercise 3: Fork Sizes (Intermediate)

```sql
CREATE TABLE fork_test (id serial, data text);
INSERT INTO fork_test SELECT i, repeat('x', 100) FROM generate_series(1, 10000) i;

-- Initial sizes
SELECT 
    pg_relation_size('fork_test', 'main') AS main,
    pg_relation_size('fork_test', 'fsm') AS fsm,
    pg_relation_size('fork_test', 'vm') AS vm;

-- After VACUUM
VACUUM fork_test;
SELECT 
    pg_relation_size('fork_test', 'main') AS main,
    pg_relation_size('fork_test', 'fsm') AS fsm,
    pg_relation_size('fork_test', 'vm') AS vm;
```

### Exercise 4: Creating Tablespaces (Advanced)

```sql
-- Note: Requires superuser and writable directory

-- Create directory first (shell)
-- mkdir -p /tmp/pg_tablespace
-- chown postgres:postgres /tmp/pg_tablespace

-- Create tablespace
CREATE TABLESPACE test_space LOCATION '/tmp/pg_tablespace';

-- Create table in tablespace
CREATE TABLE tablespace_test (id int) TABLESPACE test_space;

-- Verify location
SELECT pg_relation_filepath('tablespace_test');

-- Move table back to default
ALTER TABLE tablespace_test SET TABLESPACE pg_default;

-- Cleanup
DROP TABLE tablespace_test;
DROP TABLESPACE test_space;
```

---

## Key Takeaways

1. **Tables are files**: Each table is stored in `base/<db_oid>/<relfilenode>`.

2. **Relfilenode ≠ table name**: The file OID can change with TRUNCATE, VACUUM FULL, CLUSTER.

3. **Multiple forks per relation**: Main (data), FSM (free space), VM (visibility).

4. **1GB segments**: Large tables are split into multiple files.

5. **Tablespaces control placement**: Put data on specific disks for performance or capacity.

6. **Moving data copies files**: ALTER TABLE SET TABLESPACE can take a long time.

7. **pg_tblspc uses symlinks**: Custom tablespaces are linked from the data directory.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/storage/smgr/md.c` | Magnetic disk storage manager |
| `src/backend/catalog/storage.c` | Relation file creation |
| `src/backend/commands/tablespace.c` | Tablespace commands |
| `src/include/storage/smgr.h` | Storage manager interface |

### Documentation

- [PostgreSQL Docs: Database File Layout](https://www.postgresql.org/docs/current/storage-file-layout.html)
- [PostgreSQL Docs: Tablespaces](https://www.postgresql.org/docs/current/manage-ag-tablespaces.html)
- [PostgreSQL Docs: CREATE TABLESPACE](https://www.postgresql.org/docs/current/sql-createtablespace.html)

### Next Lessons

- **Lesson 11**: The Buffer Manager
- **Lesson 12**: Other Index Types (GIN, GiST, BRIN)
- **Module 4**: Write-Ahead Logging

---

## References

### Key System Catalogs

| Catalog | Purpose |
|---------|---------|
| `pg_class` | Relation metadata including relfilenode |
| `pg_database` | Database OIDs |
| `pg_tablespace` | Tablespace definitions |
| `pg_namespace` | Schema information |

### Key Functions

| Function | Purpose |
|----------|---------|
| `pg_relation_filepath(oid)` | Get relative file path |
| `pg_relation_filenode(oid)` | Get relfilenode |
| `pg_relation_size(oid, fork)` | Size of specific fork |
| `pg_total_relation_size(oid)` | Total size including indexes |
| `pg_tablespace_location(oid)` | Tablespace directory |
