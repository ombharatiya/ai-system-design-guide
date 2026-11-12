# Pricing and Costs

Understanding the cost structure of LLM systems is essential for production planning. This chapter covers pricing models, cost optimization strategies, and total cost of ownership analysis.

## Table of Contents

- [Pricing Models](#pricing-models)
- [Current API Pricing](#current-api-pricing)
- [Cost Calculation](#cost-calculation)
- [Cost Optimization Strategies](#cost-optimization-strategies)
- [Context Caching Economics](#context-caching-economics)
- [Self-Hosting & GPU Cloud Arbitrage](#self-hosting--gpu-cloud-arbitrage)
- [Total Cost of Ownership](#total-cost-of-ownership)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Pricing Models

### Token-Based Pricing

Most LLM APIs charge per token:

```
Cost = (input_tokens × input_rate) + (output_tokens × output_rate)
```

**Key observations:**
- Output tokens usually cost 2-8x more than input tokens (5x on Claude Opus 5.5, Sonnet 5.5 and Haiku 5.5, GPT-6 Sol, GPT-6.1 Terra, Gemini 4 Argon and Gemini 3.8 Flash; 3x on Grok 4.7; 6x on GPT-5.6 Sol and Terra at list; about 8x on Gemini 3.5 Flash-Lite; 2x on hosted MiMo-V2.6), and some open-weight hosts charge the same for both (Together AI's Llama 3.3 70B at $1.04 / $1.04), so output-heavy workloads should be priced on the output column first
- Reasoning tokens bill as output, and on the newest Claude models thinking can no longer be fully switched off
- The rate card is only the base: long-context cliffs, cache terms, speed tiers, residency and promotions multiply it

### The Multipliers on Top of the Rate Card (November 2026)

Two requests with the same token counts on the same model can differ in price by more than 10x. These are the dimensions to model explicitly:

| Dimension | How it bills | Current examples |
|-----------|--------------|------------------|
| **Long-context cliff** | The **whole request** moves to a higher rate once the prompt crosses a threshold, not just the excess | Claude Haiku 5.5 above a 100K-token prompt: 5x on every token type, with cache reads and writes counted toward the threshold. OpenAI above 272K input: 2x input and cache, 1.5x output (Batch and Flex too, and Fast on the GPT-6 and GPT-5.6 families). xAI Grok 4.6/4.7 at or above 200K: all tokens doubled. Gemini 3.1 Pro Preview above 200K. Other Anthropic Claude 4.6-and-later models: flat to 1M. Gemini 4 Argon: no prompt-length tiers |
| **Cache write / read** | Premium to write a prefix, discount to reread it within the TTL | OpenAI GPT-5.6 and later: write 1.25x (30-minute TTL) or 2.5x (12-hour TTL, Responses API only, since October 20); read 0.1x for both, 0.05x on GPT-6.1 Sol and GPT-6.1 Terra. Anthropic: write 1.25x (5 min) or 2x (1 h); read 0.1x, 0.05x on Opus 5.5 and Sonnet 5.5, 0.025x on Fable 5.1. Gemini: read 0.1x plus hourly storage; Gemini 4 Argon caches implicitly only (hits at 0.05x, no explicit caches) |
| **Service tier** | Pay less to wait, more to go first or faster | Batch 0.5x at OpenAI, Anthropic and the Gemini API; Flex 0.5x where offered (OpenAI, Gemini API, Bedrock for most models; Anthropic has no Flex tier, and GPT-6 Astra on Bedrock supports neither Flex nor Priority). OpenAI Fast (renamed from Priority on July 30) 2x, 2.5x on GPT-5.5; Ultrafast 6x on GPT-6 Astra and, since October 8, GPT-6.1 Sol for all API users. Gemini Priority about 1.8x. Bedrock Priority 1.75x. Anthropic fast mode on Opus (2x; out of preview on Opus 5.5 since October 15); Anthropic no longer sells Priority Tier |
| **Data residency** | Region-pinned processing costs about 10% more | Anthropic `inference_geo: "us"` 1.1x on all token types (Claude 4.6+, stacking on Haiku 5.5's over-100K prices). OpenAI residency endpoints for models released from March 5, 2026. Bedrock and Google Cloud regional Claude (on Google Cloud, Google managed, current models such as Opus 5.5, Sonnet 5.5, Haiku 5.5 and Fable 5.1 run only on the global or `us`/`eu` multi-region endpoints, 10% over global; single-region endpoints stop at Sonnet 4.6). Bedrock regional GPT-6 Astra ($11/$55). Mistral EU inference |
| **Promotional window** | Temporary rate with an end date, or only a floor date | Gemini 4 Argon at $2/$10 through November 30, 2026, then $4/$20. Gemini 3.6/3.7/3.8 Flash at half price through December 31, 2026. GPT-5.6 Sol $4/$20, which OpenAI says is available until November 30, 2026, with no price stated after it |
| **Time of day** | Off-peak discount | DeepSeek off-peak is 50% of peak; peak is 01:00-04:00 and 06:00-10:00 UTC on weekdays only, excluding Chinese public holidays |
| **Per-minute** | Session or audio time instead of tokens | `gpt-live-1` $0.05/min plus backend tokens; `gpt-transcribe` $0.0045/min |
| **Billed refusals** | Safety stops that cost money | Anthropic bills pre-output refusals in the `bio`, `frontier_llm` and `reasoning_extraction` categories from September 24, 2026; other categories stay unbilled |
| **Data for discount** | Cheaper if the vendor may use your traffic | Meta `muse-spark-1.3-contributor` at $0.10/$0.20 versus $1.25/$4.25 |
| **Input-only billing** | Typed answers instead of text, with output free | OpenAI Decisions API (GA October 22, on `gpt-6-luna`): 1 to 400 questions per request, $0.10 per 1M uncached input, output not charged |
| **Runtime meters** | Session or sandbox time on top of tokens | Claude Managed Agents $0.08 per running session-hour (idle free). OpenAI Agents API container rates: small $0.03, medium $0.12, large $0.48 per 20-minute session, billed by the minute with a 5-minute minimum |

A request's effective price is therefore:

```
effective_cost = residency_mult × tier_mult × (
      uncached_in × in_rate(prompt_size)
    + cache_write × write_rate(prompt_size)
    + cache_read  × read_rate(prompt_size)
    + (visible_output + reasoning_tokens) × out_rate(prompt_size)
)
# rate(prompt_size) jumps for the WHOLE request past the vendor's long-context threshold
```

### Volume and Commitment Pricing

Public list prices carry no automatic volume discount. Discounts come from negotiated commitments, and the figures below are illustrative of enterprise deals, not public terms:

| Tier | Monthly Spend | Typical negotiated discount |
|------|---------------|-----------------------------|
| Standard | $0 - $5K | 0% |
| Growth | $5K - $50K | 10-20% |
| Enterprise | $50K+ | Custom negotiation |

```
Standard: $2.50 / 1M input tokens
Committed (1-year): $2.00 / 1M input tokens (20% savings)
```

**Capacity is now bought separately from price.** Anthropic no longer sells Priority Tier commitments (existing contracts run to term, and Claude Haiku 5.5 is not supported on Priority Tier, so a Haiku 4.5 commitment does not carry over to it), so guaranteed capacity comes from custom agreements or cloud reservations such as Bedrock Reserved (1- or 3-month terms). Anthropic's self-serve tiers (Start, Build, Scale) carry hard monthly spend caps of $500, $1,000 and $200,000; hitting one returns HTTP 429 with `enforced_spend_limit_reached` and no `retry-after` until 00:00 UTC on the 1st of the next month. A budget cap is now an availability risk, so alert well before it.

---

## Current API Pricing

### November 2026 Pricing

> **Last verified: November 6, 2026.** Prices change frequently. Always re-check: [OpenAI](https://developers.openai.com/api/docs/pricing), [Anthropic](https://platform.claude.com/docs/en/about-claude/pricing), [Google](https://ai.google.dev/gemini-api/docs/pricing), [xAI](https://docs.x.ai/developers/models), [DeepSeek](https://api-docs.deepseek.com/quick_start/pricing), [Mistral](https://mistral.ai/pricing/api). Rows for models launched or repriced since October 1 were checked against vendor pages for this date; other first-party rows reflect the October 1 check, with no change found to xAI text, DeepSeek token, Gemini 3.8 Flash, GPT-6 Astra, Opus or Fable prices. Rows marked "carried over" reflect the August 15, 2026 check. Third-party host prices (OpenRouter, Together AI, Fireworks) are each host's own list price and vary by provider.
>
> **What changed between November 2 and November 6, 2026:**
> - **A new top of the gated ladder.** GPT-6-Cyber (November 5) is $20 / $100 ($2 cached), twice GPT-6 Astra, with no Batch, Flex or Fast rates.
> - **MAI-Thinking-1 has a price:** from $2 / $8 per 1M on Foundry, but a default quota of 0 TPM.
> - **Open-weight prices to add:** Kimi K3.1 Pro at $2 / $10 on Moonshot's API (weights published November 4), and the full Tencent Hy4 at $0.417 / $1.251 on OpenRouter.
> - **Lifecycle with a price effect:** Gemini 2.5 Pro and 2.5 Flash were deprecated on November 2 and from December 1 bill at Gemini 3.8 Flash and 3.5 Flash-Lite rates; GPT-5.6 Sol's promotion is now dated to November 30; Lambda's on-demand GPU prices rise about 15% on November 16 (scheduled).
>
> **What changed between October 1 and November 2, 2026:**
> - **Anthropic's fast tier fell 90%, with a cliff.** Claude Haiku 5.5 (October 7) lists at $0.10 / $0.50 for prompts up to 100,000 tokens and $0.50 / $2.50 above, on the whole request, with cache reads and writes counted toward the threshold. It is the first Claude 4.6-or-later model that is not flat to 1M.
> - **OpenAI filled the middle and the top.** GPT-6.1 Terra (October 26) is $1 / $5, half of GPT-5.6 Terra; GPT-6 Rosalind (October 30, trusted access) is $15 / $75; GPT-6.1 Sol Ultrafast opened to all API users on October 8 at $12 / $60.
> - **Meta priced a closed flagship at the frontier.** Muse Wonder (October 12) is $4 / $20 ($0.20 cached) on the Meta Model API, Opus 5.5's list price; the stronger Muse Wonder Full is sold only inside Muse Code subscriptions.
> - **Cache terms moved again.** Sonnet 5.5 cache reads fell from $0.20 to $0.10 (0.05x) on October 7, and OpenAI added a 12-hour TTL on October 20 with writes at 2.5x input.
> - **Gemini 4 Argon went GA with a dated price rise:** $2 / $10 through November 30, then $4 / $20 from December 1, with implicit caching only.
> - **Speed tiers matured.** Opus 5.5 fast mode ($8 / $40) left research preview on October 15; Opus 4.8 fast mode is removed November 30.
> - **A new billing shape:** OpenAI's Decisions API (GA October 22) bills only uncached input, at $0.10 per 1M, for typed answers.
> - **Increases to plan for:** Argon doubles December 1; Mistral Large 3 users moving to Large 4 ($0.70 / $2.10) pay 40% more; and Google now serves `gemini-3.5-flash` requests with 3.6 Flash without saying how they bill.

#### OpenAI (GPT-6 Generation)
| Model | Input / 1M | Output / 1M | Notes |
|-------|------------|-------------|-------|
| **GPT-6 Astra** | $10.00 | $50.00 | Staged release from September 3, 2026 (Daybreak program first), in the API by September 5. Cached $1.00, cache write $12.50 (30-minute) or $25.00 (12-hour). Above 272K input the whole request bills at $20 / $75. Batch and Flex $5 / $25; Fast $20 / $100; Ultrafast $60 / $300 (GA September 29). 1.05M context, 128K output. No `none` effort, temperature, top_p or logprobs; tool calling requires the Responses API. Bedrock GA September 8 ($11 / $55 for In-Region and Geo inference); also GA on Microsoft Foundry. |
| **GPT-6 Rosalind** (`gpt-6-rosalind`) | $15.00 | $75.00 | October 30, 2026, trusted-access program only. Cached $1.50; above 272K $30 / $112.50. No Batch, Flex or Fast rates listed. |
| **GPT-6-Cyber** (`gpt-6-cyber`) | $20.00 | $100.00 | November 5, 2026, Daybreak Red plus a separate model approval, `access_programs.cyber: "daybreak_red"` and IP allowlisting. Cached $2; cache write $25 (30-minute) or $50 (12-hour). 1,050,000-token context (922,000 max input, 128,000 max output). Chat Completions and Responses only; no Batch, long-context, Flex or Fast rates. |
| **GPT-6.1 Sol** | $2.00 | $10.00 | September 29, 2026. Cached $0.10 (0.05x), cache write $2.50 (30-minute) or $5.00 (12-hour); above 272K $4 / $15. Batch and Flex $1 / $5; Fast $4 / $20; Ultrafast $12 / $60 ($24 / $90 above 272K) for all API users since October 8. Default effort `medium`. On Bedrock under OpenAI's own lifecycle terms. |
| **GPT-6.1 Terra** | $1.00 | $5.00 | October 26, 2026. Cached $0.05 (0.05x), cache write $1.25 (30-minute) or $2.50 (12-hour); above 272K the whole request bills $2 / $7.50. Batch and Flex $0.50 / $2.50; Fast $2 / $10; no Ultrafast. Effort `none` to `max`, default `medium`. Also on Bedrock. |
| **GPT-6 Sol** | $2.00 | $10.00 | September 22, 2026. Cached $0.20, cache write $2.50; above 272K $4 / $15; Batch and Flex $1 / $5. An image-understanding bug was fixed on September 25 under the same model ID: rerun image evals. Bedrock GA September 22. |
| **GPT-6 Luna** | $0.10 | $0.50 | September 22, 2026. Cached $0.01, cache write $0.125; above 272K $0.20 / $0.75; Batch and Flex $0.05 / $0.25. Same September 25 image fix. Also the model behind the Decisions API ($0.10 per 1M uncached input, output not charged). |
| **GPT-5.6 Sol** | $4.00 promo ($5.00 list) | $20.00 promo ($30.00 list) | Promotional since August 21, 2026; OpenAI's pricing and model pages say it is "available until November 30, 2026" and state no price after it. The $5 / $30 list is inferred from the stated 20% and 33% cuts; budget on it from December 1. Above 272K $8 / $30 at the promo rate. Cached $0.40, cache write $5.00. No longer in OpenAI's Ultrafast table. Superseded by GPT-6 Sol. |
| **GPT-5.6 Terra** | $2.00 | $12.00 | Above 272K $4 / $18. Superseded by GPT-6.1 Terra at half the input price and well under half the output price. |
| **GPT-5.6 Luna** | $0.20 | $1.20 | Above 272K $0.40 / $1.80. Superseded by GPT-6 Luna at roughly half the price. |
| **GPT-5.6-Cyber** | $12.50 | $75.00 | August 10, 2026. Cached $1.25. 400K context, no long-context tier. Daybreak Red tier only (identity verification, legal attestations, Responses API only; requests set `access_programs.cyber: "daybreak_red"`). |
| **GPT-Rosalind** (`gpt-rosalind-research`) | $5.00 | $25.00 | Life-sciences reasoning model, GA September 8, 2026 through a trusted-access program. Cached $0.50. Still listed at this rate beside `gpt-6-rosalind`. |
| **GPT-5.5** | $5.00 | $30.00 | April 23, 2026. Cached $0.50; above 272K $10 / $45. Fast is 2.5x on this model. |
| **GPT-5.5 Pro / GPT-5.4 Pro** | $30.00 | $180.00 | No cached-input discount; above 272K about $60 / $270. Pro is also `reasoning.mode: "pro"` on the GPT-5.6 and GPT-6 families at the selected model's standard rates. |
| **GPT-5.4** | $2.50 | $15.00 | Cached $0.25 (0.1x). Above 272K $5 / $22.50. |
| **GPT-5.4-mini** | $0.75 | $4.50 | Cached $0.075. |
| **GPT-5.4-nano** | $0.20 | $1.25 | Cached $0.02. Shuts down April 1, 2027 (to GPT-6 Luna). |
| **GPT-4.1** | $2.00 | $8.00 | Cached $0.50. `gpt-4.1-2025-04-14`, `gpt-4.1-mini-2025-04-14` and fine-tunes of them shut down January 14, 2027 (to `gpt-6-luna`, which cannot be fine-tuned). |
| **GPT-4o** | $2.50 | $10.00 | Cached $1.25. The original `gpt-4o-2024-05-13` snapshot shut down on the OpenAI API October 23, 2026; `gpt-4o-2024-08-06`, `gpt-4o-2024-11-20` and their fine-tunes shut down January 14, 2027 (to `gpt-6-luna`), on about three months' notice. On Azure Foundry, gpt-4o 2024-05-13 retires January 14, 2027, and its Global Standard and Data Zone Standard deployments auto-upgrade to gpt-5.1 (2025-11-13). |
| **GPT-4o-mini** | $0.15 | $0.60 | Cached $0.075. `gpt-4o-mini-2024-07-18` and its fine-tunes shut down January 14, 2027 (to `gpt-6-luna`). |

#### Anthropic (Claude 5.x Generation)
| Model | Input / 1M | Output / 1M | Context | Notes |
|-------|------------|-------------|---------|-------|
| **Claude Opus 5.5** | $4.00 | $20.00 | 1M | September 22, 2026 (`claude-opus-5-5`) on Claude API, Bedrock, Google Cloud, Microsoft Foundry and Claude Platform on AWS. Anthropic's "start here" default. Cache write $5 (5 min) / $8 (1 h); cache read $0.20 (0.05x); Batch $2 / $10; fast mode $8 / $40, out of research preview since October 15, on the Claude API, Claude Models on Google Cloud (Anthropic managed) and Foundry's Azure-hosted deployments. 128K output (300K on Batch with a beta header). Default effort `medium` (Opus 5 defaulted to `high`); thinking cannot be disabled. ZDR available. Retirement not before September 22, 2027. |
| **Claude Sonnet 5.5** | $2.00 | $10.00 | 1M | September 28, 2026 (`claude-sonnet-5-5`), same five platforms, price unchanged from Sonnet 5. Cache write $2.50 / $4.00; cache read $0.10 (0.05x, cut from $0.20 on October 7); Batch $1 / $5. 128K output. API default effort `high` (medium in Claude Code and the Claude apps). `thinking: {type: "disabled"}` returns 400; the lowest setting is `between_tools`. ZDR available. Retirement not before September 28, 2027. |
| **Claude Fable 5.1** | $10.00 | $50.00 | 1M | September 1, 2026 (`claude-fable-5-1`), GA on all five platforms. Cache read $0.25 (0.025x, down from $1.00 on Fable 5); cache write $12.50 / $20.00; Batch $5 / $25. 128K output, default effort `high`. 30-day data retention; ZDR only where Anthropic expressly authorizes it (customers eligible for Enterprise Frontier Safeguards, until EFS is ready). Safeguard refusals can fall back server-side (`fallbacks: "default"`, beta) to Opus 4.8 or Opus 5. Retirement not before September 1, 2027. |
| **Claude Mythos 5.1** | $10.00 | $50.00 | 1M | Same model as Fable 5.1 with looser safeguards. Limited availability: every tier of the Cyber Verification Program (expanded October 6, with Project Glasswing folded in); data retention required. |
| **Claude Haiku 5.5** | $0.10 (prompt up to 100K) / $0.50 (over 100K) | $0.50 / $2.50 | 1M | October 7, 2026 (`claude-haiku-5-5`). Up to a 100K-token prompt: cache write $0.125 (5 min) / $0.20 (1 h), cache read $0.01, Batch $0.05 / $0.25. Over 100K: cache write $0.625 / $1.00, cache read $0.05, Batch $0.25 / $1.25. The higher prices apply to the **whole request**, and prompt length counts cache reads and writes, so a cache hit does not keep a long prompt in the cheap tier. 128K output. About 30% more tokens than Haiku 4.5 for the same text, and the high-resolution image tier bills a 2000x1500 image at about 2.5x Haiku 4.5's visual tokens. Not supported on Priority Tier. Retirement not before October 7, 2027. |
| **Claude Haiku 4.5** | $1.00 | $5.00 | 200K | Superseded by Haiku 5.5, which costs a tenth as much below 100K tokens. Still Active on the Claude API with no deprecation notice as of November 6; its "not sooner than October 15, 2026" floor has passed, and under Anthropic's 60-day notice policy it cannot retire there before about January 2027. Claude's consumer apps dropped it for Free, Pro and Max users on October 29. Foundry still lists November 15, nine days out. |
| **Claude Opus 5** | $5.00 | $25.00 | 1M | July 24, 2026. Legacy, still available. Fast mode $10 / $50. |
| **Claude Sonnet 5** | $2.00 | $10.00 | 1M | June 30, 2026. Legacy, still available. Its introductory price was made permanent on August 10, 2026. |
| **Claude Fable 5** | $10.00 | $50.00 | 1M | June 9, 2026. Legacy, still available. Cache read $1.00 (0.1x), four times the Fable 5.1 rate. |
| **Claude Opus 4.8** | $5.00 | $25.00 | 1M | May 28, 2026. Legacy. Fast mode $10 / $50 until November 30, 2026, when it is removed (deprecated October 30). |
| **Claude Opus 4.7 / 4.6** | $5.00 | $25.00 | 1M | Legacy. Fast mode is no longer offered (4.7 errors; 4.6 runs at standard speed and rates). |
| **Claude Sonnet 4.6** | $3.00 | $15.00 | 1M | Legacy; Sonnet 5.5 is newer and cheaper. |
| **Claude Sonnet 4.5** | $3.00 | $15.00 | 200K | Deprecated September 30, 2026; retires November 30, 2026 on the Claude API and Foundry (`claude-sonnet-4-5-20250929` to `claude-sonnet-5-5`). Bedrock: Legacy since October 8, public extended access (higher provider-set pricing) from January 8, 2027, end of life April 8, 2027. |
| **Claude Opus 4.5** | $5.00 | $25.00 | 200K | Retirement "not sooner than November 24, 2026" on the Claude API, but with no notice issued as of November 2 it cannot retire there before about January 2027. Foundry still lists November 24. |

> [!NOTE]
> **Anthropic billing rules worth memorizing (November 2026):**
> - **Flat long context, with one exception.** Claude 4.6 and later bill the full 1M window at standard rates, with fast-mode premiums flat across it too. The exception is Claude Haiku 5.5: a prompt over 100,000 tokens (cache reads and writes included) moves the whole request to 5x.
> - **Cache reads differ by model:** 0.1x on most models (including Haiku 5.5), 0.05x on Opus 5.5 and Sonnet 5.5, 0.025x on Fable 5.1. Batch is 50% and stacks with caching.
> - **Fast mode** runs on Opus 5.5 ($8 / $40, out of research preview since October 15, on the Claude API, Claude Models on Google Cloud (Anthropic managed) and Foundry's Azure-hosted deployments) and, in research preview, on Opus 5 ($10 / $50). Opus 4.8 fast mode is removed November 30, 2026. It stacks with caching and residency multipliers but is not available on the Batch API.
> - **Residency:** `inference_geo: "us"` costs 1.1x on input, output, cache writes and cache reads for Claude 4.6 and later, including Haiku 5.5's over-100K prices; older models return 400 if the parameter is set.
> - **Refusals:** from September 24, 2026, pre-output refusals with `stop_details.category` of `bio`, `frontier_llm` or `reasoning_extraction` are billed; other categories are not. Break refusal spend out by category on cost dashboards.
> - **Breaking changes that touch cost code:** API version `2026-10-21`, which the new SDK majors send by default, returns 400 on `temperature`, `top_p` and `top_k` at any value on every model. Fable 5.1, Opus 5.5 and Sonnet 5.5 return 400 on `tool_choice` of `any` or `tool`; Haiku 5.5 accepts a forced `tool_choice` but answers that turn without thinking. Thinking blocks are bound to the producing model and conversation, and on Sonnet 5.5 and Haiku 5.5 to the producing account too. Bedrock and Google Cloud (Google managed) set their own retirement schedules for Claude. Anthropic says its dates apply to the Claude API, Claude Platform on AWS, Claude Models on Google Cloud (Anthropic managed) and Foundry, but Microsoft's Foundry retirement schedule still lists earlier Foundry dates for claude-haiku-4-5 (November 15) and claude-opus-4-5 (November 24) than Anthropic's notice policy allows, and it also retires claude-opus-4-6 on February 2, 2027 and claude-sonnet-4-6 on February 10, 2027, a few days before Anthropic's own floors (February 5 and February 17); plan for the earlier date on Foundry and confirm with both vendors.

#### Google (Gemini 3.x and 4)
| Model | Input / 1M | Output / 1M | Context | Notes |
|-------|------------|-------------|---------|-------|
| **Gemini 3.8 Flash** | $0.75 intro ($1.50 from Jan 1, 2027) | $3.75 intro ($7.50 from Jan 1, 2027) | 1M (65K out) | GA September 2, 2026. Output price includes thinking tokens. Batch and Flex $0.375 / $1.875 intro ($0.75 / $3.75 from January 1); Priority $1.35 / $6.75 intro ($2.70 / $13.50 from January 1); cache read $0.075 intro ($0.15 from January 1) plus storage $0.50 per 1M tokens per hour intro ($1.00 from January 1). Thinking levels low, medium, high. Coverage notes it spends more reasoning tokens per task, which offsets part of the low rate. |
| **Gemini 3.7 Flash / 3.6 Flash** | $0.75 intro | $3.75 intro | 1M | Same introductory price and January 1, 2027 doubling. Superseded by 3.8 Flash; since October 8, `gemini-3.7-flash` requests are served by 3.8 Flash. On Vertex AI, `gemini-3.6-flash` retires November 19, 2026 and `gemini-3.7-flash` January 28, 2027 (replacement `gemini-3.8-flash`, no routing described). |
| **Gemini 4 Argon** (`gemini-4-argon`) | $2.00 through Nov 30, 2026; $4.00 from Dec 1 | $10.00 through Nov 30; $20.00 from Dec 1 | 1M shared (65,536 default output) | GA October 13, 2026, paid tier only. Output price includes thinking. Implicit caching only (hits $0.10, then $0.20); explicit caching is not supported. Batch and Flex half of Standard; Priority $3.60 / $18, then $7.20 / $36. No prompt-length tiers. Output beyond 65,536 tokens, up to about 1M, needs the long-decode continuation protocol. Budget at the December price. |
| **Gemini 3.1 Pro Preview** | $2.00 | $12.00 | 1M | Above 200K: $4.00 / $18.00. Still a preview with no shutdown date; Gemini 4 Argon is the GA model above it, and there is no Gemini 3.5 Pro. |
| **Gemini 3.5 Flash** | $1.50 | $9.00 | 1M | Requests route to `gemini-3.6-flash` since October 8; Google has not said how rerouted requests bill. Move off the ID rather than guess. |
| **Gemini 3.5 Flash-Lite** | $0.30 | $2.50 | 1M | Google's recommended target for new projects that would have used 2.5 Flash-Lite. |
| **Gemini 3.1 Flash-Lite** | $0.25 | $1.50 | 1M | Shutdown May 7, 2027 (to 3.5 Flash-Lite). |
| **Gemini 3 Flash Preview** | $0.50 | $3.00 | 1M | Preview. |
| **Gemini 2.5 Pro** | $1.25 | $10.00 | 1M | Above 200K: $2.50 / $15.00. Deprecated November 2, 2026; from December 1 requests route to `gemini-3.8-flash` and bill at 3.8 Flash rates (see below). |
| **Gemini 2.5 Flash-Lite** | $0.10 | $0.40 | 1M | Restricted to existing users; not deprecated. |

Google Search grounding: 5,000 free requests per month shared across Gemini 3 and newer models, then $14 per 1,000. A governance change rides alongside the prices: from October 26, new paid Gemini API projects store requests by default (`store=true`); existing projects keep their setting.

> [!WARNING]
> **Gemini 2.5 lifecycle (price effect):** `gemini-2.5-pro` and `gemini-2.5-flash` were deprecated on the Gemini API on November 2, 2026. From December 1, `gemini-2.5-pro` requests route to and bill at Gemini 3.8 Flash rates ($0.75 / $3.75 intro, doubling January 1, 2027), and `gemini-2.5-flash` requests route to and bill at Gemini 3.5 Flash-Lite rates ($0.30 / $2.50). `gemini-2.5-flash-lite` is not deprecated. Pinned code keeps returning 200s, so the bill and the model's behavior change together. For the breaking changes on the routed targets and the Vertex AI status, see [Model Taxonomy](01-model-taxonomy.md#gemini-31-pro-preview-google---february-2026).

#### xAI (Grok)
| Model | Input / 1M | Output / 1M | Context | Notes |
|-------|------------|-------------|---------|-------|
| **Grok 4.7** | $2.00 | $6.00 | 500K | September 21, 2026. Cached $0.50. At or above a 200K prompt the rate becomes $4 / $12 (cached $1) **for every token in the request**. A fast variant (2x speed, 2x price) is offered only in Cursor and Grok Build, not on the public API. New, larger base model than 4.6; Artificial Analysis Index v5.0 scores it 35 (high), about 20 points below the $2/$10 models. Still xAI's newest text model: no Grok 5 as of November 6. |
| **Grok 4.6** | $2.00 | $6.00 | 500K | August 12, 2026. Superseded by 4.7 at the same price and long-prompt rule. |
| **grok-4.3** | $1.25 | $2.50 | - | Redirect target for the text slugs xAI retired on May 15, 2026 (`grok-4-0709`, `grok-4-fast`, `grok-4-1-fast`, `grok-3`). Old slugs still resolve but bill at grok-4.3 rates and behavior. Azure Foundry still lists `grok-4` and `grok-4-1-fast` as GA. |

#### Open-Weight, Challenger, and Value-Tier Models via API
| Model | Input / 1M | Output / 1M | Context | Provider and License Notes |
|-------|------------|-------------|---------|----------------------------|
| **DeepSeek V4.1-Flash** (`deepseek-flash`) | $0.30 peak / $0.15 off-peak | $1.20 peak / $0.60 off-peak | 1M (384K out) | DeepSeek API from September 10, 2026; served by the V4.1-Flash-1017 build (MIT) since October 17 at the same token prices. AA Index v5.0.1: 33 at max (29 at high), the top released-weight score AA labels open; the max run cost $0.60 per index task. Concurrency limit 2,500. DeepSeek's Search & Fetch API (October 17) adds $0.75 per 1,000 successful requests for each of Search and Fetch. Cache-hit input $0.006 / $0.003. MIT weights; 552B backbone (763B checkpoint), 8B active on prefill and 16B on decode. Legacy `deepseek-v4-flash` names route here at these rates. Together AI and Fireworks list the same $0.30 / $1.20. |
| **DeepSeek V4-Pro** (`deepseek-v4-pro`, build 0813) | $1.32 peak / $0.66 off-peak | $3.96 peak / $1.98 off-peak | 1M | Unchanged since the August 16 repricing. Cache-hit input $0.044 / $0.022. DeepSeek announced on September 10 that all V4-Pro traffic would route to V4.1-Flash from September 14, then reversed that on September 11. V4.1-Pro has not launched. Concurrency limit 500, which caps fan-out. |
| **Xiaomi MiMo-V2.6-Pro** | $0.435 | $0.87 | 1M | September 21, 2026, MIT, 1.02T total / 42B active. OpenRouter price. Artificial Analysis Index v5.0.1: 30, three points behind DeepSeek V4.1-Flash-1017. |
| **Xiaomi MiMo-V2.6-Flash** | $0.14 | $0.28 | 1M | 309B / 15B active, MIT. OpenRouter price; the same numbers DeepSeek V4 Flash charged before August. |
| **Z.ai GLM-5.3** | $1.40 | $4.40 | 1M | Cached $0.26. Weights released August 28, 2026 under MIT plus a Z.ai security-review clause for Model-as-a-Service operators above US$10B revenue over 12 months. Also GA on Mistral La Plateforme (September 28; GLM 5.2 was listed to retire there October 31) and on Amazon Bedrock (October 5, US and Global profiles; AWS says inference data is not shared with the model provider). AA Index v5.0.1: 32 at max, 23 at low. |
| **Z.ai GLM-5.3-Flash** | $0.15 | $0.50 | 1M | August 26, 2026, plain MIT, 320B / 18B active. Cached $0.03. |
| **Moonshot Kimi K3** | $3.00 | $15.00 | 1M | Cached $0.30 (Fireworks lists the same $3 / $0.30 / $15). Custom license: a separate agreement is required for MaaS operators above US$20M revenue over 12 months. Not scored on AA Index v5.0; earlier scores (including the widely quoted 57.1) are on non-comparable versions. On Amazon Bedrock since September 18 with explicit prompt caching (breakpoints after at least 1,024 tokens, entries persist at least 30 minutes, cache hits do not count against input-TPM quotas); the global cross-Region profile is about 10% cheaper than the US one. |
| **Moonshot Kimi K3.1 Pro** | $2.00 | $10.00 | 1M | API October 23, 2026; weights November 4 under the custom Kimi K3.1-Pro License (separate agreement for MaaS above US$20M revenue over 12 months; UI credit above 100M MAU or US$20M monthly revenue). `kimi-k3.1-pro-highspeed` $4 / $20 for 2-3x speed (Moonshot); batch at 60% of standard. Artificial Analysis measured about 36 minutes per index task on standard versus 16.7 on High Speed. First-party prices as listed by AA, because Moonshot's tables rendered empty. AA Index v5.0.1: 46. GPT-6.1 Sol's list price at 9 fewer AA points, with weights you can take elsewhere. |
| **Qwen3.8-Max / -0902** | $2.00 | $6.00 | 1M | Alibaba Model Studio international price for both `qwen3.8-max` and the September 2 `-0902` snapshot, which is API-only. The open 2.4T / 95B-active checkpoint is still the August 12 build and has a MaaS and assistant-product gate above US$50M revenue over 12 months: self-hosters fall behind the API model within weeks. |
| **Qwen3.8-Flash** (hosted Flash-Next) | $0.15 | $0.47 | 1M | Qwen Cloud. Open weights (Qwen3.8-Flash-Next, a Qwen4 architecture preview) use Qwen Community License 1.0, which requires a separate license for any MaaS or coding/office-assistant business regardless of revenue. Qwen4 itself had not shipped as of November 6 (no date, model card or license), so do not price it in. |
| **Tencent Hy4 preview** | $0.834 | $2.501 | 1M | August 28, 2026, Apache 2.0, 770B / 49B active. Tencent TokenHub list (cache hit $0.042); third-party OpenRouter hosts match. |
| **Tencent Hy4** (full, `tencent/hy4`) | $0.417 | $1.251 | - | November 4, 2026; adds image input and computer use. OpenRouter price; Tencent TokenHub lists 6 yuan / 18 yuan per 1M (reported at half price until December 31). No weights or license yet (Tencent promises both, undated), so the preview's Apache 2.0 terms do not carry over. AA Index v5.0.1: 35, labeled proprietary. |
| **Meta Muse Wonder** (closed, `muse-wonder-1.0`) | $4.00 | $20.00 | 1M | October 12, 2026, Meta Model API and Muse Code. Cached $0.20. Opus 5.5's list price at a lower AA Index v5.0 score (43 vs 58, both max effort). Muse Wonder Full has no API price: it is available only through Muse Code subscriptions ($50 and $200 monthly plans). |
| **Meta Muse Spark 1.3** (closed) | $1.25 | $4.25 | 1M | September 2, 2026, Meta Model API and Muse Code. Cached $0.15. `muse-spark-1.3-contributor` costs $0.10 / $0.20 because Meta may use the traffic: check policy before enabling. More verbose than 1.2, so cost per task rose at flat per-token prices. |
| **StepFun Step 5 Preview** | $1.00 | $2.70 | 1M | Cache hit $0.05. 600B / 27B active; weights released around October 15 under StepFun Model License 1.0 (separate agreement for MaaS above US$20M revenue and for any branded device). AA Index v5.0: 29. Verbose: about 160M output tokens on Artificial Analysis's earlier suite against an 81M median. |
| **Mistral Large 4** | $0.70 | $2.10 | 1M | Cached $0.07. Public preview October 6, 2026; open weights October 29 (about 1T total, 49B active) under the Mistral Ecosystem Open-Weight License, which requires a separate commercial license for any MaaS business, with no revenue threshold. Reasoning modes Low, Medium and High, with **High as the default** (the preview's `None` mode is gone), so unset effort spends more tokens; AA scores 26 at High and 23 at Medium. Named replacement for Medium 3.5 and Large 3. |
| **Mistral Medium 3.5** | $1.50 | $7.50 | 256K | Deprecated October 31, 2026; retires April 30, 2027 (to Large 4). Weights under a modified MIT license that withdraws all rights from organizations above US$20M monthly revenue. EU regional inference +10%. |
| **Mistral Large 3** | $0.50 | $1.50 | 256K | Mistral API, AWS Bedrock. Deprecated on the Mistral API October 31, 2026; retires there April 30, 2027. Large 4 costs 40% more. |
| **Microsoft MAI-Thinking-1** | From $2.00 | From $8.00 | 256K total (64K max output) | August 12, 2026, Foundry public preview; price in Microsoft's Foundry blog (updated August 12). About 1T total / 35B active. Global Standard only, no PTU; default quota 0 TPM and 0 RPM, so request capacity first. |
| **MAI-Code-1.1-Flash** | $0.20 | $1.20 | check latest | Microsoft, August 11, 2026. Carried over. |
| **Inception Mercury 2.5** (diffusion) | $0.20 list ($0.04 launch promo) | $0.75 list ($0.15 launch promo) | - | 1,107 tokens/s (vendor-reported). A speed option, not a frontier-quality one. |
| **Llama 3.3 70B** | $1.04 | $1.04 | 128K | Together AI list. Groq shut down `llama-3.3-70b-versatile` for free and developer tiers on August 16, 2026. |
| **Llama 4 Scout / Maverick** | check latest | check latest | 10M / 1M | Groq retired Scout for free and developer tiers on July 17, 2026, and fast-inference hosts now center on gpt-oss, Qwen, GLM and DeepSeek. Scout's effective context degrades fast past 32K. |
| **Gemma 4 / Qwen3.8-27B** | self-host | self-host | 256K / - | Apache 2.0. |
| **Meta Muse Spark Open** | $1.25 (Fireworks) or self-host | $4.25 (Fireworks) or self-host | 1M (1,040K on Fireworks) | October 15, 2026, 816B / 42B active. Fireworks serverless (blog post October 20): cached $0.15, US-only capacity with no US premium, LoRA training through its Serverless Training API. Also on Microsoft Foundry managed compute (since October 15) and GroqCloud preview (131,072 context, contact sales). The same price as the closed Muse Spark 1.3, so choose on license and data terms. Muse Model License: "Built with Muse Spark Open" attribution, and a separate Meta agreement for MaaS operators above US$20M revenue. AA Index v5.0.1: 25. |
| **MiniMax-M3.1-Flash** | self-host | self-host | 1M | Weights October 23, 2026 (hosted service from October 19); about 428B / 23B active per the model card. MiniMax Community License: commercial use only under conditions (one-time notice to MiniMax, prior written authorization above US$20M yearly revenue, "Built with MiniMax M3.1-Flash" display). AA Index v5.0: 29. |

OpenRouter added a Batch API on September 22, 2026: generally 50% off on 70+ models, median batch completion 7 minutes in beta. That brings the batch discount to providers that do not offer one directly.

#### Voice and Transcription (Per-Minute Billing)

Voice moved off pure token billing in 2026. A voice agent's cost is now **minutes × session rate + backend model tokens**, so call duration and idle time matter as much as tokens. Close idle sessions: `gpt-live-1` bills silence and backend wait time.

| Model | Price | Notes |
|-------|-------|-------|
| **`gpt-live-1`** | $0.05 per session minute, billed per second, plus backend model and tool tokens | GA September 10, 2026. Full duplex; delegates reasoning and tools to a Responses model or your own backend. Creating a WebRTC session bills 15 seconds up front, credited once the session runs. |
| **`gpt-realtime-2.1`** | Audio $32 in / $0.40 cached / $64 out per 1M; text $4 / $0.40 / $24 | About $0.019 per minute of caller audio and $0.077 per minute of agent audio before context growth (derived). The whole conversation is re-sent on every response, so long calls get more expensive per minute. |
| **`gpt-realtime-2.1-mini`** | Audio $10 / $0.30 / $20 per 1M; text $0.60 / $0.06 / $2.40 | Replacement for `gpt-realtime-mini`. |
| **`gpt-realtime-translate`** | $0.034/min | One session per target language. |
| **`gpt-transcribe`** | $0.0045/min | Files and committed turns. Replaces `whisper-1` and the `gpt-4o-*-transcribe` models, which shut down February 26, 2027. |
| **`gpt-live-transcribe`** | $0.017/min | Streaming, about 3.8x the batch rate. |
| **Gemini 3.8 Live** | Audio $3 in / $12 out per 1M (about $0.005 / $0.018 per min); text $0.75 / $4.50 | GA September 15, 2026. Same rates for the Extended Thinking variant. |
| **Gemini 3.8 Flash TTS** | $0.50 in / $9 out per 1M promotional through December 31; $1 / $18 from January 1, 2027 | GA September 22, 2026. Flash-Lite TTS is $6 per 1M audio tokens, also doubling January 1. |
| **Gemini 3.5 Transcribe Live** | About $0.009/min blended | GA August 26, 2026. |
| **Microsoft MAI-Transcribe-2** | $0.10 per audio hour (about $0.0017/min), limited-time | Launched September 3, 2026. Budget for a higher rate once the offer ends. |
| **OpenAI TTS** (`gpt-4o-mini-tts`) | $12 per 1M audio tokens | No new OpenAI TTS model shipped in August or September. `tts-1`, `tts-1-hd` and the `gpt-4o-mini-tts` snapshots shut down January 6, 2027. |
| **xAI Grok voice** | $0.08/min plus $0.004 per text input | Flat per-minute speech-to-speech; the `grok-voice-latest` alias moved from `grok-voice-2.0` to `grok-voice-3.0` in October at the same price. xAI speech-to-text is $0.10 per hour (REST) or $0.20 per hour (streaming); TTS is $15 per 1M characters. |
| **Mistral Voxtral 3** (public preview, October 20) | TTS 3 $13 per 1M characters; Mini Transcribe 3 $0.0035/min; Mini Transcribe Realtime 3 $0.008/min; Small Transcribe 3 $0.004/min | Cached input at 10% of each rate. Streaming STT at about half `gpt-live-transcribe`'s price. Voxtral TTS 26.03 on the Mistral API retires April 30, 2027. |
| **Bundled voice-agent platforms** | About $0.04-0.14/min plus telephony | Mistral Voice Agents $0.04 with Mistral Small 4 and $0.08 with larger models (October 20, the cheapest bundled rate here), Telnyx about $0.06 (vendor-reported), Deepgram Voice Agent $0.075 ($0.05 with your own LLM and TTS), ElevenLabs Agents $0.08 ($0.16 burst), Vapi $0.05 plus providers at cost (about $0.08-0.13 in total), Retell about $0.13 in a worked example, Bland $0.12-0.14. ElevenLabs' Eleven Realtime models (rolled out to every workspace by November 2) are not flat: connected audio is $0.080 per additional minute ($0.160 burst above the concurrency limit) plus tokens at $0.20 / $6.00 per 1M for Mini and $1.80 / $9.00 for Max. Cartesia stops hosting Line SDK agents on December 1, 2026. This is the bar a self-assembled pipeline has to beat. |

For architecture tradeoffs between these options, see [Realtime Voice Agents](../18-voice-and-audio-agents/01-realtime-voice-agents.md).

#### Embedding Models
| Model | Cost / 1M tokens | Notes |
|-------|------------------|-------|
| **Cohere embed-v5.0-pro / -fast** | $0.12 / $0.08 | September 30, 2026. Shared embedding space across both, 128K input. |
| **voyage-4-large / voyage-4 / voyage-4-lite** | $0.12 / $0.06 / $0.02 | January 15, 2026. Shared space, 32K input. |
| **voyage-context-4, voyage-code-4** | $0.12 | voyage-code-4 reports +27.54% over voyage-code-3 (vendor-reported). |
| **text-embedding-3-large** | $0.13 | 3072 dims, 8,192-token input. Unchanged. |
| **text-embedding-3-small** | $0.02 | 1536 dims. Carried over. |
| **gemini-embedding-2** | $0.20 (text; Batch $0.10) | Multimodal, stable since April 2026; image $0.45, audio $6.50, video $12.00 per 1M (Google pricing page). |
| **Cohere Embed 4** | $0.10 | Previous Cohere generation; Matryoshka dims 256 / 512 / 1024 / 1536. Carried over. |

Rerankers: Voyage rerank-3 at $0.05 and rerank-3-lite at $0.02 per 1M tokens (September 30, 2026, 32K context); Cohere Rerank 5 GA October 21, 2026 at $0.05 (Pro) and $0.02 (Fast) per 1M tokens.

> [!IMPORTANT]
> **Reasoning tokens are output tokens.** On models with always-on or adaptive thinking (Claude Fable 5.1, Opus 5.5 and Sonnet 5.5; GPT-6 Astra, which has no `none` effort), you pay the output rate for thinking that never appears in the response. This can increase total request cost by 2x-10x on logic-heavy tasks. Thinking cannot be disabled on Opus 5.5, and Sonnet 5.5's lowest setting is `between_tools`, so the cost control is the **effort** setting plus a `max_tokens` cap, not a thinking toggle. Watch silent default changes: Opus 5.5 and Haiku 5.5 default to `medium` (Opus 5 used `high`), GPT-6.1 Sol and GPT-6.1 Terra to `medium`, and Sonnet 5.5 to `high` on the API. Effort is also the biggest lever on quality per dollar: Artificial Analysis scores GPT-6.1 Terra 51 at `max` and 38 at `medium`. Set effort explicitly on every request.

### Retirement and Price-Change Calendar: November 2026 to Mid-2027

Lifecycle is now tracked per **(model, platform)** pair: the same model can retire months apart on the vendor API, Bedrock, Google Cloud and Foundry. Minimum notice differs too, and the stated minimums are not guarantees: OpenAI states 6 months for GA models, 3 months for specialized variants and as little as 2 weeks for previews, yet gave about 3 months for the GA GPT-4o and GPT-4.1 snapshots on October 14 (and 20 days for `gpt-5.4-cyber`); Anthropic gives 60 days; Bedrock promises at least 6 months of Legacy for older models but gave several Claude 3.x regions about a month (Legacy October 10 to 15, end of life November 13), models launched from September 7, 2026 carry a 6-month or 45-day Legacy period, and in Legacy, existing customers can lose access after 15 days of inactivity; Foundry gives Fireworks per-token models 15 days.

**Executed between October 1 and November 6, 2026:** Google deprecated `gemini-2.5-pro` and `gemini-2.5-flash` on the Gemini API on November 2 (billing at 3.8 Flash and 3.5 Flash-Lite rates from December 1). xAI retired `grok-imagine-image-quality` in November: it and `grok-imagine-image-pro` are now served by `grok-imagine-image-2.0-low` at a lower per-image price, so set quality explicitly if you need more than low. Moonshot stopped offering K2 series models to newly registered users on October 31, and Mistral deprecated the Voxtral TTS 26.03 API model the same day. OpenAI's legacy snapshots shut down on October 23 as scheduled (`gpt-4o-2024-05-13`, `gpt-4-turbo`, `o1`, `o3-mini`, `o4-mini`, `gpt-image-1` and the others announced April 22, plus their fine-tunes), following `gpt-3.5-turbo-instruct`, `babbage-002` and `davinci-002` on September 28. Bedrock lists Claude 3 Haiku (September 10), Nova Sonic v1 (September 14), Nova Canvas and Nova Reel (September 30) and Claude Sonnet 4 (October 14) as reached end of life; Claude Opus 4.1 entered higher-priced public extended access on Bedrock October 8. Anthropic API version `2026-10-21` (October 21) made sampling parameters a 400 on every model for anyone who upgrades to the new SDK majors, and Google began serving `gemini-3.7-flash` with 3.8 Flash and `gemini-3.5-flash` with 3.6 Flash (October 8).

**Moved:** `gemini-2.5-flash-image` to March 15, 2027 (from October 2); Foundry `gpt-4.1-nano` to April 14, 2027 (from October 14); the Foundry gpt-4o 2024-05-13 auto-upgrade to January 14, 2027, now targeting gpt-5.1 instead of gpt-5.6-sol on December 9.

**Not executed:** the Claude Haiku 4.5 floor (October 15) passed with no deprecation notice; it is still Active on the Claude API as of November 6.

**Not confirmed:** Google Cloud's Vertex retirement of `gemini-2.5-pro`, `-flash` and `-flash-lite` (October 20); Foundry `gpt-4o-transcribe` and `gpt-4o-mini-transcribe` 2025-03-20 (October 15, still listed as GA); the start of GPT-Rosalind billing (October 5); OpenAI Evals going read-only (October 31); `antigravity-preview-05-2026` (October 5); Mistral's GLM 5.2 (October 31) and OCR 4.0 and Leanstral 1.5 (September 30), past their listed dates with no extension; and Bedrock Nova Premier (September 14). Treat any of these still in your code as broken until you have tested them.

| Date | Kind | Platform | What changes | Move to |
|------|------|----------|--------------|---------|
| Nov 13, 2026 | Retirement | Bedrock | Claude 3 Haiku, 3 Sonnet, 3.5 Sonnet, 3.5 Sonnet v2, 3.7 Sonnet in the remaining listed regions | A current Claude model |
| Nov 15, 2026 | Retirement | Foundry | `claude-haiku-4-5`, `codex-mini` | Claude Haiku 5.5 |
| Nov 16, 2026 | Price | Lambda | On-demand GPU prices rise about 15% at 00:00 UTC: 8x HGX H100 $4.59 and 8x HGX B200 $7.69 per GPU-hour | Re-run self-host break-even |
| Nov 17, 2026 | Retirement | Gemini API | Preview speech models `gemini-3.1-flash-live-preview`, `gemini-2.5-flash-native-audio-preview-12-2025`, `gemini-2.5-flash-preview-tts`, `gemini-2.5-pro-preview-tts`, `gemini-3.1-flash-tts-preview` (earliest-possible date, no routing) | `gemini-3.8-live`; `gemini-3.8-flash-tts` or `gemini-3.8-flash-lite-tts` |
| Nov 19, 2026 | Retirement | Foundry | o1, o1-pro, o3, o3-pro, o3-deep-research; o3-mini, o4-mini | gpt-5.6-sol; gpt-5.6-terra |
| Nov 19, 2026 | Retirement | Vertex AI | `gemini-3.6-flash` (no auto-upgrade described; a Vertex-only date) | `gemini-3.8-flash` |
| Nov 24, 2026 | Retirement | Foundry | `claude-opus-4-5` (the Claude API floor passes the same day with no notice, so not before about January 2027 there) | Opus 5.5 |
| Nov 26, 2026 | Retirement | Bedrock | AI21 Jamba 1.5 | - |
| Nov 27, 2026 | Retirement | Bedrock | Claude 3 Haiku and Claude 3 Sonnet in `ca-central-1` | A current Claude model |
| Nov 30, 2026 | Retirement | Claude API, Foundry | `claude-sonnet-4-5-20250929` | `claude-sonnet-5-5` |
| Nov 30, 2026 | Feature removal | Claude API | Opus 4.8 fast mode (deprecated October 30); `speed: "fast"` errors afterward | Opus 5.5 fast mode ($8 / $40) |
| Nov 30, 2026 | Retirement | OpenAI | Evals, Agent Builder, Reusable Prompts shut down | Promptfoo; Agents SDK |
| Nov 30, 2026 | Price | OpenAI | Last day of GPT-5.6 Sol's $4 / $20 promotion; no later price is stated, so budget the inferred $5 / $30 list | GPT-6 Sol or GPT-6.1 Sol |
| Dec 1, 2026 | Price | Gemini API | Gemini 4 Argon doubles to $4 / $20 (cache hits $0.20; Priority $7.20 / $36) | Budget at the December price |
| Dec 1, 2026 | Routing / price | Gemini API | `gemini-2.5-pro` requests route to `gemini-3.8-flash`, `gemini-2.5-flash` to `gemini-3.5-flash-lite`, billed at the target's rates | Migrate deliberately; retest tool calls |
| Dec 1, 2026 | Retirement | OpenAI | `gpt-image-1.5`, `gpt-image-1-mini`, `chatgpt-image-latest` | `gpt-image-2.5-sunburst` or `-flare`, or `gpt-image-2.6-sunburst` or `-flare` (November 4, same $8 image input / $30 image output / $5 text input per 1M; no Batch rate listed for 2.6, so batch-heavy pipelines stay on 2.5) |
| Dec 1, 2026 | Hosting | Cartesia | Line SDK agent hosting ends | Self-host the agents |
| Dec 11, 2026 | Retirement | OpenAI | `gpt-5-2025-08-07` (and mini, nano), `gpt-5-pro-2025-10-06`, `o3-2025-04-16`, `o3-pro-2025-06-10` | gpt-5.6-sol / terra / luna; pro via `reasoning.mode: "pro"` |
| Dec 13, 2026 | Retirement | Vertex AI | `gemini-live-2.5-flash-native-audio` | `gemini-3.8-live` |
| Dec 15, 2026 | Retirement | Foundry | whisper, tts, tts-hd | - |
| End of Dec 2026 | Retirement | Moonshot | All Kimi K2 series models, including K2.6 and K2.7 Code (planned) | Kimi K3.1 Pro |
| Dec 31, 2026 | Price | Gemini API | Last day of Gemini 3.6 / 3.7 / 3.8 Flash and 3.8 Flash TTS promotional pricing (2x from January 1) | Budget at list |
| Jan 6, 2027 | Retirement | OpenAI | `tts-1`, `tts-1-hd`, dated `gpt-4o-mini-tts` snapshots; fine-tuning job creation ends for existing customers | `gpt-realtime-2.1-mini`; open weights, Azure or Bedrock for tuning |
| Jan 8, 2027 | Retirement / price | Bedrock | Claude Opus 4.1 end of life; Claude Sonnet 4.5 enters public extended access at higher provider-set pricing | Claude 4.6+ |
| Jan 14, 2027 | Retirement | OpenAI | `gpt-4o-2024-08-06`, `gpt-4o-2024-11-20`, `gpt-4o-mini-2024-07-18`, `gpt-4.1-2025-04-14`, `gpt-4.1-mini-2025-04-14`, and fine-tunes of them (announced October 14) | `gpt-6-luna`, which cannot be fine-tuned |
| Jan 14, 2027 | Auto-upgrade | Foundry | gpt-4o 2024-05-13 retires; Global Standard and Data Zone Standard deployments move to gpt-5.1 (2025-11-13); Provisioned deployments do not | gpt-5.1 (latency, cost and style change with no code change) |
| Jan 20, 2027 | Retirement | OpenAI | `gpt-realtime`, `gpt-audio`, `gpt-4o-realtime`, `gpt-4o-audio` families | `gpt-realtime-2.1`, `-2.1-mini`, `gpt-audio-1.5` |
| Jan 28, 2027 | Retirement | Vertex AI | `gemini-3.7-flash` | `gemini-3.8-flash` |
| Feb 2, 2027 | Retirement | Foundry | `claude-opus-4-6` (Anthropic's floor is February 5) | Opus 5.5 |
| Feb 10, 2027 | Retirement | Foundry | `claude-sonnet-4-6` (Anthropic's floor is February 17) | Sonnet 5.5 |
| Feb 26, 2027 | Retirement | OpenAI | `whisper-1`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, `gpt-4o-transcribe-diarize` | `gpt-transcribe`, `gpt-live-transcribe` |
| Mar 15, 2027 | Retirement | Gemini API | `gemini-2.5-flash-image` | A current Gemini image model |
| Mar 30, 2027 | Retirement | Bedrock | Legacy batch: Llama 3.x, Llama 4 Scout and Maverick, DeepSeek-R1 and V3.1, Gemma 3, Qwen3-Coder, GLM 4.7, MiniMax M2 and M2.1, Kimi K2 Thinking, Devstral 2, Voxtral, Nemotron Nano | A current Bedrock model or self-host |
| Apr 1, 2027 | Retirement | OpenAI | `gpt-5.1`, `gpt-5.3-codex`, `gpt-5.4-nano` | GPT-6 Sol, GPT-6 Luna |
| Apr 8, 2027 | Retirement | Bedrock | Claude Sonnet 4.5 end of life | Sonnet 5.5 |
| Apr 14, 2027 | Retirement | Foundry | gpt-4o (2024-08-06, 2024-11-20), gpt-4o-mini, gpt-4.1, gpt-4.1-mini, gpt-4.1-nano | A current OpenAI model on Foundry |
| Apr 30, 2027 | Retirement | Mistral | Mistral Medium 3.5, Mistral Large 3 and Voxtral TTS 26.03 (all deprecated October 31) | Mistral Large 4 (+40% for Large 3 users); Voxtral TTS 3 |
| May 7, 2027 | Retirement | Gemini API | `gemini-3.1-flash-lite` | Gemini 3.5 Flash-Lite |

**Same model, different dates:** Claude Sonnet 4.5 retires on the Claude API November 30, 2026 but runs on Bedrock until April 8, 2027 (at higher extended-access prices from January 8). Claude Opus 4.1 retired on the Claude API August 5, 2026 but runs on Bedrock until January 8, 2027. `gpt-4o-2024-08-06` and its fine-tunes shut down on the OpenAI API January 14, 2027, while Foundry keeps the base model until April 14, 2027 and fine-tuned deployments until October 1, 2027. Grok 4 was retired by xAI on May 15 but Foundry lists it as GA until January 31, 2027. Gemini 3.6 Flash retires on Vertex AI November 19 on a different lifecycle from the same ID on the Gemini API, and Bedrock's Claude 3 Haiku and 3 Sonnet reach end of life November 13 in most regions but November 27 in `ca-central-1`, so the registry needs a date per (model, platform, region). A multi-cloud failover chain can quietly land on a model already retired on your primary, so the model registry needs a date per platform.

### Earlier 2026 Price History

Kept for context; these explain how current prices were reached.

- **September 2026, in one line.** The ceiling held at $10 / $50 (Claude Fable 5.1 flat to 1M, GPT-6 Astra $20 / $75 above 272K), the mid tier converged on $2 / $10 (Sonnet 5.5, GPT-6 Sol, GPT-6.1 Sol), Opus fell to $4 / $20 with Opus 5.5, GPT-6 Luna set a $0.10 / $0.50 floor, cache reads split into 0.1x, 0.05x and 0.025x by model, and DeepSeek V4.1-Flash cut Flash to $0.30 / $1.20 at peak.

- **Anthropic.** Claude Fable 5 launched June 9 at $10 / $50 (Mythos-class with safeguards; Claude Mythos 5, the same model with safeguards lifted, shared the price for Glasswing partners). Claude Opus 4.8 launched May 28 at $5 / $25 with a $10 / $50 fast mode (the Opus 4.7 fast tier had been $30 / $150). Claude Sonnet 5's introductory $2 / $10 became permanent on August 10, and the scheduled September 1 rise to $3 / $15 was canceled. Claude Sonnet 4 and Opus 4 retired June 15 and Opus 4.1 retired August 5 on the Claude API.
- **OpenAI.** GPT-5.5 launched April 23 at $5 / $30. GPT-5.6 Sol, Terra and Luna went GA July 9; on July 30 Terra was cut to $2 / $12 and Luna to $0.20 / $1.20, and Priority was renamed Fast. GPT-5.6-Cyber launched August 10. GPT-4o, GPT-4.1, GPT-4.1-mini and o4-mini left ChatGPT February 13; the Realtime API Beta was removed May 12; the Sora app shut down April 26.
- **DeepSeek.** The 75% V4 Pro discount was made permanent on May 22 ($0.435 / $0.87 from June 1), and cache-hit input was cut to 1/10 of launch price on April 26. On August 16 DeepSeek moved to peak and off-peak billing and raised V4 prices 3x to 12x depending on token type, ending its run as the unambiguous cheap option.
- **Google.** Vertex retired `gemini-3-pro-preview` on March 26; Project Mariner shut down May 4. Gemini 3.7 Flash launched August 13 at the half-price introductory rate.

---

## Cost Calculation

### Basic Cost Formula

The function below models the two things a naive `tokens × rate` calculation misses: cache splits and the whole-request long-context cliff, which now exists at Anthropic (Haiku 5.5) as well as OpenAI and xAI.

```python
# USD per 1M tokens, Standard tier, list prices as of November 6, 2026.
# lc_at = smallest prompt size that moves the WHOLE request to long-context rates.
PRICING = {
    "gpt-6-sol":         {"in": 2.00, "cached": 0.20, "out": 10.00,
                          "lc_at": 272_001, "lc_in": 4.00, "lc_cached": 0.40, "lc_out": 15.00},
    "gpt-6.1-terra":     {"in": 1.00, "cached": 0.05, "out": 5.00,
                          "lc_at": 272_001, "lc_in": 2.00, "lc_cached": 0.10, "lc_out": 7.50},
    "gpt-6-luna":        {"in": 0.10, "cached": 0.01, "out": 0.50,
                          "lc_at": 272_001, "lc_in": 0.20, "lc_cached": 0.02, "lc_out": 0.75},
    "grok-4.7":          {"in": 2.00, "cached": 0.50, "out": 6.00,
                          "lc_at": 200_000, "lc_in": 4.00, "lc_cached": 1.00, "lc_out": 12.00},
    # Haiku 5.5: 5x on every token type once the prompt (cache reads and writes included)
    # passes 100,000 tokens.
    "claude-haiku-5-5":  {"in": 0.10, "cached": 0.01, "out": 0.50,
                          "lc_at": 100_001, "lc_in": 0.50, "lc_cached": 0.05, "lc_out": 2.50},
    "claude-sonnet-5-5": {"in": 2.00, "cached": 0.10, "out": 10.00},  # flat to 1M
    "claude-opus-5-5":   {"in": 4.00, "cached": 0.20, "out": 20.00},  # flat to 1M
    # Gemini 4 Argon: the December 1, 2026 price; implicit caching only, so cached_in
    # is whatever hit volume Google reports, and cache_write should stay 0.
    "gemini-4-argon":    {"in": 4.00, "cached": 0.20, "out": 20.00},
    # Gemini 3.8 Flash: budget on the January 1, 2027 list price, not the $0.75/$3.75 intro.
    # Cache reads assumed at 0.1x of input; hourly cache storage is not modeled here.
    "gemini-3.8-flash":  {"in": 1.50, "cached": 0.15, "out": 7.50},
}

def request_cost(
    model: str,
    uncached_in: int,
    output_tokens: int,          # visible output + reasoning tokens
    cached_in: int = 0,
    cache_write: int = 0,
    write_mult: float = 1.25,    # OpenAI 1.25x (30 min) or 2.5x (12 h); Anthropic 1.25x (5 min) or 2x (1 h)
    tier_mult: float = 1.0,      # 0.5 Batch/Flex, 2.0 Fast, 6.0 OpenAI Ultrafast
    residency_mult: float = 1.0, # 1.1 for region-pinned processing
) -> float:
    p = PRICING[model]
    prompt = uncached_in + cached_in + cache_write
    long_ctx = "lc_at" in p and prompt >= p["lc_at"]

    def rate(key: str) -> float:
        return p[f"lc_{key}"] if long_ctx else p[key]

    cost = (
        uncached_in * rate("in")
        + cache_write * rate("in") * write_mult
        + cached_in * rate("cached")
        + output_tokens * rate("out")
    ) / 1_000_000
    return cost * tier_mult * residency_mult
```

**The cliff in numbers (GPT-6 Sol, 2K output):** a 270K-token prompt costs (270,000 × $2 + 2,000 × $10) / 1M = **$0.56**. A 275K-token prompt costs (275,000 × $4 + 2,000 × $15) / 1M = **$1.13**. Five thousand extra tokens doubled the bill. Claude Haiku 5.5's line is lower and steeper: a 99K-token prompt with 2K output costs (99,000 × $0.10 + 2,000 × $0.50) / 1M = **$0.0109**, and a 101K-token prompt costs (101,000 × $0.50 + 2,000 × $2.50) / 1M = **$0.0555**, about 5x for 2K more tokens. Because cached tokens count toward Haiku's threshold, caching a long prefix does not avoid the jump. On Claude Fable 5.1 a 900K-token prompt is billed at the same $10 per 1M as a 9K one; on GPT-6 Astra the same prompt bills at $20 per 1M. Long-context RAG decisions are now partly pricing decisions.

### Example Cost Calculations

**Scenario 1: RAG Chatbot**
```
Per request:
- System prompt: 500 tokens
- Retrieved context: 2,000 tokens
- User message: 100 tokens
- Response: 300 tokens

Input: 2,600 tokens, Output: 300 tokens

GPT-6 Sol or Claude Sonnet 5.5: (2600 × $2 + 300 × $10) / 1M = $0.0082 per request
GPT-6 Luna:                     (2600 × $0.10 + 300 × $0.50) / 1M = $0.00041 per request

At 10,000 requests/day on GPT-6 Sol:
Daily: $82
Monthly: $2,460
(GPT-6 Luna: $4.10/day, $123/month, if quality holds on your evals)
```

**Scenario 2: Document Summarization**
```
Per document:
- Document: 8,000 tokens
- Summary: 500 tokens

GPT-6 Sol cost: (8000 × $2 + 500 × $10) / 1M = $0.021

1,000 documents: $21
10,000 documents: $210 (Batch API: $105)
```

### Monthly Cost Projection

```python
def project_monthly_cost(
    requests_per_day: int,
    avg_input_tokens: int,
    avg_output_tokens: int,
    model: str,
    **kwargs,
) -> dict:
    per_request = request_cost(
        model, avg_input_tokens, avg_output_tokens, **kwargs
    )

    daily = per_request * requests_per_day
    monthly = daily * 30
    yearly = monthly * 12

    return {
        "per_request": per_request,
        "daily": daily,
        "monthly": monthly,
        "yearly": yearly
    }

# Example
costs = project_monthly_cost(
    requests_per_day=50000,
    avg_input_tokens=2000,
    avg_output_tokens=400,
    model="gpt-6-sol"
)
# Output: $0.008/request, $400/day, ~$12,000/month
```

---

## Cost Optimization Strategies

### Strategy 1: Model Routing

Route requests to appropriate model tiers:

```python
class ModelRouter:
    def __init__(self):
        self.classifier = load_complexity_classifier()

    def route(self, query: str, context: str) -> str:
        complexity = self.classifier.predict(query)

        if complexity < 0.3:
            return "gpt-6-luna"     # Simple queries: $0.10 / $0.50
        elif complexity < 0.7:
            return "gpt-6.1-terra"  # Medium: $1 / $5, the tier OpenAI restored in October
        else:
            return "gpt-6.1-sol"    # Complex queries: $2 / $10

    def route_with_fallback(self, query: str, context: str) -> str:
        # Try cheap model first
        response = self.try_model("gpt-6-luna", query, context)

        if self.is_quality_sufficient(response):
            return response

        # Fallback to expensive model
        return self.try_model("gpt-6.1-sol", query, context)
```

**Potential savings:** 50-70% with minimal quality impact. The tier spread is now 20x on both input and output between GPT-6 Luna and GPT-6.1 Sol, with GPT-6.1 Terra at half of Sol's price, so routing pays more than it did a year ago. The same three rungs exist on Anthropic (Haiku 5.5, Sonnet 5.5, Opus 5.5) as long as Haiku traffic stays under 100K-token prompts.

### Strategy 2: Prompt Optimization

Reduce token count without losing quality:

```python
# Before: 2,500 tokens
system_prompt = """
You are a helpful customer support assistant for Acme Corp.
You have access to our product documentation and should answer
questions accurately and helpfully. Always be polite and professional.
If you don't know something, say so rather than making things up.
Format your responses clearly with bullet points when listing items.
[... more verbose instructions ...]
"""

# After: 800 tokens
system_prompt = """
You are Acme Corp's support assistant.
Rules:
- Answer from provided context only
- Admit uncertainty
- Use bullet points for lists
- Be concise
"""

# Savings on GPT-6 Sol: 1,700 tokens × $2/1M = $0.0034 per request
# At 10K requests/day: $34/day = $1,020/month (before caching, which shrinks this further)
```

### Strategy 3: Caching

Cache responses for repeated or similar queries:

```python
class ResponseCache:
    def __init__(self, ttl_seconds: int = 3600):
        self.exact_cache = TTLCache(maxsize=10000, ttl=ttl_seconds)
        self.semantic_cache = SemanticCache(threshold=0.95)

    def get_or_generate(self, query: str, context: str) -> tuple[str, bool]:
        # Check exact cache
        cache_key = self.make_key(query, context)
        if cache_key in self.exact_cache:
            return self.exact_cache[cache_key], True  # Cache hit

        # Check semantic cache
        similar = self.semantic_cache.find_similar(query)
        if similar:
            return similar.response, True  # Semantic hit

        # Generate new response
        response = self.generate(query, context)
        self.exact_cache[cache_key] = response
        self.semantic_cache.add(query, response)

        return response, False  # Cache miss

# With 30% cache hit rate:
# Baseline: $3,000/month
# With caching: $2,100/month
# Savings: $900/month
```

### Strategy 4: Batch Processing and Service Tiers

Anything no human is waiting on belongs in a cheaper tier:

```python
# Real-time: pay full price
for query in queries:
    response = model.generate(query)

# Batch API (50% off at OpenAI, Anthropic, Google, and via OpenRouter on 70+ models):
batch_responses = model.batch_generate(queries)
# Cost: 50% of real-time pricing, results within 24 hours

# Flex tiers (OpenAI, Gemini, Bedrock): same 0.5x on the regular API, in exchange
# for slower, lower-priority processing
```

The ladder now runs from 0.5x (Batch, Flex) through Standard to 2x (OpenAI Fast, Anthropic fast mode) and 6x (OpenAI Ultrafast). A latency SLO is a cost decision: map each workload class (interactive, agent inner loop, offline) to a tier and report the blended rate.

### Strategy 5: Output Length Control

Limit response length appropriately:

```python
# Reduce unnecessary output
response = model.generate(
    prompt=prompt,
    max_tokens=300,  # Limit output (on reasoning models this budget includes thinking)
    stop=["\n\n"]    # Stop at natural break
)

# Cost impact on GPT-6 Sol:
# Before: avg 500 output tokens = $0.005 per request
# After: avg 250 output tokens = $0.0025 per request
# Savings: 50% on output costs
```

### Strategy 6: Prompt-Length Routing

Keep Claude Haiku 5.5 prompts at or under 100K (cached tokens count), OpenAI prompts at or under 272K and xAI prompts under 200K, or route long prompts to a flat-priced model (Claude Sonnet 5.5, Opus 5.5 or Fable 5.1). Trimming retrieved context by a few thousand tokens near the threshold halves the request cost on OpenAI and cuts it by 80% on Haiku 5.5; above the line, compaction, RAG, or a different model usually beats paying 2x to 5x on every token. Measure prompt length with the target model's tokenizer: Haiku 5.5 counts about 30% more tokens than Haiku 4.5 for the same text.

### Cost Optimization Summary

| Strategy | Effort | Potential Savings |
|----------|--------|-------------------|
| Model routing | Medium | 50-70% |
| **Context Caching** | Low | **90-97.5% (cached input)** |
| Prompt optimization | Low | 20-40% |
| Response caching | Medium | 20-40% |
| Batch / Flex tiers | Low | 50% (OpenAI, Anthropic, Google, OpenRouter) |
| Prompt-length routing | Low | Up to 50% on requests near OpenAI's or xAI's threshold, up to 80% near Haiku 5.5's |
| Off-peak scheduling | Low | 50% (DeepSeek) |

---

## Context Caching Economics

**The rule still holds:** if a stable prefix (system prompt, tool definitions, a shared corpus) is more than a few thousand tokens and gets reused within the TTL, cache it. What changed in 2026 is that cache terms differ by model, so the math must be done per model.

**Break-even:** caching costs a write premium of `(write_mult - 1)` once and saves `(1 - read_mult)` on every reread, so

`break-even rereads = (write_mult - 1) / (1 - read_mult)`

| Model | Input | Cache write | Cache read | TTL | Break-even rereads |
|-------|-------|-------------|------------|-----|--------------------|
| Claude Fable 5.1 | $10.00 | $12.50 (5 min) / $20.00 (1 h) | $0.25 (0.025x) | 5 min or 1 h | 0.26 / 1.03 |
| Claude Opus 5.5 | $4.00 | $5.00 / $8.00 | $0.20 (0.05x) | 5 min or 1 h | 0.26 / 1.05 |
| Claude Sonnet 5.5 | $2.00 | $2.50 / $4.00 | $0.10 (0.05x) | 5 min or 1 h | 0.26 / 1.05 |
| Claude Haiku 5.5 (prompt up to 100K) | $0.10 | $0.125 / $0.20 | $0.01 (0.1x) | 5 min or 1 h | 0.28 / 1.11 |
| GPT-6.1 Sol | $2.00 | $2.50 (30 min) / $5.00 (12 h) | $0.10 (0.05x) | 30 min or 12 h | 0.26 / 1.58 |
| GPT-6.1 Terra | $1.00 | $1.25 / $2.50 | $0.05 (0.05x) | 30 min or 12 h | 0.26 / 1.58 |
| GPT-6 Sol | $2.00 | $2.50 / $5.00 | $0.20 (0.1x) | 30 min or 12 h | 0.28 / 1.67 |
| GPT-6 Astra | $10.00 | $12.50 / $25.00 | $1.00 (0.1x) | 30 min or 12 h | 0.28 / 1.67 |
| Gemini 4 Argon | $2.00 intro ($4.00 from Dec 1) | None (implicit only) | $0.10 ($0.20 from Dec 1), 0.05x | Implicit; not controllable | No write premium, but no control over hits |
| Gemini 3.8 Flash | $0.75 intro | - | $0.075 intro ($0.15 from Jan 1, 2027) + $0.50 per 1M tokens per hour of storage intro ($1.00 from Jan 1, 2027) | Explicit caches billed per hour stored | Depends on storage time |

A break-even below 1 means **one reuse inside the TTL pays for the write**. Anthropic's 1-hour write and OpenAI's 12-hour write both need at least two rereads, so buy the long TTL only for prefixes reused across idle gaps longer than the short one.

**Where caching now changes model choice.** Take a 50-turn agent loop with a 100K-token stable prefix (tool definitions plus repository context) on a 5-minute or 30-minute TTL, counting only the prefix:

| Model | Prefix cost uncached (50 turns) | Prefix cost cached (1 write + 49 reads) |
|-------|---------------------------------|-----------------------------------------|
| Claude Fable 5.1 | $50.00 | $1.25 + 49 × $0.025 = $2.48 |
| GPT-6 Astra | $50.00 | $1.25 + 49 × $0.10 = $6.15 |
| Claude Opus 5.5 | $20.00 | $0.50 + 49 × $0.02 = $1.48 |
| Claude Sonnet 5.5 | $10.00 | $0.25 + 49 × $0.01 = $0.74 |
| GPT-6.1 Sol | $10.00 | $0.25 + 49 × $0.01 = $0.74 |
| GPT-6.1 Terra | $5.00 | $0.125 + 49 × $0.005 = $0.37 |
| Claude Haiku 5.5 (every turn over 100K) | $2.50 | $0.0625 + 49 × $0.005 = $0.31 |

Three lessons. First, with caching the prefix stops being the cost driver: cached Opus 5.5 costs about 74 cents more than cached Sonnet 5.5 over 50 turns and far less than uncached Sonnet, so the real difference between them is output and reasoning tokens. Second, identical list prices hide different cached costs: Fable 5.1 and Astra both list at $10 input, but Astra's 0.1x read makes the same cached prefix about 2.5x as expensive. Third, Haiku 5.5's cliff interacts with caching: a 100K prefix plus any user turn crosses the line, so every turn bills at the high rate even when it is almost all cache hits. Trim the same prefix to 95K and keep turns small, and the cached cost falls to about $0.06; let an agent's history grow past 100K, and Haiku stops being the cheap tier.

**Caveats that bite in production:**
- **OpenAI** (GPT-5.6 and later): minimum cacheable prompt 1,024 tokens; `prompt_cache_options.ttl` accepts `"30m"` (the default) or `"12h"` (Responses API only, since October 20), and reads cost the same for both. Earlier OpenAI models have no write charge and use `prompt_cache_retention`. The 272K long-context rate applies to cached tokens too.
- **Anthropic**: Opus 5.5's minimum cacheable prompt is 512 tokens. On Fable 5.1, Opus 5.5 and Sonnet 5.5, thinking blocks are bound to the conversation prefix: editing the system prompt, tools or earlier messages drops the block or, for accounts created on or after August 31, 2026, returns a 400. Treat history as append-only, the same discipline that keeps caches warm. The `inline-tools-2026-09-15` beta adds tools mid-conversation without losing the cache.
- **Gemini** explicit caches accrue storage per hour, so a cache that is written and then idles can cost more than it saves. Gemini 4 Argon supports implicit caching only, so explicit-cache code written for earlier Gemini models does nothing there; track its reported cached tokens instead.
- **Anthropic Haiku 5.5**: cache reads and writes count toward the 100K-token threshold, so caching lowers the per-token rate but never keeps a long prompt in the cheap tier.
- **Observe it.** OpenAI's Prompt Cache Diagnostics went GA on September 8, 2026 and Anthropic's cache diagnostics on September 23. Track hit rate as an SLO; prompt rewrites and model bumps silently zero it.

Batch discounts (50%) stack with caching at both OpenAI and Anthropic. For how these discounts map to KV reuse inside the serving stack, see [KV Cache and Context Caching](../04-inference-optimization/02-kv-cache-and-context-caching.md).

---

## Self-Hosting & GPU Cloud Arbitrage

**The Reserved vs. Serverless Tradeoff:**

| Model Size | Serverless (RunPod/Together) | Reserved (Lambda/AWS) |
|------------|-----------------------------|-----------------------|
| **Burst Capacity** | Infinite (cold starts) | Fixed |
| **Utilization** | Pay only for compute time | 24/7 fixed cost |
| **TCO Break-even**| **Cost-effective < 40% util** | **Cost-effective > 40% util** |

**Principal-level Nuance:**
"GPU Cloud Arbitrage" involves moving production workloads between providers based on **spot instance availability**. Tools like **Skypilot** automate this, saving up to 60% on self-hosting costs by following "low-demand" regions globally. MoE models cut the active compute per token, but not the memory needed to hold the weights: Llama 4 Scout (109B total) fits one H100 at INT4, Maverick (about 400B total) needs an 8x H100 node, and Xiaomi says MiMo-V2.6-Flash (309B / 15B active) fits one 8-GPU node.

**GPU prices moved the other way from API prices in 2026, and kept rising into November.** The Silicon Data rental index on November 5 put B200 at $6.70 (from $6.50 on November 1 and $5.88 on October 7, about 14% in four weeks), H200 at $3.51, H100 at $2.97 and B300 flat at $7.56 per GPU-hour; read the daily table, because each index page's FAQ shows stale "current" prices. On-demand list prices: Lambda H100 SXM $3.99 and B200 $6.69 per GPU-hour on 8-GPU instances until November 16, when its posted prices take effect at 00:00 UTC (scheduled): 8x HGX H100 $4.59 and 8x HGX B200 $7.69, about 15% more, with single-GPU H100 SXM at $4.89 and B200 SXM6 at $7.99. RunPod (updated November 4) lists H100 SXM $2.69 (community) / $3.99 (secure), B200 $5.98 / $7.99, B300 $6.94 / $8.99; its secure-cloud prices rose 14 to 18% during October. Longer terms are cheaper: on September 7, a 12-month B200 averaged $5.39 against $5.73 for 3 months.

### When Self-Hosting Makes Sense

```
API cost at scale (1M requests/month, 2,000 input + 500 output tokens each):
- Claude Opus 5.5:                $8,000 + $10,000 = $18,000/month
- Claude Sonnet 5.5 / GPT-6 Sol:  $4,000 +  $5,000 =  $9,000/month
- GPT-6.1 Terra:                  $2,000 +  $2,500 =  $4,500/month
- DeepSeek V4.1-Flash (peak):       $600 +    $600 =  $1,200/month (half off-peak)
- GPT-6 Luna or Claude Haiku 5.5:   $200 +    $250 =    $450/month (Haiku only while prompts stay under 100K)
- MiMo-V2.6-Flash (OpenRouter):     $280 +    $140 =    $420/month

Self-hosted MiMo-V2.6-Flash (309B / 15B active, MIT) on one 8x H100 node:
- GPU: 8 × $2.97/GPU-hr (November 5 index) × 730 = $17,345/month
       (8 × $3.99 Lambda on-demand = $23,302/month; $26,806 at
        Lambda's $4.59 rate from November 16)
- Engineering time: $5,000/month (0.5 FTE)
- Ops overhead: $2,000/month
- Total: ~$24,300 to $33,800/month, before N+1 redundancy
```

At this volume self-hosting loses to **every** API row, including Opus. The old rule of thumb ("self-host above roughly 500K requests per month") assumed frontier-only APIs; it does not survive a $0.10 / $0.50 closed tier and hosted open weights at $0.14 / $0.28. Break-even is now about utilization: the node has to displace roughly $24K of API spend per month (about $34K at Lambda's on-demand rate from November 16). At this request shape that is about 2.7M requests per month against Sonnet 5.5 or GPT-6 Sol, about 5.4M against GPT-6.1 Terra, but about 54M requests (roughly 134B tokens, or about 52K tokens per second around the clock) against GPT-6 Luna. Measure your own throughput on the target model before believing either number.

### Self-Hosting Cost Components

| Component | Monthly Cost | Notes |
|-----------|--------------|-------|
| GPU compute | $5K-50K | One 8x H100 node is $17K-27K; HA doubles it |
| Storage | $200-500 | Model weights, logs |
| Networking | $100-500 | Egress, load balancing |
| Engineering | $5K-15K | Partial FTE for ops |
| Monitoring | $100-500 | Observability tools |

### GPU Requirements by Model Size

| Model Size | GPU Config | Estimated Cost/Month |
|------------|------------|---------------------|
| 7B (INT4) | 1x A10G | $500-800 |
| 7B (FP16) | 1x A100 40GB | $1,500-2,500 |
| 70B (INT4) | 2x A100 80GB | $5,000-8,000 |
| 70B (FP16) | 4x A100 80GB | $10,000-15,000 |
| 405B dense (INT4), e.g. Llama 3.1 405B | 8x H100 | $17,000-27,000 |
| ~300B MoE, ~15B active (MiMo-V2.6-Flash) | 1x 8-GPU node | $17,000-27,000 (H100) |
| ~1T MoE, ~42B active (MiMo-V2.6-Pro) | 2x 8-GPU nodes (16-way tensor parallel in Xiaomi's SGLang recipe) | $35,000-54,000 (H100) |

H100 rows assume $2.97-4.59 per GPU-hour (Silicon Data index to Lambda's November 16 on-demand rate). Extreme quantization changes the small end: Ternary Bonsai 2 27B packs a Qwen3.8-27B base into 5.95 GB at 1.72 bits per weight (Apache 2.0), but needs PrismML's inference forks.

### Decision Framework

```
Choose API when:
- Volume cannot keep a GPU node busy around the clock
- No ML ops expertise
- Need highest quality (frontier models)
- Fast iteration needed

Choose self-hosting when:
- Sustained volume saturates nodes (price it against a ROUTED API bill)
- Have ML infrastructure team
- Data privacy or residency requirements an API cannot meet
- Predictable, stable workload
- Custom fine-tuning needed (OpenAI ends new fine-tuning jobs Jan 6, 2027)

Before either, check the license:
- MIT / Apache 2.0: MiMo-V2.6, GLM-5.3-Flash, DeepSeek V4.1-Flash(-1017), Ling-3.1-flash,
  Beam 1.0, Hy4 preview (preview checkpoint only), Qwen3.8-27B
- Gated: Mistral Large 4 (separate license for any MaaS business), Qwen Community 1.0
  (any MaaS or coding/office-assistant business), Muse Spark Open, Kimi K3 and
  Kimi K3.1 Pro (deal for MaaS above US$20M revenue), MiniMax-M3.1-Flash (notice for
  any commercial use, written authorization above US$20M yearly revenue), Step 5 Preview (MaaS above US$20M, plus any branded device),
  Qwen3.8-Max (MaaS/assistant above US$50M), GLM-5.3 (security review for MaaS above
  US$10B revenue), Mistral Medium 3.5 (no rights above US$20M monthly revenue)
```

---

## Total Cost of Ownership

### TCO Components

```python
def calculate_tco(scenario: dict) -> dict:
    # Direct costs
    api_or_compute = scenario["monthly_api_cost"]

    # Engineering costs
    development = scenario["dev_hours"] * scenario["engineer_rate"]
    maintenance = scenario["maintenance_hours"] * scenario["engineer_rate"]

    # Infrastructure
    vector_db = scenario["vector_db_cost"]
    monitoring = scenario["monitoring_cost"]

    # Indirect costs
    downtime_risk = scenario["expected_downtime_hours"] * scenario["revenue_per_hour"]

    monthly_tco = (
        api_or_compute +
        development / 12 +  # Amortized over year
        maintenance +
        vector_db +
        monitoring +
        downtime_risk
    )

    return {
        "monthly_tco": monthly_tco,
        "yearly_tco": monthly_tco * 12,
        "breakdown": {
            "llm": api_or_compute,
            "engineering": development / 12 + maintenance,
            "infrastructure": vector_db + monitoring,
            "risk": downtime_risk
        }
    }
```

### Example TCO Comparison

**Scenario: Customer Support Bot (50K requests/month, 3,000 input + 400 output tokens on Claude Sonnet 5.5)**

| Cost Component | API-Based | Self-Hosted |
|----------------|-----------|-------------|
| LLM costs | $500 | $2,000 (one H100) |
| Vector DB | $70 | $200 |
| Engineering (monthly) | $500 | $3,000 |
| Monitoring | $100 | $200 |
| **Monthly Total** | **$1,170** | **$5,400** |

*At this scale, API is cheaper by a wide margin.*

**Scenario: Large-Scale RAG (2M requests/month, 4,000 input + 500 output tokens)**

| Cost Component | API (all mid-tier) | API (routed: 70% GPT-6 Luna, 30% GPT-6 Sol) | Self-Hosted (one 8x H100 node) |
|----------------|--------------------|---------------------------------------------|--------------------------------|
| LLM costs | $26,000 | $8,710 | $17,100 |
| Vector DB | $500 | $500 | $1,000 |
| Engineering (monthly) | $1,000 | $1,500 | $8,000 |
| Monitoring | $200 | $200 | $500 |
| **Monthly Total** | **$27,700** | **$10,910** | **$26,600** |

*Self-hosting only beats the naive all-mid-tier API bill, and only before you add a second node for redundancy. A routed API bill beats it by more than 2x. At 2M requests per month, self-host for control, residency, licensing or fine-tuned weights, not for cost.*

---

## Interview Questions

### Q: How would you optimize costs for a high-volume RAG application?

**Strong answer:**
I would approach cost optimization in layers:

**1. Architecture optimization:**
- Model routing: Use cheap model for simple queries
- Caching: provider prefix caching on the stable prefix, plus exact and semantic response caching (30-40% of queries may be cacheable)
- Prompt compression: Minimize system prompt tokens, and keep prompts under long-context thresholds (272K on OpenAI, 100K on Claude Haiku 5.5)

**2. Model selection:**
```
Request shape: 2,600 input + 300 output tokens
Simple queries (60%): GPT-6 Luna at $0.00041/request
Complex queries (40%): GPT-6 Sol at $0.0082/request
Weighted avg: $0.0035/request (vs $0.0082 all GPT-6 Sol)
Savings: 57%
```

**3. Infrastructure:**
- Batch or Flex for embedding refreshes and offline evals (50% cheaper)
- Right-size vector DB
- Use spot instances where possible

**4. Monitoring:**
- Track cost per query type and per task, not just per token
- Track cache hit rate as an SLO
- Alert on anomalies and on spend caps before they hard-stop traffic

### Q: When would you recommend self-hosting vs using APIs?

**Strong answer:**
Decision depends on multiple factors:

**Utilization, not request count:**
- A GPU node costs $17K-23K a month whether it is busy or not, so the question is whether my sustained traffic would displace that much API spend
- Against a mid-tier API ($2 / $10) that happens at a few million requests a month; against GPT-6 Luna ($0.10 / $0.50) it takes tens of millions
- I compare against a routed API bill, never against the frontier list price

**Team capabilities:**
- No ML ops: API regardless of scale
- Strong infra team: Consider self-hosting earlier

**Quality and license:**
- Need absolute best: APIs (frontier models)
- Good enough works: Self-hosted open models, after checking the license (Mistral Large 4, Muse Spark Open, Kimi K3 and K3.1 Pro, MiniMax-M3.1-Flash, GLM-5.3, Qwen and Mistral Medium 3.5 all carry commercial gates)

**Other factors:**
- Data privacy: May force self-hosting, though ZDR-eligible APIs (Opus 5.5, Sonnet 5.5) cover many cases
- Latency control: Self-hosting gives more control
- Fine-tuning needs: OpenAI ends new fine-tuning jobs for existing customers on January 6, 2027, which pushes customization toward open weights

**My recommendation process:**
1. Start with APIs for fastest iteration
2. Build abstraction layer for model switching
3. Evaluate self-hosting when one stable workload's API spend approaches the cost of two nodes (one plus redundancy)
4. Pilot with shadow deployment before committing

### Q: Your cost model for next year uses today's per-token prices. What will it get wrong?

**Strong answer:**
Per-token list prices are the least stable input in the model. I would check eight things:

1. **Promotions expire.** Gemini 4 Argon doubles to $4 / $20 on December 1, Gemini 3.8 Flash to $1.50 / $7.50 on January 1, 2027, and OpenAI dates GPT-5.6 Sol's $4 / $20 to November 30, 2026 without stating the price after it. I budget on list prices and treat promos as upside.
2. **Long-context cliffs.** On OpenAI, a prompt one token over 272K bills the entire request at roughly 2x input and 1.5x output. On xAI the line is 200K. On Claude Haiku 5.5 it is 100K and the jump is 5x on every token, cache reads included. I model the prompt-size distribution, not the mean.
3. **Cache multipliers differ by model.** Reads are 0.1x, 0.05x or 0.025x depending on the model, OpenAI's writes cost 1.25x for 30 minutes or 2.5x for 12 hours, and Gemini 4 Argon caches only implicitly. A model swap changes cached cost even at the same list price.
4. **Speed tiers.** If product wants lower latency, Fast is 2x and Ultrafast 6x. Each workload class gets an explicit tier.
5. **Residency.** Region-pinned processing is about 1.1x at Anthropic, OpenAI, Bedrock, Google Cloud and Mistral. I apply it only to tenants that need it.
6. **Tokens per task, not tokens per call.** Models differ several-fold in tokens per task (Artificial Analysis measured GPT-6 Astra using about a third of GPT-5.6 Sol's tokens per coding task, and Step 5 Preview about twice the median output). Thinking tokens bill as output.
7. **Non-token meters.** Per-minute voice billing, managed-agent session hours, sandbox minutes, and billed refusals in some Anthropic safety categories.
8. **Forced migrations.** Retirements move traffic to new models with new prices and defaults (Sonnet 4.5 on November 30, the GPT-5 and o3 snapshots on December 11, GPT-4o and GPT-4.1 and their fine-tunes on January 14, 2027 on about three months' notice, the audio stack in early 2027), and auto-upgrades such as Foundry's gpt-4o 2024-05-13 to gpt-5.1 change cost with no code change.

The deliverable is a model with a price schedule per (model, platform) and a scenario for each date in the calendar, not a single blended rate.

---

## References

- OpenAI Pricing: https://developers.openai.com/api/docs/pricing
- OpenAI Deprecations: https://developers.openai.com/api/docs/deprecations
- OpenAI Prompt Caching: https://developers.openai.com/api/docs/guides/prompt-caching
- Anthropic Pricing: https://platform.claude.com/docs/en/about-claude/pricing
- Anthropic Model Deprecations: https://platform.claude.com/docs/en/about-claude/model-deprecations
- Google AI Pricing: https://ai.google.dev/gemini-api/docs/pricing
- xAI Pricing: https://docs.x.ai/developers/models
- DeepSeek Pricing: https://api-docs.deepseek.com/quick_start/pricing
- Mistral Pricing: https://mistral.ai/pricing/api
- Amazon Bedrock Model Lifecycle: https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html
- Microsoft Foundry Model Retirement Schedule: https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-retirement-schedule
- Lambda GPU Pricing: https://lambda.ai/pricing
- RunPod Pricing: https://www.runpod.io/pricing
- LLM Pricing Comparison: https://pricepertoken.com/

---

*Previous: [Capability Assessment](02-capability-assessment.md) | Next: [Model Selection Guide](04-model-selection-guide.md)*
