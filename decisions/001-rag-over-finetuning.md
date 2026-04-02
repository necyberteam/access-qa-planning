# Decision 001: RAG over Fine-Tuning

**Date:** 2025-12
**Status:** Accepted
**Decided by:** Team

## Context

We initially planned to fine-tune an LLM on ACCESS documentation to create a specialized model that could answer user questions. A pilot was conducted with fine-tuned models trained on curated Q&A pairs.

## Decision

Abandoned fine-tuning in favor of RAG (Retrieval-Augmented Generation).

## Rationale

- Fine-tuned models didn't reliably retain factual details
- Worse, they hallucinated additional details around what they did memorize — mixing real and fabricated information
- RAG retrieves verified answers at query time — no hallucination risk from the retrieval itself
- RAG is easier to update: add new Q&A pairs without retraining

## Alternatives Considered

- **Fine-tuning with more data** — More data didn't fix the hallucination problem; the model pattern-matched rather than memorized
- **Hybrid (fine-tuned + RAG)** — Added complexity without clear benefit over RAG alone

## See Also

- [Model Training (archived)](../archive/04-model-training.md) — detailed record of the fine-tuning experiments
