# Lesson 1: How PostgreSQL Processes a Query — The Journey from SQL to Result

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 1 — Foundation: From SQL to Execution |
| **Lesson** | 1 of 66 |
| **Estimated Time** | 45-60 minutes |
| **Prerequisites** | Basic SQL knowledge, familiarity with client-server architecture |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Describe the complete lifecycle of a SQL query in PostgreSQL
2. Explain the role of the postmaster and backend processes
3. Identify the five major stages of query processing: parsing, rewriting, planning, and execution
4. Understand why PostgreSQL uses a process-per-connection model
5. Trace the flow of a simple query through PostgreSQL's internals

### Key Terms

| Term | Definition |
|------|------------|
| **Postmaster** | The master PostgreSQL process that listens for connections and spawns backend processes |
| **Backend Process** | A dedicated process handling a single client connection |
| **Parse Tree** | The initial tree structure representing the syntactic structure of a SQL query |
| **Query Tree** | The semantically analyzed version of the parse tree with resolved names and types |
| **Plan Tree** | The execution plan chosen by the optimizer, consisting of plan nodes |
| **Executor** | The component that runs the plan tree and produces query results |

---

## Introduction

When you type `SELECT * FROM users WHERE id = 1;` into psql and press Enter, what actually happens inside PostgreSQL? The answer involves a surprisingly sophisticated journey through multiple subsystems, each with its own responsibilities and data structures.

Understanding this journey is fundamental to everything else in this course. Every performance problem you'll ever debug, every query plan you'll analyze, and every configuration decision you'll make ties back to this core processing pipeline. When a query is slow, you need to know *where* in the pipeline the time is being spent. When the planner makes a bad decision, you need to understand *what* information it was working with.

PostgreSQL's query processing architecture reflects decades of database research and engineering. It separates concerns cleanly: the parser doesn't need to know about disk I/O, and the executor doesn't need to understand SQL grammar. This separation makes PostgreSQL both maintainable and extensible.

In this lesson, we'll follow a query from the moment it arrives over the network to the moment results are sent back to the client. We'll introduce each component briefly here, then dive deep into each one in subsequent lessons.

> **Why This Matters:** When you encounter a slow query, knowing the processing stages helps you identify the bottleneck. Is it a parsing issue (unlikely)? A bad plan (common)? Excessive I/O during execution (very common)? This mental model is your diagnostic framework.

---

## Conceptual Foundation

### The Big Picture: PostgreSQL's Multi-Process Architecture

PostgreSQL uses a **process-per-connection model**. Unlike some databases that use threads, PostgreSQL spawns a separate operating system process for each client connection. This design choice has profound implications:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Operating System                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────┐                                               │
│   │  Postmaster │ ◄─── Listens on port 5432                     │
│   │   (PID 1)   │                                               │
│   └──────┬──────┘                                               │
│          │ fork()                                               │
│          ▼                                                      │
│   ┌──────────────┬──────────────┬──────────────┐               │
│   │   Backend    │   Backend    │   Backend    │               │
│   │  (Client 1)  │  (Client 2)  │  (Client 3)  │               │
│   │   PID 101    │   PID 102    │   PID 103    │               │
│   └──────────────┴──────────────┴──────────────┘               │
│          │              │              │                        │
│          └──────────────┼──────────────┘                        │
│                         ▼                                       │
│              ┌─────────────────────┐                            │
│              │    Shared Memory    │                            │
│              │  ┌───────────────┐  │                            │
│              │  │ Shared Buffers│  │                            │
│              │  │   WAL Buffers │  │                            │
│              │  │  Lock Tables  │  │                            │
│              │  └───────────────┘  │                            │
│              └─────────────────────┘                            │
│                                                                 │
│   Background Processes:                                         │
│   ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌──────────────┐     │
│   │ BGWriter │ │Checkpointer│ │WAL Writer│ │ Autovacuum   │     │
│   └──────────┘ └───────────┘ └──────────┘ └──────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why processes instead of threads?**

1. **Fault Isolation**: If a backend crashes (e.g., due to a bug or `kill -9`), it doesn't bring down the entire server. Other connections continue working.

2. **Simplicity**: No need for complex thread synchronization within a backend. Each process has its own memory space.

3. **Portability**: The model works identically across all Unix-like systems.

4. **Security**: Process boundaries provide natural isolation between connections.

The trade-off is **resource overhead**. Each backend process consumes memory (typically 5-10 MB baseline, plus work memory), and context switching between processes has a cost. This is why connection poolers like PgBouncer are essential for high-connection-count workloads.

### The Query Processing Pipeline

Once a query reaches a backend process, it passes through five distinct stages:

```
┌─────────────────────────────────────────────────────────────────┐
│                        SQL Query String                          │
│              "SELECT name FROM users WHERE id = 1"               │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 1: PARSER                                                 │
│  ┌─────────┐    ┌─────────┐                                     │
│  │  Lexer  │───▶│ Parser  │───▶ Raw Parse Tree                  │
│  │ scan.l  │    │ gram.y  │                                     │
│  └─────────┘    └─────────┘                                     │
│  • Tokenizes SQL text                                           │
│  • Validates syntax                                             │
│  • No semantic checks (doesn't know if tables exist)            │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 2: ANALYZER (Transformation Process)                     │
│  • Resolves table and column names                              │
│  • Looks up types in system catalogs                            │
│  • Transforms FuncCall → FuncExpr or Aggref                     │
│  • Produces Query Tree                                          │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 3: REWRITER                                              │
│  • Applies rewrite rules from pg_rewrite                        │
│  • Expands views into their underlying queries                  │
│  • Applies row-level security policies                          │
│  • May produce multiple query trees                             │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 4: PLANNER / OPTIMIZER                                   │
│  • Generates possible execution paths                           │
│  • Estimates costs using statistics                             │
│  • Chooses cheapest path                                        │
│  • Produces Plan Tree                                           │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 5: EXECUTOR                                              │
│  • Initializes plan state (ExecInitNode)                        │
│  • Processes nodes in demand-pull fashion (ExecProcNode)        │
│  • Returns rows to client                                       │
│  • Cleans up (ExecEndNode)                                      │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │ Query Results │
                              └───────────────┘
```

Let's briefly examine each stage:

#### Stage 1: Parser

The parser's job is purely syntactic. It answers the question: "Is this valid SQL grammar?" It doesn't know whether table `users` exists or whether column `name` is valid. The parser consists of:

- **Lexer** (`scan.l`): Breaks the SQL string into tokens (keywords, identifiers, literals, operators)
- **Parser** (`gram.y`): Applies grammar rules to build a parse tree

The output is a **raw parse tree**—a tree of `Node` structures representing the syntactic structure.

#### Stage 2: Analyzer

The analyzer (transformation process) adds semantic meaning. It consults system catalogs (`pg_class`, `pg_attribute`, etc.) to:

- Verify that referenced tables and columns exist
- Resolve column types
- Transform syntactic constructs to semantic ones (e.g., `FuncCall` → `FuncExpr`)

The output is a **query tree**—structurally similar to the parse tree but enriched with type information and resolved references.

#### Stage 3: Rewriter

The rewriter applies transformation rules stored in `pg_rewrite`. The most common use is **view expansion**: when you query a view, the rewriter substitutes the view's definition. It also handles:

- Non-`SELECT` rules (though rarely used)
- Row-level security policy injection
- Generated columns

#### Stage 4: Planner/Optimizer

The planner is the brain of PostgreSQL. Given a query tree, it must decide *how* to execute the query. For a simple `SELECT`, there might be few choices. For a complex join of 10 tables, there could be millions of possible execution strategies.

The planner:
1. Generates **paths** (potential execution strategies)
2. Estimates **costs** using table statistics
3. Selects the cheapest path
4. Converts the path to a **plan tree**

#### Stage 5: Executor

The executor runs the plan. It uses a **demand-pull** (iterator) model: the top node asks its children for rows, which ask their children, recursively down to the leaf nodes that scan actual tables.

---

## Deep Dive: Implementation Details

### Connection Establishment

When a client connects to PostgreSQL, the following sequence occurs:

```
Client                    Postmaster                  Backend
  │                           │                          │
  │── TCP SYN ───────────────▶│                          │
  │◀─ TCP SYN-ACK ────────────│                          │
  │── TCP ACK ───────────────▶│                          │
  │                           │                          │
  │── Startup Message ───────▶│                          │
  │   (protocol, user, db)    │                          │
  │                           │── fork() ───────────────▶│
  │                           │                          │
  │◀────────────────────────────── AuthenticationOk ─────│
  │◀────────────────────────────── ParameterStatus ──────│
  │◀────────────────────────────── BackendKeyData ───────│
  │◀────────────────────────────── ReadyForQuery ────────│
  │                           │                          │
  │                    (postmaster                       │
  │                     continues                        │
  │                     listening)                       │
```

The key function in the postmaster is `ServerLoop()` in `src/backend/postmaster/postmaster.c`:

```c
/* Simplified from src/backend/postmaster/postmaster.c */
static void
ServerLoop(void)
{
    for (;;)
    {
        /* Wait for connection or signal */
        selres = select(nfds, &rmask, NULL, NULL, &timeout);
        
        if (FD_ISSET(socket, &rmask))
        {
            /* New connection request */
            Port *port = ConnCreate(socket);
            
            /* Fork a new backend */
            BackendStartup(port);
        }
        
        /* Handle signals, check child processes, etc. */
    }
}
```

`BackendStartup()` calls `fork()` to create a new process. The child process (backend) then calls `BackendInitialize()` and `PostgresMain()`, which is the main loop for query processing.

### The Backend Main Loop

Each backend runs in `PostgresMain()` (in `src/backend/tcop/postgres.c`), essentially:

```c
/* Simplified from src/backend/tcop/postgres.c */
void
PostgresMain(const char *dbname, const char *username)
{
    /* Initialize memory contexts, signals, etc. */
    
    for (;;)
    {
        /* Read a command from the client */
        firstchar = ReadCommand(&input_message);
        
        switch (firstchar)
        {
            case 'Q':  /* Simple query */
                exec_simple_query(input_message.data);
                break;
                
            case 'P':  /* Parse (extended protocol) */
                exec_parse_message(...);
                break;
                
            case 'B':  /* Bind */
                exec_bind_message(...);
                break;
                
            case 'E':  /* Execute */
                exec_execute_message(...);
                break;
                
            case 'X':  /* Terminate */
                proc_exit(0);
                break;
                
            /* ... other message types ... */
        }
    }
}
```

For a simple query, `exec_simple_query()` orchestrates the entire pipeline:

```c
/* Simplified from src/backend/tcop/postgres.c */
static void
exec_simple_query(const char *query_string)
{
    /* STAGE 1: Parse */
    parsetree_list = pg_parse_query(query_string);
    
    foreach(parsetree_item, parsetree_list)
    {
        RawStmt *parsetree = lfirst(parsetree_item);
        
        /* STAGE 2 & 3: Analyze and Rewrite */
        querytree_list = pg_analyze_and_rewrite(parsetree, ...);
        
        /* STAGE 4: Plan */
        plantree_list = pg_plan_queries(querytree_list, ...);
        
        /* STAGE 5: Execute via Portal */
        portal = CreatePortal("", true, true);
        PortalDefineQuery(portal, NULL, query_string, ...);
        PortalStart(portal, NULL, 0, ...);
        
        /* Fetch and send results */
        PortalRun(portal, FETCH_ALL, ...);
        
        PortalDrop(portal, false);
    }
}
```

### Key Data Structures

Understanding the main data structures helps you navigate the source code:

#### Node (Base Type)

Almost everything in PostgreSQL's internal representation is a `Node`:

```c
/* From src/include/nodes/nodes.h */
typedef struct Node
{
    NodeTag     type;   /* Discriminator: what kind of node is this? */
} Node;

/* NodeTag examples */
typedef enum NodeTag
{
    T_Invalid = 0,
    
    /* Parse tree nodes */
    T_SelectStmt,
    T_InsertStmt,
    T_UpdateStmt,
    
    /* Query tree nodes */
    T_Query,
    T_RangeTblEntry,
    T_TargetEntry,
    
    /* Plan nodes */
    T_Plan,
    T_SeqScan,
    T_IndexScan,
    T_NestLoop,
    T_HashJoin,
    
    /* ... hundreds more ... */
} NodeTag;
```

#### RawStmt (Parse Tree Root)

The parser produces a list of `RawStmt` nodes:

```c
/* From src/include/nodes/parsenodes.h */
typedef struct RawStmt
{
    NodeTag     type;
    Node       *stmt;          /* The actual parsed statement */
    int         stmt_location; /* Character offset in original query */
    int         stmt_len;      /* Length of statement */
} RawStmt;
```

#### Query (Query Tree)

After analysis, we have a `Query` node:

```c
/* From src/include/nodes/parsenodes.h (simplified) */
typedef struct Query
{
    NodeTag     type;
    CmdType     commandType;    /* SELECT, INSERT, UPDATE, DELETE, etc. */
    
    List       *rtable;         /* Range table (list of RangeTblEntry) */
    List       *targetList;     /* Target list (list of TargetEntry) */
    Node       *jointree;       /* JOIN tree (FromExpr) */
    Node       *quals;          /* WHERE clause */
    
    List       *groupClause;    /* GROUP BY clause */
    Node       *havingQual;     /* HAVING clause */
    List       *sortClause;     /* ORDER BY clause */
    Node       *limitCount;     /* LIMIT count */
    Node       *limitOffset;    /* LIMIT offset */
    
    /* ... many more fields ... */
} Query;
```

#### PlannedStmt (Plan Tree Root)

The planner produces a `PlannedStmt`:

```c
/* From src/include/nodes/plannodes.h (simplified) */
typedef struct PlannedStmt
{
    NodeTag     type;
    CmdType     commandType;
    
    struct Plan *planTree;      /* The actual plan tree */
    List       *rtable;         /* Range table */
    
    /* Execution parameters */
    Bitmapset  *rewindPlanIDs;
    List       *rowMarks;
    List       *relationOids;   /* OIDs of relations accessed */
    
    /* ... more fields ... */
} PlannedStmt;
```

---

## Practical Examples and Tracing

### Observing Query Processing Stages

You can observe PostgreSQL's query processing using several techniques:

#### 1. Using EXPLAIN to See the Plan

```sql
EXPLAIN SELECT name FROM users WHERE id = 1;
```

Output:
```
                               QUERY PLAN
-------------------------------------------------------------------------
 Index Scan using users_pkey on users  (cost=0.29..8.30 rows=1 width=32)
   Index Cond: (id = 1)
```

This shows the executor's plan (Stage 5), which was chosen by the planner (Stage 4).

#### 2. Viewing Parse Trees with debug_print_parse

Enable in `postgresql.conf` or per-session:

```sql
SET debug_print_parse = on;
SET client_min_messages = log;

SELECT name FROM users WHERE id = 1;
```

Check the server log for the parse tree output.

#### 3. Viewing Query Trees with debug_print_rewritten

```sql
SET debug_print_rewritten = on;
```

#### 4. Viewing Plan Trees with debug_print_plan

```sql
SET debug_print_plan = on;
```

### Tracing a Simple Query Through the Code

Let's trace `SELECT 1;` through the code:

1. **Network receive**: `pq_getmessage()` receives the query string

2. **PostgresMain**: Receives 'Q' message, calls `exec_simple_query("SELECT 1")`

3. **Parser**: `pg_parse_query()` calls the flex/bison parser
   - Returns: `List` containing one `RawStmt` with a `SelectStmt` node

4. **Analyzer**: `pg_analyze_and_rewrite()` → `parse_analyze()`
   - `transformStmt()` handles the `SelectStmt`
   - Returns: `Query` node with `commandType = CMD_SELECT`

5. **Rewriter**: `pg_rewrite_query()`
   - No rules apply to `SELECT 1`
   - Returns the query unchanged

6. **Planner**: `pg_plan_query()` → `planner()` → `standard_planner()`
   - Creates a trivial `Result` plan node (no table access needed)

7. **Executor**: `PortalRun()` executes the `Result` node
   - Returns a single row with value `1`

### Using log_statement for Query Logging

```sql
SET log_statement = 'all';
```

This logs all statements to the PostgreSQL log, showing what queries are received.

---

## Performance Implications

### Connection Overhead

Each new connection requires:
1. TCP handshake (~1ms on LAN)
2. Process fork (~1-5ms)
3. Backend initialization (~5-10ms)
4. Authentication (~variable)

For short queries, connection time can dominate. **Connection pooling** (PgBouncer, Pgpool-II) is essential for workloads with many short-lived connections.

### The max_connections Trade-off

```sql
SHOW max_connections;  -- Default: 100
```

Each connection consumes:
- ~5-10 MB base memory
- A slot in the process table
- Shared memory structures

Setting `max_connections = 10000` without a connection pooler is almost always wrong. High connection counts cause:
- Memory exhaustion
- Lock contention on shared resources
- Context switch overhead

**Rule of thumb**: Keep `max_connections` at 2-4× your CPU core count for direct connections. Use a pooler for application connections.

### Parsing Overhead and Prepared Statements

For frequently-executed queries, parsing overhead can matter. **Prepared statements** skip the parse stage on subsequent executions:

```sql
PREPARE get_user(int) AS SELECT name FROM users WHERE id = $1;
EXECUTE get_user(1);
EXECUTE get_user(2);  -- No re-parsing!
```

With the extended query protocol (used by most drivers), prepared statements are automatic.

---

## Advanced Topics and Edge Cases

### The Extended Query Protocol

The simple query protocol sends one query string and gets results. The extended protocol separates:

1. **Parse**: Create a prepared statement
2. **Bind**: Attach parameter values
3. **Describe**: Get result column descriptions
4. **Execute**: Run the statement
5. **Close**: Clean up

This enables:
- Binary parameter/result formats
- Prepared statement caching
- Server-side cursors

### Transaction-Control Commands

Commands like `BEGIN`, `COMMIT`, and `ROLLBACK` are handled specially. The parser identifies them from the raw parse tree *before* starting a transaction, avoiding the need to analyze them:

```c
/* Check if we need to start a transaction */
if (!IsTransactionBlock() && 
    !IsAbortedTransactionBlockState())
{
    if (stmt requires transaction)
    {
        StartTransactionCommand();
    }
}
```

### Parallel Query Pipeline

PostgreSQL 9.6+ supports parallel query. The leader process spawns **parallel workers** (background processes) that share the workload. The `Gather` node collects results:

```
                    ┌──────────────────┐
                    │      Gather      │
                    │ (Leader Process) │
                    └────────┬─────────┘
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │  Worker  │  │  Worker  │  │  Worker  │
        │ (Scan)   │  │ (Scan)   │  │ (Scan)   │
        └──────────┘  └──────────┘  └──────────┘
```

---

## Hands-On Exercises

### Exercise 1: Connection Observation (Basic)

**Objective**: Observe PostgreSQL's process-per-connection model.

1. Open terminal and connect to PostgreSQL:
   ```bash
   psql -d postgres
   ```

2. In another terminal, find your backend process:
   ```bash
   ps aux | grep postgres
   ```

3. Note the PID of your backend. Open another psql connection and observe a new process appears.

4. Use `pg_stat_activity` to see connections:
   ```sql
   SELECT pid, usename, application_name, state, query 
   FROM pg_stat_activity 
   WHERE datname = current_database();
   ```

**Expected Outcome**: You see separate processes for each connection.

### Exercise 2: Query Stage Tracing (Intermediate)

**Objective**: Observe the parse, rewrite, and plan trees.

1. Enable debug output:
   ```sql
   SET client_min_messages = 'log';
   SET debug_print_parse = on;
   SET debug_print_rewritten = on;
   SET debug_print_plan = on;
   ```

2. Create a test table:
   ```sql
   CREATE TABLE test_trace (id serial, name text);
   INSERT INTO test_trace (name) VALUES ('Alice'), ('Bob');
   ```

3. Run a query and examine the log output:
   ```sql
   SELECT * FROM test_trace WHERE id = 1;
   ```

4. Check the PostgreSQL log file for the tree structures.

**Expected Outcome**: You see the three tree representations in the log.

### Exercise 3: Measuring Connection Overhead (Intermediate)

**Objective**: Quantify the cost of connection establishment.

1. Measure time to connect and disconnect 100 times:
   ```bash
   time for i in {1..100}; do psql -c "SELECT 1" >/dev/null 2>&1; done
   ```

2. Compare with keeping a connection open:
   ```bash
   time psql -c "SELECT generate_series(1,100)" >/dev/null 2>&1
   ```

**Expected Outcome**: Connection overhead is significant for many short queries.

### Exercise 4: Prepared Statement Benefits (Intermediate)

**Objective**: Measure the performance benefit of prepared statements.

```sql
-- Without preparation (1000 iterations)
\timing on
DO $$
BEGIN
  FOR i IN 1..1000 LOOP
    PERFORM * FROM pg_class WHERE oid = i;
  END LOOP;
END $$;

-- With preparation
PREPARE test_prep (oid) AS SELECT * FROM pg_class WHERE oid = $1;
DO $$
BEGIN
  FOR i IN 1..1000 LOOP
    EXECUTE 'EXECUTE test_prep($1)' USING i;
  END LOOP;
END $$;
```

### Exercise 5: Explore Backend State (Advanced)

**Objective**: Use system functions to explore backend state.

```sql
-- Current backend PID
SELECT pg_backend_pid();

-- See locks held by your session
SELECT * FROM pg_locks WHERE pid = pg_backend_pid();

-- Terminate another session (careful!)
-- SELECT pg_terminate_backend(pid);
```

---

## Key Takeaways

1. **PostgreSQL uses a process-per-connection model**: Each client connection gets its own backend process, providing isolation but consuming resources.

2. **The postmaster is the gatekeeper**: It listens for connections and spawns backend processes. It also monitors child processes and handles cleanup.

3. **Query processing has five stages**: Parse → Analyze → Rewrite → Plan → Execute. Each stage transforms the query representation.

4. **The parser only checks syntax**: Semantic validation (table existence, types) happens in the analyzer.

5. **The planner chooses among alternatives**: It generates multiple paths and picks the cheapest based on cost estimation.

6. **The executor uses demand-pull**: Rows flow from leaf nodes up through the plan tree on demand.

7. **Connection pooling is essential** for high-connection-count workloads to avoid resource exhaustion and fork overhead.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/postmaster/postmaster.c` | The postmaster main loop and connection handling |
| `src/backend/tcop/postgres.c` | Backend main loop (`PostgresMain`) |
| `src/backend/parser/parser.c` | Parser entry point |
| `src/backend/parser/analyze.c` | Semantic analysis (transformation) |
| `src/backend/rewrite/rewriteHandler.c` | Query rewriting |
| `src/backend/optimizer/plan/planner.c` | Planner entry point |
| `src/backend/executor/execMain.c` | Executor main entry points |

### Documentation

- [PostgreSQL Docs: The Path of a Query](https://www.postgresql.org/docs/current/query-path.html)
- [PostgreSQL Docs: How Connections Are Established](https://www.postgresql.org/docs/current/connect-estab.html)
- [PostgreSQL Wiki: Developer Information](https://wiki.postgresql.org/wiki/Developer_Information)

### Next Lessons

- **Lesson 2**: The Parser — Deep dive into `scan.l`, `gram.y`, and parse tree structure
- **Lesson 3**: The Analyzer — Semantic analysis and the transformation process
- **Lesson 4**: The Rewriter — View expansion and the rule system

---

## References

### Source Code

- `src/backend/postmaster/postmaster.c` — Postmaster process
- `src/backend/tcop/postgres.c` — Backend main loop
- `src/include/nodes/nodes.h` — Node type definitions
- `src/include/nodes/parsenodes.h` — Parse/Query tree nodes
- `src/include/nodes/plannodes.h` — Plan tree nodes

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `ServerLoop()` | postmaster.c | Main postmaster loop |
| `BackendStartup()` | postmaster.c | Forks new backend |
| `PostgresMain()` | postgres.c | Backend main loop |
| `exec_simple_query()` | postgres.c | Handles simple query protocol |
| `pg_parse_query()` | postgres.c | Entry to parser |
| `pg_analyze_and_rewrite()` | postgres.c | Analyze + rewrite |
| `pg_plan_queries()` | postgres.c | Entry to planner |
| `PortalRun()` | pquery.c | Executes via portal |

### External Resources

- Suzuki, H. "The Internals of PostgreSQL" — Chapter 3: Query Processing
- PostgreSQL source code: https://github.com/postgres/postgres
