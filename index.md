# PostgreSQL Uncovered: Internals, Trace Analysis, and Performance

A comprehensive technical course exploring PostgreSQL from the inside out. Starting with simple SQL queries, you'll progressively learn how PostgreSQL parses, plans, executes, stores, optimizes, and recovers data—understanding not just *what* it does, but *why* it's designed that way.

**Target Audience:** Backend engineers, database administrators, performance engineers, and developers who want deep understanding of PostgreSQL internals.

**Prerequisites:**
- Basic SQL knowledge
- Understanding of database concepts (tables, indexes, queries)
- Familiarity with C programming (helpful but not required)
- Basic operating systems knowledge (processes, memory, disk I/O)

---

## Module 1: Foundation — From SQL to Execution

*Learn how PostgreSQL processes queries through its multi-stage pipeline, from raw SQL text to actual results.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 1 | **How PostgreSQL Processes a Query: The Journey from SQL to Result** | Overview of the complete query lifecycle: client connection → parser → rewriter → planner → executor → results. Understanding the postmaster and backend process architecture. |
| 2 | **The Parser: Transforming SQL Into Internal Structures** | Lexical analysis with `scan.l`, grammar rules in `gram.y`, building the raw parse tree, and the `Query` node structure. Source: `src/backend/parser/`. |
| 3 | **The Analyzer: Semantic Analysis and Name Resolution** | How PostgreSQL validates SQL semantics, resolves table/column names, and transforms the parse tree into a query tree. Source: `src/backend/parser/analyze.c`. |
| 4 | **The Rewriter: Query Transformation and View Expansion** | PostgreSQL's rule system, view expansion, and query rewriting logic. Understanding `pg_rewrite` and how rules transform queries. Source: `src/backend/rewrite/`. |
| 5 | **Introduction to the Query Planner: Choosing the Best Path** | Overview of the optimizer: generating access paths, estimating costs, and selecting the optimal execution plan. The difference between Path and Plan nodes. |
| 6 | **The Executor: Running the Plan and Returning Rows** | The demand-pull executor model, plan node types, and how data flows through the execution tree. Understanding `ExecInitNode`, `ExecProcNode`, and `ExecEndNode`. Source: `src/backend/executor/`. |

---

## Module 2: Storage Architecture — How Data Lives on Disk

*Understand PostgreSQL's physical storage layer: heap files, pages, tuples, TOAST, and various index types.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 7 | **Heap Storage: How PostgreSQL Stores Table Data** | The heap access method, relation files, fork types (main, FSM, VM), and the `RelationData` structure. Source: `src/backend/access/heap/`. |
| 8 | **Pages and Tuples: The Building Blocks of PostgreSQL Storage** | Anatomy of an 8KB page: `PageHeaderData`, item pointers, tuple headers (`HeapTupleHeaderData`), and user data layout. Understanding `t_xmin`, `t_xmax`, `t_ctid`. |
| 9 | **The Free Space Map and Visibility Map** | How PostgreSQL tracks free space for insertions (FSM) and all-visible/all-frozen pages (VM). Optimization for VACUUM and index-only scans. |
| 10 | **Understanding TOAST: Handling Large Column Values** | The Oversized-Attribute Storage Technique: compression, out-of-line storage, chunk tables, and the 2KB threshold. Source: `src/backend/access/common/toast_*.c`. |
| 11 | **B-Tree Indexes: Structure, Search, and Maintenance** | PostgreSQL's B-tree implementation: `BTPageOpaqueData`, `IndexTupleData`, search with `_bt_search()`, insertions, page splits, and the HOT optimization. Source: `src/backend/access/nbtree/`. |
| 12 | **Beyond B-Trees: Hash, GiST, GIN, BRIN, and SP-GiST** | Specialized index types for different workloads: hash indexes, generalized search trees (GiST), inverted indexes (GIN), block range indexes (BRIN), and space-partitioned GiST. |
| 13 | **Table Access Methods and Custom Storage Engines** | The pluggable storage architecture (PostgreSQL 12+), the `TableAmRoutine` interface, and how to build custom storage engines. Source: `src/include/access/tableam.h`. |

---

## Module 3: Memory and Buffer Management

*Explore PostgreSQL's memory architecture: shared buffers, local memory, and the algorithms that make caching efficient.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 14 | **PostgreSQL Process Architecture and Shared Memory** | The multi-process model: postmaster, backend processes, background workers, and auxiliary processes. Shared memory layout and inter-process communication. |
| 15 | **Shared Buffers: The Heart of PostgreSQL's Memory Architecture** | The buffer pool design: `BufferDescriptors`, `BufferBlocks`, buffer tags, reference counting, and pin/unpin semantics. Source: `src/backend/storage/buffer/`. |
| 16 | **The Buffer Manager: Cache Lookup, Eviction, and Replacement Policies** | Buffer hash table lookup, the clock sweep algorithm for eviction, `usage_count`, and interaction with the background writer. Understanding buffer ring strategies. |
| 17 | **Inside PostgreSQL Writes: From Shared Buffers to Disk** | The write path: marking buffers dirty, the background writer, checkpointer flushing, and `fsync` strategies. Buffer I/O ordering and checkpoint completion target. |
| 18 | **Local Buffers and Temporary Table Storage** | How temporary tables bypass shared buffers, local buffer management, and memory usage implications for sessions. |
| 19 | **Work Memory: Sorts, Hashes, and Query Execution Limits** | The `work_mem` parameter, memory allocation for sorting and hashing, spilling to disk, and tuning for different workloads. `maintenance_work_mem` and `hash_mem_multiplier`. |

---

## Module 4: Write-Ahead Logging and Crash Recovery

*Master PostgreSQL's durability mechanism: how WAL ensures no committed transaction is ever lost.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 20 | **Why PostgreSQL Uses WAL: Durability Without Immediate Writes** | The write-ahead logging principle, sequential vs. random I/O, and how WAL provides both durability and performance. The `full_page_writes` protection. |
| 21 | **WAL Records: What Gets Logged and How** | Anatomy of a WAL record (`XLogRecord`), resource managers, LSN (Log Sequence Number), and how different operations generate WAL. Source: `src/backend/access/transam/xlog.c`. |
| 22 | **WAL Buffers and the WAL Writer** | The WAL buffer in shared memory, `wal_buffers` configuration, synchronous vs. asynchronous commit (`synchronous_commit`), and group commit optimization. |
| 23 | **Checkpoints: Balancing Performance and Recovery Time** | The checkpointer process, `checkpoint_timeout`, `max_wal_size`, checkpoint completion target, and the trade-off between checkpoint frequency and recovery time. |
| 24 | **Crash Recovery: Replaying WAL to Restore Consistency** | The recovery process: reading `pg_control`, finding the REDO point, WAL replay, and consistency checks. Understanding startup process internals. |
| 25 | **WAL Archiving and Point-in-Time Recovery (PITR)** | Continuous archiving, `archive_command`, recovery.conf (recovery.signal), and restoring to a specific timestamp using `recovery_target_time`. |

---

## Module 5: MVCC — Multi-Version Concurrency Control

*Understand PostgreSQL's concurrency model: how multiple versions of data coexist without locking readers.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 26 | **Introduction to MVCC: Reads Never Block Writes** | The philosophy of multi-version concurrency: why PostgreSQL creates new versions instead of in-place updates. Comparison with lock-based approaches. |
| 27 | **Transaction IDs and How PostgreSQL Tracks Them** | The 32-bit XID space, `xmin`/`xmax` system columns, XIDs in `pg_xact` (formerly `pg_clog`), and the commit log structure. |
| 28 | **Tuple Visibility: How PostgreSQL Decides What You See** | The visibility rules (`HeapTupleSatisfiesMVCC`), transaction snapshots, the CSN (Commit Sequence Number) approach, and the CLOG lookup. Source: `src/backend/utils/time/snapmgr.c`. |
| 29 | **Snapshots and Transaction Isolation** | How snapshots are created and maintained, the `SNAPSHOT_MVCC` structure, running transaction lists (`xip`), and snapshot reuse. |
| 30 | **Why PostgreSQL Doesn't Use Undo Logs** | Design comparison with undo-based MVCC (Oracle, MySQL InnoDB): trade-offs, space usage, rollback speed, and the VACUUM requirement. |
| 31 | **Transactional DDL: PostgreSQL's Hidden Strength** | How PostgreSQL makes DDL operations transactional, catalog updates in MVCC, and why you can rollback a `CREATE TABLE`. |

---

## Module 6: Transaction Isolation and Locking

*Deep dive into isolation levels, lock types, deadlock detection, and how PostgreSQL ensures serializable correctness.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 32 | **Inside PostgreSQL: How Transactions Really Work** | Transaction states, `TransactionId` assignment, subtransactions (savepoints), and the `PGPROC` structure. Source: `src/backend/access/transam/`. |
| 33 | **Isolation Levels in PostgreSQL: Read Committed vs. Repeatable Read** | Implementing Read Committed (new snapshot per statement) and Repeatable Read (transaction snapshot). Practical differences and anomalies prevented. |
| 34 | **Serializable Snapshot Isolation (SSI)** | How PostgreSQL implements serializable without locking: predicate locks, detecting conflicts (rw-dependencies), and the SSI abort logic. Source: `src/backend/storage/lmgr/predicate.c`. |
| 35 | **Lock Types in PostgreSQL: From ACCESS SHARE to ACCESS EXCLUSIVE** | The eight table-level lock modes, lock compatibility matrix, when each lock is acquired, and how to diagnose lock contention using `pg_locks`. |
| 36 | **Row-Level Locks and the xmax System Column** | How row locks are stored in tuple headers (not memory), `FOR UPDATE` vs. `FOR KEY SHARE`, multi-xact IDs, and heavyweight lock fallback. |
| 37 | **Deadlock Detection and Resolution** | The wait-for graph, `deadlock_timeout` parameter, victim selection, and how PostgreSQL breaks deadlock cycles. Strategies to prevent deadlocks. |

---

## Module 7: VACUUM and Dead Tuple Management

*Learn why VACUUM exists, how autovacuum works, and strategies for preventing transaction ID wraparound.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 38 | **Why VACUUM Exists: The Cost of MVCC** | Dead tuples accumulate with MVCC, table bloat, index bloat, and why PostgreSQL cannot rely on garbage collection alone. Setting expectations. |
| 39 | **VACUUM Internals: How Dead Tuples Are Reclaimed** | The heap pruning step, dead tuple collection, index vacuuming, and heap vacuuming. The three-phase VACUUM algorithm. Source: `src/backend/commands/vacuum.c`. |
| 40 | **Autovacuum: Automatic Maintenance in PostgreSQL** | The autovacuum launcher and workers, thresholds (`autovacuum_vacuum_threshold`, `autovacuum_vacuum_scale_factor`), and per-table tuning. Monitoring with `pg_stat_user_tables`. |
| 41 | **Transaction ID Wraparound: The 2-Billion Problem** | How the 32-bit XID space can wrap around, freezing old transactions (`vacuum_freeze_min_age`), and the anti-wraparound autovacuum. |
| 42 | **Aggressive VACUUM and the Visibility Map** | All-frozen pages, the `relfrozenxid` marker, and how the visibility map enables efficient freezing. Monitoring with `pg_visibility`. |
| 43 | **Why ROLLBACK Is So Fast in PostgreSQL** | No undo log to apply: rollback simply marks the transaction as aborted in `pg_xact`. The cleanup is deferred to VACUUM. |

---

## Module 8: System Catalogs and Metadata

*Explore PostgreSQL's metadata infrastructure: how the database stores information about itself.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 44 | **System Catalogs: The Metadata Control Center** | Core catalogs: `pg_class`, `pg_attribute`, `pg_type`, `pg_namespace`, `pg_proc`. How catalogs are bootstrapped. Source: `src/backend/catalog/`. |
| 45 | **The Cache Layer: syscache and relcache** | In-memory caching of catalog data, cache invalidation messages, and the performance implications of catalog lookups. Source: `src/backend/utils/cache/`. |
| 46 | **pg_statistic and Extended Statistics** | How ANALYZE collects statistics, MCVs (most common values), histograms, distinct counts, and multi-column statistics with `CREATE STATISTICS`. |

---

## Module 9: The Query Planner Deep Dive

*Master PostgreSQL's query optimizer: cost estimation, join planning, and why the planner makes the choices it does.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 47 | **From Query to Logical Plan: The Planner's Entry Point** | `standard_planner()`, preparing the query tree, and generating the `RelOptInfo` structures for base relations. Source: `src/backend/optimizer/plan/planner.c`. |
| 48 | **Path Generation: Sequential Scans, Index Scans, and Bitmaps** | How the planner generates alternative access paths (`Path` structures), parameterized paths, and path cost components (startup vs. total cost). |
| 49 | **Inside PostgreSQL's Cost Model: How the Planner Estimates Costs** | Cost functions in `costsize.c`, `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, and selectivity estimation. Tuning cost parameters for SSDs. |
| 50 | **Join Planning: Nested Loops, Hash Joins, and Merge Joins** | When each join method is chosen, cost estimation for joins, and how inner/outer relation selection affects performance. |
| 51 | **Join Ordering: Dynamic Programming and the GEQO** | Combinatorial explosion with many-table joins, the dynamic programming approach, and the Genetic Query Optimizer (GEQO) fallback for large queries. |
| 52 | **Understanding EXPLAIN and EXPLAIN ANALYZE** | Reading execution plans, actual vs. estimated rows, execution time per node, and using buffers statistics. Identifying plan regressions. |
| 53 | **Why PostgreSQL Handles This Query 10× Faster Than That One** | Case study: analyzing slow queries, identifying bad estimates, fixing with updated statistics or hints (via extensions like `pg_hint_plan`). |

---

## Module 10: Advanced Join and Aggregation Internals

*Go deeper into execution strategies for complex queries: joins, grouping, sorting, and parallel execution.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 54 | **Hash Join Internals: Build Phase, Probe Phase, and Batching** | The hash join algorithm, `HashJoinState`, in-memory vs. batched execution when memory is exceeded, and `work_mem` impact. |
| 55 | **Merge Join Internals: Sorted Input and Merge Strategy** | How merge joins work, input sorting requirements, and when the planner chooses merge over hash or nested loop. |
| 56 | **Aggregation Internals: Hash Agg vs. Group Agg** | The `HashAgg` and `GroupAggregate` operators, spilling to disk, partial and finalize aggregation, and mixed aggregate strategies. |
| 57 | **Parallel Query Execution** | Parallel workers, the Gather node, parallel-safe vs. parallel-restricted functions, and partitioning for parallel scans. `parallel_tuple_cost` and `parallel_setup_cost`. |

---

## Module 11: Performance Analysis and Tracing

*Learn techniques for diagnosing performance issues: from EXPLAIN to system-level tracing.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 58 | **pg_stat_statements: Tracking Query Performance** | Enabling and configuring the extension, interpreting statistics (calls, total_time, rows), and identifying top resource consumers. |
| 59 | **The Statistics Collector: pg_stat_* Views** | Understanding `pg_stat_user_tables`, `pg_stat_user_indexes`, `pg_statio_*` views, and diagnosing I/O patterns. |
| 60 | **Tracing Buffer I/O with EXPLAIN (BUFFERS)** | Interpreting buffer statistics (shared hit/read, local hit/read, temp read/write), identifying excessive I/O, and cache miss analysis. |
| 61 | **System-Level Tracing: strace, perf, and DTrace** | Using OS tools to trace PostgreSQL: `strace` for syscalls, `perf` for CPU profiling, and DTrace/SystemTap probes for deep instrumentation. |
| 62 | **Logging and auto_explain for Slow Query Detection** | Configuring `log_min_duration_statement`, using the `auto_explain` extension, and analyzing slow query logs in production. |

---

## Module 12: Extensions and Customization

*Understand PostgreSQL's extensibility architecture and how to add custom functionality.*

| # | Lesson Title | Description |
|---|-------------|-------------|
| 63 | **PostgreSQL Extension Architecture** | How extensions work: `CREATE EXTENSION`, control files, SQL scripts, and version upgrades. The `pg_extension` catalog. |
| 64 | **Building Custom Functions: SQL, PL/pgSQL, and C** | Function creation, language handlers, the `FunctionCallInfo` interface for C functions, and SPI (Server Programming Interface). |
| 65 | **Custom Operators and Index Support** | Creating operator classes, access method strategies, and support functions. Enabling custom types to work with indexes. |
| 66 | **Foreign Data Wrappers: Querying External Systems** | The FDW architecture, `postgres_fdw`, and implementing custom FDWs. Push-down optimization and join handling. |

---

## Appendix: Hands-On Labs

| Lab | Title | Description |
|-----|-------|-------------|
| A | **Building PostgreSQL from Source** | Compiling with debug symbols, `--enable-cassert`, and running under `gdb`. |
| B | **Using pageinspect to Examine Page Contents** | Viewing heap pages, tuple headers, and B-tree index pages with the `pageinspect` extension. |
| C | **Observing MVCC with pg_visibility** | Watching visibility map changes, all-visible and all-frozen flags during VACUUM. |
| D | **Tracing Query Execution with GDB** | Setting breakpoints in the executor, stepping through `ExecProcNode`, and inspecting plan state. |
| E | **Simulating Transaction ID Wraparound** | Using `pg_resetwal` (in a lab environment) to understand wraparound scenarios. |
| F | **Building a Simple Extension** | Creating a minimal C extension with custom functions and proper packaging. |

---

## Course Resources

### Key PostgreSQL Source Directories
- `src/backend/parser/` — Lexer, parser, analyzer
- `src/backend/optimizer/` — Query planner and cost estimation
- `src/backend/executor/` — Query execution engine
- `src/backend/storage/` — Buffer manager, storage manager
- `src/backend/access/` — Access methods (heap, indexes, transam)
- `src/backend/commands/` — SQL command implementations
- `src/backend/utils/` — Caching, memory contexts, utilities
- `src/include/` — Header files with data structures

### Recommended Reading
- [The Internals of PostgreSQL](https://www.interdb.jp/pg/) by Hironobu Suzuki
- [PostgreSQL Documentation: Internals](https://www.postgresql.org/docs/current/internals.html)
- [PostgreSQL Wiki: Developer Information](https://wiki.postgresql.org/wiki/Development_information)
- Original POSTGRES papers by Michael Stonebraker

---

*Total Lessons: 66 | Estimated Course Duration: 80-100 hours*