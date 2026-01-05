# ACCESS QA System Planning

> **Version**: 0.1.0 (Planning)

Planning documentation for the ACCESS-CI intelligent documentation agent.

## What We're Building

An AI-powered question-answering system for [ACCESS](https://access-ci.org) (Advanced Cyberinfrastructure Coordination Ecosystem: Services & Support). ACCESS allocates computing resources from Resource Providers—supercomputers, cloud platforms, and storage systems—to researchers across the US.

Users ask questions like:
- "What GPUs does Delta have?"
- "How do I request an allocation?"
- "Is Expanse currently down?"

This tool answers those questions accurately, with citations to source data.

## Executive Summary

**Current State**: A RAG-based QA system trained on PDFs and documentation. It answers questions but can become stale and lacks access to live system data.

**Problem**:
- Static knowledge goes stale between retraining
- No access to real-time data (outages, current events, user allocations)
- MCP servers exist but aren't integrated into the QA system

**Solution**: An intelligent agent that routes queries to the optimal handler:
- Static questions (hardware specs, documentation) → fine-tuned model with baked-in knowledge
- Dynamic questions (outages, user allocations) → live MCP API calls
- Combined questions → both working together

**Key Benefits**:
- Fresh answers for dynamic data via MCP integration
- Faster responses for static questions via fine-tuned model
- Maintained citation capability
- Human review ensures quality before and after training

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
      │ "What GPUs does  │ │ "What GPU        │ │ "Is Delta down?" │
      │  Delta have?"    │ │  resources are   │ │                  │
      │                  │ │  available now?" │ │  DYNAMIC         │
      │  STATIC          │ │  COMBINED        │ │                  │
      └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
               │                    │                    │
               ▼                    ▼                    ▼
      ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
      │ Fine-tuned model │ │ Model + live MCP │ │ Live MCP call    │
      │ (fast, cached)   │ │ (comprehensive)  │ │ (real-time)      │
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

## Goal

Replace the current RAG LLM with an intelligent agent system:
- **Query Classification**: Route queries to the right handler (static vs dynamic)
- **Fine-Tuned Model**: Handles static queries (baked-in knowledge from MCP data + docs)
- **Live MCP Calls**: Handles dynamic queries (outages, events, metrics, user-specific data)
- **Citations Preserved**: Maintain source links users rely on
- **Action Tools**: Future authenticated operations (create events, etc.)

## Documents

| # | Document | Purpose | Key Sections |
|---|----------|---------|--------------|
| 01 | [agent-architecture.md](./01-agent-architecture.md) | System design + roadmap | Architecture, phases, success metrics, data governance |
| 02 | [training-data.md](./02-training-data.md) | Data preparation | MCP extraction, Q&A templates, deduplication |
| 03 | [review-system.md](./03-review-system.md) | Human review (Argilla) | Pre-training approval, post-deployment feedback, domain reviewers |
| 04 | [model-training.md](./04-model-training.md) | Model & infrastructure | GH200 setup, model selection, pilot comparison |
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

### Data Pipeline (extraction → review → training)
The data pipeline docs describe a continuous flow:
1. [02-training-data.md](./02-training-data.md) - Sources, extraction, Q&A generation
2. [03-review-system.md](./03-review-system.md) - Human review (before training + after deployment)
3. [04-model-training.md](./04-model-training.md) - Training the fine-tuned model

### Action Tools (MCP Write Operations)

For AI agents to take actions on behalf of users:
1. [05-events-actions.md](./05-events-actions.md) - Overview: phased approach, key patterns
2. [06-mcp-authentication.md](./06-mcp-authentication.md) - OAuth 2.1 authentication with CILogon
3. [07-backend-integration-spec.md](./07-backend-integration-spec.md) - Contract for backend API teams
4. [drupal-announcements-api-spec.md](./drupal-announcements-api-spec.md) - Phase 1: Drupal developer spec

## Current Status

- **Phase**: Planning / Pilot Preparation
- **Next Steps**:
  1. Export existing "good" Q&A pairs from production
  2. Run MCP extraction for compute-resources + software-discovery
  3. Set up training infrastructure on GH200
  4. Begin model pilot comparing architectures

## Related Repositories

- [access_mcp](https://github.com/access-ci-org/access_mcp) - MCP servers for ACCESS data
- n8n workflows - Query routing and orchestration
- cyberteam_drupal - Drupal integration for events

## Contributing

This is a planning repository. To propose changes:
1. Create a branch
2. Edit the relevant document(s)
3. Open a PR with a description of what changed and why
