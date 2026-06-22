# Reflexion: Verbal Self-Reinforcement

**Reflexion** is an agent loop pattern that enables language agents to improve across multiple episodes through linguistic feedback — without updating model weights. Instead of gradient descent, it uses verbal reflection stored in episodic memory.

## Core Idea

When an agent fails (or partially succeeds), have it write a verbal reflection — a natural language summary of what went wrong and what to try differently. Store that reflection in memory. On the next attempt, prepend the reflection to the prompt. The agent improves by reading its own post-mortems.

## Loop Structure

```
┌──────────────────────────────────────────────────────────┐
│                    REFLEXION LOOP                        │
│                                                          │
│  Episode N:                                              │
│    context = [goal, memory.reflections, episode_history] │
│    result  = inner_loop(context)                         │
│    feedback = environment.evaluate(result)               │
│                                                          │
│  If not success:                                         │
│    reflection = reflect_llm.call(                        │
│        goal, result, feedback,                           │
│        "What went wrong? What should I try next?"        │
│    )                                                     │
│    memory.add(reflection)                                │
│                                                          │
│  Episode N+1:                                            │
│    context includes previous reflections                 │
│    → agent tries again with improved strategy            │
└──────────────────────────────────────────────────────────┘
```

## Key Components

| Component | Role |
|-----------|------|
| **Actor** | The inner agent loop (often ReAct) that attempts the task |
| **Evaluator** | Scores or provides binary feedback on the attempt |
| **Reflector** | An LLM call that converts feedback into a verbal reflection |
| **Episodic memory** | A buffer that stores reflections; prepended to future episodes |

## Memory Buffer Pattern

```python
class ReflexionMemory:
    def __init__(self, max_reflections=5):
        self.reflections = []
        self.max = max_reflections
    
    def add(self, reflection: str):
        self.reflections.append(reflection)
        if len(self.reflections) > self.max:
            self.reflections.pop(0)  # Keep most recent
    
    def as_context(self) -> str:
        if not self.reflections:
            return ""
        items = "\n".join(f"- {r}" for r in self.reflections)
        return f"Your past reflections:\n{items}\n"
```

## Minimal Implementation

```python
def reflexion_loop(goal: str, inner_loop, evaluator, reflector, max_episodes=5):
    memory = ReflexionMemory()
    
    for episode in range(max_episodes):
        # Build context including past reflections
        context = memory.as_context() + goal
        
        # Inner agent loop (e.g. ReAct)
        result = inner_loop(context)
        
        # Evaluate the result
        feedback = evaluator(goal, result)
        if feedback.success:
            return result
        
        # Reflect on failure
        reflection = reflector(
            goal=goal,
            result=result,
            feedback=feedback.message,
            prompt="What went wrong? What should you try differently next episode?"
        )
        memory.add(reflection)
        print(f"Episode {episode+1} failed. Reflection: {reflection}")
    
    return result  # Best effort after max episodes
```

## Example Trace

```
Episode 1:
  Goal: "Write a Python function to reverse a linked list"
  Result: [code with a bug in the None-check]
  Feedback: "Test case 3 failed: empty list raises AttributeError"
  Reflection: "I forgot to handle the edge case where head is None.
               Next time, add a guard clause at the start of the function."

Episode 2:
  Context includes: "I forgot to handle the edge case where head is None..."
  Result: [code with None-check added]
  Feedback: "All 5 test cases passed."
  → Success!
```

## What Makes Reflexion Different from Retry

A naive retry just runs the same loop again with the same context. Reflexion adds:
1. A dedicated **reflection step** that explicitly reasons about failure modes
2. **Persistent memory** that carries the reflection forward
3. **Verbal encoding** that the LLM can directly reason about

The reflection isn't a log entry — it's a prompt to the model on the next episode.

## Converting Feedback to Language

The key mechanism: binary or scalar environment feedback → verbal summary.

```python
REFLECT_PROMPT = """
You attempted the following task: {goal}

Your attempt produced: {result}

The evaluator said: {feedback}

In 2-3 sentences, explain:
1. What specifically went wrong
2. What you would do differently on the next attempt
"""
```

## Strengths

- **No weight updates**: Works with any LLM, no fine-tuning required
- **Interpretable**: Reflections are human-readable; you can inspect what the agent "learned"
- **General**: Works for code, reasoning, QA, decision-making tasks
- **Episode-efficient**: One reflection per episode, not one per turn
- **Stackable**: Combine with any inner loop pattern (ReAct, Plan-and-Execute)

## Limitations

- **Context window grows**: Each reflection adds to the context; bounded by `max_reflections`
- **Weaker than true exploration**: Verbal reinforcement doesn't explore the state space; it only refines within the model's existing capabilities
- **Reflection quality depends on feedback quality**: Vague feedback → vague reflections → marginal improvement
- **Not a substitute for diverse training data**: If the model fundamentally can't do something, reflecting won't fix it

## When to Use

Reflexion is well-suited for:
- Tasks with clear success/failure signals (unit tests, benchmarks, correctness checks)
- Tasks where the same agent will attempt the same goal multiple times
- Iterative refinement problems (write → test → improve → test)
- Scenarios where you want to avoid fine-tuning but need improvement over time

Don't use Reflexion for:
- One-shot tasks (no opportunity for multiple episodes)
- Tasks without evaluable success criteria
- Tasks where the failures are due to capability limits, not strategy errors

## Source

Shinn, N. et al. (2023). *Reflexion: Language Agents with Verbal Reinforcement Learning*. NeurIPS 2023.
- Paper: [arxiv.org/abs/2303.11366](https://arxiv.org/abs/2303.11366)
- Key claim (verbatim): "a novel framework to reinforce language agents not by updating weights, but instead through linguistic feedback. Reflexion agents verbally reflect on task feedback signals, then maintain their own reflective text in an episodic memory buffer to induce better decision-making in subsequent trials."
