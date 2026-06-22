# Plan-and-Execute

**Plan-and-Execute** splits the agent loop into two components: a **Planner** that reasons about the whole task upfront, and one or more **Executors** that carry out individual steps — each running their own inner loop.

## Core Idea

ReAct plans one step at a time. Plan-and-Execute asks: what if we knew the whole plan upfront? The Planner LLM generates a multi-step plan before any execution begins. Each step is handed to an Executor that runs its own loop (often a ReAct-style loop) to complete it.

## Loop Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    PLAN-AND-EXECUTE LOOP                        │
│                                                                 │
│  PHASE 1: PLAN                                                  │
│  ──────────────                                                 │
│  Planner LLM receives: goal + available tools                   │
│  Planner outputs:                                               │
│    Step 1: "Search for recent papers on X"                      │
│    Step 2: "Fetch the top 3 results"                            │
│    Step 3: "Summarize key findings from each"                   │
│    Step 4: "Write a comparative analysis"                       │
│                                                                 │
│  PHASE 2: EXECUTE                                               │
│  ─────────────────                                              │
│  For each step:                                                 │
│    Executor receives: (goal, plan_step, prior_results)          │
│    Executor runs inner loop (e.g. ReAct):                       │
│      → calls tools as needed                                    │
│      → returns step result                                      │
│                                                                 │
│  PHASE 3: REPLAN (optional)                                     │
│  ──────────────────────────                                     │
│  If a step fails or produces unexpected output:                 │
│    Replanner LLM: update remaining plan given new information   │
│    Continue execution with updated plan                         │
│                                                                 │
│  PHASE 4: SYNTHESIZE                                            │
│  ───────────────────                                            │
│  All step results combined → final output                       │
└─────────────────────────────────────────────────────────────────┘
```

## Minimal Implementation

```python
def plan_and_execute(goal: str, planner_llm, executor_llm, tools, max_replans=3):
    # Phase 1: Plan
    plan = planner_llm.call(
        f"Create a step-by-step plan to accomplish this goal using the available tools.\n"
        f"Goal: {goal}\n"
        f"Tools: {list(tools.keys())}"
    )
    
    results = []
    replans = 0
    
    # Phase 2: Execute
    for i, step in enumerate(plan.steps):
        try:
            result = react_loop(
                goal=f"Complete this step: {step}\nContext: {results}",
                tools=tools,
                llm=executor_llm
            )
            results.append({"step": step, "result": result})
        
        except Exception as e:
            if replans >= max_replans:
                raise
            
            # Phase 3: Replan on failure
            plan = planner_llm.call(
                f"Step {i+1} failed: {e}\n"
                f"Completed so far: {results}\n"
                f"Remaining steps: {plan.steps[i:]}\n"
                f"Please revise the plan."
            )
            replans += 1
    
    # Phase 4: Synthesize
    return synthesize(goal, results)
```

## Planner Prompt Pattern

```
You are a planning agent. Given a goal and a list of available tools,
create a step-by-step plan to accomplish the goal.

Each step should:
- Be specific and actionable
- Reference a specific tool or capability
- Produce a concrete output

Return a numbered list of steps. Be thorough but efficient.

Goal: {goal}
Available tools: {tools}
```

## Example Trace

```
Goal: "Research the top 3 electric vehicle manufacturers and compare their
       2024 sales figures, key models, and market positioning."

PLAN:
  Step 1: Search for 2024 EV manufacturer rankings by sales volume
  Step 2: Identify the top 3 manufacturers from search results
  Step 3: For each manufacturer, search for 2024 sales figures
  Step 4: For each manufacturer, fetch key models and market positioning
  Step 5: Write a structured comparison across all three dimensions

EXECUTE Step 1:
  → search("top EV manufacturers by sales 2024")
  → Result: Tesla, BYD, Volkswagen Group are top 3

EXECUTE Step 2:
  → Result: Confirmed: Tesla (#1), BYD (#2), VW Group (#3)

EXECUTE Step 3 (Tesla):
  → search("Tesla 2024 annual sales figures")
  → Result: 1.79 million vehicles

EXECUTE Step 3 (BYD):
  → search("BYD 2024 EV sales figures")
  → Result: 1.76 million pure EVs (plus 3.4M total including PHEVs)

... [continues for remaining steps]

SYNTHESIZE:
  → structured comparison report
```

## Strengths

- **Reasons about the whole task**: Unlike ReAct's one-step-at-a-time approach, the planner considers dependencies and sequencing upfront
- **Enables parallelism**: Independent steps can be executed concurrently by multiple executors
- **Smaller context per executor**: Each executor only needs the step, not the full history
- **Clearer progress tracking**: Steps are enumerable and checkable
- **Better for large tasks**: Tasks that clearly decompose into sub-tasks benefit most

## Limitations

- **Plan quality determines outcome quality**: A bad plan propagates to all steps
- **Less adaptive**: If the environment changes significantly mid-task, the plan may become stale
- **Replanning overhead**: Failures require replanning, which adds latency and cost
- **Over-planning on simple tasks**: For straightforward tasks, the planning overhead isn't worth it

## Comparison with ReAct

| Dimension | ReAct | Plan-and-Execute |
|-----------|-------|-----------------|
| Planning horizon | One step at a time | Whole task upfront |
| Adaptability | High (reacts to each observation) | Lower (plan may go stale) |
| Context growth | Linear (all history in one context) | Bounded per executor |
| Parallelism | Sequential | Parallel steps possible |
| Best for | Unknown paths, exploratory tasks | Decomposable, large tasks |

## When to Use

Plan-and-Execute is a good choice when:
- The task is large enough to benefit from parallelism
- The task structure is relatively predictable from the goal description
- You want to reduce context length per LLM call
- You need clear progress tracking (each step is checkable)

Use ReAct instead when:
- The task path depends heavily on early results
- The task is exploratory or unpredictable
- Simplicity is more important than efficiency

## Variants

| Variant | Description |
|---------|-------------|
| **LLMCompiler** | Parallel Plan-and-Execute; steps are executed concurrently using a dependency graph |
| **ReWOO** | Decouples planning from observations; generates all tool calls upfront, executes in batch |
| **Hierarchical P&E** | Planner generates sub-plans; each sub-plan has its own executor |

## Source

LangChain (framework authors; LangGraph implements Plan-and-Execute), *Planning Agents*:
[blog.langchain.com/planning-agents](https://blog.langchain.com/planning-agents/)

> "a 'Planner, which prompts an LLM to generate a multi-step plan to complete a large task' and 'Executor(s), which accept the user query and a step in the plan and invoke 1 or more tools to complete that task.'"
