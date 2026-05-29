# Model Taxonomy

This chapter provides a comprehensive guide to the model landscape as of **May 2026**, covering model families, capabilities, and selection criteria for production systems.

> **Last verified: May 25, 2026.** The model landscape evolves rapidly. Always cross-check with provider pricing pages and release notes.
>
> **May 2026 - what's new since the April refresh:** OpenAI GPT-5.5 (April 23) and GPT-5.5 Instant (May 5, default in ChatGPT); Claude Opus 4.7 (April 16, GA on Bedrock/Vertex/Foundry); Claude Mythos Preview (restricted; Project Glasswing partners only); Google Gemma 4 (April 2, Apache 2.0) and Gemini 3.2 Flash (quiet rollout May 5); DeepSeek V4 Pro and V4 Flash (April 24; 75% V4 Pro discount made **permanent** May 22, new list price $0.435/$0.87 per 1M from June 1); Moonshot Kimi K2.6 (April 20, 1T MoE / 32B active); Alibaba Qwen 3.6 Plus / 3.6-35B-A3B / 3.6 Max-Preview; Mistral Medium 3.5 (April 29, unified chat/reasoning/coding/vision); Meta Muse Spark (April 8, first closed-weight Meta model); Llama 4 Behemoth release paused through fall 2026 amid capability concerns. SWE-bench Verified leader: Claude Mythos Preview 93.9%; ARC-AGI-2 leader: GPT-5.5 at 85.0%.

## Table of Contents

- [Model Categories](#model-categories)
- [Frontier Models (May 2026)](#frontier-models)
- [Reasoning Models](#reasoning-models)
- [Open Source Models](#open-source-models)
- [Specialized Models](#specialized-models)
- [Embedding Models](#embedding-models)
- [Model Selection Framework & Semantic Routing](#model-selection-framework)
- [Sovereign AI & Data Residency](#sovereign-ai-and-data-residency)
- [Capability Comparison](#capability-comparison)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Model Categories

### By Capability Level (April 2026 Reality)

| Tier | Characteristics | Examples | Use Case |
|------|-----------------|----------|----------|
| **Frontier** | State-of-the-art reasoning, agentic mastery | Claude Opus 4.7, GPT-5.5, Gemini 3.1 Pro, Grok 4.3 | Complex reasoning, coding, production agents |
| **Fast/Efficient** | Sub-200ms, cost-optimized | Gemini 3.1 Flash, GPT-5.5-mini, Claude Haiku 4.5, DeepSeek V4 Flash | High-volume streaming, UI, real-time |
| **Battle-Tested** | Mature, widely-deployed, stable | Claude Sonnet 4.6, GPT-5.5 Instant, Gemini 3.1 Pro | Enterprise production workloads |
| **Small/Edge** | Private, edge, specialized | Llama 4 Scout, Mistral Small 4, Phi-4 | Local privacy, on-device, MoE-efficient |
| **Reasoning-Heavy** | Extended internal CoT | GPT-5.5 reasoning, DeepSeek-R1, Claude Opus 4.7 (thinking), Gemini 3.1 Pro Deep Think | Math, code debug, multi-step logic |

### By Reasoning Mode (2025–2026)

| Mode | Capability | Models | Use Case |
|------|------------|--------|----------|
| **Standard** | Fast, intuitive response | GPT-5.5-mini, Claude Sonnet 4.6 | Chat, simple extraction |
| **Extended Thinking** | Internal scratchpad CoT before output | Claude Opus 4.7, GPT-5.5 reasoning, DeepSeek-R1 | Math, code debugging, planning |
| **Hybrid** | User-controllable reasoning depth | Claude Opus 4.7, GPT-5.5 | Variable complexity tasks |

---

## Frontier Models (May 2026)

### Claude Opus 4.7 (Anthropic) - May 2026 NEW

| Attribute | Value |
|-----------|-------|
| Context Window | 1M tokens |
| Max Output | 128K tokens |
| Input Cost | $5.00 / 1M tokens (same as 4.6) |
| Output Cost | $25.00 / 1M tokens (same as 4.6) |
| Extended Thinking | Native, Adaptive mode |
| Multimodal | Text + Higher-resolution Vision |
| SWE-bench Verified (Adaptive) | 87.6% (May 13, 2026) |
| Released | April 16, 2026 (GA on API, Bedrock, Vertex, Microsoft Foundry) |

**Best for:** Autonomous coding agents (powers Claude Code), multi-file refactors, complex reasoning. Same pricing as 4.6 - straight upgrade for most workloads.
**Considerations:** Use Sonnet 4.6 for cost-sensitive workloads; Opus 4.7 mainly for tasks requiring peak coding/agentic quality.

### Claude Mythos Preview (Anthropic) - RESTRICTED ACCESS

| Attribute | Value |
|-----------|-------|
| Status | Unreleased - Project Glasswing partners only (~11 orgs: AWS, Apple, Cisco, Google, Microsoft, NVIDIA, Palo Alto, etc.) |
| Reason for restriction | Dual-use cybersecurity capabilities |
| SWE-bench Verified | 93.9% (May 13, 2026 - current SOTA) |
| Released | April 7, 2026 (restricted partner preview) |

**Best for:** N/A in production. Tracked here because it sets the public SOTA on SWE-bench Verified and signals where the frontier sits internally.

### Claude Opus 4.6 (Anthropic)

| Attribute | Value |
|-----------|-------|
| Context Window | 1M tokens |
| Max Output | 128K tokens |
| Input Cost | $5.00 / 1M tokens |
| Output Cost | $25.00 / 1M tokens |
| Extended Thinking | Native adaptive thinking (configurable budget_tokens) |
| Multimodal | Text + Vision |
| Highlights | Most capable Anthropic model; exceptional coding and reasoning |
| Released | February 2026 |

**Best for:** Most complex reasoning, autonomous software engineering, agentic workflows.
**Considerations:** Premium pricing; use Sonnet 4.6 for tasks that don't need peak capability.

### Claude Sonnet 4.6 (Anthropic)

| Attribute | Value |
|-----------|-------|
| Context Window | 1M tokens |
| Input Cost | $3.00 / 1M tokens |
| Output Cost | $15.00 / 1M tokens |
| Extended Thinking | Supported |
| Multimodal | Text + Vision |
| Highlights | Handles tasks previously requiring Opus tier; best cost/quality balance |
| Released | February 2026 |

**Best for:** Production coding agents (powers Claude Code), complex reasoning at scale.
**Considerations:** Now covers most Opus-level tasks at lower cost. Strong default for most workloads.

### GPT-5.4 (OpenAI)

| Attribute | Value |
|-----------|-------|
| Context Window | 272K tokens (standard); extended available |
| Input Cost | $2.50 / 1M tokens |
| Output Cost | $15.00 / 1M tokens |
| Multimodal | Text, Vision, native computer use |
| Highlights | Built-in computer-use capabilities; 33% fewer factual errors vs GPT-5.2; combines coding + agentic strengths |
| Released | March 2026 |

**Best for:** Agentic workflows with computer use, coding, professional tasks.
**Considerations:** Long-context pricing doubles at 272K+ tokens.

### GPT-5.4-mini (OpenAI)

| Attribute | Value |
|-----------|-------|
| Context Window | 272K tokens |
| Input Cost | $0.75 / 1M tokens |
| Output Cost | $4.50 / 1M tokens |
| Highlights | Best cost/performance for high-volume GPT-5 tier workloads |
| Released | March 2026 |

**Best for:** High-volume API calls, cost-optimized reasoning, production chatbots.

### GPT-5.4 Pro (OpenAI)

| Attribute | Value |
|-----------|-------|
| Context Window | 272K tokens |
| Input Cost | $30.00 / 1M tokens |
| Output Cost | $180.00 / 1M tokens |
| Highlights | Maximum reasoning power; premium tier for hardest tasks |
| Released | March 2026 |

**Best for:** Competition-level math, complex multi-step reasoning.
**Considerations:** Very expensive; use standard GPT-5.4 or mini for volume.

### GPT-5.5 (OpenAI) - May 2026 NEW

| Attribute | Value |
|-----------|-------|
| Context Window | 1M tokens |
| Input Cost | $5.00 / 1M tokens |
| Output Cost | $30.00 / 1M tokens |
| Multimodal | Text, Image, Audio, Video |
| ARC-AGI-2 | 85.0% (May 13, 2026 - leader) |
| Released | April 23, 2026 |

**Best for:** Highest-quality multimodal workloads; current ARC-AGI-2 leader. Pitched as "new class of intelligence for real work" - replaces GPT-5.4 for top-tier reasoning + multimodal.
**Considerations:** ~2× the input cost of GPT-5.4 ($2.50 → $5.00) and ~2× output ($15 → $30). Use GPT-5.5 Instant for chat workloads where the price isn't justified.

### GPT-5.5 Instant (OpenAI) - May 2026 NEW

| Attribute | Value |
|-----------|-------|
| Status | Default in ChatGPT and `chat-latest` in API since May 5, 2026 |
| Hallucination Reduction | 52.5% fewer on high-stakes prompts (medicine/law/finance) vs GPT-5.3 Instant |
| AIME 2025 | 81.2% (up from 65.4% on GPT-5.3 Instant) |
| Response Length | ~30% fewer words/lines than predecessor |
| Released | May 5, 2026 |

**Best for:** Default ChatGPT-equivalent workloads, instant chat, high-stakes domains where hallucination reduction matters.
**Considerations:** Replaces GPT-5.3 Instant as the chat default. GPT-5.2-chat-latest and GPT-5.3-chat-latest deprecated May 8, 2026.

### GPT-Realtime-2, Translate, Whisper (OpenAI) - May 2026 NEW

| Attribute | Value |
|-----------|-------|
| Capability | Realtime voice with GPT-5-class reasoning |
| Translate Coverage | 70+ input → 13 output languages |
| Pricing | $32 / $64 per 1M audio tokens (input/output) |
| Released | May 7, 2026 |

**Best for:** Real-time voice agents, multilingual translation, voice-first products. Realtime API Beta was removed May 12, 2026 - Realtime-2 is the supported path.

### Gemini 3.1 Pro (Google)

| Attribute | Value |
|-----------|-------|
| Context Window | 1M tokens |
| Input Cost | $2.00 / 1M tokens (standard); $4.00 (200K+) |
| Output Cost | $12.00 / 1M tokens (standard); $18.00 (200K+) |
| Multimodal | Native: Text, Vision, Audio, Video |
| Highlights | State-of-the-art Google reasoning; powerful agentic and coding capabilities |
| Released | February 2026 |

**Best for:** Complex reasoning, multimodal analysis, long-context workloads.
**Considerations:** Replaced Gemini 3 Pro Preview. Gemini 2.5 Pro/Flash deprecated June 2026.

### Gemini 3.1 Flash (Google)

| Attribute | Value |
|-----------|-------|
| Context Window | 1M tokens |
| Input Cost | $0.10 / 1M tokens |
| Output Cost | $3.00 / 1M tokens |
| Multimodal | Native: Text, Vision, Audio, Video |
| Highlights | Fastest Google model; best price/performance for high-volume |
| Released | March 2026 |

**Best for:** Real-time multimodal apps, high-volume pipelines, long-context RAG.

### Gemini 3.2 Flash (Google) - May 2026 NEW

| Attribute | Value |
|-----------|-------|
| Status | Quiet rollout in iOS Gemini app and Google AI Studio May 5, 2026 (no formal announcement yet) |
| Released | May 5, 2026 |

**Best for:** Likely successor to 3.1 Flash for high-volume workloads. Treat as preview - pricing and full capability disclosure pending official launch.

### Gemini Deep Research / Deep Research Max (Google) - May 2026 NEW

| Attribute | Value |
|-----------|-------|
| Built on | Gemini 3.1 Pro |
| Capabilities | MCP support; native chart/infographic generation; extended test-time compute; async background workflows |
| Released | April 21, 2026 |

**Best for:** Research agents, document synthesis, long-running async workflows. The MCP support makes it the first Google research-agent product with first-class tool integration.

### Gemini Robotics-ER 1.6 (Google DeepMind) - May 2026 NEW

| Attribute | Value |
|-----------|-------|
| Domain | Physical robotics, embodied reasoning |
| New capability | Reading gauges/sight glasses |
| Deployment | Boston Dynamics Spot |
| Released | April 14, 2026 |

**Best for:** Robotics applications requiring vision-language grounding for physical actions. Available via Gemini API and AI Studio.

### Grok 4 (xAI)

| Attribute | Value |
|-----------|-------|
| Context Window | 256K tokens |
| Input Cost | $3.00 / 1M tokens |
| Output Cost | $15.00 / 1M tokens |
| Highlights | Native tool use and real-time search; competitive reasoning |
| Released | July 2025 (Grok 4.20 beta: February 2026) |

**Best for:** Live web research, reasoning-heavy tasks, real-time X/web integration.
**Considerations:** Grok 4.1 Fast available at $0.20/$0.50 for high-volume.

### Model Comparison: Frontier Tier (May 2026)

| Model | Reasoning | Coding | Context | Agentic | Cost |
|-------|-----------|--------|---------|---------|------|
| Claude Mythos Preview (restricted) | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | n/a |
| Claude Opus 4.7 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | $$$$ |
| GPT-5.5 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | $$$$ |
| Claude Opus 4.6 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | $$$$ |
| GPT-5.4 | ★★★★★ | ★★★★★ | ★★★★ | ★★★★★ | $$$ |
| Claude Sonnet 4.6 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | $$$ |
| Gemini 3.1 Pro | ★★★★★ | ★★★★ | ★★★★★ | ★★★★ | $$ |
| Grok 4 | ★★★★ | ★★★★ | ★★★★ | ★★★★ | $$$ |
| GPT-5.4-mini | ★★★★ | ★★★★ | ★★★★ | ★★★ | $ |
| Gemini 3.1 Flash | ★★★ | ★★★ | ★★★★★ | ★★★ | $ |
| GPT-5.5 Instant | ★★★★ | ★★★★ | ★★★★ | ★★★★ | $$ |

### Production Heritage & Maturity

While frontier models lead on benchmarks, many enterprise systems rely on **battle-tested** models:

| Model Family | Production Since | Maturity Note |
|--------------|------------------|---------------|
| **GPT-4o** | May 2024 | Most mature ecosystem; lowest latency variance; highest rate limits. |
| **Claude 3.5 Sonnet / 3.7 Sonnet** | June 2024 | Gold standard for tool-use reliability and structured output. |
| **Gemini 2.5 Pro** | March 2025 | Proven at scale; stable long-context. Being deprecated June 2026 in favor of 3.x. |
| **o1 / o3** | Sept 2024 | Well-understood reasoning model failure modes; o3 superseded o1. |

**Why stay on "older" frontier models?**
1. **Consistency**: New models have "release-window" latency spikes and behavior shifts.
2. **Cost Efficiency**: Previous generation is often 50-80% cheaper after a new release.
3. **Guardrail Tuning**: Security and moderation layers are more refined.

---

## Open Source Models

### Llama 4 Family (Meta) -- NEW April 2026

| Model | Parameters | Context | Architecture | Notes |
|-------|------------|---------|--------------|-------|
| Llama 4 Scout | 17B active / 16 experts (MoE) | 10M | Sparse MoE | Industry-leading 10M context; fits single H100; beats Gemma 3, Gemini 2.0 Flash-Lite |
| Llama 4 Maverick | 17B active / 128 experts (MoE) | 1M | Sparse MoE | Beats GPT-4o and Gemini 2.0 Flash; comparable to DeepSeek V3 at half active params |
| Llama 4 Behemoth | ~288B active (est.) | - | Dense MoE | Still training; outperforms GPT-4.5, Gemini 2.0 Pro on STEM benchmarks |

**Strengths:**
- First Llama generation with Mixture-of-Experts architecture
- Natively multimodal from the ground up (text, image, video input)
- Open weights on Hugging Face; available via Meta AI on WhatsApp, Messenger, Instagram
- Scout's 10M token context window is industry-leading for open models

### Llama 3.x Family (Meta) -- Previous Generation

| Model | Parameters | Context | License | Notes |
|-------|------------|---------|---------|-------|
| Llama 3.3 70B | 70B | 128K | Llama 3.3 | Still widely deployed; strong general model |
| Llama 3.1 405B | 405B | 128K | Llama 3.1 | Largest dense Meta model; being superseded by Llama 4 |

**Note:** Llama 3.x remains widely used in production, but Llama 4 Scout/Maverick offer superior performance with lower active parameter counts thanks to MoE.

### DeepSeek Family

| Model | Parameters | Context | Status | Notes |
|-------|------------|---------|--------|-------|
| **DeepSeek V4 Pro** | 1.6T total / 49B active (MoE) | 1M | GA | Previewed April 24, 2026. Uses ~27% compute / 10% memory of V3.2 at 1M tokens. SWE-bench Verified 80.6%. NIST CAISI evaluation (May 2026) places it ~8 months behind US frontier (Elo ~800). Open weights on Hugging Face. **API: $0.435 / $0.87 per 1M input/output (75% discount made permanent May 22, 2026, effective June 1).** Cache-hit input $0.003625/M. |
| **DeepSeek V4 Flash** | 284B total / 13B active (MoE) | 1M | GA | Smaller-active variant for high-throughput workloads. **API: $0.14 / $0.28 per 1M (cache-hit $0.0028/M).** Cheapest frontier-class 1M-context API as of May 2026. |
| DeepSeek-V3.2 | 671B (MoE) | 128K | Frontier | General-purpose; 98% cache-hit discount ($0.28/$0.42 per 1M base). Largely superseded by V4 Flash for new builds. |
| DeepSeek-V3 | 671B (MoE, 37B active) | 128K | Frontier | GPT-4o level at a fraction of training cost; open weights. |
| DeepSeek-R1 | 671B (MoE) | 128K | Reasoning | Matches o1 on math/code; first open-source reasoning model. |
| DeepSeek-R1-Distill | 7B–70B | - | Reasoning | Distilled to smaller models; cost-efficient reasoning. |

**Key May 2026 context**: DeepSeek V4 Pro (released April 24, with the 75% promotional discount made permanent on May 22) closed the gap with US frontier models on multiple benchmarks at a fraction of the cost. At $0.435 / $0.87 per 1M, V4 Pro is roughly 10x cheaper than Claude Opus 4.7 ($5 / $25) and 5-10x cheaper than GPT-5.5 ($5 / $30) for comparable tasks. V4 Flash drops the floor further to $0.14 / $0.28 per 1M with the same 1M context window. The 98% cache-hit discount on both makes V4 the dominant choice for high-volume RAG and classification workloads where prompts are cache-friendly. DeepSeek R2 (reasoning successor to R1) remains delayed per reports about Huawei Ascend training challenges.

### Moonshot Kimi Family - May 2026 NEW

| Model | Parameters | Context | Notes |
|-------|------------|---------|-------|
| **Kimi K2.6** | 1T total / 32B active (MoE) | - | Released April 20, 2026. Modified MIT license. Native video input; Agent Swarm scaling to 300 sub-agents and 4,000 coordinated steps. Ties GPT-5.5 on SWE-Bench Pro (58.6%); SWE-bench Verified ~80.2%. |
| Kimi K2-Thinking-0905 | - | - | First model to hit 100% on AIME 2025 (reasoning variant). |

**Best for:** Long-horizon agent workloads, video understanding, open-weight agent stack alternative to closed frontier.

### Alibaba Qwen 3.x Family - May 2026 NEW

| Model | Parameters | License | Notes |
|-------|------------|---------|-------|
| **Qwen 3.6 Max-Preview** | ~1T MoE | Commercial preview | Released ~April 20–27, 2026. 262K context. Tops six coding benchmarks per Alibaba. |
| **Qwen 3.6-Plus** | - | - | Released April 2, 2026. Enhanced coding. |
| **Qwen 3.6-35B-A3B** | 35B / 3B active MoE | Apache 2.0 | Released April 16, 2026. Open-weight workhorse. |
| Qwen2.5-Coder-32B | 32B | Apache 2.0 | Previous-generation open coding leader. |
| Qwen2.5-72B | 72B | Apache 2.0 | Previous-generation multilingual leader. |
| Qwen2.5-7B | 7B | Apache 2.0 | Efficient self-hosted option. |

### Mistral Family

| Model | Parameters | Context | Notes |
|-------|------------|---------|-------|
| **Mistral Medium 3.5** | 128B dense | 256K | May 2026 NEW. Released April 29, 2026. Merges Magistral (reasoning) + Pixtral (vision) + Devstral 2 (coding) into one model. 77.6% on SWE-Bench Verified. $1.50/M input tokens. |
| **Voxtral TTS** | 4B open-weights | streaming | May 2026 NEW (March 23 release, CC BY-NC 4.0). 70ms latency, 9 languages, 3-second voice cloning. |
| Mistral Large 3 | 675B (MoE, 41B active) | 256K | Sparse MoE; parity with best open-weight models; #2 OSS non-reasoning on LMArena. |
| Mistral Small 4 | - | 256K | Hybrid instruct/reasoning/coding; released March 2026. |
| Mistral 3 (14B/8B/3B) | 3B–14B | - | Unified family: multilingual, multimodal, Apache 2.0. |
| Mixtral 8x22B | 141B (MoE) | - | Previous gen; still viable for throughput. |

### Google Gemma Family - May 2026 NEW

| Model | Parameters | Context | License | Notes |
|-------|------------|---------|---------|-------|
| **Gemma 4 (31B dense)** | 31B | 256K | Apache 2.0 | Released April 2, 2026. 140+ languages; native vision/audio; function calling. |
| **Gemma 4 (26B-A4B MoE)** | 26B / 4B active | 256K | Apache 2.0 | Sparse MoE variant. |
| **Gemma 4 E4B** | 8B | 256K | Apache 2.0 | Edge-suitable. |
| **Gemma 4 E2B** | 5.1B / 2.3B active | 256K | Apache 2.0 | Smallest variant; mobile/embedded. |

### Meta Muse Spark (Closed Weights) - May 2026 STRATEGIC SHIFT

| Attribute | Value |
|-----------|-------|
| License | **Closed weights** - first proprietary model from Meta Superintelligence Labs |
| Capabilities | Multimodal reasoning with Instant / Thinking / Contemplating modes |
| Released | April 8, 2026 |

**Strategic significance:** Meta's first non-open model since the original Llama era. Signals that frontier-quality work may require a closed-development feedback loop. Llama 4 Behemoth release was simultaneously paused through fall 2026 amid capability concerns. The open-vs-closed equilibrium is now two-tier: frontier closed lags 6–12 months ahead; open weights catch up via distillation, RL, and ecosystem iteration.

---

## Specialized Models

### Coding Mastery (May 2026)

| Model | Specialization | Why it wins |
|-------|----------------|-------------|
| **Claude Sonnet 4.6 / Opus 4.6** | Software Engineering | Powers Claude Code; top SWE-bench scores; 1M context |
| **GPT-5.4** | Agentic coding | Native computer-use; strong full-stack coding |
| **Llama 4 Maverick** | Open-source coding | Beats GPT-4o on coding benchmarks; open weights |
| **Qwen2.5-Coder-32B** | Self-hosted coding | Best price-to-performance for self-hosted IDEs |
| **DeepSeek-R1-Distill-70B** | Open reasoning+code | Best open reasoning model for coding at 70B |

### Reasoning & Math

| Model | Approach | Best For |
|-------|----------|----------|
| **GPT-5.4 Pro** | Maximum-compute reasoning | Competition math, hardest multi-step problems |
| **Claude Opus 4.6 (thinking)** | Adaptive thinking | Software planning, complex logic, agentic reasoning |
| **DeepSeek-R1** | RL-based thinking | Open-source logical inference, competitive math |
| **Grok 4 (DeepSearch)** | Web-grounded reasoning | Research tasks needing live information |

### Long Context (1M+)

| Model | Window | Recall Performance |
|-------|--------|-------------------|
| **Llama 4 Scout** | 10M | Industry-leading open-weight context window |
| **Gemini 3.1 Pro / Flash** | 1M | Best quality at 1M context; proven at scale |
| **Claude Opus 4.6 / Sonnet 4.6** | 1M | Full 1M at standard pricing; reliable recall |
| **Llama 4 Maverick** | 1M | Open-weight 1M context with MoE efficiency |

---

## Embedding Models

### API Embedding Models (May 2026)

| Model | Dimensions | Max Tokens | MTEB Score | Cost/1M |
|-------|------------|------------|------------|---------|
| OpenAI text-embedding-3-large | 3072 | 8191 | 64.6 | $0.13 |
| OpenAI text-embedding-3-small | 1536 | 8191 | 62.3 | $0.02 |
| Voyage-3 | 1024 | 32000 | 67.8 | $0.06 |
| Cohere embed-v3 | 1024 | 512 | 66.4 | $0.10 |
| Google text-embedding-004 | 768 | 2048 | 66.1 | $0.025 |

### Open Source Embedding Models

| Model | Dimensions | Max Tokens | MTEB | Notes |
|-------|------------|------------|------|-------|
| BGE-large-en-v1.5 | 1024 | 512 | 63.9 | Instruction-tuned |
| E5-mistral-7b-instruct | 4096 | 32768 | 66.6 | Strong with instructions |
| Nomic-embed-text-v1.5 | 768 | 8192 | 62.3 | Long context, open |
| GTE-Qwen2-7B | 3584 | 32K | 72.1 | State-of-the-art open embedding |

### Embedding Selection Guide

| Requirement | Recommended | Why |
|-------------|-------------|-----|
| Best quality | Voyage-3 or text-embedding-3-large | Highest MTEB |
| Cost-efficient | text-embedding-3-small | $0.02/1M |
| Self-hosted | GTE-Qwen2-7B | Best open MTEB |
| Long documents | Nomic or Voyage-3 | 8K+ context |
| Multilingual | Cohere embed-v3 | Built for multilingual |

---

## Model Selection Framework

### Decision Tree

```
What is your primary constraint?

├── Cost → Use smaller model, consider open source
│   ├── Very cost sensitive → GPT-5.4-mini, Claude Haiku 4.5, Gemini 3.1 Flash
│   └── Moderate budget → Claude Sonnet 4.6, GPT-5.4
│
├── Quality + Reasoning → Use frontier models
│   ├── Highest reasoning → GPT-5.4 Pro, Claude Opus 4.6
│   └── Coding + reasoning → Claude Sonnet 4.6 (Extended Thinking)
│
├── Latency → Use fast models
│   ├── <100ms response → Gemini 3.1 Flash, GPT-5.4-mini
│   └── <500ms response → Claude Haiku 4.5, Grok 4.1 Fast
│
├── Self-hosting → Use open models
│   ├── Maximum capability → Llama 4 Maverick, DeepSeek-V3
│   ├── Good balance → Llama 4 Scout, Llama 3.3 70B, Qwen2.5-72B
│   └── Edge/mobile → Mistral 3 3B, Phi-4
│
└── Privacy → Self-host or use on-prem
    └── Choose open models with appropriate license
```

### Semantic Routing

Static decision trees are being replaced by **Semantic Routers**:
- **How it works**: A small, fast embedding model vectorises the query. If it matches a "known easy" cluster → cheap model (Gemini 3.1 Flash, Claude Haiku 4.5, DeepSeek V4 Flash). If it hits an "agentic/logic" cluster → Claude Opus 4.7 or GPT-5.5 with reasoning.
- **Benefit**: Automates cost-optimization without hardcoded rules.
- **Implementation**: Tools like `semantic-router` (Python) or custom Weaviate/Pinecone classifiers.

---

## Sovereign AI and Data Residency

**The 2026 Regulatory Reality:**
Enterprises must comply with GDPR (EU), DPDPA (India), Saudi Arabia PDPL, and sectoral rules. "Sovereign AI" is now a product category.

| Solution | Provider | Use Case |
|----------|----------|----------|
| **Azure Government/Sovereign** | Microsoft | Dedicated infra in 40+ regions; approved for US Gov/EU NIS2 |
| **AWS Sovereign Cloud** | Amazon | Physically isolated VPCs; GDPR-safe EU regions |
| **Google Distributed Cloud** | Google | Air-gapped on-prem Gemini deployment |
| **Private Llama 4 / 3.3** | Meta (self-host) | Maximum data sovereignty; open weights (Llama 4 MoE or 3.3 dense) |
| **DeepSeek (self-host)** | DeepSeek (open) | Open weights; no data leaves your infra |
| **Mistral Large 3 (self-host)** | Mistral (Apache 2.0) | 675B MoE; open weights; strong multilingual |

**Tradeoff**: Sovereign clouds carry a **20-30% premium** over standard global regions but are mandatory for finance and government.

### Cost Comparison at Scale (May 2026)

Assume 1M requests/day, 1K input + 500 output tokens:

| Model | Input Cost/Day | Output Cost/Day | Total/Month |
|-------|----------------|-----------------|-------------|
| Claude Sonnet 4.6 | $3,000 | $7,500 | $315,000 |
| GPT-5.4 | $2,500 | $7,500 | $300,000 |
| Gemini 3.1 Pro | $2,000 | $6,000 | $240,000 |
| GPT-5.4-mini | $750 | $2,250 | $90,000 |
| Gemini 3.1 Flash | $100 | $1,500 | $48,000 |
| Self-hosted Llama 4 Scout* | - | - | ~$15,000 |
| Self-hosted Llama 3.3 70B* | - | - | ~$50,000 |

*Self-hosted Llama 4 Scout fits on a single H100; Llama 3.3 70B assumes 4x H100 GPUs

---

## Capability Comparison

### Benchmark Performance (May 2026)

| Model | MMLU | HumanEval | SWE-bench Verified | Notes |
|-------|------|-----------|--------------------|-------|
| **Claude Opus 4.6** | - | - | - | Top-tier across reasoning and coding; specific scores check latest |
| **GPT-5.4** | - | - | - | 33% fewer factual errors vs GPT-5.2; strong coding + agentic |
| **Claude Sonnet 4.6** | - | - | - | Approaches Opus-level on many tasks |
| **Gemini 3.1 Pro** | - | - | - | State-of-the-art Google reasoning |
| **Grok 4** | - | - | - | Competitive reasoning; real-time web integration |
| **Llama 4 Maverick** | - | - | - | Beats GPT-4o, Gemini 2.0 Flash on reported benchmarks |
| **DeepSeek-R1** | 90.8 | 92.6 | 49.2% | First open-source reasoning model; math/code strong |

*Source: Respective technical reports and LMSYS Chatbot Arena / LMArena, April 2026. Benchmark scores for newest models (Opus 4.6, GPT-5.4, Gemini 3.1) are evolving rapidly -- always verify with current leaderboards.*

### Task-Specific Recommendations (May 2026)

| Task | Recommended Models | Why |
|------|--------------------|-----|
| **Autonomous Coding Agent** | Claude Sonnet 4.6 / Opus 4.6 | Powers Claude Code; 1M context; top tool reliability |
| **Complex Reasoning** | GPT-5.4 Pro, Claude Opus 4.6 (thinking), DeepSeek-R1 | Maximum reasoning power |
| **Agentic Computer Use** | GPT-5.4 | First general-purpose model with native computer-use capabilities |
| **High-Volume API** | Gemini 3.1 Flash, GPT-5.4-mini | Lowest cost per token in class |
| **Long Context RAG** | Gemini 3.1 Pro/Flash (1M), Claude Sonnet 4.6 (1M) | Verified long-range recall |
| **Ultra-Long Context** | Llama 4 Scout (10M) | Industry-leading 10M context; open weights |
| **Multimodal Real-time** | Gemini 3.1 Flash | Real-time audio/video/text native |
| **Private Production** | Llama 4 Maverick, Llama 3.3 70B, Qwen2.5-72B | High capability with local control |
| **Open-source Coding** | Llama 4 Maverick, Qwen2.5-Coder-32B | Open weights, strong coding benchmarks |
| **Creative/Chat** | GPT-5.4 | Strong conversation quality and instruction following |

---

## Interview Questions

### Q: How would you select a model for a production RAG system?

**Strong answer:**
I evaluate across these dimensions:

**1. Quality requirements:**
- Test on representative queries from the actual domain
- Measure answer correctness, hallucination rate, citation accuracy

**2. Cost analysis:**
```
Monthly cost = requests/day × 30 × avg_tokens × rate
```
Always calculate for top 2-3 candidates.

**3. Latency requirements:**
- If <200ms TTFT needed: Gemini 3.1 Flash, Claude Haiku 4.5, GPT-5.4-mini
- If quality is paramount: Accept 2-3s with Claude Opus 4.6 or GPT-5.4

**4. Operational requirements:**
- Self-hosting: Llama 4 Scout/Maverick, DeepSeek-V3
- Compliance / data residency: Azure Sovereign or self-hosted

**5. Practical selection:**
- Start with Claude Sonnet 4.6 or GPT-5.4 for prototyping
- A/B test Gemini 3.1 Flash for 80% of queries (cost)
- Keep frontier on hard queries via semantic routing

### Q: Explain the tradeoffs between proprietary and open source models.

**Strong answer:**
| Factor | Proprietary (OpenAI, Anthropic) | Open Source (Llama, DeepSeek) |
|--------|--------------------------------|-----------------------------|
| Quality | Generally higher (slightly) | Catching up rapidly |
| Cost | Per-token pricing | Compute + ops |
| Control | Limited | Full |
| Privacy | Data goes to provider | Stays on-prem |
| Updates | Automatic | Manual |
| Customization | Limited fine-tuning | Full fine-tuning |
| Ops overhead | None | Significant |

**Key insight (2026)**: DeepSeek-V3/R1 and now Llama 4 have changed this conversation -- open models match or beat GPT-4o on many benchmarks. With Llama 4 Maverick matching DeepSeek V3 on reasoning at half the active parameters, the gap is narrower than ever.

### Q: What is the difference between GPT-5.4 Pro and Claude Opus 4.6's Extended Thinking?

**Strong answer:**
Both use internal chain-of-thought, but the mechanics differ:

- **GPT-5.4 Pro**: OpenAI's maximum-compute reasoning tier ($30/$180 per 1M tokens). Allocates high compute to reasoning. Internal thoughts are not exposed. Successor to the o3 line.
- **Claude Opus 4.6 Adaptive Thinking**: Returns thinking tokens in a separate `<thinking>` block. Configurable `budget_tokens`. You can inspect the reasoning chain for debugging. Full 1M context with 128K max output.

**Production choice**: For debugging and trust-building, Claude's visible thinking is more transparent. For maximum raw reasoning power on math/competition tasks, GPT-5.4 Pro leads. For cost-effective reasoning, Claude Sonnet 4.6 or GPT-5.4-mini are strong choices.

---

## References

- Anthropic: https://platform.claude.com/docs/en/about-claude/models/overview
- OpenAI Platform: https://developers.openai.com/api/docs/models
- Google AI: https://ai.google.dev/gemini-api/docs/models
- Meta Llama: https://www.llama.com/
- DeepSeek: https://api-docs.deepseek.com/
- xAI Grok: https://docs.x.ai/developers/models
- Mistral AI: https://docs.mistral.ai/models/
- LMArena Leaderboard: https://lmarena.ai/
- Hugging Face Open LLM Leaderboard: https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard

---

*Next: [Capability Assessment](02-capability-assessment.md)*
