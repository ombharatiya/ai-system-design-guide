# KV Cache and Context Caching

The KV Cache is the most significant memory consumer in long-context AI systems. Managing this cache effectively is the difference between a system that scales to 1M+ tokens and one that crashes at 10k.

## Table of Contents

- [The KV Cache Problem](#the-kv-cache-problem)
- [GQA: Grouped Query Attention](#gqa-grouped-query-attention)
- [Context Caching (Self-hosted)](#context-caching-self-hosted)
- [API-level Context Caching (Prompt Caching)](#api-level-context-caching-prompt-caching)
- [KV Compression and Compact Architectures](#kv-compression-and-compact-architectures)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The KV Cache Problem

During generation, the model needs the Key (K) and Value (V) tensors for all previous tokens. Storing these in memory is expensive.

**VRAM Calculation (Llama 3.3 70B, standard GQA):**
- **Tokens**: 128,000
- **Precision**: BF16 (2 bytes per value)
- **Memory**: `2 (KV) * layers (80) * context (128k) * kv_heads (8) * head_dim (128) * 2 bytes`
- **Per token**: 320 KiB
- **Total**: **~42 GB per user** in 128k context.

This formula only holds for standard multi-head or grouped-query attention. Many 2026 open flagships use latent, sparse, sliding-window, or linear attention, where KV per token differs by two orders of magnitude or more (see [KV Compression and Compact Architectures](#kv-compression-and-compact-architectures)). Size each model with its own architecture.

---

## GQA: Grouped Query Attention

GQA is the standard way to reduce KV Cache size without losing performance in conventional transformers.

| Method | Ratio | KV Cache Reduction | Quality Loss |
|--------|-------|-------------------|--------------|
| **Multi-Head (MHA)** | 1:1 | 1x (Baseline) | 0% |
| **Grouped Query (GQA)** | 8:1 | **8x** | < 0.2% |
| **Multi-Query (MQA)** | All:1 | 64x-128x | 2-3% |
| **Multi-head Latent (MLA)** | Compressed latent per token | DeepSeek-V2 reported 93.3% less KV than its 67B MHA predecessor | Small (DeepSeek-reported) |

**Nuance**: GQA allows the model to attend to the same KV "memory" from multiple "reasoning" heads, drastically reducing the memory bandwidth needed during the Decode phase.

---

## Context Caching (Self-hosted)

Production systems use **Shared KV Caches** for prompts with common prefixes (e.g., a 100-page knowledge base shared by 1,000 users).

### Tiered KV Storage
- **GPU HBM**: Instant access, strictly limited size.
- **Host DRAM**: Larger, a PCIe or NVLink-C2C hop away.
- **Disk, shared storage, or peer nodes**: Slower, nearly unlimited.

Both major open engines now implement this hierarchy. **SGLang HiCache** is three-tier (GPU, CPU host memory, storage through Mooncake, 3FS, or NIXL backends). **vLLM's tiered KV offloading** is host-centric: all KV flows through pinned host DRAM, GPU memory is freed as soon as the copy completes, and secondary tiers (filesystem, object store, remote peers) are fed from host; v0.28.0 added a disk tier. A canonical host layout means nodes with different tensor-parallel sizes can share cached chunks. NVIDIA Dynamo deprecated its separate KV block manager (KVBM) in favor of this engine-native offload, so do not start new designs on it.

Two design consequences:
- **Hybrid attention needs a smarter prefix cache.** SGLang v0.5.20 caches sliding-window state at the prefix branch point; on DeepSeek-V4-Flash with a shared system prompt, token hit rate rose from 43.8% to 60.8% and mean TTFT fell from 1.57 s to 1.07 s.
- **Host DRAM is not free capacity.** Memory makers expect demand to exceed supply through 2028 (Micron, per The Register), and some models now park parameters there too: Qwen3.8-Flash-Next adds a 51B-parameter N-gram embedding designed to be offloaded to host memory with asynchronous prefetching, which competes with your KV tier for the same DRAM.

**Multi-tenant caveat: the prefix cache is a side channel.** If tenants share a cache, a tenant can detect whether another tenant sent a given prefix from cache-hit timing. Engines mitigate this with a per-tenant `cache_salt`, but it must hold on every code path: vLLM fixed a bug (GHSA-935w-9g4m-p28p, fixed in v0.30.0) where tool-continuation turns on one API path dropped the salt and reopened the cross-tenant oracle. Salt by tenant, test it per endpoint, and stay on a patched engine.

---

## API-level Context Caching (Prompt Caching)

Major providers (OpenAI, Anthropic, Google, DeepSeek) offer **Prompt Caching** discounts, and the multipliers now differ by model, not just by vendor.

| Provider | Feature Name | Pricing (per 1M tokens) | Notes |
|----------|--------------|------------------------|----------|
| **Anthropic** | Prompt caching (explicit breakpoints) | Reads 0.1x on most models (Haiku 5.5: $0.01), 0.05x on Opus 5.5 ($0.20) and Sonnet 5.5 ($0.10, since October 7), 0.025x on Fable 5.1 ($0.25). Writes 1.25x (5-minute TTL) or 2x (1-hour) | Opus 5.5 caches prompts from 512 tokens; a beta lets you add tools mid-conversation without invalidating the cache. On Haiku 5.5, cache reads and writes count toward the 100K-token prompt length above which the whole request bills at 5x |
| **OpenAI** | Prompt caching (automatic) | GPT-5.6 and later: writes 1.25x (30-minute TTL) or 2.5x (12-hour TTL, since October 20); reads 0.1x (GPT-6 Sol $0.20), 0.05x on GPT-6.1 Sol ($0.10) and GPT-6.1 Terra ($0.05). Older models: no write fee | Minimum 1,024 tokens; choose the TTL by reuse interval |
| **Google** | Context Caching (implicit and explicit) | Gemini 3.8 Flash: reads $0.075 (0.1x, promo price through December 31, 2026; $0.15 from January 1, 2027) plus storage $0.50 per 1M tokens per hour ($1.00 from January 1, 2027); Gemini 3.1 Pro Preview reads $0.20 under 200K | Storage is billed per hour, so long-lived explicit caches need enough traffic to cover it. Gemini 4 Argon supports implicit caching only |
| **DeepSeek** | Context Caching (automatic) | V4.1-Flash cache hit $0.006 peak / $0.003 off-peak; V4-Pro $0.044 / $0.022 | Peak is 01:00-04:00 and 06:00-10:00 UTC on weekdays, excluding Chinese public holidays |

**Break-even nuance**: With a 1.25x write and a 0.1x read (Anthropic, and OpenAI on GPT-5.6+), caching pays for itself on the **first cache hit**, the second request within the TTL: 1.25 + 0.1 < 2. Anthropic's 1-hour write at 2x needs a second hit (2 + 0.1 > 2, but 2 + 0.2 < 3), and so does OpenAI's 12-hour write at 2.5x (2.5 + 0.2 < 3). Caching never lowers the prompt length a long-context threshold sees: a Haiku 5.5 request with a 120K-token cached prefix still bills at the 5x rate. The real failure mode is not break-even math but **hit rate**: TTL expiry between agent turns, prefixes that change (timestamps, reordered tools, per-user IDs near the top), and prompts below the minimum cacheable length. Both OpenAI (Responses API, September 8, 2026) and Anthropic (September 23) now expose cache diagnostics, so treat hit rate as an SLO and alert on it.

**Price spread**: DeepSeek V4.1-Flash's peak cache-hit price ($0.006) is about 33x below GPT-6 Sol's cached input ($0.20) but less than 2x below GPT-6 Luna's and Claude Haiku 5.5's ($0.01). DeepSeek has repriced several times in 2026, most recently with peak and off-peak billing from August 16 and V4.1-Flash on September 10, so date-stamp any cost model built on these numbers.

---

## KV Compression and Compact Architectures

Beyond dtype (FP8 or 4-bit KV, covered in the [Quantization Deep Dive](../03-training-and-adaptation/07-quantization-deep-dive.md#kv-cache-quantization-the-vram-saver)), there are two families of ways to shrink KV.

**1. Architectural (trained into the model, lossless relative to that model).** The 2026 open flagships all ship a non-standard attention or KV scheme:

| Model | Scheme | KV implication |
|-------|--------|----------------|
| Llama 3.3 70B | GQA, BF16 | 320 KiB per token (baseline) |
| DeepSeek V4.1-Flash | Causal encoder-decoder; decoder global KV projected from encoder states; Compressed Sparse Attention 2; FP4 main KV | About 890 bytes of global KV per token, roughly 4x below V4-Flash (DeepSeek-reported) |
| MiMo-V2.6-Pro | 60 sliding-window layers (128-token window) plus 10 global layers | Only the 10 global layers grow with context |
| Qwen3.8-Flash-Next | Gated DeltaNet linear attention in 3 of every 4 layers, Qwen Sparse Attention in the fourth | Linear layers keep a fixed-size state; only every fourth layer keeps a growing KV cache |
| GLM-5.3-Flash | Hybrid of linear (KDA) and DeepSeek-style sparse attention | Linear layers keep fixed state; sparse layers cut attention compute more than KV size |
| Tencent Hy4 preview | Gated DeepSeek-style sparse attention with an index cache | Needs engine kernels for the sparse path, or you pay dense cost |

**2. Runtime compression (applied at serving time, lossy).** Keep a few "attention sink" tokens plus a recent window (StreamingLLM), or evict tokens with low accumulated attention (H2O, SnapKV). These help long chat sessions but can silently drop the one fact a retrieval-heavy prompt needed, so run needle-style and task evals before enabling them.

The practical consequence: "how many concurrent 1M-token sessions fit on a node" now depends more on the architecture and the engine's support for it (sliding-window-aware prefix caching, sparse-attention kernels, host-resident sparse KV tiers such as vLLM's HiSparse) than on GPU count.

---

## Interview Questions

### Q: How does PagedAttention help with KV Cache management? (Simplified)

**Strong answer:**
Standard KV caches require contiguous memory allocation (one giant block of RAM). This leads to **External Fragmentation** (memory exists but is in unusable gaps). PagedAttention (used in vLLM) breaks the KV cache into small, fixed-size "pages" (like OS virtual memory). This allows the cache to be non-contiguous, meaning we can allocate memory exactly when needed and share pages between different requests that have the same prefix. The vLLM paper measured earlier systems wasting 60-80% of KV memory to fragmentation and over-reservation, against under 4% waste with paging, which is where the 2-4x throughput gain came from.

### Q: Why is Context Caching better than RAG for a 50k token document?

**Strong answer:**
With cheap context caching (DeepSeek, Gemini, Anthropic), RAG is often "overkill" for medium-sized documents.
1. **Recall**: Context caching gives 100% retrieval recall (the whole doc is in the window), whereas RAG depends on retrieval accuracy. The model can still miss details in a long window, so test with needle-style evals at your document length.
2. **Coherence**: The model can see cross-references across the whole document.
3. **Economics**: At 50k tokens, the cost of a cached input is often lower than the complexity of maintaining a vector database and retrieval pipeline.

### Q: Your capacity plan used `2 × layers × kv_heads × head_dim × bytes` per token for every model in the fleet. What breaks?

**Strong answer:**
That formula is right only for standard MHA or GQA. The 2026 open flagships mostly are not: DeepSeek uses latent or compressed sparse attention with an FP4 main KV (about 890 bytes of global KV per token on V4.1-Flash, versus 320 KiB for a BF16 Llama-3.3-70B), MiMo uses mostly sliding-window layers, and Qwen's newest design uses linear attention in three of four layers, which keeps a fixed-size state. So per-token KV varies by more than 300x across models, and the formula overestimates memory for some by orders of magnitude while ignoring fixed state and sliding-window costs for others. I would compute KV per model from its config, confirm the engine actually implements the compact scheme (otherwise you pay full-size KV), include prefix-cache behavior (sliding-window models need branch-point-aware caching to get good hit rates), and validate with a load test at the target context length. Prefill and decode can even have different active-parameter counts now, so I would size those pools separately too.

---

## References
- Kwon et al. "Efficient Memory Management with PagedAttention" (2023)
- DeepSeek-AI. "DeepSeek-V2" (MLA) arXiv:2405.04434 (2024)
- Xiao et al. "Efficient Streaming Language Models with Attention Sinks" (StreamingLLM, 2023)
- Anthropic. "Prompt Caching Documentation"
- OpenAI. [Prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching)
- vLLM Blog. ["Tiered KV offloading"](https://vllm.ai/blog/2026-09-10-tiered-kv-offloading) (2026)
- SGLang. [HiCache design](https://docs.sglang.io/advanced_features/hicache_design.html)

---

*Next: [Speculative Decoding](03-speculative-decoding.md)*
