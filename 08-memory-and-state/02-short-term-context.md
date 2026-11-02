# Short-Term Context Management

Short-term context (L1 Memory) is the high-speed interface where reasoning happens. Managing it well is no longer about "message lists" but about **KV Cache Optimization**, **Dynamic Context Allocation**, and keeping the conversation in a shape the provider can cache and the model can keep reasoning over.

## Table of Contents

- [The Context Lifecycle](#the-context-lifecycle)
- [KV Cache Tiling and PagedAttention](#kv-cache-tiling)
- [Prefix Caching (System Prompt Preservation)](#prefix-caching)
- [Sliding Windows vs. Summarization](#sliding-windows-vs-summarization)
- [Contextual Compression (Selective Dropping)](#contextual-compression)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Context Lifecycle

Context goes through three stages:
1. **Intake**: User query + recent history + system instructions.
2. **Processing**: The GPU computes the KV cache for the new tokens.
3. **Eviction**: Removing old tokens to make room for new ones once the limit is reached.

On hosted APIs you do not control eviction directly; you control what you send. On the newest Claude models that has a constraint: earlier turns must not be edited, or the thinking blocks after them are invalidated. Eviction there happens through provider compaction or context editing, not by rewriting the message list (see [Context Engineering](../05-prompting-and-context/05-context-engineering.md#append-only-context-thinking-is-bound-to-the-conversation)).

---

## KV Cache Tiling

Modern inference engines (vLLM, SGLang, TensorRT-LLM) use **PagedAttention** or an equivalent block-based allocator.
- **Idea**: Instead of allocating a contiguous block of GPU memory for the context, memory is broken into **Blocks** (pages).
- **Efficiency**: The vLLM paper measured that earlier serving systems wasted 60-80% of KV cache memory to fragmentation and over-reservation; paging cut the waste to under 4%, allowing significantly larger batch sizes and longer context windows on the same hardware.
- **Tiers**: the KV cache now spills from GPU HBM to host DRAM and then to local disk, object storage, or peer nodes (vLLM added a disk tier in v0.28.0; SGLang's HiCache spans GPU, CPU, and storage), so a long session's prefix can survive eviction from the GPU. See [PagedAttention](../04-inference-optimization/05-paged-attention.md) and [KV Cache and Context Caching](../04-inference-optimization/02-kv-cache-and-context-caching.md).

---

## Prefix Caching

This is the **Holy Grail of Latency** for any production LLM stack.
- **The Problem**: Every time an agent calls an LLM, it sends the same 2,000-token System Prompt + 50 Tool Schemas. This wastes compute.
- **The Solution**: **Persistent Prefix Caching**. The server keeps the KV cache for the "Static" part of the prompt (the prefix) in memory.
- **Result**: You only pay full price for (and wait for) the compute on the *new* part of the message. Hosted cache reads now cost about 0.02x to 0.25x of the input price depending on the model (0.1x on most).
- **Multi-tenant caution (self-hosted)**: a shared prefix cache is a timing side channel, because a cache hit is measurably faster. vLLM's control is a per-request `cache_salt`, so cache entries are only shared within a tenant; set it wherever prompts contain private data. The salt has to survive every code path: vLLM v0.30.0 fixed GHSA-935w-9g4m-p28p, where tool-continuation turns on one Responses API path dropped `cache_salt` and reopened the cross-tenant oracle. The current security floor is v0.32.0 (October 18, 2026), which also fixes CVE-2026-115912 and CVE-2026-115915, so run that or later.

---

## Sliding Windows vs. Summarization

| Method | Mechanics | Pro | Con |
|--------|-----------|-----|-----|
| **Sliding Window** | Keep last N tokens exactly. | High fidelity for recent. | "Dory" effect (forgets start). |
| **Summarization** | Compress old turns into text. | Preserves "Key Facts". | Loses nuance/formatting. |
| **Hybrid** | Keep last 10 turns + 1 summary. | Best of both worlds. | Slightly higher complexity. |
| **Provider compaction** | The API summarizes old turns and returns a compaction block you send back (Claude API, beta; on-demand mode added September 14, 2026). | Keeps prefix-caching and thinking-block rules intact. | Less control over what is kept; the summary is a trust boundary, so keep the raw transcript. |

---

## Contextual Compression

Current frontier stacks support several forms of **selective dropping**:
- **Clear stale tool results**: provider context editing (for example, Anthropic's `clear_tool_uses` strategy) removes old tool outputs and thinking blocks server-side, instead of you deleting them from the message list.
- **Offload large outputs**: persist a 20K-token search result to a file and keep a one-line reference in context.
- **Token Pruning**: Using a smaller model (LLMLingua-2 style) to compress a long retrieved document or user message before it reaches the "Reasoning" model. Apply it only to volatile, uncached content.

---

## Interview Questions

### Q: What is the difference between "Model Context Window" and "Application Context Window"?

**Strong answer:**
The **Model Context Window** is the hard limit defined by the architecture (e.g., 128K for GPT-4o, 1.05M for GPT-6.1 Sol). The **Application Context Window** is a configuration set by the engineer (e.g., 64K limit) to manage **Latency and Cost**. In production, we rarely use the full model window for every turn because attention overhead increases with context size, leading to slower generation, and quality degrades well before the limit (context rot). We use a **Buffer Zone** to leave space for the model's response, which on reasoning models must include the thinking tokens, since they count against `max_tokens`.

### Q: How does "Prefix Caching" change how you design System Prompts?

**Strong answer:**
It forces me to move **Static content to the front** and **Dynamic content to the back**. Early-LLM patterns often put the user's name or date at the very top. That breaks the prefix cache. I put "Immutable Rules" and "Tool Schemas" at the beginning and the "User Context" (which changes every turn) at the end. This ensures the first 5,000 tokens are identical across all users, maximizing cache hits on the inference server. When tools or instructions must change mid-session, I append the change (a mid-conversation system message, or an inline tool addition on Claude) rather than editing the prefix, and I track cache hit rate as an SLO now that both OpenAI and Anthropic expose cache diagnostics.

---

## References
- Kwon et al. "Efficient Memory Management for Large Language Model Serving with PagedAttention" (SOSP 2023)
- Zheng et al. "SGLang: Efficient Execution of Structured Language Model Programs" (RadixAttention, 2024)
- NVIDIA. "Optimizing Inference with TensorRT-LLM" (2025)
- Anthropic. "Prompt Caching: Scale while reducing costs" (2024/2025)
- [Claude API release notes (compaction, context editing)](https://platform.claude.com/docs/en/release-notes/overview)

---

*Next: [Long-Term Memory](03-long-term-memory.md)*
