# Training Data Preparation

> **Part of the Data Pipeline**: [Agent Architecture](./01-agent-architecture.md) → This doc → [Review System](./03-review-system.md) → [Model Training](./04-model-training.md)

## Overview

Training data comes from three sources, combined into a unified dataset:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TRAINING DATA SOURCES                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐           │
│  │  MCP EXTRACTED  │   │   USER Q&A DB   │   │  RAG DOCS Q&A   │           │
│  │                 │   │                 │   │                 │           │
│  │  ~1,300 pairs   │   │  ~500-1000      │   │  ~500-1000      │           │
│  │                 │   │  pairs          │   │  pairs          │           │
│  │  Structured     │   │                 │   │                 │           │
│  │  from API data  │   │  Real user      │   │  LLM-generated  │           │
│  │                 │   │  questions      │   │  from PDFs      │           │
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
│               │   UNIFIED DATASET   │                                       │
│               │   ~2,500-3,500      │                                       │
│               └─────────────────────┘                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Source 1: MCP Extraction

### Which Servers to Extract From

**Static Data (TRAIN INTO MODEL)**:

| Server | Tools | Update Frequency | Priority |
|--------|-------|------------------|----------|
| compute-resources | 4 | Monthly | **HIGH** |
| software-discovery | 5 | Weekly | **HIGH** |
| allocations | 9 | Weekly | MEDIUM |
| nsf-awards | 6+ | Weekly | MEDIUM |
| affinity-groups | 3+ | Monthly | LOW |

**Dynamic Data (KEEP AS LIVE MCP)** - Do NOT train these:

| Server | Why Keep Live |
|--------|---------------|
| system-status | Real-time outages |
| events | Schedules change frequently |
| announcements | Time-sensitive |
| xdmod-charts | Always need current metrics |

### Q&A Templates by Domain

#### Compute Resources (~340 pairs)

**Simple Factual**:
- "What GPUs does {resource_name} have?"
- "How many nodes does {resource_name} have?"
- "What is the memory per node on {resource_name}?"
- "What organization operates {resource_name}?"

**Discovery**:
- "What compute resources are available on ACCESS?"
- "Which ACCESS resources have GPUs?"
- "What resources does {organization} operate?"

**Comparison**:
- "Compare {resource_a} and {resource_b}"
- "Which has more GPUs, {resource_a} or {resource_b}?"

**Recommendation**:
- "What resource should I use for machine learning?"
- "Which system is best for large memory jobs?"

#### Software Discovery (~285 pairs)

**Availability**:
- "Is {software} available on ACCESS?"
- "Where can I find {software}?"
- "Is {software} available on {resource}?"
- "What version of {software} is on {resource}?"

**Resource-Specific**:
- "What software is available on {resource}?"
- "What ML frameworks are on {resource}?"

**Category-Based**:
- "What bioinformatics software is available?"
- "What visualization tools are available?"

#### Allocations (~66 pairs)

- "What fields of science use ACCESS most?"
- "What {field} projects are on ACCESS?"
- "Find projects about {topic}"

#### Negative Examples - Defer to Live (~400 pairs)

These train the model to recognize when NOT to answer from memory:

- "Is Delta currently down?" → "I need to check live system status..."
- "What events are happening this week?" → "Event schedules require live data..."
- "What's my current SU balance?" → "I cannot access user-specific information..."
- "Is my project TG-ABC123 still active?" → "Project details require live verification..."

### Expected Output from MCP Extraction

| Domain | Q&A Pairs |
|--------|-----------|
| compute-resources | ~340 |
| software-discovery | ~285 |
| allocations | ~66 |
| affinity-groups | ~60 |
| nsf-awards | ~125 |
| defer-to-live (negative) | ~400 |
| **Total** | **~1,276** |

---

## Source 2: Existing User Q&A Database

### What We Have
- Several thousand Q&A pairs from production RAG system
- Staff-labeled "good" / "bad" quality ratings
- Real user questions (authentic distribution)

### Cleanup Tasks

1. **Filter "good" only** - Keep only staff-approved answers
2. **Check freshness** - Flag answers that may be outdated (resource specs, software versions)
3. **Deduplicate** - Remove near-duplicate questions
4. **Normalize citations** - Convert to `<<SRC:...>>` format

### Quality Criteria

A Q&A pair is "good" if:
- ✅ Staff marked as "good"
- ✅ Answer is factually accurate (spot-check)
- ✅ Question is clear and specific
- ✅ Not a duplicate of another pair

### Expected Output: ~500-1000 pairs

---

## Source 3: Documentation Q&A

### Generation Pipeline

```
PDF/Doc Section  →  LLM generates 3-5 Q&A  →  Human reviews 10-20%  →  Training data
```

### LLM Prompt for Q&A Generation

Given a documentation section, generate questions a real user might ask, with answers that cite the source.

**Requirements**:
1. Questions should be natural, varied complexity
2. Answers should be complete but concise
3. Each answer must include citation marker
4. Include "how to" questions where appropriate

### Document Categories

| Category | Example Docs | Q&A Potential |
|----------|--------------|---------------|
| Getting Started | Account setup, first allocation | HIGH |
| User Guides | Resource-specific guides | HIGH |
| FAQs | Existing FAQ pages | MEDIUM (may duplicate) |
| Technical | Job submission, data transfer | HIGH |

### Expected Output: ~500-1000 pairs

---

## Deduplication Strategy

Multiple sources generate overlapping questions. Deduplication prevents wasted training capacity.

### Process

1. **Exact match removal** - Hash-based duplicate detection
2. **Semantic clustering** - Group similar questions via embeddings (threshold ~0.85)
3. **Selection from clusters** - Keep best answer from each cluster

### Priority Rules

When choosing between duplicates:
1. **User Q&A** > Doc Q&A > MCP Q&A (real questions highest value)
2. **More complete answer** > shorter answer
3. **Better citations** > missing citations
4. **More recent** > older (for time-sensitive content)

---

## Output Format

### JSONL Schema

```json
{
  "id": "qa_00001",
  "source": "mcp_extraction | user_qa | doc_generated",
  "domain": "compute:resource_specs",
  "messages": [
    {
      "role": "user",
      "content": "What GPUs does Delta have?"
    },
    {
      "role": "assistant",
      "content": "Delta at NCSA is equipped with NVIDIA A100 GPUs...\n\n<<SRC:compute-resources:delta.ncsa.access-ci.org>>"
    }
  ],
  "metadata": {
    "complexity": "simple | moderate | complex",
    "has_citation": true,
    "created_at": "2025-12-08T00:00:00Z"
  }
}
```

### Directory Structure

```
training_data/
├── raw/
│   ├── mcp/                    # Extracted from MCP servers
│   ├── user_qa/                # Exported from production
│   └── docs/                   # Generated from PDFs
├── processed/
│   ├── deduplicated.jsonl
│   └── quality_filtered.jsonl
└── final/
    ├── train.jsonl             # 90% split
    └── validation.jsonl        # 10% split
```

---

## Automated Refresh Pipeline

Data sources update at different frequencies. Automated extraction keeps training data current.

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
4. Queue for human review if changes exceed threshold

### Retraining Triggers

| Trigger | Threshold |
|---------|-----------|
| New Q&A pairs accumulated | >500 |
| Modified Q&A pairs | >200 |
| New user Q&A added | >100 |
| Scheduled | Quarterly |
| Manual | On major release |

→ *Details on human review: [04-review-system.md](./04-review-system.md)*

---

## Domain Tagging

Tags help analyze coverage and balance the dataset.

### Taxonomy

```yaml
domains:
  compute:
    - resource_specs
    - hardware_comparison
    - resource_selection
  software:
    - availability
    - versions
    - category_search
  allocations:
    - project_discovery
    - statistics
  process:
    - getting_started
    - job_submission
    - data_transfer
  status:
    - defer_to_live
```

---

## Summary

| Source | Expected Q&A | Priority |
|--------|--------------|----------|
| MCP extraction | ~1,300 | High - structured, reliable |
| User Q&A database | ~500-1000 | Highest - real questions |
| Documentation | ~500-1000 | Medium - covers gaps |
| **Total (after dedup)** | **~2,500-3,500** | |

→ *Next in pipeline: [Review System](./03-review-system.md)* - Human review before training
