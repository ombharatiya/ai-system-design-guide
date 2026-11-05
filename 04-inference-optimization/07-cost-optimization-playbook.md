# Cost Optimization Playbook

AI costs are no longer "magic." They are measurable, predictable, and highly optimizable. Per-token API prices keep falling at the mid and low tiers (in September 2026 GPT-6 Sol at $2/$10 and GPT-6 Luna at $0.10/$0.50 per 1M tokens at least halved the list prices of GPT-5.6 Sol and Luna, and Claude Opus 5.5 cut the Opus tier to $4/$20; in October Claude Haiku 5.5 launched at $0.10/$0.50, a tenth of Haiku 4.5, and GPT-6.1 Terra at $1/$5 against $2/$12 for GPT-5.6 Terra), while GPU rental prices rose through 2026. So the cost lever is mostly *routing*, *caching*, and *service-tier selection*, not just picking a cheaper provider. This chapter covers the strategies to reduce inference costs by 10x without sacrificing quality.

## Table of Contents

- [The Unit Economics of AI](#the-unit-economics-of-ai)
- [Model Cascading (Efficiency Tiers)](#model-cascading-efficiency-tiers)
- [Small Language Models (SLMs)](#small-language-models-slms-for-production)
- [Service Tiers and Price Multipliers](#service-tiers-and-price-multipliers)
- [Spot Instances and GPU Pricing](#spot-instance-strategies)
- [The "Token Tax" Optimization](#the-token-tax-optimization)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Unit Economics of AI

We measure success by **Tokens per Dollar ($)** at a given quality bar and latency SLO.

| Component | Cost Driver | Optimization |
|-----------|-------------|--------------|
| **Compute** | GPU Time ($/hr) | Better utilization (Batching). |
| **VRAM** | KV Cache Size | GQA, Quantization. |
| **Network** | Payload Size | Compression, Local serving. |
| **API** | Per-token pricing | Caching, Model selection, Service tier. |

---

## Model Cascading (Efficiency Tiers)

The most effective cost-saving strategy is to use the **cheapest model capable of the task.**

**The cascade pattern** (list prices per 1M input/output tokens as of November 2, 2026):
1. **Classifier**: A tiny self-hosted model (0.5B) determines query complexity (near $0), or a hosted decision model does: OpenAI's Decisions API returns a typed answer with a probability and bills only input, at $0.10 per 1M uncached tokens ([Research Radar theme 16](../RESEARCH-RADAR.md#16-decision-models-typed-answers-with-probabilities)). Calibrate its thresholds on your own traffic; early papers on decision models report underconfident probabilities.
2. **Tier 1 (Small/cheap)**: 90% of queries (greetings, simple Q&A, extraction) go to GPT-6 Luna ($0.10/$0.50), Claude Haiku 5.5 ($0.10/$0.50 for prompts up to 100K tokens), DeepSeek V4.1-Flash, or a self-hosted small model ($).
3. **Tier 2 (Mid)**: 9% of queries (complex reasoning) go to GPT-6.1 Terra ($1/$5), Claude Sonnet 5.5 ($2/$10), GPT-6 Sol ($2/$10), or Gemini 3.8 Flash ($0.75/$3.75 introductory through December 31, 2026; $1.50/$7.50 from January 1, 2027) ($$$).
4. **Tier 3 (Frontier reasoning)**: 1% of queries (expert-level) go to Claude Opus 5.5 ($4/$20), GPT-6 Astra ($10/$50), or Claude Fable 5.1 ($10/$50) at high reasoning effort ($$$$$).

**Net result**: 80% cost reduction vs. sending all traffic to Tier 2.

**Caching breaks tier monotonicity.** Since October 7, Sonnet 5.5 reads cached input at 0.05x ($0.10 per 1M) and Opus 5.5 at 0.05x ($0.20), so a cached Opus 5.5 token costs a tenth of an uncached Sonnet 5.5 token ($2). For an agent loop that rereads a long cached context, Opus 5.5 can cost less than poorly cached Sonnet traffic. Price cascades on *effective* cost per task, including cache hit rate, not on list input prices.

**Prompt length breaks it too.** Claude Haiku 5.5 bills the whole request at $0.50/$2.50 (5x) once the prompt passes 100,000 tokens, and cache reads and writes count toward that length. A Tier 1 route that accumulates agent history drifts into a 5x price without any routing change, so cap or compact Haiku 5.5 context below the threshold, or route long prompts elsewhere.

---

## Small Language Models (SLMs) for Production

Small open models (IBM Granite 4.2 3B and 8B, Gemma 4 12B, Qwen3.8-27B) and the cheapest API tiers (GPT-6 Luna, Claude Haiku 5.5) now match or beat the original GPT-4 from 2023 on most benchmarks.
- **Use Case**: Entity extraction, sentiment analysis, simple RAG, routing.
- **Cost**: about 100x cheaper per token (GPT-6 Luna and Claude Haiku 5.5 at $0.10 / $0.50) than the $10 / $50 ceiling tier (Claude Fable 5.1, GPT-6 Astra); self-hosted small models only go lower at high utilization. Two caveats on Haiku 5.5: the rate holds only for prompts up to 100K tokens, and its newer tokenizer produces about 30% more tokens than Haiku 4.5 for the same text, so the real saving per unit of text is smaller than the rate card suggests.
- **Latency**: < 100ms response times for short outputs.

### The Price Floor: GPT-6 Luna and DeepSeek V4.1-Flash

DeepSeek V4 Flash (released April 24, 2026) reset the floor at launch at $0.14 / $0.28 per 1M tokens. Pricing has moved since. DeepSeek introduced peak and off-peak billing on August 16, 2026, then released **V4.1-Flash** (`deepseek-flash`, September 10) at $0.30 input / $1.20 output at peak and half that off-peak ($0.15 / $0.60), with cache hits at $0.006 peak and $0.003 off-peak. Peak hours are 01:00-04:00 and 06:00-10:00 UTC on weekdays only, excluding Chinese public holidays, so weekends are entirely off-peak. **V4-Pro** (V4-Pro-0813) stays available at $1.32 / $3.96 peak and $0.66 / $1.98 off-peak; a planned September 14 reroute of V4-Pro traffic to V4.1-Flash was reversed on September 11.

The cheapest closed tiers now undercut it: **GPT-6 Luna** at $0.10 / $0.50 (cached input $0.01) beats V4.1-Flash's uncached input and output prices even off-peak ($0.15 / $0.60), and **Claude Haiku 5.5** (October 7) matches Luna's list and cached prices for prompts up to 100K tokens. DeepSeek's remaining edges are its cache-hit price ($0.006 peak, $0.003 off-peak, versus $0.01) and MIT weights you can self-host; since October 17 the `deepseek-flash` alias has been served by the V4.1-Flash-1017 build, so rerun evals on the alias you call. For high-volume batch work the practical floor is "GPT-6 Luna or Claude Haiku 5.5, or DeepSeek V4.1-Flash scheduled off-peak for cache-heavy work (RAG over a shared knowledge base, codebase agents)," chosen by eval quality on your task. Verify on the [DeepSeek pricing page](https://api-docs.deepseek.com/quick_start/pricing) and [OpenAI pricing page](https://developers.openai.com/api/docs/pricing) before committing.

---

## Service Tiers and Price Multipliers

The same model now spans roughly a 12x price range by tier alone, so a cost model needs a tier policy per workload class (interactive, agent inner loop, offline), not one per-token price.

| Lever | Multiplier on Standard | Where (as of November 2, 2026) |
|-------|------------------------|--------------------------------|
| **Batch / Flex** | 0.5x | OpenAI Batch and Flex, Gemini Batch and Flex, Bedrock Flex, Anthropic Batch; OpenRouter's Batch API (September 22, 2026) is generally 50% off on 70+ models, with a median batch finishing in 7 minutes in beta |
| **Off-peak scheduling** | 0.5x | DeepSeek off-peak hours and weekends |
| **Cache reads** | 0.1x typical; 0.05x on Opus 5.5, Sonnet 5.5, GPT-6.1 Sol and GPT-6.1 Terra; 0.025x on Fable 5.1 | All major APIs. Writes cost 1.25x on Anthropic (5-minute TTL) and OpenAI GPT-5.6+ (30-minute TTL), 2x for Anthropic's 1-hour cache, and 2.5x for OpenAI's 12-hour cache (October 20) |
| **Priority / Fast** | about 1.75x to 2.5x | OpenAI Fast 2x (2.5x on GPT-5.5); Gemini Priority about 1.8x; Bedrock Priority 1.75x; Anthropic fast mode on Opus 5.5 at $8/$40 (2x; out of research preview since October 15, on the Claude API, Claude Models on Google Cloud (Anthropic managed) and Foundry's Azure-hosted deployments; Opus 4.8 fast mode is removed November 30) |
| **Ultrafast** | 6x | OpenAI: GA on GPT-6 Astra since September 29, 2026 (also offered for Astra on Bedrock), and on GPT-6.1 Sol for all API users since October 8 ($12/$60) |
| **Long context** | Whole request repriced | OpenAI above 272K input tokens (GPT-6 Sol $4/$15, GPT-6.1 Terra $2/$7.50); Claude Haiku 5.5 above 100K prompt tokens, cache reads and writes included ($0.50/$2.50, 5x); xAI doubles all tokens at 200K+; other Claude 4.6 and later models are flat to 1M |
| **Data residency** | about 1.1x | Anthropic `inference_geo: "us"`, OpenAI residency endpoints, regional Claude on Bedrock and Google Cloud, Mistral EU |

Two traps. Long-context surcharges apply to the *whole* request once you cross the threshold: trimming a 280K-token GPT-6 Sol prompt to 270K halves its input rate and cuts its output rate by a third, and on Haiku 5.5 a 101K-token prompt costs about $0.05 in input against about $0.01 for a 99K one. And promotional prices should be budgeted at list: Gemini 4 Argon is $2/$10 only through November 30 and $4/$20 from December 1, Gemini 3.8 Flash's introductory rate ends December 31, and OpenAI commits to GPT-5.6 Sol's promo price only "at least through November 21, 2026."

---

## Spot Instance Strategies

For non-real-time workloads (batch processing, data extraction), use **GPU Spot Instances** (AWS Spot, Google Cloud Spot VMs, Azure Spot).

- **Risk**: The GPU can be reclaimed with short notice: two minutes on AWS, about 30 seconds on Google Cloud and Azure.
- **Mitigation**: Make work restartable. Checkpoint batch progress, keep requests idempotent, and requeue in-flight requests on the reclamation signal. Engines with tiered KV offload to shared storage can preserve prefix caches across a reclaim, but do not count on live migration of in-flight decode state.

### GPU Pricing in 2026: Up, and Longer Terms Are Cheaper

GPU rental prices rose through 2026 as memory-bound inference demand grew. Silicon Data's indexes on November 5, 2026 read B200 $6.70, B300 $7.56 (flat), H200 $3.51 and H100 $2.97 per GPU-hour. The B200 index rose about 14% in four weeks (from $5.88 on October 7 and $6.50 on November 1), after standing 27.6% up year to date entering September; read the daily table, because each index page's FAQ shows stale "current" prices. Term structure inverted the old advice: on September 7 the B200 index was $5.73 for 3-month terms and $5.39 for 12-month terms, so locking in longer was cheaper. RunPod's pricing page (updated November 4) lists community/secure H100 SXM $2.69/$3.99, B200 $5.98/$7.99 and B300 $6.94/$8.99, so secure-cloud prices rose 14 to 18% during October. Lambda has posted new on-demand prices that take effect November 16, 2026 (scheduled): 8x HGX H100 $4.59 and 8x HGX B200 $7.69 per GPU-hour, up about 15% from $3.99 and $6.69, which puts an 8x H100 node at about $26,806 a month. Every rise pushes self-host break-even further toward APIs. Read neocloud contract terms carefully; SemiAnalysis's ClusterMAX 3.0 flagged prepayments reaching 100% on some 1-year commitments.

---

## The "Token Tax" Optimization

- **Prompt caching**: Keep prefixes byte-stable (no timestamps or per-user IDs near the top, stable tool order) so repeated context bills at cache-read rates: 0.1x on most models, 0.05x on Opus 5.5, Sonnet 5.5, GPT-6.1 Sol and GPT-6.1 Terra, 0.025x on Fable 5.1. Match the TTL to the reuse interval: OpenAI's 12-hour cache (2.5x write) suits prefixes reused a few times a day that would miss its 30-minute window.
- **Input dominates agent cost**: Anthropic's Claude Code telemetry (September 24, 2026, vendor-reported) shows the input-to-output token ratio rising from 189:1 to 324:1. At that ratio, cache hit rate and the cached-input price drive cost, not the output price. Cursor reported 7% lower user token costs after trimming about two-thirds of its system prompt and loading tools dynamically (vendor-reported).
- **Reasoning effort**: Set it explicitly per route. Defaults differ by model and change between versions: Claude Opus 5.5 defaults to `medium` (Opus 5 was `high`), Sonnet 5.5 and Fable 5.1 to `high`, GPT-6.1 Sol and GPT-6.1 Terra to `medium`. Some newer models cannot turn thinking off at all, so effort is the remaining dial. A measured policy: arXiv 2610.34491 (October 25, 2026; AIME 2026, GPQA Diamond, Terminal-Bench 2.1, SWE-Bench Pro) found top effort cost up to 42x the lowest on tasks every level solved, that higher effort often failed tasks lower effort solved by lengthening trajectories, and that starting at the lowest effort and escalating only after a verified failure matched or beat fixed top effort on every model and benchmark while costing less for most proprietary models. That policy needs a verifier, which is the real engineering work.
- **Output Truncation**: Strictly limit `max_tokens`.
- **Concise-output instructions**: Asking for terse answers (and a fixed output schema) cuts output tokens; measure the saving on your own prompts rather than assuming a fixed percentage, since reasoning tokens often dominate and are not affected.

---

## Interview Questions

### Q: How do you justify the cost of an AI system to a CFO?

**Strong answer:**
I focus on the **ROI of Efficiency.** First, I implement "Model Cascading" to ensure that 90% of our traffic is handled by models priced well under $1 per million tokens. Second, I implement "Semantic Caching" and prompt caching to prevent paying for the same answer, or the same context, twice. Third, I assign a service tier per workload (batch at half price for offline jobs, priority only where the SLO needs it). Fourth, I set up "Inference Quotas" and "Chargeback Models" so each business unit is accountable for their usage. By treating AI as a "Commodity Resource" with tiered pricing, we can transition from "unbounded experimentation" to a "predictable OpEx" model.

### Q: When is a self-hosted individual GPU cluster cheaper than an API?

**Strong answer:**
The "Crossover Point" happens at **constant high throughput**, and it depends heavily on which API tier you are replacing. Worked with November 2026 inputs: two H100s at the Silicon Data index ($2.97 per GPU-hour on November 5) cost about $4,300 a month, enough for a 70B-class open model. At a 3:1 input-to-output blend, Claude Sonnet 5.5 or GPT-6 Sol ($2/$10) costs about $4 per million tokens, so the tie is around 1B tokens a month; against Opus 5.5 ($4/$20) it is about 540M, and against GPT-6.1 Terra ($1/$5, about $2 blended) about 2B. Against GPT-6 Luna or Claude Haiku 5.5 ($0.10/$0.50, about $0.20 blended) it is around 21B tokens a month, roughly 8,000 tokens per second around the clock, which is at or beyond what two GPUs serving a 70B model sustain, so in practice you will not beat the cheapest API tier on price alone. And a self-hosted 70B is not equivalent in quality to a frontier API, so compare against the API tier that meets your quality bar, not the most expensive one. Spiky or business-hours traffic pushes the answer toward APIs because they let you "pay for the silence," and API cache reads at 0.1x or less push break-even further out for agentic workloads. Self-hosting wins when utilization is steady and high, when data cannot leave your boundary, or when you need a fine-tuned open model, and rising GPU rental prices in 2026 raise the bar further.

---

## References
- Google Cloud. "Cost Optimization for Generative AI" (2024)
- Anyscale. "LLM Inference: API vs. Self-Hosted Costs" (2024)
- OpenAI. [API pricing](https://developers.openai.com/api/docs/pricing); Anthropic. [Pricing](https://platform.claude.com/docs/en/about-claude/pricing); DeepSeek. [Models and pricing](https://api-docs.deepseek.com/quick_start/pricing)
- Silicon Data. [B200 rental index](https://www.silicondata.com/products/silicon-index/b200), [B300 index](https://www.silicondata.com/products/silicon-index/b300), [H200 index](https://www.silicondata.com/products/silicon-index/h200) and [H100 index](https://www.silicondata.com/products/silicon-index/h100)
- OpenAI. [Decisions API guide](https://developers.openai.com/api/docs/guides/decisions)
- RunPod. [Pricing](https://www.runpod.io/pricing); Lambda. [Instances](https://lambda.com/instances)
- Sun et al. "What Effort Level Should I Choose?" (arXiv 2610.34491, 2026)

---

*Next: [Diffusion Language Models](08-diffusion-llms.md)*
