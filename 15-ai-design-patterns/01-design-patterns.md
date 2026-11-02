# AI Design Patterns

This chapter catalogs common patterns for building AI systems, similar to design patterns in software engineering. Each pattern includes when to use it, implementation guidance, and tradeoffs.

## Table of Contents

- [RAG Patterns](#rag-patterns)
- [Agent Patterns](#agent-patterns)
- [Optimization Patterns](#optimization-patterns)
- [Reliability Patterns](#reliability-patterns)
- [Cost Patterns](#cost-patterns)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## RAG Patterns

### Pattern: Naive RAG

The simplest RAG implementation:

```
Query → Embed → Search → Top K → Stuff into prompt → Generate
```

**When to use:**
- MVP and prototyping
- Simple question-answering
- When retrieval quality is sufficient

**Limitations:**
- No reranking
- No query enhancement
- May retrieve irrelevant chunks

---

### Pattern: Advanced RAG

Enhanced pipeline with multiple stages:

```
Query → Rewrite → Embed → Hybrid Search → Rerank → Filter → Generate
```

```python
class AdvancedRAG:
    async def query(self, user_query: str) -> str:
        # Step 1: Query rewriting
        enhanced_query = await self.rewrite_query(user_query)
        
        # Step 2: Hybrid retrieval
        semantic_results = await self.vector_search(enhanced_query, top_k=50)
        keyword_results = await self.bm25_search(enhanced_query, top_k=50)
        
        # Step 3: Fusion
        combined = self.reciprocal_rank_fusion(semantic_results, keyword_results)
        
        # Step 4: Reranking
        reranked = await self.rerank(enhanced_query, combined[:20])
        
        # Step 5: Generation with top results
        context = self.format_context(reranked[:5])
        return await self.generate(user_query, context)
```

**When to use:**
- Production systems
- When accuracy matters
- Complex document sets

---

### Pattern: Parent-Child Retrieval

Retrieve small chunks, return larger parent chunks:

```
Document
    └── Parent chunk (2000 tokens)
            ├── Child chunk (200 tokens) ← Retrieve on this
            ├── Child chunk (200 tokens)
            └── Child chunk (200 tokens)
```

```python
class ParentChildRetriever:
    def __init__(self, vector_store):
        self.vector_store = vector_store
    
    async def retrieve(self, query: str, top_k: int = 5) -> list[str]:
        # Search on child chunks (more precise)
        child_results = await self.vector_store.search(
            query, 
            collection="child_chunks",
            top_k=top_k * 3
        )
        
        # Get unique parent chunks
        parent_ids = set(r.metadata["parent_id"] for r in child_results)
        
        # Return parent chunks (more context)
        parents = await self.get_parents(list(parent_ids)[:top_k])
        return parents
```

**When to use:**
- Need precision in retrieval
- Need context in generation
- Document structure is hierarchical

---

### Pattern: Self-RAG

Model decides when and what to retrieve:

```python
class SelfRAG:
    async def generate(self, query: str) -> str:
        # Step 1: Decide if retrieval is needed
        needs_retrieval = await self.assess_retrieval_need(query)
        
        if needs_retrieval:
            # Step 2: Retrieve
            context = await self.retrieve(query)
            
            # Step 3: Assess relevance
            relevant_context = await self.filter_relevant(query, context)
            
            # Step 4: Generate with context
            response = await self.generate_with_context(query, relevant_context)
            
            # Step 5: Self-critique
            is_supported = await self.check_support(response, relevant_context)
            if not is_supported:
                response = await self.regenerate(query, relevant_context)
        else:
            response = await self.generate_without_context(query)
        
        return response
```

**When to use:**
- Mixed knowledge (parametric + retrieved)
- Want model to be selective
- Research and experimentation

---

### Pattern: Corrective RAG (CRAG)

Evaluate and correct retrieval quality:

```python
class CorrectiveRAG:
    async def query(self, user_query: str) -> str:
        # Initial retrieval
        docs = await self.retrieve(user_query)
        
        # Grade each document
        graded = []
        for doc in docs:
            grade = await self.grade_relevance(user_query, doc)
            graded.append((doc, grade))
        
        # Categorize results
        relevant = [d for d, g in graded if g == "relevant"]
        ambiguous = [d for d, g in graded if g == "ambiguous"]
        
        if len(relevant) >= 3:
            # Enough relevant docs
            context = relevant
        elif len(relevant) + len(ambiguous) >= 2:
            # Refine ambiguous docs
            refined = await self.refine_search(user_query, ambiguous)
            context = relevant + refined
        else:
            # Web search fallback
            web_results = await self.web_search(user_query)
            context = relevant + web_results
        
        return await self.generate(user_query, context)
```

**When to use:**
- Unreliable document corpus
- Need high accuracy
- Can afford latency for quality checks

---

## Agent Patterns

### Pattern: ReAct

Interleaved reasoning and acting:

```
Thought → Action → Observation → Thought → Action → Observation → Answer
```

See [Agent Architectures](../07-agentic-systems/01-agent-fundamentals.md) for implementation.

**When to use:**
- General-purpose agents
- Explainable decision making
- Moderate complexity tasks

---

### Pattern: Plan-and-Execute

Create a plan first, then execute steps:

```python
class PlanAndExecuteAgent:
    async def run(self, task: str) -> str:
        # Step 1: Create plan
        plan = await self.create_plan(task)
        
        # Step 2: Execute each step
        results = []
        for step in plan.steps:
            result = await self.execute_step(step, results)
            results.append(result)
            
            # Re-plan if needed
            if result.needs_replanning:
                plan = await self.replan(task, results)
        
        # Step 3: Synthesize final answer
        return await self.synthesize(task, results)
    
    async def create_plan(self, task: str) -> Plan:
        prompt = f"""
        Create a step-by-step plan to accomplish this task: {task}
        
        Return as JSON:
        {{
            "steps": [
                {{"id": 1, "description": "...", "tool": "..."}},
                ...
            ]
        }}
        """
        return await self.llm.generate(prompt)
```

**When to use:**
- Complex multi-step tasks
- Need visibility into plan
- Tasks benefit from decomposition

---

### Pattern: Critic/Verifier

One agent generates, another critiques:

```python
class CriticPattern:
    async def generate_with_critique(self, task: str, max_iterations: int = 3) -> str:
        response = await self.generator.generate(task)
        
        for i in range(max_iterations):
            # Critique the response
            critique = await self.critic.evaluate(task, response)
            
            if critique.is_acceptable:
                break
            
            # Regenerate with feedback
            response = await self.generator.regenerate(
                task, 
                previous=response, 
                feedback=critique.feedback
            )
        
        return response
```

**When to use:**
- Quality is critical
- Can afford extra latency
- Tasks have clear success criteria

---

### Pattern: Hierarchical Agents

Manager delegates to specialist workers:

```python
class ManagerAgent:
    def __init__(self):
        self.workers = {
            "research": ResearchAgent(),
            "coding": CodingAgent(),
            "writing": WritingAgent()
        }
    
    async def run(self, task: str) -> str:
        # Decompose task
        subtasks = await self.decompose(task)
        
        # Assign to workers
        results = {}
        for subtask in subtasks:
            worker = self.workers[subtask.worker_type]
            results[subtask.id] = await worker.execute(subtask)
        
        # Synthesize results
        return await self.synthesize(task, results)
```

**When to use:**
- Complex tasks spanning domains
- Different tools per subtask
- Parallelization opportunities

---

## Optimization Patterns

### Pattern: Cascading Models

Route to cheapest sufficient model:

```python
class ModelCascade:
    def __init__(self):
        # (model, input USD per 1M tokens at list price, October 2026)
        self.models = [
            ("gpt-6-luna", 0.10),         # Cheapest
            ("claude-sonnet-5-5", 2.00),  # Mid-tier
            ("claude-opus-5-5", 4.00)     # Most capable
        ]
    
    async def generate(self, query: str) -> str:
        # Classify complexity
        complexity = await self.classify_complexity(query)
        
        if complexity == "simple":
            return await self.call_model("gpt-6-luna", query)
        elif complexity == "medium":
            return await self.call_model("claude-sonnet-5-5", query)
        else:
            return await self.call_model("claude-opus-5-5", query)
```

**When to use:**
- High query volume
- Variable query complexity
- Cost optimization priority

**Routing versus cascading.** The code above routes up front on a classifier. A true cascade runs the cheap model first and escalates only when a verifier or confidence check fails, which pays when most requests are easy and failures are detectable. Two current complications: pin reasoning effort per tier, because defaults move between versions (Claude Opus 5.5 defaults to `medium`, where Opus 5 defaulted to `high`); and re-derive the tiers whenever prices move, because price is no longer a proxy for capability. On the Artificial Analysis Intelligence Index v5.0, Opus 5.5 at $4/$20 per 1M scores 58 (max effort with safeguard fallback), above the $10/$50 GPT-6 Astra at 57, and GPT-6.1 Terra at $1/$5 scores 51, above Gemini 4 Argon at 50 (high effort), which lists at $4/$20 from December 1, 2026. Route on prompt length too: Claude Haiku 5.5 bills the whole request at 5x once a prompt passes 100K tokens, so a cheap tier can become the expensive one on long inputs.

---

### Pattern: Speculative Execution

Draft with small model, verify with large:

```python
class SpeculativeExecution:
    async def generate(self, prompt: str, n_tokens: int = 5, max_tokens: int = 512) -> str:
        output = []
        
        while len(output) < max_tokens:
            # Draft with small model
            draft = await self.draft_model.generate(
                prompt + "".join(output),
                n_tokens=n_tokens
            )
            
            # Verify with large model
            verified = await self.target_model.verify(
                prompt + "".join(output),
                draft
            )
            
            # Accept verified tokens
            output.extend(verified.accepted_tokens)
            
            if verified.is_complete:
                break
        
        return "".join(output)
```

**When to use:**
- Latency-critical applications
- Have aligned draft model
- Predictable generation patterns

In production you rarely write this loop yourself; the serving engine runs it. vLLM and SGLang ship EAGLE-3 and native multi-token-prediction (MTP) heads plus DFlash block-diffusion drafters (Inco AI reports 2.7x to 3.4x throughput for DFlash 2 on Qwen3.8-27B), and vLLM also runs DeepSeek's DSpark modules, which ship bundled into the model checkpoint. Speedups shrink as batch size grows, so measure at production concurrency; vLLM says its adaptive verification keeps speculation beneficial up to a concurrency of 256. On a hosted API the equivalent lever is a paid speed tier: OpenAI Fast at 2x the standard rate and Ultrafast at 6x (on GPT-6 Astra, and on GPT-6.1 Sol since October 8, 2026), or Anthropic's fast mode for Opus 5.5 at 2x ($8/$40, out of research preview since October 15, 2026, on the Claude API, Microsoft Foundry, and Claude Models on Google Cloud). See [Speculative Decoding](../04-inference-optimization/03-speculative-decoding.md).

---

### Pattern: Caching Layers

Multi-level caching strategy:

```python
class CachingLLM:
    def __init__(self):
        self.exact_cache = ExactMatchCache()
        self.semantic_cache = SemanticCache(threshold=0.95)
    
    async def generate(self, query: str) -> str:
        # Level 1: Exact match
        cached = await self.exact_cache.get(query)
        if cached:
            return cached
        
        # Level 2: Semantic similarity
        similar = await self.semantic_cache.get_similar(query)
        if similar:
            return similar
        
        # Cache miss: Generate
        response = await self.llm.generate(query)
        
        # Store in caches
        await self.exact_cache.set(query, response)
        await self.semantic_cache.set(query, response)
        
        return response
```

**When to use:**
- Repeated similar queries
- Cost reduction priority
- Can tolerate some staleness

**Put the provider's prompt cache under both layers.** Exact and semantic caches help only when whole questions repeat. Prompt (prefix) caching helps every request that shares a stable prefix, which in an agent loop is nearly every call: Anthropic reports that Claude Code traffic runs at 324 input tokens per output token. Cache reads bill at 0.1x input on most models, 0.05x on Claude Opus 5.5, Claude Sonnet 5.5, GPT-6.1 Sol, and GPT-6.1 Terra, and 0.025x on Claude Fable 5.1, against a write premium of 1.25x (2x for Anthropic's 1-hour cache; OpenAI GPT-5.6 and later offer a 30-minute entry at 1.25x and, since October 20, 2026, a 12-hour entry at 2.5x). Gemini 4 Argon caches only implicitly, so explicit cache-control code does nothing there. Order the prompt so stable parts (system prompt, tool definitions, persistent documents) come first, and treat cache hit rate as an SLO; OpenAI and Anthropic both made cache-miss diagnostics generally available in September 2026.

---

## Reliability Patterns

### Pattern: Retry with Fallback

```python
class RetryWithFallback:
    async def generate(self, query: str) -> LLMResponse:
        providers = [
            ("openai", "gpt-6-sol"),
            ("anthropic", "claude-sonnet-5-5"),
            ("google", "gemini-3.8-flash")
        ]
        
        for provider, model in providers:
            try:
                response = await self.call(provider, model, query)
            except (RateLimitError, ServiceError, TimeoutError):
                continue
            
            # A safety refusal is an outcome, not an outage: Claude returns it
            # as HTTP 200 with stop_reason "refusal". Handle it deliberately
            # (the provider's own fallback, a user message, or a human), not by
            # shopping the request to another vendor.
            if response.stop_reason == "refusal":
                return self.handle_refusal(response)
            return response
        
        # All providers failed
        raise AllProvidersUnavailable()
```

Three things make fallback harder than the loop suggests:

- **Fallbacks must cross vendors.** Anthropic logged at least 12 major or critical incidents between August 16 and September 29, 2026, and a single OpenAI incident on September 29 degraded the API, ChatGPT, and Codex for about 5 hours 20 minutes. A fallback to a second model from the same vendor shares its failure domain.
- **A fallback is a behavior change.** Claude thinking blocks are bound to the model and conversation that produced them, so mid-conversation failover has to drop them and rebuild from the visible messages. Request shapes differ too: the newest Claude models return a 400 for forced `tool_choice`, Claude API version `2026-10-21` rejects any `temperature` on every model, and GPT-6 Astra rejects custom `temperature`. Keep conversation state in a provider-neutral format and run the eval suite against every model in the chain.
- **Refusals are not errors.** Route on `stop_reason`, not HTTP status, and since September 24, 2026 budget for them: Anthropic bills pre-output refusals in the `bio`, `frontier_llm`, and `reasoning_extraction` categories.

---

### Pattern: Circuit Breaker

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, reset_timeout: int = 60):
        self.failures = 0
        self.state = "closed"
        self.last_failure = None
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
    
    async def call(self, func, *args):
        if self.state == "open":
            if time.time() - self.last_failure > self.reset_timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError()
        
        try:
            result = await func(*args)
            self.failures = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure = time.time()
            if self.failures >= self.failure_threshold:
                self.state = "open"
            raise
```

---

### Pattern: Bulkhead

Isolate failures between components:

```python
class BulkheadExecutor:
    def __init__(self, max_concurrent: int = 10):
        self.semaphore = asyncio.Semaphore(max_concurrent)
    
    async def execute(self, func, *args):
        async with self.semaphore:
            return await func(*args)

# Separate bulkheads for different operations
rag_bulkhead = BulkheadExecutor(max_concurrent=20)
agent_bulkhead = BulkheadExecutor(max_concurrent=5)
```

---

## Cost Patterns

### Pattern: Token Budget

```python
class TokenBudget:
    def __init__(self, max_input: int, max_output: int):
        self.max_input = max_input
        self.max_output = max_output
    
    def constrain_input(self, messages: list[dict]) -> list[dict]:
        total_tokens = 0
        constrained = []
        
        for msg in reversed(messages):
            tokens = count_tokens(msg["content"])
            if total_tokens + tokens > self.max_input:
                break
            constrained.insert(0, msg)
            total_tokens += tokens
        
        return constrained
```

---

### Pattern: Cost Tracking Decorator

```python
def track_cost(model: str):
    def decorator(func):
        async def wrapper(*args, **kwargs):
            response = await func(*args, **kwargs)
            
            # Read usage from the response itself; a global token counter is
            # wrong under concurrency. Field names vary by provider.
            u = response.usage
            cost = calculate_cost(
                model,
                uncached_input=u.input_tokens,
                cache_writes=u.cache_write_tokens,  # 1.25x input (2x for a 1-hour cache)
                cache_reads=u.cache_read_tokens,    # 0.025x to 0.1x input
                output=u.output_tokens,             # includes reasoning tokens
            )
            metrics.record("llm_cost", cost, tags={"model": model})
            
            return response
        return wrapper
    return decorator

@track_cost("gpt-6-sol")
async def generate_response(query: str):
    return await llm.generate(query)
```

Price each token class separately, or the dashboard will be off by multiples: cache reads and writes bill at different rates, reasoning tokens bill as output, OpenAI bills the whole request at long-context rates once input passes 272K tokens, and service tier (Batch, Fast) and data residency multiply the total. See [Pricing and Costs](../02-model-landscape/03-pricing-and-costs.md) for the full formula.

---

## Interview Questions

### Q: Describe three RAG patterns and when to use each.

**Strong answer:**

"I will describe Naive RAG, Advanced RAG, and Parent-Child Retrieval.

**Naive RAG** is the simplest: embed query, search vectors, stuff top K into prompt, generate. I use this for MVPs and when retrieval quality is already good. It is fast to implement but has no reranking or query enhancement.

**Advanced RAG** adds multiple stages: query rewriting, hybrid search (semantic + keyword), reranking, and filtering. I use this in production when accuracy matters. The additional latency (100-200ms for reranking) is worth it for the 10-15% precision improvement.

**Parent-Child Retrieval** embeds small chunks for precise matching but returns larger parent chunks for context. I use this when documents have structure and I need both precision in retrieval and sufficient context for generation.

The pattern I choose depends on the accuracy requirements, latency budget, and document characteristics. I often start with Naive RAG to establish a baseline, then iterate to Advanced RAG."

### Q: What reliability patterns would you use for a production LLM system?

**Strong answer:**

"I implement multiple layers of reliability:

**Retry with exponential backoff** for transient failures. Rate limits and temporary errors are common with LLM APIs.

**Multi-provider fallback** so if OpenAI is having issues, I automatically route to Anthropic or Google. This requires abstracting the LLM interface, keeping conversation state in a provider-neutral format (Claude thinking blocks cannot be replayed into another model), and running my eval suite against every model in the chain so a failover does not silently change quality. I treat safety refusals separately: they come back as successful responses with a refusal stop reason, so I route on the stop reason, not the HTTP status.

**Circuit breaker** to stop hammering a failing service. After N failures, I open the circuit and route to fallback immediately, giving the primary time to recover.

**Graceful degradation** when all providers fail. Return cached responses, show fallback messages, or queue for later processing rather than erroring.

**Bulkhead isolation** to prevent one component's failures from cascading. Agent workloads get separate thread pools from RAG workloads.

**Timeouts** at every level. LLM calls can hang; I set aggressive timeouts and handle them gracefully.

The key is assuming failures will happen and designing for them rather than hoping they will not."

---

## References

- Gao et al. "Retrieval-Augmented Generation for Large Language Models: A Survey" (2024)
- Yao et al. "ReAct: Synergizing Reasoning and Acting in Language Models" (2023)
- Microsoft Patterns for AI: https://learn.microsoft.com/azure/architecture/patterns/

---

*Next: [Anti-Patterns to Avoid](02-anti-patterns.md)*
