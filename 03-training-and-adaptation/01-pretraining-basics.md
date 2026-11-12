# Pretraining Basics

Pretraining is the most computationally expensive phase of building an LLM, where a model learns general knowledge and language patterns from massive datasets.

## Table of Contents

- [The Pretraining Objective](#the-pretraining-objective)
- [Data Curriculum and Quality](#data-curriculum-and-quality)
- [Scaling Laws (Inference-Optimal)](#scaling-laws-training-vs-inference-optimal)
- [Computational Requirements](#computational-requirements)
- [Training Stability](#training-stability)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Pretraining Objective

Most modern LLMs are **Decoder-only** and use **Causal Language Modeling (CLM)**:

```python
# Objective: Minimize Cross-Entropy Loss
Loss = -sum(log P(token_i | token_1, ..., token_{i-1}))
```

The model predicts the next token given the context. This simple objective, at scale, leads to emergent reasoning capabilities. Many current models add an auxiliary **multi-token prediction (MTP)** objective (DeepSeek-V3 popularized it), which both improves training signal and leaves behind draft heads that serving engines reuse for [speculative decoding](../04-inference-optimization/03-speculative-decoding.md).

---

## Data Curriculum and Quality

The focus has shifted from "More Data" to "Better Curriculum."

### Token Budgets at the Frontier
Disclosed pretraining budgets for open models now run from about 15T to 45T tokens: Llama 3 used about 15T, DeepSeek-V3 14.8T, Qwen3 about 36T, Meta says the Llama 4 mixture was more than 30T, and Thinking Machines Lab's Inkling (July 2026) was pretrained on 45T multimodal tokens (vendor-reported). Closed labs (OpenAI, Anthropic, Google) do not publish token counts, so treat any specific figure for GPT, Claude, or Gemini as a guess. At this scale, **Deduplication** and **Quality Filtering** are the primary differentiators.

### Data Mixture Standard
| Component | Percentage | Purpose |
|-----------|------------|---------|
| Web (CommonCrawl) | 50-60% | General knowledge, diverse styles |
| Code (GitHub, StackOverflow)| 15-20% | **Critical for Logic & Reasoning** |
| Books (Project Gutenberg) | 10% | Narrative coherence, long context |
| Academic (ArXiv, PubMed) | 10% | Specialized technical knowledge |
| Synthetic (Model-generated) | 5-10% | Math, Logic, and specific instruction paths |

These shares are a typical general-purpose mix, not a law. Small reasoning-focused models go much further on synthetic data: Microsoft's Phi-4 report allocates 40% of pretraining tokens to synthetic data (see [Synthetic Data Generation](06-synthetic-data-generation.md)).

**Nuance: The "Code Effect":**
Research shows that increasing code in the pretraining mix improves a model's performance on **non-coding** reasoning tasks (e.g., math, logic puzzles) by teaching structured thinking.

**Provenance is now a design requirement.** Courts are separating whether training on a work is lawful from how the work was acquired (the Bartz v. Anthropic ruling held training lawful but piracy-sourced acquisition not, and the case settled for $1.5B). Keep acquisition records per corpus, not just a license field.

---

## Scaling Laws: Training vs. Inference Optimal

### The Chinchilla Paradigm (2022-2024)
`Data Tokens (D) ≈ 20 * Parameters (N)`
For a 70B model, this suggests ~1.4T tokens.

### The Inference-Optimal Paradigm
Modern models (Llama 3, Llama 4, Qwen3) are **heavily overtrained** relative to Chinchilla.
- **Why?**: Training cost is paid once; inference cost is paid billions of times.
- **Result**: Small models (8B) are now trained on 15T+ tokens, making them as capable as older 70B models but much cheaper to serve.

| Strategy | Token/Param Ratio | Best For |
|----------|-------------------|----------|
| Chinchilla | 20:1 | Research / Proof of Concept |
| **Inference-Optimal** | **200:1 to 2,000:1+**| Production deployment (Llama 3 8B on ~15T tokens is ~1,900:1) |

---

## Computational Requirements

The standard back-of-envelope: **training FLOPs ≈ 6 × N × D**, where N is parameters touched per token and D is training tokens (2ND for the forward pass, 4ND for the backward pass).

| Model | N (per token) | D | ≈ FLOPs | Reported compute |
|-------|---------------|---|---------|------------------|
| Llama 3.1 405B (dense) | 405B | 15.6T | 3.8 × 10^25 | 30.84M H100 GPU-hours (Meta model card) |
| DeepSeek-V3 (MoE, 671B total) | 37B active | 14.8T | ~3.3 × 10^24 | 2.788M H800 GPU-hours (DeepSeek report) |

Two lessons interviewers probe:
- **For MoE, use active parameters**, not total. Counting DeepSeek-V3's 671B total parameters would overstate its compute by about 18x. It uses 37B active parameters per token against Llama 3.1 405B's 405B dense, so on similar token counts its training FLOPs are about a tenth.
- **GPU-hours = FLOPs / (peak FLOPs × MFU)**. Model FLOPs utilization of 35-45% is typical for well-tuned large dense runs in BF16; Meta reports 390 TFLOPs per GPU in FP8 on 32K GPUs for Llama 4 Behemoth. Memory for training (weights, gradients, optimizer state at ~16 bytes per parameter in mixed precision before sharding) is what forces ZeRO/FSDP sharding and pipeline parallelism, long before FLOPs run out.

---

## Training Stability

Training at the "Ultra" scale (100k+ GPUs) faces massive stability issues.

### 1. Loss Spikes
Sudden jumps in loss that can ruin a training run.
- **Standard fix**: **Periodic Checkpointing** and **Automatic Rollbacks**.
- **Architecture fix**: **Residual Scaling** (initializing weights such that the residual branch starts at near-zero).

### 2. Precision: BF16, FP8, and FP4
- **BF16**: Still the stable default for most teams.
- **FP8 mixed precision**: Standard at frontier scale since DeepSeek-V3 showed it at 671B (fine-grained block-wise scaling, higher-precision accumulation, sensitive ops kept in BF16/FP32); Meta trained Llama 4 in FP8. FP8 doubles peak tensor-core throughput on H100/B200, but end-to-end gains are smaller because not every op runs in FP8.
- **FP4 (NVFP4 / MXFP4)**: The emerging frontier. NVIDIA trained a 12B model on 10T tokens in NVFP4 with results comparable to an FP8 baseline, using random Hadamard transforms and stochastic rounding (arXiv:2509.25149). NVIDIA rates a Vera Rubin NVL72 rack at 2,520 PFLOPS NVFP4 for training, and The Register reports Meta's MTIA 400 as a training-first part rated at 12 PFLOPS MXFP4. Expect FP4 pretraining recipes to spread through 2027; treat them as a recipe-and-kernel question, not a free 2x.

---

## Interview Questions

### Q: Why train an 8B model on 15T tokens if Chinchilla says 160B tokens is optimal?

**Strong answer:**
Chinchilla optimality focuses on the best use of a fixed **training** compute budget. However, in production, we care about the **Total Cost of Ownership (TCO)**, which is dominated by inference. By overtraining a small model, we "bake in" more intelligence into fewer parameters. This results in a model that is significantly more efficient to serve (higher TPS, lower VRAM) while matching the quality of much larger models trained to the Chinchilla point. The returns diminish (each extra trillion tokens buys less), so the stopping point is a cost decision, not a law.

### Q: What is the "curriculum" in LLM pretraining?

**Strong answer:**
Curriculum refers to the order and mixture of data. A common modern pattern is:
1. **General Knowledge Phase:** 80% of tokens (Web, Books).
2. **Reasoning Focus Phase:** 15% tokens (Code, Math, Logic).
3. **High-Quality "Cooling" Phase:** The last 1-5% of tokens are extremely high-quality, human-curated, or textbook data. This "cooling" phase helps the model jitter less and follow instructions better before any fine-tuning starts.

Labs increasingly run a longer, reasoning-dense version of phase 3 as a deliberate **mid-training** stage before RL, because it raises the base competence that RL then sharpens (see [RLVR and Reasoning Models](08-rlvr-and-reasoning-models.md)).

### Q: Estimate the compute to pretrain a 70B dense model on 15T tokens.

**Strong answer:**
6 × 70e9 × 15e12 ≈ 6.3 × 10^24 FLOPs. An H100 delivers about 1 PFLOPS dense BF16 at peak; at 40% MFU that is ~4 × 10^14 FLOPs per second, so roughly 1.6 × 10^10 GPU-seconds, about 4.4M GPU-hours. On a 16K-GPU cluster that is about 11-12 days of pure compute, before restarts, evals, and failed runs, which usually add 20-50%. As a sanity check, Meta reports 7.0M H100 GPU-hours for Llama 3.1 70B. If the model were an MoE with the same active parameters, the FLOPs would be the same, but memory and communication (expert parallelism, all-to-all) would dominate the engineering.

---

## References
- Kaplan et al. "Scaling Laws for Neural Language Models" (2020)
- Hoffmann et al. "Training Compute-Optimal Large Language Models" (Chinchilla, 2022)
- Meta AI. "The Llama 3 Herd of Models" (2024) and "The Llama 4 herd" (blog, 2025)
- DeepSeek-AI. "DeepSeek-V3 Technical Report" arXiv:2412.19437 (2024)
- NVIDIA. "Pretraining Large Language Models with NVFP4" arXiv:2509.25149 (2025)

---

*Next: [Fine-Tuning Strategies](02-fine-tuning-strategies.md)*
