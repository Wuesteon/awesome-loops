# Example: Coding Agent Loop (Write → Test → Fix → Repeat)

A coding agent loop takes a programming task, writes code, runs tests or a linter, reads the error output, and iterates until tests pass or a budget is exhausted. This is one of the most widely deployed agent loop patterns in production.

## Real Implementations

| Repo | Framework | SWE-bench score | Stars |
|------|-----------|----------------|-------|
| [langchain-ai/react-agent](https://github.com/langchain-ai/react-agent) | LangGraph (Python) | — | reference template |
| [SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) | Plain Python (~100 lines) | 74% SWE-bench Verified | minimal reference |
| Anthropic SWE-bench agent | Anthropic API directly | state-of-art (2024) | described in Building Effective Agents |

## Architecture

```
MANUAL TRIGGER: user provides task description + codebase
         │
         ▼
┌──────────────────────────────────────────────────────┐
│               CODING AGENT LOOP                       │
│                                                       │
│  State: {code, test_results, error_log, turn_count}   │
│                                                       │
│  call_model ──────────────────────────────────────►  │
│       │  LLM reads state, picks next action           │
│       │  (write_file / run_tests / read_file / done)  │
│       ▼                                               │
│  run_tools                                            │
│       │  Execute chosen tool; capture stdout/stderr   │
│       │                                               │
│       ▼                                               │
│  has tool_calls?                                      │
│  ├─ yes → back to call_model                          │
│  └─ no  → END (agent emitted final answer)            │
└──────────────────────────────────────────────────────┘
```

## LangGraph Implementation (react-agent pattern)

The canonical two-node LangGraph loop from [langchain-ai/react-agent](https://github.com/langchain-ai/react-agent):

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode

# Two nodes: the LLM and the tool executor
builder = StateGraph(AgentState)
builder.add_node("call_model", call_model)
builder.add_node("run_tools", ToolNode(tools))

builder.set_entry_point("call_model")

# Conditional edge: loop back if there are tool calls, else end
builder.add_conditional_edges(
    "call_model",
    lambda state: "run_tools" if state["messages"][-1].tool_calls else END,
)

# Tool results always go back to the model
builder.add_edge("run_tools", "call_model")

graph = builder.compile()
```

**Stopping condition**: The loop ends when the LLM's last message contains no `tool_calls` — meaning the model decided it was done and emitted a final answer rather than calling another tool.

## Mini-SWE-Agent: The Minimal Version

[mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) implements a coding agent in ~100 lines of Python. It achieves 74% on SWE-bench Verified — competitive with much larger frameworks — demonstrating that the core loop is simple:

```python
# Simplified from mini-swe-agent
MAX_TURNS = 50

def run_coding_agent(task: str, repo_path: str):
    history = [{"role": "user", "content": task}]
    
    for turn in range(MAX_TURNS):
        response = llm.call(history, tools=CODING_TOOLS)
        history.append(response)
        
        # If no tool calls, agent is done
        if not response.tool_calls:
            return response.content
        
        # Execute each tool call and feed result back
        for tool_call in response.tool_calls:
            result = execute_tool(tool_call, cwd=repo_path)
            history.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result
            })
    
    return "Reached max turns without completing task"

CODING_TOOLS = [
    "bash",        # Run any shell command (tests, linter, etc.)
    "read_file",   # Read source files
    "write_file",  # Write/overwrite files
    "list_dir",    # Explore the codebase
]
```

## Anthropic's SWE-bench Approach

Anthropic's coding agent (described in *Building Effective Agents*) adds one key loop mechanic: it writes its own reproduction script first, then iterates against that script's output rather than the full test suite. This means:

1. Read the bug report
2. **Write a minimal reproduction script** (not the fix — the reproduction)
3. Run the reproduction script → observe failure
4. Write/edit the fix
5. Re-run reproduction script → observe success or new error
6. Repeat from step 4 until reproduction script passes
7. Run full test suite to confirm no regressions

The inner loop is tight: model → bash tool → observe → model. The reproduction script acts as a fast, focused oracle.

## Tools the Coding Agent Uses

```python
CODING_TOOLS = {
    "bash": lambda cmd: subprocess.run(
        cmd, shell=True, capture_output=True, text=True, timeout=60
    ),
    "read_file": lambda path: open(path).read(),
    "write_file": lambda path, content: open(path, 'w').write(content),
    "list_dir": lambda path: os.listdir(path),
    "search_code": lambda pattern, path: subprocess.run(
        ["grep", "-r", pattern, path], capture_output=True, text=True
    ).stdout,
}
```

**Critical**: Give the agent a `bash` tool rather than separate `run_tests`, `run_linter`, `run_command` tools. A single bash tool is more flexible and mirrors how a developer actually works. The agent will figure out how to run the tests from the `Makefile` or `package.json`.

## Stopping Conditions

| Condition | Mechanism |
|-----------|-----------|
| Tests pass | Model observes clean test output → emits final answer (no tool calls) |
| Max turns | Hard cap (typically 25-50 for coding tasks) |
| Model gives up | Model emits a message saying it can't solve the task |
| Syntax error loop | Loop detector: if same file written 3 times with same error, escalate |

## Self-Correcting Code Generation (LangGraph variant)

From [learnopencv.com: LangGraph Self-Correcting Agent](https://learnopencv.com/langgraph-self-correcting-agent-code-generation/):

```python
from langgraph.graph import StateGraph, END

class CodeState(TypedDict):
    task: str
    code: str
    test_result: str
    error: str
    retry_count: int

def generate_code(state: CodeState) -> CodeState:
    code = llm.call(f"Write code for: {state['task']}\nError to fix: {state.get('error', 'none')}")
    return {**state, "code": code, "retry_count": state["retry_count"] + 1}

def run_tests(state: CodeState) -> CodeState:
    result = subprocess.run(["python", "-c", state["code"]], capture_output=True, text=True)
    return {**state, "test_result": result.stdout, "error": result.stderr}

def should_retry(state: CodeState) -> str:
    if state["error"] and state["retry_count"] < 3:
        return "generate_code"  # Loop back
    return END

builder = StateGraph(CodeState)
builder.add_node("generate_code", generate_code)
builder.add_node("run_tests", run_tests)
builder.set_entry_point("generate_code")
builder.add_edge("generate_code", "run_tests")
builder.add_conditional_edges("run_tests", should_retry)
graph = builder.compile()
```

## AutoGen Version

AutoGen requires explicit termination conditions; coding loops use `MaxMessageTermination` as a safety net alongside success detection:

```python
from autogen_agentchat.agents import AssistantAgent, CodeExecutorAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import MaxMessageTermination, TextMentionTermination

coder = AssistantAgent("coder", model_client=model)
executor = CodeExecutorAgent("executor", code_executor=LocalCommandLineCodeExecutor())

# Stop when agent says DONE or after 20 messages
termination = TextMentionTermination("DONE") | MaxMessageTermination(20)

team = RoundRobinGroupChat([coder, executor], termination_condition=termination)
result = await team.run(task="Fix the failing test in auth.py")
```

## Known Pitfalls

**Infinite fix loops**: The agent writes a fix, runs tests, sees a different error, writes a different fix — cycling without converging. Mitigate with loop detection on (file_content, error_message) pairs.

**Context overflow**: Long test output fills the context window. Truncate test output to the first N lines of errors, not the full output.

**Wrong test runner**: Agent may guess `pytest` when the project uses `jest`. Give it the test command explicitly, or let it discover it from `package.json`/`Makefile`.

**Overwriting working code**: Agent may "fix" a passing file, breaking it. Always run a baseline test first and include the baseline results in the initial context.

**Sandboxing**: The `bash` tool runs real commands. Always sandbox in a container or VM. Never give a coding agent access to production credentials.

## Sources

- [github.com/langchain-ai/react-agent](https://github.com/langchain-ai/react-agent) — canonical two-node LangGraph ReAct template
- [github.com/SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) — ~100-line coding agent, 74% SWE-bench Verified
- [anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic SWE-bench approach
- [learnopencv.com/langgraph-self-correcting-agent-code-generation](https://learnopencv.com/langgraph-self-correcting-agent-code-generation/) — self-correcting LangGraph variant
- [microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/termination.html](https://microsoft.github.io/autogen/stable//user-guide/agentchat-user-guide/tutorial/termination.html) — AutoGen termination conditions
