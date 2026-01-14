# Model Training

> **⚠️ DEPRECATED**: This approach was set aside after pilot experiments. The fine-tuned model didn't reliably retain factual details, and worse, hallucinated additional details around what it did memorize - mixing real and fabricated information. The system now uses RAG-primary architecture instead. See [Agent Architecture](./01-agent-architecture.md) for the current approach.

> **Part of the Data Pipeline**: [Q&A Data Preparation](./02-qa-data.md) → [Review System](./03-review-system.md) → This doc
>
> **Related**: [Agent Architecture](./01-agent-architecture.md)

## Overview

Fine-tune a model on ACCESS-CI data to reduce MCP tool call latency while maintaining citation capabilities.

---

## Training Infrastructure

| Spec | Value |
|------|-------|
| **Hostname** | gh1-internal.xxx.xxx.xxx |
| **Jump Host** | 128.163.xxx.xx |
| **GPU** | NVIDIA GH200 480GB (Grace Hopper Superchip) |
| **VRAM** | 98 GB HBM3 |
| **CUDA** | 12.6 |
| **Driver** | 555.42.06 |

This hardware allows training large models at bf16 precision without quantization.

---

## Model Selection

### Pilot Model

**Starting with Mistral-7B-Instruct-v0.3** for the initial pilot:

| Model | Size | VRAM (bf16) | Rationale |
|-------|------|-------------|-----------|
| **Mistral-7B-Instruct-v0.3** | 7B | ~14GB | Well-proven for fine-tuning, strong instruction-following, simple to debug |

### Scale-Up Path

After validating the workflow with Mistral-7B:

| Stage | Model | VRAM | When |
|-------|-------|------|------|
| 1. Pilot | Mistral-7B | ~14GB | Validate pipeline works |
| 2. Compare | Llama 3 8B | ~16GB | If quality acceptable, compare |
| 3. Scale | Llama 3.1 70B | ~140GB | If 8B quality insufficient |
| 4. Optional | MoE (Mixtral) | ~90GB | Only if latency becomes critical |

### Architecture Trade-offs

| Factor | MoE Advantage | Dense Advantage |
|--------|---------------|-----------------|
| Inference speed | Faster (sparse activation) | - |
| Fine-tuning stability | - | Less prone to overfitting |
| Domain fine-tuning evidence | Less proven | Well established |
| Debugging | - | Simpler |

---

## Training Configuration

### GH200-Specific Optimizations

| Setting | Value | Rationale |
|---------|-------|-----------|
| Precision | bf16 (no quantization) | 98GB VRAM allows full precision |
| Flash Attention | Enabled | ~2x faster on GH200 |
| Batch Size | 4 per device (dense), 1 (MoE) | Larger batches fit in HBM3 |
| Optimizer | adamw_torch_fused | Faster on modern GPUs |
| Gradient Checkpointing | Enabled | Trade compute for memory headroom |

### LoRA Configuration

| Parameter | MoE | Dense |
|-----------|-----|-------|
| rank (r) | 64 | 64 |
| alpha | 128 | 128 |
| target_modules | q,k,v,o + gate,w1,w2,w3 | q,k,v,o |
| dropout | 0.1 (higher for overfitting) | 0.05 |
| learning_rate | 3e-5 | 2e-5 |

---

## Citation Strategy

### Source Tagging Format

Training data includes markers that the model learns to output:

```
<<SRC:domain:entity_id>>
```

Examples:
- `<<SRC:compute-resources:delta.ncsa.access-ci.org>>`
- `<<SRC:allocations:project_12345>>`
- `<<SRC:doc:allocations-guide.pdf:page-15>>`

### Post-Processing Flow

```
Model output with <<SRC:...>> markers
           │
           ▼
Post-processor extracts markers
           │
           ▼
Look up in source registry (ID → URL mapping)
           │
           ▼
Response with clickable links
```

### Source Registry

Maps citation domains to user-facing URLs:

| Domain | URL Pattern | Example |
|--------|-------------|---------|
| compute-resources | `https://allocations.access-ci.org/resources/{resource_id}` | [Delta](https://allocations.access-ci.org/resources/delta.ncsa.access-ci.org) |
| software-discovery | `https://sds.access-ci.org/details/{software_name}` | [GROMACS](https://sds.access-ci.org/details/gromacs) |
| events | `https://support.access-ci.org/events/{event_id}` | Training workshops |
| affinity-groups | `https://support.access-ci.org/affinity-groups/{group_name}` | Community groups |
| allocations | `https://allocations.access-ci.org/` | Allocation info |
| doc | Original document URL with anchor | PDF/web documentation |

Unknown sources logged for registry update. Multiple citations per response are expected when answers reference multiple resources.

---

## Evaluation

### Golden Evaluation Set

150-200 representative queries:

| Category | Count |
|----------|-------|
| Resource specs | 40 |
| Software queries | 30 |
| Comparisons | 20 |
| Documentation | 30 |
| Dynamic (should defer) | 30 |
| Edge cases | 20 |

Each query labeled with:
- Expected answer
- Expected tools (if any)
- Query type (STATIC/DYNAMIC)
- Citation expected (yes/no)

### Evaluation Targets

| Metric | Target | Action if Below |
|--------|--------|-----------------|
| Answer accuracy | ≥95% | Add more training data for weak areas |
| Citation presence | ≥90% | Increase citation examples |
| Refusal accuracy | ≥90% | Add more negative examples |
| Latency P95 | <1.5s | Optimize serving, consider smaller model |

---

## Retraining Triggers

### Automatic Triggers

| Condition | Action |
|-----------|--------|
| New compute resource added | Queue retrain |
| >100 software packages updated | Queue retrain |
| New allocation period/cycle | Queue retrain |
| Quarterly schedule | Automatic retrain |

### Manual Triggers

- User feedback indicates stale information
- New documentation added to corpus
- Policy changes affecting allocations/access

### Incremental vs Full Retraining

| Approach | When to Use |
|----------|-------------|
| LoRA Merge + New LoRA | Minor updates (new software) |
| Continue Training | Moderate updates (new resource) |
| Full Retrain | Major changes, architecture shift |

---

## Model Deployment

### Initial Target: Fireworks.ai

For pilot and early production, deploy via **Fireworks.ai**:

| Factor | Benefit |
|--------|---------|
| Serverless | No idle costs during low-traffic phase |
| LoRA upload | Train on GH200, deploy anywhere |
| OpenAI-compatible API | Easy to switch providers later |
| ~$0.20/M tokens | Fine-tuned models same price as base |

Workflow: Train LoRA on GH200 → Upload via `firectl` CLI → Agent calls API

### Future Options

If traffic increases or requirements change:

| Option | When to Consider |
|--------|------------------|
| Together.ai, Anyscale | Better pricing at scale |
| Self-hosted (vLLM/TGI) | If GH200 or similar becomes available for serving |
| Replicate | If need custom scaling behavior |

The OpenAI-compatible API used by Fireworks means migration is mostly a config change.

### Hallucination Detection & Fallback

Fine-tuned models learn citation format well but hallucinate confidently about unknown topics, fabricating plausible-sounding citations (e.g., `<<SRC:compute-resources:stampede2.tacc.access-ci.org>>`).

**Detection**: Validate citations against known MCP entities. Invalid citations indicate hallucination. Testing showed 100% detection rate on hallucinated responses.

**Fallback**: When hallucination is detected, fall back to Q&A RAG retrieval (see below).

### Fallback Architecture

```
User Question
      │
      ▼
Fine-tuned Model (~200ms)
      │
      ▼
Citation Validation
      │
      ├── Valid → Return response (fast path)
      │
      └── Invalid → Q&A RAG Fallback
                          │
                          ├── Match found → Return retrieved answer
                          └── No match → Graceful refusal
```

### Single Corpus for Training & Retrieval

The same Q&A pairs power both systems:

| Use | Purpose |
|-----|---------|
| Fine-tuning | Train model on Q&A pairs |
| RAG retrieval | Embed questions, retrieve matching answers |

**Benefits**:
- One maintenance burden—update pairs, both systems improve
- Consistent answer style and citations
- Self-correcting: RAG retrieves what model should have said
- No drift between training and retrieval

### Q&A RAG

FAQ-style retrieval is a proven pattern. Unlike document RAG:

| Aspect | Document RAG | Q&A RAG |
|--------|--------------|---------|
| Chunking | Must split docs | Not needed |
| Answer quality | LLM synthesizes | Pre-written, verified |
| Citations | Added post-hoc | Already included |
| Latency | Retrieval + generation | Retrieval only |

**How it works**: Embed Q&A questions with sentence embeddings. Match user query to most similar question. Return the paired answer directly.

**Implementation details**: See [hallucination-mitigation-plan.md](https://github.com/necyberteam/access-qa-extraction/docs/hallucination-mitigation-plan.md) in access-qa-extraction.

---

## Training Pipeline

### Implementation Repository

The training pipeline is implemented in [access-qa-training](https://github.com/necyberteam/access-qa-training):

```
access-qa-training/
├── configs/mistral-7b.yaml     # Training configuration
├── scripts/
│   ├── setup_env.sh            # GH200 environment setup
│   └── sync_data.sh            # Data transfer from extraction
└── src/access_qa_training/
    ├── data_loader.py          # JSONL → HF Dataset
    ├── train.py                # LoRA fine-tuning with SFTTrainer
    └── evaluate.py             # Golden eval set evaluation
```

### Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  LOCAL                                 │  GH200                     │
├────────────────────────────────────────┼────────────────────────────┤
│  access-qa-extraction/                 │  access-qa-training/       │
│  └── data/output/*.jsonl ────rsync────▶│  ├── data/raw/*.jsonl      │
│                                        │  ├── qa-train              │
│                                        │  └── models/               │
└────────────────────────────────────────┴────────────────────────────┘
```

---

## Pilot Execution Checklist

### Week 1: Environment
- [ ] Create `access-qa-training` repo
- [ ] SSH to GH200, run `scripts/setup_env.sh`
- [ ] Verify PyTorch sees GPU (`torch.cuda.is_available()`)
- [ ] Test loading Mistral-7B at bf16

### Week 2: Data
- [ ] Run extraction on compute-resources (~236 pairs done)
- [ ] Run extraction on software-discovery
- [ ] Sync JSONL to GH200 (`scripts/sync_data.sh push`)
- [ ] Verify data loader works
- [ ] Create initial golden eval set (20-30 questions)

### Week 3: Training
- [ ] Run first training job (1 epoch, verify completion)
- [ ] Run full training (3 epochs)
- [ ] Log to wandb, verify metrics

### Week 4: Evaluation
- [ ] Run `qa-eval` on trained model
- [ ] Compare to base model (before fine-tuning)
- [ ] Document results
- [ ] GO/NO-GO decision on expanding pipeline

---

## Model Versioning

Track model versions and their relationship to training data for rollback and auditing.

### Version Identifier

Format: `access-qa-v{major}.{minor}.{patch}`

| Component | When to Increment |
|-----------|-------------------|
| major | Architecture change (MoE → Dense, new base model) |
| minor | Retrained with significant new data (>10% new pairs) |
| patch | Minor update, bug fix, configuration change |

### What to Track Per Version

| Field | Description | Example |
|-------|-------------|---------|
| `model_version` | Version identifier | `access-qa-v1.2.0` |
| `base_model` | Foundation model fine-tuned | `meta-llama/Llama-3-8B` |
| `dataset_version` | Training data snapshot ID | `dataset-2025-01-05-abc123` |
| `training_config` | LoRA params, learning rate, etc. | (stored as JSON) |
| `eval_metrics` | Accuracy, citation rate, latency | (stored as JSON) |
| `created_at` | Training completion timestamp | `2025-01-05T14:30:00Z` |
| `created_by` | Who triggered training | `automation` or `jsmith` |
| `status` | Deployment status | `deployed`, `archived`, `failed` |

### Dataset Snapshots

Each training run uses a frozen dataset snapshot:

| Field | Description |
|-------|-------------|
| `dataset_version` | Unique ID (hash or timestamp) |
| `qa_pair_ids` | List of Q&A pair IDs included |
| `qa_pair_count` | Total pairs in snapshot |
| `source_breakdown` | Count by source type |
| `domain_breakdown` | Count by domain |
| `created_at` | Snapshot creation time |

### Version Registry

Maintain a `model_versions.json` registry:

| Purpose | Use Case |
|---------|----------|
| Rollback | If new model quality degrades, redeploy previous version |
| Audit | Trace which Q&A pairs influenced a specific answer |
| Comparison | Compare metrics across versions |
| Debugging | Identify which training data caused a problematic response |

### Linking Model to Responses

Each model response in production includes:
- `model_version` in response metadata
- Logged to Grafana for correlation with feedback

When a user reports a bad answer:
1. Look up `model_version` from response metadata
2. Find `dataset_version` from registry
3. Search Q&A pairs that may have influenced the answer
4. Add corrected pair and queue for next training

---

## Open Questions

1. **Model serving infrastructure**: vLLM vs TGI?
2. **Learned classifier**: Train query classifier in Phase 3 or defer?
3. **Version storage**: Git-based versioning vs database registry?
