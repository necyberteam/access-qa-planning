# Decision 006: On-Premise LLM for Production Eval

**Date:** 2026-03-31
**Status:** Planned (not yet deployed)
**Decided by:** Drew

## Context

The eval pipeline needs to send real user queries + agent answers to an LLM judge for quality scoring. The current privacy policy doesn't cover sending user queries to external LLM APIs (OpenAI, Anthropic). User queries can contain PII (names, emails, account IDs).

## Decision

Deploy an on-premise LLM on UKY GPU infrastructure for production eval scoring. Pre-production eval (curated questions, no PII) can use cloud LLMs.

## Key Details

- **Infrastructure**: UKY has GPU capacity available via SSH
- **Recommended setup**: vLLM or Ollama serving Llama 3.1 70B-Instruct
- **Interface**: OpenAI-compatible API endpoint — the judge module just swaps `EVAL_JUDGE_BASE_URL`
- **Phasing**:
  1. Immediate: Cloud LLM for pre-production eval (curated questions only)
  2. Near-term: Deploy on-premise LLM, switch production scoring to it
  3. Optional: Use on-premise for all eval for consistency

## Alternatives Considered

- **Update privacy policy** — Possible but involves legal review and policy approval process
- **PII redaction before sending** — Adds complexity, may miss edge cases, reduces context quality for the judge
- **Skip production scoring** — Loses the most valuable signal (real user query quality)

## See Also

- Decision 005 (eval pipeline design)
- Spec section: "On-Premise LLM Infrastructure" in `access-agent/docs/superpowers/specs/2026-03-31-eval-pipeline-design.md`
