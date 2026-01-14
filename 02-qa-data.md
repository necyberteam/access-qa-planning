# Q&A Data Preparation

> **Part of the Data Pipeline**: [Agent Architecture](./01-agent-architecture.md) → This doc → [Review System](./03-review-system.md)

## Overview

Q&A data comes from three sources, combined into a unified dataset for RAG retrieval:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Q&A DATA SOURCES                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐           │
│  │  MCP EXTRACTED  │   │   USER Q&A DB   │   │  DOCS Q&A       │           │
│  │                 │   │                 │   │                 │           │
│  │  Structured     │   │  Real user      │   │  LLM-generated  │           │
│  │  from API data  │   │  questions      │   │  from PDFs      │           │
│  └────────┬────────┘   └────────┬────────┘   └────────┬────────┘           │
│           │                     │                     │                     │
│           └──────────────┬──────┴─────────────────────┘                     │
│                          ▼                                                  │
│               ┌─────────────────────┐                                       │
│               │   DEDUPLICATION &   │                                       │
│               │   QUALITY FILTER    │                                       │
│               └──────────┬──────────┘                                       │
│                          ▼                                                  │
│               ┌─────────────────────┐                                       │
│               │  ARGILLA REVIEW     │                                       │
│               └──────────┬──────────┘                                       │
│                          ▼                                                  │
│               ┌─────────────────────┐                                       │
│               │  QA SERVICE (RAG)   │                                       │
│               └─────────────────────┘                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why Q&A Pairs for RAG

Q&A pairs are ideal for retrieval because:

| Benefit | Why It Matters |
|---------|----------------|
| **Question-aware retrieval** | Embed the question, not the answer - matches how users ask |
| **Complete answers** | Each pair has a verified, human-approved answer ready to serve |
| **Citations included** | Answers include source markers for attribution |
| **Curated quality** | Every pair reviewed in Argilla before entering the database |

### Architecture Note

> **Historical context**: This document was originally "Training Data Preparation" for fine-tuning a model. After pilot experiments showed the fine-tuned model didn't reliably retain facts and hallucinated details around what it did memorize, we pivoted to RAG-primary architecture. The same Q&A pairs are now used for semantic retrieval instead of model training. See [Model Training (Deprecated)](./04-model-training.md) for historical context.

---

## Source 1: MCP Extraction

### Which Servers to Extract From

**Static Data (for RAG retrieval)**:

| Server | Update Frequency | Priority |
|--------|------------------|----------|
| compute-resources | Monthly | **HIGH** |
| software-discovery | Weekly | **HIGH** |
| allocations | Weekly | MEDIUM |
| nsf-awards | Weekly | MEDIUM |
| affinity-groups | Monthly | LOW |

**Dynamic Data (KEEP AS LIVE MCP)** - Do NOT create Q&A pairs:

| Server | Why Keep Live |
|--------|---------------|
| system-status | Real-time outages |
| events | Schedules change frequently |
| announcements | Time-sensitive |
| xdmod-charts | Always need current metrics |

### Q&A Templates by Domain

#### Compute Resources

| Question Type | Examples |
|---------------|----------|
| **Simple Factual** | "What GPUs does {resource} have?", "How many nodes does {resource} have?" |
| **Discovery** | "What compute resources are available?", "Which resources have GPUs?" |
| **Comparison** | "Compare {resource_a} and {resource_b}" |
| **Recommendation** | "What resource should I use for machine learning?" |

#### Software Discovery

| Question Type | Examples |
|---------------|----------|
| **Availability** | "Is {software} available on ACCESS?", "What version of {software} is on {resource}?" |
| **Resource-Specific** | "What software is available on {resource}?" |
| **Category-Based** | "What bioinformatics software is available?" |

#### Allocations

- "What fields of science use ACCESS most?"
- "What {field} projects are on ACCESS?"
- "Find projects about {topic}"

---

## Source 2: Existing User Q&A Database

### What We Have
- Q&A pairs from production RAG system
- Staff-labeled "good" / "bad" quality ratings
- Real user questions (authentic distribution)

### Cleanup Tasks

| Task | Purpose |
|------|---------|
| Filter "good" only | Keep only staff-approved answers |
| Check freshness | Flag answers that may be outdated |
| Deduplicate | Remove near-duplicate questions |
| Normalize citations | Convert to standard source marker format |

### Quality Criteria

A Q&A pair is "good" if:
- ✅ Staff marked as "good"
- ✅ Answer is factually accurate (spot-check)
- ✅ Question is clear and specific
- ✅ Not a duplicate of another pair

---

## Source 3: Documentation Q&A

Generate Q&A pairs from existing ACCESS documentation (PDFs, web pages, guides).

### Generation Process

1. **Chunk documents** into logical sections
2. **LLM generates Q&A** for each section - questions a user might ask, with answers citing the source
3. **Human reviews** generated pairs in Argilla before adding to database

### Document Categories

| Category | Example Docs |
|----------|--------------|
| Getting Started | Account setup, first allocation |
| User Guides | Resource-specific guides |
| FAQs | Existing FAQ pages (check for duplicates) |
| Technical | Job submission, data transfer |

---

## Deduplication Strategy

Multiple sources generate overlapping questions. Deduplication prevents retrieval of near-duplicates.

### Process

1. **Exact match removal** - Hash-based duplicate detection
2. **Semantic clustering** - Group similar questions via embeddings (threshold ~0.85)
3. **Selection from clusters** - Keep best answer from each cluster

### Priority Rules

When choosing between duplicates:

| Priority | Source | Rationale |
|----------|--------|-----------|
| 1 | User Q&A | Real questions, highest value |
| 2 | Doc Q&A | Covers documentation gaps |
| 3 | MCP Q&A | Structured but formulaic |

Additional factors: more complete answer > shorter; better citations > missing; more recent > older.

---

## Q&A Pair Format

Each Q&A pair includes:

| Field | Description |
|-------|-------------|
| question | The user question |
| answer | Complete answer with citation markers |
| domain | Category (e.g., `compute-resources`, `software-discovery`) |
| entity_id | Specific resource/entity the Q&A is about |
| metadata | Source type, creation date, complexity level |

### Citation Format

Answers include source markers that the agent converts to clickable links. Format: `<<SRC:domain:entity_id>>`

---

## Automated Refresh Pipeline

Data sources update at different frequencies. Automated extraction keeps Q&A data current.

### Refresh Cadences

| Source | Cadence | Human Review Trigger |
|--------|---------|---------------------|
| compute-resources | Weekly | New resource added |
| software-discovery | Weekly | >50 new packages |
| allocations | Weekly | >100 new projects |
| nsf-awards | Bi-weekly | >50 new awards |
| affinity-groups | Monthly | New group added |
| user Q&A | Daily | Always queue new |
| documentation | On change | Any doc update |

### Change Detection

For each extraction:
1. Compare new data vs cached data
2. Identify new, modified, and deleted entities
3. Generate Q&A only for changed entities
4. Queue for human review in Argilla

### Argilla → QA Service Sync

After Q&A pairs are approved in Argilla:
1. Sync runs on schedule (or triggered)
2. Approved pairs are embedded for semantic search
3. Pairs are upserted to vector database
4. New pairs are immediately available for retrieval

---

## Domain Tagging

Each Q&A pair is tagged with a domain. This enables:

| Use Case | Benefit |
|----------|---------|
| Routing to reviewers | Different teams review different domains |
| Coverage analysis | Identify gaps across data sources |
| Filtered retrieval | Search within specific domains |
| Evaluation slicing | Measure accuracy per domain |

### Taxonomy

Domains align with MCP servers and data sources:

| Category | Domains |
|----------|---------|
| MCP servers | compute-resources, software-discovery, allocations, affinity-groups, nsf-awards |
| Documentation | docs-getting-started, docs-user-guides, docs-technical |

---

## Summary

| Source | Priority | Notes |
|--------|----------|-------|
| MCP extraction | High | Structured, reliable |
| User Q&A database | Highest | Real questions |
| Documentation | Medium | Covers gaps |

Q&A pairs flow through Argilla review, then sync to access-qa-service for RAG retrieval.

→ *Next in pipeline: [Review System](./03-review-system.md)* - Human review before deployment
