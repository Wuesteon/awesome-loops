# Orchestrator-Workers

The **Orchestrator-Workers** pattern nests agent loops: an outer orchestrator loop decomposes a task and manages sub-tasks, while inner worker loops execute those sub-tasks in parallel or sequence.

## Core Idea

Some tasks are too large or too complex for a single context window and a single agent. The Orchestrator-Workers pattern scales by decomposition: the orchestrator is an agent that decides *what* needs to be done and spawns worker agents to do each piece.

## Loop Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ORCHESTRATOR-WORKERS LOOP                       │
│                                                                     │
│  ORCHESTRATOR (outer loop):                                         │
│    while not done:                                                  │
│      → reason about task state and remaining work                   │
│      → decide: spawn worker / synthesize results / finish           │
│      → if spawn: delegate sub-task to a worker                      │
│      → collect worker result                                        │
│      → update task state                                            │
│                                                                     │
│  WORKER (inner loop, per sub-task):                                 │
│    goal = sub-task from orchestrator                                │
│    while not done:                                                  │
│      → use tools (search, code, API, etc.)                          │
│      → observe results                                              │
│      → continue or return result to orchestrator                    │
└─────────────────────────────────────────────────────────────────────┘
```

## Architecture Diagram

```
                    USER REQUEST
                         │
                         ▼
              ┌──────────────────────┐
              │    ORCHESTRATOR       │
              │    (outer loop)      │
              │                      │
              │  1. Decompose task   │
              │  2. Track state      │
              │  3. Synthesize       │
              └──────────────────────┘
               │         │         │
               ▼         ▼         ▼
         ┌─────────┐ ┌─────────┐ ┌─────────┐
         │ Worker A│ │ Worker B│ │ Worker C│
         │ (ReAct) │ │ (ReAct) │ │ (P&E)  │
         │ sub-task│ │ sub-task│ │ sub-task│
         └─────────┘ └─────────┘ └─────────┘
               │         │         │
               └────┬────┘         │
                    └──────────────┘
                           │
                    Results flow back
                    to orchestrator
```

## Minimal Implementation

```python
class Orchestrator:
    def __init__(self, llm, worker_factory, tools):
        self.llm = llm
        self.worker_factory = worker_factory
        self.tools = tools
        self.results = {}
    
    def run(self, goal: str, max_turns=20):
        history = [{"role": "user", "content": goal}]
        
        for turn in range(max_turns):
            decision = self.llm.call(
                history,
                system=ORCHESTRATOR_PROMPT,
                tools=["spawn_worker", "synthesize", "finish"]
            )
            
            if decision.action == "finish":
                return decision.final_answer
            
            if decision.action == "spawn_worker":
                sub_task = decision.sub_task
                worker_id = decision.worker_id
                
                # Spawn worker (may run in parallel)
                worker = self.worker_factory(sub_task, self.tools)
                result = worker.run()
                self.results[worker_id] = result
                
                history.append({
                    "role": "tool",
                    "content": f"Worker {worker_id} completed: {result}"
                })
            
            if decision.action == "synthesize":
                final = self.synthesize(goal, self.results)
                history.append({"role": "tool", "content": final})
        
        raise MaxIterationsError()
```

## Orchestrator System Prompt Pattern

```
You are an orchestrator. Your job is to decompose complex tasks,
delegate sub-tasks to workers, collect results, and synthesize
a final answer.

You can:
- spawn_worker(sub_task, worker_id): Delegate a sub-task to a worker agent
- synthesize(): Combine all worker results collected so far
- finish(answer): Return the final answer

Think about what has been completed and what remains.
Spawn workers for independent sub-tasks.
Synthesize when all needed results are collected.
```

## Example Trace

```
Goal: "Write a comprehensive market analysis for the electric vehicle industry"

Orchestrator turn 1:
  → spawn_worker("Compile 2024 EV sales data for top 5 manufacturers", worker_A)

Worker A (ReAct loop):
  → search("EV sales 2024 Tesla BYD Volkswagen")
  → fetch(top_results)
  → return: structured sales data table

Orchestrator turn 2:
  → spawn_worker("Research EV battery technology trends 2024-2025", worker_B)
  → spawn_worker("Analyze government EV incentive policies by region", worker_C)
  [workers B and C run in parallel]

Worker B returns: battery trends report
Worker C returns: policy landscape report

Orchestrator turn 3:
  → spawn_worker("Write competitive analysis section using worker_A results", worker_D)

Worker D returns: competitive analysis section

Orchestrator turn 4:
  → synthesize()
  → combine sales data + tech trends + policy + competitive analysis
  → finish("Complete market analysis report...")
```

## Parallelism Patterns

### Fire-and-collect (async parallel)
```python
import asyncio

async def orchestrate_parallel(tasks, worker_factory):
    workers = [worker_factory(task) for task in tasks]
    results = await asyncio.gather(*[w.run_async() for w in workers])
    return results
```

### Dependency-aware execution
```python
# Execute only when dependencies are met
dag = {
    "search": [],
    "fetch": ["search"],
    "summarize": ["fetch"],
    "compare": ["summarize"]
}
```

### Speculative parallelism
```python
# Start likely-needed workers before confirmed by orchestrator
likely_needed = predict_next_workers(current_state)
speculative = [start_worker(t) for t in likely_needed]
# Cancel if not needed; use result if they finish in time
```

## Worker Specialization

Workers can be specialized for different tool sets or task types:

| Worker type | Tools | Best for |
|-------------|-------|----------|
| **Researcher** | search, fetch, extract | Information gathering |
| **Coder** | code_interpreter, test_runner | Writing and testing code |
| **Writer** | (LLM only) | Drafting prose sections |
| **Validator** | test_runner, lint, type_check | Verifying correctness |
| **Data analyst** | SQL, pandas, charts | Quantitative analysis |

## Strengths

- **Scales beyond context limits**: Each worker gets a focused, bounded task
- **Parallelism**: Independent workers run concurrently, reducing wall-clock time
- **Specialization**: Different workers can use different tools, models, or prompts
- **Clear decomposition**: Task breakdown is explicit and inspectable
- **Partial failure resilience**: One worker failure doesn't necessarily fail the whole task

## Limitations

- **Harder to debug**: Errors propagate across agent boundaries; traces span multiple loops
- **Cost scales with workers**: More parallelism = more LLM calls = higher cost
- **Orchestrator quality matters**: A poor decomposition leads to poor results regardless of worker quality
- **Coordination overhead**: Workers must return results in a form the orchestrator can use
- **Error propagation**: If Worker A returns bad data that Worker B builds on, the error compounds

## When to Use

Orchestrator-Workers works well when:
- The task clearly decomposes into independent sub-tasks
- Sub-tasks can run in parallel
- The full task is too large for a single context window
- Different sub-tasks require different tools or expertise

Use a simpler pattern (ReAct, Plan-and-Execute) when:
- Sub-tasks are sequential and tightly coupled
- The decomposition isn't clear upfront
- The overhead of orchestration exceeds the benefit

## Nesting Depth

Orchestrators can themselves be workers in a higher-level orchestrator:

```
L3 Orchestrator: "Complete the quarterly business report"
  → L2 Orchestrator: "Complete market analysis section"
       → L1 Workers: individual research tasks
  → L2 Orchestrator: "Complete financial section"
       → L1 Workers: data compilation tasks
```

Practical systems rarely go beyond 2-3 levels. Each level adds coordination overhead and debugging complexity.

## Source

Anthropic, *Building Effective Agents* (December 2024):
[anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents)

> One of the five named workflow/agent patterns. Recommended when tasks are "complex enough to benefit from distributing the work across specialized sub-agents."
