# Lesson 21: Background Processes — The Workers Behind the Scenes

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 12 — PostgreSQL Architecture |
| **Lesson** | 21 of 66 |
| **Estimated Time** | 60-75 minutes |
| **Prerequisites** | Lesson 1: How PostgreSQL Processes a Query |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Identify PostgreSQL's background processes and their roles
2. Understand how each background worker contributes to system health
3. Configure background process parameters appropriately
4. Monitor background process activity
5. Troubleshoot common background process issues

### Key Terms

| Term | Definition |
|------|------------|
| **Postmaster** | Main process that spawns all others |
| **Backend** | Process handling a client connection |
| **Background Writer** | Writes dirty buffers to disk gradually |
| **Checkpointer** | Performs checkpoints |
| **WAL Writer** | Writes WAL buffers to disk |
| **Autovacuum Launcher** | Manages autovacuum workers |
| **Stats Collector** | Collects database statistics |

---

## Introduction

PostgreSQL uses a multi-process architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│  PostgreSQL Process Architecture                                 │
│                                                                  │
│                    ┌─────────────────┐                          │
│                    │   POSTMASTER    │                          │
│                    │   (PID 1234)    │                          │
│                    │   Main daemon   │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│         ┌───────────────────┼───────────────────┐               │
│         │                   │                   │               │
│         ▼                   ▼                   ▼               │
│  ┌─────────────┐   ┌─────────────────┐   ┌───────────────┐     │
│  │Client       │   │Background       │   │System         │     │
│  │Connections  │   │Maintenance      │   │Monitoring     │     │
│  └─────────────┘   └─────────────────┘   └───────────────┘     │
│         │                   │                   │               │
│    ┌────┴────┐      ┌──────┴──────┐      ┌─────┴─────┐        │
│    ▼         ▼      ▼             ▼      ▼           ▼        │
│  Backend  Backend  Checkpointer  WAL    Stats      Logger     │
│  (conn1) (conn2)   BGWriter      Writer Collector             │
│                    Autovacuum    Archiver                      │
│                    Launcher                                     │
│                      │                                          │
│              ┌───────┴───────┐                                 │
│              ▼               ▼                                 │
│           Autovacuum    Autovacuum                             │
│           Worker 1      Worker 2                               │
└─────────────────────────────────────────────────────────────────┘
```

Each process has a specific role in maintaining database health and performance.

---

## The Postmaster

### Role

The postmaster is PostgreSQL's main daemon process:

- **Listens** for new client connections
- **Forks** backend processes for each connection
- **Launches** all background workers
- **Monitors** child processes and restarts if needed
- **Handles** shutdown/restart signals

### Viewing the Postmaster

```bash
# Find postmaster PID
cat $PGDATA/postmaster.pid | head -1

# View process tree
ps auxf | grep postgres
```

### Startup Sequence

```
1. Postmaster starts
2. Reads postgresql.conf and pg_hba.conf
3. Allocates shared memory
4. Launches background processes
5. Opens listening socket
6. Accepts connections
```

---

## Background Writer (bgwriter)

### Role

Writes dirty shared buffers to disk gradually:

```
┌─────────────────────────────────────────────────────────────────┐
│  Background Writer Purpose                                       │
│                                                                  │
│  WITHOUT bgwriter:                                              │
│  Backend needs buffer → All buffers dirty → Must write one     │
│  → Blocking I/O → Slow query!                                  │
│                                                                  │
│  WITH bgwriter:                                                 │
│  bgwriter continuously writes dirty buffers → Clean buffers    │
│  available → Backend gets buffer immediately → Fast query!     │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `bgwriter_delay` | 200ms | Sleep between rounds |
| `bgwriter_lru_maxpages` | 100 | Max pages per round |
| `bgwriter_lru_multiplier` | 2.0 | Multiplier for recent usage |
| `bgwriter_flush_after` | 512KB | Trigger OS flush after this |

### Monitoring

```sql
SELECT 
    buffers_clean,           -- Buffers written by bgwriter
    maxwritten_clean,        -- Times stopped due to hitting max
    buffers_alloc            -- Buffers allocated
FROM pg_stat_bgwriter;
```

If `maxwritten_clean` is high, increase `bgwriter_lru_maxpages`.

---

## Checkpointer

### Role

Performs checkpoints—flushing ALL dirty buffers and writing checkpoint record:

```
┌─────────────────────────────────────────────────────────────────┐
│  Checkpoint Process                                              │
│                                                                  │
│  1. Mark REDO point in WAL                                      │
│  2. Flush all dirty shared buffers to disk                      │
│  3. Flush other caches (clog, subtrans, multixact)              │
│  4. Write checkpoint record to WAL                              │
│  5. Update pg_control                                           │
│  6. Recycle old WAL segments                                    │
│                                                                  │
│  Triggers:                                                       │
│  - checkpoint_timeout (default 5 min)                           │
│  - WAL generated reaches max_wal_size                           │
│  - Manual: CHECKPOINT command                                   │
│  - Shutdown                                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `checkpoint_timeout` | 5min | Max time between checkpoints |
| `max_wal_size` | 1GB | WAL size to trigger checkpoint |
| `min_wal_size` | 80MB | Min WAL to keep |
| `checkpoint_completion_target` | 0.9 | Spread I/O over this fraction |
| `checkpoint_warning` | 30s | Warn if checkpoints too frequent |

### Monitoring

```sql
SELECT 
    checkpoints_timed,       -- Scheduled checkpoints
    checkpoints_req,         -- Requested (WAL size exceeded)
    checkpoint_write_time,   -- Time writing buffers
    checkpoint_sync_time,    -- Time syncing to disk
    buffers_checkpoint       -- Buffers written by checkpointer
FROM pg_stat_bgwriter;
```

High `checkpoints_req` means you're hitting `max_wal_size` often—consider increasing it.

---

## WAL Writer

### Role

Writes WAL buffers to disk:

```
┌─────────────────────────────────────────────────────────────────┐
│  WAL Writer Flow                                                 │
│                                                                  │
│  Backends generate WAL → WAL buffers (shared memory)            │
│                              │                                  │
│              ┌───────────────┴───────────────┐                 │
│              ▼                               ▼                 │
│        synchronous_commit=on          WAL Writer (async)       │
│        Backend syncs itself           Writes periodically      │
│              │                               │                 │
│              └───────────────┬───────────────┘                 │
│                              ▼                                  │
│                         pg_wal/                                │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `wal_writer_delay` | 200ms | Sleep between writes |
| `wal_writer_flush_after` | 1MB | Flush after this much |

### When WAL Writer Matters

- With `synchronous_commit = off`, WAL writer handles most writes
- Reduces backend I/O because WAL writer batches writes
- Doesn't affect durability when `synchronous_commit = on`

---

## Autovacuum

### Components

```
┌─────────────────────────────────────────────────────────────────┐
│  Autovacuum System                                               │
│                                                                  │
│  ┌─────────────────────────────┐                                │
│  │  Autovacuum Launcher        │                                │
│  │  - Wakes every naptime      │                                │
│  │  - Checks which tables need │                                │
│  │    vacuuming/analyzing      │                                │
│  │  - Launches workers         │                                │
│  └─────────────┬───────────────┘                                │
│                │                                                 │
│        ┌───────┴───────┐                                        │
│        ▼               ▼                                        │
│  ┌──────────────┐ ┌──────────────┐                              │
│  │Autovacuum    │ │Autovacuum    │                              │
│  │Worker 1      │ │Worker 2      │                              │
│  │              │ │              │                              │
│  │VACUUM table_a│ │VACUUM table_b│                              │
│  └──────────────┘ └──────────────┘                              │
│                                                                  │
│  Max workers: autovacuum_max_workers (default 3)                │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `autovacuum` | on | Enable autovacuum |
| `autovacuum_naptime` | 1min | Time between launcher runs |
| `autovacuum_max_workers` | 3 | Max concurrent workers |
| `autovacuum_vacuum_threshold` | 50 | Dead tuples before vacuum |
| `autovacuum_vacuum_scale_factor` | 0.2 | Fraction of table |
| `autovacuum_vacuum_cost_delay` | 2ms | Delay when hitting cost |
| `autovacuum_vacuum_cost_limit` | -1 | Cost limit per worker |

### Monitoring

```sql
-- Current autovacuum activity
SELECT pid, datname, relid::regclass, phase, 
       heap_blks_scanned, heap_blks_vacuumed
FROM pg_stat_progress_vacuum;

-- Autovacuum workers running
SELECT * FROM pg_stat_activity 
WHERE backend_type = 'autovacuum worker';

-- Tables needing vacuum
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

---

## Stats Collector

### Role

Collects and stores statistics about database activity:

```
┌─────────────────────────────────────────────────────────────────┐
│  Stats Collector Flow                                            │
│                                                                  │
│  Backends                                                        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                           │
│  │Backend 1│ │Backend 2│ │Backend 3│                           │
│  └────┬────┘ └────┬────┘ └────┬────┘                           │
│       │           │           │                                  │
│       │ Stats messages (UDP)  │                                 │
│       └───────────┴───────────┘                                 │
│                   │                                              │
│                   ▼                                              │
│         ┌─────────────────┐                                     │
│         │ Stats Collector │                                     │
│         │                 │                                     │
│         │ Aggregates and  │                                     │
│         │ writes to:      │                                     │
│         │ pg_stat_tmp/    │                                     │
│         └─────────────────┘                                     │
│                   │                                             │
│                   ▼                                             │
│         ┌─────────────────┐                                    │
│         │ pg_stat views:  │                                    │
│         │ pg_stat_activity│                                    │
│         │ pg_stat_tables  │                                    │
│         │ pg_stat_bgwriter│                                    │
│         └─────────────────┘                                    │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `track_activities` | on | Track current command |
| `track_counts` | on | Track table/index stats |
| `track_functions` | none | Track function calls |
| `track_io_timing` | off | Track I/O timing |
| `stats_temp_directory` | pg_stat_tmp | Temp stats location |

### Monitoring

```sql
-- Force stats flush
SELECT pg_stat_force_stats();

-- Reset stats
SELECT pg_stat_reset();

-- View stats
SELECT * FROM pg_stat_database;
SELECT * FROM pg_stat_user_tables;
SELECT * FROM pg_stat_user_indexes;
```

---

## WAL Archiver

### Role

Archives completed WAL segments for backup/recovery:

```
┌─────────────────────────────────────────────────────────────────┐
│  WAL Archiving                                                   │
│                                                                  │
│  pg_wal/                          Archive Location              │
│  ┌─────────────────────────────┐   ┌─────────────────────────┐ │
│  │ 000000010000000000000001    │──▶│ (backup storage)        │ │
│  │ 000000010000000000000002    │──▶│ /backup/wal/            │ │
│  │ 000000010000000000000003    │──▶│ or S3, NFS, etc.       │ │
│  │ 000000010000000000000004    │   │                         │ │
│  └─────────────────────────────┘   └─────────────────────────┘ │
│         ↑                                                        │
│  WAL archiver copies                                            │
│  completed segments                                              │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

```sql
-- Enable archiving
ALTER SYSTEM SET archive_mode = on;
ALTER SYSTEM SET archive_command = 'cp %p /backup/wal/%f';
-- %p = path to WAL file, %f = filename

-- Or with compression
ALTER SYSTEM SET archive_command = 'gzip < %p > /backup/wal/%f.gz';
```

### Monitoring

```sql
-- Archive status
SELECT * FROM pg_stat_archiver;

-- Columns:
-- archived_count: Successful archives
-- failed_count: Failed attempts
-- last_archived_wal: Last archived file
-- last_archived_time: When
-- last_failed_wal: Last failure
```

---

## Logical Replication Workers

### Role

Handle logical replication subscriptions:

```
┌─────────────────────────────────────────────────────────────────┐
│  Logical Replication Workers                                     │
│                                                                  │
│  Publisher                          Subscriber                  │
│  ┌────────────────────┐            ┌────────────────────┐      │
│  │ Publication        │            │ Subscription       │      │
│  │ ┌────────────────┐ │            │ ┌────────────────┐ │      │
│  │ │ WAL Sender     │─┼────────────┼▶│ Apply Worker   │ │      │
│  │ │ (per subscriber)│ │  changes   │ │ (per subscription) │    │
│  │ └────────────────┘ │            │ └────────────────┘ │      │
│  └────────────────────┘            └────────────────────┘      │
│                                                                  │
│  Logical Replication Launcher manages apply workers             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Viewing All Processes

```sql
-- All PostgreSQL processes
SELECT 
    pid,
    backend_type,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity;

-- Backend types:
-- client backend
-- autovacuum launcher
-- autovacuum worker
-- background writer
-- checkpointer
-- walwriter
-- archiver
-- stats collector
-- logical replication launcher
-- logical replication worker
-- parallel worker
```

```bash
# From command line
ps aux | grep postgres

# With process tree
pstree -p $(head -1 $PGDATA/postmaster.pid)
```

---

## Hands-On Exercises

### Exercise 1: Identifying Processes (Basic)

```bash
# Find all PostgreSQL processes
ps aux | grep postgres

# View process tree
pstree -p $(cat $PGDATA/postmaster.pid | head -1)
```

```sql
-- View from inside PostgreSQL
SELECT pid, backend_type, application_name, state
FROM pg_stat_activity
ORDER BY backend_type;
```

### Exercise 2: Monitoring Bgwriter (Intermediate)

```sql
-- Reset stats
SELECT pg_stat_reset_shared('bgwriter');

-- Generate I/O
CREATE TABLE bgtest AS SELECT generate_series(1, 1000000);
UPDATE bgtest SET generate_series = generate_series + 1;

-- Check bgwriter activity
SELECT 
    buffers_clean,
    maxwritten_clean,
    buffers_backend,
    buffers_alloc
FROM pg_stat_bgwriter;
```

### Exercise 3: Checkpoint Monitoring (Intermediate)

```sql
-- Enable checkpoint logging
SET log_checkpoints = on;

-- Force checkpoint
CHECKPOINT;

-- Check stats
SELECT 
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time
FROM pg_stat_bgwriter;
```

### Exercise 4: Autovacuum in Action (Advanced)

```sql
-- Create table with churn
CREATE TABLE auto_test (id serial, data text);
INSERT INTO auto_test SELECT generate_series(1, 100000), 'data';

-- Generate dead tuples
UPDATE auto_test SET data = 'updated' WHERE id % 2 = 0;

-- Watch for autovacuum
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'auto_test';

-- Monitor autovacuum workers
SELECT pid, query FROM pg_stat_activity 
WHERE backend_type = 'autovacuum worker';
```

---

## Key Takeaways

1. **Postmaster is the parent**: All other processes are children.

2. **Bgwriter reduces latency**: Keeps clean buffers available.

3. **Checkpointer ensures durability**: Flushes everything periodically.

4. **WAL writer handles async WAL**: Batches WAL writes efficiently.

5. **Autovacuum is essential**: Don't disable it!

6. **Stats collector enables monitoring**: Powers pg_stat_* views.

7. **Each process has a job**: Understanding them helps troubleshooting.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/postmaster/postmaster.c` | Main daemon |
| `src/backend/postmaster/bgwriter.c` | Background writer |
| `src/backend/postmaster/checkpointer.c` | Checkpointer |
| `src/backend/postmaster/walwriter.c` | WAL writer |
| `src/backend/postmaster/autovacuum.c` | Autovacuum |
| `src/backend/postmaster/pgstat.c` | Stats collector |

### Documentation

- [PostgreSQL Docs: Server Processes](https://www.postgresql.org/docs/current/monitoring-ps.html)
- [PostgreSQL Docs: Background Writer](https://www.postgresql.org/docs/current/runtime-config-resource.html#RUNTIME-CONFIG-RESOURCE-BACKGROUND-WRITER)

---

## References

### Process Types Summary

| Process | Purpose | Key Config |
|---------|---------|------------|
| Postmaster | Main daemon | N/A |
| Backend | Client connection | max_connections |
| Bgwriter | Write dirty buffers | bgwriter_* |
| Checkpointer | Perform checkpoints | checkpoint_* |
| WAL writer | Write WAL buffers | wal_writer_* |
| Autovacuum launcher | Manage autovacuum | autovacuum_* |
| Autovacuum worker | Run vacuum/analyze | autovacuum_* |
| Stats collector | Collect statistics | track_* |
| Archiver | Archive WAL | archive_* |
| WAL sender | Streaming replication | max_wal_senders |
| WAL receiver | Standby receives WAL | N/A |
