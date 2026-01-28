# Lesson 3: The Analyzer — Semantic Analysis and Name Resolution

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 1 — Foundation: From SQL to Execution |
| **Lesson** | 3 of 66 |
| **Estimated Time** | 60-75 minutes |
| **Prerequisites** | Lesson 2: The Parser |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain the difference between syntactic and semantic analysis
2. Describe how PostgreSQL resolves table and column names
3. Understand the structure of the Query tree vs. the parse tree
4. Trace how expressions are transformed and typed
5. Identify common semantic errors and their causes

### Key Terms

| Term | Definition |
|------|------------|
| **Semantic Analysis** | Checking the meaning and validity of a query (tables exist, types match) |
| **Name Resolution** | Mapping identifiers to actual database objects (tables, columns, functions) |
| **Range Table** | A list of all relations referenced in a query, each identified by an index |
| **RangeTblEntry (RTE)** | An entry in the range table describing a single relation |
| **Query** | The node type representing a semantically-analyzed query |
| **Var** | A node representing a reference to a column in the query tree |
| **transformExpr** | The function family that converts parse tree expressions to query tree expressions |

---

## Introduction

In the previous lesson, we saw how the parser converts SQL text into a parse tree. But the parse tree is just a syntactic representation—it tells us the structure of the query, but not whether it makes sense. Consider:

```sql
SELECT unicorn FROM rainbow WHERE magic = 'sparkle';
```

This is syntactically valid SQL. The parser will happily produce a `SelectStmt` node. But does table `rainbow` exist? Does it have a column named `unicorn`? What is the type of `magic`? Is `'sparkle'` compatible with that type?

These are **semantic** questions, and answering them is the analyzer's job.

The analyzer (also called the transformation process) takes the raw parse tree and:

1. **Resolves names**: Looks up tables, columns, and functions in the system catalogs
2. **Checks types**: Ensures expressions have valid types and operations are defined
3. **Transforms nodes**: Converts parse tree nodes to query tree nodes with type information
4. **Validates permissions**: Checks that the user can access the referenced objects

The output is a **Query tree**—a richer representation that the planner can work with.

> **Why This Matters:** When you see an error like "column X does not exist" or "operator does not exist", it's the analyzer speaking. Understanding this stage helps you interpret these errors and understand what PostgreSQL knows about your schema.

---

## Conceptual Foundation

### Parse Tree vs. Query Tree

The parse tree and query tree look similar but serve different purposes:

```
┌─────────────────────────────────────────────────────────────────┐
│  PARSE TREE (from parser)                                        │
│                                                                  │
│  • Represents syntactic structure only                           │
│  • Names are just strings ("users", "name")                     │
│  • No type information                                           │
│  • No knowledge of database schema                               │
│  • FuncCall could be function or aggregate (unknown)             │
│                                                                  │
│  Key node types: SelectStmt, ColumnRef, A_Expr, A_Const          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Analyzer (parse_analyze)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  QUERY TREE (from analyzer)                                      │
│                                                                  │
│  • Represents semantic structure                                 │
│  • Names resolved to OIDs and internal references               │
│  • Full type information on all expressions                      │
│  • References checked against catalog                            │
│  • FuncCall → FuncExpr (function) or Aggref (aggregate)         │
│                                                                  │
│  Key node types: Query, RangeTblEntry, Var, FuncExpr, Aggref    │
└─────────────────────────────────────────────────────────────────┘
```

### The Query Structure

The central node type after analysis is `Query`:

```c
/* From src/include/nodes/parsenodes.h (simplified) */
typedef struct Query
{
    NodeTag     type;
    
    CmdType     commandType;    /* CMD_SELECT, CMD_INSERT, etc. */
    QuerySource querySource;    /* Where this query came from */
    
    bool        canSetTag;      /* Can this query set command tag? */
    
    Node       *utilityStmt;    /* Utility statement (non-DML) or NULL */
    
    /* Main structural components */
    List       *rtable;         /* Range table (list of RangeTblEntry) */
    FromExpr   *jointree;       /* JOIN tree plus WHERE quals */
    List       *targetList;     /* Target list (list of TargetEntry) */
    
    /* GROUP BY / HAVING */
    List       *groupClause;
    Node       *havingQual;
    
    /* Window functions */
    List       *windowClause;
    
    /* DISTINCT */
    List       *distinctClause;
    
    /* ORDER BY / LIMIT */
    List       *sortClause;
    Node       *limitOffset;
    Node       *limitCount;
    
    /* Row locking */
    List       *rowMarks;
    
    /* CTEs */
    List       *cteList;
    
    /* Subqueries and set operations */
    Node       *setOperations;
    
    /* ... many more fields ... */
} Query;
```

### The Range Table

The **range table** is a list of all "range variables" (tables, subqueries, joins, functions) that appear in the query:

```
┌─────────────────────────────────────────────────────────────────┐
│  Query: SELECT u.name, o.total FROM users u JOIN orders o ...   │
├─────────────────────────────────────────────────────────────────┤
│  Range Table:                                                    │
│                                                                  │
│  rtable[1] RangeTblEntry {                                       │
│      rtekind: RTE_RELATION                                       │
│      relid: 16384  (OID of users table)                         │
│      alias: "u"                                                  │
│  }                                                               │
│                                                                  │
│  rtable[2] RangeTblEntry {                                       │
│      rtekind: RTE_RELATION                                       │
│      relid: 16390  (OID of orders table)                        │
│      alias: "o"                                                  │
│  }                                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

The range table index (1, 2, etc.) is called the **varno** and is used to reference columns.

### Column References: Var Nodes

In the parse tree, column references are `ColumnRef` nodes with string names. In the query tree, they become `Var` nodes with numeric references:

```
Parse Tree:                    Query Tree:
────────────                   ──────────
ColumnRef {                    Var {
  fields: ["u", "name"]          varno: 1      (rtable index)
}                                varattno: 2   (column number)
                                 vartype: 25   (OID of text)
                               }
```

```c
/* From src/include/nodes/primnodes.h */
typedef struct Var
{
    Expr        xpr;
    int         varno;          /* range table index */
    AttrNumber  varattno;       /* attribute number (column) */
    Oid         vartype;        /* type OID */
    int32       vartypmod;      /* type modifier */
    Oid         varcollid;      /* collation OID */
    Index       varlevelsup;    /* nesting level (0 = current) */
    int         location;       /* token location */
} Var;
```

---

## Deep Dive: Implementation Details

### The Analysis Entry Point

Analysis begins with `parse_analyze()`:

```c
/* From src/backend/parser/analyze.c */
Query *
parse_analyze(RawStmt *parseTree,
              const char *sourceText,
              Oid *paramTypes,
              int numParams,
              QueryEnvironment *queryEnv)
{
    ParseState *pstate = make_parsestate(NULL);
    Query      *query;
    
    /* Set up parse state with query text and parameters */
    pstate->p_sourcetext = sourceText;
    if (numParams > 0)
        setup_parse_params(pstate, paramTypes, numParams);
    
    /* Transform the parse tree into a query tree */
    query = transformTopLevelStmt(pstate, parseTree);
    
    free_parsestate(pstate);
    
    return query;
}
```

### ParseState: The Analysis Context

The `ParseState` structure tracks context during analysis:

```c
/* From src/include/parser/parse_node.h (simplified) */
typedef struct ParseState
{
    struct ParseState *parentParseState;  /* Outer query (for subqueries) */
    const char *p_sourcetext;              /* Original query text */
    
    List       *p_rtable;                  /* Range table being built */
    List       *p_joinexprs;               /* JOIN clause info */
    List       *p_namespace;               /* Currently visible names */
    
    bool        p_lateral_active;          /* In LATERAL context? */
    List       *p_ctenamespace;            /* CTE names visible */
    
    CommonTableExpr *p_parent_cte;         /* If inside a CTE */
    
    List       *p_future_ctes;             /* Forward-referenced CTEs */
    List       *p_multiassign_exprs;       /* For multi-row VALUES */
    
    List       *p_target_nsitem;           /* Namespace for INSERT/UPDATE */
    
    /* And more... */
} ParseState;
```

The `p_namespace` field is crucial—it tracks which table aliases are currently visible for column lookup.

### Statement Transformation

`transformTopLevelStmt()` calls `transformStmt()`, which dispatches based on statement type:

```c
/* From src/backend/parser/analyze.c */
Query *
transformStmt(ParseState *pstate, Node *parseTree)
{
    Query *result;
    
    switch (nodeTag(parseTree))
    {
        case T_SelectStmt:
            result = transformSelectStmt(pstate, (SelectStmt *) parseTree);
            break;
            
        case T_InsertStmt:
            result = transformInsertStmt(pstate, (InsertStmt *) parseTree);
            break;
            
        case T_UpdateStmt:
            result = transformUpdateStmt(pstate, (UpdateStmt *) parseTree);
            break;
            
        case T_DeleteStmt:
            result = transformDeleteStmt(pstate, (DeleteStmt *) parseTree);
            break;
            
        case T_ExplainStmt:
        case T_CreateStmt:
        case T_IndexStmt:
        /* ... many more cases ... */
            result = makeNode(Query);
            result->commandType = CMD_UTILITY;
            result->utilityStmt = parseTree;
            break;
            
        default:
            elog(ERROR, "unrecognized node type: %d",
                 (int) nodeTag(parseTree));
    }
    
    return result;
}
```

### SELECT Statement Transformation

Let's trace through `transformSelectStmt()`:

```c
/* From src/backend/parser/analyze.c (simplified) */
static Query *
transformSelectStmt(ParseState *pstate, SelectStmt *stmt)
{
    Query      *qry = makeNode(Query);
    
    qry->commandType = CMD_SELECT;
    
    /* Handle WITH clause (CTEs) first */
    if (stmt->withClause)
    {
        qry->hasRecursive = stmt->withClause->recursive;
        qry->cteList = transformWithClause(pstate, stmt->withClause);
    }
    
    /* Transform FROM clause - populates range table */
    transformFromClause(pstate, stmt->fromClause);
    
    /* Transform target list (SELECT expressions) */
    qry->targetList = transformTargetList(pstate, stmt->targetList,
                                          EXPR_KIND_SELECT_TARGET);
    
    /* Transform WHERE clause */
    qry->jointree = makeFromExpr(pstate->p_joinlist, NULL);
    if (stmt->whereClause)
    {
        Node *qual = transformWhereClause(pstate, stmt->whereClause,
                                          EXPR_KIND_WHERE, "WHERE");
        qry->jointree->quals = qual;
    }
    
    /* Transform GROUP BY */
    qry->groupClause = transformGroupClause(pstate, stmt->groupClause, ...);
    
    /* Transform HAVING */
    if (stmt->havingClause)
        qry->havingQual = transformWhereClause(pstate, stmt->havingClause,
                                               EXPR_KIND_HAVING, "HAVING");
    
    /* Transform ORDER BY */
    qry->sortClause = transformSortClause(pstate, stmt->sortClause, ...);
    
    /* Transform LIMIT/OFFSET */
    qry->limitOffset = transformLimitClause(pstate, stmt->limitOffset, ...);
    qry->limitCount = transformLimitClause(pstate, stmt->limitCount, ...);
    
    /* Set up range table in the query */
    qry->rtable = pstate->p_rtable;
    
    return qry;
}
```

### FROM Clause Processing

`transformFromClause()` builds the range table:

```c
/* From src/backend/parser/parse_clause.c */
void
transformFromClause(ParseState *pstate, List *frmList)
{
    ListCell   *lc;
    
    foreach(lc, frmList)
    {
        Node       *n = lfirst(lc);
        
        /* Transform each FROM item */
        RangeTblEntry *rte;
        ParseNamespaceItem *nsitem;
        
        nsitem = transformFromClauseItem(pstate, n, &rte);
        
        /* Add to namespace (makes it visible for column lookup) */
        pstate->p_namespace = lappend(pstate->p_namespace, nsitem);
    }
}
```

For a simple table reference like `FROM users`:

```c
/* transformFromClauseItem eventually calls: */
RangeTblEntry *
addRangeTableEntry(ParseState *pstate,
                   RangeVar *relation,
                   Alias *alias,
                   bool inh,
                   bool inFromCl)
{
    RangeTblEntry *rte = makeNode(RangeTblEntry);
    Relation    rel;
    
    /* Open the relation (locks it) */
    rel = parserOpenTable(pstate, relation, AccessShareLock);
    
    /* Fill in the RTE */
    rte->rtekind = RTE_RELATION;
    rte->relid = RelationGetRelid(rel);    /* OID */
    rte->relkind = rel->rd_rel->relkind;
    rte->alias = alias;
    rte->eref = makeAlias(relation->relname, NIL);
    
    /* Add column names to eref */
    buildRelationAliases(rel->rd_att, alias, rte);
    
    /* Add to range table */
    pstate->p_rtable = lappend(pstate->p_rtable, rte);
    
    table_close(rel, NoLock);  /* Keep lock until end of transaction */
    
    return rte;
}
```

### Column Resolution

When the analyzer sees a `ColumnRef`, it must find which RTE and column it refers to:

```c
/* From src/backend/parser/parse_relation.c */
Node *
colNameToVar(ParseState *pstate, const char *colname, bool localonly,
             int location)
{
    ParseNamespaceItem *nsitem;
    int         sublevels_up;
    int         attnum;
    
    /* Search visible namespaces for this column name */
    nsitem = findNSItemForColName(pstate, colname, &sublevels_up, &attnum);
    
    if (nsitem == NULL)
    {
        ereport(ERROR,
                (errcode(ERRCODE_UNDEFINED_COLUMN),
                 errmsg("column \"%s\" does not exist", colname),
                 parser_errposition(pstate, location)));
    }
    
    /* Build a Var node */
    return (Node *) makeVar(nsitem->p_rtindex,  /* varno */
                            attnum,              /* varattno */
                            attrType,            /* vartype */
                            attrTypmod,          /* vartypmod */
                            attrCollation,       /* varcollid */
                            sublevels_up);       /* varlevelsup */
}
```

### Expression Transformation

`transformExpr()` handles all expression types:

```c
/* From src/backend/parser/parse_expr.c */
Node *
transformExpr(ParseState *pstate, Node *expr, ParseExprKind exprKind)
{
    if (expr == NULL)
        return NULL;
        
    switch (nodeTag(expr))
    {
        case T_ColumnRef:
            return transformColumnRef(pstate, (ColumnRef *) expr);
            
        case T_A_Const:
            return (Node *) make_const(pstate, (A_Const *) expr);
            
        case T_A_Expr:
            return transformAExpr(pstate, (A_Expr *) expr, exprKind);
            
        case T_FuncCall:
            return transformFuncCall(pstate, (FuncCall *) expr);
            
        case T_SubLink:
            return transformSubLink(pstate, (SubLink *) expr);
            
        case T_CaseExpr:
            return transformCaseExpr(pstate, (CaseExpr *) expr);
            
        /* ... many more cases ... */
        
        default:
            elog(ERROR, "unrecognized node type: %d", nodeTag(expr));
            return NULL;
    }
}
```

### Operator Resolution

When transforming `A_Expr` (like `id = 1`), the analyzer must find the actual operator:

```c
/* From src/backend/parser/parse_oper.c (simplified) */
Operator
oper(ParseState *pstate, List *opname, Oid ltypeId, Oid rtypeId,
     bool noError, int location)
{
    Oid         operOid;
    
    /* Look up operator in pg_operator */
    operOid = OpernameGetOprid(opname, ltypeId, rtypeId);
    
    if (!OidIsValid(operOid))
    {
        if (!noError)
            ereport(ERROR,
                    (errcode(ERRCODE_UNDEFINED_FUNCTION),
                     errmsg("operator does not exist: %s %s %s",
                            format_type_be(ltypeId),
                            NameListToString(opname),
                            format_type_be(rtypeId)),
                     parser_errposition(pstate, location)));
        return NULL;
    }
    
    return SearchSysCache1(OPEROID, ObjectIdGetDatum(operOid));
}
```

The lookup uses the system catalog `pg_operator`:

```sql
SELECT oid, oprname, oprleft, oprright, oprresult, oprcode
FROM pg_operator
WHERE oprname = '=' AND oprleft = 23 AND oprright = 23;
-- 23 is the OID of int4

  oid  | oprname | oprleft | oprright | oprresult | oprcode
-------+---------+---------+----------+-----------+---------
   96  | =       |      23 |       23 |        16 | int4eq
```

### Type Coercion

If types don't match exactly, PostgreSQL tries to coerce them:

```c
/* From src/backend/parser/parse_coerce.c */
Node *
coerce_to_target_type(ParseState *pstate, Node *expr,
                      Oid exprtype, Oid targettype, int32 targettypmod,
                      CoercionContext ccontext, CoercionForm cformat,
                      int location)
{
    /* Check if implicit cast exists */
    if (can_coerce_type(1, &exprtype, &targettype, ccontext))
    {
        return coerce_type(pstate, expr, exprtype, targettype,
                           targettypmod, ccontext, cformat, location);
    }
    
    /* No implicit coercion available */
    return NULL;
}
```

For example, comparing `int4` with `int8`:

```sql
SELECT * FROM t WHERE int4_col = 9223372036854775807;
-- 9223372036854775807 is too big for int4
-- PostgreSQL coerces int4_col to int8 or casts the literal
```

---

## Practical Examples and Tracing

### Example 1: Simple Column Reference

```sql
SELECT name FROM users WHERE id = 1;
```

Analysis process:

1. **FROM clause**: Look up `users` in `pg_class`, get OID 16384
   - Create `RangeTblEntry` with `relid = 16384`
   - Add to range table at index 1

2. **Target list**: Transform `ColumnRef { fields: ["name"] }`
   - Search namespace for "name"
   - Find it in users (attribute 2, type text = OID 25)
   - Create `Var { varno=1, varattno=2, vartype=25 }`

3. **WHERE clause**: Transform `A_Expr { name: "=", lexpr: ColumnRef{id}, rexpr: A_Const{1} }`
   - Resolve `id` → `Var { varno=1, varattno=1, vartype=23 }`
   - Transform `1` → `Const { consttype=23, constvalue=1 }`
   - Look up `int4 = int4` operator → `OpExpr { opno=96, ... }`

### Example 2: Ambiguous Column

```sql
SELECT id FROM users, orders;
-- ERROR:  column reference "id" is ambiguous
```

If both tables have an `id` column, the analyzer can't resolve which one you mean. You must qualify:

```sql
SELECT users.id FROM users, orders;  -- OK
```

### Example 3: Function vs. Aggregate

```sql
SELECT upper(name), count(*) FROM users GROUP BY name;
```

The analyzer must distinguish:
- `upper(name)` → `FuncExpr` (regular function)
- `count(*)` → `Aggref` (aggregate function)

It checks `pg_proc` and sees that `count` is declared as an aggregate:

```sql
SELECT prokind FROM pg_proc WHERE proname = 'count';
-- prokind = 'a' means aggregate
```

### Example 4: Using debug_print_rewritten

```sql
SET debug_print_rewritten = on;
SET client_min_messages = log;

SELECT id, name FROM users WHERE id < 10;
```

Check the log for output showing the Query node structure with resolved OIDs and Var references.

### Example 5: Tracing Name Resolution Errors

Common errors from the analyzer:

```sql
-- Table doesn't exist
SELECT * FROM nonexistent;
-- ERROR:  relation "nonexistent" does not exist

-- Column doesn't exist
SELECT missing_column FROM users;
-- ERROR:  column "missing_column" does not exist

-- Operator doesn't exist
SELECT * FROM users WHERE name = 123;
-- (if no text = integer operator defined)
-- ERROR:  operator does not exist: text = integer

-- Ambiguous column
SELECT id FROM users, orders;
-- ERROR:  column reference "id" is ambiguous
```

---

## Performance Implications

### Catalog Lookups

The analyzer makes many catalog lookups:
- `pg_class` for table metadata
- `pg_attribute` for column information
- `pg_type` for type details
- `pg_operator` for operators
- `pg_proc` for functions

These lookups are **cached** in PostgreSQL's syscache and relcache, so repeated queries are fast.

### Lock Acquisition

When the analyzer looks up a table, it acquires an **AccessShareLock**. This lock is held until the end of the transaction, preventing the table from being dropped while the query is using it.

```sql
BEGIN;
SELECT * FROM users;  -- Acquires AccessShareLock on users

-- In another session:
DROP TABLE users;  -- Waits for the lock
```

### Prepared Statement Efficiency

With prepared statements, analysis happens only on PREPARE:

```sql
PREPARE get_user (int) AS SELECT * FROM users WHERE id = $1;
-- Analysis happens here

EXECUTE get_user(1);  -- No re-analysis, just planning and execution
EXECUTE get_user(2);  -- Still no re-analysis
```

This saves catalog lookups on repeated execution.

---

## Advanced Topics and Edge Cases

### Subqueries and Correlation

Subqueries have their own parse state with the outer query's state as parent:

```sql
SELECT * FROM users u
WHERE u.id IN (SELECT user_id FROM orders WHERE total > 100);
```

The subquery can reference `u.id` because the outer namespace is visible via `parentParseState`.

### Lateral Subqueries

`LATERAL` makes preceding tables visible:

```sql
SELECT u.*, recent.* 
FROM users u,
     LATERAL (SELECT * FROM orders WHERE user_id = u.id LIMIT 3) recent;
```

Without `LATERAL`, the subquery couldn't reference `u.id`.

### Common Table Expressions (CTEs)

CTEs are analyzed and stored before the main query:

```sql
WITH recent_orders AS (
    SELECT * FROM orders WHERE date > '2024-01-01'
)
SELECT * FROM users u
JOIN recent_orders r ON u.id = r.user_id;
```

The CTE `recent_orders` appears in `qry->cteList` and is added to the namespace as `RTE_CTE`.

### Aggregate Detection

During analysis, PostgreSQL detects whether the query contains aggregates:

```c
/* Set hasAggs if any aggregates found */
if (pstate->p_hasAggs)
{
    qry->hasAggs = true;
    /* Validate GROUP BY requirements */
    parseCheckAggregates(pstate, qry);
}
```

This is why you get "must appear in GROUP BY clause" errors during analysis, not later.

### Type Resolution for Parameters

For prepared statements with parameters (`$1, $2`), the analyzer must know or infer their types:

```sql
PREPARE mystery AS SELECT $1 + 1;
-- ERROR:  could not determine data type of parameter $1

PREPARE typed (int) AS SELECT $1 + 1;
-- OK: $1 is declared as int
```

---

## Hands-On Exercises

### Exercise 1: Error Identification (Basic)

**Objective**: Identify which errors come from the analyzer.

For each query, predict the error message:

```sql
-- 1
SELECT naem FROM users;

-- 2
SELECT * FROM uesrs;

-- 3
SELECT id, name FROM users WHERE id = 'not_a_number';

-- 4
SELECT upper FROM users;  -- upper is a function name
```

### Exercise 2: Range Table Exploration (Intermediate)

**Objective**: Understand range table construction.

1. Write a query that produces a range table with 3 entries:
   ```sql
   -- Hint: Join three tables or use subqueries
   ```

2. Use `debug_print_rewritten` to view the range table.

### Exercise 3: Type Coercion (Intermediate)

**Objective**: Explore PostgreSQL's type coercion.

```sql
-- Which of these work without explicit casts?
SELECT 1 + 1.5;
SELECT 'hello' || 123;
SELECT date '2024-01-01' + 7;
SELECT true AND 1;

-- Check what coercions are available:
SELECT castsource::regtype, casttarget::regtype, castcontext
FROM pg_cast
WHERE castsource = 'integer'::regtype
ORDER BY castsource, casttarget;
```

### Exercise 4: Understanding Column Lookup (Advanced)

**Objective**: Trace column resolution in multi-table queries.

```sql
CREATE TABLE t1 (id int, a text);
CREATE TABLE t2 (id int, b text);

-- What happens with these?
SELECT id FROM t1, t2;              -- Error: ambiguous
SELECT t1.id FROM t1, t2;           -- OK
SELECT id FROM t1 JOIN t2 USING (id);  -- OK: USING creates merged column
SELECT id FROM t1 NATURAL JOIN t2;     -- OK: NATURAL JOIN
```

### Exercise 5: Subquery Scoping (Advanced)

**Objective**: Understand how subqueries access outer columns.

```sql
-- Which column references are valid?
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id      -- Can reference u
      AND o.total > (
          SELECT avg(total)
          FROM orders
          WHERE user_id = u.id  -- Can still reference u
      )
);
```

---

## Key Takeaways

1. **The analyzer connects syntax to semantics**: It transforms parse tree names into actual database object references with OIDs and type information.

2. **The range table is the foundation**: Every table, subquery, or function in FROM gets an entry, and all column references point into the range table.

3. **Var nodes replace ColumnRef**: After analysis, column references become `Var` nodes with `varno` (table) and `varattno` (column) indices.

4. **Type checking happens here**: Operator and function lookups verify that expressions are type-valid.

5. **Errors from the analyzer are semantic**: "does not exist", "ambiguous", "operator does not exist" all come from this stage.

6. **Locks are acquired during analysis**: Even a SELECT acquires AccessShareLock on referenced tables.

7. **Prepared statements cache the Query tree**: Re-execution skips analysis entirely.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/parser/analyze.c` | Main analysis entry points |
| `src/backend/parser/parse_clause.c` | FROM, WHERE, GROUP BY handling |
| `src/backend/parser/parse_expr.c` | Expression transformation |
| `src/backend/parser/parse_relation.c` | Name resolution |
| `src/backend/parser/parse_coerce.c` | Type coercion |
| `src/backend/parser/parse_oper.c` | Operator lookup |
| `src/backend/parser/parse_func.c` | Function lookup |
| `src/include/nodes/parsenodes.h` | Query node definition |

### Documentation

- [PostgreSQL Docs: The Rule System](https://www.postgresql.org/docs/current/rule-system.html) (covers query tree structure)
- [PostgreSQL Docs: System Catalogs](https://www.postgresql.org/docs/current/catalogs.html)

### Next Lessons

- **Lesson 4**: The Rewriter — View expansion and the rule system
- **Lesson 5**: Introduction to the Query Planner
- **Lesson 44**: System Catalogs — Deep dive into pg_class, pg_attribute, etc.

---

## References

### Source Code

- `src/backend/parser/analyze.c` — Statement analysis
- `src/backend/parser/parse_clause.c` — Clause transformation
- `src/backend/parser/parse_relation.c` — Table and column lookup
- `src/include/nodes/parsenodes.h` — Query structure definition
- `src/include/nodes/primnodes.h` — Var, Const, and expression nodes

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `parse_analyze()` | analyze.c | Entry point |
| `transformStmt()` | analyze.c | Dispatch by statement type |
| `transformFromClause()` | parse_clause.c | Build range table |
| `transformExpr()` | parse_expr.c | Expression transformation |
| `colNameToVar()` | parse_relation.c | Column name resolution |
| `addRangeTableEntry()` | parse_relation.c | Add RTE for table |

### Key Structures

| Structure | Location | Purpose |
|-----------|----------|---------|
| `Query` | parsenodes.h | Analyzed query |
| `RangeTblEntry` | parsenodes.h | Range table entry |
| `Var` | primnodes.h | Column reference |
| `Const` | primnodes.h | Constant value |
| `FuncExpr` | primnodes.h | Function call |
| `OpExpr` | primnodes.h | Operator expression |
| `Aggref` | primnodes.h | Aggregate reference |
