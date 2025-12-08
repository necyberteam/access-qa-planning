# ACCESS QA System Planning

> **Version**: 0.1.0 (Planning)

Planning documentation for the ACCESS-CI intelligent documentation agent - combining a fine-tuned model with live MCP (Model Context Protocol) data access.

## Executive Summary

**Problem**: The current RAG-based QA system has high latency for simple questions and makes repeated API calls for data that rarely changes.

**Solution**: An intelligent agent that routes queries to the optimal handler:
- Static questions (hardware specs, documentation) → fine-tuned model with baked-in knowledge
- Dynamic questions (outages, user allocations) → live MCP API calls
- Combined questions → both working together

**Key Benefits**:
- 40-60% fewer MCP calls for static queries
- Sub-2-second latency for common questions
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

## Document Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        01-agent-architecture.md                              │
│                                                                             │
│  System design + implementation roadmap                                     │
│  Query routing, components, phases, success metrics                         │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DATA PIPELINE                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ 02-training-data → 03-review-system → 04-model-training             │    │
│  │                                                                      │    │
│  │ Sources &         Human review        Train the                     │    │
│  │ extraction        (before & after)    fine-tuned model              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         05-events-crud                                       │
│                                                                             │
│  PoC for authenticated action tools                                         │
│  Auth patterns, CRUD via conversation                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Documents

| # | Document | Purpose | Key Sections |
|---|----------|---------|--------------|
| 01 | [agent-architecture.md](./01-agent-architecture.md) | System design + roadmap | Architecture, phases, success metrics, data governance |
| 02 | [training-data.md](./02-training-data.md) | Data preparation | MCP extraction, Q&A templates, deduplication |
| 03 | [review-system.md](./03-review-system.md) | Human review | Pre-training approval, post-deployment feedback, Web UI |
| 04 | [model-training.md](./04-model-training.md) | Model & infrastructure | GH200 setup, model selection, PoC comparison |
| 05 | [events-crud.md](./05-events-crud.md) | PoC for action tools | Auth patterns, CRUD via conversation |

## Reading Paths

### Executive Overview
Start with [01-agent-architecture.md](./01-agent-architecture.md) - covers the full system design and implementation phases.

### Data Pipeline (extraction → review → training)
The data pipeline docs describe a continuous flow:
1. [02-training-data.md](./02-training-data.md) - Sources, extraction, Q&A generation
2. [03-review-system.md](./03-review-system.md) - Human review (before training + after deployment)
3. [04-model-training.md](./04-model-training.md) - Training the fine-tuned model

### Future: Action Tools
[05-events-crud.md](./05-events-crud.md) - PoC for authenticated operations via MCP tools

## Current Status

- **Phase**: Planning / PoC Preparation
- **Next Steps**:
  1. Export existing "good" Q&A pairs from production
  2. Run MCP extraction for compute-resources + software-discovery
  3. Set up training infrastructure on GH200
  4. Begin model PoC comparing architectures

## Related Repositories

- [access_mcp](https://github.com/access-ci-org/access_mcp) - MCP servers for ACCESS data
- n8n workflows - Query routing and orchestration
- cyberteam_drupal - Drupal integration for events

## Contributing

This is a planning repository. To propose changes:
1. Create a branch
2. Edit the relevant document(s)
3. Open a PR with a description of what changed and why
