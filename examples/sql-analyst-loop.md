# Example: SQL Analyst Loop (Question → Query → Analyze → Requery → Answer)

A SQL analyst loop takes a natural language question, generates SQL, runs it against a database, inspects the result (or the error), and either returns the answer or regenerates the query. It's one of the highest-ROI agent loops for data teams: it replaces hours of manual query iteration with an automated cycle.

## Real Implementations

| Repo | Framework | Description |
|------|-----------|-------------|
| [mallahyari/langgraph-sql-agent](https://github.com/mallahyari/langgraph-sql-agent) | LangGraph | Full graph with schema introspection + retry cap |
| [adamfaik/sql-agent](https://github.com/adamfaik/sql-agent) | LangGraph + SQLAlchemy | Corrective retry from SQL execution errors |

## Architecture

```
MANUAL TRIGGER: user types a natural language question
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│                    SQL ANALYST LOOP                           │
│                                                               │
│  State: {question, schema, sql, result, error, retry_count}   │
│                                                               │
│  1. introspect_schema                                         │
│     → fetch table names, columns, sample rows                 │
│                                                               │
│  2. generate_sql                                              │
│     → LLM writes SQL from question + schema context           │
│                                                               │
│  3. execute_sql                                               │
│     → run query; capture result or error                      │
│                                                               │
│  4. route:                                                    │
│     ├─ error AND retry_count < 3 → back to generate_sql       │
│     ├─ result looks incomplete → generate_sql (refinement)    │
│     └─ result looks good → analyze_and_answer                 │
│                                                               │
│  5. analyze_and_answer                                        │
│     → LLM interprets rows; writes human-readable answer       │
└──────────────────────────────────────────────────────────────┘
```

## LangGraph Implementation

From [mallahyari/langgraph-sql-agent](https://github.com/mallahyari/langgraph-sql-agent):

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Optional

class SQLAgentState(TypedDict):
    question: str
    schema: str
    sql_query: str
    query_result: Optional[str]
    error: Optional[str]
    answer: str
    retry_count: int

# Node 1: Fetch DB schema for context
def get_schema(state: SQLAgentState) -> SQLAgentState:
    schema = db.get_table_info()  # SQLAlchemy introspection
    return {**state, "schema": schema}

# Node 2: Generate SQL from question + schema
def generate_sql(state: SQLAgentState) -> SQLAgentState:
    prompt = f"""
    Given this database schema:
    {state['schema']}
    
    Write a SQL query to answer: {state['question']}
    
    Previous attempt (if any): {state.get('sql_query', 'none')}
    Previous error (if any): {state.get('error', 'none')}
    
    Return only the SQL query, no explanation.
    """
    sql = llm.call(prompt)
    return {**state, "sql_query": sql, "retry_count": state["retry_count"] + 1}

# Node 3: Execute the query
def execute_sql(state: SQLAgentState) -> SQLAgentState:
    try:
        result = db.run(state["sql_query"])
        return {**state, "query_result": result, "error": None}
    except Exception as e:
        return {**state, "query_result": None, "error": str(e)}

# Node 4: Turn raw result into a human-readable answer
def analyze_result(state: SQLAgentState) -> SQLAgentState:
    answer = llm.call(
        f"Question: {state['question']}\n"
        f"SQL result: {state['query_result']}\n"
        f"Answer the question in plain English based on the data."
    )
    return {**state, "answer": answer}

# Routing logic
def route_after_execution(state: SQLAgentState) -> str:
    if state["error"] and state["retry_count"] < 3:
        return "generate_sql"   # Retry with error context
    if state["query_result"] is not None:
        return "analyze_result" # Success path
    return END                  # Exhausted retries

# Build the graph
builder = StateGraph(SQLAgentState)
builder.add_node("get_schema", get_schema)
builder.add_node("generate_sql", generate_sql)
builder.add_node("execute_sql", execute_sql)
builder.add_node("analyze_result", analyze_result)

builder.set_entry_point("get_schema")
builder.add_edge("get_schema", "generate_sql")
builder.add_edge("generate_sql", "execute_sql")
builder.add_conditional_edges("execute_sql", route_after_execution)
builder.add_edge("analyze_result", END)

graph = builder.compile()
```

## Stopping Conditions

| Condition | Route |
|-----------|-------|
| Query succeeds | `execute_sql` → `analyze_result` → END |
| Query fails, retry_count < 3 | `execute_sql` → `generate_sql` (with error in context) |
| retry_count >= 3 | END (return error message to user) |

**Key insight**: The retry cap is 3, not 25. SQL errors are usually syntax errors or schema mismatches — if the model can't fix them in 3 tries with the error message in context, more iterations won't help without human intervention.

## What Goes in the Error Context

When routing back to `generate_sql`, include the full error message:

```python
# Bad: vague context
prompt = f"The previous query failed. Write a new one."

# Good: specific error in context
prompt = f"""
The previous SQL query:
{state['sql_query']}

Failed with this error:
{state['error']}

Common causes:
- Column name doesn't exist: check the schema
- Ambiguous column name: qualify with table name
- Type mismatch: cast appropriately

Write a corrected SQL query.
"""
```

## Schema Introspection

The quality of the schema context determines 80% of the loop's success. Give the model:

```python
def build_schema_context(db, sample_rows=3):
    context = []
    for table in db.get_table_names():
        columns = db.get_columns(table)
        samples = db.execute(f"SELECT * FROM {table} LIMIT {sample_rows}")
        context.append(f"""
Table: {table}
Columns: {', '.join(f"{c['name']} ({c['type']})" for c in columns)}
Sample rows:
{format_rows(samples)}
""")
    return "\n".join(context)
```

**Include sample rows**: A model that sees `status: ('active', 'inactive', 'pending')` writes better WHERE clauses than one that only sees `status: VARCHAR`.

## Result Validation

Before routing to `analyze_result`, optionally validate the result looks reasonable:

```python
def validate_result(state: SQLAgentState) -> str:
    result = state["query_result"]
    
    # Empty result on an aggregation query is suspicious
    if result == "[]" and "COUNT" in state["sql_query"]:
        if state["retry_count"] < 3:
            return "generate_sql"  # Try a different approach
    
    # Result has wrong columns
    expected_cols = extract_expected_columns(state["question"])
    actual_cols = get_column_names(result)
    if expected_cols and not any(e in actual_cols for e in expected_cols):
        return "generate_sql"
    
    return "analyze_result"
```

## Multi-Step SQL (Sub-Query Loops)

For complex questions requiring multiple queries:

```python
# Question: "Which customers who bought product X also bought product Y 
#            within 30 days?"

# Step 1: Find customers who bought X
customers_X = db.run("SELECT customer_id FROM orders WHERE product_id = 'X'")

# Step 2: Among those, find who bought Y within 30 days
result = db.run(f"""
    SELECT o1.customer_id 
    FROM orders o1
    JOIN orders o2 ON o1.customer_id = o2.customer_id
    WHERE o1.product_id = 'X'
      AND o2.product_id = 'Y'
      AND ABS(JULIANDAY(o2.order_date) - JULIANDAY(o1.order_date)) <= 30
""")
```

The agent decides when a multi-step approach is needed based on the complexity of the question.

## Integration with BI Tools

```python
# Return results in a format BI tools can consume
def format_for_viz(result, question):
    answer = analyze_result(result, question)
    return {
        "answer": answer,
        "sql": state["sql_query"],
        "data": result,
        "chart_suggestion": suggest_chart_type(result),  # bar / line / table
        "exportable_csv": to_csv(result)
    }
```

## Known Pitfalls

**Schema drift**: The introspected schema is a snapshot. If the DB schema changes (columns renamed, tables dropped), the loop fails until the schema is re-introspected.

**Hallucinated table names**: The model may invent table names not in the schema. Fix: include `Only use these tables: {table_names}` in the prompt.

**Injection risk**: Never interpolate user input directly into SQL. Always use the LLM to translate, then validate the SQL against a whitelist of allowed operations before executing.

**Large result sets**: A `SELECT *` on a million-row table fills the context window. Add `LIMIT 100` to all generated queries unless the question explicitly requires all rows.

**Ambiguous questions**: "Show me sales" — by day? by region? by product? Add a clarification step before the loop if the question is underspecified.

## Sources

- [github.com/mallahyari/langgraph-sql-agent](https://github.com/mallahyari/langgraph-sql-agent) — full LangGraph SQL agent with schema introspection and retry
- [github.com/adamfaik/sql-agent](https://github.com/adamfaik/sql-agent) — corrective retry pattern from SQL execution errors
