# Example: ReWOO Loop (Plan All Tools Upfront → Execute in Batch → Synthesize)

**ReWOO** (Reasoning Without Observation) is a named agent loop pattern that decouples planning from execution. Instead of interleaving reasoning and tool calls (like ReAct), ReWOO generates a complete tool-call plan upfront — before any tools are executed — then executes all tools in batch and synthesizes the results. This dramatically reduces token consumption compared to ReAct.

## Real Implementation

| Repo | Framework | Description |
|------|-----------|-------------|
| [langchain-ai/langgraph (rewoo tutorial)](https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/rewoo/rewoo.ipynb) | LangGraph | Official ReWOO implementation as a LangGraph notebook |

## Why ReWOO Exists

In ReAct, every tool call requires a full LLM call:
```
LLM call 1: Thought + Action 1
Tool execution 1 → Observation 1
LLM call 2: Thought + Action 2 (with all of 1's context)
Tool execution 2 → Observation 2
LLM call 3: ...
```

For N tool calls, ReAct requires N LLM calls, each with a growing context.

ReWOO does it in 2 LLM calls regardless of N:
```
LLM call 1 (Planner): Generate the complete plan for all N tool calls
Tool executions 1..N (in parallel if independent)
LLM call 2 (Solver): Synthesize all results into the final answer
```

This is **verified** to reduce token consumption (3-0 adversarial verification).

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ReWOO LOOP                               │
│                                                                  │
│  PHASE 1: PLAN (one LLM call)                                   │
│  ─────────────────────────────                                   │
│  Planner receives: question + available tools                    │
│  Planner outputs: complete plan with all tool calls              │
│    Plan:                                                         │
│    #E1 = search("current CEO of Apple")                         │
│    #E2 = search("net worth of #E1")  ← can reference prior vars │
│    #E3 = calculate("#E2 / 1000000")                             │
│                                                                  │
│  PHASE 2: EXECUTE (parallel where possible)                     │
│  ──────────────────────────────────────────                      │
│  Tool 1: search("current CEO of Apple") → "Tim Cook"            │
│  Tool 2: search("net worth of Tim Cook") → "$2.2B"              │
│  Tool 3: calculate("2200000000 / 1000000") → "2200"             │
│                                                                  │
│  PHASE 3: SOLVE (one LLM call)                                  │
│  ─────────────────────────────                                   │
│  Solver receives: question + plan + all results                  │
│  Solver outputs: final answer with reasoning                     │
└─────────────────────────────────────────────────────────────────┘
```

## Stopping Condition

ReWOO's stopping condition is **deterministic** — it does not depend on the model deciding to stop. The loop ends when all planned tool calls have been executed and the Solver LLM has produced its output. There is no iterative loop in the traditional sense — it's a fixed three-phase pipeline.

This is a key difference from ReAct: ReWOO always terminates in exactly 2 LLM calls + N tool executions.

## LangGraph Implementation

From the [official LangGraph ReWOO tutorial](https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/rewoo/rewoo.ipynb):

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class ReWOOState(TypedDict):
    task: str
    plan_string: str        # Raw plan text from Planner
    steps: list             # Parsed (tool, var, description) tuples
    results: dict           # {var_name: result} for each executed step
    result: str             # Final answer from Solver

# --- Planner ---
PLANNER_PROMPT = """For the following task, make plans that can solve the problem step by step.
For each plan, indicate which external tool together with tool input to retrieve evidence.
Store the evidence into a variable #E that can be called by later tools.

Tools can be one of the following:
(1) Google[input]: Worker that searches results from Google.
(2) LLM[input]: A pretrained LLM like yourself. Only if you can answer confidently.
(3) Wikipedia[input]: Search Wikipedia for the input.

For example,
Task: Thomas Edison's friend was ...
Plan: Search for Thomas Edison's friends. 
#E1 = Google[Thomas Edison's friends]
Plan: Find who ...
#E2 = Wikipedia[...]
...

Begin!
Task: {task}"""

def get_plan(state: ReWOOState) -> ReWOOState:
    task = state["task"]
    result = planner_llm.call(PLANNER_PROMPT.format(task=task))
    # Parse into steps: [(tool, var_name, description), ...]
    steps = parse_plan(result)
    return {**state, "plan_string": result, "steps": steps}

# --- Tool Executor ---
def tool_execution(state: ReWOOState) -> ReWOOState:
    """Execute each tool call, substituting prior results for #E variables."""
    results = {}
    
    for tool_name, var_name, description in state["steps"]:
        # Substitute any #E references in the description
        for prev_var, prev_result in results.items():
            description = description.replace(prev_var, prev_result)
        
        # Execute the tool
        if tool_name == "Google":
            results[var_name] = google_search(description)
        elif tool_name == "Wikipedia":
            results[var_name] = wikipedia_search(description)
        elif tool_name == "LLM":
            results[var_name] = llm.call(description)
    
    return {**state, "results": results}

# --- Solver ---
SOLVER_PROMPT = """Solve the following task or problem. To solve the problem, 
we have made step-by-step Plan and retrieved corresponding Evidence to each Plan.
Use them with caution since long evidence might contain irrelevant information.

{plan}

Now solve the question or task according to provided Evidence above.
Respond with the answer directly with no extra words.

Task: {task}
Response:"""

def solve(state: ReWOOState) -> ReWOOState:
    # Build the plan+evidence string
    plan = state["plan_string"]
    for var_name, result in state["results"].items():
        plan += f"\n{var_name} = {result}"
    
    answer = solver_llm.call(SOLVER_PROMPT.format(plan=plan, task=state["task"]))
    return {**state, "result": answer}

# Build graph — deterministic, no conditional edges needed
builder = StateGraph(ReWOOState)
builder.add_node("plan", get_plan)
builder.add_node("tool", tool_execution)
builder.add_node("solve", solve)

builder.set_entry_point("plan")
builder.add_edge("plan", "tool")
builder.add_edge("tool", "solve")
builder.add_edge("solve", END)

graph = builder.compile()
```

## Variable Substitution: The Key Mechanism

The planner can reference earlier results using `#E` variables:

```
Plan: Search for the current CEO of Apple.
#E1 = Google[current CEO of Apple]

Plan: Find the net worth of the CEO found in #E1.
#E2 = Google[net worth of #E1]

Plan: Convert the net worth from #E2 to millions.
#E3 = LLM[Convert this to millions: #E2]
```

During execution, `#E1` is replaced with the actual search result before `#E2`'s tool is called. This lets the plan express data dependencies without requiring interleaved LLM calls.

## ReWOO vs. ReAct: When to Use Each

| Dimension | ReAct | ReWOO |
|-----------|-------|-------|
| LLM calls for N tools | N | 2 |
| Token consumption | High (grows per turn) | Low (bounded) |
| Adaptability | High (responds to each observation) | Low (plan is fixed) |
| Parallel tool execution | No (sequential by default) | Yes (independent steps) |
| Error recovery | Yes (model sees errors, adapts) | No (plan doesn't update on errors) |
| Best for | Unpredictable tasks | Research/retrieval with known structure |

**Use ReWOO when:**
- The task is a knowledge lookup with multiple retrievals
- You want to minimize token costs
- Tool calls can be planned upfront without needing to see results first
- Some tool calls can run in parallel

**Use ReAct when:**
- Each step depends on the actual (not predicted) result of the prior step
- The task requires error handling and adaptation mid-execution
- The path through the task is unpredictable

## Parallel Execution

Since ReWOO plans all tool calls upfront, independent steps can run concurrently:

```python
import asyncio

async def tool_execution_parallel(state: ReWOOState) -> ReWOOState:
    """Execute independent steps in parallel."""
    results = {}
    
    # Build dependency graph from #E references
    deps = build_dependency_graph(state["steps"])
    
    # Execute steps level by level (topological sort)
    for level in topological_levels(deps):
        level_results = await asyncio.gather(*[
            execute_step(step, results) 
            for step in level
        ])
        for var_name, result in level_results:
            results[var_name] = result
    
    return {**state, "results": results}
```

For three independent searches, this cuts wall-clock time by ~3x.

## Known Pitfalls

**Plan quality is everything**: If the planner misunderstands the task or plans the wrong tools, there's no correction mechanism. Unlike ReAct, errors don't get fed back to the model.

**Variable substitution errors**: If `#E1` returns "No results found" and `#E2` searches for "No results found's net worth", the plan breaks silently. Add validation after substitution.

**Long plan strings**: Very complex tasks generate long plans that may exceed context. ReWOO works best for tasks with 3-8 tool calls.

**No streaming progress**: Since all tools execute before the solver runs, users see no intermediate output. For long-running tasks, add progress events from the tool executor.

## Sources

- [github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/rewoo/rewoo.ipynb](https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/rewoo/rewoo.ipynb) — official LangGraph ReWOO tutorial
- Original paper: Xu et al., *ReWOO: Decoupling Reasoning from Observations for Efficient Augmented Language Models* (2023) — [arxiv.org/abs/2305.18323](https://arxiv.org/abs/2305.18323)
