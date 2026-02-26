# MCP Q&A Extraction - Implementation Spec

> **Implementation guide for**: [access-qa-extraction](https://github.com/necyberteam/access-qa-extraction)
>
> **Related docs**: [Q&A Data Preparation](./02-qa-data.md) | [Review System](./03-review-system.md)
>
> **Updated**: 2026-02-23 to reflect implemented two-shot pipeline and entity-replace Argilla push.

## Overview

Pipeline that extracts structured data from MCP servers, uses an LLM in two passes to generate Q&A pairs, scores them with a judge, and pushes them to Argilla for human review.

```
┌─────────────────┐     ┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│  MCP Servers    │     │  Change         │     │   LLM (two-shot)     │     │    Argilla      │
│  (fetch data)   │────▶│  Detection      │────▶│   battery+discovery  │────▶│  entity-replace │
└─────────────────┘     └─────────────────┘     │   + judge scoring    │     └─────────────────┘
                                                 └──────────────────────┘
```

After human review in Argilla, approved pairs are synced to access-qa-service (handled separately).

---

## Components

### 1. MCP Data Fetchers

One extractor per MCP server. Each fetcher calls MCP server tools via HTTP (using `MCPClient`) or paginates a public API directly, then returns a list of entities with their attributes.

**Servers implemented:**

| Server | Port | Entity Count | Strategy | Key Attributes |
|--------|------|--------------|----------|----------------|
| compute-resources | 3002 | ~26 | list-all | name, GPUs, nodes, memory, description |
| software-discovery | 3004 | ~1,400 | search-terms (~34 curated terms) | name, versions, resources, AI metadata |
| allocations | 3006 | ~5,360 | direct API pagination | field of science, PI, resources |
| nsf-awards | 3007 | ~10K cap | direct API pagination | title, abstract, PI, funding |
| affinity-groups | 3011 | ~54 | list-all | name, description, coordinator, events |

**Extraction strategies:**
- **list-all** — server returns everything with an empty query (compute-resources, affinity-groups)
- **search-terms** — curated list of domain keywords, results deduplicated (software-discovery)
- **direct API pagination** — fetches directly from a public REST API, no MCP (allocations, nsf-awards)

---

### 2. Q&A Generator — Two-Shot Pipeline

Each entity goes through two sequential LLM calls. Both calls share the same user prompt (the entity data); they differ only in the system prompt.

**LLM backends supported:** Anthropic, OpenAI, local (Ollama/vLLM via OpenAI-compatible API), Transformers. Configured via `LLM_BACKEND` env var.

#### Call 1: Battery

System prompt from `build_battery_system_prompt(domain)`. Instructs the LLM:

> "Generate exactly one Q&A pair for each field group listed below. Skip a group only if the data genuinely doesn't contain information for it."

Field groups are defined in `FIELD_GUIDANCE` (per domain) — a list of `{fields, instruction, condition}` specs. Example for compute-resources:

| # | Instruction | Data fields | Condition |
|---|-------------|-------------|-----------|
| 1 | Overview — what is this resource? | name, description, resourceType | — |
| 2 | Organization — who operates it? | organization_names | — |
| 3 | GPU hardware | hardware.gpus | only if hasGpu is true |
| 4 | CPU/compute hardware | hardware.compute_nodes | only if non-empty |
| 5 | Storage | hardware.storage | only if non-empty |
| 6 | Features & capabilities | feature_names | — |
| 7 | Access — how does a researcher get access? | accessAllocated | — |

Result: ~5-7 pairs with guaranteed field coverage. Pair IDs: `{domain}_{entity_id}_{seq_n}`.

#### Call 2: Discovery

System prompt from `build_discovery_system_prompt(domain, existing_pairs)`. Receives the battery pairs as context and is told:

> "Find what's missing or interesting that the existing pairs didn't capture — notable partnerships, unique architectures, unusual use cases, specific numbers not yet mentioned. If the existing pairs already cover everything, output `[]`."

Result: 0-3 additional pairs. The combined battery + discovery output forms the final set for this entity.

#### Citation format

Every answer must end with `<<SRC:{domain}:{entity_id}>>`. Validated post-hoc by `citation_validator.py`.

---

### 3. Change Detection (Incremental Cache)

Avoid regenerating Q&A for unchanged entities.

**Approach:** Hash-based diff via `IncrementalCache` (`generators/incremental.py`).

1. Fetch and clean entity data
2. `compute_entity_hash(entity_data)` → `sha256(json.dumps(sorted))[:16]`
3. Compare against stored hash in `.extraction_cache.json`
4. If changed or new: run two-shot generation + judge, store new hash + pairs
5. If unchanged: replay cached pairs (including judge scores)

**Cache storage:** `data/output/.extraction_cache.json` — a flat dict keyed `{domain}_{entity_id}`.

Enabled with `--incremental` CLI flag. On first run the cache is empty; every subsequent run skips unchanged entities entirely (all three LLM calls).

---

### 4. Quality Evaluation (Judge)

After generation, all pairs for an entity are scored in a single batch by a cheaper "judge" LLM.

**LLM:** Configured via `LLM_JUDGE_BACKEND` / `LLM_JUDGE_MODEL` (defaults to main backend). Recommended: `gpt-4o-mini` or `claude-haiku`.

**Evaluation dimensions:**

| Dimension | What it checks | Score range |
|-----------|----------------|-------------|
| Faithfulness | Does every claim in the answer match source_data? | 0.0 - 1.0 |
| Relevance | Does the answer address the question? | 0.0 - 1.0 |
| Completeness | Does the answer cover key facts from the source? | 0.0 - 1.0 |
| Confidence | `min(faithfulness, relevance, completeness)` | 0.0 - 1.0 |

**Suggested decision logic:**

| Confidence | Suggested Decision |
|------------|--------------------|
| ≥ 0.8 | `approved` |
| < 0.8 | `needs_review` |

**Implementation:** `evaluate_pairs(pairs, source_data, judge_client)` in `generators/judge.py`. Mutates `pair.metadata` in-place — does not create new pairs. Wrapped in try/except; if the judge fails, pairs keep `None` scores and the pipeline continues.

Skip with `--no-judge`.

**Phase 1 (current):** All pairs go to Argilla with scores. Humans make final decision.

**Phase 2 (future):** After calibrating with ~200 human reviews, high-confidence pairs can auto-approve. Calibration: compare `suggested_decision` vs human `review_decision` in Argilla.

---

### 5. Argilla Push — Entity-Replace

Push Q&A pairs to Argilla using **entity-replace semantics**: for each `source_ref`, delete existing records and push fresh ones.

**Why entity-replace instead of semantic dedup:**
- Semantic dedup only compares questions — it cannot detect when an answer is outdated
- Entity-replace is simpler: one entity = one authoritative set of pairs
- Annotation preservation handles the main downside (see below)

**Entity-replace loop (per `source_ref`):**

1. Query existing records filtered by `source_ref`
2. **Archive annotated records** — if any record has a submitted human response (`response.status == "submitted"`), copy it to `qa-review-archive-superseded` before deletion. Each archived record gets:
   - `archived_at` — ISO timestamp
   - `replaced_reason` — `"source_data_changed"`
   - `annotation_depth` — `"approved_only"` (rubber-stamp) or `"has_edits"` (reviewer edited Q/A or left notes)
3. Delete all existing records for this `source_ref`
4. Push fresh records with new pairs + judge scores + embeddings

**Dataset:** `qa-review` (configurable). Embedding model: `all-MiniLM-L6-v2` (384-dim).

**Record fields:**

| Field | Type | Description |
|-------|------|-------------|
| question | text | The generated question |
| answer | text | The generated answer with citation |
| source_data | text | Original entity JSON (for reviewer verification) |
| eval_issues | text[] | Issues flagged by judge |
| domain | metadata | MCP server domain |
| source_ref | metadata | MCP URI (`mcp://compute-resources/resources/delta`) |
| faithfulness_score | metadata | 0.0-1.0 |
| relevance_score | metadata | 0.0-1.0 |
| completeness_score | metadata | 0.0-1.0 |
| confidence_score | metadata | min(three scores) |
| suggested_decision | metadata | `approved` or `needs_review` |
| granularity | metadata | `comprehensive` or `comparison` |

**Push triggers:**
- `qa-extract extract ... --push-to-argilla` — extract then push in one step
- `qa-extract push data/output/file.jsonl` — push an existing JSONL file

---

### 6. Comparison Generator

Programmatic (no LLM) cross-entity Q&A pairs, generated after all extractors complete.

Groups entities by shared attributes and creates questions like:
- "Which resources have NVIDIA A100 GPUs?"
- "Which projects use Delta?"
- "Which affinity groups are in the Science category?"

`granularity = "comparison"` on these pairs. Uses `raw_data` from each extractor (normalized per-entity dicts). No hallucination risk.

---

### 7. CLI

```bash
# Extract from specific servers
qa-extract extract compute-resources
qa-extract extract compute-resources allocations

# Extract from all servers
qa-extract extract compute-resources software-discovery allocations nsf-awards affinity-groups

# Useful flags
qa-extract extract compute-resources --max-entities 2    # cap LLM calls for testing
qa-extract extract compute-resources --no-judge           # skip judge scoring
qa-extract extract compute-resources --incremental        # skip unchanged entities
qa-extract extract compute-resources --push-to-argilla    # push after extraction

# Inspect output
qa-extract stats data/output/file.jsonl
qa-extract validate data/output/file.jsonl

# Push existing JSONL to Argilla
qa-extract push data/output/file.jsonl
```

---

## Configuration

```bash
# MCP Server URLs
MCP_COMPUTE_RESOURCES_URL=http://localhost:3002
MCP_SOFTWARE_DISCOVERY_URL=http://localhost:3004
MCP_ALLOCATIONS_URL=http://localhost:3006
MCP_NSF_AWARDS_URL=http://localhost:3007
MCP_AFFINITY_GROUPS_URL=http://localhost:3011

# LLM backend (anthropic | openai | local | transformers)
LLM_BACKEND=anthropic
ANTHROPIC_API_KEY=sk-ant-...
# or
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o

# Judge (cheaper model recommended)
LLM_JUDGE_BACKEND=openai
LLM_JUDGE_MODEL=gpt-4o-mini

# Argilla
ARGILLA_URL=http://localhost:6900
ARGILLA_API_KEY=argilla.apikey
```

---

## Directory Structure

```
access-qa-extraction/
├── src/access_qa_extraction/
│   ├── cli.py                      # Typer CLI: extract, push, stats, validate
│   ├── config.py                   # ExtractionConfig, MCPServerConfig
│   ├── mcp_client.py               # Async HTTP client for MCP tool endpoints
│   ├── llm_client.py               # BaseLLMClient + 4 backends, get_llm_client()
│   ├── models.py                   # QAPair, Message, QAMetadata
│   ├── question_categories.py      # FIELD_GUIDANCE, battery/discovery prompt builders
│   ├── citation_validator.py       # Validates <<SRC:>> citations
│   ├── argilla_client.py           # Entity-replace push with annotation archiving
│   ├── extractors/
│   │   ├── base.py                 # BaseExtractor (MCPClient context manager)
│   │   ├── compute_resources.py    # list-all via MCP
│   │   ├── software_discovery.py   # search-terms via MCP
│   │   ├── allocations.py          # direct API pagination (overrides run())
│   │   ├── nsf_awards.py           # direct API pagination (overrides run())
│   │   └── affinity_groups.py      # list-all via MCP
│   ├── generators/
│   │   ├── incremental.py          # Hash cache, compute_entity_hash()
│   │   ├── judge.py                # evaluate_pairs() — LLM-as-judge scoring
│   │   └── comparisons.py          # Programmatic cross-entity Q&A
│   └── output/
│       └── jsonl_writer.py         # Writes QAPair lists to .jsonl
├── data/
│   └── output/                     # JSONL output + .extraction_cache.json
├── tests/
├── docs/
│   └── TRACE-TOUR.extract.md       # Annotated execution trace (22 waypoints)
└── pyproject.toml
```

---

## Implementation Status

### ✅ Phase 1: Core Pipeline
Extractors for all 5 servers. Two-shot LLM generation (battery + discovery). Incremental hash cache. Comparison generator.

### ✅ Phase 2: Quality Evaluation
Judge LLM scores all pairs per entity (faithfulness, relevance, completeness, confidence). Scores stored in cache and JSONL output.

### ✅ Phase 3: Argilla Integration
Entity-replace push with annotation archiving. Full metadata schema including judge scores, granularity, source_ref, eval_issues. Embedding generation (all-MiniLM-L6-v2).

### 🔲 Phase 4: Production
- PR `feat/two-shot` → `main`
- Per-domain output cleanup (allocations grammar, affinity-groups coverage)
- Answer open questions with Andrew (PI emails in training data?)
- Software-discovery live testing (needs `SDS_API_KEY`)

### 🔲 Phase 5: Automation
- GitHub Actions or cron for weekly extraction runs
- Monitoring and alerting on extraction failures

### 🔲 Phase 6: Auto-Approval (Future)
After calibrating evaluator against ~200 human reviews: auto-approve high-confidence pairs, bypass Argilla, push directly to vector DB.

---

## Open Questions

1. **PI emails in training data** — NSF award data includes PI email addresses. OK to include in Q&A pairs pushed to Argilla and eventually to RAG?

2. **Variable pair count** — Battery generates one pair per field group (~5-7); discovery adds 0-3 more. Total per entity varies. Is this OK, or should we target a fixed count?

3. **Software-discovery volume** — ~1,400 packages × 3 LLM calls each = significant cost. Should we cap, or use a cheaper model for software?
