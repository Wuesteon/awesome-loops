# Evaluator-Optimizer

The **Evaluator-Optimizer** pattern is a loop where one LLM call generates a response and another evaluates it — repeating until quality is sufficient or a budget is exhausted. It's Anthropic's name for the generate-critique-refine cycle.

## Core Idea

Separate the generation role from the evaluation role. The generator produces; the evaluator judges. The evaluation (with its reasons) is fed back to the generator as context for the next attempt. This mirrors the human edit cycle: write, get feedback, revise.

## Loop Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                  EVALUATOR-OPTIMIZER LOOP                        │
│                                                                  │
│  ┌──────────────┐         ┌──────────────────────────────────┐   │
│  │   GENERATOR  │ ──────► │           EVALUATOR              │   │
│  │              │         │                                  │   │
│  │ Draft output │         │  Score: 6/10                     │   │
│  │              │         │  Issues: missing edge case X,    │   │
│  │              │         │  section Y is unclear            │   │
│  └──────────────┘         └──────────────────────────────────┘   │
│         ▲                              │                         │
│         │          feedback            │                         │
│         └──────────────────────────────┘                         │
│                                                                  │
│  Loop until: score >= threshold OR turns >= max_iterations       │
└──────────────────────────────────────────────────────────────────┘
```

## Minimal Implementation

```python
def evaluator_optimizer(
    task: str,
    generator_llm,
    evaluator_llm,
    threshold: float = 0.85,
    max_iterations: int = 5
):
    draft = generator_llm.call(task)
    
    for i in range(max_iterations):
        evaluation = evaluator_llm.call(
            f"Evaluate this output for the task: {task}\n\n"
            f"Output:\n{draft}\n\n"
            f"Score from 0.0 to 1.0 and list specific issues."
        )
        
        if evaluation.score >= threshold:
            return draft  # Quality sufficient
        
        # Feed evaluation back to generator
        draft = generator_llm.call(
            f"Task: {task}\n\n"
            f"Previous attempt:\n{draft}\n\n"
            f"Evaluator feedback (score {evaluation.score}):\n{evaluation.issues}\n\n"
            f"Please improve based on this feedback."
        )
    
    return draft  # Best effort after max iterations
```

## Example Trace

```
Task: "Write a Python function to parse CSV with proper error handling"

Turn 1 — Generator:
  def parse_csv(filepath):
      with open(filepath) as f:
          return list(csv.reader(f))

Turn 1 — Evaluator:
  Score: 0.45
  Issues:
  - No error handling for FileNotFoundError
  - No error handling for malformed rows
  - No type hints
  - No docstring
  - Returns nested lists, not dicts

Turn 2 — Generator (with feedback):
  def parse_csv(filepath: str) -> list[dict]:
      """Parse a CSV file and return a list of row dicts."""
      try:
          with open(filepath, newline='', encoding='utf-8') as f:
              reader = csv.DictReader(f)
              return [row for row in reader]
      except FileNotFoundError:
          raise FileNotFoundError(f"CSV file not found: {filepath}")
      except csv.Error as e:
          raise ValueError(f"Malformed CSV: {e}")

Turn 2 — Evaluator:
  Score: 0.92 — Approved ✓
```

## Evaluator Prompt Patterns

### Binary evaluator (pass/fail)
```
Does this output correctly complete the task?
Task: {task}
Output: {output}

Answer YES or NO, then explain why.
```

### Scored evaluator (0-10 or 0.0-1.0)
```
Rate this output on a scale of 0 to 10.
Task: {task}
Output: {output}

Criteria:
- Correctness (0-4): Does it solve the task correctly?
- Completeness (0-3): Are all requirements covered?
- Quality (0-3): Is it well-written/structured?

Return JSON: {"score": N, "issues": ["issue1", "issue2"], "approved": bool}
```

### Constitutional evaluator (against principles)
```
Review this output against these principles:
1. No hallucinated facts
2. No unsafe content
3. Follows the requested format
4. Answers the actual question asked

Output: {output}

List any violations. If none, output "APPROVED".
```

## Strengths

- **Separates concerns**: Generator optimizes for output quality; evaluator optimizes for correctness and coverage
- **Interpretable feedback**: Evaluation reasons are readable and debuggable
- **Works with same-model**: Generator and evaluator can be the same model with different system prompts
- **Quality ceiling**: The loop can only improve to the level the evaluator can detect
- **Analogous to human process**: Mirrors how humans review and revise work

## Limitations

- **Cost doubles per iteration**: Every turn requires two LLM calls
- **Evaluator bias**: The evaluator may have blind spots that the generator can exploit without actually improving
- **Score gaming**: If the generator learns to optimize for evaluator scores, it may diverge from true quality
- **Feedback exhaustion**: Diminishing returns as the draft approaches the evaluator's quality ceiling
- **Same model, same blind spots**: If generator and evaluator are the same model, they share the same failure modes

## When to Use

Evaluator-Optimizer works best when:
- Quality criteria can be articulated as a rubric or set of questions
- The evaluation step is significantly cheaper than re-doing the task from scratch
- Multiple dimensions of quality matter and need separate tracking
- The improvement ceiling from iteration is worth the added latency and cost

Consider other patterns when:
- The task is binary (correct or not) — a single ReAct loop with tests is simpler
- The evaluation cost is comparable to the generation cost
- Speed is critical and iteration latency is unacceptable

## Variants

| Variant | Description |
|---------|-------------|
| **Constitutional AI** | Evaluator checks against a list of principles; generator revises to comply |
| **Self-evaluation** | Same LLM call generates draft and critique (no separate evaluator) |
| **Peer review** | Multiple evaluators vote; majority rules on whether to continue |
| **Tournament** | Multiple generators produce candidates; evaluator picks the best |
| **Adversarial** | Evaluator is specifically prompted to find flaws, not score quality |

## Integration with Other Patterns

Evaluator-Optimizer combines naturally with:
- **Plan-and-Execute**: Evaluate and optimize each plan step's output before moving to the next
- **Reflexion**: After a failed episode, use an evaluator to produce the reflection
- **Orchestrator-Workers**: The orchestrator evaluates worker outputs; sub-standard ones are re-run

## Source

Anthropic, *Building Effective Agents* (December 2024):
[anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents)

> "In the evaluator-optimizer workflow, one LLM call generates a response while another provides evaluation and feedback in a loop."

Analogized to "iterative writing processes where a human writer and editor work together."
