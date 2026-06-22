# Anatomy of an Agent Loop

An agent loop is an iterative execution cycle. At its core, it is a `while` loop that runs until a task is complete or a stopping condition fires.

## The Minimal Loop

```python
history = [system_prompt, user_goal]
done = False

while not done:
    action = llm.call(history)          # LLM reasons and picks next action
    result = tools.execute(action)      # Tool is called; real observation collected
    history.append(result)              # Observation fed back into context
    done = stopping_condition(result)   # Check whether to continue
```

This is the six-line core that everything else builds on. The loop differs from a one-shot prompt because the model **learns from each result in-context** — not via weight updates — and takes the next step based on real ground truth from the environment.

## The Four Stages

Every iteration of an agent loop goes through four stages:

```
┌─────────────────────────────────────────────────────────────┐
│                     AGENT LOOP ITERATION                    │
│                                                             │
│  1. ASSEMBLE CONTEXT                                        │
│     ├── System prompt + goal                                │
│     ├── Available tools (schemas)                           │
│     ├── Prior observations (history or retrieved memory)    │
│     └── Any injected context (user message, event payload)  │
│                           │                                 │
│                           ▼                                 │
│  2. REASON & SELECT                                         │
│     ├── LLM processes assembled context                     │
│     ├── Optionally generates a thought / reasoning trace    │
│     └── Selects the next action (tool call or final answer) │
│                           │                                 │
│                           ▼                                 │
│  3. EXECUTE                                                 │
│     ├── Tool is called with LLM-generated parameters        │
│     ├── Result is a real-world observation                  │
│     │   (search results, code output, API response, etc.)   │
│     └── Errors are also valid observations                  │
│                           │                                 │
│                           ▼                                 │
│  4. OBSERVE & FEED BACK                                     │
│     ├── Observation appended to history                     │
│     ├── Stopping condition evaluated                        │
│     └── Loop continues or exits                             │
└─────────────────────────────────────────────────────────────┘
```

## The Key Insight: Ground Truth

The loop's power comes from "gaining ground truth from the environment at each step." The LLM doesn't hallucinate about what the search returned — it gets the actual search results. It doesn't guess what the code output was — it gets the actual output.

This is why loops outperform single-shot prompts for complex tasks: each observation is a fact, and the model reasons on facts, not predictions about facts.

## Stopping Conditions

A loop needs at least one exit path. Types:

| Condition | Description | Example |
|-----------|-------------|---------|
| **Success** | The goal is verifiably met | Evaluator scores output ≥ 9/10 |
| **Terminal action** | The model emits a "done" or "finish" action | `Action: finish("answer")` |
| **Max iterations** | Hard cap on turns | `if turn > 25: break` |
| **Token/cost budget** | Spend threshold reached | `if cost > $0.50: stop` |
| **Timeout** | Wall-clock limit | `if elapsed > 300s: stop` |
| **Human escalation** | Confidence too low; pause and ask | `if confidence < 0.4: ask_human()` |

> **Important:** The claim that loops *must* be hard-bounded with mandatory human handoffs was refuted by adversarial verification (1-2 vote). Stopping conditions are necessary; the form they take depends on the use case.

## What Happens Without Good Stopping Conditions

Without stopping conditions, loops can:
- Run indefinitely, consuming tokens and money
- Get stuck in cycles (same action → same observation → same action)
- Produce diminishing-returns iterations that erode quality
- Fail silently (no output, no error, just... spinning)

## The Loop vs. One-Shot vs. Chain

| Pattern | Structure | When to use |
|---------|-----------|-------------|
| **One-shot prompt** | Single LLM call | Simple, well-defined tasks with no tool use |
| **Chain** | Fixed sequence of LLM calls | Predictable multi-step pipelines |
| **Loop** | Iterative; next step depends on observation | Tasks where the path depends on results |
| **Agent loop** | Loop with tool execution and environmental feedback | Open-ended tasks requiring real-world interaction |

## Common Misconceptions

**"A loop is just a chain with a for-loop."**
No. A chain's steps are predetermined. A loop's steps are determined by what the model observes at runtime. The model may take 2 turns or 20 depending on what it finds.

**"Loops always need human checkpoints."**
Not true for all loops. Many production loops run fully autonomously with only programmatic stopping conditions. Human checkpoints are appropriate for high-stakes or irreversible actions.

**"The five-stage Perceive-Reason-Plan-Act-Observe model."**
This specific five-stage taxonomy was refuted by adversarial verification (0-3 vote). The simpler four-stage model (Assemble → Reason → Execute → Observe) is better supported by primary sources.

---

*Sources: Anthropic (Building Effective Agents), Oracle Developer Blog, Forward-Future Loop Library, ReAct paper (Yao et al. 2022)*
