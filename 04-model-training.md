# Model Training

> **Part of the Data Pipeline**: [Training Data](./02-training-data.md) → [Review System](./03-review-system.md) → This doc
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

Maps source IDs to display info:

| Source ID | Display Name | URL |
|-----------|--------------|-----|
| compute-resources:delta.ncsa.access-ci.org | Delta Hardware Specifications | https://access-ci.org/resource/delta/ |
| doc:allocations-guide.pdf:page-15 | ACCESS Allocations Guide, Page 15 | https://access-ci.org/docs/allocations-guide.pdf#page=15 |

Unknown sources logged for registry update.

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

## Model Serving

### Options

| Server | Notes |
|--------|-------|
| vLLM | Optimized for high throughput |
| TGI (Text Generation Inference) | HuggingFace, good for single-model |

Consider quantization (GPTQ/AWQ) for faster inference if latency targets not met.

### Confidence-Aware Fallback

When model response shows low confidence:
- Check for hedging language ("I'm not sure", "might be")
- Check for missing expected citations
- Fall back to live MCP call

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
