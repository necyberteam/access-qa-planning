# Decision 005: LLM-as-Judge Eval Pipeline

**Date:** 2026-03-31
**Status:** Accepted
**Decided by:** Drew, Vikram (design philosophy)

## Context

The agent is deploying to production with new capabilities (UKY-first architecture, MCP tool calling, domain agents). There was no systematic way to evaluate answer quality beyond mechanical checks (tool selection, keyword presence) and operational metrics (success rates, response times).

Stakeholders need quality reporting at three levels: team (weekly detail), program leadership (monthly narrative), and RP operators (per-resource view).

## Decision

Build an LLM-as-judge evaluation pipeline with human review via Argilla for calibration.

## Key Design Choices

- **5-dimension rubric** — correctness (30%), completeness (25%), relevance (20%), citation quality (15%), hedging (10%). Same rubric for judge and human reviewers.
- **Judge evaluates synthesis quality, not source quality** — If UKY has outdated docs and the agent faithfully reports them, that's correct. Data quality is a separate concern.
- **Argilla for human review** — Judge scores appear as suggestions; humans accept or override. Divergence data tunes the judge prompt.
- **On-premise LLM required for production scoring** — Privacy policy doesn't cover sending real user queries to external LLMs. UKY GPU infrastructure available.
- **Small modular tools** (Vikram's principle) — Eval module in access-agent, Argilla as separate service, interact through clear interfaces.
- **Branch-scoped Argilla datasets** — Created per eval run, deleted on branch merge. Production dataset is persistent.

## Alternatives Considered

- **Gold-standard expected answers** — Go stale fast as docs change; judge-against-sources is more durable
- **Extend existing student review tool** — Don't have access to codebase; building a new interface (Argilla) was faster and more modular
- **Scoring production answers first** — Blocked by privacy; pre-production eval gives immediate value

## See Also

- Spec: `access-agent/docs/superpowers/specs/2026-03-31-eval-pipeline-design.md`
- Implementation: `access-agent/src/eval/` and `access-agent/eval/README.md`
- [Decision 006: On-Premise LLM for Production Eval](./006-onprem-llm-for-eval.md)
- [Decision 007: Production Baseline Comparison Eval](./007-production-baseline-comparison.md) — uses this pipeline for the new-agent-vs-current-production comparison
