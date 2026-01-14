# ACCESS QA System Planning

> **Version**: 0.2.0 (RAG-Primary Architecture)

Planning documentation for the ACCESS-CI intelligent documentation agent.

## What We're Building

An AI-powered question-answering system for [ACCESS](https://access-ci.org) (Advanced Cyberinfrastructure Coordination Ecosystem: Services & Support). ACCESS allocates computing resources from Resource Providers—supercomputers, cloud platforms, and storage systems—to researchers across the US.

Users ask questions like:
- "What GPUs does Delta have?"
- "How do I request an allocation?"
- "Is Expanse currently down?"

This tool answers those questions accurately, with citations to source data.

## Executive Summary

The ACCESS QA system is an intelligent agent that answers researcher questions about computing resources, allocations, and system status. It classifies each query and routes it to the appropriate handler: factual questions about resource specs and documentation are answered from a database of human-verified Q&A pairs, while questions about real-time data like outages or user allocations are answered via live API calls to MCP servers. All responses include citations linking back to source data.

The system is built on three main components: the access-agent (LangGraph) handles query classification and response synthesis, access-qa-service provides RAG retrieval from curated Q&A pairs stored in PostgreSQL with pgvector, and 10 MCP servers provide real-time access to ACCESS APIs. Human reviewers curate Q&A pairs through Argilla before they enter the system. Future phases will add authenticated actions, allowing users to create announcements and events conversationally.

## User Journey

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          USER ASKS A QUESTION                               │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                 ┌────────────────────┼────────────────────┐
                 │                    │                    │
                 ▼                    ▼                    ▼
      ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
      │ "What GPUs does  │ │ "What GPUs does  │ │ "Is Delta down?" │
      │  Delta have?"    │ │  Delta have and  │ │                  │
      │                  │ │  is it running?" │ │  DYNAMIC         │
      │  STATIC          │ │  COMBINED        │ │                  │
      └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
               │                    │                    │
               ▼                    ▼                    ▼
      ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
      │ RAG retrieval    │ │ RAG + live MCP   │ │ Live MCP call    │
      │ (verified Q&A)   │ │ (comprehensive)  │ │ (real-time)      │
      └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
               │                    │                    │
               └────────────────────┼────────────────────┘
                                    │
                                    ▼
      ┌─────────────────────────────────────────────────────────────┐
      │              RESPONSE WITH CITATIONS                         │
      │  "Delta has 4x NVIDIA A100 GPUs per node [source link]"     │
      └─────────────────────────────────────────────────────────────┘
```

## System Architecture

```
┌─────────────────┐
│  access-agent   │  LangGraph orchestration
│  (Python)       │  Query classification → routing → synthesis
└────────┬────────┘
         │ HTTP
         ▼
┌─────────────────┐     ┌─────────────────┐
│  QA Service     │     │  MCP Servers    │
│  (FastAPI)      │     │  (TypeScript)   │
│  pgvector RAG   │     │  10 servers     │
└────────┬────────┘     └────────┬────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│  PostgreSQL     │     │  ACCESS APIs    │
│  + pgvector     │     │  (live data)    │
└─────────────────┘     └─────────────────┘
         ▲
         │ sync
┌─────────────────┐
│    Argilla      │  Human review
│  (Q&A curation) │
└─────────────────┘
```

## Documents

| # | Document | Purpose | Key Sections |
|---|----------|---------|--------------|
| 01 | [agent-architecture.md](./01-agent-architecture.md) | System design + roadmap | Architecture, phases, success metrics, data governance |
| 02 | [02-qa-data.md](./02-qa-data.md) | Q&A data preparation | MCP extraction, Q&A templates, deduplication |
| 03 | [review-system.md](./03-review-system.md) | Human review (Argilla) | Pre-deployment approval, post-deployment feedback, domain reviewers |
| 04 | [model-training.md](./04-model-training.md) | Model training (deprecated) | Historical reference for fine-tuning approach |
| 05 | [events-actions.md](./05-events-actions.md) | MCP action tools | Announcements (Phase 1), Events (Phase 2) |
| 06 | [mcp-authentication.md](./06-mcp-authentication.md) | Authentication architecture | OAuth 2.1, CILogon proxy, token strategy |
| 07 | [backend-integration-spec.md](./07-backend-integration-spec.md) | Backend API contract | Service tokens, X-Acting-User, authorization patterns |
| 08 | [observability.md](./08-observability.md) | Distributed tracing & monitoring | Grafana Cloud, OpenTelemetry, dashboards |

### Implementation Specs

| Document | Purpose |
|----------|---------|
| [drupal-announcements-api-spec.md](./drupal-announcements-api-spec.md) | Drupal API spec for Announcements (Phase 1 pilot) |

## Reading Paths

### Executive Overview
Start with [01-agent-architecture.md](./01-agent-architecture.md) - covers the full system design and implementation phases.

### Data Pipeline (extraction → review → RAG)
The data pipeline docs describe a continuous flow:
1. [02-qa-data.md](./02-qa-data.md) - Sources, extraction, Q&A generation
2. [03-review-system.md](./03-review-system.md) - Human review via Argilla
3. Q&A pairs sync to access-qa-service for RAG retrieval

### Action Tools (MCP Write Operations)

For AI agents to take actions on behalf of users:
1. [05-events-actions.md](./05-events-actions.md) - Overview: phased approach, key patterns
2. [06-mcp-authentication.md](./06-mcp-authentication.md) - OAuth 2.1 authentication with CILogon
3. [07-backend-integration-spec.md](./07-backend-integration-spec.md) - Contract for backend API teams
4. [drupal-announcements-api-spec.md](./drupal-announcements-api-spec.md) - Phase 1: Drupal developer spec

## Current Status

- **Phase**: Production / Continuous Improvement
- **Completed**:
  - RAG-primary architecture implemented and deployed
  - access-qa-service running with pgvector + HNSW indexing
  - access-agent with query classification (static/dynamic/combined)
  - 10 MCP servers deployed for live data access
  - Argilla integration for human review
- **Key Learnings**:
  - Fine-tuned models didn't reliably retain facts
  - Worse, they hallucinated details around what they did memorize
  - RAG retrieves verified answers - no hallucination risk
- **Next Steps**:
  1. Expand Q&A pairs to cover more documentation
  2. Implement authenticated actions (Phase 2)

## Related Repositories

| Repository | Description |
|------------|-------------|
| [access-agent](https://github.com/necyberteam/access-agent) | LangGraph agent with RAG + MCP integration |
| [access-qa-service](https://github.com/necyberteam/access-qa-service) | FastAPI service for Q&A retrieval (pgvector) |
| [access-mcp](https://github.com/necyberteam/access_mcp) | MCP servers for ACCESS data (10 servers) |
| [access-qa-extraction](https://github.com/necyberteam/access-qa-extraction) | Q&A pair extraction from MCP servers |
| [access-qa-training](https://github.com/necyberteam/access-qa-training) | **DEPRECATED** - Fine-tuning pipeline (archived) |

## Contributing

This is a planning repository. To propose changes:
1. Create a branch
2. Edit the relevant document(s)
3. Open a PR with a description of what changed and why
