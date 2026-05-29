# Pricing and Costs

Understanding the cost structure of LLM systems is essential for production planning. This chapter covers pricing models, cost optimization strategies, and total cost of ownership analysis.

## Table of Contents

- [Pricing Models](#pricing-models)
- [Current API Pricing](#current-api-pricing)
- [Cost Calculation](#cost-calculation)
- [Cost Optimization Strategies](#cost-optimization-strategies)
- [Context Caching Economics](#context-caching-economics)
- [Self-Hosting & GPU Cloud Arbitrage](#self-hosting-economics)
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
- Output tokens cost 2-5x more than input tokens
- Pricing varies significantly by model tier
- Some providers offer batch discounts

### Tiered Pricing

Some providers offer volume discounts:

| Tier | Monthly Spend | Discount |
|------|---------------|----------|
| Standard | $0 - $5K | 0% |
| Growth | $5K - $50K | 10-20% |
| Enterprise | $50K+ | Custom negotiation |

### Commitment-Based Pricing

Pre-purchase tokens at discounted rates:

```
Standard: $2.50 / 1M input tokens
Committed (1-year): $2.00 / 1M input tokens (20% savings)
```

---

## Current API Pricing

### May 2026 Pricing

> **Last verified: May 25, 2026.** Prices change frequently. Always re-check: [OpenAI](https://developers.openai.com/api/docs/pricing), [Anthropic](https://platform.claude.com/docs/en/about-claude/pricing), [Google](https://ai.google.dev/gemini-api/docs/pricing), [xAI](https://docs.x.ai/developers/models), [DeepSeek](https://api-docs.deepseek.com/quick_start/pricing)
>
> **Deprecations effective in 2026:** OpenAI retired GPT-4o, GPT-4.1, GPT-4.1-mini, o4-mini from ChatGPT on Feb 13, 2026; gpt-5.2-chat-latest and gpt-5.3-chat-latest deprecated May 8, 2026; Realtime API Beta removed May 12, 2026; Sora app shut down April 26, 2026 (API EOL Sep 24, 2026). Google Vertex retired `gemini-3-pro-preview` Mar 26, 2026; Project Mariner shut down May 4, 2026. Gemini 2.5 Pro/Flash deprecated June 17, 2026.
>
> **Price moves:** DeepSeek made its 75% V4 Pro discount **permanent** on May 22, 2026: from June 1, 2026 the new list price drops to 25% of the original ($0.435 / $0.87 per 1M input/output), and the cache-hit input price for all DeepSeek models was cut to 1/10 of the launch price on April 26, 2026. DeepSeek V4 Flash ($0.14 / $0.28 per 1M, 1M context) is now the cheapest frontier-class API by a wide margin.

#### OpenAI (GPT-5.x Generation)
| Model | Input / 1M | Output / 1M | Notes |
|-------|------------|-------------|-------|
| **GPT-5.5** ⭐ NEW | $5.00 | $30.00 | Released April 23, 2026. 1M context. New class of multimodal flagship. |
| **GPT-5.5 Instant** ⭐ NEW | check latest | check latest | Default in ChatGPT and `chat-latest` since May 5, 2026. 52.5% fewer hallucinations on high-stakes prompts. |
| **GPT-Realtime-2** ⭐ NEW | $32.00 (audio) | $64.00 (audio) | Released May 7, 2026. GPT-5-class realtime voice. |
| **GPT-Realtime-Translate** ⭐ NEW | (audio pricing) | (audio pricing) | 70+ input → 13 output languages. |
| **GPT-5.4 Pro** | $30.00 | $180.00 | Maximum reasoning; long-context doubles to $60/$270 |
| **GPT-5.4** | $2.50 | $15.00 | Flagship; native computer use; cached input $1.25 |
| **GPT-5.4-mini** | $0.75 | $4.50 | Best cost/performance in GPT-5 tier |
| **GPT-5.4-nano** | check latest | check latest | Smallest GPT-5.4 variant; released March 2026 |
| **GPT-4o** | $2.50 | $10.00 | Retired from ChatGPT Feb 13, 2026; API access varies |
| **GPT-4o-mini** | $0.15 | $0.60 | Legacy; check API availability |

#### Anthropic (Claude 4.x Generation)
| Model | Input / 1M | Output / 1M | Context | Notes |
|-------|------------|-------------|---------|-------|
| **Claude Opus 4.7** ⭐ NEW | $5.00 | $25.00 | 1M | Released April 16, 2026 on API, Bedrock, Vertex, Microsoft Foundry. Higher-resolution vision, improved SWE. |
| **Claude Opus 4.6** | $5.00 | $25.00 | 1M | 128K max output; adaptive thinking at standard rates |
| **Claude Sonnet 4.6** | $3.00 | $15.00 | 1M | Covers most Opus-level tasks at lower cost |
| **Claude Haiku 4.5** | $1.00 | $5.00 | 200K | Fastest Anthropic model; check latest for exact price |
| **Claude Mythos Preview** | n/a | n/a | - | Restricted to ~11 Project Glasswing partners; not generally available. |

> [!NOTE]
> **Claude 1M context at standard pricing**: Opus 4.6 and Sonnet 4.6 include the full 1M token context window at standard rates -- no premium tier for long context. Batch API offers 50% discount. Cache hits cost 10% of standard input price.

#### Google (Gemini 3.x Generation)
| Model | Input / 1M | Output / 1M | Context | Notes |
|-------|------------|-------------|---------|-------|
| **Gemini 3.1 Pro** | $2.00 | $12.00 | 1M | 200K+ context: $4.00/$18.00 |
| **Gemini 3.1 Flash** | $0.10 | $3.00 | 1M | Best price/performance; high-volume |
| **Gemini 2.5 Flash-Lite** | $0.10 | $0.40 | 1M | Deprecated June 2026 |

> [!WARNING]
> **Gemini 2.5 deprecation**: Gemini 2.5 Pro and 2.5 Flash are scheduled for deprecation on June 17, 2026. Migrate to Gemini 3.x models.

#### xAI (Grok)
| Model | Input / 1M | Output / 1M | Context | Notes |
|-------|------------|-------------|---------|-------|
| **Grok 4** | $3.00 | $15.00 | 256K | Native tool use; real-time search |
| **Grok 4.1 Fast** | $0.20 | $0.50 | 2M | High-volume, low-cost |
| **Grok 3 mini** | check latest | check latest | - | Faster, less accurate |

#### Open-Weight Models via API (May 2026)
| Model | Input / 1M | Output / 1M | Context | Provider Examples |
|-------|------------|-------------|---------|-------------------|
| **DeepSeek-V3.2** | $0.28 | $0.42 | 128K | DeepSeek API. 98% cache-hit discount. Effective rates can drop 10–30× via routing. |
| **DeepSeek V4 Pro** ⭐ NEW | $0.435 | $0.87 | 1M | DeepSeek API. 75% promotional discount made **permanent**: from June 1, 2026 the new list price is 25% of the original ($1.74 / $3.48). Cache-hit input: $0.003625/M. ~27% compute / 10% memory of V3.2 at 1M tokens. |
| **DeepSeek V4 Flash** ⭐ NEW | $0.14 | $0.28 | 1M | DeepSeek API. Cache-hit input: $0.0028/M (98% discount). 13B-active MoE. Currently the cheapest frontier-class 1M-context API. |
| **Mistral Medium 3.5** ⭐ NEW | $1.50 | check latest | 256K | Mistral API. Unified chat/reasoning/coding/vision; 77.6% SWE-Bench Verified. |
| **Kimi K2.6** ⭐ NEW | check latest | check latest | - | Moonshot API. 1T MoE / 32B active; agent swarm to 300 sub-agents. |
| **Qwen 3.6-35B-A3B** ⭐ NEW | check latest | check latest | - | Apache 2.0 weights; self-host or via API providers. |
| **Llama 4 Scout** | $0.11 | $0.34 | 10M | Together AI, Groq, Fireworks. Note: effective context degrades fast past 32K. |
| **Llama 4 Maverick** | $0.27 | $0.85 | 1M | Together AI, Groq, Fireworks. MoE-aware serving required. |
| **DeepSeek-V3** | $0.25 | $1.10 | 128K | DeepSeek API, Together AI |
| **DeepSeek-R1** | $0.55 | $2.19 | 128K | DeepSeek API |
| **Mistral Large 3** | $0.50 | $1.50 | 256K | Mistral API, AWS Bedrock |
| **Llama 3.3 70B** | ~$0.10–0.20 | ~$0.30–0.60 | 128K | Groq, Together AI |
| **Qwen2.5-Coder-32B** | ~$0.50 | ~$1.00 | 32K | Together AI |
| **Gemma 4 (31B / 26B-A4B MoE / E4B / E2B)** ⭐ NEW | self-host | self-host | 256K | Apache 2.0. 140+ languages; native vision/audio; function calling. |

#### Embedding Models (May 2026)
| Model | Cost / 1M tokens | Dimension |
|-------|------------------|-----------|
| **Cohere Embed 4** ⭐ NEW | $0.10 | 256 / 512 / 1024 / 1536 (Matryoshka) |
| **text-embedding-3-large** | $0.13 | 3072 |
| **text-embedding-3-small** | $0.02 | 1536 |
| **Voyage-3** | $0.06 | 1024 |
| **Cohere embed-v3** | $0.10 | 1024 |

> [!IMPORTANT]
> **Inference-time Compute Costs:** For models with "Extended Thinking" or reasoning modes (GPT-5.4 Pro, Claude Opus 4.6), you are charged for **internal thinking tokens** even if not shown to the user. This can increase total request cost by 2x-10x for logic-heavy tasks. Always set a `budget_tokens` cap in production.

---

## Cost Calculation

### Basic Cost Formula

```python
def calculate_request_cost(
    input_tokens: int,
    output_tokens: int,
    model: str
) -> float:
    pricing = {
        "gpt-5.4": {"input": 2.50, "output": 15.00},
        "gpt-5.4-mini": {"input": 0.75, "output": 4.50},
        "claude-sonnet-4.6": {"input": 3.00, "output": 15.00},
        "claude-opus-4.6": {"input": 5.00, "output": 25.00},
        "gemini-3.1-flash": {"input": 0.10, "output": 3.00},
    }
    
    rates = pricing[model]
    cost = (
        (input_tokens / 1_000_000) * rates["input"] +
        (output_tokens / 1_000_000) * rates["output"]
    )
    return cost
```

### Example Cost Calculations

**Scenario 1: RAG Chatbot**
```
Per request:
- System prompt: 500 tokens
- Retrieved context: 2,000 tokens
- User message: 100 tokens
- Response: 300 tokens

Input: 2,600 tokens, Output: 300 tokens

GPT-5.4 cost: (2600 × $2.50 + 300 × $15) / 1M = $0.0110 per request

At 10,000 requests/day:
Daily: $95
Monthly: $2,850
```

**Scenario 2: Document Summarization**
```
Per document:
- Document: 8,000 tokens
- Summary: 500 tokens

GPT-5.4 cost: (8000 × $2.50 + 500 × $15) / 1M = $0.0275

1,000 documents: $27.50
10,000 documents: $275
```

### Monthly Cost Projection

```python
def project_monthly_cost(
    requests_per_day: int,
    avg_input_tokens: int,
    avg_output_tokens: int,
    model: str
) -> dict:
    per_request = calculate_request_cost(
        avg_input_tokens, avg_output_tokens, model
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
    model="gpt-5.4"
)
# Output: ~$18,750/month
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
            return "gpt-5.4-mini"  # Simple queries
        elif complexity < 0.7:
            return "gpt-5.4-mini"  # Medium, try cheap first
        else:
            return "gpt-5.4"  # Complex queries

    def route_with_fallback(self, query: str, context: str) -> str:
        # Try cheap model first
        response = self.try_model("gpt-5.4-mini", query, context)

        if self.is_quality_sufficient(response):
            return response

        # Fallback to expensive model
        return self.try_model("gpt-5.4", query, context)
```

**Potential savings:** 50-70% with minimal quality impact

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

# Savings: 1,700 tokens × $2.50/1M = $0.00425 per request
# At 10K requests/day: $42.50/day = $1,275/month
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

### Strategy 4: Batch Processing

Process multiple requests together for efficiency:

```python
# Real-time: pay full price
for query in queries:
    response = model.generate(query)

# Batch API (OpenAI offers 50% discount):
batch_responses = model.batch_generate(queries)
# Cost: 50% of real-time pricing
```

### Strategy 5: Output Length Control

Limit response length appropriately:

```python
# Reduce unnecessary output
response = model.generate(
    prompt=prompt,
    max_tokens=300,  # Limit output
    stop=["\n\n"]    # Stop at natural break
)

# Cost impact:
# Before: avg 500 output tokens = $0.0075 per request (GPT-5.4)
# After: avg 250 output tokens = $0.00375 per request
# Savings: 50% on output costs
```

### Cost Optimization Summary

| Strategy | Effort | Potential Savings |
|----------|--------|-------------------|
| Model routing | Medium | 50-70% |
| **Context Caching** | Low | **60-90% (Input)** |
| Prompt optimization | Low | 20-40% |
| Response caching | Medium | 20-40% |
| Batch processing | Low | 50% (OpenAI/Anthropic) |

---

## Context Caching Economics

**The "Golden Rule" for RAG (still true in 2026).**
If you have a fixed system prompt or a shared knowledge base (prefix) larger than 10,000 tokens, **Context Caching** is mandatory.

**Break-even Analysis (Claude Sonnet 4.6):**
- **Standard Input**: $3.00 / 1M tokens
- **Cached Input**: $0.30 / 1M tokens (90% discount)
- **Cache Write Fee**: $3.75 / 1M tokens (5-min TTL at 1.25x); $6.00 (1-hour TTL at 2x)

`Break-even = (Write Fee) / (Standard Rate - Cached Rate) ≈ 1.4 requests (5-min) or 2.2 requests (1-hour)`

If your long prefix is used by **more than 2 users**, caching it is strictly cheaper than sending it raw every time. Both OpenAI and Anthropic now offer batch API discounts (50% off) that stack with caching.

---

## Self-Hosting & GPU Cloud Arbitrage

**The Reserved vs. Serverless Tradeoff:**

| Model Size | Serverless (RunPod/Together) | Reserved (Lambda/AWS) |
|------------|-----------------------------|-----------------------|
| **Burst Capacity** | Infinite (cold starts) | Fixed |
| **Utilization** | Pay only for compute time | 24/7 fixed cost |
| **TCO Break-even**| **Cost-effective < 40% util** | **Cost-effective > 40% util** |

**Principal-level Nuance:**
"GPU Cloud Arbitrage" involves moving production workloads between providers based on **spot instance availability**. Tools like **Skypilot** automate this, saving up to 60% on self-hosting costs by following "low-demand" regions globally. The rise of MoE models (Llama 4 Scout fits on a single H100, Maverick on ~2x H100, DeepSeek V4 Flash on 4x H100) has further reduced self-hosting GPU requirements compared to dense models.

### When Self-Hosting Makes Sense

```
Break-even analysis:

API cost at scale:
- 1M requests/month
- 2,500 tokens average
- GPT-5.4: ~$37,500/month
- Claude Sonnet 4.6: ~$30,000/month

Self-hosted equivalent (Llama 4 Maverick via MoE):
- 2x H100 80GB: ~$6/hour × 730 = $4,380/month
- Engineering time: $5,000/month (0.5 FTE)
- Ops overhead: $2,000/month
- Total: ~$11,380/month

Savings vs GPT-5.4: $26,120/month = 70%
Savings vs Claude Sonnet 4.6: $18,620/month = 62%
```

### Self-Hosting Cost Components

| Component | Monthly Cost | Notes |
|-----------|--------------|-------|
| GPU compute | $5K-20K | Depends on model size |
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
| 405B (INT4) | 8x H100 | $20,000-30,000 |

### Decision Framework

```
Choose API when:
- Volume < 100K requests/month
- No ML ops expertise
- Need highest quality (frontier models)
- Fast iteration needed

Choose self-hosting when:
- Volume > 500K requests/month
- Have ML infrastructure team
- Data privacy requirements
- Predictable, stable workload
- Custom fine-tuning needed
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

**Scenario: Customer Support Bot (50K requests/month)**

| Cost Component | API-Based | Self-Hosted |
|----------------|-----------|-------------|
| LLM costs | $5,000 | $3,000 |
| Vector DB | $70 | $200 |
| Engineering (monthly) | $500 | $3,000 |
| Monitoring | $100 | $200 |
| **Monthly Total** | **$5,670** | **$6,400** |

*At this scale, API is cheaper due to engineering overhead.*

**Scenario: Large-Scale RAG (2M requests/month)**

| Cost Component | API-Based | Self-Hosted |
|----------------|-----------|-------------|
| LLM costs | $50,000 | $15,000 |
| Vector DB | $500 | $1,000 |
| Engineering (monthly) | $1,000 | $8,000 |
| Monitoring | $200 | $500 |
| **Monthly Total** | **$51,700** | **$24,500** |

*At this scale, self-hosting is significantly cheaper.*

---

## Interview Questions

### Q: How would you optimize costs for a high-volume RAG application?

**Strong answer:**
I would approach cost optimization in layers:

**1. Architecture optimization:**
- Model routing: Use cheap model for simple queries
- Caching: 30-40% of queries may be cacheable
- Prompt compression: Minimize system prompt tokens

**2. Model selection:**
```
Simple queries (60%): GPT-5.4-mini at $0.003/request
Complex queries (40%): GPT-5.4 at $0.011/request
Weighted avg: $0.0062/request (vs $0.011 all GPT-5.4)
Savings: 44%
```

**3. Infrastructure:**
- Batch embedding updates (50% cheaper)
- Right-size vector DB
- Use spot instances where possible

**4. Monitoring:**
- Track cost per query type
- Alert on anomalies
- Regular cost reviews

### Q: When would you recommend self-hosting vs using APIs?

**Strong answer:**
Decision depends on multiple factors:

**Volume threshold:**
- Below 100K/month: Almost always API
- 100K-500K: Evaluate case by case
- Above 500K: Often self-hosting wins

**Team capabilities:**
- No ML ops: API regardless of scale
- Strong infra team: Consider self-hosting earlier

**Quality requirements:**
- Need absolute best: APIs (frontier models)
- Good enough works: Self-hosted open models

**Other factors:**
- Data privacy: May force self-hosting
- Latency control: Self-hosting gives more control
- Fine-tuning needs: Self-hosting enables more customization

**My recommendation process:**
1. Start with APIs for fastest iteration
2. Build abstraction layer for model switching
3. Evaluate self-hosting when spend exceeds $10K/month
4. Pilot with shadow deployment before committing

---

## References

- OpenAI Pricing: https://developers.openai.com/api/docs/pricing
- Anthropic Pricing: https://platform.claude.com/docs/en/about-claude/pricing
- Google AI Pricing: https://ai.google.dev/gemini-api/docs/pricing
- xAI Pricing: https://docs.x.ai/developers/models
- Mistral Pricing: https://docs.mistral.ai/getting-started/changelog
- Lambda Labs GPU Pricing: https://lambdalabs.com/service/gpu-cloud
- RunPod Pricing: https://www.runpod.io/pricing
- LLM Pricing Comparison: https://pricepertoken.com/

---

*Previous: [Capability Assessment](02-capability-assessment.md) | Next: [Model Selection Guide](04-model-selection-guide.md)*
