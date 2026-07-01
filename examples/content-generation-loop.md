# Example: Content Generation Loop (Draft → Critique → Revise → Publish)

A content generation loop produces high-quality written output by running a generate-critique-revise cycle. One LLM drafts; another critiques; the draft is revised until the critique is empty or a quality threshold is met. This is the evaluator-optimizer pattern applied to writing.

## Real Implementations

| Repo | Framework | Description |
|------|-----------|-------------|
| [langchain-ai/langgraph-reflection](https://github.com/langchain-ai/langgraph-reflection) | LangGraph | Generate → reflect → revise; loops while critiques exist |
| Anthropic evaluator-optimizer | Anthropic API | Described in Building Effective Agents as the canonical content loop |

## Architecture

```
MANUAL TRIGGER: user provides topic, brief, or requirements
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│               CONTENT GENERATION LOOP                         │
│                                                               │
│  1. plan        → outline / structure / key points           │
│  2. draft       → first full draft from outline              │
│  3. reflect     → critic LLM: list all issues                │
│       ├─ critiques non-empty? → revise (with critiques)      │
│       └─ critiques empty?    → format_and_publish            │
│  4. revise      → generator LLM: fix the listed issues       │
│       └─ back to reflect                                      │
│  5. format_and_publish → final formatting + delivery          │
└──────────────────────────────────────────────────────────────┘
```

## LangGraph Reflection Implementation

From [langchain-ai/langgraph-reflection](https://github.com/langchain-ai/langgraph-reflection):

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, list

class ContentState(TypedDict):
    topic: str
    requirements: str
    draft: str
    critiques: list[str]
    revision_count: int

# Node 1: Generate initial draft
def generate_draft(state: ContentState) -> ContentState:
    prompt = f"""
    Write a {state['requirements']} about: {state['topic']}
    
    Previous draft (if revising):
    {state.get('draft', 'none')}
    
    Issues to fix (if revising):
    {chr(10).join(state.get('critiques', [])) or 'none — first draft'}
    """
    draft = generator_llm.call(prompt)
    return {**state, "draft": draft, "revision_count": state["revision_count"] + 1}

# Node 2: Critique the draft
def reflect_on_draft(state: ContentState) -> ContentState:
    prompt = f"""
    Critically review this draft for the task: {state['requirements']} about {state['topic']}
    
    Draft:
    {state['draft']}
    
    List ONLY the specific issues that must be fixed. 
    If the draft is good enough to publish, return an empty list.
    Be concise — one issue per bullet.
    
    Return JSON: {{"critiques": ["issue 1", "issue 2"]}}
    """
    result = critic_llm.call(prompt)
    critiques = parse_json(result)["critiques"]
    return {**state, "critiques": critiques}

# Routing: loop while there are critiques, exit when empty
def route_after_reflection(state: ContentState) -> str:
    if state["critiques"] and state["revision_count"] < 5:
        return "generate_draft"  # Revise
    return "publish"             # Done

def publish(state: ContentState) -> ContentState:
    formatted = format_for_delivery(state["draft"])
    deliver(formatted)
    return {**state, "published": True}

# Build graph
builder = StateGraph(ContentState)
builder.add_node("generate_draft", generate_draft)
builder.add_node("reflect_on_draft", reflect_on_draft)
builder.add_node("publish", publish)

builder.set_entry_point("generate_draft")
builder.add_edge("generate_draft", "reflect_on_draft")
builder.add_conditional_edges("reflect_on_draft", route_after_reflection)
builder.add_edge("publish", END)

graph = builder.compile()
```

**The key mechanic**: `reflect_on_draft` re-calls `generate_draft` if `critiques` is non-empty; routes to `publish` when critiques are empty. The loop converges when the critic has nothing left to say.

## Stopping Conditions

| Condition | Route |
|-----------|-------|
| Critiques list is empty | `reflect_on_draft` → `publish` → END |
| Max revisions reached (default 5) | `reflect_on_draft` → `publish` → END |
| Draft flagged as unpublishable | Optional: escalate to human review |

## Separating Generator and Critic

Using different models or different system prompts for generator vs. critic prevents the "agreeable evaluator" failure mode:

```python
# Generator: optimistic, creative, focused on producing
generator_llm = claude.with_system("""
You are a skilled writer. Produce the best possible draft.
Focus on clarity, engagement, and meeting the requirements.
""")

# Critic: adversarial, focused on finding problems
critic_llm = claude.with_system("""
You are a demanding editor. Your job is to find every problem with this draft.
Be specific and actionable. Don't say "improve X" — say exactly what's wrong.
Return an empty list only when the draft genuinely needs no changes.
""")
```

**Using the same model with different system prompts works well** — the different framing shifts the model's behavior sufficiently.

## Critique Quality Matters More Than Draft Quality

The loop's convergence depends on the critic being right. Poor critique → endless revision:

```python
# Bad critique (too vague):
critiques = ["The article could be better", "Needs more depth"]

# Good critique (specific and actionable):
critiques = [
    "Section 2 contradicts section 4: section 2 says X is beneficial, section 4 says X is harmful. Reconcile.",
    "The conclusion doesn't follow from the evidence presented — claims Y but only evidence for Z is given.",
    "Third paragraph is 400 words with a single idea. Split into two paragraphs."
]
```

Vague critiques produce vague revisions. Enforce specificity in the critic prompt.

## Specializing the Loop by Content Type

### Blog post loop
```python
BLOG_REQUIREMENTS = """
- 800-1200 words
- Strong hook in first paragraph
- 3-5 main sections with headers
- Concrete examples for each claim
- Actionable takeaway in conclusion
- Conversational tone, no jargon
"""
```

### Technical documentation loop
```python
DOCS_REQUIREMENTS = """
- Clear purpose statement in first paragraph
- Prerequisites listed before main content
- Code examples for every concept
- Common pitfalls section
- Links to related docs
- All code examples must be runnable
"""
```

### Marketing copy loop
```python
COPY_REQUIREMENTS = """
- Lead with the benefit, not the feature
- Social proof within first 100 words
- Single clear CTA
- No passive voice
- Under 200 words
- A/B test the headline: provide 3 options
"""
```

## Human-in-the-Loop Checkpoint

For published content, add a human review step before delivery:

```python
def route_after_reflection(state: ContentState) -> str:
    if state["critiques"] and state["revision_count"] < 5:
        return "generate_draft"
    
    # Final human checkpoint for sensitive content
    if state.get("requires_human_review"):
        return "human_review"
    
    return "publish"

async def human_review(state: ContentState) -> ContentState:
    # Send draft to Slack/email for approval
    approval = await request_human_approval(
        content=state["draft"],
        timeout=3600  # 1 hour to approve
    )
    if approval.approved:
        return {**state, "critiques": []}  # Clear critiques → publish
    else:
        # Human gave feedback — treat as critiques
        return {**state, "critiques": approval.feedback.split("\n")}
```

## Example Trace

```
Topic: "How to choose between PostgreSQL and MongoDB"
Requirements: Technical blog post, 1000 words, concrete examples

Draft 1 → Reflect:
  Critiques:
  - "Claims MongoDB is faster for reads — no evidence given"
  - "No concrete example for schema-flexible use case"
  - "PostgreSQL JSONB not mentioned despite being directly relevant"

Draft 2 → Reflect:
  Critiques:
  - "PostgreSQL JSONB example has syntax error: uses {$gt: 5} which is MongoDB syntax"

Draft 3 → Reflect:
  Critiques: []  ← empty → PUBLISH
```

Three iterations, each targeted to a specific flaw. The final draft is substantially better than draft 1 with no human involvement.

## Known Pitfalls

**Oscillation**: Draft adds X, critic says remove X, draft removes X, critic says add X. Detect with semantic similarity on consecutive drafts.

**Critic fatigue**: After several revisions, the critic starts approving mediocre drafts because the delta between revisions is small. Add a "final review" step that evaluates the draft against the original requirements from scratch.

**Scope creep**: Each revision adds content; the draft grows and drifts from the requirements. Enforce word count and topic scope in the critic prompt.

**Factual drift**: Revisions may introduce new facts not in the original research. Add a fact-check step if factual accuracy is critical.

## Sources

- [github.com/langchain-ai/langgraph-reflection](https://github.com/langchain-ai/langgraph-reflection) — generate-reflect-revise loop; loops while critiques non-empty
- [anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents) — evaluator-optimizer pattern (the canonical name for this loop type)
