# Example: Sales Outreach Loop (Research → Personalize → Send → Track → Follow-Up)

A sales outreach loop automates the research-and-personalization cycle that makes cold outreach effective. Instead of sending generic templates, the agent researches each prospect, writes a personalized message, tracks engagement, and follows up based on behavior — running autonomously after a manual trigger or on a cron schedule.

## Real Implementation

| Repo | Framework | Description |
|------|-----------|-------------|
| [kaymen99/sales-outreach-automation-langgraph](https://github.com/kaymen99/sales-outreach-automation-langgraph) | LangGraph | Full pipeline: research → personalize → send; multi-agent |

## Architecture

```
TRIGGER: manual (list of prospects) OR cron (new leads from CRM)
         │
         ▼
┌──────────────────────────────────────────────────────────────────┐
│               SALES OUTREACH LOOP                                 │
│                                                                   │
│  For each prospect in the queue:                                  │
│                                                                   │
│  1. research_prospect                                             │
│     → company website, LinkedIn, recent news, job postings       │
│                                                                   │
│  2. identify_pain_points                                          │
│     → LLM: infer relevant problems from research                 │
│                                                                   │
│  3. personalize_message                                           │
│     → LLM: write outreach that references specific findings      │
│                                                                   │
│  4. review_gate (optional human checkpoint)                       │
│     → human approves or edits before sending                     │
│                                                                   │
│  5. send_message                                                  │
│     → email / LinkedIn / Slack                                   │
│                                                                   │
│  6. track_engagement                                              │
│     → poll for open, click, reply                                │
│                                                                   │
│  7. route:                                                        │
│     ├─ replied? → handle_reply (new loop)                        │
│     ├─ opened but no reply after N days? → send_followup         │
│     ├─ no open after N days? → send_followup (different angle)   │
│     └─ max_followups reached? → mark_complete, move on           │
└──────────────────────────────────────────────────────────────────┘
```

## LangGraph Implementation

From [kaymen99/sales-outreach-automation-langgraph](https://github.com/kaymen99/sales-outreach-automation-langgraph):

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Optional

class OutreachState(TypedDict):
    prospect: dict           # name, company, title, email, linkedin
    research: str            # web research findings
    pain_points: list[str]   # inferred problems
    message: str             # drafted outreach
    sent_at: Optional[str]
    engagement: dict         # opened, clicked, replied
    followup_count: int
    status: str              # active / replied / exhausted

# Node 1: Research the prospect
def research_prospect(state: OutreachState) -> OutreachState:
    prospect = state["prospect"]
    
    findings = []
    
    # Company website
    company_page = fetch_url(prospect["company_url"])
    findings.append(f"Company overview: {summarize(company_page)}")
    
    # Recent news
    news = web_search(f"{prospect['company']} news recent announcements")
    findings.append(f"Recent news: {news[:500]}")
    
    # Job postings (reveals priorities and pain points)
    jobs = web_search(f"{prospect['company']} hiring site:linkedin.com OR site:greenhouse.io")
    findings.append(f"Current hiring: {jobs[:300]}")
    
    return {**state, "research": "\n".join(findings)}

# Node 2: Identify pain points from research
def identify_pain_points(state: OutreachState) -> OutreachState:
    prompt = f"""
    Based on this research about {state['prospect']['company']},
    what business problems are they likely facing that our product solves?
    
    Research:
    {state['research']}
    
    Our product: [your product description]
    
    Return 2-3 specific, evidence-backed pain points.
    """
    pain_points_text = llm.call(prompt)
    pain_points = parse_bullet_list(pain_points_text)
    return {**state, "pain_points": pain_points}

# Node 3: Write personalized outreach
def personalize_message(state: OutreachState) -> OutreachState:
    prompt = f"""
    Write a personalized cold outreach email to {state['prospect']['name']},
    {state['prospect']['title']} at {state['prospect']['company']}.
    
    Key findings about their company:
    {state['research'][:600]}
    
    Pain points to address:
    {chr(10).join(f"- {p}" for p in state['pain_points'])}
    
    Rules:
    - Under 150 words
    - Reference one specific finding from the research
    - One clear CTA (15-minute call)
    - No generic opener ("I hope this finds you well")
    - No feature list — lead with their problem
    - First-name only in greeting
    """
    message = llm.call(prompt)
    return {**state, "message": message}

# Node 4: Send
def send_message(state: OutreachState) -> OutreachState:
    send_email(
        to=state["prospect"]["email"],
        subject=generate_subject(state),
        body=state["message"]
    )
    return {**state, "sent_at": now_iso(), "status": "sent"}

# Node 5: Check engagement
def check_engagement(state: OutreachState) -> OutreachState:
    engagement = email_tracker.get_status(state["prospect"]["email"])
    return {**state, "engagement": engagement}

# Node 6: Write follow-up
def write_followup(state: OutreachState) -> OutreachState:
    followup_num = state["followup_count"] + 1
    
    angles = {
        1: "different value angle — address a different pain point",
        2: "social proof — reference a customer in their industry",
        3: "breakup email — brief, assume they're not interested, leave door open"
    }
    
    prompt = f"""
    Write follow-up #{followup_num} to {state['prospect']['name']}.
    
    Original message:
    {state['message']}
    
    Engagement: {state['engagement']}
    
    Angle for this follow-up: {angles.get(followup_num, 'brief check-in')}
    Under 80 words.
    """
    followup = llm.call(prompt)
    return {**state, "message": followup, "followup_count": followup_num}

# Routing
def route_after_engagement_check(state: OutreachState) -> str:
    if state["engagement"].get("replied"):
        return "handle_reply"
    if state["followup_count"] >= 3:
        return "mark_exhausted"
    if days_since(state["sent_at"]) >= 3:
        return "write_followup"
    return "wait"  # Check again later

# Build graph
builder = StateGraph(OutreachState)
for node_name, fn in [
    ("research_prospect", research_prospect),
    ("identify_pain_points", identify_pain_points),
    ("personalize_message", personalize_message),
    ("send_message", send_message),
    ("check_engagement", check_engagement),
    ("write_followup", write_followup),
]:
    builder.add_node(node_name, fn)

builder.set_entry_point("research_prospect")
builder.add_edge("research_prospect", "identify_pain_points")
builder.add_edge("identify_pain_points", "personalize_message")
builder.add_edge("personalize_message", "send_message")
builder.add_edge("send_message", "check_engagement")
builder.add_conditional_edges("check_engagement", route_after_engagement_check)
builder.add_edge("write_followup", "send_message")

graph = builder.compile()
```

## Trigger Pattern: Cron + CRM Integration

```python
# Run every morning: pull new leads from CRM, start a loop for each
@cron("0 8 * * 1-5")  # 8 AM weekdays
async def process_new_leads():
    new_leads = crm.get_leads(status="new", limit=20)
    
    for lead in new_leads:
        # Start an outreach loop for each new prospect
        await graph.ainvoke({
            "prospect": lead,
            "research": "",
            "pain_points": [],
            "message": "",
            "sent_at": None,
            "engagement": {},
            "followup_count": 0,
            "status": "queued"
        })
        
        crm.update_lead(lead["id"], status="outreach_active")
```

## Stopping Conditions

| Condition | Action |
|-----------|--------|
| Prospect replied | Pause loop; route to human SDR for reply handling |
| Max follow-ups (3) reached without reply | Mark prospect as `exhausted`; stop |
| Prospect unsubscribed | Immediately stop; mark `opted_out` |
| Prospect bounced | Mark `invalid_email`; stop |
| Human manually stops | Cancel loop; mark as `manually_closed` |

## The Human Review Gate

For most teams, the loop should pause for human review before the first send — especially early on, before you trust the personalization quality:

```python
def personalize_message(state: OutreachState) -> OutreachState:
    # ... generate message ...
    
    if REQUIRE_HUMAN_REVIEW:
        # Post to Slack for review
        slack.post(
            channel="#outreach-review",
            text=f"Review outreach to {state['prospect']['name']}:\n\n{message}",
            actions=["Approve", "Edit", "Skip"]
        )
        # Loop pauses here; resumes when human acts
        return {**state, "message": message, "status": "pending_review"}
    
    return {**state, "message": message}
```

Remove the gate once you're confident in the quality. This is the right way to ramp autonomous outreach.

## What Good Research Looks Like

The personalization is only as good as the research. High-signal sources:

| Source | What to look for |
|--------|-----------------|
| Company blog | Strategic priorities, recent launches, pain points they're writing about |
| Job postings | Tech stack (reveals infrastructure), hiring surges (reveals growth areas), leadership hires |
| Press releases | Funding, acquisitions, new markets — all create buying triggers |
| LinkedIn activity | What the target person posts/comments on — reveals their personal interests |
| G2 / Capterra reviews | What customers complain about in their current solution |

## Example Message (Before/After)

**Generic (before loop)**:
```
Hi Sarah, I hope this email finds you well! I'm reaching out because I think 
our platform could help DataCo. We offer AI-powered analytics with real-time 
dashboards. Would you be open to a 15-minute call?
```

**Personalized (after loop)**:
```
Hi Sarah, saw DataCo is hiring three data engineers and just closed a Series B — 
congrats. Usually means the BI stack is hitting its limits right as the team 
doubles. We helped Acme go from quarterly reports to live dashboards without 
adding headcount. Worth a 15-minute call to see if it fits?
```

The second message references specific research (hiring, funding), implies a pain point from that research (BI stack scaling), and provides social proof. The loop produced this automatically.

## Known Pitfalls

**Over-personalization feels creepy**: Referencing that you saw someone's 3-year-old tweet is off-putting. Stick to public company-level information and recent professional activity.

**Rate limits on send**: Email providers throttle bulk sends. Add delays between sends: `await asyncio.sleep(random.randint(60, 180))` between prospects.

**Deliverability**: AI-written emails may have patterns spam filters catch. Vary sentence structure, avoid excessive punctuation, warm up the sending domain.

**CRM sync**: The loop state must stay in sync with the CRM. If the SDR manually marks a lead as "disqualified" in Salesforce, the loop should stop. Poll CRM status at each follow-up decision point.

**Reply handling**: This loop stops at first reply — it doesn't handle replies. Reply handling requires a separate loop (read reply → classify intent → draft response → route to human if needed).

## Sources

- [github.com/kaymen99/sales-outreach-automation-langgraph](https://github.com/kaymen99/sales-outreach-automation-langgraph) — full LangGraph sales outreach pipeline
