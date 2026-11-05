# Quantization Deep Dive

Quantization is the process of reducing the precision of model weights (e.g., from 16-bit to 4-bit) to save memory and increase inference speed. This is the primary tool for deploying large models on consumer and single-GPU hardware, and in 2026 it is also the default for datacenter serving, where FP4 weights with an FP8 KV cache are the headline configuration.

## Table of Contents

- [The Precision-Performance Tradeoff](#the-precision-performance-tradeoff)
- [Quantization Methods (NF4, AWQ, FP8, FP4)](#quantization-methods)
- [File and Serving Formats](#file-and-serving-formats)
- [KV Cache Quantization (The VRAM Saver)](#kv-cache-quantization-the-vram-saver)
- [Quantization-Aware Training](#quantization-aware-training-qat)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Precision-Performance Tradeoff

Traditional models use **BF16** (16-bit). Quantization seeks to reduce this to **8-bit (FP8)**, **4-bit (Int4, NF4, NVFP4, MXFP4)**, or even **~1.6-bit ternary** (BitNet b1.58 and its descendants).

| Precision | Bits | Weight size (8B Model) | Quality Loss | Hardware |
|-----------|------|------------------------|--------------|-------------------|
| **BF16** | 16 | 16 GB | 0% (Baseline) | All Modern |
| **FP8** | 8 | 8 GB | < 1% | H100 / H200 / B200 / RTX 4090 and later; AMD MI300 and later |
| **4-bit integer (NF4, AWQ, GPTQ)**| 4 | 4.5-5 GB | 1-2% | All Modern |
| **4-bit float (NVFP4 / MXFP4)** | 4 | ~4.5 GB | Small with good calibration and kernels | NVFP4: Blackwell, Rubin. MXFP4: AMD MI350/MI455X and Blackwell |
| **2-bit (naive PTQ)** | 2 | 2.5 GB | 10-15% | Research / Specialized |
| **Ternary (trained or adapted for it)** | ~1.6-1.7 | ~1.7 GB | Vendor claims near-FP16 on some 27B-class models | Custom kernels only |

The 2-bit and ternary rows show why "bits" alone does not predict quality: naive post-training quantization to 2-bit wrecks reasoning, while a model adapted for ternary weights can hold up. PrismML's Ternary Bonsai 2 27B (September 2026, Apache 2.0, derived from Qwen3.8-27B) averages 1.72 bits per weight in a 5.95 GB file and reports 98.2% of FP16 quality across 14 thinking-mode benchmarks (vendor-reported). It needs PrismML's forks of llama.cpp and MLX; stock llama.cpp cannot load it, which is the usual price of exotic formats.

---

## Quantization Methods

### 1. NF4 (NormalFloat4)
The gold standard for fine-tuning (QLoRA). It assumes weights follow a normal distribution and maps them to a set of 16 values.

### 2. AWQ (Activation-aware Weight Quantization)
AWQ observes that about **1% of weights are "salient"** (they multiply large activations) and dominate quantization error. Keeping them in higher precision would fix that but needs hardware-unfriendly mixed precision, so AWQ instead **scales the salient input channels up before quantization** (and folds the inverse scale into the preceding op), which protects them while every weight still ends up in 4-bit.
- **Pro**: Better accuracy than GPTQ at the same bit width in many settings, with no backpropagation and a small calibration set.

### 3. FP8
Hardware-native on Hopper and later through NVIDIA's Transformer Engine (and on AMD MI300-class parts).
- **Why it wins**: It provides the speed of Int8 but with far more dynamic range, making it stable for both training and inference. E4M3 is used for weights and activations, E5M2 where range matters more (gradients).

### 4. FP4: NVFP4 and MXFP4
The 2026 headline serving precision. Both are 4-bit floats (E2M1) with block scaling; they differ in block size and scale format:

| Format | Block size | Scale | Where |
|--------|-----------|-------|-------|
| **NVFP4** | 16 values | FP8 (E4M3) per block plus an FP32 per-tensor scale | NVIDIA Blackwell, Rubin |
| **MXFP4** (OCP Microscaling) | 32 values | Power-of-two (E8M0) per block | AMD MI350 and MI455X, NVIDIA Blackwell |

Evidence it is mainstream: NVIDIA rates a Vera Rubin NVL72 rack at 3,600 PFLOPS NVFP4 for inference (a sparse figure; the dense training rating is 2,520 PFLOPS); MLPerf Inference v6.1 (September 2026) had submissions moving from FP8 to FP4, including NVFP4 weights with an FP8 KV cache; DeepSeek-V4 ships FP4 MoE expert weights with FP8 for most other parameters. Checkpoints are increasingly portable across vendors: SGLang v0.5.18 requantizes NVFP4 checkpoints to MXFP4 at load on AMD, reporting 97.5% to 100.2% GSM8K recovery across five large MoE models.

**The kernel caveat.** Lower precision only wins when the kernel path is good. NVIDIA's own Dynamo recipe for Nemotron 3.5 Lightning on B200/GB200 measured BF16 **80% faster** than NVFP4 with Marlin kernels and still 12% faster than NVFP4 with CuTeDSL kernels. vLLM v0.30.0 made FlashInfer CuTeDSL the default NVFP4 W4A16 path on SM100/103 instead of Marlin, so the same checkpoint can change speed across an engine upgrade. Benchmark the exact model, engine version, and kernel backend before you migrate.

---

## File and Serving Formats

### GGUF (llama.cpp)
- **Deployment**: CPU + GPU offloading; the local format for llama.cpp, Ollama, and LM Studio.
- **Pros**: Cross-platform (Mac, Linux, Windows), single file, highly portable.
- **Cons**: Slower than GPU-native serving formats under concurrency.

### EXL2 / EXL3 (ExLlama)
- **Deployment**: GPU-only (NVIDIA), single-user local inference.
- **Pros**: Very fast on consumer NVIDIA GPUs with flexible bits per weight; ExLlamaV3's EXL3 format is a streamlined variant of QTIP.
- **Cons**: NVIDIA-only and not a multi-tenant serving format.

### Datacenter serving formats
- **AWQ / GPTQ** int4 checkpoints, served with Marlin-family kernels in vLLM, SGLang, and TensorRT-LLM.
- **FP8, NVFP4, MXFP4** checkpoints produced with tools such as NVIDIA Model Optimizer or llm-compressor, often published officially by the model vendor (for example IBM's Granite 4.2 ships FP8, MXFP4, NVFP4, GGUF and MLX builds).
- **bitsandbytes (NF4)** is a training-time format first: vLLM moved bitsandbytes to an out-of-tree plugin in v0.28.0 (August 2026). Merge QLoRA adapters and requantize to a serving format before production.

---

## KV Cache Quantization (The VRAM Saver)

In long-context serving, the **KV Cache** often consumes more VRAM than the model weights themselves. For Llama 3.1 8B (32 layers, 8 KV heads, head dim 128):

- **BF16 KV cache**: 128 KiB per token, so 128K tokens ≈ 16 GB (as much as the BF16 weights) and 1M tokens ≈ 128 GB.
- **FP8 KV cache**: half that (64 KiB per token); **4-bit KV**: a quarter.

**Nuance**: Serving engines (vLLM, SGLang, TensorRT-LLM) quantize KV on write, so FP8 KV roughly doubles the number of concurrent long-context requests per GPU at small quality cost, which is why FP4 weights plus FP8 KV is the default pairing. The bigger lever in 2026 is architecture, not dtype: DeepSeek V4.1-Flash stores FP4 main KV with compressed sparse attention for about 890 bytes per token of global KV, versus 320 KiB per token for a BF16 Llama-3.1-70B. See [KV Cache and Context Caching](../04-inference-optimization/02-kv-cache-and-context-caching.md).

---

## Quantization-Aware Training (QAT)

Instead of quantizing a model *after* it's trained (Post-training Quantization), QAT simulates quantization *during* the training process.
- **Result**: The model learns to compensate for the lost precision.
- **Status**: The default for small and on-device models where 4-bit is the target. Google ships QAT int4 builds of Gemma (including Gemma 4 12B), and Apple says its third-generation foundation models used quantization-aware training. Quantization-aware distillation, where the full-precision model is the teacher, is the same idea with a stronger signal (see [Knowledge Distillation](05-knowledge-distillation.md#quantization-aware-distillation)).

---

## Interview Questions

### Q: Why do we use NF4 instead of standard Float4 for QLoRA?

**Strong answer:**
Standard Float4 has a fixed grid that doesn't map well to the actual distribution of LLM weights, which typically follow a zero-centered normal distribution. NF4 (NormalFloat4) is a data type that is mathematically optimized so that each quantization bin contains an equal number of values from the normal distribution. This prevents "clustering" of weights and ensures that the model preserves as much information (entropy) as possible, leading to significantly higher accuracy than standard 4-bit integers.

### Q: How does AWQ differ from GPTQ?

**Strong answer:**
GPTQ quantizes layer by layer and uses approximate second-order (Hessian) information from a calibration set to update the remaining weights so each layer's output error stays small. AWQ is activation-aware: it uses calibration activations to find the roughly 1% of salient input channels, then scales those channels up before quantization so their relative rounding error shrinks, folding the inverse scale into the previous operation. Everything still ends up in 4-bit, which keeps kernels simple; the common misconception is that AWQ stores the salient weights in higher precision. AWQ needs no backpropagation or weight reconstruction, generalizes well across domains because it overfits the calibration set less, and often beats GPTQ at 4-bit and below.

### Q: The team wants to move serving from FP8 to NVFP4 to cut cost. What do you check?

**Strong answer:**
Four things. First, hardware: NVFP4 needs Blackwell or Rubin tensor cores; on AMD the equivalent is MXFP4, and some engines can requantize NVFP4 checkpoints to MXFP4 at load, so check quality recovery on that path. Second, kernels: lower precision is not automatically faster. NVIDIA's own Dynamo recipe measured BF16 80% faster than NVFP4 on Marlin kernels and 12% faster on CuTeDSL, so I would benchmark the exact model, engine version, and kernel backend at our real batch sizes and context lengths, on a throughput-versus-interactivity curve rather than one number. Third, quality: run our task evals, not just perplexity, with special attention to long-context and reasoning tasks, and prefer a vendor-published or QAT checkpoint over our own PTQ. Fourth, the KV cache: weights shrink 2x, but at long context the KV cache may dominate memory, so pair FP4 weights with an FP8 KV cache and recompute how many concurrent sequences fit. The decision is cost per million tokens at our SLO, measured, not the bit width.

---

## References
- Dettmers et al. "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)
- Frantar et al. "GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers" (2022)
- Lin et al. "AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration" (2023)
- Ma et al. "The Era of 1-bit LLMs: All Large Language Models are in 1.58 Bits" (BitNet b1.58, 2024)
- Open Compute Project. "OCP Microscaling Formats (MX) Specification" (2023)
- NVIDIA. [Vera Rubin NVL72](https://www.nvidia.com/en-us/data-center/vera-rubin-nvl72/)

---

*Next: [Training Reasoning Models: RLVR and GRPO](08-rlvr-and-reasoning-models.md)*
