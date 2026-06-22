# Awesome Loops [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> A curated list of resources, patterns, and examples for agent loops — the iterative execution architecture at the heart of autonomous AI systems.

An **agent loop** is the core of agentic AI: an LLM assembles context, reasons to pick an action, executes it (usually a tool call), observes the result, and feeds that observation back into the next iteration — repeating until the task is done or a stopping condition fires. In its simplest form, this reduces to a six-line `while` loop. In production, it becomes one of the most consequential design decisions in your system.

---

## Contents

- [Theory](#theory)
  - [What Is an Agent Loop?](#what-is-an-agent-loop)
  - [Agents vs. Workflows](#agents-vs-workflows)
  - [What Makes a Good Loop?](#what-makes-a-good-loop)
- [Loop Patterns](#loop-patterns)
  - [ReAct](#react)
  - [Reflexion](#reflexion)
  - [Plan-and-Execute](#plan-and-execute)
  - [Evaluator-Optimizer](#evaluator-optimizer)
  - [Orchestrator-Workers](#orchestrator-workers)
- [Trigger Types](#trigger-types)
  - [Manual / On-Demand](#manual--on-demand)
  - [Scheduled / Cron](#scheduled--cron)
  - [Event-Driven](#event-driven)
- [Production Hardening](#production-hardening)
- [Loop Libraries and Frameworks](#loop-libraries-and-frameworks)
- [Use Cases and Examples](#use-cases-and-examples)
- [Academic Papers](#academic-papers)
- [Further Reading](#further-reading)

---

## Theory

### What Is an Agent Loop?

At its core, an agent loop is:

```
while not done:
    context = assemble_context(history, tools)
    action  = llm.reason_and_select(context)
    result  = execute(action)
    history.append(result)
    done    = stopping_condition(result, history)
```

The loop differs from a one-shot prompt because the agent **learns from each result in-context** (not via weight updates) and takes the next useful step based on real observations — tool call results, code output, API responses.

> "They are typically just LLMs using tools based on environmental feedback in a loop."
> — Anthropic, *Building Effective Agents* (2024)

Each iteration has four stages:

| Stage | What happens |
|-------|-------------|
| **Assemble context** | Load memory, prior results, available tools, current goal |
| **Reason & select** | LLM picks the next action from available tools/choices |
| **Execute** | Tool is called; real-world observation is collected |
| **Observe & feed back** | Result is appended to context; loop checks stopping condition |

### Agents vs. Workflows

Anthropic draws the key distinction:

- **Workflows**: LLMs and tools orchestrated through **predefined code paths**. The control flow is set by the programmer.
- **Agents**: The LLM **dynamically directs its own process** and tool usage, maintaining control over how it accomplishes the task.

In practice, most production systems sit on a spectrum. A loop pattern can be either: a workflow loop where the iteration structure is coded, or an agentic loop where the LLM decides whether to continue and what to do next.

### What Makes a Good Loop?

A good loop answers four questions (from the Forward-Future loop library):

1. **What is the agent trying to accomplish?** — The goal must be explicit and evaluable.
2. **How will it know whether the latest attempt worked?** — Success criteria need to be checkable, ideally by a second LLM call or a deterministic test.
3. **What should it do with what it learned?** — The observation from each iteration must be usefully encoded back into the next context window.
4. **When should it finish or ask for help?** — A stopping condition (success, failure, budget, or human escalation) must exist.

> Note: The research found that loops do **not** need to be hard-bounded with mandatory human handoffs — that's a strong best practice, not an absolute rule. Stopping conditions are necessary; the *form* they take varies.

---

## Loop Patterns

### ReAct

**Reasoning + Acting** — interleave thought traces and actions in a single loop.

```
Thought: I need to find the population of France.
Action: search("population of France 2024")
Observation: France has approximately 68 million people.
Thought: I have the answer.
Action: finish("France has ~68 million people.")
```

**How it works:**
- The model generates a `Thought` (reasoning trace) before each `Action`
- The `Action` calls an external tool (search, code, API)
- The `Observation` is the tool's return value, appended to the prompt
- The loop continues until the model emits a `finish` action

**Strengths:** Grounds the model in real tool outputs; reduces hallucination compared to chain-of-thought-only reasoning; handles exceptions mid-task.

**Limitations:** One LLM call per tool invocation; plans only one sub-problem at a time; can yield sub-optimal trajectories on complex tasks.

**Source:** Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*, ICLR 2023 — [arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629)

---

### Reflexion

**Verbal self-reinforcement** — learn across episodes through linguistic feedback stored in episodic memory.

```
Episode 1:  attempt task → fail
Reflect:    "I failed because I didn't check X first. Next time I should Y."
Episode 2:  load reflection as context → attempt → succeed
```

**How it works:**
- The agent attempts a task and receives binary/scalar feedback from the environment
- A *reflection* LLM call converts that feedback into a **verbal summary** of what went wrong and what to try next
- The reflection is stored in an **episodic memory buffer**
- On the next episode, the reflection is prepended to the prompt as additional context
- No weight updates — all learning is in-context

**Strengths:** Enables multi-episode improvement without fine-tuning; interpretable (you can read the reflections); works with any base LLM.

**Limitations:** Memory buffer grows; limited by context window; weaker than true exploration for novel environments.

**Source:** Shinn et al., *Reflexion: Language Agents with Verbal Reinforcement Learning*, NeurIPS 2023 — [arxiv.org/abs/2303.11366](https://arxiv.org/abs/2303.11366)

---

### Plan-and-Execute

**Upfront planning + delegated execution** — split the loop into a planner and one or more executors.

```
Planner:    "Step 1: search for X. Step 2: summarize. Step 3: write report."
Executor 1: execute Step 1 → result
Executor 2: execute Step 2 → result
Executor 3: execute Step 3 → final output
```

**How it works:**
- A **Planner LLM** generates a full multi-step plan for the task upfront
- One or more **Executor LLMs** receive the user query + a single plan step and invoke tools to complete it
- Steps can be re-planned if an executor fails

**Strengths:** Forces the model to reason about the whole task; enables parallelism across steps; reduces context bloat per executor.

**Limitations:** Less adaptive than ReAct if the environment changes mid-task; plan quality determines outcome quality.

**Source:** LangChain, *Planning Agents* — [blog.langchain.com/planning-agents](https://blog.langchain.com/planning-agents/)

---

### Evaluator-Optimizer

**Generate → evaluate → refine** — one LLM produces, another critiques, repeat.

```
Generator:  draft response
Evaluator:  "The draft is missing X and Y. Score: 6/10."
Generator:  revised response (with evaluator feedback in context)
Evaluator:  "Better. Score: 9/10. Done."
```

**How it works:**
- A **generator** LLM produces a candidate response
- An **evaluator** LLM (or the same model with a different system prompt) scores or critiques the output
- The critique is fed back to the generator as context
- The loop runs until the evaluator approves or a max-iteration budget is hit

**Strengths:** Analogous to human edit cycles; evaluator can use different criteria than the generator; works well for writing, code, and structured outputs.

**When to use:** When quality feedback can be clearly articulated and the iteration cost is justified by the improvement ceiling.

**Source:** Anthropic, *Building Effective Agents* — [anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents)

---

### Orchestrator-Workers

**Multi-agent loops** — an orchestrator loop spawns and coordinates worker loops.

```
Orchestrator:   decompose task → spawn Worker A, Worker B, Worker C
Worker A loop:  runs its own ReAct loop on subtask A
Worker B loop:  runs its own ReAct loop on subtask B
Orchestrator:   collect results → synthesize → done
```

**How it works:**
- The **orchestrator** is itself an agent loop that decides how to decompose the task and which workers to invoke
- Each **worker** runs its own loop (often ReAct or Plan-and-Execute) on a scoped subtask
- Results flow back to the orchestrator, which may spawn further workers or synthesize a final answer

**Strengths:** Handles tasks too large or complex for a single context window; enables specialization; parallelizable.

**Strengths:** Harder to debug; error propagation across agent boundaries; cost scales with worker count.

**Source:** Anthropic, *Building Effective Agents* — [anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents)

---

## Trigger Types

How a loop starts is as important as how it runs. There are three families of triggers:

### Manual / On-Demand

The loop starts when a human explicitly asks for it.

| Mechanism | Example |
|-----------|---------|
| UI button / chat message | User sends a message; the agent loop begins |
| `workflow_dispatch` (GitHub) | Developer clicks "Run workflow" in GitHub UI |
| CLI command | `gh aw run my-agent-workflow` |
| REST API call | `POST /api/agents/run` with a payload |
| Webhook | External service POSTs to your endpoint; loop starts |

**Best for:** Tasks requiring human intent, tasks with variable inputs, debugging and testing.

---

### Scheduled / Cron

The loop starts on a time-based trigger, without a human in the loop.

| Format | Example |
|--------|---------|
| Standard cron syntax | `0 9 * * 1-5` — 9 AM on weekdays |
| Fixed intervals | `hourly`, `daily`, `weekly`, `monthly` |
| Fuzzy schedules (gh-aw) | `"daily around 14:00"` — with UTC offset support |
| Range schedules (gh-aw) | `"daily between 9:00 and 17:00"` |

```yaml
# GitHub Agentic Workflows example
on:
  schedule:
    - cron: "0 9 * * 1-5"
```

**Best for:** Monitoring, reporting, data sync, periodic summarization, proactive outreach.

> **Note:** GitHub Agentic Workflows (gh-aw) is in technical preview as of 2026. Syntax may change.

---

### Event-Driven

The loop starts when something happens in an external system.

| Event source | Examples |
|-------------|----------|
| Repository activity (gh-aw) | New issue, PR opened, comment added |
| Webhook payload | Stripe payment, Slack message, form submission |
| Message queue | SQS, Kafka, Pub/Sub message arrives |
| File system watch | New file dropped in a folder |
| Database trigger | Row inserted, record updated |

```yaml
# GitHub Agentic Workflows example
on:
  issues:
    types: [opened]
  pull_request:
    types: [opened, synchronize]
  issue_comment:
    types: [created]
```

**Best for:** Real-time response, reactive automation, human-in-the-loop at the trigger point, event-sourced architectures.

---

## Production Hardening

The six-line while loop is the teaching model. Real systems add these layers on top:

### Stopping Conditions

Every loop needs at least one exit path:

- **Success condition**: The goal is met (evaluator approves, task marked complete)
- **Max iterations**: Hard cap on loop turns (e.g., `if turns > 25: stop`)
- **Token/cost budget**: Stop when spend exceeds threshold
- **Timeout**: Wall-clock limit for time-sensitive loops
- **Human escalation**: Pause and ask when confidence is low or action is irreversible

### Loop Detection

Agents can get stuck in cycles. Detect with:
- Hashing recent (action, observation) pairs; abort if the same hash repeats
- Tracking the last N actions; if identical, break out
- Semantic similarity check: if the last 3 observations are near-duplicates, escalate

### Memory Architecture

| Layer | What it stores | Lifespan |
|-------|---------------|---------|
| **In-context** | Current loop's observations and reasoning | One loop run |
| **Episodic (Reflexion-style)** | Verbal reflections from prior runs | Across episodes |
| **Semantic (vector store)** | Embeddings of past results for retrieval | Long-term |
| **Procedural** | Tool definitions, system prompts, schemas | Persistent |

### Context Compaction

Long-running loops fill the context window. Strategies:
- **Summarize**: Compress old observations into a rolling summary
- **Truncate**: Drop oldest turns, keep system prompt and goal
- **RAG**: Store observations in a vector DB; retrieve relevant ones per turn
- **Sliding window**: Keep the last N turns, always include first turn (the goal)

### Multi-Level Loops (L1/L2/L3)

Complex systems nest loops:
- **L1 (inner)**: Single tool call → observe → next call
- **L2 (task)**: Attempt one goal → reflect → retry
- **L3 (outer)**: Plan → delegate to L2 loops → synthesize

---

## Loop Libraries and Frameworks

| Name | Description | Trigger types |
|------|-------------|--------------|
| [Forward-Future Loop Library](https://github.com/Forward-Future/loop-library) | Curated collection of named loop patterns with the "four questions" framework | Manual |
| [awesome-agent-loops](https://github.com/serenakeyitan/awesome-agent-loops) | Community list of agent loop resources and examples | — |
| [LangGraph](https://github.com/langchain-ai/langgraph) | Graph-based orchestration for stateful multi-actor loops; implements ReAct, Plan-and-Execute, Reflexion | Manual, event |
| [AutoGen](https://github.com/microsoft/autogen) | Multi-agent conversation loops with configurable stopping conditions | Manual, event |
| [CrewAI](https://github.com/crewaiinc/crewai) | Role-based multi-agent loops with task delegation | Manual, cron |
| [Temporal](https://github.com/temporalio/temporal) | Durable execution for long-running loops with retry, timeout, and human escalation | Manual, cron, event |
| [n8n](https://github.com/n8n-io/n8n) | Visual workflow loops with 400+ integrations; manual, schedule, and webhook triggers | Manual, cron, event |
| [AgenticFlow](https://docs.agenticflow.ai) | Agent workflow platform; webhook + cron/schedule triggers | Cron, event (webhook) |
| [GitHub Agentic Workflows](https://github.github.com/gh-aw) | GitHub-native agentic loops triggered by repo events, schedules, or manually | Manual, cron, event |

---

## Use Cases and Examples

### Monitoring Loop (Cron)

Runs every hour; checks a data source; takes action if a condition is met.

```python
# Pseudocode
while True:
    data = fetch_latest_metrics()
    analysis = llm.analyze(data, goal="detect anomalies")
    if analysis.anomaly_detected:
        notify_team(analysis.summary)
    sleep_until_next_cron()
```

**Real examples:** Security log monitoring, price tracking, uptime checks, social media listening.

---

### Issue Triage Loop (Event-Driven)

Triggers when a new GitHub issue is opened; classifies, labels, and responds.

```yaml
on:
  issues:
    types: [opened]
```

```python
loop:
  read_issue(event.issue)
  → classify(title, body)  # bug / feature / question
  → apply_label(classification)
  → draft_response(classification, context)
  → post_comment(response)
  → done
```

**Real examples:** Customer support routing, PR review assignment, bug severity triage, spam detection.

---

### Research Loop (Manual + ReAct)

User triggers a deep research task; agent iterates over search and synthesis.

```
Goal: "Write a report on agent loop architectures"

Turn 1: search("agent loop architectures 2024") → 10 results
Turn 2: fetch(url_1) → extract key claims
Turn 3: fetch(url_2) → extract key claims
Turn 4: cross-reference_claims() → identify contradictions
Turn 5: synthesize_report(all_findings) → draft
Turn 6: evaluate(draft) → "Missing section on Reflexion" → continue
Turn 7: fetch(reflexion_paper) → add section
Turn 8: evaluate(draft) → "Complete" → done
```

---

### Code Review Loop (Evaluator-Optimizer)

An agent writes code; a second agent reviews it; loop until approved.

```
Generator:  write function to parse JSON with error handling
Evaluator:  "Missing edge case: empty string input. Score: 7/10."
Generator:  add empty-string check
Evaluator:  "Still missing: None input. Score: 8/10."
Generator:  add None check
Evaluator:  "All cases covered. Score: 10/10. Approved."
```

---

### Multi-Agent Report Loop (Orchestrator-Workers)

Orchestrator decomposes a large task; workers run in parallel; orchestrator synthesizes.

```
Orchestrator: decompose("market analysis for 5 sectors")
→ spawn Worker[sector=tech]   → ReAct loop → findings
→ spawn Worker[sector=health] → ReAct loop → findings
→ spawn Worker[sector=energy] → ReAct loop → findings
→ spawn Worker[sector=finance]→ ReAct loop → findings
→ spawn Worker[sector=retail] → ReAct loop → findings
Orchestrator: synthesize(all_findings) → final report
```

---

### Self-Improving Loop (Reflexion)

Agent attempts a coding challenge, reflects on failure, improves on next attempt.

```
Episode 1: attempt("sort list without built-ins") → wrong output
Reflect:   "I used a comparison that failed on equal elements. Use stable sort."
Episode 2: attempt + reflection context → correct output ✓
```

---

## Academic Papers

| Paper | Pattern | Year | Link |
|-------|---------|------|------|
| *ReAct: Synergizing Reasoning and Acting in Language Models* | ReAct | 2022 (ICLR 2023) | [arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629) |
| *Reflexion: Language Agents with Verbal Reinforcement Learning* | Reflexion | 2023 (NeurIPS 2023) | [arxiv.org/abs/2303.11366](https://arxiv.org/abs/2303.11366) |
| *Plan-and-Solve Prompting* | Plan-and-Execute | 2023 | [arxiv.org/abs/2305.04091](https://arxiv.org/abs/2305.04091) |
| *Tree of Thoughts: Deliberate Problem Solving with Large Language Models* | Tree-of-Thoughts | 2023 | [arxiv.org/abs/2305.10601](https://arxiv.org/abs/2305.10601) |
| *Self-Consistency Improves Chain of Thought Reasoning in Language Models* | Self-consistency | 2022 | [arxiv.org/abs/2203.11171](https://arxiv.org/abs/2203.11171) |
| *ReWOO: Decoupling Reasoning from Observations for Efficient Augmented Language Models* | ReWOO | 2023 | [arxiv.org/abs/2305.18323](https://arxiv.org/abs/2305.18323) |

---

## Further Reading

- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic (Dec 2024). The canonical reference for workflow patterns and the agent/workflow distinction.
- [Forward-Future Loop Library](https://signals.forwardfuture.ai/loop-library/#library) — Curated named loop patterns with the four-questions framework.
- [Planning Agents](https://blog.langchain.com/planning-agents/) — LangChain's comparison of ReAct vs. Plan-and-Execute with benchmarks.
- [What Is the AI Agent Loop?](https://blogs.oracle.com/developers/what-is-the-ai-agent-loop-the-core-architecture-behind-autonomous-ai-systems) — Oracle's accessible introduction to the core loop architecture.
- [Loop Engineering](https://www.requesty.ai/blog/loop-engineering-how-to-build-ai-agent-loops-that-run-themselves) — Practitioner guide on building loops that run themselves.
- [How to Schedule AI Agents](https://agentc2.ai/blog/how-to-schedule-ai-agents-cron-triggers) — Cron triggers and scheduling patterns for production agents.
- [AgenticFlow Trigger Docs](https://docs.agenticflow.ai/workflows/triggers) — Webhook and schedule trigger reference.
- [GitHub Agentic Workflows Triggers](https://github.github.com/gh-aw/reference/triggers/) — Manual, cron, and event trigger reference for gh-aw (technical preview).
- [HuggingFace Agents Course](https://huggingface.co/learn/agents-course) — Free course covering the agent loop from first principles.

---

## Contributing

Contributions welcome. This is a curation, not a collection — only add items you can personally recommend and that are actively maintained. See [contributing.md](contributing.md).

Items that don't belong here: archived repos, undocumented tools, tutorials that duplicate existing links, promotional content without substance.

---

*Research basis: 23 primary and secondary sources, 109 claims extracted, 25 adversarially verified (23 confirmed, 2 refuted). See [research-notes.md](research-notes.md) for methodology and open questions.*
