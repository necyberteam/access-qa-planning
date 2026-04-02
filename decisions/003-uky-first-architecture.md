# Decision 003: UKY-First Architecture

**Date:** 2026-03
**Status:** Accepted
**Decided by:** Joe Bacal, based on eval battery results

## Context

The agent classified queries as static, dynamic, or combined, and only consulted UKY RAG for static/combined queries. Dynamic queries (e.g., "Is Delta down?") skipped RAG entirely and went straight to MCP tool planning. This meant dynamic queries had no fallback content if tools failed.

## Decision

Always consult UKY RAG regardless of query classification. Classification determines what happens *with* the RAG data, not whether to skip it.

- Static queries: RAG answer served directly
- Dynamic/combined queries: RAG and tool planning run in parallel (no latency penalty)
- Domain queries: RAG context available as fallback if domain tools fail

## Rationale

- UKY often has useful context even for "purely dynamic" questions
- Parallel execution (RAG + plan via asyncio.gather) eliminates the latency concern
- When tools fail or return empty, UKY content is available for synthesis instead of a dead end
- Eval battery showed improvement: 18/18 questions pass, zero length failures

## Key Implementation Details

- `rag_answer_node` defaults to "general" endpoint when classifier sets `rag_endpoint=null`
- Circuit breaker skips pointless tool retries when all results are empty/failed
- Direct UKY serving (without LLM rewrite) when tools add no value

## See Also

- PR #1 in access-agent — full implementation
