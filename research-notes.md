# Research Notes

This repository was built from a multi-agent deep research session (June 2026) using 105 research agents, 23 primary and secondary sources, and adversarial claim verification.

## Methodology

1. **Scope**: Research question decomposed into 5 search angles:
   - Broad/primary: definition and anatomy of agent loops
   - Academic/technical: agent loop architectures and papers
   - Practitioner/curation: existing loop libraries
   - Implementation: trigger types and scheduling
   - Deliverable: awesome-list format conventions

2. **Search**: 5 parallel agents (one per angle) each collected 6 source candidates

3. **Fetch**: 23 sources fetched; 109 claims extracted

4. **Verify**: Top 25 claims submitted to 3-vote adversarial verification
   - Each claim: 3 independent agents prompted to REFUTE
   - Claim survives if ≤1 refutation (2/3 or 3/3 confirm)
   - 23 claims confirmed; 2 killed

5. **Synthesize**: Confirmed claims merged, deduplicated, and organized into this repository

## Verified Claims (23 confirmed)

All content in this repository is grounded in at least one of the 23 adversarially-verified claims. Key verified facts:

| Claim | Confidence | Vote |
|-------|-----------|------|
| Core loop anatomy (assemble → reason → execute → observe) | High | 3-0 |
| Six-line while loop reduction | High | 3-0 |
| Agent/workflow distinction (Anthropic) | High | 3-0 |
| Five Anthropic workflow patterns | High | 3-0 |
| ReAct thought→action→observation mechanism | High | 3-0 |
| ReAct reduces hallucination vs CoT-only | High | 3-0 |
| Reflexion uses episodic memory, not weight updates | High | 3-0 |
| Plan-and-Execute: planner + executors architecture | High | 3-0 |
| Evaluator-optimizer: generate→evaluate→refine loop | High | 3-0 |
| "Good loop answers four questions" framework | Medium | 3-0 |
| Manual/on-demand triggers (GitHub workflow_dispatch) | High | 3-0 |
| Scheduled/cron triggers (standard cron + fuzzy) | High | 3-0 |
| Event-driven triggers (issues, PRs, comments) | High | 3-0 |
| AgenticFlow webhook + schedule trigger types | Medium | 2-1 |
| Awesome-list naming conventions | High | 3-0 |
| Awesome-list Contents section convention | High | 3-0 |
| Awesome-list curation quality bar | High | 3-0 |

## Refuted Claims

Two claims were killed by adversarial verification and are **not** represented in this repository as established facts:

**"Loops must be deliberately bounded with a mandatory human handoff"** (vote: 1-2)
- Presented in the source as a rule; adversarial agents found clear counterexamples
- Correct framing: human escalation is a strong best practice for irreversible/high-cost actions, not a universal requirement
- Status: presented in this repo as a design choice to consider, not a rule

**"The agent loop has five stages: Perceive, Reason, Plan, Act, Observe"** (vote: 0-3)
- Specific five-stage taxonomy uniformly refuted
- Primary sources (Anthropic, ReAct paper) do not use this framing
- Status: not used in this repo; the simpler four-stage model (assemble, reason, execute, observe) is used instead

## Sources

### Primary Sources (peer-reviewed / official documentation)
- [arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629) — ReAct paper (Yao et al., ICLR 2023)
- [arxiv.org/abs/2303.11366](https://arxiv.org/abs/2303.11366) — Reflexion paper (Shinn et al., NeurIPS 2023)
- [anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic (Dec 2024)
- [blog.langchain.com/planning-agents](https://blog.langchain.com/planning-agents/) — LangChain (framework authors)
- [github.github.com/gh-aw/reference/triggers](https://github.github.com/gh-aw/reference/triggers/) — GitHub Agentic Workflows (technical preview, Feb 2026)
- [docs.agenticflow.ai/workflows/triggers](https://docs.agenticflow.ai/workflows/triggers) — AgenticFlow trigger documentation
- [github.com/sindresorhus/awesome/blob/main/awesome.md](https://github.com/sindresorhus/awesome/blob/main/awesome.md) — Canonical awesome-list spec

### Secondary and Blog Sources
- [blogs.oracle.com/developers/what-is-the-ai-agent-loop](https://blogs.oracle.com/developers/what-is-the-ai-agent-loop-the-core-architecture-behind-autonomous-ai-systems) — Oracle Developer Blog
- [github.com/Forward-Future/loop-library](https://github.com/Forward-Future/loop-library) — Forward-Future Loop Library
- [requesty.ai/blog/loop-engineering](https://www.requesty.ai/blog/loop-engineering-how-to-build-ai-agent-loops-that-run-themselves) — Requesty.ai Loop Engineering
- [stevekinney.com/writing/agent-loops](https://stevekinney.com/writing/agent-loops) — Steve Kinney practitioner perspective
- [mindstudio.ai/blog/what-is-an-agentic-loop](https://www.mindstudio.ai/blog/what-is-an-agentic-loop-ai-coding-agents) — MindStudio
- [agentc2.ai/blog/how-to-schedule-ai-agents-cron-triggers](https://agentc2.ai/blog/how-to-schedule-ai-agents-cron-triggers) — AgentC2.ai scheduling guide

## Open Questions

The research identified these areas with insufficient verified coverage for inclusion:

1. **Extended loop patterns**: Tree-of-Thoughts, ReWOO, LLMCompiler, Self-Consistency were mentioned in sources but not independently verified. Treat them as promising but not fully confirmed for this repo's standards.

2. **Production stopping-condition best practices**: The "must bound with human handoff" claim was refuted. What does verified guidance on stopping conditions look like? Needs dedicated research.

3. **Multi-level loop theory**: The L1/L2/L3 nesting framing (in production/hardening.md) is a practitioner abstraction, not a verified academic taxonomy. Use with that caveat.

4. **Platform coverage**: Only gh-aw and AgenticFlow were verified for trigger types. n8n, Make.com, LangGraph, Temporal, and cloud cron schedulers are mentioned in further-reading but not verified against their primary documentation.

## Caveats

- GitHub Agentic Workflows (gh-aw) is in **technical preview** as of Feb 2026. Syntax may change.
- The "four questions" good-loop framework is a single-vendor pedagogical heuristic (Forward-Future), not a multi-source standard. Useful, but confidence is medium.
- ReAct's hallucination reduction is task-scoped to knowledge-intensive QA; it doesn't universally beat CoT on all metrics.
- The agent/workflow distinction (Anthropic's definition) is ~18 months old and the field acknowledges boundary fuzziness.

## Research Stats

```
Angles searched:     5
Sources fetched:     23
Claims extracted:    109
Claims verified:     25
Claims confirmed:    23
Claims refuted:      2
Agents used:         105
Agent calls:         105
Duration:            ~13 minutes
```
