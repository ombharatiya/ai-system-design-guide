# Speculative Decoding

Speculative decoding is a now-standard technique that allows large language models (LLMs) to generate multiple tokens per forward pass, effectively breaking the memory-bandwidth bottleneck for sequential decoding. In 2026 it is increasingly something the model publisher ships with the checkpoint rather than an optional serving trick.

## Table of Contents

- [The Core Concept](#the-core-concept)
- [Draft-Verify Paradigm](#draft-verify-paradigm)
- [Drafters in 2026: Heads, MTP, Diffusion, DSpark](#drafters-in-2026-heads-mtp-diffusion-dspark)
- [Lookahead, N-gram, and Suffix Decoding](#lookahead-n-gram-and-suffix-decoding)
- [Load-Aware Speculation (Adaptive Verification)](#load-aware-speculation-adaptive-verification)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Core Concept

LLM decoding is memory-bound: loading 140GB of weights (70B model) to produce a single 2-byte token is inefficient. 
**Speculative Decoding** uses a cheaper method to "guess" the next $N$ tokens and uses the large model to verify them all in a single parallel "Prefill-style" pass.

---

## Draft-Verify Paradigm

1. **Drafting**: A cheap drafter (a small model, extra heads on the target, or a bundled module) proposes $K$ candidate tokens.
2. **Verification**: The large "Target Model" processes all $K$ tokens at once.
3. **Acceptance**: The target model's logits are used to accept or reject candidates. If token $i$ is rejected, all tokens after it are discarded.

Illustrative numbers for the classic separate-draft setup:

| Model | Size | Speed | Latency per token |
|-------|------|-------|-------------------|
| **Draft** | 1B | Fast | 5ms |
| **Target**| 70B| Slow | 50ms |
| **Speculative**| - | **Fast**| **15ms - 25ms** |

**Net Result**: 2x to 3x speedup in wall-clock time with **zero loss in quality** at low concurrency. The guarantee is exact: with standard speculative sampling the output distribution is provably the target's, and with greedy decoding the output is identical token for token. What varies is speed, which depends on the **acceptance length** (accepted tokens per verification step).

The guarantee holds only for standard verification. Engines now ship opt-in lossy policies (vLLM v0.33.0 added FLy in November 2026, reporting 1.04x to 1.31x more decode throughput on MI355X and B300) that trade exact output for speed, so treat any non-default verification policy as a model change and rerun evals.

---

## Drafters in 2026: Heads, MTP, Diffusion, DSpark

The industry has moved away from separately trained draft models, which add VRAM, a second KV cache, and tokenizer-matching pain, toward drafters tied to the target. Medusa heads started that move. vLLM stopped documenting them first, then removed Medusa and the MLP speculator outright in v0.33.0 (November 2, 2026), and the methods below replaced them. A config that still sets `"method":"medusa"` fails on upgrade, so move it to EAGLE-3 or the model's own MTP module.

| Method | What drafts | Notes |
|--------|-------------|-------|
| **EAGLE-3** | A light head fed by the target's hidden features from several layers | Paper reports up to 6.5x speedup and 1.38x throughput at batch 64 in SGLang; the common default for open models without native MTP |
| **Native MTP** | Multi-token-prediction heads trained with the model (DeepSeek-V3 onward) | Shipped built in by MiMo-V2.6 and Qwen3.8-Flash-Next (a 4B MTP module) |
| **DFlash / DFlash 2** | A small block-diffusion model that proposes a whole block in parallel | DFlash 2 (Inco AI, August 18, 2026) reports 2.7x to 3.4x throughput on Qwen3.8-27B (vendor-reported); merged into SGLang, vLLM, and llama.cpp between August 19 and 27 |
| **DSpark** | DeepSeek's speculative module, shipped inside the checkpoint | DeepSeek ships the module inside its checkpoints: V4-Flash-DSpark and V4-Pro-DSpark (June 27, MIT) first, then V4-Flash-0731, V4-Pro-0813, and V4.1-Flash (September 10, including the October 17 build), all MIT; other DSpark drafts exist for Kimi K3 (RadixArk) and MiniMax-M3 (NVIDIA), and publishers ship their own for Nemotron 3.5 (NVIDIA) and LFM2.5 (Liquid) |
| **Separate draft model** | A smaller model from the same family | Still useful when nothing above exists; PARD (parallel draft) variants cut drafting latency |

Two consequences for system design:
- **Self-host estimates should assume the publisher's drafter is on.** Enabling DSpark on vLLM is one flag: `--speculative-config '{"method":"dspark","num_speculative_tokens":7}'`. RadixArk's 2.25B Kimi-K3-DSpark draft reports acceptance lengths around 5.4 to 5.5 on GSM8K and HumanEval, so a large share of decode steps emit several tokens.
- **Eval parity must include the speculative path.** Speculative decoding should be output-preserving, but kernels, sampling settings, and engine bugs are not. Run your evals with speculation on, the way production runs. Industry benchmarks have caught up: MLPerf Inference v6.1 allows speculative decoding in the gpt-oss-120b Interactive scenario.

---

## Lookahead, N-gram, and Suffix Decoding

Model-free alternatives find candidate continuations in text the system has already seen: lookahead decoding uses the model's own recent n-grams; n-gram (prompt lookup) and suffix decoding match against the prompt and prior outputs.
- **Best For**: Structured data, code edits, RAG answers that quote the context, and highly repetitive technical writing. They cost almost nothing when they miss, which makes them a good default for agent loops that rewrite files.

---

## Load-Aware Speculation (Adaptive Verification)

The classic objection: speculation spends spare compute, and at high batch there is none, so a static draft length $K$ hurts throughput. The 2026 answer is per-request, load-aware verification budgets.

**vLLM adaptive verification** (introduced August 14, 2026 for DSpark) scores every (request, position) draft slot by its survival probability, the running product of the drafter's per-position confidences, and verifies only the global top-B slots, where B comes from a step-cost model profiled at startup. The data shows why a fixed $K$ wastes compute: on DeepSeek-V4-Pro-0813, the first token of a 7-token draft survives over 70% of the time and the last under 10%. With adaptive verification, vLLM reports speculation staying beneficial up to concurrency 256 while keeping long-draft gains at low concurrency (vendor-reported).

Constraints to know:
- At launch it was DSpark-only, required full CUDA graphs, and did not support LoRA or pipeline parallelism.
- v0.29.0 added per-request acceptance stats in responses (`--per-request-spec-decode-metrics`), which you should log per tenant and per prompt type.
- v0.30.0 extended it under Model Runner V2 to every draft-model speculator (MTP, EAGLE-3, DFlash) through an online acceptance estimator, provided the target's attention backend supports variable-length verification.

Hardware is following the same idea: NVIDIA lists running a Groq 3 LPX rack as an external speculative drafter next to Vera Rubin GPUs as one of its supported serving modes.

---

## Interview Questions

### Q: Why doesn't Speculative Decoding work well for high-temperature creative writing?

**Strong answer:**
Speculative decoding relies on the "Draft Model" being able to accurately predict what the "Target Model" would say. In high-temperature creative writing, the probability distribution is "flatter," and the model is encouraged to pick less-likely tokens. This leads to a very low **Acceptance Rate** (the draft model's guesses are frequently rejected). A rejection never costs correctness, because the verification pass still yields one corrected token, but at low acceptance you are doing roughly sequential decoding plus the drafter's latency and the wasted verification compute, so you end up slower than with speculation off.

### Q: How do draft heads (Medusa, EAGLE-3, native MTP) differ from a separate draft model?

**Strong answer:**
A separate draft model takes up extra VRAM, needs its own KV cache, must share the tokenizer, and drifts out of sync when the target is updated. Draft heads attach to the target instead: Medusa adds heads that each predict a fixed offset from the final hidden state, EAGLE-3 runs a light head on hidden features from several target layers (higher acceptance than Medusa), and native MTP heads are trained with the model itself, so they match its distribution best. All of them reuse the target's forward pass and KV, which removes the second-model overhead. In 2026 the practical difference is ownership: publishers such as DeepSeek, Xiaomi, and Qwen ship the drafter in the checkpoint, so the default should be "use the bundled MTP or DSpark module, fall back to EAGLE-3 or a diffusion drafter, and only train your own if neither exists."

### Q: "Speculative decoding hurts throughput at large batch, so turn it off in production." Do you agree?

**Strong answer:**
It was true for a static draft length: at high concurrency the GPU is already compute-saturated, so verifying seven speculative tokens per request mostly burns compute on tokens that get rejected, since acceptance falls off sharply with position. The current answer is to make the verification budget load-aware rather than to turn speculation off. vLLM's adaptive verification scores each draft position by its survival probability and verifies only the globally best slots within a step budget, so at low load requests get long drafts and at high load only high-confidence early positions are checked; vLLM reports a net benefit up to concurrency 256. I would enable it where the engine supports it, log per-request acceptance, and check feature interactions (at launch it did not support LoRA or pipeline parallelism). For workloads with very low acceptance, such as high-temperature generation, I would still disable it per route.

---

## References
- Leviathan et al. "Fast Inference from Transformers via Speculative Decoding" (2023); Chen et al. "Accelerating Large Language Model Decoding with Speculative Sampling" (2023)
- Cai et al. "Medusa: Simple LLM Acceleration via Multiple Decoding Heads" (2024)
- Li et al. "EAGLE-3: Scaling up Inference Acceleration of Large Language Models via Training-Time Test" arXiv:2503.01840 (2025)
- DeepSeek-AI. "DeepSeek-V3 Technical Report" (multi-token prediction) arXiv:2412.19437
- "DFlash" (block-diffusion drafter) arXiv:2602.06036; DeepSeek, "DSpark" arXiv:2607.05147
- Fu et al. "Lookahead Decoding" (2024)
- vLLM Blog. ["DSpark and adaptive verification"](https://vllm.ai/blog/2026-08-14-dspark-adaptive-verification) (2026)

---

*Next: [Batching Strategies](04-batching-strategies.md)*
