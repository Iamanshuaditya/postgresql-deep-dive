# Lesson 27: Streaming Replication — High Availability with Physical Replicas

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 13 — Replication |
| **Lesson** | 27 of 66 |
| **Estimated Time** | 90-120 minutes |
| **Prerequisites** | Lesson 14: Write-Ahead Logging (WAL) |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain how streaming replication works at the WAL level
2. Set up a primary-replica configuration
3. Understand synchronous vs asynchronous replication
4. Monitor replication lag and health
5. Perform failover and recovery operations

### Key Terms

| Term | Definition |
|------|------------|
| **Primary** | The read-write server (formerly "master") |
| **Replica/Standby** | Read-only server receiving WAL (formerly "slave") |
| **WAL Sender** | Primary process streaming WAL to replica |
| **WAL Receiver** | Replica process receiving WAL from primary |
| **Replication Slot** | Ensures WAL is retained until replica receives it |
| **Hot Standby** | Replica that accepts read queries |
| **Replication Lag** | Delay between primary and replica |

---

## Introduction

Streaming replication continuously ships WAL records from primary to replicas:

```
┌─────────────────────────────────────────────────────────────────┐
│  Streaming Replication Architecture                              │
│                                                                  │
│  PRIMARY SERVER                     REPLICA SERVER              │
│  ┌────────────────────────┐        ┌────────────────────────┐  │
│  │                        │        │                        │  │
│  │  Client Connections    │        │  Read-Only Queries     │  │
│  │  (read/write)          │        │  (hot standby)         │  │
│  │         │              │        │         ↑              │  │
│  │         ▼              │        │         │              │  │
│  │  ┌──────────────┐      │        │  ┌──────────────┐      │  │
│  │  │ PostgreSQL   │      │        │  │ PostgreSQL   │      │  │
│  │  │ Backend      │      │        │  │ Recovery     │      │  │
│  │  └──────┬───────┘      │        │  └──────▲───────┘      │  │
│  │         │ writes       │        │         │ applies      │  │
│  │         ▼              │        │         │              │  │
│  │  ┌──────────────┐      │        │  ┌──────────────┐      │  │
│  │  │ WAL Buffers  │      │        │  │ WAL Buffers  │      │  │
│  │  └──────┬───────┘      │        │  └──────▲───────┘      │  │
│  │         │              │        │         │              │  │
│  │         ▼              │        │         │              │  │
│  │  ┌──────────────┐      │   TCP  │  ┌──────────────┐      │  │
│  │  │ WAL Sender   │──────┼───────▶│  │ WAL Receiver │      │  │
│  │  └──────────────┘      │  WAL   │  └──────────────┘      │  │
│  │         │              │ stream │         │              │  │
│  │         ▼              │        │         ▼              │  │
│  │  ┌──────────────┐      │        │  ┌──────────────┐      │  │
│  │  │ pg_wal/      │      │        │  │ pg_wal/      │      │  │
│  │  │ WAL Files    │      │        │  │ WAL Files    │      │  │
│  │  └──────────────┘      │        │  └──────────────┘      │  │
│  └────────────────────────┘        └────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Setting Up Streaming Replication

### Step 1: Configure the Primary

```sql
-- postgresql.conf on primary
wal_level = replica              -- Minimum for replication
max_wal_senders = 10             -- Max concurrent replicas
max_replication_slots = 10       -- For replication slots
wal_keep_size = 1GB              -- WAL to keep for lagging replicas
hot_standby = on                 -- Allow read queries on replica

-- Optional: synchronous replication
synchronous_commit = on
synchronous_standby_names = 'replica1'
```

### Step 2: Create Replication User

```sql
-- On primary
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret';

-- pg_hba.conf: Allow replication connections
-- host    replication     replicator    192.168.1.0/24    scram-sha-256
```

### Step 3: Create Replication Slot (Recommended)

```sql
-- On primary: Create slot BEFORE starting replica
SELECT pg_create_physical_replication_slot('replica1_slot');

-- View slots
SELECT slot_name, active, restart_lsn FROM pg_replication_slots;
```

### Step 4: Take Base Backup

```bash
# On replica server
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/data \
    --checkpoint=fast --wal-method=stream --slot=replica1_slot -v
```

### Step 5: Configure the Replica

```sql
-- Create standby.signal file (empty file)
touch /var/lib/postgresql/data/standby.signal

-- postgresql.conf on replica
primary_conninfo = 'host=primary_host user=replicator password=secret'
primary_slot_name = 'replica1_slot'
hot_standby = on
```

### Step 6: Start the Replica

```bash
pg_ctl start -D /var/lib/postgresql/data
```

---

## Synchronous vs Asynchronous Replication

### Asynchronous (Default)

```
┌─────────────────────────────────────────────────────────────────┐
│  Asynchronous Replication                                        │
│                                                                  │
│  1. Client: INSERT INTO orders...                               │
│  2. Primary: Write to WAL, commit                               │
│  3. Primary: Return "COMMIT" to client ✓                        │
│  4. (Meanwhile) WAL sender ships WAL to replica                 │
│  5. Replica: Receives and applies WAL                           │
│                                                                  │
│  ✓ Fast: Client doesn't wait for replica                       │
│  ✗ Risk: Data loss if primary fails before replica receives    │
└─────────────────────────────────────────────────────────────────┘
```

### Synchronous

```
┌─────────────────────────────────────────────────────────────────┐
│  Synchronous Replication                                         │
│                                                                  │
│  1. Client: INSERT INTO orders...                               │
│  2. Primary: Write to WAL                                       │
│  3. Primary: WAIT for replica confirmation                      │
│  4. Replica: Receives WAL, confirms                             │
│  5. Primary: Return "COMMIT" to client ✓                        │
│                                                                  │
│  ✓ Zero data loss: Transaction confirmed on replica            │
│  ✗ Slower: Client waits for network round-trip                 │
│  ✗ Availability risk: If replica down, writes block            │
└─────────────────────────────────────────────────────────────────┘
```

### Configuring Synchronous Replication

```sql
-- On primary postgresql.conf
synchronous_commit = on
synchronous_standby_names = 'replica1'  -- application_name of replica

-- Multiple replicas: wait for ANY 1
synchronous_standby_names = 'ANY 1 (replica1, replica2, replica3)'

-- Wait for ALL
synchronous_standby_names = 'FIRST 2 (replica1, replica2)'

-- On replica primary_conninfo
primary_conninfo = '... application_name=replica1'
```

### Sync Commit Levels

| Level | Durability | Performance |
|-------|------------|-------------|
| off | Async commits | Fastest |
| local | Commit on primary | Fast |
| remote_write | Replica received | Medium |
| on | Replica flushed | Slow |
| remote_apply | Replica applied | Slowest |

---

## Replication Slots

### Why Replication Slots?

```
Without slot:
- Primary removes old WAL after checkpoint
- Slow replica might miss WAL
- Replica falls out of sync, must re-sync!

With slot:
- Slot tracks replica's position
- WAL retained until replica confirms receipt
- Guaranteed no missing WAL
```

### Managing Slots

```sql
-- Create physical slot
SELECT pg_create_physical_replication_slot('replica1_slot');

-- View slots and their status
SELECT 
    slot_name,
    active,
    restart_lsn,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots;

-- Drop inactive slot (CAREFUL: may lose sync)
SELECT pg_drop_replication_slot('old_replica_slot');
```

### Slot Risk: Disk Space

If replica is down, WAL accumulates:

```sql
-- Monitor WAL retention due to slots
SELECT 
    slot_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
WHERE NOT active;

-- Set maximum WAL retention for slots
max_slot_wal_keep_size = 10GB  -- PG 13+
```

---

## Monitoring Replication

### Replication Status on Primary

```sql
-- View connected replicas
SELECT 
    client_addr,
    application_name,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag,
    sync_state
FROM pg_stat_replication;
```

### Replication Status on Replica

```sql
-- Check replica status
SELECT 
    pg_is_in_recovery() AS is_replica,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn,
    pg_last_xact_replay_timestamp() AS last_replayed_time,
    now() - pg_last_xact_replay_timestamp() AS replay_lag;
```

### Replication Lag Monitoring

```sql
-- Calculate lag in bytes
SELECT 
    pg_current_wal_lsn() AS primary_lsn,  -- Run on primary
    pg_last_wal_receive_lsn(),            -- Run on replica
    pg_wal_lsn_diff(pg_current_wal_lsn(), pg_last_wal_receive_lsn()) AS bytes_behind;

-- Lag in time (approximate)
SELECT now() - pg_last_xact_replay_timestamp() AS lag_time;
```

---

## Failover and Recovery

### Manual Failover

```bash
# 1. Stop the primary (or it's already down)
pg_ctl stop -D /primary/data

# 2. Promote the replica to primary
pg_ctl promote -D /replica/data

# Or via SQL:
SELECT pg_promote();
```

### After Promotion

```sql
-- The promoted server is now read-write
-- standby.signal file is removed automatically

-- For remaining replicas, update primary_conninfo
-- to point to the new primary
```

### Using pg_rewind (Rejoining Old Primary)

```bash
# After failover, old primary can rejoin as replica
# (if it didn't diverge too much)

pg_rewind --target-pgdata=/old_primary/data \
          --source-server="host=new_primary user=replicator"

# Then configure as replica and start
```

---

## Cascading Replication

### Chain Replicas

```
┌─────────────────────────────────────────────────────────────────┐
│  Cascading Replication                                           │
│                                                                  │
│  Primary → Replica1 → Replica2 → Replica3                       │
│     │         │          │          │                           │
│     │         │          │          └── Feeds from Replica2     │
│     │         │          └───────────── Feeds from Replica1     │
│     │         └──────────────────────── Feeds from Primary      │
│     └────────────────────────────────── Main write server       │
│                                                                  │
│  Benefits:                                                       │
│  • Reduces load on primary                                      │
│  • Geographic distribution                                       │
│  • Scale read capacity                                          │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

```sql
-- On Replica2 (feeding from Replica1)
primary_conninfo = 'host=replica1_host user=replicator password=secret'
```

---

## Practical Examples

### Example 1: Check Replication Health

```sql
-- On Primary: Get replication overview
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag
FROM pg_stat_replication;

-- Expected output for healthy replication:
-- application_name | client_addr | state     | sync_state | lag
-- replica1         | 10.0.0.2    | streaming | async      | 16 kB
```

### Example 2: Monitor Slot Usage

```sql
-- Check if slots are causing WAL bloat
SELECT 
    slot_name,
    active,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;
```

### Example 3: Test Failover

```bash
# 1. Verify replica is up to date
psql -h replica -c "SELECT pg_last_xact_replay_timestamp();"

# 2. Stop primary
pg_ctl stop -D /primary/data -m fast

# 3. Promote replica
pg_ctl promote -D /replica/data

# 4. Verify promotion
psql -h replica -c "SELECT pg_is_in_recovery();"
# Should return 'f' (false = no longer in recovery)
```

---

## Hands-On Exercises

### Exercise 1: View Replication Status (Basic)

```sql
-- On primary
SELECT * FROM pg_stat_replication;

-- On replica
SELECT pg_is_in_recovery();
SELECT pg_last_wal_replay_lsn();
SELECT pg_last_xact_replay_timestamp();
```

### Exercise 2: Create and Monitor Slot (Intermediate)

```sql
-- Create a test slot
SELECT pg_create_physical_replication_slot('test_slot');

-- Check slot status
SELECT * FROM pg_replication_slots;

-- See WAL retained by slot
SELECT 
    slot_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
FROM pg_replication_slots;

-- Clean up
SELECT pg_drop_replication_slot('test_slot');
```

### Exercise 3: Simulate Lag (Advanced)

```sql
-- On replica: Pause replay
SELECT pg_wal_replay_pause();

-- On primary: Generate some WAL
CREATE TABLE lag_test AS SELECT generate_series(1, 100000);

-- Check lag
SELECT pg_size_pretty(
    pg_wal_lsn_diff(
        pg_current_wal_lsn(),
        (SELECT replay_lsn FROM pg_stat_replication)
    )
);

-- Resume replay
SELECT pg_wal_replay_resume();
```

---

## Key Takeaways

1. **Streaming replication ships WAL**: Real-time, byte-by-byte.

2. **Asynchronous is default**: Faster but potential data loss.

3. **Synchronous guarantees durability**: At cost of latency.

4. **Replication slots prevent WAL deletion**: But can cause disk bloat.

5. **Hot standby enables read queries**: Replicas serve reads.

6. **Monitor replication lag**: Critical for failover decisions.

7. **pg_promote handles failover**: Simple command, careful planning.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/replication/walsender.c` | WAL sender |
| `src/backend/replication/walreceiver.c` | WAL receiver |
| `src/backend/replication/slot.c` | Replication slots |
| `src/backend/access/transam/xlog.c` | WAL management |

### Documentation

- [PostgreSQL Docs: Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION)
- [PostgreSQL Docs: Replication Slots](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION-SLOTS)
- [PostgreSQL Docs: Hot Standby](https://www.postgresql.org/docs/current/hot-standby.html)

---

## References

### Key Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| wal_level | replica | WAL detail level |
| max_wal_senders | 10 | Max streaming connections |
| max_replication_slots | 10 | Max replication slots |
| synchronous_standby_names | '' | Sync replica names |
| hot_standby | on | Allow read queries on replica |
| wal_keep_size | 0 | Extra WAL to retain |
| max_slot_wal_keep_size | -1 | Max WAL for slots |

### Monitoring Views

| View | Purpose |
|------|---------|
| pg_stat_replication | Replication status (primary) |
| pg_stat_wal_receiver | WAL receiver status (replica) |
| pg_replication_slots | Slot information |
