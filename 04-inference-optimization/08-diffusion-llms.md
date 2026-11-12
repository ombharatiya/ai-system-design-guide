# Diffusion Language Models

Almost every LLM you use is autoregressive (AR): it emits one token per forward pass, left to right. **Diffusion language models** take a different path borrowed from image generation: start from a fully masked sequence and iteratively denoise *all positions in parallel* over a handful of steps. The payoff is throughput, reported figures over 1,000 tokens per second, an order of magnitude above speed-optimized AR models. The cost is some quality, and the maturity is early. This chapter explains how they work, where they win, and what an engineer should actually do with them in 2026.

## Table of Contents

- [How They Work](#how-they-work)
- [The Speed Advantage and the Tradeoff](#the-speed-advantage-and-the-tradeoff)
- [Diffusion Systems in 2026](#diffusion-systems-in-2026)
- [Hybrids: Draft with Diffusion, Verify with AR](#hybrids-draft-with-diffusion-verify-with-ar)
- [Where Diffusion Wins and Where AR Still Wins](#where-diffusion-wins-and-where-ar-still-wins)
- [Maturity and What to Do Today](#maturity-and-what-to-do-today)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## How They Work

An AR model factorizes a sequence left to right: each token is predicted from the past only, one per forward pass. A **masked diffusion model** instead learns to reconstruct a whole sequence from a corrupted (masked) version, refining all positions at once over a few denoising steps, with **bidirectional** attention (each position sees both past and future).

The canonical reference is **LLaDA** (arXiv:2502.09992). The mechanism, at a teachable level:
- **Forward process:** start from clean text; at noise level `t`, replace each token with a `[MASK]` with probability `t`. At `t=1` the sequence is fully masked.
- **Reverse process:** a Transformer (a "mask predictor") is trained to predict the original token at every masked position simultaneously, optimizing a likelihood *lower bound* (not the exact likelihood AR models use).
- **Sampling:** begin fully masked and denoise from `t=1` to `0`. At each step the model predicts all masked tokens, and an **unmasking schedule** decides how many of the highest-confidence predictions to commit and how many to re-mask for the next step. Fewer steps means faster and lower quality; one token per step approaches AR quality but forfeits the speed.

Parallel decoding and bidirectional context fall out of this directly, and they give diffusion a notable trick: LLaDA reports beating GPT-4o on a reversal task (completing a poem backwards), because the objective is not locked to left-to-right order.

The cleanest mental model is not "AR vs diffusion" but an **interpolation** between them, established by Block Diffusion (BD3-LMs, arXiv:2503.09573): split the sequence into blocks, go AR across blocks and diffusion within a block. This recovers two things pure diffusion struggles with, KV caching and variable-length output, and the block size is the knob (block size 1 is AR, block size equals the whole sequence is pure diffusion). Almost every fast 2026 system ships some block or semi-AR variant.

---

## The Speed Advantage and the Tradeoff

The speedup comes from one place: **tokens committed per forward pass.** AR emits one; diffusion commits many over a few steps, so wall-clock throughput rises when the model can confidently unmask several positions at once. It is a latency and throughput win, not inherently a quality win.

Reported throughput (all vendor or benchmark claims, conditions vary): commercial diffusion models report on the order of 700 to 1,500 tokens per second on a single datacenter GPU, which their makers frame as several times faster than speed-optimized AR baselines (up to ~10x for the best-case code-completion path, closer to ~5x for reasoning workloads). Independent measurement of one commercial model put it around 1,100 tokens per second.

The tradeoff is intrinsic to the unmasking schedule: committing more tokens per step is faster but can produce locally incoherent output, so the practical best quality comes from unmasking fewer tokens per step, which slows things down. Where the quality gap stands in 2026, as reported:
- **On code, the gap is small.** Diffusion models roughly match similar-tier AR models on code-completion benchmarks; code is the consensus sweet spot.
- **On knowledge and hard reasoning, the gap is larger**, and no diffusion model is frontier-class. A recognized theoretical headwind is that masked diffusion optimizes a likelihood *bound*, not exact likelihood.

---

## Diffusion Systems in 2026

Treat versions, speeds, and prices as a perishable snapshot.

| System | Type | Notes (reported) |
|--------|------|------------------|
| **Mercury 2.5** (Inception Labs) | Closed, general | The clearest production option: OpenAI-compatible API on Inception, Baseten, and OpenRouter. Inception reports 1,107 tokens/sec on widely available NVIDIA GPUs, a 260K context, and list pricing of $0.20 input / $0.75 output per 1M tokens (launch promotion $0.04 / $0.15). Marketed as a fast reasoning model, a claim some analysts dispute given diffusion's multi-step-reasoning limits; verify on your own evals. |
| **Mercury Voice** (Inception Labs) | Closed, voice agents | Enterprise GA September 29, 2026: a diffusion reasoning model for voice agents with a reported 320 ms median time to first answer token (p95 750 ms), three reasoning-effort levels, 128K context, $0.40 / $1.50 per 1M list (50% off at launch). |
| **Gemini Diffusion** (Google) | Closed demo | Impressive demo speeds; **no public API** (not on the Gemini API model list as of November 2, 2026). Do not architect around it. |
| **DiffusionGemma** (Google) | Open weights | An open-weight Gemma-family diffusion model, reported over 1,000 tokens/sec on one datacenter GPU. Do not conflate it with Gemini Diffusion. |
| **LLaDA / LLaDA 2.x** | Open weights | The foundational open model and its MoE successors; competitive with similar-size AR models, not frontier. LLaDA2.2-mini (Ant Group, September 5, 2026, Apache 2.0) is a 16B / 1.4B-active MoE with 128K context and edit tokens (DELETE and INSERT) so it can revise text, not just fill masks; vendor scores of 47.68 on BFCL v4 and 57.50 on tau2-Bench show agentic diffusion models exist but trail AR models on tool use. |
| **Dream 7B** | Open weights | Reported to match or exceed similar-size AR models, with a tunable steps knob and strong planning (Sudoku, Countdown); a fully-open code variant exists. |
| **DFlash 2** (Inco AI) | Diffusion drafter | Not a standalone chat model: a block-diffusion drafter for AR targets, reporting 2.7x to 3.4x throughput on Qwen3.8-27B (vendor-reported). Merged into SGLang, vLLM, and llama.cpp in late August 2026. |

---

## Hybrids: Draft with Diffusion, Verify with AR

The most production-relevant work fuses the two paradigms:

- **Diffusion as a speculative drafter, now shipping.** A diffusion model proposes a block of tokens in parallel and an AR model verifies them, preserving the AR target distribution, analogous to [speculative decoding](03-speculative-decoding.md). TiDAR (arXiv:2511.08923) reports roughly 5x the tokens per second of AR while closing the quality gap. The production version arrived in 2026: block-diffusion drafters (DFlash, arXiv:2602.06036; DFlash 2, August 18) were merged into SGLang (August 19), vLLM (August 21), and llama.cpp (August 27), and Ollama's MLX engine supports DFlash. Small open models are also shipping diffusion-decoding adapters, such as the "Uno" adapters for MBZUAI's K2 Horizon 0.9B and 7B.
- **Block / semi-AR adaptation.** Fast-dLLM v2 (arXiv:2509.26328) adapts an existing AR model (Qwen) into a block-diffusion model with a block-wise KV cache and confidence-aware parallel decoding, the practical recipe for getting cacheable, fast diffusion decoding out of checkpoints you already have.
- **RL for diffusion reasoning.** d1 (arXiv:2504.12216) is the first GRPO-style RL for masked diffusion LLMs (diffu-GRPO), pairing masked SFT with an efficient RL stage, and reports consistent gains over the base diffusion model on math and planning benchmarks.

The near-term production path is not pure diffusion but these hybrids, which recover KV caching and quality while keeping much of the speed. The drafter half of that prediction has already happened: if you self-host an AR model on a mainstream engine, you can use diffusion today without changing models.

---

## Where Diffusion Wins and Where AR Still Wins

**Diffusion advantages (structural, not just hype):**
- **Low-latency, high-throughput serving**, the headline use case, where round-trip latency dominates (interactive coding, autocomplete, real-time agents).
- **Code**, where generation is iterative back-and-forth refinement rather than strictly sequential.
- **Infilling and multi-region editing**, native to the masking objective: mask the relevant locations and regenerate all of them at once (for example, add a parameter consistently across a signature, its docstring, call sites, and tests in one shot) instead of orchestrating multiple AR calls.
- **Reversal and order-insensitive reasoning and planning**, where bidirectional context helps.

**Where AR still wins in 2026:** long context and agentic/tool-use workflows (a 2026 reality-check study documents diffusion weaknesses in long-context handling, KV-cache efficiency, and function calling); frontier quality on knowledge and hard reasoning; and very long or variable-length generation, which pure diffusion handles poorly and block/semi-AR variants exist to patch. AR also has years of serving-stack and KV-cache optimization that diffusion's ecosystem has not yet matched.

---

## Maturity and What to Do Today

The honest read: mostly early, with a thin layer of real commercial offerings on a fast-moving open research base. What an engineer should do:

1. **Do not replace your AR frontier model** for quality-critical, long-context, or agentic work.
2. **Do pilot diffusion where latency and throughput dominate and outputs are short-to-medium and structured**, code completion and infill, bulk transforms, structured extraction. A commercial OpenAI-compatible diffusion API is the lowest-friction A/B test; an open-weight model if you need to self-host.
3. **Use the hybrids now.** A diffusion drafter for your existing AR model (DFlash 2 in vLLM, SGLang, or llama.cpp) is the lowest-risk way to get diffusion's speed, because the AR target still decides every token. Measure acceptance length and throughput at your real concurrency.
4. **Benchmark on your tasks with realistic decoding settings.** The quality you get is a direct function of how aggressively you unmask, and vendor tokens-per-second numbers assume aggressive settings.
5. **Treat "reasoning diffusion model" marketing skeptically** and verify on your own reasoning evals.

---

## Interview Questions

### Q: How does a diffusion language model differ from an autoregressive one, and where is that an advantage?

**Strong answer:**
An autoregressive model generates left to right, one token per forward pass, each conditioned only on the past. A diffusion language model starts from a fully masked sequence and iteratively denoises all positions in parallel over a few steps, with bidirectional attention. The practical advantage is throughput: it commits many tokens per forward pass instead of one, so reported speeds run over 1,000 tokens per second, several times faster than speed-optimized AR. Bidirectionality also helps on infilling and editing, where you mask several locations and regenerate them coherently in one shot, and on order-insensitive tasks. The cost is a quality gap on knowledge and hard reasoning, partly because masked diffusion optimizes a likelihood bound rather than exact likelihood, and weaker long-context and tool-use behavior. So I would reach for it on latency-critical, short-to-medium, structured generation like code, not as a frontier-model replacement.

### Q: If diffusion LLMs are faster, why hasn't everyone switched?

**Strong answer:**
Because the speed comes with real tradeoffs and the ecosystem is young. The quality gap is still meaningful on knowledge and hard reasoning, and the speed itself is a function of how aggressively you unmask: the best quality comes from committing fewer tokens per step, which gives back much of the speed. Pure diffusion also handles long context and variable-length output poorly and lacks the mature KV-cache and serving optimizations AR has accumulated. The likely path is not a wholesale switch but hybrids, block or semi-autoregressive models and diffusion-as-a-speculative-drafter, that recover caching and quality while keeping much of the throughput. In 2026 the pragmatic move is to pilot diffusion where latency dominates and outputs are structured, and keep the AR frontier model for everything else.

---

## References

- Nie et al., "Large Language Diffusion Models" (LLaDA) arXiv:2502.09992
- Arriola et al., "Block Diffusion" (BD3-LMs) arXiv:2503.09573
- "TiDAR: Think in Diffusion, Talk in Autoregression" arXiv:2511.08923
- "Fast-dLLM v2" arXiv:2509.26328
- Zhao et al., "d1: Scaling Reasoning in Diffusion LLMs via RL" arXiv:2504.12216
- "Dream 7B" arXiv:2508.15487
- Inception Labs, [Mercury](https://www.inceptionlabs.ai/) (commercial diffusion LLM) and ["Introducing Mercury 2.5"](https://www.inceptionlabs.ai/blog/introducing-mercury-2-5)
- "DFlash" (block-diffusion speculative drafter) arXiv:2602.06036

---

*Next: [On-Device and Edge Deployment](09-on-device-and-edge-deployment.md)*
