# Decision 002: UKY RAG over pgvector Q&A Pairs

**Date:** 2026-02
**Status:** Accepted (pgvector on hold, not removed)
**Decided by:** Team, based on A.2 evaluation data

## Context

The system had two RAG backends:
- **pgvector Q&A pairs** — 683 curated pairs in PostgreSQL with vector similarity search, reviewed through Argilla
- **UKY Document RAG** — University of Kentucky's document-based RAG endpoint covering all ACCESS documentation

Both were run in parallel (A.2 dual-RAG logging) to compare quality on real user queries.

## Decision

Use UKY RAG as the primary (and currently only) RAG backend. pgvector Q&A pair lookup is disabled but the code and data remain intact for potential future use.

## Rationale

- UKY docs provided broader coverage — the 683 Q&A pairs were too narrow for the diversity of real user queries
- pgvector's 0.85 similarity threshold rarely matched real user input (messy phrasing, typos, pasted errors)
- UKY handled the long tail of questions that didn't have an exact Q&A pair match
- Maintaining curated Q&A pairs was labor-intensive relative to the coverage they provided

## What We Kept

- The pgvector functions and qa_client.py are intact in the codebase
- The Argilla Q&A curation pipeline still works
- Q&A pairs could be re-enabled for "slam-dunk" exact-match scenarios if needed

## See Also

- [Q&A Data Preparation (archived)](../archive/02-qa-data.md) — the original Q&A extraction and curation pipeline
- Commit `caf7256` — A.2 dual-RAG logging implementation
