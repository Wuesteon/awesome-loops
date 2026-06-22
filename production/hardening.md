# Production Hardening for Agent Loops

The six-line while loop is the teaching model. Every loop that runs in production needs additional layers. This document covers the most important ones.

## The Production Gap

```
Teaching model:           Production reality:
───────────────           ───────────────────
while not done:           while not done:
  action = llm(ctx)         check_budget()
  result = run(action)      action = llm(compact(ctx))
  ctx.append(result)        result = run_with_retry(action)
  done = check(result)      ctx.append(result)
                            detect_loops(ctx)
                            done = multi_condition_check(result, ctx)
                            if needs_human(result):
                              pause_and_escalate()
```

The core is the same. The difference is in the reliability, safety, and cost layers around it.

---

## 1. Stopping Conditions

Every loop needs at least one exit condition beyond "the model says done."

### Layer 1: Success detection
```python
def is_success(result, goal) -> bool:
    # Option A: deterministic check (fastest, most reliable)
    return run_tests(result) == "all_pass"
    
    # Option B: LLM evaluator (flexible, slower)
    score = evaluator_llm.call(f"Does this satisfy the goal?\nGoal: {goal}\nResult: {result}")
    return score.approved
    
    # Option C: structural check (for formatted outputs)
    return validate_schema(result, expected_schema)
```

### Layer 2: Iteration budget
```python
MAX_TURNS = 25  # Set based on task complexity
COST_BUDGET = 0.50  # USD per task run

if turn_count >= MAX_TURNS:
    log_warning("Hit iteration limit without success")
    return best_effort_result

if accumulated_cost >= COST_BUDGET:
    log_warning("Hit cost budget")
    return best_effort_result
```

### Layer 3: Wall-clock timeout
```python
import signal

def timeout_handler(signum, frame):
    raise TimeoutError("Loop exceeded wall-clock limit")

signal.signal(signal.SIGALRM, timeout_handler)
signal.alarm(300)  # 5-minute limit

try:
    result = agent_loop(goal)
finally:
    signal.alarm(0)
```

### Layer 4: Human escalation
```python
ESCALATION_CONDITIONS = [
    lambda r: r.confidence < 0.3,          # Low confidence
    lambda r: r.action_type == "irreversible",  # Destructive action
    lambda r: r.cost_estimate > 100,        # High-cost action
    lambda r: r.affects_production,         # Production impact
]

if any(cond(result) for cond in ESCALATION_CONDITIONS):
    pause_loop()
    ask_human(result.summary)
    # Resume or abort based on human decision
```

---

## 2. Loop Detection

Agents can get stuck cycling through the same (action, observation) pairs. Detect and break out.

### Hash-based detection
```python
from collections import deque
import hashlib

recent_hashes = deque(maxlen=10)

def check_loop(action, observation):
    fingerprint = hashlib.md5(
        f"{action}{observation}".encode()
    ).hexdigest()
    
    if fingerprint in recent_hashes:
        raise LoopDetectedError(
            f"Same action+observation seen before: {action}"
        )
    
    recent_hashes.append(fingerprint)
```

### Semantic similarity detection
```python
def is_semantically_stuck(observations: list[str], threshold=0.95) -> bool:
    if len(observations) < 3:
        return False
    
    recent = observations[-3:]
    embeddings = embed(recent)
    
    # If all recent observations are near-identical
    similarities = [cosine_sim(embeddings[0], embeddings[i]) for i in range(1, 3)]
    return all(s >= threshold for s in similarities)
```

### Action repetition
```python
action_counts = Counter()

def check_repetition(action):
    action_counts[action.type] += 1
    
    if action_counts[action.type] > 5:
        raise RepetitionError(
            f"Action '{action.type}' called {action_counts[action.type]} times"
        )
```

---

## 3. Context Management

Long loops fill the context window. Manage it explicitly.

### Rolling summary
```python
CONTEXT_LIMIT = 8000  # tokens
SUMMARY_THRESHOLD = 6000  # trigger compaction at this threshold

def manage_context(history: list[dict]) -> list[dict]:
    current_tokens = count_tokens(history)
    
    if current_tokens < SUMMARY_THRESHOLD:
        return history
    
    # Summarize older turns, keep system prompt and recent turns
    system = history[0]  # Always keep system prompt
    recent = history[-4:]  # Keep last 4 turns verbatim
    old = history[1:-4]   # Summarize everything in between
    
    summary = summarizer_llm.call(
        "Summarize these agent loop turns, preserving key findings and decisions:\n"
        + format_turns(old)
    )
    
    return [
        system,
        {"role": "assistant", "content": f"[Summary of prior turns]: {summary}"},
        *recent
    ]
```

### Selective retrieval (RAG over history)
```python
class LoopMemory:
    def __init__(self, vector_store):
        self.store = vector_store
        self.recent = deque(maxlen=5)  # Always include 5 most recent turns
    
    def add(self, turn: dict):
        self.store.insert(turn)
        self.recent.append(turn)
    
    def retrieve_for(self, query: str, k=3) -> list[dict]:
        # Get k semantically relevant turns + 5 most recent
        relevant = self.store.search(query, k=k)
        return list(self.recent) + [r for r in relevant if r not in self.recent]
```

### Observation truncation
```python
MAX_OBSERVATION_TOKENS = 2000

def truncate_observation(obs: str) -> str:
    tokens = count_tokens(obs)
    if tokens <= MAX_OBSERVATION_TOKENS:
        return obs
    
    # Keep beginning and end; summarize middle
    beginning = obs[:500]
    end = obs[-500:]
    return (
        f"{beginning}\n"
        f"[...{tokens - 1000} tokens truncated...]\n"
        f"{end}"
    )
```

---

## 4. Tool Execution Hardening

Tool calls are where loops interact with the real world. Harden them.

### Retry with backoff
```python
import time

def run_with_retry(tool_fn, *args, max_retries=3, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return tool_fn(*args)
        except TemporaryError as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)
            time.sleep(delay)
        except PermanentError:
            raise  # Don't retry permanent failures
```

### Input validation
```python
def validate_tool_call(tool_name: str, params: dict) -> None:
    schema = TOOL_SCHEMAS[tool_name]
    
    # Validate against JSON Schema
    jsonschema.validate(params, schema)
    
    # Additional safety checks
    if tool_name == "shell_command":
        if any(dangerous in params["command"] for dangerous in ["rm -rf", "DROP TABLE"]):
            raise UnsafeToolCallError(f"Blocked dangerous command: {params['command']}")
```

### Timeout per tool call
```python
async def run_tool_with_timeout(tool_fn, params, timeout_seconds=30):
    try:
        return await asyncio.wait_for(
            asyncio.coroutine(tool_fn)(**params),
            timeout=timeout_seconds
        )
    except asyncio.TimeoutError:
        return {"error": f"Tool timed out after {timeout_seconds}s"}
```

---

## 5. Cost Tracking

Loops can be expensive. Track costs at every turn.

```python
class CostTracker:
    def __init__(self, budget_usd: float):
        self.budget = budget_usd
        self.spent = 0.0
        self.turns = 0
    
    def record_llm_call(self, input_tokens: int, output_tokens: int, model: str):
        cost = calculate_cost(model, input_tokens, output_tokens)
        self.spent += cost
        self.turns += 1
        
        if self.spent >= self.budget:
            raise BudgetExceededError(
                f"Loop spent ${self.spent:.4f}, budget was ${self.budget:.4f}"
            )
    
    def summary(self) -> dict:
        return {
            "turns": self.turns,
            "spent_usd": round(self.spent, 4),
            "budget_usd": self.budget,
            "remaining_usd": round(self.budget - self.spent, 4)
        }
```

---

## 6. Observability

You can't debug what you can't see. Trace every loop.

### Structured loop logging
```python
import json
import uuid
from datetime import datetime, UTC

class LoopTracer:
    def __init__(self, goal: str):
        self.run_id = str(uuid.uuid4())
        self.goal = goal
        self.turns = []
        self.start_time = datetime.now(UTC)
    
    def record_turn(self, turn_num, action, observation, cost):
        self.turns.append({
            "turn": turn_num,
            "timestamp": datetime.now(UTC).isoformat(),
            "action": action,
            "observation_preview": observation[:200],
            "cost_usd": cost,
        })
    
    def export(self) -> dict:
        return {
            "run_id": self.run_id,
            "goal": self.goal,
            "start": self.start_time.isoformat(),
            "end": datetime.now(UTC).isoformat(),
            "turn_count": len(self.turns),
            "turns": self.turns,
        }
```

### Key metrics to track

| Metric | Why it matters |
|--------|---------------|
| **Turns per task** | High = possible loop or over-complex task |
| **Cost per task** | Baseline for budget planning |
| **Success rate** | Track degradation over time |
| **Time to first tool call** | LLM reasoning latency |
| **Tool error rate by tool** | Identifies flaky integrations |
| **Human escalation rate** | Calibrate confidence thresholds |
| **Loop detection rate** | How often agents get stuck |

---

## 7. Human-in-the-Loop Integration

When to pause and when to proceed autonomously.

### Confidence-based escalation
```python
AUTONOMOUS_ACTIONS = {"search", "fetch", "read_file", "calculate"}
REQUIRES_CONFIRMATION = {"write_file", "send_email", "delete", "deploy"}
ALWAYS_ESCALATE = {"drop_database", "rm_rf", "send_to_all_users"}

def should_escalate(action) -> bool:
    if action.type in ALWAYS_ESCALATE:
        return True
    if action.type in REQUIRES_CONFIRMATION:
        return True
    if action.confidence < 0.4:
        return True
    if action.estimated_cost > 100:
        return True
    return False
```

### Async human checkpoint
```python
async def human_checkpoint(action, timeout_seconds=300) -> bool:
    approval_request = await create_approval_request(
        action=action.describe(),
        context=action.context,
        expires_at=now() + timeout_seconds
    )
    
    # Notify human (Slack, email, UI notification)
    await notify_human(approval_request.url)
    
    # Wait for response
    response = await wait_for_approval(approval_request.id, timeout_seconds)
    
    if response is None:  # Timeout
        return False  # Default deny on timeout
    
    return response.approved
```

---

## Quick Reference Checklist

Before shipping a production loop, verify:

- [ ] Has a deterministic stopping condition (not just "model says done")
- [ ] Has a max-iterations guard
- [ ] Has a cost/token budget
- [ ] Has loop detection (hash or semantic)
- [ ] Handles tool timeouts and retries
- [ ] Manages context window growth (summary or truncation)
- [ ] Logs every turn with run ID for tracing
- [ ] Escalates to human for irreversible or high-cost actions
- [ ] Has been load-tested for parallelism if concurrent loops are expected
- [ ] Has monitoring and alerting on cost, error rate, and stuck-loop rate
