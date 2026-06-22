# Example: Research Loop (Manual + ReAct)

A research loop accepts an open-ended research question, fans out across multiple sources, cross-references findings, and synthesizes a structured report. This is ReAct in its most natural habitat: an open-ended task where the path depends entirely on what the agent finds.

## Use Cases

- Deep-dive research reports
- Due diligence on companies, people, or technologies
- Competitive intelligence
- Academic literature review
- Fact-checking and claim verification

## Architecture

```
MANUAL TRIGGER: user provides research question
         │
         ▼
┌──────────────────────────────────────────────────────┐
│                   RESEARCH LOOP (ReAct)               │
│                                                       │
│  Goal: answer the research question with citations    │
│                                                       │
│  Turn 1-3: broad search to identify key sources      │
│  Turn 4-8: fetch and extract from top sources        │
│  Turn 9-12: cross-reference claims, find gaps        │
│  Turn 13-15: targeted searches to fill gaps          │
│  Turn 16+: synthesize findings into structured report │
│                                                       │
│  Exit when: evaluator approves report quality        │
└──────────────────────────────────────────────────────┘
```

## Implementation

### Single-agent research loop

```python
from anthropic import Anthropic

client = Anthropic()

RESEARCH_SYSTEM_PROMPT = """
You are a research agent. Your task is to thoroughly research a topic and 
produce a well-cited report.

You have access to these tools:
- search(query): Web search, returns top results with snippets
- fetch(url): Fetch the full content of a URL
- extract_claims(text): Extract key factual claims from text
- note(key, value): Store a finding in your research notes
- get_notes(): Retrieve all stored notes
- finish(report): Submit your final report

Research methodology:
1. Start with broad searches to map the landscape
2. Identify the most authoritative sources
3. Fetch and extract claims from key sources
4. Cross-reference claims across sources
5. Fill gaps with targeted searches
6. Synthesize into a structured report with citations

Use finish() only when you have sufficient coverage across multiple sources.
"""

EVALUATOR_PROMPT = """
Evaluate this research report on a scale of 0-10.

Research question: {question}
Report: {report}

Check:
- Coverage: Are key aspects of the question addressed? (0-3)
- Evidence: Are claims backed by cited sources? (0-3)  
- Balance: Are different perspectives included? (0-2)
- Clarity: Is it well-structured and readable? (0-2)

Return JSON: {{"score": N, "approved": bool, "gaps": ["gap1", "gap2"]}}
Approve if score >= 8.
"""

def research_loop(question: str, max_turns=30, max_iterations=3):
    tools = {
        "search": web_search,
        "fetch": fetch_url,
        "extract_claims": extract_key_claims,
        "note": store_note,
        "get_notes": retrieve_notes,
    }
    
    notes = {}
    
    for iteration in range(max_iterations):
        # Run the research ReAct loop
        history = [
            {"role": "user", "content": f"Research question: {question}"}
        ]
        
        for turn in range(max_turns):
            response = client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=4096,
                system=RESEARCH_SYSTEM_PROMPT,
                messages=history,
                tools=build_tool_schemas(tools)
            )
            
            if response.stop_reason == "end_turn":
                # Check if the agent submitted a report via finish()
                report = extract_finish_content(history)
                if report:
                    break
            
            # Execute tool calls
            for tool_use in extract_tool_calls(response):
                if tool_use.name == "finish":
                    report = tool_use.input["report"]
                    break
                
                result = tools[tool_use.name](**tool_use.input)
                history.append({"role": "assistant", "content": response.content})
                history.append({"role": "user", "content": format_tool_result(tool_use.id, result)})
            else:
                continue
            break
        
        # Evaluate the report
        eval_response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=300,
            messages=[{
                "role": "user",
                "content": EVALUATOR_PROMPT.format(question=question, report=report)
            }]
        )
        evaluation = parse_json(eval_response.content[0].text)
        
        if evaluation["approved"]:
            return report
        
        # If not approved, continue with identified gaps
        gap_context = f"\n\nPrevious attempt gaps to address: {evaluation['gaps']}"
        question = question + gap_context
    
    return report  # Best effort after max iterations
```

### Multi-agent research (parallel fan-out)

For broader research, fan out multiple agents simultaneously:

```python
import asyncio

async def parallel_research(question: str, angles: list[str]):
    """
    Research different angles in parallel, then synthesize.
    """
    
    async def research_angle(angle: str):
        return await research_loop(f"{question} — focus on: {angle}")
    
    # Fan out across angles simultaneously
    results = await asyncio.gather(*[
        research_angle(angle) for angle in angles
    ])
    
    # Synthesize
    synthesis = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=8192,
        messages=[{
            "role": "user",
            "content": (
                f"Synthesize these research reports into one comprehensive report.\n"
                f"Question: {question}\n\n"
                + "\n\n---\n\n".join(f"Angle: {a}\n{r}" 
                                     for a, r in zip(angles, results))
            )
        }]
    )
    
    return synthesis.content[0].text

# Usage
report = await parallel_research(
    question="What are the best practices for agent loop design in 2025?",
    angles=[
        "academic research and papers",
        "industry frameworks and tools",
        "practitioner experience and case studies",
        "failure modes and anti-patterns"
    ]
)
```

## Example Trace

```
Question: "What are the key design decisions in agent loop architecture?"

Turn 1: search("agent loop architecture design 2024 2025")
        → 8 results with snippets

Turn 2: search("ReAct vs Plan-and-Execute agent comparison")
        → 6 results

Turn 3: fetch("https://anthropic.com/research/building-effective-agents")
        → 3000 tokens extracted
        note("anthropic_patterns", "Five patterns: prompt chaining, routing...")

Turn 4: fetch("https://arxiv.org/abs/2210.03629")  # ReAct paper
        → 2500 tokens
        note("react", "Interleaves thought-action-observation...")

Turn 5: fetch("https://blog.langchain.com/planning-agents/")
        → 2000 tokens
        note("plan_execute", "Planner + executors, better for large tasks...")

Turn 6: extract_claims(get_notes())
        → "Key claims: ReAct, Reflexion, Plan-and-Execute..."

Turn 7: search("agent loop stopping conditions best practices")
        → Found gap: no coverage of stopping conditions

Turn 8: fetch(stopping_conditions_article)
        → note("stopping", "Success, budget, timeout, escalation...")

Turn 9: get_notes()
        → Review all collected findings

Turn 10: finish("""
# Agent Loop Architecture: Key Design Decisions

## 1. Loop Pattern Selection
[ReAct, Reflexion, Plan-and-Execute based on task type...]

## 2. Stopping Conditions
[...]

## References
- Anthropic (2024): Building Effective Agents
- Yao et al. (2022): ReAct paper
...
""")

Evaluator: Score 9/10 — Approved ✓
```

## Stopping Conditions

| Condition | Action |
|-----------|--------|
| Report evaluates as approved | Return report |
| Max turns reached without finish | Submit partial report; flag as incomplete |
| Max iterations reached | Return best attempt; note gaps |
| Source fetch fails repeatedly | Continue with available sources |

## Quality Patterns

**Adversarial verification**: After collecting claims, spawn a separate agent to try to refute each one before including it in the report.

```python
async def verify_claim(claim: str) -> bool:
    refuter = await research_angle(f"Find evidence that contradicts: {claim}")
    verdict = evaluate_refutation(claim, refuter)
    return verdict.claim_survives
```

**Source diversity**: Ensure the report draws from multiple independent sources, not just one.

```python
def check_source_diversity(notes: dict) -> bool:
    domains = extract_domains(notes.values())
    return len(set(domains)) >= 3  # At least 3 distinct domains
```

## Cost Profile

For a 15-turn research loop with Claude Sonnet:
- ~15 LLM calls × ~3000 input + ~500 output tokens
- ~$0.25 per research task
- Multi-agent parallel (4 angles + synthesis): ~$1.00-1.50
- Acceptable for high-value research; cost budget the loop
