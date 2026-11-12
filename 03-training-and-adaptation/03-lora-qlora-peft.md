# LoRA, QLoRA, and PEFT

Parameter-Efficient Fine-Tuning (PEFT) is the industry standard for adapting LLMs. This chapter covers the mechanics and advanced variants of LoRA and other PEFT methods.

## Table of Contents

- [The PEFT Revolution](#the-peft-revolution)
- [LoRA Mechanics](#lora-mechanics)
- [QLoRA: 4-bit Fine-Tuning](#qlora-4-bit-fine-tuning)
- [Advanced Variants (DoRA, VeRA, RS-LoRA)](#advanced-variants)
- [Multi-LoRA Serving (Adapters)](#multi-lora-serving-adapters)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The PEFT Revolution

Full fine-tuning of frontier-scale open models (Llama 3.1 405B dense, DeepSeek V4 Pro at roughly 1.6T total parameters, Kimi K3 at 2.8T total / 104B active) is out of reach for most enterprises, and closed frontier models cannot be fully fine-tuned by customers at all. PEFT allows:
1. **Memory Efficiency**: Train a 70B model on a single 80 GB GPU with QLoRA.
2. **Speed**: 2x faster training by updating <1% of weights.
3. **Modularity**: Swap "skills" (adapters) onto a shared base model without reloading weights.

---

## LoRA Mechanics

LoRA (Low-Rank Adaptation) injects trainable rank-decomposition matrices into the transformer layers.

```python
# The LoRA Equation for a Weight Matrix W:
h = Wx + (BA)x * (alpha/r)
```
- **W**: Pretrained weights (Frozen, Gradient = None)
- **A, B**: LoRA adapters (Trainable)
- **r**: Rank (e.g., 8, 16, 64)
- **alpha**: Scaling factor (typically 2 * rank)

### Principal Nuance: Target Modules
Historically, we only targeted query/value projections (`q_proj`, `v_proj`).
**Modern standard**: Target **all** linear layers (`q, k, v, o, gate, up, down`) for maximum stability and performance, even at lower ranks.

---

## QLoRA: 4-bit Fine-Tuning

QLoRA pushes efficiency further by quantizing the base model to 4-bit (NF4) while maintaining 16-bit gradients.

| Optimization | Method | Benefit |
|--------------|--------|---------|
| **NF4 Quantization** | Normalized Float 4 | Better info density than standard Int4 |
| **Double Quant** | Quantizing the quant constants | Saves ~0.37 bits per parameter (about 3 GB on a 65B model) |
| **Paged Optimizers** | NVIDIA Unified Memory | Pages optimizer state to CPU RAM during memory spikes instead of OOM |

---

## Advanced Variants

### 1. DoRA (Weight-Decomposed Low-Rank Adaptation)
DoRA decomposes the weight update into **Magnitude** and **Direction**.
- **Result**: The paper reports consistent gains over LoRA at the same rank on LLaMA, LLaVA, and VL-BART tasks, closing part of the gap to full fine-tuning for a small extra training cost.
- **Why it wins**: It allows the model to adjust how much it changes vs. what it's changing independently.

### 2. VeRA (Vector-based Random Matrix Adaptation)
Instead of trainable low-rank matrices `A` and `B`, VeRA shares one pair of frozen random matrices across layers and trains only small per-layer scaling vectors.
- **Efficiency**: Each layer trains only two small vectors, so VeRA needs a **small fraction** of LoRA's trainable parameters; how small depends on the LoRA rank you compare against (the gap grows with rank).
- **Use Case**: Massive-scale Multi-LoRA serving.

### 3. RS-LoRA (Rank-Stabilized LoRA)
Uses a scaling factor of `alpha / sqrt(r)`.
- **Benefit**: Allows you to increase rank (to 256+) without the model becoming unstable or requiring a lower learning rate.

---

## Multi-LoRA Serving (Adapters)

Production systems now serve one base model (e.g., Llama 3.3 70B or Qwen3.8-27B) and dynamically swap adapters in the same batch.

```python
# vLLM / SGLang multi-LoRA pattern:
# Request 1 -> Base + Finance_Adapter
# Request 2 -> Base + Legal_Adapter
# Request 3 -> Base + Medical_Adapter
```
**The Tech**: batched LoRA kernels (Punica's SGMV) let one decode step apply different adapters to different rows of the batch, and S-LoRA's **Unified Paging** manages adapter weights and KV cache in one paged pool, with idle adapters kept in host memory. S-LoRA reported serving thousands of adapters with up to 30x the throughput of HuggingFace PEFT and up to 4x that of vLLM at the time (2023). In vLLM, `--enable-lora`, `--max-loras` (adapters resident per batch), and `--max-lora-rank` are the capacity knobs; overhead grows with rank and with the number of distinct adapters per batch, so benchmark your own mix.

**Check feature interactions before you commit.** LoRA support often lags other serving features: vLLM's adaptive speculative verification, for example, did not support LoRA when it launched in August 2026. If a tenant's adapter silently disables speculative decoding or a fast kernel path, that tenant's latency profile differs from the base model's.

---

## Interview Questions

### Q: Why is the LoRA alpha parameter usually set to 2x the rank?

**Strong answer:**
The update is scaled by `alpha/r`, with B zero-initialized and A random, so training starts exactly at the base model. The original LoRA paper fixes `alpha` (to the first rank tried) so that `alpha/r` shrinks as rank grows, which roughly normalizes the update size and lets you change `r` without retuning the learning rate. `alpha = 2r` is a community heuristic, not a law: it pins the scale at 2, giving a stronger update that tends to train faster at common ranks (8 to 64). The catch is at high rank: rsLoRA shows `alpha/r` over-shrinks the update as `r` grows and that `alpha/sqrt(r)` is the stable scaling, which is why high-rank runs (256+) use rsLoRA. In an interview, the point is that `alpha` and learning rate are coupled; pick one convention and tune the other.

### Q: What is DoRA, and why would you use it over standard LoRA?

**Strong answer:**
DoRA (Weight-Decomposed Low-Rank Adaptation) is a 2024 technique that separates the pretrained weight into magnitude and direction components, similar to Weight Normalization, and applies LoRA only to the direction while training the magnitude vector directly. Standard LoRA couples the two, so it struggles to make small directional changes with large magnitude changes (or the reverse), a pattern full fine-tuning shows. The paper reports consistent gains over LoRA at the same rank for a modest increase in training compute, and the magnitude vector can be merged back at inference, so serving cost is unchanged. I would treat it as a cheap A/B against LoRA on high-stakes domain adaptation, not an automatic default.

---

## References
- Hu et al. "LoRA: Low-Rank Adaptation of Large Language Models" (2021)
- Liu et al. "DoRA: Weight-Decomposed Low-Rank Adaptation" (2024)
- Dettmers et al. "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)
- Kopiczko et al. "VeRA: Vector-based Random Matrix Adaptation" (2024)
- Sheng et al. "S-LoRA: Serving Thousands of Concurrent LoRA Adapters" (2023)
- Kalajdzievski. "A Rank Stabilization Scaling Factor for Fine-Tuning with LoRA" (rsLoRA, 2023)

---

*Next: [RLHF and DPO](04-rlhf-and-dpo.md)*
