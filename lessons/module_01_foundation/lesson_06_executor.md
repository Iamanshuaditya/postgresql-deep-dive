# Lesson 6: The Executor — Running the Query Plan

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 1 — Foundation: From SQL to Execution |
| **Lesson** | 6 of 66 |
| **Estimated Time** | 60-75 minutes |
| **Prerequisites** | Lesson 5: Introduction to the Query Planner |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain the demand-pull (Volcano) execution model
2. Describe the three phases: initialization, execution, and cleanup
3. Understand how data flows through the plan tree
4. Trace execution through common node types
5. Interpret EXPLAIN ANALYZE output for execution timing

### Key Terms

| Term | Definition |
|------|------------|
| **Executor** | The component that runs the plan tree and produces results |
| **Plan Node** | A single operation in the plan tree (scan, join, sort, etc.) |
| **Plan State** | Runtime state for a plan node during execution |
| **Demand-Pull** | Execution model where parent nodes request rows from children |
| **TupleTableSlot** | A "container" holding a single tuple during processing |
| **ExecInitNode** | Initialize a plan node for execution |
| **ExecProcNode** | Get the next tuple from a plan node |
| **ExecEndNode** | Clean up a plan node after execution |

---

## Introduction

We've traced a query through parsing, analysis, rewriting, and planning. Now we have a plan tree—a recipe for how to retrieve the data. The **executor** is the chef that follows this recipe to actually produce results.

The executor takes the plan tree and:

1. **Initializes** each node (allocates buffers, opens files)
2. **Executes** by pulling tuples through the tree
3. **Cleans up** after completion (frees resources, closes files)

PostgreSQL uses the **Volcano model** (also called the iterator or demand-pull model), which is common in relational databases. Each node in the plan tree acts as an iterator: it has an `open`, `next`, and `close` method. The root node asks for rows, which cascades down the tree until leaf nodes fetch actual data.

This model is elegant and composable—you can add new node types without changing the framework. A Sort node doesn't care whether its input comes from a SeqScan, IndexScan, or another Sort. It just calls `next()` on its child and gets tuples.

Understanding the executor helps you:

1. **Interpret EXPLAIN ANALYZE timing**: The "actual time" numbers tell you where time is spent
2. **Understand memory usage**: Sort and Hash nodes consume work_mem
3. **Debug slow queries**: Is the bottleneck in scanning, joining, or post-processing?
4. **Reason about parallel execution**: Parallel workers run their own executor instances

> **The Demand-Pull Principle**: Data flows upward, control flows downward. The root says "give me a row," and this request propagates down until a leaf node (a table scan) fetches actual data and passes it back up.

---

## Conceptual Foundation

### The Volcano/Iterator Model

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client                                    │
│                          │                                       │
│                    "Next row, please"                            │
│                          ▼                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      Sort Node                            │  │
│  │   (needs all input rows before producing output)          │  │
│  │                          │                                 │  │
│  │                    "Next row"                              │  │
│  │                          ▼                                 │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │                   Hash Join                         │  │  │
│  │  │                    /      \                         │  │  │
│  │  │              "Next row"  "Build hash table"         │  │  │
│  │  │                  /            \                     │  │  │
│  │  │  ┌─────────────────┐  ┌─────────────────┐          │  │  │
│  │  │  │    Seq Scan     │  │   Index Scan    │          │  │  │
│  │  │  │   (orders)      │  │  (customers)    │          │  │  │
│  │  │  └─────────────────┘  └─────────────────┘          │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Data Flow: ↑ (tuples flow up)                                  │
│  Control Flow: ↓ ("next" requests flow down)                    │
└─────────────────────────────────────────────────────────────────┘
```

Each node implements a simple interface:

| Method | Purpose |
|--------|---------|
| `open()` / `Init` | Allocate resources, initialize state |
| `next()` / `Proc` | Return the next tuple, or NULL if done |
| `close()` / `End` | Release resources, clean up |

### The Three Phases of Execution

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: INITIALIZATION (ExecutorStart)                        │
│                                                                  │
│  • Traverse plan tree, call ExecInitNode for each node          │
│  • Create PlanState nodes (runtime state)                       │
│  • Allocate TupleTableSlots for tuple storage                   │
│  • Open relations, acquire locks                                 │
│  • Set up expression evaluation state                           │
│                                                                  │
│  Result: Plan State Tree (ready to execute)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2: EXECUTION (ExecutorRun)                               │
│                                                                  │
│  Loop:                                                          │
│    • Call ExecProcNode on root → cascades down tree             │
│    • Leaf nodes fetch from storage, return tuples               │
│    • Intermediate nodes process and pass tuples up              │
│    • Root returns tuple to client                               │
│    • Repeat until NULL (no more tuples) or LIMIT reached        │
│                                                                  │
│  Result: Query results sent to client                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3: CLEANUP (ExecutorEnd)                                 │
│                                                                  │
│  • Traverse plan state tree, call ExecEndNode for each          │
│  • Free memory (sort buffers, hash tables)                      │
│  • Close relations, release locks                               │
│  • Update statistics (rows processed, etc.)                     │
│                                                                  │
│  Result: Resources freed, ready for next query                  │
└─────────────────────────────────────────────────────────────────┘
```

### Plan Node vs. Plan State

The planner produces a **Plan tree** with static information. The executor creates a **PlanState tree** with runtime information:

```
Plan Tree (from planner):           PlanState Tree (in executor):
─────────────────────────           ────────────────────────────
SeqScan {                           SeqScanState {
  scanrelid: 1               ──►      plan: → SeqScan
  qual: [...]                         state: → EState
}                                     ss_currentRelation: → open Relation
                                      ss_currentScanDesc: → HeapScanDesc
                                      ss_ScanTupleSlot: → TupleTableSlot
                                      ps_ResultTupleSlot: → TupleTableSlot
                                    }
```

### TupleTableSlot: The Tuple Container

Tuples are passed around in **TupleTableSlots**—a universal container:

```c
/* Simplified from src/include/executor/tuptable.h */
typedef struct TupleTableSlot
{
    NodeTag     type;
    uint16      tts_flags;      /* Various flags */
    TupleDesc   tts_tupleDescriptor;  /* Row descriptor (column types) */
    
    /* Values and nulls arrays (for virtual tuples) */
    Datum      *tts_values;
    bool       *tts_isnull;
    
    /* For heap tuples */
    HeapTuple   tts_tuple;
    MinimalTuple tts_mintuple;
} TupleTableSlot;
```

Slots can hold tuples in different forms:
- **Virtual**: Just Datum/isnull arrays (for computed values)
- **Heap**: A proper HeapTuple (from disk or index)
- **Minimal**: Compact form without header (for hash tables)

---

## Deep Dive: Implementation Details

### Executor Entry Points

The executor is invoked from `PortalRun()`:

```c
/* From src/backend/executor/execMain.c */

/* Phase 1: Initialize */
void
ExecutorStart(QueryDesc *queryDesc, int eflags)
{
    EState     *estate;
    PlanState  *planstate;
    
    /* Create executor state */
    estate = CreateExecutorState();
    queryDesc->estate = estate;
    
    /* Set up tuple receiver */
    estate->es_output_cid = GetCurrentCommandId(false);
    
    /* Initialize plan tree */
    planstate = ExecInitNode(queryDesc->plannedstmt->planTree, 
                              estate, eflags);
    queryDesc->planstate = planstate;
}

/* Phase 2: Execute */
void
ExecutorRun(QueryDesc *queryDesc, ScanDirection direction, 
            uint64 count, bool execute_once)
{
    PlanState  *planstate = queryDesc->planstate;
    TupleTableSlot *slot;
    uint64      nprocessed = 0;
    
    /* Fetch tuples one at a time */
    while (nprocessed < count)
    {
        slot = ExecProcNode(planstate);
        
        if (TupIsNull(slot))
            break;  /* No more tuples */
        
        /* Send tuple to destination (client, table, etc.) */
        (*dest->receiveSlot)(slot, dest);
        nprocessed++;
    }
}

/* Phase 3: Cleanup */
void
ExecutorEnd(QueryDesc *queryDesc)
{
    /* Cleanup plan tree */
    ExecEndNode(queryDesc->planstate);
    
    /* Free executor state */
    FreeExecutorState(queryDesc->estate);
}
```

### ExecInitNode: The Dispatcher

`ExecInitNode` dispatches to node-specific initialization:

```c
/* From src/backend/executor/execProcnode.c */
PlanState *
ExecInitNode(Plan *node, EState *estate, int eflags)
{
    PlanState  *result;
    
    if (node == NULL)
        return NULL;
    
    switch (nodeTag(node))
    {
        case T_SeqScan:
            result = (PlanState *) ExecInitSeqScan((SeqScan *) node,
                                                    estate, eflags);
            break;
        case T_IndexScan:
            result = (PlanState *) ExecInitIndexScan((IndexScan *) node,
                                                      estate, eflags);
            break;
        case T_NestLoop:
            result = (PlanState *) ExecInitNestLoop((NestLoop *) node,
                                                     estate, eflags);
            break;
        case T_HashJoin:
            result = (PlanState *) ExecInitHashJoin((HashJoin *) node,
                                                     estate, eflags);
            break;
        case T_Sort:
            result = (PlanState *) ExecInitSort((Sort *) node,
                                                 estate, eflags);
            break;
        case T_Agg:
            result = (PlanState *) ExecInitAgg((Agg *) node,
                                                estate, eflags);
            break;
        /* ... many more node types ... */
    }
    
    return result;
}
```

### ExecProcNode: The Heart of Execution

```c
/* From src/backend/executor/execProcnode.c */
static inline TupleTableSlot *
ExecProcNode(PlanState *node)
{
    TupleTableSlot *result;
    
    /* Call node's execution function */
    result = node->ExecProcNode(node);
    
    return result;
}
```

Each node type has its own `ExecProcNode` function. For a SeqScan:

```c
/* From src/backend/executor/nodeSeqscan.c (simplified) */
static TupleTableSlot *
ExecSeqScan(PlanState *pstate)
{
    SeqScanState *node = castNode(SeqScanState, pstate);
    TupleTableSlot *slot;
    
    /* Get next tuple from heap */
    slot = SeqNext(node);
    
    /* If no more tuples, return empty slot */
    if (TupIsNull(slot))
        return slot;
    
    /* Check qualification expression */
    if (node->ps.qual)
    {
        bool passesQual = ExecQual(node->ps.qual, econtext);
        if (!passesQual)
            return ExecSeqScan(pstate);  /* Skip, get next */
    }
    
    /* Return the tuple */
    return slot;
}
```

### Expression Evaluation

Expressions (WHERE clauses, projections) are evaluated using `ExecEvalExpr`:

```c
/* Expression evaluation is done via a compiled/JIT form */
/* The ExprState has a function pointer for evaluation */

/* From src/backend/executor/execExprInterp.c */
Datum
ExecInterpExpr(ExprState *state, ExprContext *econtext, bool *isNull)
{
    /* Walk through the expression steps */
    while (true)
    {
        struct ExprEvalStep *step = state->steps;
        
        switch (step->opcode)
        {
            case EEOP_CONST:
                /* Return a constant value */
                *isNull = step->d.constval.isnull;
                return step->d.constval.value;
                
            case EEOP_FUNCEXPR:
                /* Call a function */
                return ExecEvalFunc(state, step, econtext, isNull);
                
            case EEOP_QUAL:
                /* Check a boolean expression */
                if (!DatumGetBool(value))
                    return (Datum) 0;
                break;
                
            /* ... many more opcodes ... */
        }
    }
}
```

### Scan Nodes

Scan nodes are the leaves of the plan tree—they fetch data from storage:

```c
/* Sequential Scan: Read heap pages in order */
static TupleTableSlot *
SeqNext(SeqScanState *node)
{
    HeapScanDesc scan = node->ss_currentScanDesc;
    HeapTuple    tuple;
    
    /* Get next tuple from heap */
    tuple = heap_getnext(scan, ForwardScanDirection);
    
    if (tuple == NULL)
        return ExecClearTuple(slot);  /* No more tuples */
    
    /* Store tuple in slot */
    ExecStoreBufferHeapTuple(tuple, slot, scan->rs_cbuf);
    
    return slot;
}
```

### Join Nodes

Join nodes combine tuples from two inputs:

```c
/* Nested Loop: For each outer, scan inner */
static TupleTableSlot *
ExecNestLoop(PlanState *pstate)
{
    NestLoopState *node = castNode(NestLoopState, pstate);
    TupleTableSlot *outerTupleSlot;
    TupleTableSlot *innerTupleSlot;
    
    /* Get current outer tuple */
    outerTupleSlot = node->js.ps.ps_OuterTupleSlot;
    
    for (;;)
    {
        /* Need new outer tuple? */
        if (node->nl_NeedNewOuter)
        {
            outerTupleSlot = ExecProcNode(outerPlan(node));
            
            if (TupIsNull(outerTupleSlot))
                return NULL;  /* Outer exhausted */
            
            /* Rescan inner for this outer */
            ExecReScan(innerPlan(node));
            node->nl_NeedNewOuter = false;
        }
        
        /* Get next inner tuple */
        innerTupleSlot = ExecProcNode(innerPlan(node));
        
        if (TupIsNull(innerTupleSlot))
        {
            /* Inner exhausted, need new outer */
            node->nl_NeedNewOuter = true;
            continue;
        }
        
        /* Check join condition */
        if (ExecQual(node->js.joinqual, econtext))
        {
            /* Build result tuple */
            return ExecProject(node->js.ps.ps_ProjInfo);
        }
    }
}
```

---

## Common Plan Node Types

### Scan Nodes

| Node | Description |
|------|-------------|
| `SeqScan` | Read all heap pages sequentially |
| `IndexScan` | Use index to find matching rows, fetch from heap |
| `IndexOnlyScan` | Return data directly from covering index |
| `BitmapHeapScan` | Use bitmap to identify pages, then scan them |
| `TidScan` | Fetch rows by TID (ctid) |
| `FunctionScan` | Read from set-returning function |
| `ValuesScan` | Read from VALUES clause |
| `CteScan` | Read from CTE (WITH clause) |

### Join Nodes

| Node | Description |
|------|-------------|
| `NestLoop` | For each outer row, scan inner |
| `MergeJoin` | Merge two sorted inputs |
| `HashJoin` | Build hash table from inner, probe with outer |

### Materialization Nodes

| Node | Description |
|------|-------------|
| `Sort` | Sort input rows (uses work_mem, spills to disk) |
| `Materialize` | Buffer input rows for re-scanning |
| `Hash` | Build hash table for HashJoin |
| `Memoize` | Cache results for repeated lookups |

### Aggregation & Grouping

| Node | Description |
|------|-------------|
| `Agg` | Compute aggregates (SUM, COUNT, etc.) |
| `Group` | Group sorted input |
| `WindowAgg` | Window function aggregation |

### Set Operations

| Node | Description |
|------|-------------|
| `Append` | Concatenate inputs (UNION ALL) |
| `MergeAppend` | Merge sorted inputs |
| `SetOp` | UNION, INTERSECT, EXCEPT |
| `RecursiveUnion` | Recursive CTE execution |

### Other Nodes

| Node | Description |
|------|-------------|
| `Result` | Compute expressions without scanning |
| `Limit` | Stop after N rows |
| `Unique` | Remove duplicates from sorted input |
| `Gather` | Collect results from parallel workers |
| `GatherMerge` | Merge sorted results from parallel workers |

---

## Practical Examples and EXPLAIN ANALYZE

### Understanding EXPLAIN ANALYZE Output

```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders WHERE customer_id = 42 ORDER BY order_date;
```

```
                                                QUERY PLAN
-----------------------------------------------------------------------------------------------------------
 Sort  (cost=150.25..152.50 rows=50 width=120) (actual time=2.150..2.180 rows=47 loops=1)
   Sort Key: order_date
   Sort Method: quicksort  Memory: 30kB
   Buffers: shared hit=15
   ->  Index Scan using orders_customer_idx on orders  (cost=0.29..140.25 rows=50 width=120)
                                                       (actual time=0.030..1.850 rows=47 loops=1)
         Index Cond: (customer_id = 42)
         Buffers: shared hit=15
 Planning Time: 0.250 ms
 Execution Time: 2.350 ms
```

**Interpreting the output:**

| Component | Meaning |
|-----------|---------|
| `cost=150.25..152.50` | Estimated startup..total cost |
| `rows=50` | Estimated output rows |
| `actual time=2.150..2.180` | Real time in ms (startup..total) |
| `rows=47` | Actual rows returned |
| `loops=1` | How many times this node executed |
| `Sort Method: quicksort` | Algorithm used |
| `Memory: 30kB` | Memory consumed |
| `Buffers: shared hit=15` | Pages read from cache |

### Time Breakdown

The time shown is **inclusive**—it includes time spent in child nodes:

```
 Sort: actual time=2.150..2.180
   └── Index Scan: actual time=0.030..1.850

Sort exclusive time = 2.180 - 1.850 = 0.33 ms
Index Scan time = 1.850 ms
Total time ≈ 2.18 ms (matches Sort total)
```

### loops and Per-Loop Time

For nested loops and correlated subqueries:

```sql
EXPLAIN ANALYZE
SELECT o.*, (SELECT name FROM customers c WHERE c.id = o.customer_id)
FROM orders o
WHERE o.total > 1000;
```

```
 Seq Scan on orders o  (actual time=0.015..5.200 rows=150 loops=1)
   Filter: (total > 1000)
   SubPlan 1
     ->  Index Scan using customers_pkey on customers c
               (actual time=0.025..0.030 rows=1 loops=150)
           Index Cond: (id = o.customer_id)
```

The subplan runs **150 times** (once per order row). Real subplan time = 0.03 × 150 = 4.5ms

### Buffer Statistics

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM large_table;
```

```
 Seq Scan on large_table  (actual time=0.010..450.000 rows=1000000 loops=1)
   Buffers: shared hit=5000 read=3000 dirtied=0 written=0
```

| Metric | Meaning |
|--------|---------|
| `shared hit` | Pages found in buffer cache |
| `read` | Pages read from disk |
| `dirtied` | Pages modified (for writes) |
| `written` | Pages flushed to disk |

High `read` vs `hit` ratio indicates poor caching.

---

## Performance Implications

### work_mem and Sorting

The executor uses `work_mem` for sorting and hashing:

```sql
-- See how sort behavior changes
SET work_mem = '64kB';
EXPLAIN ANALYZE SELECT * FROM orders ORDER BY order_date;
```

```
 Sort  (actual time=150.000..180.000 rows=100000 loops=1)
   Sort Key: order_date
   Sort Method: external merge  Disk: 15000kB  ← Spilled to disk!
```

```sql
SET work_mem = '256MB';
EXPLAIN ANALYZE SELECT * FROM orders ORDER BY order_date;
```

```
 Sort  (actual time=80.000..95.000 rows=100000 loops=1)
   Sort Key: order_date
   Sort Method: quicksort  Memory: 25000kB  ← In-memory, faster
```

### Hash Table Memory

Hash joins also use work_mem:

```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;
```

```
 Hash Join  (actual time=5.000..50.000 rows=100000 loops=1)
   Hash Cond: (o.customer_id = c.id)
   ->  Seq Scan on orders o  (...)
   ->  Hash  (actual time=4.500..4.500 rows=10000 loops=1)
         Buckets: 16384  Batches: 1  Memory Usage: 800kB
```

If `Batches` > 1, the hash table didn't fit in memory and batched to disk.

### Parallel Execution

For large scans, PostgreSQL can use parallel workers:

```sql
EXPLAIN ANALYZE SELECT count(*) FROM orders;
```

```
 Finalize Aggregate  (actual time=100.000..100.100 rows=1 loops=1)
   ->  Gather  (actual time=50.000..99.500 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (actual time=45.000..45.100 rows=1 loops=3)
               ->  Parallel Seq Scan on orders  (...)
```

The work is divided among 3 processes (leader + 2 workers).

---

## Hands-On Exercises

### Exercise 1: Basic EXPLAIN ANALYZE (Basic)

**Objective**: Understand execution flow.

```sql
CREATE TABLE exec_test (id serial PRIMARY KEY, val int, data text);
INSERT INTO exec_test (val, data) 
SELECT i % 100, 'Data ' || i FROM generate_series(1, 100000) i;
CREATE INDEX ON exec_test(val);
ANALYZE exec_test;

-- Compare these:
EXPLAIN ANALYZE SELECT * FROM exec_test WHERE id = 500;
EXPLAIN ANALYZE SELECT * FROM exec_test WHERE val = 50;
EXPLAIN ANALYZE SELECT * FROM exec_test WHERE val < 10 ORDER BY val;
```

**Questions**:
- Which nodes are used?
- How accurate are the row estimates?

### Exercise 2: Buffer Analysis (Intermediate)

**Objective**: Understand I/O patterns.

```sql
-- Clear OS cache if possible, or use pgbench for cold-cache test

-- First run (cold cache)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM exec_test;

-- Second run (warm cache)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM exec_test;
```

**Compare** the `shared hit` vs `read` values.

### Exercise 3: work_mem Impact (Intermediate)

**Objective**: See how memory affects execution.

```sql
-- Small work_mem
SET work_mem = '64kB';
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM exec_test ORDER BY data;

-- Large work_mem
SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM exec_test ORDER BY data;
```

**Compare** sort methods and execution times.

### Exercise 4: Join Execution (Intermediate)

**Objective**: Trace data flow through joins.

```sql
CREATE TABLE t1 (id serial PRIMARY KEY, a int);
CREATE TABLE t2 (id serial PRIMARY KEY, t1_id int, b int);
INSERT INTO t1 SELECT generate_series(1, 1000), random() * 100;
INSERT INTO t2 SELECT generate_series(1, 10000), random() * 1000, random() * 100;
CREATE INDEX ON t2(t1_id);
ANALYZE t1, t2;

-- What join method?
EXPLAIN ANALYZE SELECT * FROM t1 JOIN t2 ON t1.id = t2.t1_id;

-- Force nested loop
SET enable_hashjoin = off;
SET enable_mergejoin = off;
EXPLAIN ANALYZE SELECT * FROM t1 JOIN t2 ON t1.id = t2.t1_id;
```

**Observe** loops count and per-loop timing.

### Exercise 5: Subquery Execution (Advanced)

**Objective**: Understand correlated subquery execution.

```sql
EXPLAIN ANALYZE
SELECT t1.*, (SELECT count(*) FROM t2 WHERE t2.t1_id = t1.id) as cnt
FROM t1;

EXPLAIN ANALYZE
SELECT t1.*, t2_count.cnt
FROM t1 
LEFT JOIN LATERAL (
    SELECT count(*) as cnt FROM t2 WHERE t2.t1_id = t1.id
) t2_count ON true;
```

**Compare** the execution strategies.

---

## Key Takeaways

1. **The executor uses demand-pull**: Parent nodes request tuples from children; data flows upward, control flows downward.

2. **Three phases**: Init (set up state), Proc (fetch tuples), End (clean up).

3. **Plan nodes become plan states**: Runtime information like current positions and buffers.

4. **TupleTableSlots are the currency**: All tuple passing uses slots.

5. **EXPLAIN ANALYZE shows reality**: Compare actual vs. estimated for diagnosis.

6. **work_mem affects materialization**: Sorting and hashing use work_mem; exceeding it causes disk spills.

7. **loops tells you how often**: Critical for understanding nested loop and subquery costs.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/executor/execMain.c` | Executor entry points |
| `src/backend/executor/execProcnode.c` | Node dispatch functions |
| `src/backend/executor/nodeSeqscan.c` | Sequential scan |
| `src/backend/executor/nodeIndexscan.c` | Index scan |
| `src/backend/executor/nodeNestloop.c` | Nested loop join |
| `src/backend/executor/nodeHashjoin.c` | Hash join |
| `src/backend/executor/nodeSort.c` | Sort node |
| `src/include/executor/tuptable.h` | TupleTableSlot |

### Documentation

- [PostgreSQL Docs: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL Docs: Executor](https://www.postgresql.org/docs/current/executor.html)
- [PostgreSQL Internals: Query Processing](https://www.postgresql.org/docs/current/overview.html)

### Next Lessons

- **Lesson 7**: Understanding EXPLAIN in Depth
- **Module 10**: Advanced Execution Topics
- **Module 3**: Memory Management (work_mem, shared_buffers)

---

## References

### Source Code

- `src/backend/executor/` — All executor code
- `src/include/executor/` — Executor headers
- `src/include/nodes/execnodes.h` — Execution node structures
- `src/include/nodes/plannodes.h` — Plan node structures

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `ExecutorStart()` | execMain.c | Begin execution |
| `ExecutorRun()` | execMain.c | Fetch tuples |
| `ExecutorEnd()` | execMain.c | Cleanup |
| `ExecInitNode()` | execProcnode.c | Initialize node |
| `ExecProcNode()` | execProcnode.c | Get next tuple |
| `ExecEndNode()` | execProcnode.c | End node |

### Key Structures

| Structure | Location | Purpose |
|-----------|----------|---------|
| `EState` | execnodes.h | Executor global state |
| `PlanState` | execnodes.h | Base plan state |
| `ScanState` | execnodes.h | Scan node state |
| `JoinState` | execnodes.h | Join node state |
| `TupleTableSlot` | tuptable.h | Tuple container |
