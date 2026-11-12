# Attention Mechanisms

Attention is the core innovation that enables transformers. This chapter covers the mathematical foundations, variants, and optimizations that are essential for system design and interviews.

## Table of Contents

- [Attention Fundamentals](#attention-fundamentals)
- [Scaled Dot-Product Attention](#scaled-dot-product-attention)
- [Multi-Head Attention](#multi-head-attention)
- [Attention Patterns](#attention-patterns)
- [Efficient Attention Variants](#efficient-attention-variants)
- [Flash Attention (v2 to v4)](#flash-attention)
- [Multi-head Latent Attention (MLA)](#multi-head-latent-attention-mla)
- [KV Cache Optimizations & Context Caching](#kv-cache-optimizations--context-caching)
- [Practical Implications](#practical-implications)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Attention Fundamentals

### The Core Idea

Attention allows each position in a sequence to gather information from all other positions. Unlike recurrence (which passes information step by step), attention creates direct connections.

**Mental model for distributed systems engineers:**
- RNN: Message passing along a chain
- Attention: Pub/sub where every node can query every other node

### Query, Key, Value Framework

Attention uses three projections of the input:

| Component | Role | Analogy |
|-----------|------|---------|
| Query (Q) | What am I looking for? | Search query |
| Key (K) | What do I contain? | Document index |
| Value (V) | What do I contribute? | Document content |

```python
# Input: x of shape [batch, seq_len, d_model]

Q = x @ W_q  # [batch, seq_len, d_k]
K = x @ W_k  # [batch, seq_len, d_k]
V = x @ W_v  # [batch, seq_len, d_v]
```

---

## Scaled Dot-Product Attention

The fundamental attention operation:

```python
def scaled_dot_product_attention(Q, K, V, mask=None):
    d_k = Q.shape[-1]
    
    # Compute attention scores
    scores = Q @ K.transpose(-2, -1)  # [batch, seq_len, seq_len]
    scores = scores / math.sqrt(d_k)  # Scale
    
    # Apply mask (for causal attention)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))
    
    # Convert to probabilities
    attention_weights = F.softmax(scores, dim=-1)
    
    # Weighted sum of values
    output = attention_weights @ V
    
    return output, attention_weights
```

### Why Scale by Square Root of d_k?

**Interview favorite**: This question tests numerical intuition.

Without scaling, dot products grow with dimension:
- For q and k of dimension d whose components are independent, zero-mean and unit-variance
- E[q . k] = 0, but Var[q . k] = d
- Standard deviation = sqrt(d)

When d is large (512 or more), dot products can be very large or very small. Softmax on large values approaches one-hot, causing vanishing gradients.

```python
# Demonstration
import numpy as np

d = 512
q = np.random.randn(d)
k = np.random.randn(d)

unscaled = np.dot(q, k)      # Magnitude ~ sqrt(512) ~ 22
scaled = unscaled / np.sqrt(d)  # Magnitude ~ 1
```

### Causal Masking

For autoregressive generation, each position can only attend to previous positions:

```python
def create_causal_mask(seq_len):
    # Lower triangular matrix
    mask = torch.tril(torch.ones(seq_len, seq_len))
    return mask

# Example for seq_len=4:
# [[1, 0, 0, 0],
#  [1, 1, 0, 0],
#  [1, 1, 1, 0],
#  [1, 1, 1, 1]]
```

Positions with mask=0 get score of negative infinity, becoming 0 after softmax.

---

## Multi-Head Attention

Instead of one attention function, use multiple "heads" that attend to different aspects:

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
    
    def forward(self, x, mask=None):
        batch_size, seq_len, d_model = x.shape
        
        # Project to Q, K, V
        Q = self.W_q(x)  # [batch, seq_len, d_model]
        K = self.W_k(x)
        V = self.W_v(x)
        
        # Reshape to multiple heads
        Q = Q.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = K.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        # Now: [batch, num_heads, seq_len, d_k]
        
        # Attention per head
        attn_output, _ = scaled_dot_product_attention(Q, K, V, mask)
        
        # Concatenate heads
        attn_output = attn_output.transpose(1, 2).contiguous()
        attn_output = attn_output.view(batch_size, seq_len, d_model)
        
        # Final projection
        output = self.W_o(attn_output)
        return output
```

**Why multiple heads?**
1. Different heads learn different patterns (syntax, semantics, coreference)
2. Provides representational diversity (ensemble effect)
3. Enables parallel computation across heads

### Head Count Patterns

| Model | d_model | Heads | d_k per head |
|-------|---------|-------|--------------|
| BERT-base | 768 | 12 | 64 |
| GPT-2 | 768 | 12 | 64 |
| GPT-3 175B | 12288 | 96 | 128 |
| Llama 2 70B | 8192 | 64 | 128 |

The d_k of 64 or 128 is remarkably consistent across model sizes.

---

## Attention Patterns

### What Attention Learns

Different heads specialize in different patterns:

| Pattern Type | What It Captures | Example |
|--------------|------------------|---------|
| Positional | Adjacent tokens | Next/previous word |
| Syntactic | Grammatical relations | Subject-verb |
| Semantic | Meaning relations | Coreference |
| Delimiter | Punctuation, structure | Section boundaries |
| Rare | Infrequent patterns | Rare word copying |

### Visualizing Attention

Attention weights can be visualized as heatmaps showing which positions attend to which:

```
Query positions (rows) vs Key positions (columns)

"The cat sat on the mat"

         The  cat  sat  on   the  mat
The     [□    ○    ○    ○    ○    ○ ]
cat     [●    □    ○    ○    ○    ○ ]
sat     [○    ●    □    ○    ○    ○ ]
on      [○    ○    ●    □    ○    ○ ]
the     [○    ○    ○    ○    □    ○ ]
mat     [○    ●    ○    ●    ●    □ ]

● = high attention, ○ = low attention
```

"mat" attends strongly to "cat" (semantic), "on" (syntactic), and "the" (determiner).

---

## Efficient Attention Variants

Standard attention is O(n^2) in sequence length. Many variants reduce this:

### Sparse Attention

Attend only to a subset of positions rather than all:

| Variant | Pattern | Complexity | Example |
|---------|---------|------------|---------|
| Local | Window around each position | O(n * w) | Longformer |
| Strided | Every k-th position | O(n^2 / k) | Sparse Transformer |
| Global | Special tokens attend everywhere | O(n * g) | Longformer, BigBird |
| Block | Block-diagonal attention | O(n * b) | BigBird |

**Longformer pattern:**
```
Local window + Global tokens

[G] [L] [L] [L] [L] [G] [L] [L] [L] [L]

G: Global tokens (attend to/from all)
L: Local tokens (attend within window)
```

### Linear Attention

Replace softmax with linearizable alternatives:

```python
# Standard attention (quadratic)
attention = softmax(Q @ K.T) @ V

# Linear attention approximation
attention = (Q @ (K.T @ V))  # Associativity trick
```

**Variants:**
- Performer: Random feature approximation
- Linear Transformer: elu(Q) @ (elu(K).T @ V)
- Gated linear attention (Gated DeltaNet, Kimi Delta Attention): adds learned gates and a delta-rule state update, and is what 2026 hybrids actually ship

**Tradeoff:** Faster but quality degrades, especially for tasks requiring precise attention (exact recall of a token far back). That is why production models interleave linear layers with full or sparse attention layers instead of going all-linear.

### Complexity Comparison

| Method | Time | Space | Quality | Notes |
|--------|------|-------|---------|-------|
| Standard | O(n^2) | O(n^2) | Best | Baseline |
| Sparse (Longformer) | O(n) | O(n) | Near best | For long docs |
| Linear (Performer) | O(n) | O(n) | Degraded | Best for very long |
| Flash Attention | O(n^2) | O(n) | Best | Exact attention, IO-aware kernel |
| Hybrid (linear or SWA + some full/sparse layers) | Mostly O(n) | Mostly O(n) | Near best | Common in 2026 open flagships |

### Hybrid and Sparse Attention in 2026 Open Models

The variants above stopped being research. The five flagship open releases below, all from late August to September 2026, ship non-standard attention or KV schemes per their model cards. Not every release does: IBM's Granite 4.2 (August 25) is a plain GQA dense transformer.

| Model (release) | Attention scheme | KV consequence |
|-----------------|------------------|----------------|
| Qwen3.8-Flash-Next (Aug 26) | Gated DeltaNet linear attention in 3 of every 4 layers, Qwen Sparse Attention in the fourth | Only 1 layer in 4 keeps a growing KV cache; linear layers hold fixed-size state |
| GLM-5.3-Flash (Aug 26) | Kimi Delta Attention (linear) in 3 of every 4 layers, DeepSeek-style sparse attention over an MLA latent in the fourth | About 1 layer in 4 keeps a growing (compressed) KV cache; Z.ai says the hybrid sharply cuts long-context serving cost |
| Tencent Hy4 preview (Aug 28) | Gated DeepSeek Sparse Attention with IndexCache, over an MLA latent cache | Cache stores a 512-dim latent plus a 64-dim RoPE key per token per layer; each query reads only its top 2,048 selected keys |
| DeepSeek V4.1-Flash (Sep 10) | Causal encoder-decoder (decoder global KV projected from encoder states), Compressed Sparse Attention 2, FP4 main KV | 890 bytes of global KV per token; DeepSeek says ~4x less than V4-Flash and ~437x less than DeepSeek-V1 |
| MiMo-V2.6-Pro (Sep 21) | 60 sliding-window layers (128-token window) plus 10 global layers | Only 10 of 70 layers grow with context |

All five support a 1M-token context (Qwen3.8-Flash-Next through YaRN from 262K native).

**What this changes for system design:**
- **KV sizing is per-architecture.** `2 x layers x kv_heads x head_dim x bytes` is right for Llama-style GQA (320 KiB per token for a 70B) but does not describe these models: DeepSeek V4.1-Flash reports 890 bytes of global KV per token, roughly 370x less than that Llama figure. The 890 bytes cover only the global cache: the model card also describes sliding-window KV, which is capped at the window length rather than growing with context, so the ratio describes long-context growth, not total cache memory on a short request. Compare per-token KV bytes from configs and model cards, not parameter counts, and check which parts of the cache a published figure includes.
- **Sparse attention reads less; it does not automatically store less.** Indexer-based schemes pick top-k keys per query, which cuts attention compute and bandwidth at long context, but the cache only shrinks when sparse attention is paired with compression (an MLA latent in Hy4, compressed FP4 KV in V4.1-Flash).
- **Prefix caching needs hybrid-aware engines.** Sliding-window and linear layers carry state that is not a plain per-token KV list. SGLang v0.5.20's SWA-aware radix tree raised token hit rate from 43.8% to 60.8% and cut mean TTFT from 1.57 s to 1.07 s on DeepSeek-V4-Flash with a shared system prompt (SGLang-reported). Check that your engine version supports the architecture's cache layout before assuming prefix-cache savings.

---

## Flash Attention

Flash Attention is the state-of-the-art implementation that achieves O(n) memory while computing exact attention.

### The Problem It Solves

Standard attention requires materializing the n x n attention matrix:
- For 8K context: 64M floats = 256 MB per layer per head
- For 100K context: 10B floats = 40 GB per layer per head

This memory requirement limits batch sizes and context lengths.

### How It Works

Flash Attention uses tiling and recomputation to avoid storing the full attention matrix:

```
Standard: Q, K -> Attention Matrix (n x n) -> Output
Flash:    Q, K -> Tiles (block_size x block_size) -> Incremental Output
```

**Key ideas:**
1. Process attention in blocks that fit in SRAM
2. Never materialize full attention matrix in HBM
3. Recompute attention during backward pass (faster than loading from HBM)

### FlashAttention-2 (Work Partitioning)
Optimized for A100/H100 by improving parallelism across heads and sequence length.

### FlashAttention-3 (FP8 & H100 Optimization)
**The standard kernel on Hopper (H100/H200):**
- **Asynchronous Execution**: Overlaps GEMM (matrix mult) and softmax operations using warp specialization and TMA (Tensor Memory Accelerator).
- **FP8 Support**: Close to 1.2 PFLOPS in FP8 versus up to 740 TFLOPS in FP16 on H100, using block quantization and incoherent processing (a Hadamard rotation that spreads outliers) to keep FP8 error low.
- **Speedup**: ~1.5x-2.0x faster than FlashAttention-2 in FP16 (all figures paper-reported).

### FlashAttention-4 (Hopper and Blackwell)
FlashAttention-3 targets Hopper only. **FlashAttention-4** is written in CuTe DSL, targets both Hopper and Blackwell (H100, B200), and ships as its own package (`flash-attn-4`), which was still on 4.0.0 beta releases through September 2026. Serving engines on Blackwell-class parts also ship other attention backends (FlashInfer and vendor kernels), so check which backend your engine actually selects before quoting attention throughput.

---

## Multi-head Latent Attention (MLA)

Introduced in DeepSeek-V2 and carried into V3, **MLA is the modern alternative to GQA** for extreme KV cache pressure.

Instead of just grouping heads, MLA compresses the Key and Value vectors into a **low-dimensional latent space** before storing them in the cache.

```
Query (Up-projected) ────────┐
                             ▼
Key, Value (Down-projected) ─▶ [Low-dim Latent Cache] ─▶ [Output]
                             ▲
                             └─ Projection Matrices
```

| Metric | MHA | GQA | MLA |
|--------|-----|-----|-----|
| KV Cache Size | 100% | 12.5% | **A few percent** (DeepSeek-V2 caches 576 values per token per layer, about 1.8% of MHA at its 128 heads; DeepSeek reports a 93.3% smaller cache than its dense DeepSeek 67B) |
| Quality | Baseline | Near-Baseline | **Matched or beat MHA in DeepSeek's ablations** |
| Latency | Baseline | Faster | **Fastest at long context (reduced I/O)** |

**Why MLA wins**: The cache holds one compressed latent per token plus a small positional key. At decode time the up-projection matrices are absorbed into the query and output projections, so the latent is never expanded back into full K and V. RoPE would block that absorption, which is why MLA uses **decoupled RoPE**: a separate small set of rotary dimensions carries position. The result is far less memory bandwidth per decode step at long context.

MLA spread beyond DeepSeek (Kimi K2 uses it, and the sparse-attention layers of Tencent Hy4 and GLM-5.3-Flash sit on a 512-dim KV latent), while DeepSeek's own V4 line moved on to a hybrid of Compressed Sparse Attention and Heavily Compressed Attention; see [Hybrid and Sparse Attention in 2026 Open Models](#hybrid-and-sparse-attention-in-2026-open-models).

---

## KV Cache Optimizations & Context Caching

### Context Caching (System-level)
API providers (OpenAI, Gemini, Anthropic, DeepSeek) offer **Context Caching** (prompt caching).
- **How it works**: Pre-computes and stores the KV tensors for a long "prefix" (e.g., a 100k token law book) and reuses them when a later request starts with the same bytes.
- **Latency benefit**: Prefill for the cached prefix is skipped, so TTFT savings scale with how much of the prompt is shared.
- **Cost benefit**: Cache reads bill at 0.1x input on most current OpenAI, Anthropic and Google models, 0.05x on Claude Opus 5.5, Claude Sonnet 5.5, GPT-6.1 Sol and GPT-6.1 Terra, and 0.025x on Claude Fable 5.1. Other vendors differ (xAI's Grok 4.7 is 0.25x; DeepSeek V4.1-Flash cache hits are 0.02x). Writes cost extra: 1.25x for OpenAI's 30-minute cache (GPT-5.6 and later) and Anthropic's 5-minute cache, 2x for Anthropic's 1-hour cache, 2.5x for OpenAI's 12-hour cache; Google bills explicit cache storage per hour instead (Gemini 4 Argon offers implicit caching only). See [Pricing and Costs](../02-model-landscape/03-pricing-and-costs.md) for current rates and [KV Cache and Context Caching](../04-inference-optimization/02-kv-cache-and-context-caching.md) for design patterns.

### Sliding Window Attention (SWA)
Limits a layer's attention to a fixed window (Mistral 7B: 4,096 tokens) so that layer's KV cache stops growing with context. Modern models interleave SWA layers with a few global layers so long-range recall survives: Gemma 3 uses five local layers per global layer, and MiMo-V2.6-Pro uses 60 sliding-window layers (128-token window) and 10 global ones.

### Multi-Query Attention (MQA)

Share a single K and V across all query heads:

```python
# Standard MHA
Q: [batch, num_heads, seq, d_k]  # 32 heads
K: [batch, num_heads, seq, d_k]  # 32 separate K
V: [batch, num_heads, seq, d_k]  # 32 separate V

# MQA
Q: [batch, num_heads, seq, d_k]  # 32 heads
K: [batch, 1, seq, d_k]          # 1 shared K
V: [batch, 1, seq, d_k]          # 1 shared V
```

**Effect:** 32x reduction in KV cache size, some quality loss.

### Grouped-Query Attention (GQA)

Share K and V across groups of query heads:

```python
# GQA with 8 KV heads for 64 query heads (8:1 ratio)
Q: [batch, 64, seq, d_k]  # 64 query heads
K: [batch, 8, seq, d_k]   # 8 KV heads
V: [batch, 8, seq, d_k]   # 8 KV heads

# Each KV head serves 8 query heads
```

**Effect:** 8x KV cache reduction with minimal quality loss.

**Models using GQA:**
- Llama 2 70B and Llama 3.x 70B: 8 KV heads for 64 query heads
- Llama 3.1 405B: 8 KV heads for 128 query heads
- Llama 4 Maverick: 8 KV heads for 40 query heads
- Mistral 7B: 8 KV heads for 32 query heads
- Gemma: Various configurations

### Comparison

| Attention | KV Cache | Quality | Models |
|-----------|----------|---------|--------|
| MHA | Full | Best | GPT-3 |
| GQA | 1/8 typical | Near best | Llama 2 70B, Llama 3, Mistral |
| MQA | 1/n_heads | Reduced | PaLM, Falcon |
| MLA | Compressed latent, a few percent of MHA | Matched MHA in DeepSeek's ablations | DeepSeek-V2 and V3 family, Kimi K2, Tencent Hy4 |

---

## Practical Implications

### For System Design

1. **Batch size vs context tradeoff:**
   - Total GPU memory = Model + KV cache * batch_size
   - Longer contexts mean smaller batches
   - GQA models can serve more concurrent requests

2. **Latency budget allocation:**
   - Attention is O(n^2) compute; Flash Attention makes its memory O(n), not its compute
   - Prefill (processing prompt) scales with prompt length
   - Decode (generating) scales with generated + prompt length

3. **Memory bandwidth bottleneck:**
   - Generation is often memory-bound
   - Each decode step re-reads the weights (dominant at small batch) and the KV cache (dominant at long context and large batch)
   - Larger batches amortize the weight reads; the KV reads grow with every request added

### Prefill vs Decode

| Phase | Compute Pattern | Bottleneck |
|-------|-----------------|------------|
| Prefill | Process all input tokens | Compute (GPU cores) |
| Decode | Generate one token at a time | Memory (bandwidth) |

This is why TTFT (time to first token) and TPS (tokens per second) are measured separately.

### Context Length Scaling

| Context | Attention Compute | KV Cache, Llama 3.x 70B (GQA, BF16) | Same shape with full MHA |
|---------|-------------------|-------------------------------------|--------------------------|
| 4K | Baseline | 1.3 GB | 10.7 GB |
| 8K | 4x | 2.7 GB | 21.5 GB |
| 32K | 64x | 10.7 GB | 86 GB |
| 128K | 1024x | 43 GB | 344 GB |

Long context requires:
- Flash Attention (memory efficient)
- GQA, MQA or MLA (smaller KV cache), or a hybrid attention architecture
- FP8 KV cache (halves the BF16 numbers above) and offload to host DRAM
- Potentially model parallelism

---

## Interview Questions

### Q: Explain the attention mechanism and why it scales quadratically.

**Strong answer:**
Attention computes pairwise interactions between all positions. For n positions:

1. Q @ K^T produces an n x n matrix of scores
2. Each attention score is a dot product of a query and key
3. Total: n^2 dot products

This is quadratic in sequence length. For 8K tokens, that is 64 million pairwise scores per layer per head. For 128K tokens, it is 16 billion.

The quadratic scaling limits context length. Solutions include:
- Flash Attention: O(n^2) compute but O(n) memory
- Sparse attention: O(n) by attending to subsets
- Linear attention: O(n) approximations
- Hybrids: linear or sliding-window layers for most of the depth, plus a few full or sparse attention layers for precise recall (the pattern most 2026 open flagships ship)

### Q: What is the KV cache and why is it critical for serving?

**Strong answer:**
During autoregressive generation, we produce one token at a time. Without caching, each new token would require recomputing K and V for all previous positions.

The KV cache stores K and V tensors from previous positions. On each new token:
1. Compute Q, K, V only for the new position
2. Append new K, V to cache
3. Attend to full cached K, V

This reduces per-token complexity from O(n) to O(1) for projection computation.

The cost is memory: KV cache scales linearly with sequence length. For Llama 70B at 8K context it is about 2.7 GB per request with GQA (8 KV heads), and would be about 21 GB with full MHA. At 128K it is about 43 GB even with GQA. This directly limits batch size and throughput.

GQA and MQA reduce this by sharing K, V across query heads; MLA compresses K, V into a latent; FP8 KV halves the bytes.

### Q: Compare MHA, GQA, and MQA.

**Strong answer:**
| Variant | K,V heads | KV Cache | Quality | Use Case |
|---------|-----------|----------|---------|----------|
| MHA | Equal to Q heads | Full | Best | Training, quality-critical |
| GQA | Fewer than Q heads | Reduced | Near MHA | Production serving |
| MQA | 1 | Minimal | Reduced | Memory-constrained |

MHA: Each query head has its own K and V. Best quality but largest KV cache.

GQA: Groups of query heads share K and V. Llama 2 70B uses 8 KV heads for 64 query heads (8:1 ratio). 8x smaller cache with minimal quality loss.

MQA: All query heads share one K and V. Maximum memory savings but measurable quality reduction. Used by PaLM.

For serving, GQA is the best tradeoff. It enables larger batch sizes (higher throughput) with quality nearly identical to MHA.

### Q: How does Flash Attention achieve O(n) memory?

**Strong answer:**
Standard attention materializes the full n x n attention matrix in GPU memory. Flash Attention avoids this by:

1. **Tiling:** Process blocks of Q and K that fit in on-chip SRAM
2. **Online softmax:** Compute softmax incrementally without storing all scores
3. **Recomputation:** During backward pass, recompute attention rather than loading saved values

The key insight is that on-chip SRAM is an order of magnitude faster than HBM but tiny. The FlashAttention paper's A100 numbers: 192 KB of SRAM per SM (about 20 MB across 108 SMs) at roughly 19 TB/s, versus 40-80 GB of HBM at 1.5-2.0 TB/s. By doing more arithmetic in SRAM and fewer HBM reads/writes, Flash Attention is both faster and uses less memory.

The result is exact attention (not an approximation) with O(n) memory and 2-4x speedup over standard attention in the original paper. FlashAttention-3 targets Hopper; FlashAttention-4 adds Blackwell.

### Q: How would you size the KV cache for a new 2026 open-weight model?

**Strong answer:**
Not with the textbook formula alone. `2 x layers x kv_heads x head_dim x bytes` assumes every layer stores full K and V for every token, which holds for Llama-style GQA (320 KiB per token for a 70B in BF16) but not for current open flagships:

- **Hybrid layers:** MiMo-V2.6-Pro has 60 sliding-window layers with a 128-token window and 10 global layers, so only the global layers grow with context. Qwen3.8-Flash-Next keeps fixed-size linear-attention state in three of every four layers.
- **Compressed or shared KV:** DeepSeek V4.1-Flash projects the decoder's global KV from encoder states and stores FP4 main KV, for 890 bytes of global KV per token (DeepSeek-reported), more than 300x less than the Llama 70B figure. That excludes its sliding-window KV, a fixed per-request amount bounded by the window, so the gap is widest at long context.
- **Precision:** FP8 KV halves BF16; FP4 halves it again, with quality checks.

My process: read `config.json` (layer types, window sizes, KV heads or latent dims, KV dtype), take the vendor's KV-per-token number if published, then measure actual cache usage in the target engine at my real context distribution, since engine support for hybrid cache layouts varies by version. Capacity then follows from (HBM minus weights) divided by per-request KV at the P95 context length, plus whatever host-memory offload tier the engine supports.

---

## References

- Vaswani et al. "Attention Is All You Need" (2017)
- Dao et al. "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness" (2022)
- Dao "FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning" (2023)
- Shah et al. "FlashAttention-3: Fast and Accurate Attention with Asynchrony and Low-precision" (2024)
- DeepSeek-AI "DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model" (MLA, 2024)
- Beltagy et al. "Longformer: The Long-Document Transformer" (2020)
- Ainslie et al. "GQA: Training Generalized Multi-Query Transformer Models" (2023)
- Shazeer "Fast Transformer Decoding: One Write-Head is All You Need" (MQA, 2019)
- [Flash Attention Repository](https://github.com/Dao-AILab/flash-attention)

---

*Previous: [Tokenization Deep Dive](02-tokenization-deep-dive.md) | Next: [Transformer Architecture](04-transformer-architecture.md)*
