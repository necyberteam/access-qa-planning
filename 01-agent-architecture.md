# ACCESS Documentation Agent

> **Related**: [Training Data](./02-training-data.md) | [Review System](./03-review-system.md) | [Model Training](./04-model-training.md) | [Events Actions](./05-events-actions.md)

## Overview

This is an AI-powered question-answering system for [ACCESS](https://access-ci.org) users. ACCESS (Advanced Cyberinfrastructure Coordination Ecosystem: Services & Support) allocates computing resources—supercomputers, cloud platforms, and storage systems—from Resource Providers to researchers across the US.

The agent combines a fine-tuned model with live MCP (Model Context Protocol) data access. This architecture reduces latency for common queries while maintaining real-time accuracy for dynamic data.

### Current State
- **In Production**: RAG LLM trained on PDFs and documentation (provides citations/links)

### Target State
An intelligent agent system where:
- **Query Router**: Classifies queries as static, dynamic, or combined
- **Fine-Tuned Model**: Handles static queries (baked-in knowledge from MCP data + docs)
- **Live MCP Calls**: Handles dynamic queries (outages, events, metrics, user-specific data)
- **Citations Preserved**: Maintain link/source capability users rely on
- **Feedback Loop**: User feedback flows back to improve training data
- **Action Tools**: Authenticated operations like event creation (future)

---

## Why This Architecture?

### The Problem

Current RAG-based system:
- Trained on PDFs and documentation - knowledge goes stale between updates
- No access to real-time data (outages, events, user allocations)
- MCP servers exist with live ACCESS data, but aren't integrated

### The Solution

Split queries into two paths:

| Query Type | Example | Handling |
|------------|---------|----------|
| **Static** | "What GPUs does Delta have?" | Fine-tuned model (baked-in knowledge) |
| **Dynamic** | "Is Delta currently down?" | Live MCP call |
| **Combined** | "What GPU resources are available and working?" | Model + MCP together |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER QUERY                                      │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           QUERY CLASSIFIER                                   │
│                                                                             │
│  Determines: STATIC | DYNAMIC | COMBINED                                    │
│  Based on: keywords, patterns, query structure                              │
└──────────┬──────────────────────┬──────────────────────┬────────────────────┘
           │                      │                      │
           ▼                      ▼                      ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│     STATIC       │   │    COMBINED      │   │     DYNAMIC      │
│                  │   │                  │   │                  │
│  Fine-tuned      │   │  Model + MCP     │   │  MCP only        │
│  model only      │   │  together        │   │                  │
└────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CITATION POST-PROCESSOR                               │
│                                                                             │
│  Extracts <<SRC:...>> markers → Looks up URLs → Formats links               │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RESPONSE                                        │
│                         (with clickable citations)                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Governance

What can be baked into the model vs. what must stay live:

| Data Type | Can Train Into Model? | Notes |
|-----------|----------------------|-------|
| Public resource specs (hardware, GPUs) | **Yes** | Core training data |
| Public software modules | **Yes** | Versions change - add freshness disclaimer |
| Public NSF awards | **Yes** | Public record |
| General ACCESS documentation | **Yes** | From RAG PDFs |
| Affinity group info | **Yes** | Stable |
| **Per-user allocations** | **NO** | Privacy/accuracy - always live |
| **Project-specific details** | **NO** | Privacy/accuracy - always live |
| **User SU balances** | **NO** | Must be real-time |
| System outages/status | **NO** | Real-time critical |
| Upcoming events | **NO** | Changes frequently |
| Usage metrics (XDMoD) | **NO** | Time-sensitive |

**Policy**: Never bake user-specific, project-specific, or sensitive allocation details into the model.

### Staleness Boundaries

How long can baked-in data be before it needs refresh or disclaimer?

| Data Type | Acceptable Staleness | Action When Exceeded |
|-----------|---------------------|---------------------|
| Hardware specs | 90 days | Retrain or add disclaimer |
| Software versions | 30 days | Add "verify current version" note |
| Documentation | 60 days | Check for updates |
| NSF awards | 90 days | Acceptable (public record) |
| Allocation metadata | 30 days | Retrain |

Model responses should include freshness context when relevant: *"As of [training date], Delta has..."*

---

## Query Classification

### Stage 1: Rule-Based (Deploy First)

Pattern matching for quick classification:

**Dynamic patterns** (→ live MCP):
- Time words: "currently", "right now", "today", "this week"
- Status words: "outage", "down", "maintenance", "status"
- User-specific: "my project", "my allocation", "my balance"
- Events: "upcoming", "next", "scheduled"

**Static patterns** (→ fine-tuned model):
- Factual: "what is", "describe", "explain", "how does"
- Specs: "hardware", "specs", "specifications", "capabilities"
- Documentation: "how do I", "guide", "tutorial"

**Default**: When uncertain, use COMBINED (safe fallback)

### Stage 2: Learned Classifier (Future)

Train a classifier on production query logs:
- Input: query text
- Output: STATIC / DYNAMIC / COMBINED probability
- Retrain periodically as patterns evolve

### Confidence-Aware Routing

The fine-tuned model may produce low-confidence answers for edge cases or stale data. When this happens, validate against live MCP:

```
User Query
    │
    ▼
Query Classifier ──▶ STATIC
    │
    ▼
Fine-Tuned Model
    │
    ├── High Confidence ──▶ Return response
    │
    └── Low Confidence ──▶ Validate with MCP ──▶ Return validated response
                                │
                                └── Log for review (possible drift)
```

**Low confidence indicators:**
- Hedging language: "I'm not sure", "might be", "possibly"
- Missing expected citations
- Token probability below threshold
- Query about recently updated data (software versions, etc.)

**Benefits:**
- Catches model drift early
- Maintains accuracy without always calling MCP
- Generates training signal for weak areas

---

## Component Details

### Fine-Tuned Model

Handles static/semi-static knowledge:

| Data Source | Update Frequency | Examples |
|-------------|------------------|----------|
| Compute resource specs | Monthly | GPU counts, memory, node types |
| Software packages | Weekly | Available modules, versions |
| Allocation metadata | Weekly | Public project info, fields of science |
| NSF awards | Weekly | Funding data, PI info |
| Documentation | On change | How-to guides, policies |

The model outputs `<<SRC:...>>` citation markers that link back to source data.

→ *Details: [Model Training](./04-model-training.md)*

### Live MCP Servers

Handle dynamic data that must be current:

| Server | Data Type | Why Live |
|--------|-----------|----------|
| system-status | Outages, maintenance | Real-time critical |
| events | Upcoming workshops | Schedules change |
| announcements | News, updates | Time-sensitive |
| xdmod-metrics | Usage statistics | Always current |
| allocations (user-specific) | Balances, project status | Per-user, real-time |

→ *Details: [Events Actions](./05-events-actions.md)* for authenticated action patterns

### n8n Orchestrator

Coordinates the workflow:

1. Receives query from QA Bot
2. Runs query classifier
3. Routes to appropriate handler(s)
4. Combines results if needed
5. Runs citation post-processor
6. Returns formatted response

### QA Bot (Frontend)

- Sends queries to n8n webhook
- Receives formatted responses with citations
- Collects user feedback (thumbs up/down)
- Passes user context for authenticated operations

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TRAINING PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MCP Extraction ──┐                                                         │
│                   │                                                         │
│  User Q&A DB ─────┼──▶ Review System ──▶ Approved Data ──▶ Model Training   │
│                   │    (human review)                                       │
│  Documentation ───┘                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PRODUCTION SYSTEM                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Query ──▶ Query Classifier ──▶ Fine-Tuned Model ──┐                   │
│                        │                                │                   │
│                        └──▶ Live MCP Servers ───────────┼──▶ Response       │
│                                                         │                   │
│                              Citation Post-Processor ◀──┘                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FEEDBACK LOOP                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Feedback (thumbs up/down) ──▶ Review System ──▶ Training Data Update  │
│                                                                             │
│  Drift Detection (model vs MCP) ──▶ Retrain Trigger                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Success Metrics

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Answer correctness** | ≥95% vs ground-truth | Human eval on golden set |
| **MCP call reduction** | 40-60% fewer calls for static queries | API call count per query |
| **Citation coverage** | ≥90% of answers include valid citation | Automated + spot check |
| **Static query latency** | P95 < 2s end-to-end | Response time tracking |
| **Query routing accuracy** | ≥95% static/dynamic classification | Classification metrics |
| **User satisfaction** | ≥80% positive feedback | Thumbs up/down in UI |

---

## Implementation Phases

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           IMPLEMENTATION PHASES                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  PHASE 0        PHASE 1          PHASE 2          PHASE 3          PHASE 4     │
│  Baseline &     Data Pipeline    Model Training   Integration      Production   │
│  Evaluation     + Review UI                       + n8n            & Monitoring │
│                                                                                 │
│  ┌─────────┐   ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌─────────┐  │
│  │ Golden  │   │ Extract  │     │ Train &  │     │ Query    │     │ Deploy  │  │
│  │ Eval    │ → │ MCP Data │  →  │ Evaluate │  →  │ Router + │  →  │ Monitor │  │
│  │ Set     │   │ + Review │     │ Model    │     │ MCP Live │     │ Iterate │  │
│  └─────────┘   └──────────┘     └──────────┘     └──────────┘     └─────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Phase 0: Baseline & Evaluation

**Goal**: Create evaluation framework and understand current system.

- [ ] Document current RAG citation format
- [ ] Create golden evaluation set (150-200 queries)
- [ ] Export existing "good" user Q&A pairs

→ *Details: [Training Data](./02-training-data.md)*

### Phase 1: Data Pipeline + Review UI

**Goal**: Set up data extraction and human review system.

- [ ] Build MCP extraction pipeline
- [ ] Deploy review system web UI
- [ ] Begin collecting approved Q&A pairs

→ *Details: [Training Data](./02-training-data.md), [Review System](./03-review-system.md)*

### Phase 2: Model Training

**Goal**: Train and evaluate fine-tuned model.

- [ ] Set up GH200 training environment
- [ ] Run pilot comparing model architectures
- [ ] Train production model on approved data

→ *Details: [Model Training](./04-model-training.md)*

### Phase 3: Integration

**Goal**: Build query router and connect all components.

- [ ] Implement query classifier in n8n
- [ ] Connect fine-tuned model to workflow
- [ ] Add citation post-processor
- [ ] Add confidence-aware fallback

### Phase 4: Production & Monitoring

**Goal**: Deploy, monitor, and maintain.

- [ ] Run golden eval set through new system
- [ ] Deploy to staging, then production
- [ ] Set up drift detection
- [ ] Establish retraining pipeline

---

## Pilot

Before full implementation, validate with minimal data.

### Pilot Goals
1. Test whether fine-tuning works for ACCESS domain
2. Compare MoE vs Dense architectures
3. Validate before building full infrastructure

### Pilot Data

| Source | Notes |
|--------|-------|
| Existing user Q&A ("good" labeled) | Real questions from production - highest value |
| MCP extraction (compute-resources + software-discovery) | Structured data from APIs |
| Documentation Q&A | LLM-generated from key docs |
| Negative/refusal examples | Teach model when to defer to live MCP |

Target: enough data to validate fine-tuning approach (exact counts TBD after extraction).

### Pilot Success Criteria

- [ ] Model learns ACCESS-CI domain knowledge
- [ ] Citations appear in >80% of responses
- [ ] Refusal behavior triggers for dynamic queries
- [ ] Inference latency <2s on GH200

### Pilot Baseline Comparison

To validate the approach, compare fine-tuned model against:

| Baseline | What It Tests |
|----------|---------------|
| Current RAG system | Is fine-tuning actually better? |
| Raw base model (no fine-tuning) | How much does domain training help? |
| LLM + live MCP tools | What's the latency/accuracy tradeoff? |

**LLM + live MCP tools**: Use Claude (or equivalent) with MCP tool calls for every query. This is essentially the current approach - always hitting the APIs. Shows the latency cost we're trying to reduce with baked-in knowledge.

Run golden eval set against all three. Fine-tuned model should beat each baseline on relevant metrics (accuracy for RAG, latency for MCP-only).

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Model quality insufficient | Pilot compares architectures; iterate on data |
| GH200 memory tight | Start with smaller model, batch_size=1 |
| Citation accuracy problems | Tokenization checks; post-processor validation |
| Query classifier mistakes | Default to COMBINED (safe); confidence fallback |
| Model gives stale answers | Drift monitoring; freshness disclaimers |
| Privacy concerns | Data governance matrix; never bake user-specific data |

### Fallback Hierarchy

When components fail, degrade gracefully:

```
1. Fine-tuned model + MCP (optimal)
        │
        ▼ (model unavailable)
2. Live MCP only (slower, but accurate)
        │
        ▼ (MCP unavailable)
3. Cached MCP responses (stale but functional)
        │
        ▼ (cache miss)
4. Base LLM with disclaimer ("I don't have current ACCESS data")
```

Each level maintains citation capability where possible.

---

## Open Questions

### Technical
1. Model serving infrastructure: vLLM vs TGI?
2. Learned classifier: Phase 3 or defer?
3. Query classifier accuracy target: 95%? 99%?

### Policy
4. Who approves pushing new model to production?

---

## Future: Authenticated Actions

The architecture supports authenticated operations:

```
User (logged in) ──▶ QA Bot ──▶ n8n ──▶ Events MCP ──▶ Drupal API
                        │
                        └── JWT token passed for user identity
```

→ *Details: [Events Actions](./05-events-actions.md)* - Pilot for AI agents to take actions on behalf of users
