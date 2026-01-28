# Lesson 4: The Rewriter — Query Transformation and View Expansion

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 1 — Foundation: From SQL to Execution |
| **Lesson** | 4 of 66 |
| **Estimated Time** | 50-60 minutes |
| **Prerequisites** | Lesson 3: The Analyzer |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain the purpose of the rewriter in query processing
2. Describe how views are expanded into their base table queries
3. Understand the rule system and `pg_rewrite` catalog
4. Differentiate between INSTEAD and ALSO rules
5. Trace how row-level security policies are injected

### Key Terms

| Term | Definition |
|------|------------|
| **Rewriter** | The component that transforms query trees by applying rules |
| **Rule** | A stored transformation that modifies queries on a relation |
| **View** | A virtual table defined by a SELECT rule |
| **INSTEAD Rule** | A rule that replaces the original query entirely |
| **ALSO Rule** | A rule that adds additional queries alongside the original |
| **pg_rewrite** | The system catalog storing rewrite rules |
| **Row-Level Security (RLS)** | Policies that filter rows based on user identity |

---

## Introduction

After the analyzer produces a query tree, PostgreSQL doesn't immediately hand it to the planner. First, the **rewriter** transforms the query by applying rules stored in the system catalog `pg_rewrite`.

The most visible use of the rewriter is **view expansion**. When you query a view, you're really querying a virtual table. The rewriter transparently replaces the view reference with the view's underlying SELECT statement. This is why views have zero runtime overhead compared to writing the query directly—after rewriting, the planner sees the same thing.

But the rule system is more general than views. You can define rules that:

- Transform SELECT queries on a table
- Intercept INSERT/UPDATE/DELETE and redirect them
- Add additional actions when data is modified
- Implement row-level security by injecting filter conditions

The rewriter is a powerful but sometimes surprising feature. Understanding it helps you:

1. **Debug view behavior**: Why does my view query behave differently than the equivalent direct query?
2. **Understand performance**: Views don't inherently slow things down—the rewriter eliminates them.
3. **Implement advanced patterns**: Updatable views, audit logging, and access control can use rules.
4. **Avoid pitfalls**: Rules have subtle semantics that can cause unexpected behavior.

> **Historical Note**: PostgreSQL's rule system dates back to the original POSTGRES project at Berkeley. It was initially designed for a row-level implementation but was replaced with the current query-rewriting approach in 1995.

---

## Conceptual Foundation

### The Rewriter's Position in Query Processing

```
┌─────────────────────────────────────────────────────────────────┐
│                      Original SQL Query                          │
│              SELECT * FROM active_users;                         │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │    Parser     │
                              └───────┬───────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │   Analyzer    │
                              └───────┬───────┘
                                      │
                                      ▼ Query Tree (references view)
┌─────────────────────────────────────────────────────────────────┐
│  REWRITER                                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 1. Check pg_rewrite for rules on referenced relations       │ │
│  │ 2. Find: active_users has SELECT rule (it's a view)         │ │
│  │ 3. Substitute view's query for the view reference           │ │
│  │ 4. Apply any RLS policies                                   │ │
│  │ 5. Output: transformed query tree(s)                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼ Expanded Query Tree
┌─────────────────────────────────────────────────────────────────┐
│                      Equivalent Query                            │
│      SELECT * FROM users WHERE active = true;                    │
│                                                                  │
│      (The view has been "inlined")                              │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼
                              ┌───────────────┐
                              │   Planner     │
                              └───────────────┘
```

### What is a View, Really?

A view is just a SELECT rule with a special name:

```sql
CREATE VIEW active_users AS
SELECT * FROM users WHERE active = true;
```

This is equivalent to:

```sql
CREATE TABLE active_users (/* same columns as users */);
CREATE RULE "_RETURN" AS ON SELECT TO active_users
DO INSTEAD SELECT * FROM users WHERE active = true;
```

The `_RETURN` rule name is special—it marks this as a view. You can see it in `pg_rewrite`:

```sql
SELECT ev_class::regclass, rulename, ev_type, is_instead
FROM pg_rewrite
WHERE ev_class = 'active_users'::regclass;

 ev_class     | rulename | ev_type | is_instead
--------------+----------+---------+------------
 active_users | _RETURN  | 1       | t
```

(`ev_type = 1` means SELECT)

### How View Expansion Works

When you query a view:

```sql
SELECT name FROM active_users WHERE id < 100;
```

The rewriter:

1. Sees `active_users` in the FROM clause
2. Finds the `_RETURN` rule in `pg_rewrite`
3. Replaces the view reference with a subquery containing the view's definition
4. Merges WHERE clauses where possible

Result after rewriting:

```sql
SELECT name FROM (SELECT * FROM users WHERE active = true) AS active_users
WHERE id < 100;
```

The planner then flattens this (usually) into:

```sql
SELECT name FROM users WHERE active = true AND id < 100;
```

### Rule Types

PostgreSQL supports rules with different behaviors:

| Rule Type | Event | Behavior |
|-----------|-------|----------|
| SELECT | Query | INSTEAD only (views) |
| INSERT | Insert | INSTEAD or ALSO |
| UPDATE | Update | INSTEAD or ALSO |
| DELETE | Delete | INSTEAD or ALSO |

- **INSTEAD**: The rule's action replaces the original operation
- **ALSO**: The rule's action is performed in addition to the original

---

## Deep Dive: Implementation Details

### The pg_rewrite Catalog

Rules are stored in `pg_rewrite`:

```sql
\d pg_rewrite

       Column       |     Type     
--------------------+--------------
 oid                | oid          
 rulename           | name         
 ev_class           | oid          -- Relation the rule applies to
 ev_type            | char         -- '1'=SELECT, '2'=UPDATE, '3'=INSERT, '4'=DELETE
 ev_enabled         | char         -- 'O'=enabled, 'D'=disabled, etc.
 is_instead         | boolean      -- INSTEAD or ALSO
 ev_qual            | pg_node_tree -- Rule condition (WHERE clause)
 ev_action          | pg_node_tree -- Rule action (query trees)
```

The `ev_qual` and `ev_action` columns store serialized query trees.

### Rewriter Entry Point

The rewriter is called from `pg_analyze_and_rewrite()`:

```c
/* From src/backend/tcop/postgres.c */
List *
pg_analyze_and_rewrite(RawStmt *parsetree,
                       const char *query_string,
                       Oid *paramTypes,
                       int numParams,
                       QueryEnvironment *queryEnv)
{
    Query      *query;
    List       *querytree_list;
    
    /* Analyze */
    query = parse_analyze(parsetree, query_string, paramTypes, numParams,
                          queryEnv);
    
    /* Rewrite */
    querytree_list = pg_rewrite_query(query);
    
    return querytree_list;
}
```

Note that the rewriter can return **multiple** query trees (for ALSO rules).

### The QueryRewrite Function

```c
/* From src/backend/rewrite/rewriteHandler.c */
List *
QueryRewrite(Query *parsetree)
{
    List       *querylist;
    List       *results;
    ListCell   *lc;
    
    /* Apply non-SELECT rules first */
    querylist = RewriteQuery(parsetree, NIL);
    
    results = NIL;
    foreach(lc, querylist)
    {
        Query  *query = lfirst_node(Query, lc);
        
        /* Now expand views (SELECT rules) */
        query = fireRIRrules(query, NIL);
        
        results = lappend(results, query);
    }
    
    return results;
}
```

### View Expansion: fireRIRrules

"RIR" stands for "Relation, In, Range-table" - it processes SELECT rules:

```c
/* From src/backend/rewrite/rewriteHandler.c (simplified) */
static Query *
fireRIRrules(Query *parsetree, List *activeRIRs)
{
    int         rt_index;
    RangeTblEntry *rte;
    
    /* Process each range table entry */
    rt_index = 0;
    foreach(lc, parsetree->rtable)
    {
        rt_index++;
        rte = lfirst_node(RangeTblEntry, lc);
        
        /* Skip if not a regular relation */
        if (rte->rtekind != RTE_RELATION)
            continue;
            
        /* Check for SELECT rules (views) */
        if (rte->relkind == RELKIND_VIEW)
        {
            /* Get the view's rule */
            rule = GetViewRule(rte->relid);
            
            /* Apply the rule */
            rte = ApplyRetrieveRule(parsetree, rte, rt_index, rule, activeRIRs);
        }
    }
    
    /* Recursively process subqueries */
    query_tree_walker(parsetree, fireRIRonSubLink, ...);
    
    return parsetree;
}
```

### ApplyRetrieveRule: The Core Expansion

```c
/* From src/backend/rewrite/rewriteHandler.c (simplified) */
static RangeTblEntry *
ApplyRetrieveRule(Query *parsetree,
                  RangeTblEntry *rte,
                  int rt_index,
                  RewriteRule *rule,
                  List *activeRIRs)
{
    Query      *rule_action;
    
    /* Get a copy of the rule's query */
    rule_action = copyObject(linitial(rule->actions));
    
    /* Recursively expand any views in the rule's query */
    rule_action = fireRIRrules(rule_action, activeRIRs);
    
    /* Change the RTE to a subquery type */
    rte->rtekind = RTE_SUBQUERY;
    rte->subquery = rule_action;
    rte->relid = InvalidOid;
    
    return rte;
}
```

The key insight: the view's RTE is converted from `RTE_RELATION` to `RTE_SUBQUERY`, and the view's SELECT statement becomes the subquery.

### Row-Level Security

RLS policies are applied during rewriting:

```c
/* From src/backend/rewrite/rowsecurity.c */
void
get_row_security_policies(Query *root, RangeTblEntry *rte,
                          int rt_index, List **securityQuals,
                          List **withCheckOptions)
{
    /* Check if RLS is enabled on this relation */
    if (!RelationHasRowSecurity(rte->relid))
        return;
        
    /* Get applicable policies from pg_policy */
    policies = GetRowSecurityPolicies(rte->relid, cmd);
    
    /* Build security quals (WHERE conditions) */
    foreach(lc, policies)
    {
        RowSecurityPolicy *policy = lfirst(lc);
        
        /* Add policy's USING clause to security quals */
        *securityQuals = lappend(*securityQuals, policy->qual);
    }
}
```

These security quals are added to the query's WHERE clause, ensuring users only see authorized rows.

---

## Practical Examples and Tracing

### Example 1: Simple View Expansion

```sql
-- Create a view
CREATE VIEW recent_orders AS
SELECT * FROM orders WHERE order_date > CURRENT_DATE - INTERVAL '30 days';

-- Query the view
SELECT customer_id, total FROM recent_orders WHERE total > 100;
```

Before rewriting (query tree references `recent_orders`):
```
Query
├── rtable[1]: RTE_RELATION (recent_orders)
├── targetList: [customer_id, total]
└── quals: total > 100
```

After rewriting (view expanded):
```
Query
├── rtable[1]: RTE_SUBQUERY
│   └── subquery: Query
│       ├── rtable[1]: RTE_RELATION (orders)
│       ├── targetList: [*]
│       └── quals: order_date > CURRENT_DATE - '30 days'
├── targetList: [customer_id, total]
└── quals: total > 100
```

The planner will merge these into a single scan of `orders` with combined conditions.

### Example 2: Viewing the Rewritten Query

```sql
SET debug_print_rewritten = on;
SET client_min_messages = log;

SELECT * FROM recent_orders WHERE total > 100;
-- Check server log for rewritten query tree
```

### Example 3: INSERT Rule (Updatable View)

```sql
-- Make a view updatable via rules
CREATE VIEW user_emails AS
SELECT id, email FROM users;

CREATE RULE user_emails_insert AS ON INSERT TO user_emails
DO INSTEAD
INSERT INTO users (id, email, name, active)
VALUES (NEW.id, NEW.email, 'Unknown', true);

-- Now you can insert into the view
INSERT INTO user_emails (id, email) VALUES (100, 'test@example.com');
-- This becomes: INSERT INTO users (id, email, name, active) VALUES (100, 'test@example.com', 'Unknown', true);
```

### Example 4: ALSO Rule for Audit Logging

```sql
CREATE TABLE users (id serial, name text);
CREATE TABLE users_audit (id int, old_name text, new_name text, changed_at timestamp);

CREATE RULE users_audit_update AS ON UPDATE TO users
DO ALSO
INSERT INTO users_audit (id, old_name, new_name, changed_at)
VALUES (OLD.id, OLD.name, NEW.name, now());

-- Now updates also create audit records
UPDATE users SET name = 'Bob' WHERE id = 1;
-- Performs both the UPDATE and the INSERT into audit table
```

### Example 5: Row-Level Security

```sql
-- Enable RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Create policy: users can only see their own documents
CREATE POLICY documents_owner_policy ON documents
USING (owner_id = current_user_id());

-- Query as user with current_user_id() = 42
SELECT * FROM documents WHERE status = 'active';

-- After rewriting, becomes:
SELECT * FROM documents WHERE status = 'active' AND owner_id = 42;
```

### Example 6: Examining pg_rewrite

```sql
-- See all views and their rules
SELECT 
    c.relname AS view_name,
    r.rulename,
    pg_get_ruledef(r.oid) AS rule_definition
FROM pg_rewrite r
JOIN pg_class c ON r.ev_class = c.oid
WHERE c.relkind = 'v'
ORDER BY c.relname;
```

---

## Performance Implications

### Views Have Zero Overhead (Usually)

Because the rewriter inlines views, there's no "view lookup" at runtime:

```sql
-- These are equivalent after rewriting and planning:
SELECT * FROM my_view WHERE x = 1;
SELECT * FROM (view definition) WHERE x = 1;
-- After planner optimization:
SELECT * FROM base_table WHERE x = 1 AND (view conditions);
```

The planner optimizes the combined query, potentially pushing filters down and using indexes.

### When Views Can Hurt

Views become problematic when they prevent optimization:

1. **Aggregations in views**: The planner can't push filters inside aggregation
   ```sql
   CREATE VIEW order_totals AS
   SELECT customer_id, SUM(total) as total FROM orders GROUP BY customer_id;
   
   -- This filter happens AFTER aggregation, can't use index
   SELECT * FROM order_totals WHERE customer_id = 1;
   ```

2. **DISTINCT or LIMIT in views**: Similar barrier to optimization

3. **Security barrier views**: Views created with `WITH (security_barrier)` prevent some optimizations to maintain security

### Rules vs. Triggers

For non-SELECT operations, triggers are often preferred over rules:

| Aspect | Rules | Triggers |
|--------|-------|----------|
| Execution | Query transformation | Row-level |
| Timing | Before planning | During execution |
| OLD/NEW | Reference entire result set | Reference single row |
| Performance | Can be faster for bulk ops | Better for row-at-a-time |
| Complexity | Easy to get wrong | More intuitive |
| Debugging | Hard to trace | Easier to debug |

The PostgreSQL documentation recommends triggers for most use cases.

---

## Advanced Topics and Edge Cases

### Circular View Dependencies

PostgreSQL prevents infinite recursion:

```sql
CREATE VIEW a AS SELECT * FROM b;
CREATE VIEW b AS SELECT * FROM a;
-- ERROR: infinite recursion detected in rules for relation "a"
```

The rewriter tracks active rules to detect cycles.

### Conditional Rules

Rules can have conditions:

```sql
CREATE RULE log_high_value AS ON INSERT TO orders
WHERE NEW.total > 1000
DO ALSO
INSERT INTO high_value_log VALUES (NEW.id, NEW.total, now());
```

Only inserts with `total > 1000` trigger the logging.

### Multiple Rules

When multiple rules apply, they're executed in name order:

```sql
CREATE RULE rule_a AS ON INSERT TO t DO ALSO ...;
CREATE RULE rule_b AS ON INSERT TO t DO ALSO ...;
-- rule_a executes before rule_b
```

### INSTEAD OF Triggers (Better Than Rules)

For updatable views, `INSTEAD OF` triggers are clearer:

```sql
CREATE TRIGGER user_emails_insert
INSTEAD OF INSERT ON user_emails
FOR EACH ROW
EXECUTE FUNCTION handle_user_emails_insert();
```

### Views and Search Path

View definitions capture names at creation time:

```sql
SET search_path = myschema;
CREATE VIEW myview AS SELECT * FROM mytable;

-- Later, even with different search_path:
SELECT * FROM myview;  -- Still references myschema.mytable
```

---

## Hands-On Exercises

### Exercise 1: View Expansion Tracing (Basic)

**Objective**: Observe view expansion in action.

1. Create a simple view:
   ```sql
   CREATE VIEW pg_tables_public AS
   SELECT tablename FROM pg_tables WHERE schemaname = 'public';
   ```

2. Enable debug output and query the view:
   ```sql
   SET debug_print_rewritten = on;
   SET client_min_messages = log;
   SELECT * FROM pg_tables_public;
   ```

3. Check the log for the expanded query.

### Exercise 2: Exploring pg_rewrite (Intermediate)

**Objective**: Understand the rule catalog.

```sql
-- Create a view
CREATE VIEW test_view AS SELECT 1 AS num;

-- Examine its rule
SELECT rulename, ev_type, is_instead,
       pg_get_ruledef(oid) AS definition
FROM pg_rewrite
WHERE ev_class = 'test_view'::regclass;

-- Clean up
DROP VIEW test_view;
```

### Exercise 3: ALSO Rule Implementation (Intermediate)

**Objective**: Create an audit trail using rules.

```sql
CREATE TABLE products (id serial PRIMARY KEY, name text, price numeric);
CREATE TABLE products_log (id int, action text, logged_at timestamp);

-- Create ALSO rule for inserts
CREATE RULE log_product_insert AS ON INSERT TO products
DO ALSO INSERT INTO products_log VALUES (NEW.id, 'INSERT', now());

-- Test it
INSERT INTO products (name, price) VALUES ('Widget', 9.99);
SELECT * FROM products_log;
```

### Exercise 4: INSTEAD Rule (Advanced)

**Objective**: Make a view insertable via rules.

```sql
CREATE VIEW products_summary AS
SELECT id, name FROM products;

-- Make it insertable
CREATE RULE products_summary_insert AS ON INSERT TO products_summary
DO INSTEAD INSERT INTO products (id, name, price) VALUES (NEW.id, NEW.name, 0);

INSERT INTO products_summary (name) VALUES ('Gadget');
SELECT * FROM products;
```

### Exercise 5: Row-Level Security (Advanced)

**Objective**: Implement multi-tenant data isolation.

```sql
CREATE TABLE tenant_data (
    id serial,
    tenant_id int,
    data text
);

-- Enable RLS
ALTER TABLE tenant_data ENABLE ROW LEVEL SECURITY;

-- Create policy
CREATE POLICY tenant_isolation ON tenant_data
USING (tenant_id = current_setting('app.tenant_id')::int);

-- Test
SET app.tenant_id = '1';
INSERT INTO tenant_data (tenant_id, data) VALUES (1, 'Tenant 1 data');
INSERT INTO tenant_data (tenant_id, data) VALUES (2, 'Tenant 2 data');

SELECT * FROM tenant_data;  -- Only sees tenant 1's data
```

---

## Key Takeaways

1. **The rewriter transforms queries before planning**: It applies rules stored in `pg_rewrite` to modify the query tree.

2. **Views are just SELECT rules**: The `_RETURN` rule replaces the view reference with its definition. Views have no inherent overhead.

3. **View expansion is transparent**: After rewriting, the planner sees a query against base tables, enabling full optimization.

4. **Rules can intercept DML**: INSERT, UPDATE, DELETE rules can add actions (ALSO) or replace operations (INSTEAD).

5. **RLS policies are injected during rewriting**: Security quals are added to ensure users only see authorized data.

6. **Triggers are usually better than rules**: For non-view use cases, triggers are more intuitive and easier to debug.

7. **The rewriter can output multiple queries**: ALSO rules create additional queries that execute alongside the original.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/rewrite/rewriteHandler.c` | Main rewriter entry points |
| `src/backend/rewrite/rewriteDefine.c` | Rule definition handling |
| `src/backend/rewrite/rowsecurity.c` | Row-level security implementation |
| `src/include/rewrite/rewriteHandler.h` | Rewriter interface |
| `src/backend/commands/view.c` | View creation |

### Documentation

- [PostgreSQL Docs: The Rule System](https://www.postgresql.org/docs/current/rules.html)
- [PostgreSQL Docs: CREATE RULE](https://www.postgresql.org/docs/current/sql-createrule.html)
- [PostgreSQL Docs: Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

### Next Lessons

- **Lesson 5**: Introduction to the Query Planner
- **Lesson 6**: The Executor
- **Lesson 44**: System Catalogs — including pg_rewrite

---

## References

### Source Code

- `src/backend/rewrite/rewriteHandler.c` — Main rewriting logic
- `src/backend/rewrite/rewriteDefine.c` — Rule management
- `src/backend/rewrite/rowsecurity.c` — RLS implementation
- `src/include/catalog/pg_rewrite.h` — pg_rewrite catalog definition

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `QueryRewrite()` | rewriteHandler.c | Main entry point |
| `fireRIRrules()` | rewriteHandler.c | Process SELECT rules |
| `ApplyRetrieveRule()` | rewriteHandler.c | Expand a view |
| `get_row_security_policies()` | rowsecurity.c | Apply RLS |
| `DefineRule()` | rewriteDefine.c | Create a rule |

### System Catalogs

| Catalog | Purpose |
|---------|---------|
| `pg_rewrite` | Stores rewrite rules |
| `pg_policy` | Stores row security policies |
| `pg_class` | Relation metadata (relkind = 'v' for views) |
