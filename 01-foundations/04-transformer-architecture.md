# Transformer Architecture

This chapter provides a comprehensive view of the complete transformer architecture, bringing together the components from previous chapters into a unified understanding.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Input Processing](#input-processing)
- [The Transformer Block](#the-transformer-block)
- [Output Processing](#output-processing)
- [Modern Architecture Variations (Hybrid MoE, MLA)](#mixture-of-experts-moe--hybrid-architectures)
- [Untied vs. Tied Embeddings](#untied-vs-tied-embeddings)
- [Scaling Properties](#scaling-properties)
- [Architecture Comparison Table](#architecture-comparison-table)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Architecture Overview

A decoder-only transformer (the architecture used by GPT, Claude, Llama) consists of:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Token Embeddings                            │
│              + Position Embeddings (or RoPE)                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│    ┌─────────────────────────────────────────────────────┐      │
│    │                  Transformer Block                   │      │
│    │  ┌─────────────────────────────────────────────┐    │      │
│    │  │              RMSNorm/LayerNorm              │    │      │
│    │  └───────────────────┬─────────────────────────┘    │      │
│    │                      ▼                              │      │
│    │  ┌─────────────────────────────────────────────┐    │      │
│    │  │         Masked Multi-Head Attention         │    │      │
│    │  │            (with KV Cache)                  │    │      │
│    │  └───────────────────┬─────────────────────────┘    │      │
│    │                      │                              │      │
│    │                  + Residual                         │      │
│    │                      │                              │      │
│    │  ┌─────────────────────────────────────────────┐    │      │
│    │  │              RMSNorm/LayerNorm              │    │      │
│    │  └───────────────────┬─────────────────────────┘    │      │
│    │                      ▼                              │      │
│    │  ┌─────────────────────────────────────────────┐    │      │
│    │  │             Feed-Forward Network            │    │      │
│    │  │               (SwiGLU/GELU)                 │    │      │
│    │  └───────────────────┬─────────────────────────┘    │      │
│    │                      │                              │      │
│    │                  + Residual                         │      │
│    └──────────────────────┴──────────────────────────────┘      │
│                           │                                     │
│                    Repeat × N layers                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Output RMSNorm                             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Language Model Head                           │
│              (Linear: hidden_dim → vocab_size)                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
                         Logits
```

---

## Input Processing

### Token Embedding

Convert token IDs to dense vectors:

```python
class TokenEmbedding(nn.Module):
    def __init__(self, vocab_size, d_model):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, d_model)
    
    def forward(self, token_ids):
        return self.embedding(token_ids)
```

**Dimensions:**
- Input: [batch_size, seq_len] token IDs
- Output: [batch_size, seq_len, d_model] embeddings

### Position Information

Position is incorporated via one of:

**1. Rotary Position Embedding (RoPE):**
Applied within attention, not added to embeddings:
```python
def apply_rope(q, k, positions):
    # Rotate q and k vectors based on position
    freqs = compute_frequencies(positions)
    q_rotated = rotate_embeddings(q, freqs)
    k_rotated = rotate_embeddings(k, freqs)
    return q_rotated, k_rotated
```

**2. Learned Position Embeddings:**
Added directly to token embeddings:
```python
position_embeddings = nn.Embedding(max_seq_len, d_model)
x = token_embeddings + position_embeddings(positions)
```

**Modern models (Llama, Mistral, GPT-4) use RoPE** for better length generalization.

---

## The Transformer Block

### Pre-Norm Structure

Modern transformers use pre-normalization:

```python
class TransformerBlock(nn.Module):
    def __init__(self, config):
        super().__init__()
        self.attn_norm = RMSNorm(config.d_model)
        self.attn = GroupedQueryAttention(
            d_model=config.d_model,
            n_heads=config.n_heads,
            n_kv_heads=config.n_kv_heads
        )
        self.ff_norm = RMSNorm(config.d_model)
        self.ff = SwiGLUFFN(
            d_model=config.d_model,
            d_ff=config.d_ff
        )
    
    def forward(self, x, mask=None, kv_cache=None):
        # Attention with residual
        h = x + self.attn(self.attn_norm(x), mask, kv_cache)
        
        # FFN with residual
        out = h + self.ff(self.ff_norm(h))
        
        return out
```

### Attention Component

```python
class GroupedQueryAttention(nn.Module):
    def __init__(self, d_model, n_heads, n_kv_heads):
        super().__init__()
        self.n_heads = n_heads
        self.n_kv_heads = n_kv_heads
        self.head_dim = d_model // n_heads
        
        self.q_proj = nn.Linear(d_model, n_heads * self.head_dim)
        self.k_proj = nn.Linear(d_model, n_kv_heads * self.head_dim)
        self.v_proj = nn.Linear(d_model, n_kv_heads * self.head_dim)
        self.o_proj = nn.Linear(n_heads * self.head_dim, d_model)
    
    def forward(self, x, mask, kv_cache):
        B, T, D = x.shape
        
        # Project
        q = self.q_proj(x).view(B, T, self.n_heads, self.head_dim)
        k = self.k_proj(x).view(B, T, self.n_kv_heads, self.head_dim)
        v = self.v_proj(x).view(B, T, self.n_kv_heads, self.head_dim)
        
        # Apply RoPE
        q, k = apply_rope(q, k, positions)
        
        # Update KV cache
        if kv_cache is not None:
            k = torch.cat([kv_cache.k, k], dim=1)
            v = torch.cat([kv_cache.v, v], dim=1)
            kv_cache.update(k, v)
        
        # Repeat KV heads for GQA
        k = k.repeat_interleave(self.n_heads // self.n_kv_heads, dim=2)
        v = v.repeat_interleave(self.n_heads // self.n_kv_heads, dim=2)
        
        # Attention (using Flash Attention in practice)
        attn_out = flash_attention(q, k, v, mask)
        
        # Output projection
        out = self.o_proj(attn_out.view(B, T, -1))
        return out
```

### Feed-Forward Network

```python
class SwiGLUFFN(nn.Module):
    def __init__(self, d_model, d_ff):
        super().__init__()
        # SwiGLU has 3 projections instead of 2
        self.gate_proj = nn.Linear(d_model, d_ff, bias=False)
        self.up_proj = nn.Linear(d_model, d_ff, bias=False)
        self.down_proj = nn.Linear(d_ff, d_model, bias=False)
    
    def forward(self, x):
        gate = F.silu(self.gate_proj(x))  # SiLU = Swish
        up = self.up_proj(x)
        return self.down_proj(gate * up)
```

**FFN hidden dimension** is typically 2.7x the model dimension for SwiGLU (vs 4x for standard FFN with GELU).

### RMSNorm

```python
class RMSNorm(nn.Module):
    def __init__(self, d_model, eps=1e-6):
        super().__init__()
        self.weight = nn.Parameter(torch.ones(d_model))
        self.eps = eps
    
    def forward(self, x):
        rms = torch.sqrt(torch.mean(x ** 2, dim=-1, keepdim=True) + self.eps)
        return self.weight * (x / rms)
```

Simpler and faster than LayerNorm since it skips mean centering.

---

## Output Processing

### Final Normalization

Apply RMSNorm after the last transformer block:

```python
hidden_states = self.output_norm(hidden_states)
```

### Language Model Head

Project to vocabulary size:

```python
class LMHead(nn.Module):
    def __init__(self, d_model, vocab_size):
        super().__init__()
        self.linear = nn.Linear(d_model, vocab_size, bias=False)
    
    def forward(self, x):
        return self.linear(x)  # Returns logits
```

## Untied vs. Tied Embeddings

**Standard Pattern (GPT-3, Llama 2):** Weight Tying
- Output head shares weights with input embeddings.
- **Pro**: Saves memory (vocab_size * hidden_dim).
- **Con**: Forces input and output latent spaces to be identical, which can be suboptimal.

**2025 Frontier Pattern (Llama 3/4, GPT-5.2):** Untied Embeddings
- Output head has its own weights.
- **Why?**: Larger vocabularies (128k+) make the embedding table a significant portion of the model. Untying allows the output head to specialize in "predictive logic" while input embeddings focus on "semantic understanding."
- **System Impact**: Increases parameter count but often improves perplexity for multilingual and code tasks.

### Getting Predictions

```python
# During generation
logits = lm_head(hidden_states[:, -1, :])  # Last position only
next_token = sample(logits)

# During training
logits = lm_head(hidden_states)  # All positions
loss = cross_entropy(logits, targets)
```

---

## Modern Architecture Variations

### Llama 2/3 Architecture

| Component | Implementation |
|-----------|----------------|
| Attention | Grouped Query Attention (GQA) |
| Position | Rotary Position Embedding (RoPE) |
| Normalization | RMSNorm (pre-norm) |
| Activation | SwiGLU |
| Bias | No bias in linear layers |

### Mistral Architecture

Same as Llama but adds:
- **Sliding Window Attention:** Each layer only attends to 4K tokens
- Still achieves effective 32K+ context via stacking

### Mixture of Experts (MoE) & Hybrid Architectures

State-of-the-art models often use **Hybrid MoE/Dense** blocks:
- **Periodic Dense Layers**: Every few MoE layers, a dense layer is added to ensure "global" knowledge is shared across all experts.
- **Expert Parallelism**: Distributing different experts across different GPUs. This makes **inter-node bandwidth** (NVLink/InfiniBand) a primary architecture bottleneck.

### Multi-head Latent Attention (MLA) Integration
The standard attention block in [DeepSeek-V3 / V4](file:///Users/om/play/ai-system-design-guide/01-foundations/03-attention-mechanisms.md#multi-head-latent-attention-mla) and equivalent modern architectures replaces the standard Q/K/V projections with low-rank latent compressions.
- **Architectural Shift**: The "KV Cache" is now a compressed latent representation, changing the memory/compute ratio of the entire transformer block.

### Comparison of Choices

| Choice | Old Approach | Modern Approach | Benefit |
|--------|--------------|-----------------|---------|
| Norm | Post-LN | Pre-LN / RMSNorm | Training stability, speed |
| Position | Sinusoidal/Learned | RoPE | Better extrapolation |
| Activation | GELU | SwiGLU | Quality (+1% on benchmarks) |
| Attention | MHA | GQA | 8x smaller KV cache |
| Bias | With bias | No bias | Fewer parameters, similar quality |

---

## Scaling Properties

### Parameter Counts

| Component | Parameters |
|-----------|------------|
| Token embedding | vocab_size * d_model |
| Per layer Q/K/V | 3 * d_model * d_model (for MHA) |
| Per layer O proj | d_model * d_model |
| Per layer FFN | 3 * d_model * d_ff (for SwiGLU) |
| LM head | d_model * vocab_size (often tied) |

**Approximation for decoder-only:**
```
Total ≈ 12 * n_layers * d_model^2 (for d_ff = 4 * d_model, MHA)
```

### Compute Requirements

**Training:** FLOPs per token ≈ 6 * parameters (forward + backward)

**Inference:** FLOPs per token ≈ 2 * parameters (forward only)

### Scaling Laws

The Chinchilla scaling law suggests optimal allocation:

```
D (data tokens) ≈ 20 * N (parameters)
```

For a 70B model, train on ~1.4T tokens for compute-optimal training.

**But:** Many modern models overtrain relative to Chinchilla for better inference efficiency. Llama was trained on 2T+ tokens.

---

## Architecture Comparison Table

| Model | Params | Layers | d_model | Heads | KV Heads | FFN | Context |
|-------|--------|--------|---------|-------|----------|-----|---------|
| GPT-3 | 175B | 96 | 12288 | 96 | 96 | GELU | 2K |
| Llama 2 70B | 70B | 80 | 8192 | 64 | 8 | SwiGLU | 4K |
| Llama 3 405B| 405B | 126 | 16384 | 128 | 16 | SwiGLU | 128K |
| DeepSeek V3 | 671B | 128 | 7168 | 128 | MLA | MoE | 128K |
| Llama 4 (spec)| 1T+ | 140+ | 18432 | 192 | 24 | MoE/H | 1M+ |

*Mistral uses sliding window attention for effective long context.

---

## Interview Questions

### Q: Walk me through the forward pass of a transformer.

**Strong answer:**
For a decoder-only model generating text:

1. **Tokenization:** Convert input text to token IDs

2. **Embedding:** Look up token embeddings from the embedding table

3. **For each transformer layer:**
   - Apply RMSNorm to input
   - Compute Q, K, V projections
   - Apply RoPE to Q and K for position
   - For generation: append new K, V to KV cache
   - Compute attention (masked, so each position only sees previous)
   - Project attention output and add residual
   - Apply RMSNorm
   - Pass through SwiGLU feed-forward network
   - Add residual

4. **Output norm:** Apply final RMSNorm

5. **LM head:** Project to vocabulary size to get logits

6. **Sample:** Select next token from logits using temperature/top-p

For generation, repeat steps 3-6 for each new token, reusing the KV cache from previous positions.

### Q: What is the difference between pre-norm and post-norm?

**Strong answer:**
The difference is where layer normalization is applied relative to sublayers (attention, FFN):

**Post-norm (original transformer):**
```
x = LayerNorm(x + Sublayer(x))
```
Normalize after adding residual.

**Pre-norm (modern transformers):**
```
x = x + Sublayer(LayerNorm(x))
```
Normalize before the sublayer.

Pre-norm is preferred because:
1. Gradients flow more directly through residual connections
2. Training is more stable, especially for deep models
3. Less sensitive to initialization and learning rate
4. No need for learning rate warmup

The cost is slightly lower final performance in some benchmarks, but the training stability is worth it for large models.

### Q: Explain GQA and why it matters for serving.

**Strong answer:**
Grouped Query Attention (GQA) shares Key and Value heads across groups of Query heads.

Standard Multi-Head Attention: 64 query heads, 64 KV heads (1:1)
GQA: 64 query heads, 8 KV heads (8:1)

Implementation: Each KV head is used by 8 query heads via repetition.

**Why it matters:**
The KV cache stores K and V for all positions during generation. For Llama 70B at 8K context:
- MHA: 2.6 MB/token * 8K = 21 GB per request
- GQA (8:1): ~2.6 GB per request

8x reduction enables:
- Larger batch sizes (more concurrent users)
- Longer contexts
- Lower GPU memory requirements

Quality impact: Minimal. Research shows GQA achieves 99%+ of MHA quality.

### Q: What changed between GPT-2 and Llama 2?

**Strong answer:**
Key architecture improvements:

| Component | GPT-2 | Llama 2 |
|-----------|-------|---------|
| Norm | Post-LayerNorm | Pre-RMSNorm |
| Position | Learned absolute | RoPE (rotary) |
| Activation | GELU | SwiGLU |
| Attention | MHA | GQA (for 70B) |
| Bias | Present | Removed |

Impact:
- RMSNorm: Faster and equally effective
- RoPE: Better length extrapolation
- SwiGLU: ~1% quality improvement
- GQA: 8x smaller KV cache for serving
- No bias: Fewer parameters, no quality loss

These changes enable training larger models more stably and serving them more efficiently.

---

## References

- Vaswani et al. "Attention Is All You Need" (2017)
- Touvron et al. "Llama: Open and Efficient Foundation Language Models" (2023)
- Touvron et al. "Llama 2: Open Foundation and Fine-Tuned Chat Models" (2023)
- Zhang and Sennrich. "Root Mean Square Layer Normalization" (2019)
- Shazeer. "GLU Variants Improve Transformer" (2020)
- Su et al. "RoFormer: Enhanced Transformer with Rotary Position Embedding" (2021)
- Jiang et al. "Mistral 7B" (2023)

---

*Previous: [Attention Mechanisms](03-attention-mechanisms.md) | Next: [Embeddings and Vector Spaces](05-embeddings-and-vector-spaces.md)*
