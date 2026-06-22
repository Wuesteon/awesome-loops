# Agents vs. Workflows

One of the most important distinctions in agentic AI — and one of the most commonly blurred — is the difference between **agents** and **workflows**.

## Anthropic's Definition

> "Workflows are systems where LLMs and tools are orchestrated through **predefined code paths**. Agents, on the other hand, are systems where LLMs **dynamically direct their own processes** and tool usage, maintaining control over how they accomplish tasks."
>
> — Anthropic, *Building Effective Agents* (December 2024)

## The Core Difference: Who Controls the Loop?

| Dimension | Workflow | Agent |
|-----------|----------|-------|
| **Control flow** | Programmer-defined code path | LLM decides at runtime |
| **Tool selection** | Predetermined or routed by code | LLM selects from available tools |
| **Branching logic** | `if/else` in your code | LLM's reasoning output |
| **Loop continuation** | Coded stopping condition | LLM decides whether to continue |
| **Adaptability** | Fixed to designed paths | Adapts to unexpected observations |
| **Predictability** | High | Lower (by design) |
| **Debuggability** | High | Harder — requires trace inspection |

## Both Use Loops

Both workflows and agents can use loops. The difference is in what controls the loop:

**Workflow loop (evaluator-optimizer):**
```python
for i in range(max_iterations):
    draft = generator_llm.call(task)
    score = evaluator_llm.call(draft)
    if score >= threshold:
        break
return draft
```
The loop structure, iteration limit, and success threshold are all coded. The LLMs are called, but they don't control the loop.

**Agent loop:**
```python
while True:
    response = llm.call(history)
    if response.is_final_answer:
        break
    result = tools.execute(response.tool_call)
    history.append(result)
```
The LLM decides whether to call another tool or emit a final answer. It controls the loop's continuation.

## The Spectrum

In practice, most production systems sit on a spectrum:

```
Pure workflow                                          Pure agent
     │                                                     │
     ▼                                                     ▼
All paths     Conditional     Loop with      LLM decides   LLM fully
pre-coded     branching       coded bounds   most steps    autonomous
              by code
```

This is not a flaw — it's a design choice. The more autonomy you give the LLM, the more adaptive but less predictable the system becomes.

## When to Use Each

**Prefer a workflow when:**
- The task structure is well-understood and stable
- Predictability and auditability matter more than flexibility
- Latency or cost per task is a constraint
- You need to guarantee certain steps always happen (compliance, safety)
- You're building something new and want to validate before adding autonomy

**Prefer an agent loop when:**
- The path to the answer isn't known in advance
- The task requires responding to unexpected tool results
- The scope of the task may vary significantly across runs
- You're working on open-ended research, exploration, or creative tasks

> **Anthropic's recommendation:** Start with workflows. Only add autonomy when workflows demonstrably fail to handle the task's complexity. Simpler systems are more reliable.

## Why the Boundary Is Fuzzy

Industry practitioners acknowledge the agent/workflow line is contested:
- Many "agents" are really sophisticated workflows with LLM-powered routing
- Many "workflows" contain agent-like sub-loops
- The term "agentic" is often used for marketing, not precision

What matters more than the label is understanding **which system controls which decisions** — and making that explicit in your architecture.

---

*Sources: Anthropic (Building Effective Agents, Dec 2024)*
