# Lesson 28: Logical Replication — Selective, Cross-Version Data Streaming

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 13 — Replication |
| **Lesson** | 28 of 66 |
| **Estimated Time** | 75-90 minutes |
| **Prerequisites** | Lesson 27: Streaming Replication |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain how logical replication differs from streaming replication
2. Set up publications and subscriptions
3. Understand the logical decoding process
4. Handle conflicts and limitations
5. Use logical replication for migrations and data distribution

### Key Terms

| Term | Definition |
|------|------------|
| **Logical Replication** | Table-level, row-based replication using SQL-level changes |
| **Publication** | Set of tables/operations to replicate from publisher |
| **Subscription** | Connection from subscriber that pulls changes |
| **Logical Decoding** | Converting WAL into logical change events |
| **Replication Identity** | How rows are identified (usually primary key) |
| **Output Plugin** | Converts decoded WAL to specific format |

---

## Introduction

### Physical vs Logical Replication

```
┌─────────────────────────────────────────────────────────────────┐
│  Physical (Streaming) Replication                               │
│  ─────────────────────────────                                  │
│  • Replicates: Raw WAL bytes                                    │
│  • Granularity: Entire database cluster                         │
│  • Replica: Exact binary copy, read-only                        │
│  • Versions: Must be same PostgreSQL version                    │
│  • Use: High availability, read replicas                        │
│                                                                  │
│  Logical Replication                                             │
│  ───────────────────                                            │
│  • Replicates: Row-level changes (INSERT, UPDATE, DELETE)       │
│  • Granularity: Specific tables                                 │
│  • Replica: Independent database, can be read-write             │
│  • Versions: Can differ (within supported range)                │
│  • Use: Migration, selective sync, data distribution            │
└─────────────────────────────────────────────────────────────────┘
```

### How Logical Replication Works

```
┌─────────────────────────────────────────────────────────────────┐
│  Logical Replication Flow                                        │
│                                                                  │
│  PUBLISHER                         SUBSCRIBER                   │
│  ┌────────────────────────┐       ┌────────────────────────┐   │
│  │ Client: INSERT...      │       │                        │   │
│  │         ↓              │       │                        │   │
│  │ ┌─────────────────┐    │       │                        │   │
│  │ │ WAL Generation  │    │       │                        │   │
│  │ └────────┬────────┘    │       │                        │   │
│  │          ↓             │       │                        │   │
│  │ ┌─────────────────┐    │       │ ┌─────────────────┐    │   │
│  │ │ Logical Decoding│    │       │ │ Logical Worker  │    │   │
│  │ │ (WAL → changes) │────┼──────▶│ │ (apply changes) │    │   │
│  │ └─────────────────┘    │       │ └────────┬────────┘    │   │
│  │          │             │       │          ↓             │   │
│  │ Sends: table=users     │       │ ┌─────────────────┐    │   │
│  │        INSERT (1,Bob)  │       │ │ Local INSERT    │    │   │
│  │                        │       │ │ INTO users...   │    │   │
│  │ Publication: pub1      │       │ └─────────────────┘    │   │
│  │ Tables: users, orders  │       │                        │   │
│  └────────────────────────┘       │ Subscription: sub1     │   │
│                                   └────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Setting Up Logical Replication

### Step 1: Configure Publisher

```sql
-- postgresql.conf on publisher
wal_level = logical  -- Required for logical decoding
max_replication_slots = 10
max_wal_senders = 10
```

### Step 2: Create Publication

```sql
-- On publisher

-- Publish all tables
CREATE PUBLICATION my_publication FOR ALL TABLES;

-- Publish specific tables
CREATE PUBLICATION user_pub FOR TABLE users, profiles;

-- Publish with specific operations
CREATE PUBLICATION orders_inserts FOR TABLE orders 
    WITH (publish = 'insert');  -- Only INSERTs

-- View publications
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;
```

### Step 3: Create Subscription

```sql
-- On subscriber

-- Ensure tables exist with same structure
CREATE TABLE users (LIKE users_on_publisher INCLUDING ALL);

-- Create subscription
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=publisher_host dbname=mydb user=replicator password=secret'
    PUBLICATION my_publication;

-- View subscriptions
SELECT * FROM pg_subscription;
SELECT * FROM pg_stat_subscription;
```

### Step 4: Monitor

```sql
-- On subscriber: Check subscription status
SELECT 
    subname,
    received_lsn,
    latest_end_lsn,
    last_msg_send_time,
    last_msg_receipt_time
FROM pg_stat_subscription;

-- On publisher: Check replication slots
SELECT * FROM pg_replication_slots WHERE slot_type = 'logical';
```

---

## Replication Identity

### Why Replication Identity Matters

Logical replication needs to identify which row to UPDATE/DELETE:

```sql
-- Default: Use primary key
ALTER TABLE users REPLICA IDENTITY DEFAULT;

-- Use a specific index (must be unique, not partial)
CREATE UNIQUE INDEX users_email_idx ON users(email);
ALTER TABLE users REPLICA IDENTITY USING INDEX users_email_idx;

-- Include all columns (inefficient but always works)
ALTER TABLE users REPLICA IDENTITY FULL;

-- No identity (only INSERT works)
ALTER TABLE users REPLICA IDENTITY NOTHING;
```

### Identity Requirements

| Identity | UPDATE/DELETE | Notes |
|----------|---------------|-------|
| DEFAULT (PK) | ✓ | Best option |
| USING INDEX | ✓ | Unique index required |
| FULL | ✓ | Sends all columns (slow) |
| NOTHING | ✗ | Only INSERT supported |

---

## Initial Data Sync

### Automatic Initial Copy

```sql
-- By default, subscription copies existing data
CREATE SUBSCRIPTION my_sub
    CONNECTION '...'
    PUBLICATION my_pub
    WITH (copy_data = true);  -- Default

-- Skip initial copy (data already synced)
CREATE SUBSCRIPTION my_sub
    CONNECTION '...'
    PUBLICATION my_pub
    WITH (copy_data = false);
```

### Copy Status

```sql
-- Check table sync status
SELECT 
    srsubid,
    srrelid::regclass,
    srsubstate,  -- i=init, d=data copy, s=sync, r=ready
    srsublsn
FROM pg_subscription_rel;
```

---

## Handling Conflicts

### Common Conflict Types

```
┌─────────────────────────────────────────────────────────────────┐
│  Conflict Scenarios                                              │
│                                                                  │
│  1. Primary Key Violation                                        │
│     Publisher: INSERT (id=1, ...)                               │
│     Subscriber: Already has id=1                                │
│     → ERROR: duplicate key                                      │
│                                                                  │
│  2. Row Not Found                                                │
│     Publisher: UPDATE WHERE id=1                                │
│     Subscriber: Row id=1 doesn't exist                          │
│     → ERROR (by default) or skip                                │
│                                                                  │
│  3. Foreign Key Violation                                        │
│     Publisher: INSERT order with customer_id=5                  │
│     Subscriber: customer_id=5 doesn't exist                     │
│     → ERROR: foreign key constraint                             │
└─────────────────────────────────────────────────────────────────┘
```

### Conflict Resolution

```sql
-- Subscription stops on error by default
-- Check for errors:
SELECT * FROM pg_stat_subscription;

-- View replication worker logs
-- (errors logged to PostgreSQL logs)

-- Skip a transaction (dangerous!)
-- 1. Find the stuck LSN
SELECT * FROM pg_stat_subscription;

-- 2. Skip to next transaction
ALTER SUBSCRIPTION my_sub SET (disable_on_error = true);
ALTER SUBSCRIPTION my_sub SKIP (lsn = '0/12345678');
ALTER SUBSCRIPTION my_sub ENABLE;
```

### Prevent Conflicts

1. **Don't write to replicated tables on subscriber**
2. **Ensure referential integrity is maintained**
3. **Use sequences with different ranges**
4. **Delay subscriber writes until sync complete**

---

## Publication Options

### Publish Specific Operations

```sql
-- Only replicate inserts (good for append-only tables)
CREATE PUBLICATION inserts_only FOR TABLE logs
    WITH (publish = 'insert');

-- Replicate inserts and updates, not deletes
CREATE PUBLICATION no_deletes FOR TABLE orders
    WITH (publish = 'insert, update');

-- All operations (default)
CREATE PUBLICATION all_ops FOR TABLE users
    WITH (publish = 'insert, update, delete, truncate');
```

### Add/Remove Tables

```sql
-- Add table to publication
ALTER PUBLICATION my_pub ADD TABLE new_table;

-- Remove table
ALTER PUBLICATION my_pub DROP TABLE old_table;

-- Set specific tables
ALTER PUBLICATION my_pub SET TABLE users, orders, products;
```

### Row Filters (PG 15+)

```sql
-- Only replicate rows matching condition
CREATE PUBLICATION active_users FOR TABLE users
    WHERE (active = true);

-- Column list (PG 15+)
CREATE PUBLICATION partial_users FOR TABLE users (id, email)
    WHERE (active = true);
```

---

## Use Cases

### Use Case 1: Version Upgrade

```sql
-- Old version (publisher)
CREATE PUBLICATION upgrade_pub FOR ALL TABLES;

-- New version (subscriber)
CREATE SUBSCRIPTION upgrade_sub
    CONNECTION 'host=old_server ...'
    PUBLICATION upgrade_pub;

-- Wait for sync, then switch applications to new server
```

### Use Case 2: Partial Replication

```sql
-- Headquarters: Full database
-- Branch offices: Only their data

CREATE PUBLICATION branch_nyc FOR TABLE orders
    WHERE (branch = 'NYC');

CREATE PUBLICATION branch_la FOR TABLE orders
    WHERE (branch = 'LA');
```

### Use Case 3: Data Aggregation

```sql
-- Multiple source databases → Central warehouse
-- Source 1
CREATE PUBLICATION sales_pub FOR TABLE daily_sales;

-- Warehouse
CREATE SUBSCRIPTION source1_sub CONNECTION '...' PUBLICATION sales_pub;
CREATE SUBSCRIPTION source2_sub CONNECTION '...' PUBLICATION sales_pub;
-- Data from all sources lands in same table
```

---

## Limitations

### What's NOT Replicated

| Feature | Replicated? |
|---------|-------------|
| Table data (INSERT/UPDATE/DELETE) | ✓ |
| DDL (CREATE/ALTER/DROP) | ✗ |
| Sequences | ✗ (values only, not definitions) |
| Large objects | ✗ |
| Materialized views | ✗ |
| Foreign tables | ✗ |
| Partitioned table (pre-PG 13) | ✗ |
| TRUNCATE (pre-PG 11) | ✗ |

### Schema Changes

```sql
-- DDL must be applied manually on both sides!

-- On publisher
ALTER TABLE users ADD COLUMN phone text;

-- On subscriber (before or after, depending on direction)
ALTER TABLE users ADD COLUMN phone text;

-- Then refresh publication/subscription if needed
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;
```

---

## Monitoring

### Publisher Monitoring

```sql
-- Replication slots for logical replication
SELECT 
    slot_name,
    plugin,
    slot_type,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots
WHERE slot_type = 'logical';

-- Publication stats
SELECT * FROM pg_stat_publication;
```

### Subscriber Monitoring

```sql
-- Subscription status
SELECT 
    subname,
    pid,
    received_lsn,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_subscription;

-- Per-table sync status
SELECT 
    srsubid,
    srrelid::regclass AS table_name,
    CASE srsubstate
        WHEN 'i' THEN 'initializing'
        WHEN 'd' THEN 'copying data'
        WHEN 's' THEN 'synchronized'
        WHEN 'r' THEN 'ready'
    END AS state
FROM pg_subscription_rel;
```

---

## Hands-On Exercises

### Exercise 1: Basic Logical Replication (Basic)

```sql
-- On Publisher
CREATE TABLE log_test (id serial PRIMARY KEY, msg text);
CREATE PUBLICATION log_pub FOR TABLE log_test;

-- On Subscriber  
CREATE TABLE log_test (id serial PRIMARY KEY, msg text);
CREATE SUBSCRIPTION log_sub
    CONNECTION 'host=publisher_host dbname=mydb user=rep'
    PUBLICATION log_pub;

-- Test: Insert on publisher
INSERT INTO log_test (msg) VALUES ('test message');

-- Verify on subscriber
SELECT * FROM log_test;
```

### Exercise 2: Selective Replication (Intermediate)

```sql
-- Publish only INSERTs for audit table
CREATE PUBLICATION audit_pub FOR TABLE audit_log
    WITH (publish = 'insert');

-- Subscriber receives only new entries, not updates/deletes
```

### Exercise 3: Handle a Conflict (Advanced)

```sql
-- Create situation: duplicate key
-- Publisher has id=1, subscriber already has id=1

-- Check subscription status
SELECT * FROM pg_stat_subscription;

-- View errors in logs
-- Fix: Delete conflicting row or skip transaction
```

---

## Key Takeaways

1. **Logical replication is table-level**: Unlike physical which is cluster-level.

2. **Uses logical decoding**: WAL → row changes.

3. **Cross-version compatible**: Enables rolling upgrades.

4. **Publication + Subscription model**: Flexible pub/sub pattern.

5. **Conflicts must be handled**: No automatic resolution.

6. **DDL not replicated**: Schema changes are manual.

7. **Replication identity required**: For UPDATE/DELETE.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/replication/logical/` | Logical replication |
| `src/backend/replication/pgoutput.c` | Output plugin |
| `src/backend/replication/logical/decode.c` | Logical decoding |

### Documentation

- [PostgreSQL Docs: Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [PostgreSQL Docs: Publications](https://www.postgresql.org/docs/current/sql-createpublication.html)
- [PostgreSQL Docs: Subscriptions](https://www.postgresql.org/docs/current/sql-createsubscription.html)

---

## References

### Key Objects

| Object | Purpose |
|--------|---------|
| pg_publication | Publication definitions |
| pg_publication_tables | Tables in publications |
| pg_subscription | Subscription definitions |
| pg_subscription_rel | Per-table sync status |
| pg_stat_subscription | Subscription statistics |

### Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| wal_level | replica | Must be 'logical' |
| max_replication_slots | 10 | Total slots |
| max_logical_replication_workers | 4 | Apply workers |
| max_sync_workers_per_subscription | 2 | Sync workers |
