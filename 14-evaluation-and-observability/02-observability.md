# LLM Observability

Observability for LLM systems requires adapting the three pillars of logs, metrics, and traces for the unique characteristics of AI applications.

## Table of Contents

- [Why LLM Observability is Different](#why-llm-observability-is-different)
- [The Three Pillars](#the-three-pillars)
  - [Tracing LLM Pipelines](#traces)
  - [When Instrumentation Goes Silent](#when-instrumentation-goes-silent)
- [Key Metrics](#key-metrics)
- [Quality Monitoring](#quality-monitoring)
- [Cost Tracking](#cost-tracking)
- [Alerting Strategy](#alerting-strategy)
- [Observability Tools](#observability-tools)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Why LLM Observability is Different

Traditional observability focuses on:
- Request/response patterns
- Latency and throughput
- Error rates
- Resource utilization

LLM systems add:
- **Quality is a first-class metric**: A fast, available system producing bad outputs is failing
- **Non-determinism**: Same input can produce different outputs
- **Token economics**: Cost scales with usage in complex ways
- **Multi-component pipelines**: RAG has retrieval, reranking, generation steps
- **Subjective correctness**: Often no ground truth to compare against

---

## The Three Pillars

### Logging

```python
class LLMLogger:
    def log_request(
        self,
        request_id: str,
        model: str,
        messages: list[dict],
        parameters: dict
    ):
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "request_id": request_id,
            "type": "llm_request",
            "model": model,
            "parameters": parameters,
            "input_tokens": self.count_tokens(messages),
            # Hash for privacy, full content in secure store
            "content_hash": self.hash_content(messages)
        }
        self.logger.info(json.dumps(log_entry))
    
    def log_response(
        self,
        request_id: str,
        response: str,
        latency_ms: float,
        tokens: dict
    ):
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "request_id": request_id,
            "type": "llm_response",
            "latency_ms": latency_ms,
            "input_tokens": tokens["input"],
            "output_tokens": tokens["output"],
            "ttft_ms": tokens.get("ttft_ms"),
            "content_hash": self.hash_content(response)
        }
        self.logger.info(json.dumps(log_entry))
```

**What to log:**
- Request ID for correlation
- Model and parameters, including effort level, and the model ID the response says actually served the request (defaults and routing change under you)
- Token counts split into uncached input, cache reads, cache writes, output, and reasoning tokens (reasoning bills as output but is often invisible in the response text)
- Latency (TTFT and total)
- SDK and prompt versions, so a regression can be tied to a change
- Content (hashed if privacy-sensitive)

### Metrics

```python
from prometheus_client import Counter, Histogram, Gauge

# Request metrics
llm_requests_total = Counter(
    "llm_requests_total",
    "Total LLM requests",
    ["model", "status"]
)

llm_latency_seconds = Histogram(
    "llm_latency_seconds",
    "LLM request latency",
    ["model"],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0]
)

llm_ttft_seconds = Histogram(
    "llm_ttft_seconds",
    "Time to first token",
    ["model"],
    buckets=[0.05, 0.1, 0.2, 0.5, 1.0, 2.0]
)

# Token metrics
tokens_used_total = Counter(
    "tokens_used_total",
    "Total tokens consumed",
    ["model", "direction"]  # direction: uncached_input, cache_read, cache_write, output
)

# Cost metrics
llm_cost_dollars = Counter(
    "llm_cost_dollars",
    "LLM cost in dollars",
    ["model"]
)

# Quality metrics (sampled)
quality_score = Gauge(
    "llm_quality_score",
    "Sampled quality score",
    ["model", "criterion"]
)
```

### Traces

End-to-end tracing for RAG pipelines:

```python
from opentelemetry import trace

tracer = trace.get_tracer("rag_pipeline")

async def rag_query(query: str) -> str:
    with tracer.start_as_current_span("rag_query") as span:
        span.set_attribute("query", query)
        
        # Embedding step
        with tracer.start_as_current_span("embed_query") as embed_span:
            query_embedding = await embed(query)
            embed_span.set_attribute("embedding_dim", len(query_embedding))
        
        # Retrieval step
        with tracer.start_as_current_span("vector_search") as search_span:
            results = await vector_db.search(query_embedding, top_k=10)
            search_span.set_attribute("results_count", len(results))
            search_span.set_attribute("top_score", results[0].score if results else 0)
        
        # Reranking step
        with tracer.start_as_current_span("rerank") as rerank_span:
            reranked = await reranker.rerank(query, results)
            rerank_span.set_attribute("reranked_count", len(reranked))
        
        # Generation step
        with tracer.start_as_current_span("generate") as gen_span:
            response = await llm.generate(query, context=reranked[:5])
            gen_span.set_attribute("model", llm.model)
            gen_span.set_attribute("output_tokens", count_tokens(response))
        
        return response
```

For attribute names, follow the OpenTelemetry GenAI semantic conventions (`gen_ai.*`) rather than inventing your own; LLM-specific backends key their views on them, and Langfuse's v4 SDK exports only Langfuse, `gen_ai.*`, and known LLM-instrumentation spans by default.

### When Instrumentation Goes Silent

The most dangerous observability failure is a quiet dashboard that looks healthy. In August 2026 both major Python provider SDKs changed the HTTP layer that tracing and test tooling hook into: OpenAI Python SDK 3.0.0 (August 12) made `httpx2` its default client and stopped installing `httpx`, and Anthropic Python SDK 1.0.0 (August 20) moved to `httpx2` as well. Anthropic's migration guide warns that OpenTelemetry's HTTPX instrumentor, Sentry's httpx integration, `respx`, `pytest-httpx`, and `vcrpy` can silently miss SDK requests unless `httpx2.alias_httpx()` runs before anything imports `httpx`. The result is missing spans in production and unmocked live calls in tests, with no error anywhere.

Two controls catch this class of failure:

- **A telemetry canary in CI.** One test makes a real (or recorded) model call through the production client and asserts that the expected span, token counts, and cost metric were emitted. Run it on every dependency bump.
- **Daily reconciliation.** Compare tokens and dollars in your telemetry against the provider's usage or billing export. A gap of more than a few percent means something stopped reporting.

---

## Key Metrics

### Operational Metrics

| Metric | Description | Typical Alert Threshold |
|--------|-------------|------------------------|
| Request rate | Requests per second | Anomaly detection |
| Error rate | Failed requests / total | > 5% |
| Latency p50 | Median response time | > 2s |
| Latency p95 | 95th percentile | > 5s |
| Latency p99 | 99th percentile | > 10s |
| TTFT | Time to first token | > 1s |
| Token throughput | Tokens per second | < baseline |

### Quality Metrics

| Metric | Description | Collection Method |
|--------|-------------|-------------------|
| Quality score | LLM-as-judge rating | Sampled (1-5%) |
| Faithfulness | RAG answer grounded in context | Sampled |
| Relevance | Answer addresses the question | Sampled |
| User satisfaction | Thumbs up/down, ratings | User feedback |
| Task completion | Did user achieve goal? | Implicit signals |

### Cost Metrics

| Metric | Description | Granularity |
|--------|-------------|-------------|
| Cost per request | Average cost | Per model |
| Daily cost | Total daily spend | Overall + per model |
| Cost per user action | Cost to complete user goal | Per task type |
| Cost per resolved task | Spend divided by tasks that actually succeeded | Per agent and model |
| Cache hit rate | Cached input tokens / total input tokens | Per model and prompt template |
| Token efficiency | Value delivered per token | Per use case |

---

## Quality Monitoring

### Sampling Strategy

```python
class QualitySampler:
    def __init__(self, model: str, sample_rate: float = 0.05):
        self.model = model
        self.sample_rate = sample_rate
        self.judge = LLMJudge()
    
    async def maybe_evaluate(
        self,
        request_id: str,
        query: str,
        context: list[str],
        response: str
    ):
        # Sample randomly
        if random.random() > self.sample_rate:
            return
        
        # Evaluate quality
        scores = await self.judge.evaluate(
            query=query,
            context=context,
            response=response,
            criteria=["relevance", "faithfulness", "helpfulness"]
        )
        
        # Record metrics
        for criterion, score in scores.items():
            quality_score.labels(
                model=self.model,
                criterion=criterion
            ).set(score)
        
        # Store for analysis
        await self.store_evaluation(request_id, scores)
```

### Drift Detection

```python
class QualityDriftDetector:
    def __init__(self, window_size: int = 1000):
        self.window_size = window_size
        self.baseline_scores = []
        self.current_scores = []
    
    def add_score(self, score: float):
        self.current_scores.append(score)
        
        if len(self.current_scores) >= self.window_size:
            self.check_drift()
            self.current_scores = []
    
    def check_drift(self):
        if not self.baseline_scores:
            self.baseline_scores = self.current_scores.copy()
            return
        
        # Statistical test for drift
        baseline_mean = np.mean(self.baseline_scores)
        current_mean = np.mean(self.current_scores)
        
        # Simple threshold-based detection
        drift_threshold = 0.1  # 10% degradation
        if (baseline_mean - current_mean) / baseline_mean > drift_threshold:
            self.alert_drift(baseline_mean, current_mean)
    
    def alert_drift(self, baseline: float, current: float):
        alert = {
            "type": "quality_drift",
            "baseline_score": baseline,
            "current_score": current,
            "degradation_pct": (baseline - current) / baseline * 100
        }
        self.send_alert(alert)
```

---

## Cost Tracking

### Real-Time Cost Calculation

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Rate:
    """USD per 1M tokens."""
    input: float        # uncached input
    output: float       # output, including reasoning tokens
    cache_read: float   # input served from the prompt cache
    cache_write: float  # input written to the cache (Anthropic 5-minute tier; OpenAI 1.25x on the 30-minute cache, 2.5x on the 12-hour option)

class CostTracker:
    # Standard-tier list prices, November 2026. Load these from config in production:
    # promotions, long-context tiers, speed tiers, and residency premiums change monthly.
    # claude-haiku-5-5 ($0.10/$0.50, cache read $0.01) holds only up to a 100K-token prompt;
    # above that the whole request bills $0.50/$2.50, so it needs a tiered rate, not one Rate.
    PRICING = {
        "claude-opus-5-5":   Rate(input=4.00, output=20.00, cache_read=0.20, cache_write=5.00),
        "claude-sonnet-5-5": Rate(input=2.00, output=10.00, cache_read=0.10, cache_write=2.50),
        "gpt-6-sol":         Rate(input=2.00, output=10.00, cache_read=0.20, cache_write=2.50),
        "gpt-6-luna":        Rate(input=0.10, output=0.50,  cache_read=0.01, cache_write=0.125),
    }

    BUCKETS = ("uncached_input", "cache_read", "cache_write", "output")

    def track(self, model: str, usage: dict, request_id: str) -> float:
        # `usage` must already be split into the four disjoint BUCKETS. Providers
        # disagree on whether cached tokens are counted inside the input total,
        # so normalize per provider upstream or you will double-count.
        rate = self.PRICING.get(model)
        if rate is None:
            # Never price an unknown model at $0: that hides the most expensive mistakes.
            raise KeyError(f"No price configured for {model!r}")

        total_cost = (
            usage["uncached_input"] * rate.input
            + usage["cache_read"] * rate.cache_read
            + usage["cache_write"] * rate.cache_write
            + usage["output"] * rate.output
        ) / 1_000_000

        # Record metrics
        llm_cost_dollars.labels(model=model).inc(total_cost)
        for bucket in self.BUCKETS:
            tokens_used_total.labels(model=model, direction=bucket).inc(usage[bucket])

        # Log for analysis
        self.log_cost(request_id, model, usage, total_cost)

        return total_cost
```

Four billing rules break naive per-token math:

- **Cache reads dominate agent bills.** Anthropic's aggregate Claude Code usage data (published September 24, 2026, vendor-reported) puts the input-to-output token ratio at 324:1, up from 189:1 in March. At that ratio the cached-input price and your cache hit rate move cost far more than the output price. Cache-read discounts also vary by model: most Claude models bill reads at 0.1x input, Opus 5.5 and Sonnet 5.5 at 0.05x (Sonnet's dropped from 0.1x on October 7, 2026), and Fable 5.1 at 0.025x. Write multipliers vary too: OpenAI's 12-hour cache option (October 20, 2026) bills writes at 2.5x input.
- **Long-context tiers reprice the whole request.** OpenAI bills the entire request at long-context rates once input passes 272K tokens (GPT-6 Sol goes from $2/$10 to $4/$15), and xAI bills every token at 2x on Grok 4.7 prompts of 200K tokens or more. Anthropic's 4.6-and-later models are flat to 1M with one exception: Claude Haiku 5.5 bills $0.10/$0.50 up to a 100K-token prompt and $0.50/$2.50 on the whole request above it, and the prompt length counts cache reads and writes. Log prompt length per request, or a cheap model's bill will jump 5x without a traffic change.
- **Speed and residency tiers multiply the rate.** OpenAI's Fast tier is 2x (Ultrafast is 6x on GPT-6 Astra, and GPT-6.1 Sol Ultrafast is $12/$60 for all API users since October 8, 2026), and data-residency options run about 10% above list (Anthropic's `inference_geo: "us"` is 1.1x).
- **Promotions expire.** Gemini 4 Argon's $2/$10 runs through November 30, 2026 and becomes $4/$20 on December 1; Gemini 3.8 Flash's introductory $0.75/$3.75 runs through December 31, 2026 and becomes $1.50/$7.50 on January 1, 2027. Budget on list prices.

So tag every cost record with the tier, the region, and the cache buckets, and put cache hit rate on the same dashboard as spend.

### Cost Attribution

```python
class CostAttributor:
    def attribute_cost(
        self,
        request_id: str,
        user_id: str,
        team: str,
        use_case: str,
        cost: float
    ):
        # Store for billing and analysis
        attribution = {
            "request_id": request_id,
            "user_id": user_id,
            "team": team,
            "use_case": use_case,
            "cost": cost,
            "timestamp": datetime.utcnow()
        }
        
        self.store(attribution)
        
        # Update running totals
        self.update_user_total(user_id, cost)
        self.update_team_total(team, cost)
        
        # Check budgets
        if self.exceeds_budget(team):
            self.alert_budget_exceeded(team)
```

---

## Alerting Strategy

### Alert Configuration

```yaml
alerts:
  # Availability
  - name: high_error_rate
    condition: error_rate > 0.05
    for: 5m
    severity: critical
    runbook: "Check provider status, verify API keys, review recent changes, fail over to a different vendor"
    
  # Latency
  - name: high_latency_p95
    condition: latency_p95 > 10s
    for: 5m
    severity: warning
    runbook: "Check model, reduce context size, verify provider status"
    
  # Cost
  - name: cost_spike
    condition: hourly_cost > 2 * rolling_avg_hourly_cost
    for: 1h
    severity: warning
    runbook: "Check for traffic spike, review recent deployments, verify caching"

  - name: cache_hit_rate_drop
    condition: cache_hit_rate < baseline_cache_hit_rate - 0.15
    for: 1h
    severity: warning
    runbook: "Diff the prompt prefix (tool order, timestamps, injected IDs), check for a model or SDK change"

  - name: served_model_changed
    condition: served_model != pinned_model
    for: 5m
    severity: warning
    runbook: "Check provider changelog and gateway routing; re-run the regression suite before accepting"
    
  # Quality
  - name: quality_degradation
    condition: avg_quality_score < 3.5 over 1h
    for: 30m
    severity: warning
    runbook: "Review recent changes, check model performance, sample responses"
    
  # Resource
  - name: rate_limit_approaching
    condition: rate_limit_usage > 0.8
    for: 15m
    severity: warning
    runbook: "Consider model routing, implement backpressure"
```

### Alert Prioritization

| Severity | Response Time | Examples |
|----------|---------------|----------|
| Critical | < 15 min | Service down, > 50% error rate |
| High | < 1 hour | > 10% error rate, P99 > 30s |
| Warning | < 4 hours | Quality degradation, cost spike |
| Info | Next business day | Trend changes, capacity planning |

Plan the critical tier around vendor-wide outages, not single-model failures. Between August 16 and September 29, 2026, Anthropic's status feed logged at least 12 major or critical incidents, and OpenAI's September 29 incident degraded the API, ChatGPT, and Codex together for about 5 hours 20 minutes. When an outage takes out a vendor's API and its coding agent at once, a fallback chain that stays inside one vendor does not help; route across vendors, and alert on the fallback rate so you know when you are running on the backup.

---

## Observability Tools

### LLM-Specific Tools

| Tool | Focus | Best For | Watch |
|------|-------|----------|-------|
| LangSmith | Tracing and evals; trace-to-fine-tune in public beta since September 24, 2026 | LangChain and LangGraph apps | Usage-based pricing in LangChain Standard Units (1 LSU = $1) replaced per-run and uptime pricing on October 1, 2026, so re-forecast trace volume; SaaS traces with extended retention are kept at most 180 days from September 14, 2026 (self-hosted and BYOC unchanged) |
| Langfuse | Open-source, OpenTelemetry-based tracing and evals | Self-hosted, privacy | Part of ClickHouse since January 2026; Python SDK v4 changed the API and filters exported spans to LLM scopes by default |
| Weights & Biases | Experiment tracking; Weave for LLM tracing and evals | ML teams that also track training and fine-tuning runs | Part of CoreWeave since 2025 |
| Arize Phoenix | LLM tracing and evals | Production monitoring | Elastic-2.0 license (source-available, not OSI open source); Dynatrace completed its $915M acquisition of Arize on October 1, 2026 and says Phoenix remains available |
| Helicone | API proxy logging | Simple integration | Acquired by Mintlify (announced March 2026); confirm the roadmap before adopting |

**Choose for exit, not just features.** This layer is consolidating fast, so vendor risk is a selection criterion. Emit OpenTelemetry with `gen_ai.*` attributes from your own code so you can switch backends, keep the traces you need for audit or compliance in storage you control (a SaaS tracing store with a 180-day cap is not an audit log), and export eval datasets and judge prompts in a format you own.

### Integration Example: Langfuse (Python SDK v4)

The Langfuse Python SDK has been OpenTelemetry-based since the v3 rewrite (June 2025) and was at 4.16.0 at the end of September 2026. The v2 pattern in older tutorials (`langfuse.trace()`, `trace.span()`, `trace.generation()`, `span.end()`) is obsolete; use the `@observe()` decorator and `start_as_current_observation()`:

```python
from langfuse import get_client, observe, propagate_attributes

langfuse = get_client()  # credentials from the LANGFUSE_* environment variables

@observe(name="rag_query")  # root observation; captures input and return value
async def traced_rag_query(query: str, user_id: str, session_id: str) -> str:
    with propagate_attributes(user_id=user_id, session_id=session_id):
        with langfuse.start_as_current_observation(as_type="span", name="retrieve") as span:
            embedding = await embed(query)
            results = await vector_db.search(embedding, top_k=10)
            span.update(output={"count": len(results), "ids": [r.id for r in results]})

        context = results[:5]
        with langfuse.start_as_current_observation(
            as_type="generation",
            name="generate",
            model="claude-sonnet-5-5",
            input={"query": query, "context": [r.text for r in context]},
        ) as gen:
            response = await llm.generate(query, context=context)
            gen.update(
                output=response.text,
                usage_details={  # Langfuse's own keys; flat buckets must not overlap
                    "input": response.usage.input_tokens,
                    "output": response.usage.output_tokens,
                },
            )

    return response.text
```

Two v4 behaviors to know. Decorated functions record their arguments and return values, so keep embeddings, raw documents, and sensitive fields out of observed signatures or mask them. And v4 exports only Langfuse, `gen_ai.*`, and known LLM-instrumentation spans by default, so database and HTTP spans from the same process will not appear unless you widen the filter.

---

## Interview Questions

### Q: What metrics would you track for a production LLM system?

**Strong answer:**

"I organize metrics into three categories:

**Operational metrics:** These are table stakes for any service.
- Request rate and error rate
- Latency percentiles: p50, p95, p99
- Time to first token (TTFT) for streaming
- Availability

**Quality metrics:** This is what makes LLM observability unique.
- Sampled quality scores using LLM-as-judge (1-5% sample rate)
- Binary checks (policy violations, missing citations) on all traffic with a distilled or decision-model judge, which is now cheap enough to run at 100%
- For RAG: faithfulness and relevance scores
- User feedback: thumbs up/down, explicit ratings
- Task completion rate where measurable

**Cost metrics:**
- Cost per request by model
- Daily/weekly cost trends
- Cost per successful user action
- Token efficiency

I set alerts for operational issues (error rate > 5%, P95 > SLA) and quality drift (average score drops 10% from baseline). Cost alerts for spikes help catch runaway usage.

The key insight is that a fast, available LLM system producing bad outputs is still failing. Quality must be a first-class metric."

### Q: How do you detect quality degradation in production?

**Strong answer:**

"I use several approaches:

**Continuous sampling:** I evaluate 1-5% of requests using LLM-as-judge. This gives me a quality signal without evaluating everything.

**Drift detection:** I maintain a baseline quality distribution and use statistical tests to detect when current scores drift significantly. A 10% degradation triggers a warning.

**User feedback:** Thumbs up/down, explicit ratings if available. This is ground truth for user satisfaction.

**Implicit signals:** Task completion, retry rate, escalation rate, session length. If users are struggling more, quality may have dropped.

**What to do when I detect degradation:**
1. Check for recent deployments or prompt changes
2. Sample specific responses to diagnose the issue
3. Check if it is model-specific (provider issue) or universal
4. Roll back if necessary, then investigate

I also maintain a golden test set of queries with expected behaviors that I run on every deployment to catch regressions before production."

### Q: After a routine dependency upgrade, your LLM dashboards go quiet: fewer spans, lower token counts. Traffic and the provider bill are unchanged. What happened, and how do you stop it from happening again?

**Strong answer:**

"Unchanged bills with falling telemetry means the instrumentation broke, not the traffic. My first suspect is the HTTP transport. In August 2026 the OpenAI Python SDK 3.0 and Anthropic Python SDK 1.0 both moved to `httpx2`, and instrumentation that hooks `httpx` (the OpenTelemetry HTTPX instrumentor, Sentry's integration, and mocking libraries like `respx` and `vcrpy`) can silently stop seeing SDK calls unless `httpx2.alias_httpx()` runs before anything imports `httpx`. Other suspects with the same symptom: a tracing SDK major version that changed its API or its default span filter, or a sampling config that shipped with the upgrade.

To confirm, I compare a day of provider usage exports against my token metrics, then make one call in staging and check whether its span appears.

To prevent a repeat:
1. **Telemetry canary in CI**: one test makes a model call through the production client and asserts the span, token counts, and cost metric were emitted. It runs on every dependency bump.
2. **Daily reconciliation**: telemetry tokens and dollars versus the provider's billing export, alerting on a gap over a few percent.
3. **Upgrade as a set**: provider SDKs, the HTTP client, and instrumentation packages are pinned together and bumped together.

The general lesson is that observability needs its own tests. A dashboard with no errors is only good news if you have proven it would show them."

---

## References

- OpenTelemetry: https://opentelemetry.io/
- OpenTelemetry GenAI semantic conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- Langfuse: https://langfuse.com/docs
- Langfuse Python SDK v3 to v4 upgrade guide: https://langfuse.com/docs/observability/sdk/upgrade-path/python-v3-to-v4
- LangSmith: https://docs.smith.langchain.com/
- OpenAI Python SDK `httpx2` notes: https://github.com/openai/openai-python/blob/main/httpx2.md

---

*Next: [Benchmarks and Leaderboards](03-benchmarks-and-leaderboards.md)*
