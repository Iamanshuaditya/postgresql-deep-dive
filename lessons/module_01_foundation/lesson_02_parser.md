# Lesson 2: The Parser — Transforming SQL Into Internal Structures

---

## Front Matter

| Property | Value |
|----------|-------|
| **Module** | 1 — Foundation: From SQL to Execution |
| **Lesson** | 2 of 66 |
| **Estimated Time** | 60-75 minutes |
| **Prerequisites** | Lesson 1: How PostgreSQL Processes a Query |

### Learning Objectives

By the end of this lesson, you will be able to:

1. Explain the role of the lexer and parser in query processing
2. Understand how `scan.l` and `gram.y` work together
3. Describe the structure of a parse tree and key node types
4. Trace how a simple SELECT statement becomes a `SelectStmt` node
5. Distinguish between reserved and unreserved keywords

### Key Terms

| Term | Definition |
|------|------------|
| **Lexer** | The component that breaks SQL text into tokens (lexical analysis) |
| **Token** | A meaningful unit identified by the lexer: keyword, identifier, literal, operator |
| **Parser** | The component that validates syntax and builds the parse tree |
| **Parse Tree** | A tree of Node structures representing the syntactic structure of SQL |
| **Flex** | The tool that generates the lexer from `scan.l` |
| **Bison** | The tool that generates the parser from `gram.y` |
| **SelectStmt** | The Node type representing a SELECT statement in the parse tree |

---

## Introduction

In the previous lesson, we saw that the parser is the first stage of query processing. It takes raw SQL text and produces a structured representation that subsequent stages can work with. But how does PostgreSQL understand that `SELECT * FROM users WHERE id = 1` means "retrieve all columns from the users table for rows where id equals 1"?

The answer lies in two classic compiler construction tools: **Flex** (for lexical analysis) and **Bison** (for parsing). PostgreSQL's parser is defined using these tools, with the lexer specification in `scan.l` and the grammar in `gram.y`. These files are processed at compile time to generate C code that becomes part of the PostgreSQL executable.

Understanding the parser is valuable for several reasons:

1. **Error Messages**: When you see "syntax error at or near...", the parser generated that message. Understanding parsing helps you interpret these errors.

2. **SQL Compatibility**: PostgreSQL's SQL dialect is defined by `gram.y`. If you want to know whether a construct is supported, the grammar is the authoritative source.

3. **Extensions**: If you're building extensions that add new syntax (like PostGIS's geometry types), you need to understand how the parser works.

4. **Query Structure**: The parse tree structure influences all subsequent processing. Understanding it helps you reason about query optimization.

> **A Note on Complexity**: The PostgreSQL grammar file `gram.y` is over 18,000 lines of code. We won't cover every rule, but we'll understand the structure and trace through key examples.

---

## Conceptual Foundation

### The Two-Stage Parsing Process

Parsing in PostgreSQL (and most compilers) happens in two distinct phases:

```
┌───────────────────────────────────────────────────────────────────┐
│                      SQL Query String                             │
│     "SELECT name, email FROM users WHERE active = true"           │
└─────────────────────────────────┬─────────────────────────────────┘
                                  │
                                  ▼
┌───────────────────────────────────────────────────────────────────┐
│  LEXER (Lexical Analysis)                                         │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ scan.l → Flex → scan.c                                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Input:  "SELECT name, email FROM users WHERE active = true"      │
│                                                                   │
│  Output: Stream of tokens:                                        │
│    [SELECT] [IDENT:name] [,] [IDENT:email] [FROM]                │
│    [IDENT:users] [WHERE] [IDENT:active] [=] [TRUE_P]             │
│                                                                   │
└─────────────────────────────────┬─────────────────────────────────┘
                                  │
                                  ▼
┌───────────────────────────────────────────────────────────────────┐
│  PARSER (Syntactic Analysis)                                      │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ gram.y → Bison → gram.c                                     │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Input:  Token stream from lexer                                  │
│                                                                   │
│  Output: Parse Tree (SelectStmt node with child nodes)            │
│                                                                   │
│              SelectStmt                                           │
│             /    |    \                                           │
│      targetList fromClause whereClause                            │
│         /  \         |           |                                │
│     name  email   users      BoolExpr                             │
│                              (active = true)                      │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Why Two Stages?

Separating lexical analysis from parsing is a classic compiler design pattern with several benefits:

1. **Simplicity**: The lexer handles character-level details (whitespace, string escaping, comments), keeping the grammar cleaner.

2. **Efficiency**: Lexers are typically implemented as finite automata, which are very fast. Parsers use more complex algorithms.

3. **Error Handling**: The lexer can report character-level errors ("invalid character") while the parser reports structural errors ("syntax error").

4. **Maintainability**: Changes to tokenization (e.g., adding a new operator) don't require grammar changes.

### The Lexer: `scan.l`

The lexer is defined in `src/backend/parser/scan.l`. This file uses Flex syntax, which consists of patterns and actions:

```
┌─────────────────────────────────────────────────────────────────┐
│                         scan.l Structure                         │
├─────────────────────────────────────────────────────────────────┤
│  %{                                                              │
│    /* C code: includes, variables */                             │
│  %}                                                              │
│                                                                  │
│  /* Definitions */                                               │
│  space       [ \t\n\r\f]                                         │
│  digit       [0-9]                                               │
│  ident_start [A-Za-z\200-\377_]                                  │
│  ident_cont  [A-Za-z\200-\377_0-9$]                             │
│                                                                  │
│  %%                                                              │
│                                                                  │
│  /* Rules: pattern → action */                                   │
│                                                                  │
│  {space}+    { /* ignore whitespace */ }                         │
│                                                                  │
│  SELECT      { return SELECT; }                                  │
│  FROM        { return FROM; }                                    │
│  WHERE       { return WHERE; }                                   │
│                                                                  │
│  {ident_start}{ident_cont}*  {                                   │
│      /* Check if keyword, else return IDENT */                   │
│      return ScanKeywordLookup(yytext);                           │
│  }                                                               │
│                                                                  │
│  {digit}+    {                                                   │
│      yylval.ival = atoi(yytext);                                 │
│      return ICONST;                                              │
│  }                                                               │
│                                                                  │
│  %%                                                              │
│                                                                  │
│  /* C code: helper functions */                                  │
└─────────────────────────────────────────────────────────────────┘
```

The lexer recognizes several categories of tokens:

| Category | Examples | Token Type |
|----------|----------|------------|
| Keywords | SELECT, FROM, WHERE, JOIN | Specific token per keyword |
| Identifiers | users, column_name, "My Table" | IDENT |
| Integer literals | 42, 0, -17 | ICONST |
| Float literals | 3.14, 1e-10 | FCONST |
| String literals | 'hello', E'escape\n' | SCONST |
| Operators | =, <>, >=, + | Op or specific token |
| Punctuation | (, ), [, ], ; | Character itself |

### Keywords: Reserved vs. Unreserved

PostgreSQL distinguishes between different types of keywords:

```sql
-- Reserved keywords CANNOT be used as identifiers without quoting
SELECT select FROM table;  -- ERROR: "select" and "table" are reserved

-- You must quote them
SELECT "select" FROM "table";  -- OK

-- Unreserved keywords CAN be used as identifiers
SELECT name FROM users;  -- "name" is unreserved, works fine
CREATE TABLE name (id int);  -- Also works
```

The keyword classification is defined in `src/include/parser/kwlist.h`:

```c
/* From src/include/parser/kwlist.h (simplified) */

/* Reserved keywords */
PG_KEYWORD("all", ALL, RESERVED_KEYWORD, BARE_LABEL)
PG_KEYWORD("and", AND, RESERVED_KEYWORD, BARE_LABEL)
PG_KEYWORD("select", SELECT, RESERVED_KEYWORD, BARE_LABEL)
PG_KEYWORD("table", TABLE, RESERVED_KEYWORD, BARE_LABEL)
PG_KEYWORD("where", WHERE, RESERVED_KEYWORD, BARE_LABEL)

/* Unreserved keywords */
PG_KEYWORD("name", NAME_P, UNRESERVED_KEYWORD, BARE_LABEL)
PG_KEYWORD("type", TYPE_P, UNRESERVED_KEYWORD, BARE_LABEL)
PG_KEYWORD("value", VALUE_P, UNRESERVED_KEYWORD, BARE_LABEL)

/* Type/function name keywords */
PG_KEYWORD("int", INT_P, COL_NAME_KEYWORD, BARE_LABEL)
PG_KEYWORD("varchar", VARCHAR, COL_NAME_KEYWORD, BARE_LABEL)
```

### The Parser: `gram.y`

The parser is defined in `src/backend/parser/gram.y`. This file uses Bison syntax, which defines a context-free grammar:

```
┌─────────────────────────────────────────────────────────────────┐
│                         gram.y Structure                         │
├─────────────────────────────────────────────────────────────────┤
│  %{                                                              │
│    /* C code: includes, helper function declarations */          │
│  %}                                                              │
│                                                                  │
│  /* Token declarations */                                        │
│  %token <str>   IDENT SCONST                                     │
│  %token <ival>  ICONST                                           │
│  %token SELECT FROM WHERE AND OR NOT                             │
│                                                                  │
│  /* Type declarations for non-terminals */                       │
│  %type <node>   stmt SelectStmt simple_select                    │
│  %type <list>   target_list from_clause                          │
│  %type <node>   where_clause a_expr                              │
│                                                                  │
│  /* Precedence rules (for expression parsing) */                 │
│  %left  OR                                                       │
│  %left  AND                                                      │
│  %right NOT                                                      │
│  %nonassoc '=' '<' '>' ...                                       │
│                                                                  │
│  %%                                                              │
│                                                                  │
│  /* Grammar rules */                                             │
│                                                                  │
│  stmtblock: stmtmulti                                            │
│           ;                                                      │
│                                                                  │
│  stmtmulti: stmtmulti ';' stmt                                   │
│           | stmt                                                 │
│           ;                                                      │
│                                                                  │
│  stmt: SelectStmt                                                │
│      | InsertStmt                                                │
│      | UpdateStmt                                                │
│      | ...                                                       │
│      ;                                                           │
│                                                                  │
│  SelectStmt:                                                     │
│      select_no_parens                                            │
│      | select_with_parens                                        │
│      ;                                                           │
│                                                                  │
│  simple_select:                                                  │
│      SELECT opt_all_clause target_list                           │
│      from_clause where_clause ...                                │
│      {                                                           │
│          SelectStmt *n = makeNode(SelectStmt);                   │
│          n->targetList = $3;                                     │
│          n->fromClause = $4;                                     │
│          n->whereClause = $5;                                    │
│          $$ = (Node *) n;                                        │
│      }                                                           │
│      ;                                                           │
│                                                                  │
│  %%                                                              │
│                                                                  │
│  /* C code: helper functions */                                  │
└─────────────────────────────────────────────────────────────────┘
```

Each grammar rule has the form:

```
non_terminal: components { action }
            | alternative_components { action }
            ;
```

The action (in curly braces) is C code that executes when the rule is matched. It typically creates a node and assigns it to `$$`, which becomes the value of the non-terminal.

---

## Deep Dive: Implementation Details

### The Node System

All parse tree nodes inherit from the base `Node` type:

```c
/* From src/include/nodes/nodes.h */
typedef struct Node
{
    NodeTag     type;           /* Discriminator tag */
} Node;
```

The `NodeTag` is an enum identifying the node type:

```c
/* From src/include/nodes/nodetags.h (partial) */
typedef enum NodeTag
{
    T_Invalid = 0,
    
    /* Parse tree statement nodes */
    T_RawStmt = 100,
    T_SelectStmt,
    T_InsertStmt,
    T_UpdateStmt,
    T_DeleteStmt,
    T_CreateStmt,
    T_AlterTableStmt,
    
    /* Expression nodes */
    T_A_Expr = 200,
    T_ColumnRef,
    T_A_Const,
    T_FuncCall,
    T_SubLink,
    
    /* ... hundreds more ... */
} NodeTag;
```

### The SelectStmt Structure

Let's examine the structure representing a SELECT statement:

```c
/* From src/include/nodes/parsenodes.h */
typedef struct SelectStmt
{
    NodeTag     type;           /* T_SelectStmt */
    
    /*
     * These fields are used only in "leaf" SelectStmts.
     */
    List       *distinctClause; /* NULL, DISTINCT, DISTINCT ON list */
    IntoClause *intoClause;     /* target for SELECT INTO */
    List       *targetList;     /* the SELECT list (ResTarget nodes) */
    List       *fromClause;     /* the FROM clause */
    Node       *whereClause;    /* WHERE qualification */
    List       *groupClause;    /* GROUP BY clauses */
    bool        groupDistinct;  /* is this GROUP BY DISTINCT? */
    Node       *havingClause;   /* HAVING conditional-expression */
    List       *windowClause;   /* WINDOW window_name AS (...) */
    
    /*
     * In a "leaf" node, these represent VALUES list.
     */
    List       *valuesLists;    /* Lists of expressions */
    
    /*
     * Sort/limit info
     */
    List       *sortClause;     /* ORDER BY clause */
    Node       *limitOffset;    /* OFFSET expression */
    Node       *limitCount;     /* LIMIT expression (or NULL) */
    LimitOption limitOption;    /* LIMIT type (WITH TIES, etc) */
    
    /*
     * Locking
     */
    List       *lockingClause;  /* FOR UPDATE/SHARE clauses */
    WithClause *withClause;     /* WITH clause */
    
    /*
     * These fields are used in UNION/INTERSECT/EXCEPT queries.
     */
    SetOperation op;            /* type of set operation */
    bool        all;            /* ALL specified? */
    struct SelectStmt *larg;    /* left child */
    struct SelectStmt *rarg;    /* right child */
} SelectStmt;
```

### Target List: What to SELECT

The `targetList` contains `ResTarget` nodes:

```c
/* From src/include/nodes/parsenodes.h */
typedef struct ResTarget
{
    NodeTag     type;
    char       *name;           /* column alias (AS name) or NULL */
    List       *indirection;    /* subscripts, field names, and '*' */
    Node       *val;            /* the expression itself */
    int         location;       /* token position, or -1 if unknown */
} ResTarget;
```

For `SELECT name, email FROM users`, the `targetList` contains two `ResTarget` nodes:
- One with `val` = `ColumnRef` for "name"
- One with `val` = `ColumnRef` for "email"

### Column References

When you reference a column, it becomes a `ColumnRef`:

```c
/* From src/include/nodes/parsenodes.h */
typedef struct ColumnRef
{
    NodeTag     type;
    List       *fields;         /* list of String or A_Star nodes */
    int         location;       /* token position */
} ColumnRef;
```

For `users.name`, `fields` contains: `["users", "name"]`
For `*`, `fields` contains: `[A_Star]`

### Expressions: A_Expr

Operators and expressions are represented by `A_Expr`:

```c
/* From src/include/nodes/parsenodes.h */
typedef enum A_Expr_Kind
{
    AEXPR_OP,                   /* normal operator */
    AEXPR_OP_ANY,               /* scalar op ANY (array) */
    AEXPR_OP_ALL,               /* scalar op ALL (array) */
    AEXPR_DISTINCT,             /* IS DISTINCT FROM */
    AEXPR_NOT_DISTINCT,         /* IS NOT DISTINCT FROM */
    AEXPR_NULLIF,               /* NULLIF(a, b) */
    AEXPR_IN,                   /* [NOT] IN */
    AEXPR_LIKE,                 /* [NOT] LIKE */
    AEXPR_ILIKE,                /* [NOT] ILIKE */
    AEXPR_SIMILAR,              /* [NOT] SIMILAR */
    AEXPR_BETWEEN,              /* BETWEEN/NOT BETWEEN */
    AEXPR_NOT_BETWEEN,
    AEXPR_BETWEEN_SYM,          /* symmetric BETWEEN */
    AEXPR_NOT_BETWEEN_SYM
} A_Expr_Kind;

typedef struct A_Expr
{
    NodeTag     type;
    A_Expr_Kind kind;           /* which kind of expression */
    List       *name;           /* operator name (list of String) */
    Node       *lexpr;          /* left argument, or NULL if none */
    Node       *rexpr;          /* right argument, or NULL if none */
    int         location;       /* token position */
} A_Expr;
```

For `id = 1`, we get:
```
A_Expr {
    kind: AEXPR_OP
    name: ["="]
    lexpr: ColumnRef { fields: ["id"] }
    rexpr: A_Const { val: Integer 1 }
}
```

### Constants: A_Const

Literal values are represented by `A_Const`:

```c
/* From src/include/nodes/parsenodes.h */
typedef struct A_Const
{
    NodeTag     type;
    union ValUnion val;         /* the value (with its type) */
    bool        isnull;         /* SQL NULL? */
    int         location;       /* token position */
} A_Const;

typedef union ValUnion
{
    Integer     ival;
    Float       fval;           /* stored as string */
    Boolean     boolval;
    String      sval;
    BitString   bsval;
} ValUnion;
```

### Memory Allocation: makeNode

Parse tree nodes are allocated using `makeNode`:

```c
/* From src/include/nodes/nodes.h */
#define makeNode(_type_) ((_type_ *) newNode(sizeof(_type_), T_##_type_))

/* From src/backend/nodes/nodes.c */
Node *
newNode(size_t size, NodeTag tag)
{
    Node *result;
    
    result = (Node *) palloc0fast(size);
    result->type = tag;
    
    return result;
}
```

All nodes are allocated in a **memory context**, which enables efficient cleanup—when a query finishes, the entire context can be freed at once.

### Parser Entry Points

The main entry point is `raw_parser()`:

```c
/* From src/backend/parser/parser.c */
List *
raw_parser(const char *str, RawParseMode mode)
{
    core_yyscan_t yyscanner;
    base_yy_extra_type yyextra;
    int         yyresult;
    
    /* Initialize the flex scanner */
    yyscanner = scanner_init(str, &yyextra.core_yy_extra, ...);
    
    /* Initialize the bison parser */
    parser_init(&yyextra);
    
    /* Call the parser */
    yyresult = base_yyparse(yyscanner);
    
    /* Clean up */
    scanner_finish(yyscanner);
    
    if (yyresult)   /* error */
        return NIL;
    
    return yyextra.parsetree;
}
```

This returns a `List` of `RawStmt` nodes (since a query string can contain multiple statements separated by semicolons).

---

## Practical Examples and Tracing

### Example 1: Simple SELECT

Let's trace how `SELECT name FROM users WHERE id = 1;` is parsed:

```
┌─────────────────────────────────────────────────────────────────┐
│  SQL: SELECT name FROM users WHERE id = 1;                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  LEXER OUTPUT (Token Stream):                                    │
│                                                                  │
│  SELECT    → Token: SELECT                                       │
│  name      → Token: IDENT (yylval.str = "name")                 │
│  FROM      → Token: FROM                                         │
│  users     → Token: IDENT (yylval.str = "users")                │
│  WHERE     → Token: WHERE                                        │
│  id        → Token: IDENT (yylval.str = "id")                   │
│  =         → Token: '='                                          │
│  1         → Token: ICONST (yylval.ival = 1)                    │
│  ;         → Token: ';'                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  PARSER OUTPUT (Parse Tree):                                     │
│                                                                  │
│  RawStmt                                                         │
│  └── stmt: SelectStmt                                            │
│      ├── targetList: List                                        │
│      │   └── ResTarget                                           │
│      │       └── val: ColumnRef                                  │
│      │           └── fields: ["name"]                            │
│      ├── fromClause: List                                        │
│      │   └── RangeVar                                            │
│      │       ├── relname: "users"                                │
│      │       └── inh: true                                       │
│      └── whereClause: A_Expr                                     │
│          ├── kind: AEXPR_OP                                      │
│          ├── name: ["="]                                         │
│          ├── lexpr: ColumnRef                                    │
│          │   └── fields: ["id"]                                  │
│          └── rexpr: A_Const                                      │
│              └── val: Integer(1)                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Example 2: JOIN Query

```sql
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.total > 100;
```

Parse tree (simplified):

```
SelectStmt
├── targetList:
│   ├── ResTarget { val: ColumnRef { fields: ["u", "name"] } }
│   └── ResTarget { val: ColumnRef { fields: ["o", "total"] } }
├── fromClause:
│   └── JoinExpr
│       ├── jointype: JOIN_INNER
│       ├── larg: RangeVar { relname: "users", alias: "u" }
│       ├── rarg: RangeVar { relname: "orders", alias: "o" }
│       └── quals: A_Expr  /* u.id = o.user_id */
│           ├── lexpr: ColumnRef { fields: ["u", "id"] }
│           └── rexpr: ColumnRef { fields: ["o", "user_id"] }
└── whereClause: A_Expr  /* o.total > 100 */
    ├── name: [">"]
    ├── lexpr: ColumnRef { fields: ["o", "total"] }
    └── rexpr: A_Const { val: 100 }
```

### Example 3: Subquery

```sql
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders);
```

The subquery becomes a `SubLink` node:

```
SelectStmt
├── targetList:
│   └── ResTarget { val: ColumnRef { fields: [A_Star] } }
├── fromClause:
│   └── RangeVar { relname: "users" }
└── whereClause: SubLink
    ├── subLinkType: ANY_SUBLINK
    ├── testexpr: ColumnRef { fields: ["id"] }
    └── subselect: SelectStmt  /* the subquery */
        ├── targetList:
        │   └── ResTarget { val: ColumnRef { fields: ["user_id"] } }
        └── fromClause:
            └── RangeVar { relname: "orders" }
```

### Viewing Parse Trees with debug_print_parse

Enable parse tree output:

```sql
SET debug_print_parse = on;
SET client_min_messages = log;

SELECT name FROM users WHERE id = 1;
```

Check the PostgreSQL log for output like:

```
LOG:  parse tree:
DETAIL:     {QUERY 
   :commandType 1 
   :querySource 0 
   :canSetTag true 
   ...
```

### Using pg_parse_query Programmatically

If you're writing C extensions, you can use the parser directly:

```c
#include "postgres.h"
#include "parser/parser.h"

void
example_parse(const char *query_string)
{
    List       *parsetree_list;
    ListCell   *lc;
    
    parsetree_list = raw_parser(query_string, RAW_PARSE_DEFAULT);
    
    foreach(lc, parsetree_list)
    {
        RawStmt    *rawstmt = lfirst_node(RawStmt, lc);
        Node       *stmt = rawstmt->stmt;
        
        elog(INFO, "Statement type: %d", nodeTag(stmt));
    }
}
```

---

## Performance Implications

### Parsing is Fast, Usually

For typical queries, parsing takes microseconds—negligible compared to planning and execution. However:

1. **Very Long Queries**: Queries with thousands of elements (e.g., `INSERT INTO t VALUES (...), (...), ...` with 10,000 rows) can have measurable parse times.

2. **Complex Expressions**: Deeply nested expressions require more parser stack space.

3. **Repeated Parsing**: If you execute the same query 1000 times without using prepared statements, you parse it 1000 times.

### Prepared Statements Avoid Re-parsing

With the extended query protocol (used by most drivers), parsing happens once:

```python
# Python example with psycopg2
cursor.execute("SELECT * FROM users WHERE id = %s", (1,))  # Parsed
cursor.execute("SELECT * FROM users WHERE id = %s", (2,))  # NOT re-parsed
```

### Grammar Complexity and Parse Time

The PostgreSQL grammar is LALR(1), meaning it can be parsed deterministically with one token of lookahead. However, some constructs require the parser to try multiple alternatives:

```sql
-- Is this a function call or a column?
SELECT foo(bar) FROM t;  -- Could be function foo with arg bar
                         -- Or column foo with weird syntax

-- Is this a type cast or something else?
SELECT foo::bar FROM t;  -- Cast or... ?
```

The grammar handles these through productions with actions that check context.

---

## Advanced Topics and Edge Cases

### Syntax Errors

When the parser encounters an error, it attempts to provide a helpful message:

```sql
SELECT * FORM users;
-- ERROR:  syntax error at or near "users"
-- LINE 1: SELECT * FORM users;
--                       ^
```

The `^` indicator uses the `location` field stored in tokens and nodes.

### Dollar-Quoted Strings

PostgreSQL supports alternative string quoting:

```sql
SELECT $$This string doesn't need 'escape' handling$$;
SELECT $tag$Nested $$ works too$tag$;
```

The lexer handles this with states:

```c
/* In scan.l */
<xdolq>{dolqdelim}        {
    if (strcmp(yytext, dolqstart) == 0)
    {
        /* End of dollar-quoted string */
        BEGIN(INITIAL);
        return SCONST;
    }
}
```

### Unicode Escapes

PostgreSQL supports Unicode escapes in strings and identifiers:

```sql
SELECT U&'\0041\0042\0043';  -- Returns 'ABC'
SELECT U&"column\0020name";  -- Identifier with space
```

### Case Sensitivity

SQL keywords are case-insensitive. The lexer normalizes them:

```sql
SELECT * FROM users;
select * FROM users;
SeLeCt * fRoM users;
-- All equivalent
```

Identifiers are normally case-folded to lowercase:

```sql
SELECT * FROM Users;       -- Looks for table "users"
SELECT * FROM "Users";     -- Looks for table "Users" (quoted = exact)
```

### Adding New Syntax (Extension Example)

If you were adding a new statement type, you would:

1. Add token(s) to `kwlist.h` if needed
2. Add grammar rules to `gram.y`
3. Define a new node type in `parsenodes.h`
4. Implement the `makeNode` and handling code

---

## Hands-On Exercises

### Exercise 1: Token Identification (Basic)

**Objective**: Identify tokens in SQL queries.

For each query, list the tokens the lexer produces:

1. `SELECT 42;`
2. `SELECT 'hello';`
3. `SELECT a + b * c FROM t;`
4. `SELECT * FROM "Mixed Case" WHERE id = 1;`

**Expected Output**:
1. `SELECT, ICONST(42), ';'`
2. `SELECT, SCONST('hello'), ';'`
3. (work it out)
4. Note the quoted identifier

### Exercise 2: Parse Tree Drawing (Intermediate)

**Objective**: Sketch parse tree structures.

Draw the parse tree (like the examples above) for:

```sql
SELECT a.x, b.y
FROM alpha a, beta b
WHERE a.id = b.id AND b.active = true;
```

### Exercise 3: Error Messages (Intermediate)

**Objective**: Understand parser error reporting.

Try these invalid queries and explain why each fails:

```sql
-- 1
SELECT FROM users;

-- 2
SELECT * FROM WHERE id = 1;

-- 3
SELECT * FROM users WERE id = 1;

-- 4
SELECT 1 +;
```

### Exercise 4: Reserved Keywords (Intermediate)

**Objective**: Explore keyword classification.

1. Check if you can create a table named `table`:
   ```sql
   CREATE TABLE table (id int);
   ```

2. Try with quotes:
   ```sql
   CREATE TABLE "table" (id int);
   ```

3. Look up the keyword classification in `kwlist.h` (or online PostgreSQL documentation).

### Exercise 5: Parse Tree Debugging (Advanced)

**Objective**: View actual parse tree output.

1. Enable parse tree logging:
   ```sql
   SET debug_print_parse = on;
   SET client_min_messages = debug1;
   ```

2. Run these queries and examine the log:
   ```sql
   SELECT 1 + 2;
   SELECT * FROM pg_class WHERE oid = 1;
   SELECT a.relname FROM pg_class a WHERE a.relkind = 'r';
   ```

3. Find the parse tree output in the PostgreSQL log file.

---

## Key Takeaways

1. **PostgreSQL uses Flex and Bison**: The lexer (`scan.l`) tokenizes SQL, and the parser (`gram.y`) validates syntax and builds the parse tree.

2. **The parse tree is purely syntactic**: It doesn't know if tables exist or if types match—that's the analyzer's job.

3. **Everything is a Node**: Parse tree nodes share a common base with a `NodeTag` discriminator, enabling generic tree traversal.

4. **Keywords matter**: Reserved keywords can't be used as identifiers without quoting. Check `kwlist.h` or documentation for classification.

5. **SelectStmt is central**: It contains all parts of a SELECT query—target list, FROM clause, WHERE clause, and more.

6. **Locations are tracked**: Every token and node records its position in the original query for error reporting.

7. **Prepared statements skip re-parsing**: Using the extended protocol or explicit PREPARE avoids repeated parsing.

---

## Further Reading

### Source Files to Explore

| File | Description |
|------|-------------|
| `src/backend/parser/scan.l` | Lexer definition (Flex) |
| `src/backend/parser/gram.y` | Grammar definition (Bison) |
| `src/backend/parser/parser.c` | Parser entry point |
| `src/include/parser/kwlist.h` | Keyword list and classifications |
| `src/include/nodes/parsenodes.h` | Parse tree node definitions |
| `src/include/nodes/primnodes.h` | Primitive expression nodes |

### Documentation

- [PostgreSQL Docs: The Parser Stage](https://www.postgresql.org/docs/current/parser-stage.html)
- [PostgreSQL Docs: SQL Key Words](https://www.postgresql.org/docs/current/sql-keywords-appendix.html)
- Flex manual: https://westes.github.io/flex/manual/
- Bison manual: https://www.gnu.org/software/bison/manual/

### Next Lessons

- **Lesson 3**: The Analyzer — Semantic analysis and name resolution
- **Lesson 4**: The Rewriter — View expansion and the rule system
- **Lesson 5**: Introduction to the Query Planner

---

## References

### Source Code

- `src/backend/parser/scan.l` — Lexer specification
- `src/backend/parser/gram.y` — Grammar specification
- `src/backend/parser/parser.c` — Parser entry points
- `src/include/nodes/parsenodes.h` — Parse node definitions
- `src/include/parser/kwlist.h` — Keyword definitions

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `raw_parser()` | parser.c | Main parser entry point |
| `scanner_init()` | scan.l | Initialize lexer |
| `base_yyparse()` | gram.y (generated) | Bison parser entry |
| `ScanKeywordLookup()` | kwlookup.c | Look up keyword type |
| `makeNode()` | nodes.h | Allocate parse tree node |

### Key Structures

| Structure | Location | Purpose |
|-----------|----------|---------|
| `SelectStmt` | parsenodes.h | SELECT statement |
| `ResTarget` | parsenodes.h | Target list entry |
| `ColumnRef` | parsenodes.h | Column reference |
| `A_Expr` | parsenodes.h | Expression |
| `A_Const` | parsenodes.h | Constant value |
| `RangeVar` | primnodes.h | Table reference |
| `JoinExpr` | primnodes.h | JOIN expression |
