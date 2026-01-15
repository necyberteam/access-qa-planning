# MCP Q&A Extraction - Implementation Spec

> **Implementation guide for**: [access-qa-extraction](https://github.com/necyberteam/access-qa-extraction)
>
> **Related docs**: [Q&A Data Preparation](./02-qa-data.md) | [Review System](./03-review-system.md)

## Overview

Build a pipeline that extracts structured data from MCP servers, uses an LLM to generate Q&A pairs, and pushes them to Argilla for human review.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  MCP Servers    │     │  Change         │     │   LLM (GPT-4)   │     │    Argilla      │
│  (fetch data)   │────▶│  Detection      │────▶│   Q&A gen       │────▶│   (review)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
```

After human review in Argilla, approved pairs are synced to access-qa-service (handled separately).

---

## Components

### 1. MCP Data Fetchers

One fetcher per MCP server. Each fetcher:
- Calls MCP server tools via HTTP (using existing `MCPClient`)
- Returns a list of entities with their attributes

**Servers to implement:**

| Server | Priority | Entity Count | Entity Type | Key Attributes |
|--------|----------|--------------|-------------|----------------|
| compute-resources | HIGH | ~23 | Resource | name, GPUs, nodes, memory, description |
| software-discovery | HIGH | ~1,400 | Software Package | name, versions, resources, AI metadata |
| allocations | MEDIUM | TBD | Project/Allocation | field of science, PI, resources |
| nsf-awards | MEDIUM | TBD | Award | title, abstract, PI |
| affinity-groups | LOW | TBD | Group | name, description, focus area |

**Note on volume:** Software-discovery has ~1,400 packages. This requires batching and careful cost management for LLM calls.

**Example fetcher interface:**

```python
class ComputeResourcesFetcher:
    async def fetch_all(self) -> list[dict]:
        """Fetch all compute resources from MCP server."""
        # Returns list of resource dicts with all attributes
```

### 2. Q&A Generator

Takes entity data and generates Q&A pairs using an LLM.

**LLM**: OpenAI GPT-4 (or gpt-4-turbo for cost savings)

**Generation approach:**
- Pass entity data to LLM with a domain-specific prompt
- LLM generates Q&A pairs based on entity complexity (see guidance below)
- Each answer must include citation marker: `<<SRC:domain:entity_id>>`

**Q&A volume guidance (based on actual MCP data):**

| Entity Complexity | Example | Expected Q&A Pairs |
|-------------------|---------|-------------------|
| Simple | Granite (storage archive) - single description | 2-4 pairs |
| Medium | Ookami, KyRIC - one node type, limited features | 4-6 pairs |
| Complex | Delta, Bridges-2 - multiple node types, GPU configs, storage | 8-12 pairs |

Rather than hardcoding a number, the prompt should instruct the LLM to cover all key facts without padding. A storage system with one paragraph of info doesn't need 8 questions.

**Prompt structure:**

```
You are generating Q&A pairs for a knowledge base about ACCESS-CI computing resources.

Given this entity data:
{entity_json}

Generate question-answer pairs covering the key facts a researcher would want to know.
- Cover all important facts without padding or repetition
- Simple resources may only need 2-4 pairs; complex resources with multiple node types, GPU configurations, etc. may need 8-12
- Questions should be natural, as a researcher would ask them
- Answers should be factual and concise
- Every answer MUST end with a citation marker in this format:
  <<SRC:{domain}:{entity_id}>>

Output as JSON array:
[
  {"question": "...", "answer": "... <<SRC:compute-resources:delta.ncsa.access-ci.org>>"},
  ...
]
```

**Question types to cover:**
- Simple factual: "What GPUs does Delta have?"
- Discovery: "Which resources have A100 GPUs?"
- Comparison: "How does Delta compare to Expanse?" (for related entities)
- Recommendation: "What resource is best for machine learning?"

### 3. Change Detection

Avoid regenerating Q&A for unchanged entities.

**Approach:** Hash-based diff

1. Fetch entity from MCP
2. Compute hash of entity data (deterministic JSON → SHA256)
3. Compare to stored hash from last run
4. If different or new: generate Q&A, update hash
5. If same: skip

**Hash storage:** Simple JSON file per domain

```json
// data/hashes/compute-resources.json
{
  "delta.ncsa.access-ci.org": "a1b2c3d4...",
  "expanse.sdsc.access-ci.org": "e5f6g7h8...",
  "last_run": "2025-01-15T10:30:00Z"
}
```

### 4. Deduplication Check

Before pushing to Argilla, check for similar existing records.

**Approach:**
1. Generate embedding for the question (same model as access-qa-service)
2. Call Argilla SDK `find_similar_records`
3. If similarity > 0.9, flag in metadata as potential duplicate
4. Still push to Argilla - let reviewer decide

**Why not reject duplicates?**
- New Q&A might be better than existing
- Reviewer can compare and choose
- Avoids silent data loss

### 5. Argilla Push

Push Q&A pairs to Argilla for human review.

**Dataset:** `qa-review` (or as configured)

**Record fields:**

| Field | Type | Description |
|-------|------|-------------|
| question | text | The generated question |
| answer | text | The generated answer with citation |
| domain | metadata | MCP server domain (e.g., `compute-resources`) |
| entity_id | metadata | Entity identifier |
| source_ref | metadata | MCP URI (e.g., `mcp://compute-resources/resources/delta`) |
| source_data | metadata | Original entity JSON (for reviewer verification) |
| is_duplicate | metadata | Boolean flag if similar record exists |
| generated_at | metadata | Timestamp |

### 6. CLI

Command-line interface for running extraction.

```bash
# Extract from one server
qa-extract compute-resources

# Extract from all servers
qa-extract all

# Dry run (show what would be generated, don't push)
qa-extract compute-resources --dry-run

# Force regeneration (ignore hashes)
qa-extract compute-resources --force

# Specify output for debugging
qa-extract compute-resources --output ./debug-output.jsonl
```

---

## Configuration

Environment variables:

```bash
# MCP Server URLs
MCP_COMPUTE_RESOURCES_URL=http://localhost:3002
MCP_SOFTWARE_DISCOVERY_URL=http://localhost:3003
MCP_ALLOCATIONS_URL=http://localhost:3004
MCP_AFFINITY_GROUPS_URL=http://localhost:3005
MCP_NSF_AWARDS_URL=http://localhost:3006

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4-turbo  # or gpt-4

# Argilla
ARGILLA_URL=https://argilla.example.com
ARGILLA_API_KEY=...
ARGILLA_DATASET=qa-review

# Embedding model (for deduplication)
EMBEDDING_MODEL=all-MiniLM-L6-v2
```

---

## Directory Structure

```
access-qa-extraction/
├── src/access_qa_extraction/
│   ├── __init__.py
│   ├── cli.py                 # Click CLI entrypoint
│   ├── config.py              # Settings via pydantic-settings
│   ├── mcp_client.py          # HTTP client for MCP (exists)
│   ├── models.py              # QAPair, Entity models
│   ├── fetchers/
│   │   ├── __init__.py
│   │   ├── base.py            # BaseFetcher abstract class
│   │   ├── compute_resources.py
│   │   ├── software_discovery.py
│   │   ├── allocations.py
│   │   ├── nsf_awards.py
│   │   └── affinity_groups.py
│   ├── generator.py           # LLM Q&A generation
│   ├── deduplication.py       # Embedding + similarity check
│   ├── change_detection.py    # Hash-based diff
│   └── argilla_push.py        # Push to Argilla
├── data/
│   ├── hashes/                # Change detection hashes
│   └── output/                # Debug output (gitignored)
├── tests/
├── pyproject.toml
└── README.md
```

---

## Implementation Order

### Phase 1: Core Pipeline (one server)

1. **Fetcher for compute-resources** - Fetch all resources, return as list of dicts
2. **Q&A Generator** - Call OpenAI, parse response, validate citations
3. **CLI basics** - `qa-extract compute-resources --dry-run`
4. **Test end-to-end** - Verify Q&A quality manually

### Phase 2: Argilla Integration

5. **Argilla push** - Push records to dataset
6. **Deduplication** - Embed questions, check for similar records
7. **Change detection** - Hash storage, skip unchanged entities

### Phase 3: Remaining Servers

8. **software-discovery fetcher** - Larger dataset, may need batching
9. **allocations fetcher**
10. **nsf-awards fetcher**
11. **affinity-groups fetcher**

### Phase 4: Automation

12. **Scheduling** - GitHub Actions or cron for weekly runs
13. **Monitoring** - Log extraction stats, alert on failures

---

## Q&A Generation Prompts

### Compute Resources

```
You are generating Q&A pairs about ACCESS-CI compute resources for researchers.

Resource data:
{entity_json}

Generate 3-8 Q&A pairs covering:
- Hardware specs (GPUs, CPUs, memory, storage)
- Queue/partition information
- What workloads it's suited for
- How to access it

Every answer MUST end with: <<SRC:compute-resources:{entity_id}>>

Output as JSON array of {question, answer} objects.
```

### Software Discovery

```
You are generating Q&A pairs about software available on ACCESS-CI resources.

Software data:
{entity_json}

Generate 3-5 Q&A pairs covering:
- What the software does
- Which resources have it installed
- Version information
- How to load/use it

Every answer MUST end with: <<SRC:software-discovery:{entity_id}>>

Output as JSON array of {question, answer} objects.
```

---

## Error Handling

| Error | Handling |
|-------|----------|
| MCP server unavailable | Log error, skip server, continue with others |
| OpenAI rate limit | Exponential backoff, retry up to 3 times |
| OpenAI returns invalid JSON | Log, retry once with "please output valid JSON" |
| Argilla push fails | Log error, save to local JSONL for manual retry |
| Duplicate detection fails | Log warning, push without duplicate flag |

---

## Testing

### Unit Tests
- Fetcher parsing (mock MCP responses)
- Q&A generator output validation
- Hash computation determinism
- Citation marker extraction

### Integration Tests
- End-to-end with test MCP server
- Argilla push to test dataset

### Manual Validation
- Review generated Q&A quality
- Check citation accuracy
- Verify deduplication works

---

## Success Criteria

| Metric | Target |
|--------|--------|
| Q&A pairs generated per resource | 3-8 |
| Citation marker present | 100% |
| Invalid JSON from LLM | <5% of calls |
| Duplicate detection accuracy | >90% |
| Time to extract compute-resources | <2 minutes |

---

## Handling High-Volume Sources (Software Discovery)

With ~1,400 software packages, software-discovery requires special handling:

**Cost estimation:**
- ~1,400 packages × ~3 Q&A pairs avg = ~4,200 Q&A pairs
- GPT-4-turbo: ~$0.01-0.03 per package = $14-42 for full extraction
- With change detection, ongoing costs are much lower (only changed packages)

**Recommended approach:**

1. **Batch fetching** - Fetch all packages in one call (API supports high limits)
2. **Sequential LLM calls with rate limiting** - Process one package at a time, respect rate limits
3. **Checkpoint progress** - Save state after each batch so extraction can resume if interrupted
4. **Prioritize by usage** - Consider extracting popular packages first (tensorflow, python, etc.)

**Q&A scope for software:**
- Most packages need only 2-3 Q&A pairs (what it is, where available, how to load)
- Complex packages with extensive AI metadata might get 4-5 pairs
- Don't generate pairs for packages with null/empty descriptions

---

## Open Questions

1. **Comparison Q&A** - Should generator produce cross-entity comparisons? ("Delta vs Expanse") Or leave for later?

2. **Argilla dataset schema** - Need to confirm field names match what access-qa-service expects for sync.

3. **Software prioritization** - Should we extract all 1,400 packages, or start with a curated list of common ones?
