# Trigger Types for Agent Loops

How a loop *starts* is as important as how it runs. There are three families of triggers, each suited to different use cases.

## Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                      TRIGGER FAMILIES                            │
│                                                                  │
│  MANUAL          SCHEDULED (CRON)        EVENT-DRIVEN            │
│  ──────          ─────────────────        ────────────           │
│  Human asks      Time-based              Something happened      │
│  → loop starts   → loop starts           → loop starts          │
│                                                                  │
│  Webhook/API     Cron expression         Repo event              │
│  CLI command     Fixed interval          Message queue           │
│  UI button       Fuzzy schedule          File system             │
│                                          Database trigger        │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. Manual / On-Demand

The loop starts when a human explicitly requests it.

### Mechanisms

| Mechanism | Description |
|-----------|-------------|
| **Chat message** | User sends a message; the agent loop begins |
| **`workflow_dispatch`** | GitHub UI, API, or CLI triggers a workflow manually |
| **CLI command** | `gh aw run my-agent-workflow` |
| **REST API call** | `POST /api/agents/run` with a JSON payload |
| **Webhook** | External service POSTs to your endpoint; loop starts |
| **UI button** | A "Run" button in a dashboard or application |

### GitHub Agentic Workflows example

```yaml
on:
  workflow_dispatch:
    inputs:
      topic:
        description: 'Research topic'
        required: true
```

### When to use

- Tasks requiring unique per-run human intent or input
- Variable-scope tasks (the goal changes each time)
- Debugging, testing, and development
- Any task where the human decides when it's needed

### Design considerations

- Provide a clear confirmation before expensive or irreversible actions
- Return a task ID so the human can check status asynchronously
- Support dry-run mode for testing the loop without side effects

---

## 2. Scheduled / Cron

The loop starts on a time-based schedule, without a human present.

### Mechanisms

| Format | Example |
|--------|---------|
| **Standard cron** | `0 9 * * 1-5` — 9 AM, weekdays |
| **Fixed intervals** | `hourly`, `daily`, `weekly`, `monthly` |
| **Fuzzy schedules** (gh-aw) | `"daily around 14:00"` |
| **Range schedules** (gh-aw) | `"daily between 9:00 and 17:00"` with UTC offset support |

### GitHub Agentic Workflows example

```yaml
on:
  schedule:
    - cron: "0 9 * * 1-5"
```

### AgenticFlow example

```json
{
  "trigger": "schedule",
  "config": {
    "type": "cron",
    "expression": "0 */6 * * *"
  }
}
```

Or with interval presets:

```json
{
  "trigger": "schedule",
  "config": {
    "type": "interval",
    "preset": "daily"
  }
}
```

### Cron syntax reference

```
┌──────── minute (0-59)
│ ┌────── hour (0-23)
│ │ ┌──── day of month (1-31)
│ │ │ ┌── month (1-12)
│ │ │ │ ┌ day of week (0-7, 0=Sunday)
│ │ │ │ │
* * * * *

Examples:
0 * * * *     = every hour, on the hour
0 9 * * 1-5   = 9 AM, Monday through Friday
*/15 * * * *  = every 15 minutes
0 0 1 * *     = midnight on the 1st of every month
```

### When to use

- Monitoring and alerting (check metrics every N minutes)
- Periodic reporting (daily summaries, weekly digests)
- Data synchronization (pull from external API daily)
- Proactive outreach (send reminders at scheduled times)
- Background maintenance (clean up, index, summarize)

### Design considerations

- Consider timezone carefully — prefer UTC, make offset explicit
- Add jitter to prevent thundering-herd if many loops fire at the same second
- Handle the case where the previous run is still in progress (skip, queue, or cancel)
- Log every scheduled run, even if it has nothing to do

---

## 3. Event-Driven

The loop starts when something happens in an external system.

### Mechanisms

| Source | Examples |
|--------|----------|
| **Repository activity** (gh-aw) | New issue, PR opened, comment created |
| **Webhook payload** | Stripe payment, Slack message, form submission |
| **Message queue** | SQS, Kafka, Pub/Sub message arrives |
| **File system** | New file dropped in a watched folder |
| **Database trigger** | Row inserted, column updated |
| **Semantic triggers** | Text in a message matches a pattern |

### GitHub Agentic Workflows examples

```yaml
# Trigger on new issues
on:
  issues:
    types: [opened]

# Trigger on pull request events
on:
  pull_request:
    types: [opened, synchronize, reopened]

# Trigger on comments
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  discussion_comment:
    types: [created]
```

### Webhook pattern (generic)

```python
@app.post("/webhook")
async def handle_event(payload: dict):
    event_type = payload["type"]
    if event_type in HANDLED_EVENTS:
        await start_agent_loop(
            goal=derive_goal(payload),
            context=payload
        )
    return {"status": "accepted"}
```

### When to use

- Real-time response required (issue triage, support routing)
- Reactive automation (process payment → send receipt → update CRM)
- Human-in-the-loop at the trigger point (the event is the human's action)
- Event-sourced architectures
- Anything where latency from polling would be unacceptable

### Design considerations

- Validate webhook signatures to prevent spoofed triggers
- Use idempotency keys to prevent duplicate loop runs on retried events
- Queue events if the loop is slow; don't process synchronously in the webhook handler
- Implement dead-letter queues for failed event processing
- Rate-limit per source to prevent runaway triggering

---

## Combining Trigger Types

Real systems often use multiple trigger types together:

**Example: CI/CD agent**
- **Event-driven**: Start on PR open (review code immediately)
- **Scheduled**: Nightly full audit of all open PRs
- **Manual**: `workflow_dispatch` for on-demand full suite run

**Example: Customer support agent**
- **Event-driven**: Ticket created → immediate triage loop
- **Scheduled**: Hourly check for tickets awaiting follow-up > 4 hours
- **Manual**: Support lead triggers escalation review on demand

---

## Source Notes

GitHub Agentic Workflows (gh-aw) is in **technical preview** as of 2026. Trigger syntax and fuzzy-schedule formats may change. Treat its documentation as current-but-volatile.

AgenticFlow's trigger taxonomy (webhook + schedule) focuses on automated execution; manual/table/API runs are also supported in their platform but are classified separately from "triggers" in their documentation.

---

*Sources: GitHub Agentic Workflows trigger reference (technical preview, 2026); AgenticFlow trigger documentation; AgentC2 AI scheduling guide*
