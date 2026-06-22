# ReAct: Reasoning + Acting

**ReAct** (Reasoning + Acting) is the foundational agent loop pattern. It interleaves reasoning traces and tool-use actions in a single iterative loop, grounding the model's reasoning in real-world observations.

## Core Idea

Instead of asking the model to reason about a task in isolation (chain-of-thought), ReAct asks it to alternate between:
1. **Thought**: Write out a reasoning trace
2. **Action**: Call a tool
3. **Observation**: Receive the tool's result

And repeat until the task is complete.

## Loop Structure

```
┌─────────────────────────────────────┐
│           ReAct Iteration           │
│                                     │
│  Thought: "I need to find X"        │
│       ↓                             │
│  Action: search("query for X")      │
│       ↓                             │
│  Observation: "Search returned..."  │
│       ↓                             │
│  Thought: "Based on that, I need Y" │
│       ↓                             │
│  Action: fetch("url for Y")         │
│       ↓                             │
│  Observation: "Page contains..."    │
│       ↓                             │
│  Thought: "I have enough to answer" │
│       ↓                             │
│  Action: finish("final answer")     │
└─────────────────────────────────────┘
```

## Minimal Implementation

```python
def react_loop(goal: str, tools: dict, llm, max_turns=20):
    history = [
        {"role": "system", "content": REACT_SYSTEM_PROMPT},
        {"role": "user", "content": goal}
    ]
    
    for turn in range(max_turns):
        response = llm.call(history)
        
        if response.action == "finish":
            return response.answer
        
        # Execute the tool
        tool_fn = tools[response.action]
        observation = tool_fn(**response.action_input)
        
        # Feed observation back
        history.append({"role": "assistant", "content": str(response)})
        history.append({"role": "user", "content": f"Observation: {observation}"})
    
    raise MaxIterationsError(f"Did not finish in {max_turns} turns")
```

## System Prompt Pattern

```
You run in a loop of Thought, Action, Observation.
At the end of the loop you output an Answer.

Use Thought to describe your reasoning about the question.
Use Action to run one of these tools:
  - search(query): Search the web
  - fetch(url): Get the content of a URL
  - calculate(expression): Evaluate a math expression
  - finish(answer): Return the final answer

Format:
Thought: [your reasoning]
Action: tool_name
Action Input: {"param": "value"}

Wait for the Observation before continuing.
```

## Example Trace

```
User: What is the population of the capital of France?

Thought: I need to find France's capital first.
Action: search
Action Input: {"query": "capital of France"}
Observation: Paris is the capital of France.

Thought: Now I need the population of Paris.
Action: search
Action Input: {"query": "population of Paris 2024"}
Observation: The population of Paris proper is approximately 2.1 million (2024).

Thought: I have the answer.
Action: finish
Action Input: {"answer": "Paris is the capital of France, with a population of approximately 2.1 million."}
```

## Strengths

- **Grounds reasoning in facts**: Each thought follows from a real observation, not a predicted one
- **Reduces hallucination**: Model reasons *after* seeing tool results, not before
- **Handles exceptions mid-task**: If a tool fails, the model sees the error and adapts
- **Transparent**: The thought trace makes reasoning inspectable
- **Simple**: The loop structure is minimal and easy to implement

## Limitations

- **One tool call per iteration**: Requires a separate LLM call for each action (no parallel tool use in standard ReAct)
- **Plans one step at a time**: Doesn't reason about the whole task upfront; can miss more efficient paths
- **Sub-optimal trajectories**: Sequential one-step planning can miss shortcuts that upfront planning would catch
- **Context grows linearly**: Each turn adds thought + action + observation to the context window
- **Can loop without progress**: Without good stopping conditions, may get stuck

## When to Use

ReAct is the right default for:
- Tasks that require tool use and where the path is unknown upfront
- Knowledge-intensive QA (search, lookup, verify)
- Code execution loops (write → run → observe error → fix → run)
- Tasks where the model needs to respond to unexpected results

Consider Plan-and-Execute instead when:
- The task is large and parallelizable
- You need upfront reasoning about the whole task
- You want to reduce context growth per sub-task

## Variants

| Variant | Difference |
|---------|-----------|
| **ReAct + CoT** | Add a chain-of-thought warmup before the loop |
| **ReAct + memory** | Retrieve relevant past observations instead of loading all history |
| **ReAct + parallel tools** | Allow multiple tool calls per iteration (extension to standard) |
| **ReAct + reflection** | Add a reflection step after failures (combines with Reflexion pattern) |

## Source

Yao, S. et al. (2022). *ReAct: Synergizing Reasoning and Acting in Language Models*. ICLR 2023.
- Paper: [arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629)
- Key result: ReAct reduces the false-positive hallucination rate compared to chain-of-thought-only on knowledge-intensive tasks. Note: ReAct slightly lags pure CoT on HotpotQA EM (27.4 vs 29.4) but outperforms on hallucination metrics.
