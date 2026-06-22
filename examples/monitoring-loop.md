# Example: Monitoring Loop (Scheduled / Cron)

A monitoring loop runs on a schedule, checks a data source, and takes action when a condition is met. This is one of the most common and useful agent loop patterns for production systems.

## Use Cases

- Security log anomaly detection
- Price tracking and alerts
- Uptime and performance monitoring
- Social media mention monitoring
- Database metric thresholds
- Business KPI alerts

## Architecture

```
CRON TRIGGER (every 15 minutes)
         │
         ▼
┌────────────────────┐
│   MONITOR LOOP     │
│                    │
│  1. Fetch data     │ ──► data source (API, DB, log)
│  2. Analyze        │ ──► LLM: "Is anything anomalous?"
│  3. Check          │
│     ├─ normal? → log and exit
│     └─ anomaly? → take action
│  4. Action         │ ──► notify / escalate / remediate
└────────────────────┘
```

## Implementation

### Basic monitoring loop

```python
import schedule
import time
from anthropic import Anthropic

client = Anthropic()
BASELINE = None  # Set on first run

def monitoring_loop():
    global BASELINE
    
    # 1. Fetch data
    metrics = fetch_current_metrics()
    
    # 2. Analyze with LLM
    prompt = f"""
    Analyze these system metrics and determine if anything is anomalous.
    
    Current metrics:
    {format_metrics(metrics)}
    
    Baseline (last 24h average):
    {format_metrics(BASELINE) if BASELINE else "No baseline yet (first run)"}
    
    Return JSON: {{
        "anomaly_detected": bool,
        "severity": "low" | "medium" | "high" | "critical",
        "findings": ["finding 1", "finding 2"],
        "recommended_action": "description or null"
    }}
    """
    
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=500,
        messages=[{"role": "user", "content": prompt}]
    )
    
    analysis = parse_json(response.content[0].text)
    
    # 3. Check
    if not analysis["anomaly_detected"]:
        log_clean_check(metrics)
        update_baseline(metrics)
        return
    
    # 4. Take action based on severity
    if analysis["severity"] in ["high", "critical"]:
        alert_oncall(
            severity=analysis["severity"],
            findings=analysis["findings"],
            action=analysis["recommended_action"]
        )
    elif analysis["severity"] == "medium":
        create_ticket(analysis["findings"])
    else:
        log_warning(analysis["findings"])
    
    # Update baseline
    BASELINE = metrics

def fetch_current_metrics():
    return {
        "error_rate": get_error_rate(),
        "p99_latency_ms": get_p99_latency(),
        "active_users": get_active_users(),
        "db_connections": get_db_connection_count(),
        "memory_usage_pct": get_memory_usage(),
    }

# Run every 15 minutes
schedule.every(15).minutes.do(monitoring_loop)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### GitHub Agentic Workflows version

```yaml
name: System Health Monitor

on:
  schedule:
    - cron: "*/15 * * * *"  # Every 15 minutes
  workflow_dispatch:         # Also allow manual trigger

jobs:
  monitor:
    runs-on: ubuntu-latest
    steps:
      - name: Check metrics
        uses: actions/claude-code@v1
        with:
          prompt: |
            Check the current system health:
            1. Fetch metrics from ${{ env.METRICS_API_URL }}
            2. Compare against baseline stored in the monitoring artifact
            3. If anomalous, create a GitHub issue with severity label
            4. Post a summary to the monitoring-alerts Slack channel
          tools: |
            - fetch_url
            - create_github_issue
            - post_slack_message
            - read_artifact
            - write_artifact
```

## The Loop Within the Loop

The monitoring agent itself runs an inner ReAct-style loop when taking action:

```
Anomaly detected (high severity)

Turn 1: fetch_runbook("high_error_rate")
        → Returns: "Step 1: Check recent deploys..."

Turn 2: search_github("recent merges to main last 2 hours")
        → Returns: [list of PRs]

Turn 3: correlate(error_spike_time, merge_times)
        → Returns: "Error spike correlates with PR #234"

Turn 4: fetch_pr_details(234)
        → Returns: PR description, changed files

Turn 5: alert_oncall(
          summary="Error spike likely caused by PR #234",
          evidence=evidence,
          suggested_action="Consider rolling back"
        )
```

## Stopping Conditions

| Condition | Action |
|-----------|--------|
| Clean check | Log, update baseline, exit |
| Anomaly detected | Take severity-appropriate action, exit |
| Fetch fails | Retry 3x with backoff; alert if all fail |
| LLM call fails | Alert ops; use deterministic fallback |
| Previous run still active | Skip this run (don't stack) |

## Best Practices

**Idempotency**: Monitoring loops must be safe to run multiple times. Avoid creating duplicate alerts for the same anomaly.

```python
def is_already_alerted(anomaly_fingerprint: str) -> bool:
    return cache.exists(f"alert:{anomaly_fingerprint}")

def alert_with_dedup(findings, ttl_minutes=60):
    fingerprint = hash(str(sorted(findings)))
    if not is_already_alerted(fingerprint):
        send_alert(findings)
        cache.set(f"alert:{fingerprint}", True, ttl=ttl_minutes * 60)
```

**Baseline drift**: Update the baseline slowly (rolling average), not hard-replace, to avoid false positives after a healthy spike.

**Run tracking**: Every monitoring run should log its result, even clean ones.

```python
def log_monitor_run(result: str, metrics: dict, anomaly: bool):
    structured_log({
        "event": "monitoring_run",
        "timestamp": now_iso(),
        "result": result,
        "anomaly": anomaly,
        "metrics": metrics,
        "run_id": current_run_id()
    })
```

**Escalation ladder**:
```
low → create ticket
medium → create ticket + Slack DM to on-call
high → page on-call + create incident
critical → page all-hands + create incident + auto-rollback
```

## Cost Profile

For a monitoring loop checking 10 metrics every 15 minutes:
- ~4 LLM calls per hour (assuming 30s per run with retries)
- ~500 input tokens + ~200 output tokens per call
- With Claude Haiku: ~$0.002/hour = ~$1.50/month
- With Claude Sonnet: ~$0.02/hour = ~$14/month

Use the cheapest model that can reliably detect your anomaly types.
