# FinOps and Token Economics

This chapter is about the **economics and discipline** of running LLMs at scale: how to model, attribute, budget, and structurally reduce AI spend. It is not a rehash of inference internals; for the tactical levers (quantization, batching, speculative decoding, KV cache) see [Cost Optimization Playbook](../04-inference-optimization/07-cost-optimization-playbook.md) and the [inference chapters](../04-inference-optimization/01-inference-fundamentals.md).

The anchor finding that frames the whole chapter, from Datadog's 2026 State of AI Engineering: **system prompts are about 69% of input tokens, yet only about 28% of calls use prompt caching.** The single largest, lowest-effort cost lever in most production stacks is sitting unused. AI products also run as a cost-of-goods business, not zero-marginal-cost SaaS, so margin thinking is now an engineering concern.

## Table of Contents

- [The Cost Model](#the-cost-model)
- [Caching: The Top Cost Lever](#caching-the-top-cost-lever)
- [Batch and Async Economics](#batch-and-async-economics)
- [The FinOps Discipline](#the-finops-discipline)
- [Structural Cost Decisions](#structural-cost-decisions)
- [Cost Anti-Patterns](#cost-anti-patterns)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Cost Model

Pricing is quoted per million tokens, split input versus output, and **output costs materially more than input**, commonly 3-5x and sometimes more, because generation is autoregressive and compute-bound while input is a parallel prefill pass. (Current per-model prices live in [Pricing and Costs](../02-model-landscape/03-pricing-and-costs.md); they deflate fast, so model the *structure*, not the cents.)

Every request decomposes into stacked spend layers. Modeling each separately is what makes cost predictable and attributable:

| Layer | Driven by | Behavior | Primary lever |
|-------|-----------|----------|---------------|
| System prompt / instructions | Fixed scaffolding, tool defs, few-shot | ~69% of input tokens; paid every call if uncached | Prompt caching |
| Retrieved / context tokens | RAG chunks, injected docs, long context | Scales with k and chunk size; can dwarf everything | RAG vs long context; chunk budgeting |
| Conversation / memory | Chat history, agent scratchpad | Grows unbounded without summarization | Windowing, summarization, compaction |
| Model tier | Frontier vs mid vs small/self-host | 10-100x spread across tiers | Right-sizing, cascades, routing |
| Output length | Verbosity, format, max_tokens | Billed at the higher output rate | `max_tokens` caps, terse output contracts |
| Reasoning / thinking tokens | Extended-thinking modes | Billed at output rate, invisible in the response | Gate thinking by task complexity |
| Retry / overhead | Transient errors, guardrail re-runs | Multiplies on failure | Bounded retries, circuit breakers |
| Agent multi-step | Plan-act-observe loops, sub-agents | Multiplies the whole stack per step | Step ceilings, per-run budgets |

**Reasoning models and agents are cost multipliers, and the two worst because they are invisible.** Extended-thinking tokens are billed at the output rate but do not appear in the response, so a "short" call can cost an order of magnitude more than its visible output suggests; reported analyses put the multiplier anywhere from ~3x to ~15x depending on the task. Thinking is also harder to switch off than it was: Claude Opus 5.5 cannot disable it, and Sonnet 5.5's lowest setting still emits thinking between tool calls, so reasoning effort is the knob to budget. Agents multiply the *entire* token stack on every step, so the right unit of measurement is **cost per task, not cost per call**. Reported bands: a chat turn is cents, while an agentic multi-step task can run from tens of cents to several dollars.

**Agents are input-dominated, so the cached-input price sets the bill.** Anthropic's Claude Code telemetry (September 24, 2026, vendor-reported) shows the input-to-output token ratio rising from 189:1 to 324:1 between March and September, with context per request up 2.6x. At that ratio the output price barely matters; the cache-read rate and the cache-hit rate decide cost per task. The harness is the lever: Anthropic says its harness changes cut cache-miss input by more than half, and Cursor reported 7% lower user token costs by trimming about 66% of its system prompt, loading built-in tools only when needed, and placing explicit cache breakpoints (both vendor-reported).

A teachable cost-per-request formula:

```
cost = uncached_input × in_rate + cached_input × cache_read_rate + cache_write_tokens × write_rate
       + (output_tokens + reasoning_tokens) × out_rate
cost_per_task = cost_per_request × expected_steps × (1 + retry_rate) + runtime_meters
```

where `runtime_meters` covers anything billed by time rather than tokens (hosted agent sessions, sandboxes, realtime voice minutes), and every rate is first multiplied by the service tier, residency and long-context factors below.

### Price Structure Beyond the Per-Token Rate

The list price is the start of the model, not the end. These multipliers now decide real bills:

| Multiplier | Where it applies | Effect | Design response |
|------------|------------------|--------|-----------------|
| Long-context cliff | OpenAI above 272K input (whole request); Claude Haiku 5.5 above a 100K-token prompt, cache reads and writes included (whole request); xAI at 200K+ prompt (all tokens doubled); Gemini 3.1 Pro Preview above 200K | GPT-6 Sol goes from $2/$10 to $4/$15 per 1M, so a 273K prompt pays about twice the input cost of a 272K one; Haiku 5.5 goes from $0.10/$0.50 to $0.50/$2.50, a 5x jump | Stay under the threshold or retrieve instead; other Anthropic models (Claude 4.6 and later) bill flat to 1M |
| Service tier | All major providers | Batch and Flex 0.5x; OpenAI Fast 2x (2.5x on GPT-5.5) and Ultrafast 6x (GPT-6 Astra, GPT-6.1 Sol); Gemini Priority ~1.8x; Bedrock Priority 1.75x; Claude fast mode on Opus 5.5 at $8/$40 vs $4/$20 | A tier policy per workload class (see [Batch and Async Economics](#batch-and-async-economics)) |
| Data residency | Anthropic `inference_geo: "us"` (1.1x on Claude 4.6 and later), OpenAI residency endpoints, Bedrock and Google Cloud regional endpoints, Mistral EU | About +10% on every token | Pin geography only for tenants that require it |
| Introductory pricing | Gemini 4 Argon $2/$10 through November 30, 2026, then $4/$20; Gemini 3.8 Flash $0.75/$3.75 through December 31, 2026, then $1.50/$7.50; GPT-5.6 Sol promo $4/$20 promised only "at least through" November 21 (list $5/$30) | Budgets built on promo rates are understated by up to 2x | Forecast on list price |
| Billed refusals | Anthropic: pre-output refusals in `bio`, `frontier_llm` and `reasoning_extraction` billed since September 24, 2026 | Red-team and eval suites probing those areas cost money | Break out spend by `stop_details.category` |
| Runtime meters | Claude Managed Agents: $0.08 per running session-hour plus tokens, no Batch discount; OpenAI Agents API: hosted sandbox compute, billed only during active turns since October 9; gpt-live-1: $0.05 per session minute plus backend tokens | A second meter beside tokens | Model idle and runtime behavior in cost per task |

---

## Caching: The Top Cost Lever

This is the headline lever precisely because of the anchor stat: the layer that is ~69% of input tokens (the system prompt) is static and ideal for caching, yet only ~28% of calls cache it. The gap between potential and actual is the biggest, cheapest saving available.

**Provider prefix caching** charges a small premium to write a prefix and a steep discount to read it back. By late 2026 the major vendors have converged on reads at about 0.1x the input rate, and the newest models go lower:

| Provider | Cache write | Cache read | TTL and mechanics |
|----------|-------------|------------|-------------------|
| OpenAI (GPT-5.6 and later) | 1.25x input (30-minute TTL) or 2.5x (12-hour TTL, since October 20) | 0.1x (0.05x on GPT-6.1 Sol and GPT-6.1 Terra) | Automatic above 1,024 tokens |
| Anthropic | 1.25x (5-minute) or 2x (1-hour) | 0.1x; 0.05x on Opus 5.5 and Sonnet 5.5; 0.025x on Fable 5.1 | Explicit `cache_control` breakpoints; on Haiku 5.5, cached tokens count toward the 100K-token price threshold |
| Google (Gemini 3.8 Flash) | Storage at $0.50 per 1M tokens per hour (introductory; $1.00 from January 1, 2027) | $0.075 vs $0.75 input (0.1x, at introductory rates through December 31, 2026; $0.15 vs $1.50 from January 1, 2027) | Storage billed while the cache lives; Gemini 4 Argon supports implicit caching only, so explicit cache code does not apply to it |

Because the read discount is so deep, a 1.25x write pays for itself on the first cache hit, and a 2x one-hour write comes out ahead from the second hit (break-even is about 1.1 reads). OpenAI's 12-hour write at 2.5x also comes out ahead from the second hit (break-even about 1.7 reads at a 0.1x read rate). The older teaching point that OpenAI caching is about 50% off with no write fee is wrong for current OpenAI flagships: GPT-5.6 and later charge 1.25x to write, and even GPT-5.5 and GPT-5.4, which have no write fee, read at 0.1x. Two caveats to state plainly when teaching: the headline savings apply to the **cached prefix only**, not the whole bill, and read rates now differ by 4x within a single vendor, so compute cached cost per task per model rather than with one multiplier.

How to actually capture it (the discipline):
- **Order prompts static to dynamic.** Put system instructions, tool definitions, and few-shot examples first (the stable prefix), and the user query last. Exact-prefix matching means any change near the front invalidates the entire downstream cache.
- **Stabilize the prefix.** No timestamps, request IDs, or per-call nonces in the cached region; pin tool-definition ordering. When a tool must appear mid-session, Anthropic's `inline-tools-2026-09-15` beta adds it in a mid-conversation system message without invalidating the cached prefix.
- **Watch the TTL economics.** A prefix reused less often than its TTL never hits. Both vendors now sell a longer TTL for a bigger write: Anthropic's 1-hour cache at 2x, and OpenAI's 12-hour cache at 2.5x (October 20) alongside its 30-minute option. The 12-hour option pays for prefixes reused at least twice in 12 hours but rarely within 30 minutes, such as a per-customer knowledge pack. Bursty low-reuse traffic may not benefit at all.
- **Instrument cache-hit rate per call** as a first-class metric. Prompt rewrites, model version bumps, and reordering silently drop hit rate. Both vendors now expose it: OpenAI shipped a prompt-caching dashboard (August 20, 2026) and Prompt Cache Diagnostics in the Responses API (GA September 8), and Anthropic's cache diagnostics went GA on September 23.

Distinct from prefix caching, **exact-match and semantic caching** serve whole responses for repeated *queries*. Exact-match keys on the literal request (cheap, zero false positives); semantic caching embeds the query and serves cached responses for similar hits (higher hit rate on natural-language traffic, but a false-hit risk worth guarding). Layer them: exact-match, then semantic, then prefix, and measure hit rate per layer. Conceptually this is the billing-layer monetization of the same KV reuse described in [KV Cache and Context Caching](../04-inference-optimization/02-kv-cache-and-context-caching.md).

---

## Batch and Async Economics

OpenAI, Anthropic, Google and Bedrock all offer a roughly **50% discount on asynchronous work** (input and output): Batch APIs with a completion ceiling around 24 hours, and at OpenAI, Google and Bedrock a Flex tier at the same 0.5x for latency-tolerant synchronous calls. OpenRouter added a cross-provider Batch API on September 22, 2026 (generally 50% off on 70+ models; OpenRouter reports that in its beta the median batch finished in 7 minutes and 90% within an hour). The decision rule is simple: **use batch whenever no human or system is waiting on the token.** High-value batch workloads include evaluation and regression suites, bulk classification and labeling, corpus-scale summarization and document processing, backfills after a prompt or model change, and A/B testing prompt variants. For the entire offline tier of a product, not batching leaves about half the money on the table.

The third lane is **provisioned/reserved throughput** (AWS Bedrock Provisioned Throughput with 1- or 6-month commitments, Bedrock's Reserved tier with 1- or 3-month capacity reservations, Azure OpenAI PTUs): reserved capacity at an hourly rate regardless of usage, reported to save on the order of 15-70% on sustained workloads, economical only at high, predictable utilization. Anthropic no longer sells Priority Tier capacity commitments (existing contracts run to term), so guaranteed Claude capacity now comes through custom agreements or the clouds' reserved and priority tiers. The mental model mirrors cloud compute: pay-per-token (including batch) for spiky or uncertain demand, and reserved capacity once utilization is high and steady.

The fourth lane is **paying for speed**: OpenAI's Fast tier (2x; renamed from Priority on July 30, 2026) and Ultrafast (6x; GA on GPT-6 Astra since September 29 and on GPT-6.1 Sol for all API users since October 8), Gemini Priority (~1.8x), Bedrock Priority (1.75x), and Claude fast mode (2x on Opus 5.5; out of research preview since October 15, on the Claude API, Foundry and Google Cloud). One model can now span a 12x price range by tier alone, from 0.5x to 6x, so the cost model needs a tier policy per workload class:

| Workload | Who is waiting | Tier |
|----------|----------------|------|
| Evals, backfills, bulk labeling | No one | Batch (0.5x) |
| Background agents, async enrichment | A system; minutes are fine | Flex (0.5x) where offered |
| Chat, interactive agents | A human; seconds | Standard |
| Voice and hard latency SLOs | A human; sub-second | Fast or Priority (1.75-2x); Ultrafast (6x) only where a measured SLO pays for it |

---

## The FinOps Discipline

The FinOps Foundation's framing: **inference is 80-90% of total GenAI spend** in many deployments, so the discipline centers on per-request inference economics, not training. The operational core:

- **Attribution.** Tag every call by team, feature, customer/tenant, model, route, and environment. The technical enabler is a token proxy or [gateway](03-ai-gateways-and-model-routing.md) in front of the API that identifies the source of each call. Without attribution there is no way to compute unit economics or see which use cases earn their cost.
- **Showback before chargeback.** Start with visibility dashboards (per-provider, per-model, per-team, per-tenant, with daily forecasts and spike alerts), then graduate to billing teams once the tags are trustworthy.
- **Unit economics.** Track cost per user, per conversation, per resolved ticket or case, and AI cost as a percentage of revenue and of gross margin. Teach an AI product like a cost-of-goods business.
- **Margin reality.** Reported snapshots put AI-product gross margins roughly 25-30 points below the 80-90% of traditional SaaS, because every request has a variable token cost. This is why **outcome-based pricing** (per resolved ticket, per completed task) is rising, with reported anchors like a fixed price per resolved support ticket. The imperative: know your cost per resolution before you price per resolution.
- **Budgets that fail closed.** Provider caps are blunt instruments. Anthropic's usage tiers (Start, Build, Scale) carry monthly spend caps of $500, $1,000 and $200,000; hitting one returns HTTP 429 with `enforced_spend_limit_reached` and no `retry-after` until 00:00 UTC on the 1st of the next month, for the whole organization. Claude Managed Agents session budgets (August 7, 2026) pause an individual session with `budget_reached`, which is the granularity you want. Set your own per-feature and per-session budgets well below provider caps, so a runaway feature trips your alert instead of a provider cutoff for every product.
- **Tooling.** Gateways give real-time per-request control and spend caps; FinOps platforms (Helicone, Vantage, Finout, Amnic, and cloud cost tools) give cross-cloud allocation and chargeback. Mature stacks run both. Verify a tool's current status before standardizing on it; this category churns (Helicone, for example, was acquired by Mintlify in March 2026), so keep cost data exportable in your own warehouse.

---

## Structural Cost Decisions

These architecture-level choices move cost by 2-50x, beyond per-call tuning:

- **Right-sizing, cascades, and routing.** Route to the cheapest model that clears a quality bar, and escalate only on low confidence. Reported savings of 45-85% at ~95% quality retention (FrugalGPT is the canonical reference), with the escalation rate as the live cost variable. See [AI Gateways and Model Routing](03-ai-gateways-and-model-routing.md).
- **Self-host vs API break-even.** The reported break-even against a frontier API sits in the high tens to hundreds of millions of tokens per month, but the load-bearing warning is hidden cost: raw GPU rental is only 30-40% of true cost, so apply a ~2.5-3x multiplier, and engineering labor often exceeds infrastructure. 2026 pushed the break-even further toward APIs: GPU rental rose (Silicon Data's B200 index was up 27.6% year to date entering September, then climbed from $5.86 per GPU-hour in early October to $6.50 on November 1), while API prices fell (GPT-6 Luna and Claude Haiku 5.5 at $0.10/$0.50 per 1M undercut most self-hosted open-weight serving for routing and classification tiers). For most teams, managed APIs are cheaper once the full stack is counted; self-host wins at high, predictable, well-utilized volume or for data-residency reasons. See [LLM Infrastructure](01-llm-infrastructure.md).
- **RAG vs long context.** Retrieval is dramatically cheaper per query than stuffing a long context, since you pay for a few relevant chunks instead of a giant prompt. Long context wins for small static document sets; RAG wins for large or frequently changing corpora and high query volume. See [RAG Fundamentals](../06-retrieval-systems/01-rag-fundamentals.md).
- **Distillation.** Fine-tuning a small model to within a couple of accuracy points of a frontier model on a locked eval is reported to cut per-token cost by 5-40x, with payback in weeks to months at high volume; it wins on narrow, high-volume tasks and fails on open-ended long-tail work. Plan where the student model will live: OpenAI is winding down its fine-tuning platform, existing customers can no longer create new fine-tuning jobs from January 6, 2027, and its October 14 notice shuts down GPT-4o, GPT-4o mini, GPT-4.1 and GPT-4.1 mini snapshots, and fine-tunes of them, on January 14, 2027, pointing to GPT-6 Luna, which cannot be fine-tuned. Open weights or another platform is the safer home. See [Knowledge Distillation](../03-training-and-adaptation/05-knowledge-distillation.md) and the [distillation case study](../16-case-studies/19-customer-distillation-pipeline.md).
- **Output and prompt engineering.** Terse output contracts, `max_tokens` caps, structured outputs, and trimming few-shot examples once a model is reliable are reported to cut tokens 20-40% at minimal quality loss.
- **Who pays for inference.** A new option is emerging: Sign in with ChatGPT (announced at OpenAI DevDay on September 29, 2026; limited trial for selected partners) can let eligible Responses API requests run on the user's own ChatGPT plan instead of the developer's API account. If it opens up, it moves some inference cost off your P&L, at the price of tying identity and AI billing to one vendor.

---

## Cost Anti-Patterns

| Anti-pattern | Mechanism | Fix |
|--------------|-----------|-----|
| No caching | Re-paying for the static system prompt (~69% of input) every call | Stable prefix plus provider prefix caching |
| Oversized model | Frontier model on tasks a small model handles | Right-size, cascade, route |
| Unbounded output | No `max_tokens`, verbose formats | Caps and terse output contracts |
| Reasoning on by default | Extended thinking for trivial tasks | Gate thinking by task complexity |
| Retry storms | Transient error triggers unbounded retries | Bounded retries and circuit breakers |
| Runaway agent loops | A plan-act loop never terminates; tool errors read as "retry" | Hard step, token, and retry ceilings inside the loop |
| Unbounded memory | History accrues without summarization | Windowing and summarization |
| Long-context stuffing | A giant context as the default retrieval | RAG for large or changing corpora |
| No attribution | Untagged shared spend | Gateway token proxy plus tags |
| Real-time for offline work | Sync API for evals, backfills, labeling | Batch API or Flex tier |
| Prompt just over a long-context threshold | OpenAI bills the whole request at long-context rates above 272K input; Claude Haiku 5.5 bills it at 5x above a 100K-token prompt | Trim or retrieve to stay under, or route to a flat-priced model |
| Residency on everything | ~1.1x on every token for tenants that do not need it | Pin geography per tenant at the gateway |
| Budgeting on promo prices | Intro rates end (Gemini 4 Argon doubles on December 1, Gemini 3.8 Flash on January 1, 2027) | Forecast on list price |
| Retrying a spend-cap 429 | The cap does not reset until the 1st of the month | Branch on error code; fail over and page |

Reported real incidents make the agent-loop row concrete: runaway agents have burned tens of thousands of dollars over a single weekend before anyone noticed. Hard ceilings inside the loop, not after-the-fact alerts, are the defense; managed runtimes now offer them natively (Claude Managed Agents session budgets).

---

## Interview Questions

### Q: Your LLM bill doubled month over month with flat traffic. How do you find and fix it?

**Strong answer:**
First, attribution: if every call is not tagged by feature, team, model, and route through a gateway or proxy, that is the first fix, because you cannot debug what you cannot see. With attribution I would break spend into the token-spend layers and look for the usual culprits: a prompt change that broke cache-hit rate (the system prompt is ~69% of input tokens, so a cache regression is huge), extended thinking switched on for simple tasks (billed at the output rate and invisible in the response), an agent loop whose step count crept up, unbounded output or conversation history, or a retry storm. Then the pricing-structure culprits that flat traffic hides: prompts that grew past a long-context threshold (above 272K input, OpenAI bills the whole request at the higher rate; on Claude Haiku 5.5 the line is a 100K-token prompt and the jump is 5x), traffic that drifted onto a Fast or residency-pinned tier, a model swap whose cache-read rate differs, or an introductory price that ended. The highest-ROI fix is almost always restoring prompt caching, then right-sizing the model and capping output. I would also move any offline work (evals, backfills) to the batch API or a Flex tier for roughly half off, and set per-feature budgets with spike alerts so the next doubling pages someone on day one.

### Q: Why do AI products have worse gross margins than SaaS, and what do engineers do about it?

**Strong answer:**
Because every request carries a variable token cost, so an AI product behaves like a cost-of-goods business rather than zero-marginal-cost software; reported margins run roughly 25-30 points below typical SaaS. Engineers attack it on two fronts. Structurally: cache the static prompt prefix, right-size and cascade models, prefer RAG to long-context stuffing, distill high-volume narrow tasks onto a small model, and batch the offline tier. Operationally: instrument unit economics (cost per conversation, per resolved outcome) so pricing can move toward outcome-based models, which only works if you know your cost per resolution. The cost levers are an engineering responsibility, not just a finance one.

### Q: A team wants to move a 300K-token-context workload from Claude Sonnet 5.5 to GPT-6 Sol because both list at $2/$10. What do you check?

**Strong answer:**
That the list prices match is the least informative fact. First, the long-context cliff: above 272K input tokens OpenAI bills the whole GPT-6 Sol request at $4/$15, while Sonnet 5.5 bills flat to 1M (Haiku 5.5 is the Anthropic exception, with its own cliff at 100K). Uncached, a 300K-in, 2K-out call costs about $0.62 on Sonnet 5.5 ($0.60 input plus $0.02 output) and about $1.23 on GPT-6 Sol, so the "same price" move roughly doubles per-call cost unless the context is trimmed under 272K or replaced with retrieval. If the team wants OpenAI anyway, GPT-6.1 Terra ($1/$5, repriced to $2/$7.50 above 272K) lands at about $0.62 for the same call, so the decision goes back to quality on our evals. Second, caching: since October 7 Sonnet 5.5 reads cache at $0.10, half GPT-6 Sol's $0.20, so a cache-heavy loop favors Sonnet further. TTLs differ too: Anthropic sells 5-minute and 1-hour writes at 1.25x and 2x, OpenAI 30-minute and 12-hour writes at 1.25x and 2.5x, so I would match each to the workload's reuse interval. Third, reasoning: Sonnet 5.5 defaults to high effort and cannot fully disable thinking, so I would compare total billed output, not visible output, on our own traces. Fourth, the multipliers: residency (about 10% on either side if pinned) and service tier. Finally, quality and contract: run our eval suite, and check request-shape differences such as forced tool choice. I would then decide on measured cost per task, not on the rate card.

---

## References

- Datadog, [State of AI Engineering 2026](https://www.datadoghq.com/state-of-ai-engineering/)
- FinOps Foundation, [Optimizing GenAI Usage](https://www.finops.org/wg/optimizing-genai-usage/) and [FinOps for AI](https://www.finops.org/wg/finops-for-ai-overview/)
- Anthropic, [prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching); OpenAI, [prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)
- Anthropic, [pricing](https://platform.claude.com/docs/en/about-claude/pricing), [service tiers](https://platform.claude.com/docs/en/api/service-tiers) and [rate limits](https://platform.claude.com/docs/en/api/rate-limits); OpenAI, [pricing](https://developers.openai.com/api/docs/pricing)
- Anthropic, [Message Batches API](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing); OpenAI, [Batch API](https://platform.openai.com/docs/guides/batch)
- Anthropic, [Claude Opus 5.5 and coding-session telemetry](https://claude.com/blog/claude-opus-5-5-built-for-coding-sessions-that-use-more-context)
- Chen et al., "FrugalGPT" arXiv:2305.05176
- Bessemer, [the AI pricing and monetization playbook](https://www.bvp.com/atlas/the-ai-pricing-and-monetization-playbook)

---

*Previous: [AI Gateways and Model Routing](03-ai-gateways-and-model-routing.md)*
