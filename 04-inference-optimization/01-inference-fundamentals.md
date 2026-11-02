# Inference Fundamentals

Inference is the process of generating predictions from a trained model. Inference optimization has shifted from "simple speedups" to "architectural efficiency" to handle reasoning-heavy and agentic workloads on Hopper (H100/H200), Blackwell (B200/B300), and now Vera Rubin class hardware, which NVIDIA moved into full production in May 2026.

## Table of Contents

- [The Two Phases of Inference](#the-two-phases-of-inference)
- [Bottlenecks: Compute-Bound vs. Memory-Bound](#bottlenecks-compute-bound-vs-memory-bound)
- [Performance Metrics: TTFT, TPOT, and Goodput](#performance-metrics)
- [Hardware-Enabled Optimizations (FP8 and FP4)](#hardware-enabled-optimizations-fp8-and-fp4)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Two Phases of Inference

LLM inference is not a single operation; it consists of two distinct computational phases.

### 1. The Prefill Phase (Prompt Processing)
The model processes the entire input prompt in a single pass.
- **Computation**: High-parallelism matrix multiplications.
- **Bottleneck**: **Compute-bound** (limited by GPU TFLOPS).
- **Time Complexity**: $O(N)$ for the linear layers and $O(N^2)$ for full attention, where $N$ is input length (but parallelized).

### 2. The Decode Phase (Token Generation)
The model generates tokens one by one, where each token depends on the previous ones.
- **Computation**: Matrix-vector work: every weight (and the sequence's KV cache) is read from memory once per step to produce one token per sequence.
- **Bottleneck**: **Memory-bound** (limited by memory bandwidth).
- **Time Complexity**: $O(M)$ where $M$ is output length (sequential).

Some 2026 architectures budget the two phases differently on purpose: DeepSeek V4.1-Flash activates about 8B parameters per token during prefill and 16B during decode. That breaks the usual "active parameters × tokens" cost math and is one more reason to size prefill and decode capacity separately.

---

## Bottlenecks: Compute-Bound vs. Memory-Bound

Understanding where your system is bottlenecked is critical for choosing the right optimization.

| Phase | Bottleneck | Why? | Primary Optimization |
|-------|------------|------|----------------------|
| **Prefill** | Compute (FLOPs) | Parallel processing saturates the GPU's arithmetic units. | FlashAttention, FP8/FP4 precision, prefix caching. |
| **Decode** | Memory Bandwidth | Weights must be loaded from HBM for *every single token*. | Quantization (4-bit), GQA/MLA, Batching, speculative decoding. |

**The Memory Wall insight**
Memory bandwidth has not scaled as fast as compute. An H100 offers roughly 1 PFLOPS of dense BF16 against 3.35 TB/s of HBM3, about 300 FLOPs per byte. A Vera Rubin GPU, per NVIDIA's rack figures (3,600 PFLOPS NVFP4 across 72 GPUs, 19.2 TB/s of HBM4 each), offers about 50 PFLOPS of NVFP4 against 19.2 TB/s, roughly 2,600 FLOPs per byte. A decode step needs a large batch (or speculative tokens) to use that compute, which makes decode the primary target for production optimization and makes HBM capacity, now supply-constrained (Micron expects demand above supply through 2028, per The Register), a capacity-planning input.

---

## Performance Metrics

| Metric | Full Form | Goal | Importance |
|--------|-----------|------|------------|
| **TTFT** | Time To First Token | < 200ms | User-perceived responsiveness. |
| **TPOT / ITL** | Time Per Output Token / Inter-Token Latency | < 30ms | Reading speed and conversational flow. |
| **Throughput** | Tokens/Second (Agg) | Maximize | Determining cost per query. |
| **Goodput** | Requests/sec meeting **both** TTFT and ITL SLOs | Maximize | What you can actually sell at your SLO. |
| **Latency** | End-to-End Time | < 2.0s | Total turn-around for the agent. |

### Measure on the Pareto Curve, Not at Batch 1

Throughput and per-user speed trade off against each other through batch size, so any single number hides the operating point. Vendor "tokens per second" headlines are usually batch-1 top speeds that customers rarely see under load. Compare systems on a **throughput-versus-interactivity curve** at your SLO and on traffic that looks like yours.

How much the operating point matters: SemiAnalysis's InferenceX AgentX scenario (agentic, multi-turn, high prefix reuse) reports Vera Rubin NVL72 at about **67x** the throughput per TCO of GB300 on DeepSeek V4 Pro at 170 tok/s per user under ownership-cost assumptions, but only **62% more** total tokens than GB300 NVL72 at a more realistic 80 tok/s P90 on 3-year rental pricing. Same hardware, same model, a 40x difference in the headline, purely from the operating point and the cost model. Independent measurement shrinks it further: Artificial Analysis's AA-AgentPerf (October 15, 2026) found Rubin at **1.14x to 1.45x** GB300's peak throughput per megawatt depending on the model, with larger leads only at high per-user speeds. Treat any large vendor or analyst multiple as one point on a curve until someone else measures the curve. MLPerf Inference v6.1 (September 2026) added agentic and end-to-end RAG tests for the same reason.

---

## Hardware-Enabled Optimizations (FP8 and FP4)

**FP8 (8-bit Floating Point)** is native on Hopper and later, and remains the safe default for weights and especially for the KV cache.

- **Benefit**: Up to 2x peak tensor throughput versus FP16/BF16, with typically small (<1%) accuracy loss when calibrated.
- **How it works**: FP8 E4M3 spends bits on exponent range rather than mantissa precision, so it covers the dynamic range of LLM activations far better than INT8 and needs simpler calibration.

**FP4 (NVFP4 on NVIDIA Blackwell and Rubin, MXFP4 on AMD and Blackwell)** is the 2026 headline serving precision: FP4 weights with an FP8 KV cache. NVIDIA rates Vera Rubin NVL72 at 3,600 PFLOPS NVFP4 per rack, and MLPerf Inference v6.1 saw submissions move from FP8 to FP4.

**Principal-level Nuance**: Precision is now a kernel-and-hardware decision, not a bit-width decision. NVIDIA's own Dynamo recipe for Nemotron 3.5 Lightning measured BF16 80% faster than NVFP4 with Marlin kernels and 12% faster with CuTeDSL kernels. Serving frameworks use per-block and per-tensor dynamic scaling to keep outliers from degrading the model, but whether FP4 is faster depends on the engine version and kernel backend you actually run. See the [Quantization Deep Dive](../03-training-and-adaptation/07-quantization-deep-dive.md).

---

## Interview Questions

### Q: Why is LLM generation slower than classification?

**Strong answer:**
Classification is a "Prefill-only" task; it processes the entire input and produces a single output in one parallel pass, making it compute-optimal. LLM generation, however, is **auto-regressive**. Each token depends on the previous one, forcing a sequential "Decode" loop. Because each step in this loop is memory-bound (reading gigabytes of weights and KV cache to emit one token per sequence), the system spends most of its time waiting for memory transfers rather than doing math.

### Q: How do you optimize TTFT vs. TPOT?

**Strong answer:**
To optimize **TTFT**, you must optimize the Prefill phase: use FlashAttention-3, increase compute parallelism (Tensor Parallelism), or use Prefix Caching to skip the prefill entirely for common prompts. 
To optimize **TPOT**, you must optimize the Memory Bandwidth during Decode: use quantization (4-bit weights) to reduce the data moved from VRAM, use Grouped Query Attention (GQA) to reduce KV cache size, or use Speculative Decoding to generate multiple tokens per memory load.

### Q: A vendor pitches an accelerator at "3,000 tokens per second." How do you evaluate it?

**Strong answer:**
I would ask what that number is a point on. A per-user top speed at batch 1 says little about cost, because cost comes from aggregate throughput at the interactivity my product needs. So I would ask for, or measure, the full throughput-versus-per-user-speed curve on my model, at my context lengths, with my prefix-reuse pattern (agentic traffic reuses long prefixes heavily), and read off goodput at my TTFT and ITL SLOs. Then I would convert to cost per million tokens using real rental or ownership pricing, not list TCO. The spread can be enormous: one 2026 analyst benchmark showed the same Rubin-versus-GB300 comparison ranging from 67x to 1.6x depending on the operating point and whether you own or rent, and the first independent run (Artificial Analysis's AA-AgentPerf) measured only 1.14x to 1.45x peak throughput per megawatt. I would also check memory limits at long context (SRAM-heavy designs can cap batch size sharply), whether speculative decoding was on, and whether the result came from an independent harness such as MLPerf or InferenceX rather than the vendor's own setup.

---

## References
- Pope et al. "Efficiently Scaling Transformer Inference" (2022)
- NVIDIA. "Transformer Engine Documentation" (2024)
- vLLM Blog. "Understanding LLM Inference Latency" (2023)
- SemiAnalysis. ["Vera Rubin NVL72: agentic inference"](https://newsletter.semianalysis.com/p/vera-rubin-nvl72-agentic-inference) (2026)
- Artificial Analysis. ["Vera Rubin NVL72 on AA-AgentPerf"](https://artificialanalysis.ai/articles/vera-rubin-nvl72-agentperf) (2026)
- MLCommons. [MLPerf Inference v6.1 results](https://mlcommons.org/2026/09/mlperf-inference-v6-1-results/) (2026)

---

*Next: [KV Cache and Context Caching](02-kv-cache-and-context-caching.md)*
