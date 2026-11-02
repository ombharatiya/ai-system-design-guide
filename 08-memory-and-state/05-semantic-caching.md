# Semantic Caching

Caching has evolved from exact string matching to **Semantic Matching**. Semantic caching reuses completions for "equivalent" queries, cutting latency from seconds to milliseconds on a hit. The savings depend entirely on how repetitive your traffic is: FAQ-style support traffic can see large reductions, while open-ended agent work rarely repeats. It is also distinct from provider prompt caching, which discounts repeated *input* rather than skipping the call.

## Table of Contents

- [Exact Cache vs. Semantic Cache](#exact-cache-vs-semantic-cache)
- [Semantic Cache vs. Provider Prompt Cache](#semantic-cache-vs-provider-prompt-cache)
- [The Semantic Matching Pipeline](#the-semantic-matching-pipeline)
- [RedisVL and GPTCache](#redisvl-and-gptcache)
- [Evaluation: Hit Rate vs. Semantic Drift](#evaluation-hit-rate-vs-semantic-drift)
- [Multimodal Semantic Caching](#multimodal-semantic-caching)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Exact Cache vs. Semantic Cache

| Feature | Exact Cache (Redis/Memcached) | Semantic Cache (RedisVL/Qdrant) |
|---------|-------------------------------|---------------------------------|
| **Key** | Hashed query string | Query embedding vector |
| **Match**| 100% string identity | Cosine Similarity > Threshold |
| **Efficiency**| Low (Minor typos break cache) | High (Understands intent) |
| **Risk** | Zero | Semantic Drift (Returning wrong answer) |

---

## Semantic Cache vs. Provider Prompt Cache

These solve different problems and stack well together:

| | Semantic cache (yours) | Provider prompt cache |
|---|------------------------|-----------------------|
| **What is reused** | The whole response | The KV cache of a repeated input prefix |
| **Saves** | The entire LLM call: input, output, and latency | Input cost and prefill time only |
| **Discount** | 100% on a hit | Cache reads at 0.1x input on most models, 0.05x on Claude Opus 5.5, Claude Sonnet 5.5, GPT-6.1 Sol and GPT-6.1 Terra, 0.025x on Claude Fable 5.1 |
| **Correctness risk** | A near-match can return the wrong answer | None: the model still generates a fresh answer |
| **Best for** | Repetitive questions with stable answers | Long shared prefixes: system prompts, tools, documents |

Deep prompt-cache discounts shrank the case for semantic caching on input-heavy workloads: if most of a request's cost is a long shared prefix, the provider cache already removes 90% or more of it with no drift risk. Semantic caching still wins when **output tokens or latency** dominate, because only a response cache skips generation entirely.

---

## The Semantic Matching Pipeline

1. **Embed**: The incoming query is converted into a vector (e.g., using `text-embedding-3-small`).
2. **Search**: Search the cache for the nearest neighbor, filtered by tenant and permission scope.
3. **Threshold Check**: If `distance < 0.05` (very similar), return the cached result.
4. **LLM Verification**: For high-stakes queries, a small "Verifier Model" (e.g., GPT-6 Luna, Claude Haiku 5.5, Gemini 3.8 Flash) checks if the cached response actually answers the new query.
5. **Update**: If no hit, call the LLM and store the new result in the vector cache.

**The cache key must include scope.** A response generated from user A's documents, permissions, or memory must never be served to user B, however similar the question. Partition the cache by tenant (and by permission set where answers depend on access), and never cache responses that embed personal data unless the partition is per user.

---

## RedisVL and GPTCache

Standard stack:
- **RedisVL**: Provides low-latency vector search directly within a Redis instance.
- **Hybrid Caching**: Using Redis for both metadata (keys) and vector payloads.
- **TTL**: Semantic caches should have a TTL (Time-To-Live). The common pattern is **Dynamic TTL**: popular answers live longer while "stale" information is evicted regularly.
- **Invalidate on change**: version the cache by model ID, system prompt version, and source-data version, so a prompt change or a document update cannot serve answers generated under the old ones.

---

## Evaluation: Hit Rate vs. Semantic Drift

Measure the cache like a classifier, because every hit is a prediction that two queries are equivalent:

| Metric | What it tells you |
|--------|-------------------|
| **Hit rate** | Share of requests served from cache (the savings side) |
| **False-hit rate** | Share of hits where the cached answer is wrong for the new query, measured by sampling hits and grading them with a judge or humans |
| **Latency on hit and on miss** | A miss pays embedding plus vector search on top of the LLM call |
| **Staleness** | Share of hits served after the underlying data changed |

Tune the threshold on a labeled set of query pairs (equivalent and not equivalent) to the false-hit rate the product can tolerate, then let the hit rate fall where it falls. A cache with a 40% hit rate and a 0.1% false-hit rate is a better product than one with 70% hits and 5% wrong answers.

---

## Multimodal Semantic Caching

With native multimodal frontier models (Gemini 3.8 Flash, GPT-6.1 Sol, Claude Opus 5.5), we now cache **Image and Audio queries**.
- **Visual Similarity**: Caching the description of an image if a semantically similar image was processed before.
- **Audio Fingerprinting**: Caching transcripts for similar voice commands.

---

## Interview Questions

### Q: What is "Semantic Drift" in caching, and how do you prevent it?

**Strong answer:**
Semantic Drift occurs when the similarity threshold is too loose (e.g., 0.8 instead of 0.95). A query like *"How do I fix my car?"* might match a cached response for *"How do I wash my car?"*. To prevent this, we use **Multi-Stage Validation**: 1) Vector similarity check, 2) **Entity-Match check** (ensures both queries involve "Car" and the same "Verb"), and 3) **Threshold Tightening**: for technical or medical queries, we require $>0.98$ similarity to return a cached result. We also sample hits continuously and grade them, so the false-hit rate is a tracked metric rather than an assumption.

### Q: Why is a Semantic Cache sometimes *more* expensive than a raw LLM call at low volume?

**Strong answer:**
Because every request, hit or miss, pays the cache's overhead: an embedding call and a vector search. The embedding itself is nearly free (`text-embedding-3-small` is $0.02 per 1M tokens, so a 50-token query costs about a millionth of a dollar), but the search adds latency, often 5 to 20 ms in-process and more across a network, plus the cost of running and maintaining the cache. At a low hit rate, most requests pay that overhead and still call the LLM, so median latency gets worse and the savings are small. Add the false-hit risk and the engineering cost, and semantic caching only becomes a clear win at **High Scale** with repetitive traffic, where the hit rate is high enough to offset the "Embedding Tax."

### Q: You already use provider prompt caching. Should you also add a semantic cache?

**Strong answer:**
It depends on where the cost sits. Prompt caching discounts the repeated input prefix (reads at 0.1x or less of the input price) with no correctness risk, so for agent workloads that are input-dominated it captures most of the savings already. A semantic cache skips the call entirely, so it pays off when output tokens and latency dominate and the questions repeat: support FAQs, product lookups, documentation answers. I would check the traffic first: cluster a week of queries, estimate the hit rate at a threshold tuned for an acceptable false-hit rate, and compare the projected savings against the drift risk and the per-tenant partitioning work. Often the answer is a semantic cache for a narrow, high-volume intent and prompt caching everywhere else.

---

## References
- Redis. "RedisVL: Python Client for Redis Vector Library" (2025)
- Bang, F. "GPTCache: An Open-Source Semantic Cache for LLM Applications Enabling Faster Answers and Cost Savings" (NLP-OSS 2023)
- [OpenAI. Prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching)
- See also [FinOps and Token Economics](../11-infrastructure-and-mlops/04-finops-and-token-economics.md) for provider cache pricing.

---

*Next: [State Management Patterns](06-state-management-patterns.md)*
