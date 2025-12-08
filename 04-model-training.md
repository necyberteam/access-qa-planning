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
| **Hostname** | gh1-internal.ccs.uky.edu |
| **Jump Host** | 128.163.202.89 |
| **GPU** | NVIDIA GH200 480GB (Grace Hopper Superchip) |
| **VRAM** | 98 GB HBM3 |
| **CUDA** | 12.6 |
| **Driver** | 555.42.06 |

This hardware allows training large models at bf16 precision without quantization.

---

## Model Selection

### Pilot Comparison Strategy

Compare MoE (Mixture of Experts) vs Dense architectures before committing:

| Model | Type | Size | Active Params | VRAM (bf16) | Role |
|-------|------|------|---------------|-------------|------|
| **Qwen2-MoE-14B** | MoE | 14B | 2.7B | ~28GB | Pilot - MoE candidate |
| **Llama 3 8B** | Dense | 8B | 8B | ~16GB | Pilot - Dense candidate |

### Production Scale-Up

After pilot determines winner:

| If Winner | Scale To | VRAM |
|-----------|----------|------|
| MoE | Mixtral 8x7B | ~90GB |
| Dense | Llama 3 70B | ~140GB |

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

## Pilot Execution Checklist

### Data Preparation
- [ ] Export "good" Q&A pairs from existing user Q&A database
- [ ] Run MCP extraction on compute-resources and software-discovery
- [ ] Generate Q&A pairs from key documentation sections
- [ ] Write refusal/defer-to-live examples for dynamic queries
- [ ] Document final counts after extraction

### Training
- [ ] Set up environment on GH200 (Python venv, CUDA 12.6, PyTorch)
- [ ] Train Qwen2-MoE-14B on pilot dataset
- [ ] Train Llama 3 8B on same dataset
- [ ] Track with wandb

### Evaluation
- [ ] Run golden eval set on both models
- [ ] Compare answer accuracy
- [ ] Compare citation presence
- [ ] Compare inference latency
- [ ] Document architecture decision

### Decision
- [ ] GO/NO-GO on full implementation
- [ ] If GO: proceed to full training with selected architecture

---

## Open Questions

1. **Model serving infrastructure**: vLLM vs TGI?
2. **Learned classifier**: Train query classifier in Phase 3 or defer?
