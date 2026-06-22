# Example: Issue Triage Loop (Event-Driven)

An issue triage loop fires when a new GitHub issue is created. The agent reads the issue, classifies it, applies labels, assigns it to the right team, and posts an initial response — all without human intervention.

## Use Cases

- GitHub issue auto-labeling and routing
- Customer support ticket classification
- Bug report severity assessment
- Feature request prioritization
- Spam/duplicate detection

## Architecture

```
EVENT: issue.opened
         │
         ▼
┌────────────────────────┐
│    TRIAGE LOOP          │
│                         │
│  1. Read issue          │ ──► issue body, title, author
│  2. Classify            │ ──► LLM: type, severity, team
│  3. Check duplicates    │ ──► search similar issues
│  4. Apply labels        │ ──► GitHub API
│  5. Assign              │ ──► GitHub API
│  6. Draft response      │ ──► LLM: friendly initial reply
│  7. Post comment        │ ──► GitHub API
│  8. Done                │
└────────────────────────┘
```

## Implementation

### GitHub Actions + Claude Code

```yaml
name: Issue Triage

on:
  issues:
    types: [opened]

permissions:
  issues: write
  contents: read

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - name: Triage issue
        uses: anthropics/claude-code-action@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            A new GitHub issue has been filed. Triage it:

            Issue #${{ github.event.issue.number }}: ${{ github.event.issue.title }}
            
            Body:
            ${{ github.event.issue.body }}
            
            Author: ${{ github.event.issue.user.login }}

            Steps:
            1. Classify the issue as one of: bug, feature-request, question, documentation, duplicate, invalid
            2. Assess severity for bugs: critical/high/medium/low
            3. Search for duplicate issues (use the search_issues tool)
            4. Apply the appropriate labels
            5. If it's a duplicate, link to the original and close
            6. Post a friendly initial response that:
               - Acknowledges the report
               - Confirms the classification
               - Sets expectations for next steps
               - Asks any clarifying questions needed
```

### Python implementation (webhook-based)

```python
from anthropic import Anthropic
import github_client as gh

client = Anthropic()

CLASSIFICATION_PROMPT = """
You are an expert software project maintainer. Triage this GitHub issue.

Issue: {title}
Body: {body}
Author: {author}

Return JSON:
{{
  "type": "bug" | "feature" | "question" | "docs" | "duplicate" | "invalid",
  "severity": "critical" | "high" | "medium" | "low" | null,
  "team": "backend" | "frontend" | "infra" | "docs" | "security",
  "duplicate_of": <issue_number> | null,
  "labels": ["label1", "label2"],
  "needs_clarification": ["question if needed"],
  "response_tone": "empathetic" | "technical" | "brief"
}}
"""

RESPONSE_PROMPT = """
Write a friendly GitHub issue response based on this triage:

Issue: {title}
Classification: {triage}
Needs clarification: {questions}

The response should:
- Thank the reporter
- Confirm what you understood the issue to be
- State the next steps (will investigate, needs more info, etc.)
- Ask any clarification questions naturally
- Be concise (3-5 sentences max)

Do not include any markdown headers or bullet points in the response.
"""

def triage_issue(issue_number: int, title: str, body: str, author: str):
    # Step 1: Classify
    classification_response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Fast and cheap for classification
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": CLASSIFICATION_PROMPT.format(
                title=title, body=body, author=author
            )
        }]
    )
    triage = parse_json(classification_response.content[0].text)
    
    # Step 2: Check for duplicates
    if triage["duplicate_of"]:
        gh.close_issue(issue_number, comment=(
            f"This appears to be a duplicate of #{triage['duplicate_of']}. "
            f"Please continue the discussion there."
        ))
        return
    
    # Step 3: Apply labels
    gh.add_labels(issue_number, triage["labels"])
    
    # Step 4: Assign to team
    team_assignees = TEAM_ASSIGNEES[triage["team"]]
    if team_assignees:
        gh.assign_issue(issue_number, team_assignees[0])  # Round-robin in production
    
    # Step 5: Generate response
    response_text = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": RESPONSE_PROMPT.format(
                title=title,
                triage=triage,
                questions=triage["needs_clarification"]
            )
        }]
    ).content[0].text
    
    # Step 6: Post comment
    gh.post_comment(issue_number, response_text)
    
    print(f"Issue #{issue_number} triaged: {triage['type']}/{triage['severity']}")
```

## Example Response Output

```
Input: 
  Title: "App crashes when uploading files > 50MB"
  Body: "I tried to upload a 75MB video and the app just crashes 
         with no error message. Chrome on macOS."

Triage output:
  type: bug
  severity: high
  team: backend
  labels: [bug, severity-high, file-upload]
  needs_clarification: ["What version of the app?", "Any console errors?"]

Posted comment:
  "Thanks for reporting this, @username! This looks like a file upload 
  size issue — we've flagged it as a high-severity bug and assigned it 
  to our backend team.

  To help us investigate faster: what version of the app are you using, 
  and were there any errors in the browser console when it crashed?

  We'll update you once we have more information."
```

## The Loop Within

When checking for duplicates, the agent runs an inner search loop:

```
Step: search_for_duplicates

Turn 1: search_issues("file upload crash")
        → 12 results

Turn 2: check_similarity(current_issue, result[0])
        → "Different issue: login crash, not file upload"

Turn 3: check_similarity(current_issue, result[3])
        → "Possible duplicate: 'Upload fails for large files' (#847)"

Turn 4: fetch_issue(847)
        → Read full body; compare carefully

Turn 5: decide: "Same root issue — mark as duplicate of #847"
```

## Stopping Conditions

| Outcome | Exit |
|---------|------|
| Duplicate found | Close with link; done |
| Classified + labeled + responded | Done |
| Issue body is empty/spam | Label "invalid"; request more info; done |
| Classification confidence low | Apply "needs-triage" label; done (escalate to human) |

## Variations

**PR triage**: Same pattern for pull requests — classify by size, type (feature/fix/refactor), auto-request reviewers, check for missing tests or docs.

**Comment-triggered**: Extend the trigger to `issue_comment` — agent can respond to follow-up questions, update labels when more info is provided, or close when resolved.

**Multi-repo routing**: For a monorepo or org-wide triage, extend to route to the right sub-team or repository.

## Cost Profile

Per-issue cost with Claude Haiku:
- 2 LLM calls (classify + respond) ~500 input + 200 output tokens each
- ~$0.0003 per issue
- 1000 issues/month = ~$0.30/month

Extremely cost-effective for the automation value delivered.
