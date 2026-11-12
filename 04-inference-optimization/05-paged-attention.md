# PagedAttention

PagedAttention is the foundational algorithm behind high-throughput serving engines (vLLM, SGLang, TensorRT-LLM). It solves the "Memory Fragmentation" problem that previously limited LLM scalability.

## Table of Contents

- [The Contiguous Memory Problem](#the-contiguous-memory-problem)
- [How PagedAttention Works](#how-pagedattention-works-vllm)
- [Managing Virtual Memory (Block Manager)](#managing-virtual-memory-block-manager)
- [KV Cache Sharing (Copy-on-Write)](#kv-cache-sharing-copy-on-write)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Contiguous Memory Problem

Standard deep learning frameworks allocate memory in large, contiguous blocks. 
For an LLM request, you might pre-allocate memory for a `max_sequence_length` of 8192 tokens.

**The Waste:**
1. **Internal Fragmentation**: If the user only generates 10 tokens, 99.9% of that reserved block is wasted.
2. **External Fragmentation**: Memory is broken into gaps too small for a new "large block," even if total free memory is high.

---

## How PagedAttention Works (vLLM)

PagedAttention draws inspiration from Virtual Memory in Operating Systems.

1. **Tokens to Blocks**: The KV cache for a request is broken into small, fixed-size **Blocks** (e.g., 16 tokens per block).
2. **Logical vs. Physical**: The model thinks it's attending to a contiguous sequence (Logical memory), but the blocks are scattered throughout VRAM (Physical memory).
3. **The Lookup Table**: A **Block Table** maps logic indices to physical addresses.

**Primary Benefit**: Memory waste drops from ~60-80% down to **less than 4%**.

---

## Managing Virtual Memory (Block Manager)

Serving frameworks (vLLM, SGLang) act as "mini-OSs" for GPUs.

- **Allocation**: When a new request starts, the Block Manager assigns it a set of empty physical blocks.
- **Preemption**: If VRAM is full, the scheduler preempts lower-priority requests and frees their blocks, rebuilding the KV later by recomputation or by reloading it from a slower tier.
- **Tiered offload**: Blocks no longer have to die when evicted from HBM. vLLM's tiered KV offloading copies blocks to pinned host DRAM and from there to disk, object storage, or peer nodes; SGLang's HiCache does the same across GPU, host memory, and storage backends. The block table is what makes this cheap: a block is a fixed-size unit that can be moved, hashed, and shared. See [KV Cache and Context Caching](02-kv-cache-and-context-caching.md#context-caching-self-hosted).

---

## KV Cache Sharing (Copy-on-Write)

PagedAttention enables effortless sharing of "Common Prefixes."

**The Scenario**: 100 users are chatting with the same 5,000-token system prompt.
- **Traditional**: Store that 5,000-token KV cache 100 times (**500k tokens** in VRAM).
- **PagedAttention**: Store it **once** via the Block Table and have all 100 users point to the same physical blocks.
- **Copy-on-Write**: If a user generates a unique token, a new block is created just for them, while the shared blocks remain unchanged.

**How engines find shared prefixes.** vLLM's automatic prefix caching hashes each full block together with the hash of everything before it, so identical prefixes map to identical block hashes; SGLang's RadixAttention keeps the same information in a radix tree. Two production details:
- **Hashes must be deterministic across nodes** for distributed KV sharing to hit. vLLM v0.29.0 made its prefix-cache root hash deterministic, so you no longer need to pin `PYTHONHASHSEED` across replicas.
- **Sharing must respect tenancy.** A shared block is also a timing side channel; engines mix a per-tenant `cache_salt` into the hash, and every API path has to apply it. vLLM learned this the hard way: tool-continuation turns on the Responses API (`/v1/responses`) dropped the salt until 0.30.0 (GHSA-935w-9g4m-p28p). The current vLLM security floor is set by later advisories, not this one; see [LLM Security](../12-security-and-access/01-llm-security.md).

---

## Interview Questions

### Q: Why does PagedAttention significantly increase throughput?

**Strong answer:**
PagedAttention increases throughput by allowing for much larger **batch sizes**. Because it eliminates internal and external memory fragmentation, we can pack many more requests into the same GPU VRAM. In traditional serving, we might only fit 4 requests because we have to "reserve" max-length blocks; with PagedAttention, we can fit 20-30 requests because we only use memory for the tokens that actually exist. Larger batches lead to better GPU utilization and significantly higher aggregate tokens per second.

### Q: Explain the "Block Table" in the context of vLLM.

**Strong answer:**
The Block Table is a mapping structure that bridges the gap between the model's expectation of contiguous data and the physical reality of scattered memory. Each entry in the table corresponds to a "Logical Block" of tokens. It stores the physical address of the GPU memory where that block's key and value tensors are stored. This allows the framework to dynamically allocate and free memory in small chunks, enabling prefix sharing with copy-on-write, preemption without fragmentation, and offloading blocks to host memory or disk and bringing them back.

---

## References
- Kwon et al. "Efficient Memory Management for Large Language Model Serving with PagedAttention" (SOSP 2023)
- vLLM Documentation. "PagedAttention Logic" (2024)
- Zheng et al. "SGLang: Efficient Execution of Structured Language Model Programs" (RadixAttention, 2024)

---

*Next: [Serving Infrastructure](06-serving-infrastructure.md)*
