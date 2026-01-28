# Lesson 31: Connection Pooling — Managing Database Connections Efficiently

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 12 — PostgreSQL Architecture |
| **Lesson** | 31 of 66 |
| **Estimated Time** | 60-75 minutes |
| **Prerequisites** | Lesson 21: Background Processes |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Understand why connection pooling is essential
2. Explain how PostgreSQL handles connections internally
3. Compare pooling modes and their trade-offs
4. Configure PgBouncer for production use
5. Monitor and troubleshoot connection pool health

### Key Terms

| Term | Definition |
|------|------------|
| **Connection Pooling** | Reusing database connections across clients |
| **Backend Process** | PostgreSQL process handling one connection |
| **PgBouncer** | Lightweight connection pooler for PostgreSQL |
| **Session Pooling** | Connection assigned for entire client session |
| **Transaction Pooling** | Connection assigned per transaction |
| **Statement Pooling** | Connection assigned per statement |

---

## Introduction

### Why Connection Pooling?

```
┌─────────────────────────────────────────────────────────────────┐
│  The Connection Problem                                          │
│                                                                  │
│  PostgreSQL Architecture:                                        │
│  • Each connection = 1 backend process                           │
│  • Each process uses ~5-10 MB RAM                               │
│  • Process creation has overhead (~100ms)                       │
│                                                                  │
│  Web Application with 100 servers, 10 connections each:         │
│  = 1,000 connections = 1,000 processes = 10 GB RAM             │
│                                                                  │
│  Problem:                                                        │
│  • Most connections are idle (waiting for queries)              │
│  • max_connections limit (typically 100-500)                    │
│  • Memory exhaustion                                             │
│  • Context switching overhead                                    │
│                                                                  │
│  Solution: Connection Pooling                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  100 app servers → Pooler → 50 actual connections        │  │
│  │  (1000 logical)     ↓        (50 backend processes)      │  │
│  │                 Reuses idle                                │  │
│  │                 connections                                │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## PostgreSQL's Connection Model

### One Process Per Connection

```
┌─────────────────────────────────────────────────────────────────┐
│  PostgreSQL Process Architecture                                 │
│                                                                  │
│  Client 1 ──────┐                                               │
│  Client 2 ──────┼──▶ Postmaster ──┬──▶ Backend 1 (Client 1)    │
│  Client 3 ──────┤       │         ├──▶ Backend 2 (Client 2)    │
│  Client 4 ──────┘       │         ├──▶ Backend 3 (Client 3)    │
│                         │         └──▶ Backend 4 (Client 4)    │
│                         │                                       │
│                         └──▶ (forks new process per connection)│
│                                                                  │
│  Each backend process:                                          │
│  • Has its own memory (5-10 MB baseline)                        │
│  • Maintains connection state                                   │
│  • Keeps result caches, prepared statements                     │
│  • Lives until client disconnects                               │
└─────────────────────────────────────────────────────────────────┘
```

### Connection Overhead

```sql
-- Check max connections
SHOW max_connections;  -- Default: 100

-- Current connection count
SELECT count(*) FROM pg_stat_activity;

-- Connection states
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
-- idle: waiting for queries (wasting resources)
-- active: running queries
-- idle in transaction: holding resources
```

---

## Pooling Modes

### Session Pooling

```
┌─────────────────────────────────────────────────────────────────┐
│  Session Pooling                                                 │
│                                                                  │
│  Client A connects ──→ Gets Backend 1 (keeps for entire session)│
│  Client A: SELECT...    Uses Backend 1                          │
│  Client A: UPDATE...    Uses Backend 1                          │
│  Client A disconnects ─→ Backend 1 returned to pool            │
│                                                                  │
│  Pros:                                                          │
│  ✓ Full feature compatibility                                   │
│  ✓ Prepared statements work                                     │
│  ✓ Session variables preserved                                  │
│                                                                  │
│  Cons:                                                          │
│  ✗ Limited connection reuse                                    │
│  ✗ Long-lived sessions block connections                       │
│                                                                  │
│  Use when: Need session state, prepared statements              │
└─────────────────────────────────────────────────────────────────┘
```

### Transaction Pooling

```
┌─────────────────────────────────────────────────────────────────┐
│  Transaction Pooling (Most Common)                               │
│                                                                  │
│  Client A: BEGIN      ──→ Gets Backend 1                        │
│  Client A: SELECT...      Uses Backend 1                        │
│  Client A: COMMIT     ──→ Backend 1 returned to pool           │
│                                                                  │
│  Client B: BEGIN      ──→ Gets Backend 1 (reused!)             │
│  Client B: INSERT...      Uses Backend 1                        │
│  Client B: COMMIT     ──→ Backend 1 returned                   │
│                                                                  │
│  Pros:                                                          │
│  ✓ Maximum connection reuse                                     │
│  ✓ Handles many clients with few connections                   │
│                                                                  │
│  Cons:                                                          │
│  ✗ Prepared statements don't work across transactions          │
│  ✗ SET commands lost between transactions                      │
│  ✗ Temporary tables problematic                                │
│                                                                  │
│  Use when: Stateless applications, high concurrency             │
└─────────────────────────────────────────────────────────────────┘
```

### Statement Pooling

```
┌─────────────────────────────────────────────────────────────────┐
│  Statement Pooling (Aggressive)                                  │
│                                                                  │
│  Client A: SELECT... ──→ Gets Backend 1, returns immediately   │
│  Client A: SELECT... ──→ Gets Backend 2 (different!)           │
│                                                                  │
│  Pros:                                                          │
│  ✓ Absolute maximum connection reuse                           │
│                                                                  │
│  Cons:                                                          │
│  ✗ No multi-statement transactions                             │
│  ✗ Rarely used in practice                                     │
│                                                                  │
│  Use when: Only single-statement queries, no transactions       │
└─────────────────────────────────────────────────────────────────┘
```

---

## PgBouncer

### What is PgBouncer?

```
┌─────────────────────────────────────────────────────────────────┐
│  PgBouncer Architecture                                          │
│                                                                  │
│  App Server 1 ─┐                                                │
│  App Server 2 ─┼──▶ PgBouncer ──▶ PostgreSQL                   │
│  App Server 3 ─┤    (Pooler)      (Limited connections)        │
│  App Server 4 ─┘                                                │
│                                                                  │
│  1000 app       100 pool         50 actual                      │
│  connections    connections      backends                        │
│                                                                  │
│  PgBouncer is:                                                  │
│  • Lightweight (~2KB per connection)                            │
│  • Single-threaded, event-driven (uses libevent)               │
│  • Supports all pooling modes                                   │
│  • Can run on app servers or dedicated hosts                    │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Configuration

```ini
# pgbouncer.ini

[databases]
# database = connection string
myapp = host=localhost port=5432 dbname=myapp

# Wildcard: all databases
* = host=localhost port=5432

[pgbouncer]
# Pooler settings
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Pool mode
pool_mode = transaction

# Pool size
default_pool_size = 20
max_client_conn = 1000
max_db_connections = 50

# Timeouts
server_idle_timeout = 600
query_timeout = 30
```

### User Authentication

```bash
# userlist.txt
# Format: "username" "password" or "username" "md5hash"
"myapp" "scram-sha-256$4096:salt$hash"

# Generate hash
psql -c "SELECT concat('\"', usename, '\" \"', passwd, '\"') 
         FROM pg_shadow WHERE usename = 'myapp'"
```

### Starting PgBouncer

```bash
# Start
pgbouncer /etc/pgbouncer/pgbouncer.ini

# Daemon mode
pgbouncer -d /etc/pgbouncer/pgbouncer.ini

# Reload config
pgbouncer -R /etc/pgbouncer/pgbouncer.ini
```

---

## PgBouncer Administration

### Admin Console

```sql
-- Connect to admin console
psql -p 6432 -U pgbouncer pgbouncer

-- Show pools
SHOW POOLS;
-- database | user | cl_active | cl_waiting | sv_active | sv_idle

-- Show clients
SHOW CLIENTS;

-- Show servers (actual PostgreSQL connections)
SHOW SERVERS;

-- Show configuration
SHOW CONFIG;

-- Show statistics
SHOW STATS;
```

### Key Metrics

```sql
-- Pool utilization
SHOW POOLS;

/*
 database | user | cl_active | cl_waiting | sv_active | sv_idle
 myapp    | app  | 50        | 0          | 20        | 10

 cl_active: Clients with queries running
 cl_waiting: Clients waiting for connection
 sv_active: Server connections running queries  
 sv_idle: Server connections available
*/

-- If cl_waiting > 0, need more pool connections!

-- Statistics
SHOW STATS;
/*
 database | total_xact_count | total_query_count | avg_xact_time
 myapp    | 1234567          | 9876543           | 12345
*/
```

### Reload and Pause

```sql
-- Reload configuration (no restart needed)
RELOAD;

-- Pause a database (for maintenance)
PAUSE myapp;

-- Resume
RESUME myapp;

-- Graceful shutdown
SHUTDOWN;
```

---

## Transaction Pooling Compatibility

### What Works

| Feature | Works? |
|---------|--------|
| Regular queries | ✓ |
| Transactions (BEGIN/COMMIT) | ✓ |
| Savepoints | ✓ |
| CTEs | ✓ |
| JOINs | ✓ |

### What Doesn't Work (by default)

| Feature | Works? | Workaround |
|---------|--------|------------|
| Prepared statements | ✗ | Use session mode or disable |
| SET commands | ✗ | Use SET LOCAL or startup params |
| LISTEN/NOTIFY | ✗ | Use session mode |
| Advisory locks | ✗ | Use session mode |
| Temp tables | ✗ | Avoid or use session mode |

### Prepared Statements Fix

```ini
# pgbouncer.ini - Disable server-side prepared statements
# Application must handle this

# For most ORMs, set:
# prepared_statements = false in application config
```

```python
# SQLAlchemy example
engine = create_engine(
    "postgresql://...",
    connect_args={"prepare_threshold": None}  # Disable prepared statements
)
```

---

## Sizing the Pool

### Optimal Pool Size

```
Rule of Thumb:
connections = (core_count * 2) + effective_spindle_count

For SSD:
connections = core_count * 2 + 1

Example: 4-core server with SSD
= 4 * 2 + 1 = 9 connections
```

### PgBouncer Settings

```ini
# Per-database pool size
default_pool_size = 20

# Minimum idle connections (keep warm)
min_pool_size = 5

# Reserve for admin connections
reserve_pool_size = 5
reserve_pool_timeout = 5

# Maximum total server connections
max_db_connections = 100
```

### Client Limits

```ini
# Max clients PgBouncer accepts
max_client_conn = 1000

# Per-user limits
myapp = host=... pool_size=30 max_db_connections=50
```

---

## Monitoring

### PgBouncer Metrics

```sql
-- Connection pool status
SHOW POOLS;

-- Key things to watch:
-- cl_waiting > 0 → Pool exhausted, clients waiting
-- sv_active = pool_size → All connections busy
-- avg_query / avg_xact → Query performance
```

### PostgreSQL-Side Monitoring

```sql
-- Who's connected?
SELECT usename, application_name, client_addr, state
FROM pg_stat_activity
WHERE usename = 'myapp';

-- Connection by pooler
SELECT client_addr, count(*)
FROM pg_stat_activity
GROUP BY client_addr;
```

### Prometheus/Grafana

```bash
# pgbouncer_exporter for Prometheus
pgbouncer_exporter --pgBouncer.connectionString="postgres://..."
```

---

## Alternative Poolers

### Pgpool-II

- More features (load balancing, HA)
- Heavier weight
- Statement-level load balancing

### Built-in Pooling (PG 14+)

```sql
-- Not true pooling, but connection limiting
ALTER SYSTEM SET max_connections = 200;
```

### Application-Level Pooling

```python
# SQLAlchemy connection pool
engine = create_engine(
    "postgresql://...",
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True
)
```

---

## Hands-On Exercises

### Exercise 1: Monitor Connections (Basic)

```sql
-- Check current connections
SELECT 
    usename,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY usename, state
ORDER BY count DESC;

-- Find idle connections (wasting resources)
SELECT 
    pid,
    usename,
    state,
    query_start,
    now() - query_start AS idle_time
FROM pg_stat_activity
WHERE state = 'idle'
ORDER BY idle_time DESC;
```

### Exercise 2: PgBouncer Stats (Intermediate)

```sql
-- Connect to PgBouncer admin
psql -p 6432 -U pgbouncer pgbouncer

-- Check pool health
SHOW POOLS;

-- Look for:
-- cl_waiting > 0 (need more connections)
-- sv_idle = 0 (all connections busy)

-- Check average times
SHOW STATS;
```

### Exercise 3: Simulate Connection Pressure (Advanced)

```bash
# Use pgbench to simulate many connections
pgbench -c 100 -j 10 -T 60 -p 6432 myapp

# Watch pool behavior
watch -n 1 'psql -p 6432 -U pgbouncer pgbouncer -c "SHOW POOLS"'
```

---

## Key Takeaways

1. **Each connection = one process**: PostgreSQL's model doesn't scale to thousands.

2. **Pooling is essential**: For any production workload.

3. **Transaction pooling is most common**: Best reuse, some limitations.

4. **PgBouncer is lightweight**: ~2KB per connection vs ~10MB per PostgreSQL backend.

5. **Size pools carefully**: Too few = waiting, too many = no benefit.

6. **Monitor cl_waiting**: If > 0, pool is exhausted.

7. **Know the limitations**: Prepared statements, temp tables, SET commands.

---

## Further Reading

### Documentation

- [PgBouncer Documentation](https://www.pgbouncer.org/)
- [PostgreSQL Docs: Connection Settings](https://www.postgresql.org/docs/current/runtime-config-connection.html)

### Tools

- [PgBouncer](https://www.pgbouncer.org/)
- [Pgpool-II](https://www.pgpool.net/)
- [PgBouncer Exporter](https://github.com/prometheus-community/pgbouncer_exporter)

---

## References

### PgBouncer Pool Modes

| Mode | Connection Assigned | Best For |
|------|---------------------|----------|
| session | Entire client session | Stateful apps |
| transaction | Per transaction | Most apps |
| statement | Per statement | Rare, single queries |

### Key Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| pool_mode | session | Pooling mode |
| default_pool_size | 20 | Connections per pool |
| max_client_conn | 100 | Max clients |
| max_db_connections | 0 | Max to PostgreSQL |
| query_timeout | 0 | Query time limit |
