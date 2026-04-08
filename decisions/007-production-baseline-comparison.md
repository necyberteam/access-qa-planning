# Decision 007: Production Baseline Comparison Eval

**Date:** 2026-04-08
**Status:** Proposed
**Decided by:** Drew, pending leadership review

## Context

The ACCESS AI Assistant is ready to deploy to production to replace the current production Q&A system, which is a direct, unmediated UKY RAG endpoint — user questions go straight to the RAG service and the raw response is rendered in the chat UI. There is no query classification, no MCP tool calling, no synthesis layer, no resource scoping.

The new agent ([Decision 003](./003-uky-first-architecture.md)) wraps the RAG service with a LangGraph pipeline that adds: query classification, parallel RAG + MCP tool execution, domain agents for write operations, synthesis over multiple sources, and resource-scoped RAG for RP documentation pages.

Before switching production traffic to the new agent, we need evidence that it delivers responses **at least as good as** the current production system on the same questions. Leadership and ACCESS operations cybersecurity review have both asked for this evidence.

[Decision 005](./005-llm-judge-eval-pipeline.md) established an LLM-judge eval pipeline for ongoing quality tracking against question batteries. That pipeline judges synthesis quality against sources, which is the right signal for ongoing quality work but is not on its own a production-readiness comparison — it doesn't run the current production system, doesn't compare two systems side-by-side, and doesn't define a go/no-go threshold.

## Decision

Run a three-way eval comparison and use the results to decide production go/no-go. The three systems are:

| Label | What it is | Why it's in the comparison |
|-------|------------|----------------------------|
| **Raw RAG baseline** | Current production: direct RAG endpoint, no agent wrapper | The thing we are replacing |
| **Agent, RAG-only** | New agent with MCP capabilities disabled via `ENABLED_CAPABILITIES=ask_question` | Isolates the value added by the LangGraph pipeline (classification, synthesis, scoped RAG) on top of the same RAG backend |
| **Agent, full capabilities** | New agent with all MCP capabilities enabled | The configuration we want to ship |

All three systems answer the same question batteries. All three are scored by the same LLM judge using the same rubric from Decision 005.

**Go/no-go criterion:** The full-capabilities agent must meet both of the following on each question battery:

1. **Aggregate:** Composite score ≥ raw RAG baseline composite score minus 0.15. (Small regressions are acceptable if they come with meaningful wins elsewhere; larger regressions block the launch.)
2. **No severe regressions:** No individual question where `baseline_score - agent_score > 1.5` on the composite scale. Individual regressions at this magnitude must be reviewed by a human via Argilla before launch.

If the agent is worse than baseline in aggregate by more than 0.15, or if any question has a severe regression, production launch is blocked pending investigation and remediation.

## Implementation

### Prerequisite: the capability registry runtime enforcement work

This comparison depends on the capability registry refactor described in [11-capability-registry.md](../active/11-capability-registry.md#capabilities-as-runtime-source-of-truth). Specifically, the `ENABLED_CAPABILITIES` env var must let us run the agent with only RAG capabilities enabled. Without that, we'd need a separate runner mode or code branch to produce the "Agent, RAG-only" configuration.

### Prerequisite: a raw-RAG runner mode for the eval pipeline

The eval pipeline currently only runs questions through the agent graph. We need a runner mode that calls the RAG service directly and returns a `RunResult` shaped the same way agent runs produce one. This is a small addition to `src/eval/runner.py`:

- Add `system: Literal["agent", "raw_rag"]` parameter to `run_question()`
- When `system="raw_rag"`: call the existing RAG client directly and return the raw response as the final answer, with no tool results and no node trace
- Add `--system` flag to `python -m src.eval run`
- Store the system label on the eval run record so `compare` can distinguish runs

### Question batteries

Use both existing batteries ([from the eval pipeline](./005-llm-judge-eval-pipeline.md)):

- `friendly_battery.json` — 50 well-phrased questions across capability areas. The cleaner signal.
- `real_user_battery.json` — 50 real user queries with typos and vague phrasing. The harder test where classification and disambiguation matter most.

Six runs total: three systems × two batteries.

### Report structure

Split the comparison into two views to pre-empt the "isn't that cherry-picking?" objection:

- **Static questions:** questions the RAG baseline has a chance of answering correctly because the answer is in the RAG corpus. Head-to-head comparison on roughly equal footing.
- **Dynamic questions:** questions that require live data the RAG corpus cannot have (current system status, specific allocation balances, recent events). The agent can answer these via MCP tools; the baseline cannot. These are agent-only wins and should be reported as a capability extension, not as part of the head-to-head aggregate score.

The existing eval report generator has per-category breakdowns; extend it to also report by classification type (`static` / `dynamic` / `combined`).

### Human calibration via Argilla

After the runs complete, push a curated subset to Argilla for human review:

- All severe regressions (`baseline - agent > 1.5`) — required before launch
- All severe wins (`agent - baseline > 1.5`) — sanity-check the judge, confirm wins are real
- A 10% random sample of the rest — catches systematic judge bias

Target: ~30 reviewed records, ~30 minutes of human time. The goal is not exhaustive review; it is calibrating the judge's aggregate ranking against human judgment. If the calibration confirms the aggregate story, the aggregate scores become the production-readiness evidence.

## Caveats and limitations

1. **Judge model is a variable, held constant across runs.** All three systems are scored by the same judge model (`gpt-4o-mini` pre-production, per [Decision 006](./006-onprem-llm-for-eval.md)). This means the comparison is valid even though the judge itself is an imperfect instrument — both sides of the comparison face the same imperfection. When the on-premise judge deploys, the comparison should be re-run with the new judge to confirm the ranking holds.

2. **LLM judges tend to prefer longer, more structured answers.** The agent synthesizes from multiple sources and tends to produce longer responses than the raw RAG baseline. This could bias the judge toward the agent independent of actual quality. Human calibration on severe wins is the check for this.

3. **The question batteries are small (100 questions total).** A 0.1 difference in composite score at n=50 is not statistically meaningful. This is why the aggregate threshold is 0.15 (noticeable but not demanding), and why individual question regressions are reviewed separately.

4. **Pre-production judge uses a cloud LLM.** [Decision 006](./006-onprem-llm-for-eval.md) requires on-premise LLM for scoring production traffic with real user queries. This comparison uses curated questions only — no real user data, no PII — so the cloud judge is acceptable.

5. **Dynamic questions are not in the aggregate score.** Questions requiring live MCP data cannot be answered by the raw RAG baseline at all. Including them in the head-to-head would artificially inflate the agent's score. They are reported separately as "capability extensions the new system provides that the old system cannot."

## Alternatives considered

- **Compare new agent to itself across commits (no baseline).** This is what the existing eval pipeline supports. It answers "did this change regress quality?" but not "is the new agent better than what we're replacing?" Leadership specifically asked the second question.

- **Compare using production logs from both systems.** Would require running both systems in parallel on real traffic for some period, with the new system shadow-scoring but not user-facing. Richer signal, but privacy-constrained (real user queries can't be sent to cloud judge per Decision 006), slower (need weeks of traffic), and adds production operational burden. Curated batteries are good enough for a go/no-go decision; production shadow testing can happen post-launch as a second-phase validation.

- **Skip the baseline comparison, rely on leadership trust.** The eval pipeline has been running; the agent is in active use in pre-production; Joe has been testing it against his own battery. This path works if leadership trusts the team's judgment. The comparison makes that trust auditable and gives reviewers something concrete to point at.

## Success definition

This decision is implemented successfully when:

1. Three-way comparison has been run against both batteries
2. Results meet the go/no-go criterion above, or failing questions have been remediated
3. Human calibration via Argilla has validated the judge's aggregate ranking
4. A written comparison report exists in a form suitable for ACCESS leadership review
5. Leadership has reviewed the report and approved production launch

## See also

- [Decision 005: LLM-as-Judge Eval Pipeline](./005-llm-judge-eval-pipeline.md)
- [Decision 006: On-Premise LLM for Production Eval](./006-onprem-llm-for-eval.md)
- [Capability Registry: Runtime Source of Truth](../active/11-capability-registry.md#capabilities-as-runtime-source-of-truth) (enables the RAG-only agent run)
- [Eval Pipeline Spec](https://github.com/necyberteam/access-agent/blob/main/docs/superpowers/specs/2026-03-31-eval-pipeline-design.md) (implementation reference)
