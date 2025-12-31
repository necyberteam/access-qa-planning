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
│               │   UNIFIED DATASET   │                                       │
│               └─────────────────────┘                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why Q&A Pairs, Not Raw Documents

Research and industry practice show that fine-tuning on Q&A pairs is more effective than training on raw documentation:

| Approach | Pros | Cons |
|----------|------|------|
| **Q&A Pairs** | Teaches question-answering format directly; clearer training signal; reduces hallucination | Requires generation/curation effort |
| **Raw Docs** | Easy to collect | Model learns document structure, not Q&A format; higher hallucination rate |

Q&A pairs also capture what users *actually ask*, not just what documentation covers. Real user questions from production are especially valuable.

---

## Source 1: MCP Extraction

### Which Servers to Extract From

**Static Data (TRAIN INTO MODEL)**:

| Server | Tools | Update Frequency | Priority |
|--------|-------|------------------|----------|
| compute-resources | 2 | Monthly | **HIGH** |
| software-discovery | 3 | Weekly | **HIGH** |
| allocations | 1 | Weekly | MEDIUM |
| nsf-awards | 1 | Weekly | MEDIUM |
| affinity-groups | 1 | Monthly | LOW |

**Dynamic Data (KEEP AS LIVE MCP)** - Do NOT train these:

| Server | Tools | Why Keep Live |
|--------|-------|---------------|
| system-status | 1 | Real-time outages |
| events | 1 | Schedules change frequently |
| announcements | 1 | Time-sensitive |
| xdmod-charts | 3 | Always need current metrics |

### Q&A Templates by Domain

#### Compute Resources

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

#### Software Discovery

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

#### Allocations

- "What fields of science use ACCESS most?"
- "What {field} projects are on ACCESS?"
- "Find projects about {topic}"

#### Negative Examples - Defer to Live

These train the model to recognize when NOT to answer from memory:

- "Is Delta currently down?" → "I need to check live system status..."
- "What events are happening this week?" → "Event schedules require live data..."
- "What's my current SU balance?" → "I cannot access user-specific information..."
- "Is my project TG-ABC123 still active?" → "Project details require live verification..."

### Expected Output from MCP Extraction

Counts TBD after running extraction pipeline. Will depend on:
- Number of resources, software packages, allocations in each MCP server
- Number of Q&A templates per entity type
- Coverage of negative/deferral examples needed

---

## Source 2: Existing User Q&A Database

### What We Have
- Q&A pairs from production RAG system (count TBD)
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

---

## Source 3: Documentation Q&A

Generate Q&A pairs from existing ACCESS documentation (PDFs, web pages, guides).

### Generation Process

1. **Chunk documents** into logical sections
2. **LLM generates Q&A** for each section - questions a user might ask, with answers citing the source
3. **Human reviews** generated pairs before adding to training data

### LLM Prompt for Q&A Generation

Given a documentation section, generate questions a real user might ask, with answers that cite the source.

**Requirements**:
1. Questions should be natural, varied complexity
2. Answers should be complete but concise
3. Each answer must include citation marker
4. Include "how to" questions where appropriate

### Document Categories

| Category | Example Docs |
|----------|--------------|
| Getting Started | Account setup, first allocation |
| User Guides | Resource-specific guides |
| FAQs | Existing FAQ pages (check for duplicates) |
| Technical | Job submission, data transfer |

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

| Trigger | Threshold | Rationale |
|---------|-----------|-----------|
| New Q&A pairs accumulated | >10% of training set | Significant new knowledge |
| Modified Q&A pairs | >5% of training set | Enough drift to matter |
| Negative user feedback | >2% of responses | Quality degradation signal |
| Scheduled | Quarterly | Catch gradual drift |
| Manual | On major release | New resources, policy changes |

> *Details on human review: [03-review-system.md](./03-review-system.md)*

---

## Domain Tagging

Each Q&A pair is tagged with a domain category. This enables:

1. **Coverage analysis** - Identify gaps (e.g., few questions about data transfer)
2. **Dataset balancing** - Ensure the model doesn't over-learn one domain
3. **Targeted improvements** - When users report issues in a domain, add more examples there
4. **Evaluation slicing** - Measure accuracy per domain, not just overall

### How Tags Are Applied

- **MCP extraction**: Automatic based on source server (compute-resources → `compute`)
- **User Q&A**: Manual tagging during review, or classifier-assisted
- **Documentation**: Based on document section/category

### Taxonomy

```yaml
domains:
  compute:
    - resource_specs      # "What GPUs does Delta have?"
    - hardware_comparison # "Compare Delta and Expanse"
    - resource_selection  # "What system for ML training?"
  software:
    - availability        # "Is TensorFlow on Delta?"
    - versions           # "What version of Python?"
    - category_search    # "What visualization tools?"
  allocations:
    - project_discovery  # "Find quantum computing projects"
    - statistics         # "What fields use ACCESS most?"
  process:
    - getting_started    # "How do I get an account?"
    - job_submission     # "How do I submit a job?"
    - data_transfer      # "How do I transfer files?"
  status:
    - defer_to_live      # Questions that should trigger MCP calls
```

---

## Summary

| Source | Priority | Notes |
|--------|----------|-------|
| MCP extraction | High | Structured, reliable - count depends on entities in MCP servers |
| User Q&A database | Highest | Real questions - count depends on quality filter pass rate |
| Documentation | Medium | Covers gaps - count depends on corpus size |

Final dataset size TBD after extraction and deduplication.

→ *Next in pipeline: [Review System](./03-review-system.md)* - Human review before training
