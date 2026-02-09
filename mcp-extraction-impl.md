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

### 5. Quality Evaluation

Before pushing to Argilla, each Q&A pair is scored by an LLM judge. This enables reviewers to prioritize their work and provides data for future automation.

**Evaluation dimensions:**

| Dimension | What it checks | Score range |
|-----------|----------------|-------------|
| Faithfulness | Does every claim in the answer match source_data? | 0.0 - 1.0 |
| Relevance | Does the answer address the question? | 0.0 - 1.0 |
| Completeness | Does the answer cover key facts from the source? | 0.0 - 1.0 |
| Confidence | min(faithfulness, relevance, completeness) | 0.0 - 1.0 |

**LLM for evaluation:** `gpt-4o-mini` or `claude-haiku-3.5`

**Cost:** ~$0.18-1.00 per 1000 pairs (roughly 10-25% of generation cost)

**Evaluator prompt:**

```
You are evaluating a Q&A pair generated from structured data about HPC resources.

SOURCE DATA:
{source_data_json}

QUESTION: {question}
ANSWER: {answer}

Score each dimension 0.0-1.0:

1. FAITHFULNESS: Does every claim in the answer match the source data?
   - 1.0 = All claims verifiable from source
   - 0.5 = Some claims not in source but plausible
   - 0.0 = Contains incorrect or fabricated information

2. RELEVANCE: Does the answer address what was asked?
   - 1.0 = Directly answers the question
   - 0.5 = Partially answers or includes irrelevant info
   - 0.0 = Does not answer the question

3. COMPLETENESS: Does the answer include key facts from the source?
   - 1.0 = Covers all relevant information
   - 0.5 = Missing some important details
   - 0.0 = Severely incomplete

Output JSON:
{"faithfulness": 0.X, "relevance": 0.X, "completeness": 0.X, "issues": ["issue1", ...]}
```

**Suggested decision logic:**

| Confidence | Suggested Decision | Meaning |
|------------|-------------------|---------|
| ≥ 0.8 | `approved` | High quality, quick review |
| < 0.8 | `needs_review` | Requires careful human review |

**Implementation:**

```python
class QAEvaluator:
    def __init__(self, llm_client: BaseLLMClient):
        self.llm = llm_client

    def evaluate(self, pair: QAPair) -> QAEvaluation:
        """Score a Q&A pair against its source_data."""
        response = self.llm.generate(
            system=EVALUATOR_SYSTEM_PROMPT,
            user=format_evaluation_prompt(pair),
        )
        scores = parse_evaluation_response(response.text)
        return QAEvaluation(
            faithfulness=scores["faithfulness"],
            relevance=scores["relevance"],
            completeness=scores["completeness"],
            confidence=min(scores["faithfulness"], scores["relevance"], scores["completeness"]),
            issues=scores.get("issues", []),
            suggested_decision="approved" if confidence >= 0.8 else "needs_review",
        )
```

**Phase 1 (current):** All pairs go to Argilla with scores. Humans make final decision.

**Phase 2 (future):** After calibrating with ~200 human reviews, high-confidence pairs can auto-approve and bypass human review. The threshold is tuned based on agreement between evaluator and human decisions.

**Calibration approach:** Compare evaluator's `suggested_decision` against human `review_decision` in Argilla. Track:
- Precision: Of pairs evaluator marked `approved`, what % did humans approve?
- Recall: Of pairs humans approved, what % did evaluator mark `approved`?
- False negatives: Pairs evaluator flagged `needs_review` but humans approved (wastes reviewer time)
- False positives: Pairs evaluator marked `approved` but humans rejected (quality risk)

Adjust threshold to minimize false positives while keeping false negatives acceptable.

**Related:** [ARES](https://github.com/stanford-futuredata/ARES) - Stanford framework for automated RAG evaluation with Prediction-Powered Inference for statistically rigorous results.

### 6. Argilla Push

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

**Evaluation fields (from Quality Evaluation step):**

| Field | Type | Description |
|-------|------|-------------|
| faithfulness_score | float | 0.0-1.0 score from evaluator |
| relevance_score | float | 0.0-1.0 score from evaluator |
| completeness_score | float | 0.0-1.0 score from evaluator |
| confidence_score | float | min(faithfulness, relevance, completeness) |
| eval_issues | text[] | Issues identified by evaluator |
| suggested_decision | enum | `approved` or `needs_review` |

These fields enable reviewers to:
- Sort by confidence (review low-confidence pairs first)
- Filter by suggested decision
- See evaluator's reasoning in `eval_issues`

### 7. CLI

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

# Evaluation (optional - defaults to gpt-4o-mini)
EVAL_LLM_BACKEND=openai        # or anthropic
EVAL_LLM_MODEL=gpt-4o-mini     # or claude-haiku-3.5
```

---

## Directory Structure

```
access-qa-extraction/
├── src/access_qa_extraction/
│   ├── __init__.py
│   ├── cli.py                 # Typer CLI entrypoint
│   ├── config.py              # Settings via pydantic-settings
│   ├── mcp_client.py          # HTTP client for MCP
│   ├── llm_client.py          # LLM abstraction (Anthropic, OpenAI, local)
│   ├── models.py              # QAPair, QAEvaluation models
│   ├── extractors/
│   │   ├── __init__.py
│   │   ├── base.py            # BaseExtractor abstract class
│   │   ├── compute_resources.py
│   │   ├── software_discovery.py
│   │   ├── allocations.py
│   │   ├── nsf_awards.py
│   │   └── affinity_groups.py
│   ├── evaluator.py           # LLM-as-judge quality scoring
│   ├── deduplication.py       # Embedding + similarity check
│   ├── change_detection.py    # Hash-based diff
│   └── argilla_client.py      # Push to Argilla
├── data/
│   ├── hashes/                # Change detection hashes
│   └── output/                # Debug output (gitignored)
├── tests/
├── pyproject.toml
└── README.md
```

---

## Implementation Order

### Phase 1: Core Pipeline (one server) ✅

1. **Extractor for compute-resources** - Fetch all resources, generate Q&A via LLM
2. **CLI basics** - `qa-extract extract compute-resources --dry-run`
3. **Test end-to-end** - Verify Q&A quality manually

### Phase 2: Argilla Integration ✅

4. **Argilla push** - Push records to dataset with embeddings
5. **Deduplication** - Vector similarity check before insert
6. **CLI flags** - `--push-to-argilla`, `--no-dedup`

### Phase 3: Remaining Servers ✅

7. **software-discovery extractor** - Search-terms strategy
8. **allocations extractor** - Broad-queries strategy
9. **nsf-awards extractor** - Broad-queries strategy
10. **affinity-groups extractor** - List-all strategy

### Phase 4: Quality Evaluation ← **CURRENT**

11. **QAEvaluator class** - LLM-as-judge scoring (faithfulness, relevance, completeness)
12. **Evaluation model** - `QAEvaluation` dataclass with scores and issues
13. **Argilla metadata** - Add evaluation scores and suggested_decision fields
14. **CLI integration** - Evaluate before push, add `--skip-eval` flag
15. **Calibration tooling** - Compare evaluator vs human decisions after ~200 reviews

### Phase 5: Automation

16. **Scheduling** - GitHub Actions or cron for weekly runs
17. **Monitoring** - Log extraction stats, alert on failures
18. **Change detection** - Hash storage, skip unchanged entities (deferred - regeneration is cheap enough)

### Phase 6: Auto-Approval (Future)

19. **Threshold tuning** - Analyze Phase 4 calibration data
20. **Auto-approve flow** - High-confidence pairs bypass Argilla, push directly to vector DB
21. **Audit logging** - Track auto-approved pairs for quality monitoring

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
| LLM rate limit | Exponential backoff, retry up to 3 times |
| LLM returns invalid JSON | Log, retry once with "please output valid JSON" |
| Argilla push fails | Log error, save to local JSONL for manual retry |
| Duplicate detection fails | Log warning, push without duplicate flag |
| Evaluation fails | Log warning, set confidence=0.0 and suggested_decision=`needs_review` |

---

## Testing

### Unit Tests
- Extractor parsing (mock MCP responses)
- Q&A generator output validation
- Citation marker extraction
- Evaluator response parsing

### Integration Tests
- End-to-end with test MCP server
- Argilla push to test dataset
- Evaluation scoring pipeline

### Manual Validation
- Review generated Q&A quality
- Check citation accuracy
- Verify deduplication works
- Spot-check evaluator scores against human judgment

---

## Success Criteria

### Generation Quality

| Metric | Target |
|--------|--------|
| Q&A pairs generated per resource | 3-8 |
| Citation marker present | 100% |
| Invalid JSON from LLM | <5% of calls |
| Duplicate detection accuracy | >90% |
| Time to extract compute-resources | <2 minutes |

### Evaluation Quality (after calibration)

| Metric | Target | Notes |
|--------|--------|-------|
| Evaluator precision | >90% | Of `suggested: approved`, humans approve 90%+ |
| Evaluator recall | >80% | Of human-approved, evaluator suggests 80%+ |
| False positive rate | <5% | Evaluator says approved, human rejects |
| Evaluation cost | <$1/1000 pairs | Using gpt-4o-mini or haiku |

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
