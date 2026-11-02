# Production RAG at Scale

Production RAG is no longer a weekend project. It is a distributed system with retrieval pipelines, caching layers, routing logic, self-correction loops, multi-tenant isolation, and cost controls, all operating under strict latency SLAs. When RAG fails in production, the cause is usually upstream of the model: the right passage was never chunked cleanly, indexed, or retrieved, so the generator never saw it. That is why the enterprise deployments that succeed treat the knowledge source (not the model) as the primary investment.

## Table of Contents

- [RAG vs Long Context](#rag-vs-long-context)
- [Query Routing and Classification](#query-routing-and-classification)
- [Semantic Caching for RAG](#semantic-caching-for-rag)
- [Multi-Index Strategies](#multi-index-strategies)
- [RAG Pipeline Optimization](#rag-pipeline-optimization)
- [Corrective RAG: Self-Checking Retrieval](#corrective-rag-self-checking-retrieval)
- [Adaptive Retrieval](#adaptive-retrieval)
- [Cost Optimization Patterns](#cost-optimization-patterns)
- [Failure Modes and Debugging](#failure-modes-and-debugging)
- [Monitoring and Alerting](#monitoring-and-alerting)
- [Scaling to Millions of Documents](#scaling-to-millions-of-documents)
- [Multi-Tenant RAG Isolation](#multi-tenant-rag-isolation)
- [Real-World Architecture Examples](#real-world-architecture-examples)
- [System Design Interview Angle](#system-design-interview-angle)
- [References](#references)

---

## RAG vs Long Context

With most frontier families supporting 1M-token context windows (Claude Opus 5.5 and Sonnet 5.5, GPT-6 Sol and Astra at 1.05M, Gemini 3.8 Flash, Muse Spark 1.3, and open models such as Xiaomi MiMo-V2.6 and Llama 4 Maverick), the question is no longer "RAG or long context?" but "When does each win?"

### The Decision Matrix

```
                    Small Corpus           Large Corpus
                    (<100K tokens)         (>1M tokens)
                 +---------------------+---------------------+
  Static Data    |  Long Context Wins  |  RAG Required       |
  (rarely        |  - Stuff it all in  |  - Can't fit in     |
   changes)      |  - Simpler arch     |    context window   |
                 |  - No index needed  |  - Index + retrieve |
                 +---------------------+---------------------+
  Dynamic Data   |  Hybrid Approach    |  RAG Required       |
  (updates       |  - Cache context    |  - Incremental      |
   frequently)   |  - Invalidate on    |    indexing          |
                 |    change           |  - Real-time updates |
                 +---------------------+---------------------+
  Multi-User     |  RAG Preferred      |  RAG Required       |
  (per-user      |  - Personalized     |  - Tenant isolation  |
   data)         |    retrieval        |  - Access control    |
                 +---------------------+---------------------+
```

### Head-to-Head Comparison

| Dimension | RAG | Long Context (1M tokens) |
|-----------|-----|--------------------------|
| **Avg Query Cost** | ~$0.001-0.015 (5K context tokens, GPT-6 Luna or Claude Haiku 5.5 to Claude Sonnet 5.5) | ~$0.10-0.25 input with a warm Claude cache (Sonnet 5.5 to Fable 5.1); $2-4 uncached |
| **Avg Latency (p50)** | ~1s | ~30-45s |
| **Precision on Specific Facts** | High (targeted retrieval) | Degrades in middle |
| **Cross-Document Synthesis** | Weak (limited context) | Strong (sees everything) |
| **Corpus Size Limit** | Unlimited | ~1M tokens |
| **Data Freshness** | Minutes (incremental index) | Requires full reload; any edit invalidates the cache after it |
| **Cost per 1M queries** | ~$1K-15K | ~$200K+ with a warm cache; $2M-4M uncached |

### The "Lost in the Middle" Problem

LLMs do not attend uniformly across their context window. Information positioned in the middle of a long context sees 30%+ accuracy degradation compared to information at the beginning or end. RAG sidesteps this entirely by placing only the most relevant chunks into a short, focused context.

### The Cache-Read Break-Even

Cheaper cache reads moved the line for **stable** corpora. On the same model, a corpus kept warm in the provider's prompt cache costs less input per query than RAG when:

```
corpus_tokens x cache_read_multiplier  <  retrieved_context_tokens
```

| Model (list prices, Nov 2, 2026) | Cache read | Corpus that costs the same as a 5K-token RAG context |
|----------------------------------|------------|------------------------------------------------------|
| Claude Fable 5.1 | 0.025x ($0.25 per 1M) | ~200K tokens |
| Claude Opus 5.5 | 0.05x ($0.20 per 1M) | ~100K tokens |
| Claude Sonnet 5.5 | 0.05x ($0.10 per 1M, since Oct 7) | ~100K tokens |
| GPT-6.1 Sol | 0.05x ($0.10 per 1M) | ~100K tokens |
| GPT-6.1 Terra | 0.05x ($0.05 per 1M) | ~100K tokens |
| Most other Claude and OpenAI models, including Claude Haiku 5.5 | 0.1x | ~50K tokens |

What the formula leaves out:
- **Cache writes and TTLs.** Claude charges 1.25x for 5-minute writes and 2x for 1-hour writes (Opus 5.5: $5 and $8 per 1M); OpenAI charges 1.25x on GPT-5.6 and later for its 30-minute cache and 2.5x for the 12-hour option added October 20, 2026. A corpus queried less often than the TTL pays the write again and again; the 12-hour write pays for itself only if the corpus is reread across the hours the shorter TTL would have expired. Gemini 4 Argon has no explicit caching at all, only implicit. Gemini's explicit caches read at 0.1x on Gemini 3.8 Flash but also bill storage by the hour ($0.50 per 1M tokens per hour through December 31, 2026, $1.00 from January 1, 2027), so a warm 200K-token corpus costs $0.10 to $0.20 an hour before a single read.
- **Long-context surcharges.** OpenAI bills the whole request at long-context rates once input passes 272K tokens, and xAI doubles all tokens at 200K and above. Anthropic prices flat to 1M on Claude 4.6 and later except Claude Haiku 5.5, which bills the whole request at 5x once the prompt passes 100K tokens, counting cache reads and writes. A cached corpus above 100K on Haiku 5.5 therefore pays the high tier on every query, which can flip the break-even back to retrieval.
- **Quality and latency.** A cheap cache read does not fix lost-in-the-middle or the prefill time of a very long prompt, and it only helps if the corpus fits.

Recompute this per model instead of quoting fixed thresholds. The practical change is at the small end: a 100K-200K-token handbook or policy set that rarely changes is now often cheaper and simpler as a cached prefix than as an index.

### Best Practice: The Hybrid Pattern

The winning architecture combines both: use RAG to retrieve the top candidates from a large corpus, then load those candidates into a long context window for cross-document reasoning.

```
  User Query
      |
      v
+------------------+     +-------------------+
|  RAG Retrieval   |---->|  Long Context     |
|  (Find top 20    |     |  Synthesis        |
|   from 10M docs) |     |  (Reason across   |
+------------------+     |   20 docs deeply) |
                          +-------------------+
                                  |
                                  v
                          Final Answer with
                          Cross-Doc Citations
```

**Rule of Thumb**: If your corpus fits in context AND you can afford the latency AND you can afford the cost, use long context. Otherwise, use RAG. For most production systems with cost and latency constraints, RAG remains the correct default.

### A Third Option: Compile the Knowledge Once

A newer product category precomputes structured knowledge at ingestion instead of retrieving chunks per query. The clearest example is **Pinecone Nexus** (GA August 6, 2026): a subject-matter expert describes the work as a Manifest (entities, relationships, answer shapes); Nexus compiles source documents once into summaries, structured extracts and entity-relationship graphs, and agents query them in one call through the KnowQL language. The data plane runs in the customer's AWS, Google Cloud or Azure account. Pinecone reports (vendor-run) that on tau-Knowledge, GPT-5.5 with Nexus scored 47.4% versus 46.4% without, at 77% lower cost per task and about 35% fewer model calls.

| | Retrieve at query time (RAG) | Compile once |
|---|---|---|
| **Where cost lands** | Every query | Ingestion and recompilation |
| **Where errors land** | Retrieval misses, visible per query | Compilation errors, baked into every answer until recompiled |
| **Handles unanticipated questions** | Yes | Only within the shapes the Manifest anticipated |
| **Corpus drift** | Incremental re-index | Recompile affected artifacts |

Compile-once fits stable, high-volume domains with recurring question shapes (policies, case law, product catalogs). Keep query-time retrieval for long-tail, exploratory questions, and keep it as the fallback even where you compile.

---

## Query Routing and Classification

Not every query needs retrieval. A production system classifies incoming queries and routes them to the optimal handling path.

### The Four-Path Router

```
                         User Query
                             |
                             v
                    +------------------+
                    |  Query Classifier |
                    |  (LLM or trained  |
                    |   classifier)     |
                    +--------+---------+
                             |
            +--------+-------+-------+--------+
            |        |               |        |
            v        v               v        v
        +------+ +--------+    +--------+ +--------+
        |Direct| |Simple  |    |Complex | |Agentic |
        | LLM  | |  RAG   |    |  RAG   | |  RAG   |
        +------+ +--------+    +--------+ +--------+
        "What    "What is      "Compare   "Analyze
        is 2+2?" our refund    Q3 vs Q4   all legal
                  policy?"     revenue    risks in
                               trends"    these 50
                                          contracts"
```

### Classification Signals

| Signal | Direct LLM | Simple RAG | Complex RAG | Agentic RAG |
|--------|-----------|------------|-------------|-------------|
| **Requires private data** | No | Yes | Yes | Yes |
| **Single-hop answer** | Yes | Yes | No | No |
| **Needs multiple sources** | No | No | Yes | Yes |
| **Requires reasoning chain** | No | No | Maybe | Yes |
| **Time-sensitive data** | No | Maybe | Maybe | Yes |

### Implementation: Lightweight Router

```python
class QueryRouter:
    """Routes queries to the optimal retrieval strategy."""

    def __init__(self, classifier_model: str = "gpt-6-luna"):
        self.classifier = classifier_model
        self.route_counts = Counter()  # for monitoring

    async def classify(self, query: str, user_context: dict) -> str:
        # Step 1: Rule-based fast path
        if self._is_trivial(query):
            return "direct_llm"

        # Step 2: Check if query references private/org data
        if not self._needs_retrieval(query, user_context):
            return "direct_llm"

        # Step 3: LLM-based complexity classification
        complexity = await self._assess_complexity(query)

        if complexity == "simple":
            return "simple_rag"
        elif complexity == "multi_hop":
            return "complex_rag"
        else:
            return "agentic_rag"

    def _is_trivial(self, query: str) -> bool:
        """Fast regex/keyword check for trivial queries."""
        trivial_patterns = [
            r"^(what is|define|explain)\s+\w+$",
            r"^(hi|hello|thanks|bye)",
        ]
        return any(re.match(p, query.lower()) for p in trivial_patterns)

    async def _assess_complexity(self, query: str) -> str:
        """Use a small, fast model to classify complexity."""
        prompt = f"""Classify this query's retrieval complexity:
        - "simple": needs one document lookup
        - "multi_hop": needs 2-3 lookups, comparison, or synthesis
        - "agentic": needs planning, tool use, or iterative search

        Query: {query}
        Classification:"""

        result = await llm_call(self.classifier, prompt, max_tokens=10)
        return result.strip().lower()
```

### Domain-Specific Routing

For systems with multiple knowledge domains, route queries to the correct index before retrieval.

```python
# Rule-based domain routing
DOMAIN_RULES = {
    "revenue|sales|quota|ARR":     "financial_index",
    "policy|handbook|PTO|benefits": "hr_index",
    "API|endpoint|SDK|integration": "engineering_index",
    "compliance|GDPR|SOC2|audit":   "legal_index",
}

# Embedding-based domain routing (for ambiguous queries)
class DomainRouter:
    def __init__(self):
        self.domain_centroids = {}  # pre-computed per domain

    def route(self, query_embedding: list[float]) -> str:
        similarities = {
            domain: cosine_sim(query_embedding, centroid)
            for domain, centroid in self.domain_centroids.items()
        }
        return max(similarities, key=similarities.get)
```

---

## Semantic Caching for RAG

Semantic caching recognizes when a new query has essentially the same meaning as a prior query and reuses the cached result. Production systems report up to 68% cost reduction and 65x latency improvement with well-tuned semantic caches.

### Three-Layer Caching Architecture

```
  User Query
      |
      v
+---------------------+
| Layer 1: Exact Cache |  Hash(query) -> response
| (Redis/Memcached)    |  TTL: 1 hour
| Hit rate: ~15-25%    |  Latency: <5ms
+----------+----------+
           | miss
           v
+---------------------+
| Layer 2: Semantic    |  Embed(query) -> nearest neighbor
| Cache (Vector DB)    |  Threshold: cosine > 0.95
| Hit rate: ~20-35%    |  Latency: <50ms
+----------+----------+
           | miss
           v
+---------------------+
| Layer 3: Document    |  Cache retrieved chunks
| Cache               |  Skip re-embedding
| (saves embedding $) |  TTL: until doc changes
+----------+----------+
           | miss
           v
    Full RAG Pipeline
```

### Semantic Cache Implementation

```python
class SemanticCache:
    """Cache RAG responses by query semantic similarity."""

    def __init__(self, vector_store, similarity_threshold: float = 0.95):
        self.vector_store = vector_store
        self.threshold = similarity_threshold
        self.response_store = {}  # query_id -> cached response

    async def get(self, query: str) -> Optional[CachedResponse]:
        # Step 1: Exact match (fast path)
        exact_key = hashlib.sha256(query.encode()).hexdigest()
        if exact_key in self.response_store:
            return self.response_store[exact_key]

        # Step 2: Semantic match
        query_embedding = await embed(query)
        results = self.vector_store.search(
            query_embedding, top_k=1
        )

        if results and results[0].score >= self.threshold:
            cached_id = results[0].metadata["response_id"]
            cached = self.response_store.get(cached_id)
            if cached and not cached.is_expired():
                return cached

        return None

    async def put(
        self, query: str, response: str,
        sources: list[str], ttl_seconds: int = 3600
    ):
        query_embedding = await embed(query)
        response_id = str(uuid4())

        # Store the embedding for future similarity lookups
        self.vector_store.upsert(
            id=response_id,
            embedding=query_embedding,
            metadata={"response_id": response_id}
        )

        # Store the actual response
        self.response_store[response_id] = CachedResponse(
            response=response,
            sources=sources,
            created_at=time.time(),
            ttl=ttl_seconds,
        )
```

### Cache Invalidation Strategies

| Strategy | Trigger | Use Case |
|----------|---------|----------|
| **TTL-based** | Fixed time expiry | General queries, news |
| **Event-driven** | Document update webhook | Knowledge bases |
| **Version-tagged** | Doc version mismatch | Compliance-critical |
| **Confidence-gated** | Low retrieval score | Volatile domains |

**Critical Rule**: Always cache the source document IDs alongside the response. When any source document is updated, invalidate all cache entries that reference it.

```python
# Webhook-based cache invalidation
@app.post("/webhook/document-updated")
async def on_document_updated(doc_id: str):
    # Find all cache entries that used this document
    affected = cache_index.find_by_source(doc_id)
    for entry in affected:
        semantic_cache.invalidate(entry.response_id)
    logger.info(f"Invalidated {len(affected)} cache entries for doc {doc_id}")
```

---

## Multi-Index Strategies

A single monolithic index does not scale. Production systems partition their vector indexes by domain, tenant, or document type to improve retrieval precision and operational isolation.

### Index Partitioning Patterns

```
Pattern 1: Per-Domain Indexes
+--------+  +--------+  +--------+  +--------+
|  Legal |  |   HR   |  |Finance |  |  Eng   |
| Index  |  | Index  |  | Index  |  | Index  |
+--------+  +--------+  +--------+  +--------+
    |            |            |           |
    +----------- +-----+------+-----------+
                       |
                 Query Router
                       |
                  User Query


Pattern 2: Per-Tenant Indexes (Silo Model)
+----------+  +----------+  +----------+
| Tenant A |  | Tenant B |  | Tenant C |
|  Index   |  |  Index   |  |  Index   |
| (Acme)   |  | (Globex) |  | (Wayne)  |
+----------+  +----------+  +----------+


Pattern 3: Shared Index with Metadata Filtering (Pool Model)
+-------------------------------------------+
|           Shared Vector Index              |
|  +-------+  +-------+  +-------+          |
|  | doc_1 |  | doc_2 |  | doc_3 |  ...     |
|  | t:A   |  | t:B   |  | t:A   |          |
|  +-------+  +-------+  +-------+          |
|                                            |
|  WHERE tenant_id = "A"  <-- filter         |
+-------------------------------------------+
```

### When to Use Each Pattern

| Pattern | Isolation | Cost | Operational Complexity | Best For |
|---------|-----------|------|----------------------|----------|
| **Per-Domain** | Medium | Medium | Medium | Internal tools with distinct knowledge domains |
| **Per-Tenant Silo** | Strongest | High | High | Enterprise SaaS, regulated industries |
| **Shared Pool** | Weakest | Low | Low | SMB SaaS, cost-sensitive products |
| **Hybrid Bridge** | Configurable | Medium | High | Mixed customer base (enterprise + SMB) |

### Hierarchical Index Strategy

For very large corpora, use a two-tier index: a coarse "summary index" for routing, and fine-grained "chunk indexes" for precision.

```
  Query: "What is the refund policy for enterprise plans?"
      |
      v
+--------------------+
| Summary Index      |  Contains doc-level summaries
| (10K entries)      |  Fast, broad search
+--------+-----------+
         |
         | Top 3 matching docs identified
         v
+--------------------+
| Chunk Index        |  Contains 500-token chunks
| (2M entries)       |  Precise, targeted search
| Filtered to 3 docs |
+--------+-----------+
         |
         v
   Top 5 chunks -> LLM
```

---

## RAG Pipeline Optimization

A naive sequential RAG pipeline adds latency at every step. Production pipelines use parallelism, batching, and async processing to meet sub-second SLAs.

### Sequential vs Optimized Pipeline

```
SEQUENTIAL (Naive):
Query -> Embed(200ms) -> Search(150ms) -> Rerank(300ms) -> Generate(800ms)
Total: ~1450ms

OPTIMIZED (Parallel + Cached):
Query ----+---> Embed(200ms) ---> Vector Search(150ms) ---+
          |                                                |--> RRF Merge -> Rerank(300ms) -> Generate(800ms)
          +---> BM25 Keyword Search(100ms) ---------------+
          |
          +---> Cache Check(5ms) -- HIT --> Return cached (5ms total)

With cache miss: ~1050ms (embedding + keyword in parallel)
With cache hit:  ~5ms
```

### Parallel Retrieval

```python
async def parallel_retrieve(
    query: str,
    query_embedding: list[float],
    indexes: list[str],
) -> list[Chunk]:
    """Run vector search, keyword search, and graph traversal in parallel."""

    tasks = [
        vector_search(query_embedding, index="main", top_k=20),
        bm25_search(query, index="main", top_k=20),
        # Optionally, graph-based retrieval for entity queries
        graph_search(query, max_hops=2, top_k=10),
    ]

    # All retrieval strategies execute concurrently
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # Filter out failures (graceful degradation)
    valid_results = [r for r in results if not isinstance(r, Exception)]

    # Merge with Reciprocal Rank Fusion
    merged = reciprocal_rank_fusion(valid_results, k=60)

    return merged[:20]  # top 20 after fusion
```

### Batched Embedding

When processing ingestion or multiple queries simultaneously, batch embedding calls to maximize GPU utilization.

```python
class EmbeddingBatcher:
    """Batch embedding requests to reduce per-call overhead."""

    def __init__(self, model: str, batch_size: int = 64, max_wait_ms: int = 50):
        self.model = model
        self.batch_size = batch_size
        self.max_wait = max_wait_ms / 1000
        self.queue: asyncio.Queue = asyncio.Queue()
        self._running = True

    async def embed(self, text: str) -> list[float]:
        """Submit a single text and wait for its embedding."""
        future = asyncio.Future()
        await self.queue.put((text, future))
        return await future

    async def _batch_loop(self):
        """Background loop that collects and processes batches."""
        while self._running:
            batch = []
            try:
                # Wait for at least one item
                item = await asyncio.wait_for(
                    self.queue.get(), timeout=1.0
                )
                batch.append(item)

                # Collect more items up to batch_size or max_wait
                deadline = time.time() + self.max_wait
                while len(batch) < self.batch_size and time.time() < deadline:
                    try:
                        item = await asyncio.wait_for(
                            self.queue.get(),
                            timeout=max(0, deadline - time.time())
                        )
                        batch.append(item)
                    except asyncio.TimeoutError:
                        break

                # Process the batch
                texts = [t for t, _ in batch]
                embeddings = await embed_batch(self.model, texts)

                for (_, future), emb in zip(batch, embeddings):
                    future.set_result(emb)

            except asyncio.TimeoutError:
                continue
```

### Streaming Generation with Early Retrieval

Start retrieval before the user finishes typing (on pause detection) and stream generation tokens as they are produced.

```
Timeline:
0ms     User starts typing...
300ms   Pause detected -> trigger retrieval speculatively
500ms   User submits query
        Retrieval already 200ms in -> finishes at 650ms
650ms   Reranking begins
950ms   First generation token streams to user
1800ms  Full response complete

vs. without speculation:
0ms     User submits query
200ms   Embedding
350ms   Retrieval
650ms   Reranking
1500ms  First token
2300ms  Full response complete
```

---

## Corrective RAG: Self-Checking Retrieval

Corrective RAG (CRAG) adds a verification layer between retrieval and generation. The system evaluates whether retrieved documents actually answer the query before generating a response.

### The CRAG Decision Loop

```
  User Query
      |
      v
  Retrieve Top-K
      |
      v
+------------------+
| Relevance Grader  |  "Are these docs relevant to the query?"
| (LLM or trained   |
|  classifier)      |
+--------+---------+
         |
    +----+----+--------+
    |         |        |
    v         v        v
 CORRECT   AMBIGUOUS  WRONG
    |         |        |
    v         v        v
 Generate  Supplement  Discard &
 directly  with web    re-retrieve
           search      with reformulated
                       query
```

The AMBIGUOUS branch needs a web search backend, and gateways now sell one. Cloudflare's Web Search API (beta, October 2, 2026) runs inside AI Gateway and returns structured web snippets from Ceramic.ai, Exa and Linkup at the partners' list prices with no markup, billed and logged through the gateway. Whoever supplies it, treat web results as untrusted input: they can carry prompt injection, so put them in a separate, labeled context block and never let them outrank a tenant's own documents in the prompt.

### Implementation

```python
class CorrectiveRAG:
    """Self-correcting RAG pipeline with retrieval quality checks."""

    def __init__(self, max_corrections: int = 2):
        self.max_corrections = max_corrections

    async def answer(self, query: str) -> RAGResponse:
        attempts = 0
        current_query = query
        all_sources = []

        while attempts <= self.max_corrections:
            # Step 1: Retrieve
            chunks = await retrieve(current_query, top_k=10)

            # Step 2: Grade relevance
            grade = await self._grade_relevance(query, chunks)

            if grade.verdict == "correct":
                # High-confidence retrieval, generate directly
                return await self._generate(query, chunks, all_sources)

            elif grade.verdict == "ambiguous":
                # Supplement with additional search
                web_results = await web_search(current_query)
                chunks = self._merge_and_dedupe(chunks, web_results)
                return await self._generate(query, chunks, all_sources)

            else:  # "wrong"
                # Reformulate query and retry
                current_query = await self._reformulate(
                    original_query=query,
                    failed_query=current_query,
                    reason=grade.reason,
                )
                all_sources.extend(chunks)
                attempts += 1

        # Exhausted retries: generate best-effort with disclaimer
        return await self._generate_with_caveat(query, all_sources)

    async def _grade_relevance(
        self, query: str, chunks: list[Chunk]
    ) -> RelevanceGrade:
        """Use LLM to grade whether chunks answer the query."""
        prompt = f"""Given this query and retrieved documents, assess relevance.

Query: {query}

Documents:
{self._format_chunks(chunks)}

Respond with:
- verdict: "correct" (docs clearly answer the query)
- verdict: "ambiguous" (docs partially relevant, need supplementing)
- verdict: "wrong" (docs are irrelevant to the query)
- reason: brief explanation

JSON response:"""

        result = await llm_call(prompt, response_format="json")
        return RelevanceGrade(**json.loads(result))
```

### Self-RAG: Critic Tokens

Self-RAG extends this pattern with inline critic tokens. The model evaluates its own output at each step:

1. **[Retrieve]**: Should I retrieve? (Yes/No)
2. **[Relevant]**: Is retrieved info relevant? (Yes/No)
3. **[Supported]**: Is my answer supported by the evidence? (Fully/Partially/No)
4. **[Useful]**: Is this answer actually useful? (Score 1-5)

If any critic check fails, the model loops back to an earlier step.

---

## Adaptive Retrieval

Not every query benefits from retrieval. Adaptive retrieval decides dynamically whether to retrieve, how much to retrieve, and from which sources.

### The Retrieval Decision Tree

```
  User Query
      |
      v
  "Does this query need external knowledge?"
      |
  +---+---+
  |       |
  No      Yes
  |       |
  v       v
Direct   "How complex is the retrieval need?"
 LLM      |
answer  +-+--+---------+
        |    |         |
        v    v         v
     Single Multi    Agentic
      hop   hop      (planning
        |    |       required)
        v    v         |
     1 index 2-3       v
     top-5  indexes  Full agent
             top-10  loop
```

### Query Complexity Estimator

```python
class AdaptiveRetriever:
    """Decides retrieval strategy based on query characteristics."""

    async def retrieve(self, query: str) -> RetrievalPlan:
        # Fast heuristics first
        if self._is_general_knowledge(query):
            return RetrievalPlan(strategy="none", reason="general knowledge")

        if self._is_simple_lookup(query):
            return RetrievalPlan(
                strategy="single_hop",
                indexes=["primary"],
                top_k=5,
            )

        # LLM-based assessment for ambiguous cases
        plan = await self._plan_retrieval(query)
        return plan

    def _is_general_knowledge(self, query: str) -> bool:
        """Check if query is about widely known facts."""
        general_indicators = [
            "what is", "who is", "define", "explain the concept",
        ]
        has_org_refs = bool(re.search(
            r"(our|my|the company|internal|proprietary)", query.lower()
        ))
        is_general = any(
            query.lower().startswith(g) for g in general_indicators
        )
        return is_general and not has_org_refs

    def _is_simple_lookup(self, query: str) -> bool:
        """Check if query can be answered with a single document."""
        single_hop_patterns = [
            r"what is (the|our) .+ policy",
            r"how (do I|to) .+",
            r"where (can I|do I) find",
        ]
        return any(re.search(p, query.lower()) for p in single_hop_patterns)
```

### Token-Budget Aware Retrieval

Scale the retrieval effort based on available token budget and expected response complexity.

```python
def plan_retrieval_budget(query: str, max_budget_tokens: int = 4000):
    """Allocate token budget across retrieval and generation."""

    complexity = estimate_complexity(query)  # 1-5 scale

    if complexity <= 2:
        # Simple query: small context, save tokens for generation
        return {"context_tokens": 1000, "generation_tokens": 3000, "top_k": 3}
    elif complexity <= 4:
        # Medium: balanced
        return {"context_tokens": 2500, "generation_tokens": 1500, "top_k": 8}
    else:
        # Complex: heavy retrieval, concise generation
        return {"context_tokens": 3500, "generation_tokens": 500, "top_k": 15}
```

### Adaptive Effort as a Managed Feature

Managed search services now ship this escalation pattern as configuration. Azure AI Search knowledge bases went GA in REST API 2026-04-01 (extractive retrieval; query planning and answer synthesis remain preview), and the 2026-08-01-preview API adds `retrievalReasoningEffort: "auto"`: run a lightweight retrieval pass first and escalate to LLM query planning, up to medium effort, only when grounding is insufficient. The same preview streams retrieve results over server-sent events and lets you bypass reranking per knowledge source. If you build the router yourself, copy the shape: the cheap path must be the default, and the escalation trigger must be a measured grounding signal, not a guess about query complexity.

---

## Cost Optimization Patterns

At scale, RAG costs compound across embedding, retrieval, reranking, and generation. Unoptimized systems can spend 10-50x more than necessary.

### Cost Breakdown of a Typical RAG Query

```
Component         Cost per Query    % of Total    Optimization
-----------------------------------------------------------------
Embedding         $0.000005         ~1%           Batch + cache
Vector Search     $0.00001          ~2%           Index optimization
Reranking         $0.0001           ~5-15%        Skip for simple queries
LLM Generation    $0.0004-0.008     ~80-95%       Model tiering, caching
-----------------------------------------------------------------
Total (naive)     ~$0.001-0.01
Total (optimized) ~$0.0002-0.002    (5-10x reduction)
```

### Tiered Model Strategy

```
                Query Complexity
                Low         Medium        High
             +----------+----------+----------+
 Generation  |  Small   |  Mid     |  Large   |
 Model       |  Model   |  Model   |  Model   |
             | (GPT-6   | (Sonnet  | (Opus    |
             |  Luna)   |  5.5)    |  5.5)    |
             | ~$0.0004 | ~$0.008  | ~$0.016  |
             +----------+----------+----------+

 Reranking   |  Skip    | Lightweight| Cross-  |
             |          | reranker   | encoder |
             +----------+----------+----------+
```

Costs are per request at 2.6K input and 300 output tokens, at November 2, 2026 list prices (unchanged since October 1), before reasoning tokens. Note where the spread is: Claude Opus 5.5 ($4 / $20) is only 2x Sonnet 5.5 ($2 / $10), while Sonnet 5.5 or GPT-6 Sol is 20x GPT-6 Luna. Most of the routing value is now at the small/mid boundary, so spend classifier effort there; the mid/large boundary barely pays for its misroutes.

### Progressive Detail Pattern

Answer with minimal retrieval first. Only escalate if the user asks follow-up questions or if confidence is low.

```python
class ProgressiveRAG:
    """Start cheap, escalate only when needed."""

    async def answer(self, query: str, session: Session) -> str:
        # Level 1: Try semantic cache
        cached = await self.cache.get(query)
        if cached:
            return cached.response  # Cost: ~$0

        # Level 2: Fast retrieval + small model
        chunks = await retrieve(query, top_k=3)
        response = await generate(
            query, chunks, model="gpt-6-luna"
        )

        # Check confidence
        if response.confidence > 0.85:
            await self.cache.put(query, response)
            return response.text  # Cost: ~$0.0003

        # Level 3: Deep retrieval + reranking + larger model
        chunks = await retrieve(query, top_k=15)
        reranked = await rerank(query, chunks, top_k=5)
        response = await generate(
            query, reranked, model="claude-sonnet-5-5"  # replaces claude-sonnet-4-5 (retires Nov 30, 2026)
        )

        if response.confidence > 0.7:
            await self.cache.put(query, response)
            return response.text  # Cost: ~$0.008

        # Level 4: Full agentic pipeline (expensive but thorough)
        return await self.agentic_pipeline.run(query)  # Cost: ~$0.05
```

### Cost Guardrails

```python
class CostGuard:
    """Prevent runaway costs in production RAG."""

    def __init__(self):
        self.daily_budget = 500.0  # $500/day
        self.per_query_limit = 0.10  # $0.10 max per query
        self.per_user_hourly = 1.0  # $1/user/hour

    async def check(self, user_id: str, estimated_cost: float) -> bool:
        daily_spent = await self.get_daily_spend()
        if daily_spent + estimated_cost > self.daily_budget:
            raise BudgetExceededError("Daily budget exhausted")

        user_spent = await self.get_user_hourly_spend(user_id)
        if user_spent + estimated_cost > self.per_user_hourly:
            raise RateLimitError("User hourly budget exceeded")

        if estimated_cost > self.per_query_limit:
            # Downgrade to cheaper strategy
            return False  # signals caller to use cheaper path

        return True
```

### Build vs Buy: Managed RAG Unit Prices

Managed retrieval now publishes per-query prices, which turns build-versus-buy into arithmetic. Cloudflare AI Search went GA on October 1, 2026: Workers AI, Vectorize, R2 and Browser Run underneath, vector and keyword search in parallel with fusion and optional reranking, optional OCR for scanned PDFs, and image retrieval through Qwen3-VL-Embedding. Billing starts November 1, 2026:

| Item | Price | Free per month (every Workers plan) |
|------|-------|-------------------------------------|
| Ingestion | $0.75 per 1M tokens, plus $0.50 per 1M for image processing | 5M tokens |
| Storage | $2.00 per GB-month | 10 GB |
| Semantic queries (vector or hybrid) | $0.75 per 1K | 1,000 |
| Full-text queries | $0.10 per 1K | 1,000 |

At 1M hybrid queries a month, retrieval costs $750: far less than the engineering time to run your own stack. At 1,000 QPS (about 2.6B queries a month) it is about $1.9M a month for retrieval alone, which pays for a lot of self-hosted vector DB nodes and an on-call rotation. The crossover sits in the tens of millions of queries per month; compute it with your own infrastructure and staffing costs. Other vendors are packaging the same stack: Cohere Compass Cloud (private beta, September 25, 2026) bundles parsing, dense plus sparse embedding, permission-aware retrieval and reranking behind an MCP server that agents use to narrow the corpus step by step.

**Lifecycle risk is the other half of "buy".** OpenAI shut down the Assistants API on August 26, 2026. Vector stores survived: the Responses API `file_search` tool takes the same `vector_store_ids` with `max_num_results` and metadata filters, so indexes needed no re-ingestion, but thread and run orchestration code had to be rewritten. Keep retrieval storage and orchestration separable so either can move without the other.

---

## Failure Modes and Debugging

Production RAG systems have compounding failure probabilities. With 95% reliability at each of three stages, overall reliability drops to 0.95 x 0.95 x 0.95 = 0.86. Understanding failure modes is essential.

### The RAG Failure Taxonomy

```
+------------------------------------------------------------------+
|                    RAG Failure Modes                               |
+------------------------------------------------------------------+
|                                                                    |
|  RETRIEVAL FAILURES          GENERATION FAILURES                   |
|  +---------------------+    +-------------------------+           |
|  | Missing documents   |    | Hallucination despite   |           |
|  | (not indexed)       |    | good context            |           |
|  +---------------------+    +-------------------------+           |
|  | Wrong chunks        |    | Ignoring retrieved      |           |
|  | (low precision)     |    | context                 |           |
|  +---------------------+    +-------------------------+           |
|  | Missed chunks       |    | Over-reliance on one    |           |
|  | (low recall)        |    | source                  |           |
|  +---------------------+    +-------------------------+           |
|  | Stale embeddings    |    | Citation fabrication    |           |
|  | (drift)             |    |                         |           |
|  +---------------------+    +-------------------------+           |
|                                                                    |
|  SYSTEM FAILURES             QUALITY FAILURES                      |
|  +---------------------+    +-------------------------+           |
|  | Index unavailable   |    | Chunking artifacts      |           |
|  +---------------------+    +-------------------------+           |
|  | Embedding service   |    | Context window overflow |           |
|  | timeout             |    +-------------------------+           |
|  +---------------------+    | Answer too vague        |           |
|  | Reranker OOM        |    | (over-hedging)          |           |
|  +---------------------+    +-------------------------+           |
|                                                                    |
+------------------------------------------------------------------+
```

### Many Retrieval Failures Are Chunking Failures

The retriever can only return chunks the chunker created. A fact split across two chunks, or a chunk stripped of the section header that gave it meaning, cannot be retrieved however good the embeddings and reranker are. So when the checklist below points at retrieval, inspect the chunks before you swap the embedding model. Common chunking failures:

- **Chunk too small**: Loses context. "It costs $200": what costs $200?
- **Chunk too large**: Dilutes relevance. A 2000-token chunk where only 1 sentence is relevant.
- **Boundary splits**: A table or list is split across two chunks.
- **Missing metadata**: Chunks lack headers, document titles, or section context.

### Debugging Checklist

```
When RAG quality drops, investigate in this order:

1. RETRIEVAL QUALITY (check first: most common root cause)
   [ ] Log the query and retrieved chunks side by side
   [ ] Compute retrieval precision@K manually for 20 failing queries
   [ ] Check if relevant documents exist in the index at all
   [ ] Compare BM25 vs vector results: if BM25 wins, embeddings are stale

2. CHUNKING QUALITY (check second)
   [ ] Sample 50 random chunks: do they make sense in isolation?
   [ ] Check chunk boundaries for tables, lists, code blocks
   [ ] Verify metadata (title, section, doc_id) is present

3. RERANKING QUALITY (check third)
   [ ] Compare pre-rerank vs post-rerank orderings
   [ ] Check if reranker is pushing relevant results down

4. GENERATION QUALITY (check last)
   [ ] Test with perfect context (manually curated): does LLM still fail?
   [ ] Check for context window overflow (truncated chunks)
   [ ] Verify system prompt is not conflicting with retrieved context
```

### Agentic RAG Failure Modes

Agentic RAG introduces three additional failure patterns:

1. **Retrieval Thrash**: Agent repeatedly retrieves without converging on an answer. Traces show near-duplicate queries and oscillating search terms. Fix: limit to 3-5 retrieval iterations and track query uniqueness per session.

2. **Tool Storms**: Agent calls tools excessively in a single turn. Fix: set per-query tool call limits and cost ceilings.

3. **Context Bloat**: Agent accumulates too many retrieved chunks, overflowing the context window. Fix: implement a sliding window that drops the oldest chunks when context exceeds threshold.

---

## Monitoring and Alerting

Production RAG requires dedicated monitoring beyond standard application metrics. Treat systematic evaluation as a day-one requirement rather than the "ship first, eval later" pattern of earlier RAG generations; see [RAG Evaluation Patterns](13-rag-evaluation-patterns.md) for the metrics and judge tiers.

### The RAG Monitoring Stack

```
+--------------------------------------------------------------------+
|                    RAG Observability Layers                          |
+--------------------------------------------------------------------+
|                                                                      |
|  L1: INFRASTRUCTURE          L2: PIPELINE                           |
|  +----------------------+   +-----------------------------+         |
|  | Latency (p50/p95/p99)|   | Retrieval precision@K      |         |
|  | Error rates          |   | Retrieval recall@K         |         |
|  | Throughput (QPS)     |   | Reranker effectiveness     |         |
|  | Cache hit rate       |   | Chunk utilization rate     |         |
|  | Index size/growth    |   | Context window fill rate   |         |
|  +----------------------+   +-----------------------------+         |
|                                                                      |
|  L3: QUALITY                 L4: BUSINESS                           |
|  +----------------------+   +-----------------------------+         |
|  | Faithfulness score   |   | User satisfaction (thumbs) |         |
|  | Answer relevancy     |   | Task completion rate       |         |
|  | Hallucination rate   |   | Escalation to human rate   |         |
|  | Citation accuracy    |   | Cost per successful query  |         |
|  +----------------------+   +-----------------------------+         |
|                                                                      |
+--------------------------------------------------------------------+
```

### Key Metrics and Alerts

| Metric | Target | Alert Threshold | Action |
|--------|--------|-----------------|--------|
| **p95 Latency** | <2s | >5s | Scale retrieval infra |
| **Cache Hit Rate** | >40% | <20% | Tune similarity threshold |
| **Retrieval Precision@5** | >0.7 | <0.5 | Re-evaluate chunking |
| **Faithfulness** | >0.9 | <0.8 | Audit generation prompts |
| **Hallucination Rate** | <5% | >10% | Tighten grounding prompt |
| **Empty Retrieval Rate** | <2% | >5% | Check index coverage |
| **Cost per Query** | <$0.005 | >$0.02 | Review model tiering |

### End-to-End Trace Logging

Every query should produce a trace that links all pipeline stages with a single request ID.

```python
@dataclass
class RAGTrace:
    request_id: str
    timestamp: datetime
    query: str
    route: str                    # "simple_rag", "complex_rag", etc.
    cache_hit: bool
    retrieval_latency_ms: float
    chunks_retrieved: int
    chunks_after_rerank: int
    rerank_latency_ms: float
    generation_model: str
    generation_latency_ms: float
    total_latency_ms: float
    input_tokens: int
    output_tokens: int
    estimated_cost: float
    faithfulness_score: float     # 0-1, computed async
    user_feedback: Optional[str]  # thumbs up/down

    def to_dict(self) -> dict:
        return asdict(self)
```

### Automated Quality Sampling

Run offline evaluation on a sample of production queries to detect quality drift before users notice.

```python
async def nightly_quality_check(sample_size: int = 200):
    """Sample production queries and evaluate RAG quality."""
    traces = await get_recent_traces(limit=sample_size)

    scores = []
    for trace in traces:
        # Re-run the query with evaluation
        eval_result = await evaluate_rag_response(
            query=trace.query,
            response=trace.response,
            retrieved_chunks=trace.chunks,
            metrics=["faithfulness", "relevancy", "context_precision"],
        )
        scores.append(eval_result)

    avg_faithfulness = mean([s.faithfulness for s in scores])
    avg_relevancy = mean([s.relevancy for s in scores])

    if avg_faithfulness < 0.85:
        alert("RAG faithfulness degraded", severity="high")
    if avg_relevancy < 0.70:
        alert("RAG relevancy degraded", severity="medium")

    publish_metrics("rag.nightly.faithfulness", avg_faithfulness)
    publish_metrics("rag.nightly.relevancy", avg_relevancy)
```

---

## Scaling to Millions of Documents

Moving from thousands to millions of documents introduces challenges in indexing throughput, retrieval latency, and index management.

### Scaling Dimensions

```
Documents:   1K  -->  100K  -->  1M  -->  100M
             |        |         |         |
Chunks:      10K      1M        10M       1B
             |        |         |         |
Index Size:  50MB     5GB       50GB      5TB
             |        |         |         |
Strategy:    Single   Single    Sharded   Distributed
             Node     Node +    Index     Cluster +
                      Replicas             Tiered
```

### Ingestion Pipeline at Scale

```
  Document Sources
  (S3, DBs, APIs, File Shares)
         |
         v
+-------------------+
| Ingestion Queue   |  (Kafka / SQS)
| - Deduplication   |
| - Priority queue  |
+--------+----------+
         |
    +----+----+----+----+
    |    |    |    |    |     Parallel workers
    v    v    v    v    v
  +--+ +--+ +--+ +--+ +--+
  |W1| |W2| |W3| |W4| |W5|  Parse + Chunk + Embed
  +--+ +--+ +--+ +--+ +--+
    |    |    |    |    |
    +----+----+----+----+
         |
         v
+-------------------+
| Vector DB Cluster |
| (Sharded by       |
|  doc_type or      |
|  tenant_id)       |
+-------------------+
```

### Sharding Strategies

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Hash-based** | shard = hash(doc_id) % N | Even distribution | Cross-shard queries needed |
| **Range-based** | shard by date range | Time-based queries fast | Uneven shard sizes |
| **Domain-based** | shard by document type | No cross-shard queries | Unbalanced domains |
| **Tenant-based** | shard by tenant_id | Perfect isolation | Many small shards |

### Index Maintenance

At millions of documents, index maintenance becomes a critical operational concern.

```python
class IndexMaintenanceScheduler:
    """Scheduled tasks for index health at scale."""

    async def run_daily(self):
        # 1. Detect and re-embed stale documents
        stale_docs = await find_docs_with_old_embeddings(
            older_than_days=90,
            embedding_model_version="v2"  # current is v3
        )
        if stale_docs:
            await enqueue_reembedding(stale_docs)

        # 2. Remove orphaned vectors (doc deleted but vector remains)
        orphans = await find_orphaned_vectors()
        if orphans:
            await delete_vectors(orphans)

        # 3. Compact and optimize indexes
        for shard in await list_shards():
            if shard.fragmentation_pct > 20:
                await compact_shard(shard.id)

        # 4. Verify index health
        for shard in await list_shards():
            health = await check_shard_health(shard.id)
            if not health.ok:
                alert(f"Shard {shard.id} unhealthy: {health.reason}")
```

### Re-Embedding Without Downtime

"New embedding model means a full re-index" is no longer the only answer:

- **Shared embedding spaces.** The Voyage 4 family (voyage-4-large $0.12, voyage-4 $0.06, voyage-4-lite $0.02 per 1M tokens, plus open-weight voyage-4-nano) shares one space, as do Cohere's `embed-v5.0-pro` and `-fast` (September 30, 2026). Index once with the large model and query with the small one; you can change the query-side model without touching the index. Qdrant's Constella research preview (September 29, 2026) pushes the idea further: a fixed Stella (400M) document index with swappable query encoders, where the 34.5M-parameter Constella Nano keeps about 91% of Stella's BEIR nDCG@10 at 12x lower CPU latency (vendor-reported).
- **Online backfill.** When the document side must change, add a second vector field, backfill it in the background, shadow-query both on the golden set, then cut over. Milvus 3.0 (July 2026) supports adding, backfilling and dropping columns online, which makes this a hot path over hundreds of millions of rows.
- **Embedding inside the database.** MongoDB Atlas Automated Embedding (GA August 13, 2026) and turbopuffer native embedding (GA September 29, 2026) remove the sync pipeline, but they tie the index to a vendor-chosen model version. Ask how model upgrades and re-embedding are scheduled and billed before adopting.

### Memory Tiers and Quantization

At 1,536 float32 dimensions a vector is 6 KB, so a billion chunks is about 6 TB before the graph. Rotation-based low-bit quantization is the main lever, and it is **opt-in or preview, not a default**: Qdrant added Hadamard-rotated TurboQuant in 1.18, a 4-bit-only Turbo4 storage type in 1.19 (9x smaller than float32 plus a 4-bit copy, at the cost of peak recall, because the originals are gone), and `turbo8` and `turbo16` datatypes in 1.20; Weaviate 1.39 previews 4-bit Rotational Quantization (6,144 bytes to 784 for a 1,536-dim vector), and 1.40 adds a centered RQ4 variant. Qdrant 1.19 also replaced `on_disk` / `always_ram` with one `memory` setting (pinned, cached or cold) across vectors, HNSW, sparse and payload indexes.

The production pattern: quantized vectors pinned in RAM for candidate search, full-precision originals cold on disk for rescoring the top candidates, and ANN recall measured against exact kNN after every quantization change (see [RAG Evaluation Patterns](13-rag-evaluation-patterns.md#separate-the-embedder-from-the-index)).

### Read Replicas for Retrieval

Separate read and write paths so that ingestion never degrades query latency.

```
  Ingestion Pipeline              Query Pipeline
        |                              |
        v                              v
  +-----------+     Replication   +-----------+
  |  Primary  | ----------------> |  Replica  |
  |  (Write)  |                   |  (Read)   |
  +-----------+                   +-----------+
                                  |  Replica  |
                                  |  (Read)   |
                                  +-----------+
                                  |  Replica  |
                                  |  (Read)   |
                                  +-----------+
```

---

## Multi-Tenant RAG Isolation

Multi-tenant RAG is the most common production pattern for SaaS products. Getting isolation wrong means data leaks between tenants, which is a critical security failure.

### Three Isolation Models

```
SILO MODEL (Strongest Isolation)
+----------+  +----------+  +----------+
| Tenant A |  | Tenant B |  | Tenant C |
| +------+ |  | +------+ |  | +------+ |
| |Index | |  | |Index | |  | |Index | |
| +------+ |  | +------+ |  | +------+ |
| |Cache | |  | |Cache | |  | |Cache | |
| +------+ |  | +------+ |  | +------+ |
+----------+  +----------+  +----------+
Cost: $$$$    Best for: Enterprise, Regulated Industries


POOL MODEL (Cost-Efficient)
+-------------------------------------------+
|              Shared Index                  |
|  [A] [B] [A] [C] [B] [A] [C] [B] [C]    |
|                                            |
|  Every query includes:                     |
|  WHERE tenant_id = ? (MANDATORY)           |
+-------------------------------------------+
Cost: $       Best for: SMB SaaS


BRIDGE MODEL (Hybrid)
+----------+  +----------------------------+
| Tenant A |  |     Shared Pool            |
| (Enterprise) | [B] [C] [D] [E] [F] [G]  |
| +------+ |  |                            |
| |Dedicated|  | WHERE tenant_id = ?       |
| |Index | |  +----------------------------+
| +------+ |
+----------+
Cost: $$      Best for: Mixed customer base
```

### Security: Defense in Depth

```python
class TenantIsolatedRetriever:
    """Enforces tenant isolation at every retrieval layer."""

    async def retrieve(
        self, query: str, tenant_id: str, user_id: str
    ) -> list[Chunk]:
        # Layer 1: Tenant ID is MANDATORY in every query
        if not tenant_id:
            raise SecurityError("tenant_id required for retrieval")

        # Layer 2: Validate user belongs to tenant
        if not await self.authz.user_in_tenant(user_id, tenant_id):
            raise AuthorizationError("User not in tenant")

        # Layer 3: Apply tenant filter at the database level
        chunks = await self.vector_db.search(
            query_embedding=await embed(query),
            filter={"tenant_id": {"$eq": tenant_id}},  # ALWAYS filtered
            top_k=10,
        )

        # Layer 4: Post-retrieval verification
        for chunk in chunks:
            assert chunk.metadata["tenant_id"] == tenant_id, \
                f"Cross-tenant leak detected: {chunk.id}"

        # Layer 5: Audit log
        await self.audit_log.record(
            action="retrieve",
            tenant_id=tenant_id,
            user_id=user_id,
            chunk_ids=[c.id for c in chunks],
        )

        return chunks
```

### Leaks the Tenant Filter Does Not Stop

A mandatory `tenant_id` filter stops cross-tenant *results*. It does not stop cross-tenant *signals*:

| Channel | How it leaks | Mitigation |
|---------|--------------|------------|
| **Shared BM25 statistics** | In a pooled index, IDF is computed over every tenant's documents, so one tenant's vocabulary shifts another's ranking and scores reflect corpus-wide term frequencies | Per-tenant IDF (Qdrant 1.19 added it for sparse/BM25 search), or a siloed lexical index |
| **Inference prefix cache** | On self-hosted engines a shared prefix cache is a timing oracle for whether another tenant sent the same prefix | Per-tenant cache salt on every code path. vLLM GHSA-935w-9g4m-p28p (fixed in 0.30.0) was one path that dropped the salt; run vLLM 0.32.0 or later, the current security floor (it also fixes CVE-2026-115912 and CVE-2026-115915) |
| **Semantic response cache** | An answer built from Tenant A's documents is served to Tenant B's similar query | Key the cache by tenant and ACL set, never globally |
| **Engine vulnerabilities** | Bugs reachable by any role allowed to build indexes | pgvector 0.8.7 (October 1, 2026) fixes CVE-2026-103484, an IVFFlat build overflow that can lead to code execution (0.8.6 and below). Do not grant index creation to app roles, and confirm your managed Postgres has shipped the patch |

Silo can also mean the customer's own cloud account now: Pinecone BYOC (GA September 23, 2026, on AWS, Google Cloud and Azure) runs the data plane, including vectors, documents, metadata and request payloads, in the customer's cloud, while the vendor control plane manages it through outbound calls only.

### Tenant-Aware Ingestion

Tenant context must be injected at every stage of the pipeline, from ingestion through to generation.

```
Document Upload (Tenant A)
        |
        v
  +---------------------+
  | Validate Ownership  |  Does this doc belong to Tenant A?
  +---------------------+
        |
        v
  +---------------------+
  | Chunk + Embed       |  Attach tenant_id to every chunk
  +---------------------+
        |
        v
  +---------------------+
  | Index with Metadata |  {"tenant_id": "A", "doc_id": "...", ...}
  +---------------------+
        |
        v
  +---------------------+
  | Invalidate Cache    |  Clear Tenant A's cache entries
  +---------------------+             for affected documents
```

### Noisy Neighbor Prevention

In the pool model, one tenant's heavy usage can degrade performance for all tenants.

```python
class TenantRateLimiter:
    """Per-tenant rate limiting and resource quotas."""

    def __init__(self):
        self.tenant_limits = {
            "free":       {"qps": 5,   "daily_queries": 500},
            "pro":        {"qps": 50,  "daily_queries": 10_000},
            "enterprise": {"qps": 200, "daily_queries": 100_000},
        }

    async def check(self, tenant_id: str, tier: str) -> bool:
        limits = self.tenant_limits[tier]

        current_qps = await self.redis.get(f"qps:{tenant_id}")
        if current_qps and int(current_qps) >= limits["qps"]:
            raise RateLimitError(f"QPS limit ({limits['qps']}) exceeded")

        daily_count = await self.redis.get(f"daily:{tenant_id}")
        if daily_count and int(daily_count) >= limits["daily_queries"]:
            raise RateLimitError("Daily query limit exceeded")

        # Increment counters
        pipe = self.redis.pipeline()
        pipe.incr(f"qps:{tenant_id}")
        pipe.expire(f"qps:{tenant_id}", 1)  # 1-second window
        pipe.incr(f"daily:{tenant_id}")
        pipe.expire(f"daily:{tenant_id}", 86400)
        await pipe.execute()

        return True
```

---

## Real-World Architecture Examples

### Example 1: Customer Support RAG

```
+------------------------------------------------------------------+
|                   Customer Support RAG System                     |
+------------------------------------------------------------------+
|                                                                    |
|  Customer Query                                                    |
|       |                                                            |
|       v                                                            |
|  +------------+    +---------+    +------------------+             |
|  | Query      |--->| Semantic|--->| Intent           |             |
|  | Normalizer |    | Cache   |    | Classifier       |             |
|  +------------+    +---------+    +--------+---------+             |
|                     (hit->skip)            |                       |
|                                   +--------+---------+             |
|                                   |                  |             |
|                                   v                  v             |
|                             +-----------+    +-------------+      |
|                             | Knowledge |    | Order/Acct  |      |
|                             | Base RAG  |    | Database    |      |
|                             | (articles,|    | (SQL lookup)|      |
|                             |  FAQs)    |    +-------------+      |
|                             +-----------+           |              |
|                                   |                 |              |
|                                   +--------+--------+              |
|                                            |                       |
|                                            v                       |
|                                   +------------------+             |
|                                   | Response Gen     |             |
|                                   | (with citations  |             |
|                                   |  + confidence)   |             |
|                                   +--------+---------+             |
|                                            |                       |
|                                   +--------+---------+             |
|                                   |                  |             |
|                                   v                  v             |
|                            confidence > 0.8    confidence < 0.8   |
|                            Auto-respond        Route to human      |
|                                                                    |
+------------------------------------------------------------------+

Scale: 50K articles, 2M customer interactions/month
Latency SLA: p95 < 3s
Cache hit rate: ~45%
Auto-resolution rate: ~60%
```

### Example 2: Enterprise Knowledge Platform

```
+------------------------------------------------------------------+
|              Enterprise Multi-Tenant Knowledge Platform            |
+------------------------------------------------------------------+
|                                                                    |
|  +------------------+                                              |
|  | Auth + Tenant    |                                              |
|  | Resolution       |                                              |
|  +--------+---------+                                              |
|           |                                                        |
|           v                                                        |
|  +------------------+                                              |
|  | Query Router     |                                              |
|  +--+----+----+-----+                                              |
|     |    |    |                                                     |
|     v    v    v                                                     |
|  +----+ +----+ +--------+                                          |
|  |Docs| |Wiki| |Tickets |  Per-domain indexes                     |
|  |Idx | |Idx | |Idx     |  (all tenant-filtered)                   |
|  +----+ +----+ +--------+                                          |
|     |    |    |                                                     |
|     +----+----+                                                     |
|          |                                                          |
|          v                                                          |
|  +------------------+                                              |
|  | Cross-Encoder    |                                              |
|  | Reranker         |                                              |
|  +--------+---------+                                              |
|           |                                                        |
|           v                                                        |
|  +------------------+     +-------------------+                    |
|  | Tiered LLM       |<--->| Permission Filter |                    |
|  | Generation        |     | (doc-level ACLs)  |                    |
|  +------------------+     +-------------------+                    |
|           |                                                        |
|           v                                                        |
|  +------------------+                                              |
|  | Response + Audit |                                              |
|  | Trail            |                                              |
|  +------------------+                                              |
|                                                                    |
+------------------------------------------------------------------+

Scale: 200 tenants, 10M documents total, 500K queries/day
Isolation: Bridge model (5 enterprise silos + shared pool)
Ingestion: Async via Kafka, ~50K docs/day
```

### Example 3: Legal Document Analysis

```
  User: "Summarize indemnification clauses across all vendor contracts"
      |
      v
  +---------------------+
  | Agentic RAG Planner |
  +---------------------+
      |
      | Plan: 1. Find all vendor contracts
      |        2. Extract indemnification clauses
      |        3. Synthesize comparison
      |
      v
  +---------------------+    +-------------------+
  | Step 1: Metadata    |--->| Filter: doc_type  |
  | Search              |    | = "vendor_contract"|
  +---------------------+    +-------------------+
      |                            |
      | 47 contracts found         |
      v                            v
  +---------------------+    +-------------------+
  | Step 2: Section     |--->| Filter: section   |
  | Retrieval           |    | = "indemnification"|
  +---------------------+    +-------------------+
      |                            |
      | 43 relevant sections       |
      v                            |
  +---------------------+         |
  | Step 3: Long Context|<--------+
  | Synthesis           |
  | (load 43 sections   |
  |  into 1M context)   |
  +---------------------+
      |
      v
  Comparative summary with
  per-contract citations
```

---

## System Design Interview Angle

### Q: Design a RAG system that serves 10,000 queries per second across 500 tenants with a p99 latency of 2 seconds.

**Strong Answer:**

I would design this in four layers.

**Layer 1: Routing and Caching.** A query router classifies each incoming query (direct LLM, simple RAG, complex RAG). A three-tier cache (exact match, semantic cache, document cache) handles roughly 40-50% of traffic. This means only 5,000-6,000 QPS actually hit the retrieval pipeline.

**Layer 2: Retrieval.** I would use the bridge isolation model: the top 20 enterprise tenants get dedicated indexes (silo), and the remaining 480 share a pooled index with mandatory tenant_id filtering. Retrieval runs hybrid search (vector + BM25) in parallel, with Reciprocal Rank Fusion to merge results. The vector database cluster is sharded by tenant tier and replicated for read throughput.

**Layer 3: Generation.** A tiered model strategy routes simple queries to a small model and complex queries to a larger model. This keeps average cost low while maintaining quality for hard queries. Per-tenant rate limiting prevents noisy neighbors.

**Layer 4: Observability.** Every query produces a trace with latency breakdowns, retrieval scores, and cost. Nightly quality checks sample 500 queries and evaluate faithfulness and relevancy. Alerts fire if p95 latency exceeds 3 seconds or faithfulness drops below 0.85.

**Cost estimate**: 10K QPS is 864M queries a day, so the per-request math dominates everything. With 50% cache hits, 432M generations a day remain. At 2.6K input and 300 output tokens per request and October 2026 list prices, a 70/30 split between GPT-6 Luna (~$0.0004 per request) and a $2 / $10 mid-tier model such as Claude Sonnet 5.5 or GPT-6 Sol (~$0.008) costs about $120K plus $1.06M, roughly **$1.2M a day** before prompt caching of the shared system prompt. That number drives the real design decisions: cache the static prompt prefix (0.1x reads or less), push the cache-hit rate up, negotiate committed-use pricing, and self-host the small tier, since about 300M small-model requests a day is far past the point where a dedicated GPU fleet beats a per-token API.

### Q: How do you handle the case where a RAG system retrieves irrelevant documents but the LLM generates a plausible-sounding answer anyway?

**Strong Answer:**

This is the most dangerous RAG failure mode because it produces confident-sounding hallucinations grounded in real (but irrelevant) documents. I would address it at three points:

First, at the retrieval stage, implement a relevance grader, a classifier (or LLM call) that scores each retrieved chunk against the query. If all chunks score below a threshold, the system should either escalate to a web search (Corrective RAG pattern) or respond with "I don't have enough information" rather than generating from weak context.

Second, at the generation stage, use constrained prompting that instructs the model to explicitly state when evidence is insufficient. Include a confidence score in the output and route low-confidence answers to human review.

Third, in monitoring, track the correlation between retrieval scores and user feedback. If users are giving thumbs-down on queries where retrieval scores were high, the reranker or the chunking strategy is likely the root cause. Log the full trace (query, retrieved chunks, generated answer, user feedback) so you can debug specific failure cases.

### Q: Your RAG system's costs have tripled over the last month with no increase in query volume. How do you diagnose and fix this?

**Strong Answer:**

I would investigate in this order:

First, check the **cache hit rate**. If it has dropped, that means more queries are hitting the full pipeline. Common causes: a semantic cache threshold change, cache invalidation running too aggressively after a data update, or a shift in query distribution that does not match the cached queries.

Second, check the **model routing distribution**. If the query classifier is routing more queries to the expensive large model, that alone can triple costs. Look at whether query complexity has shifted or if the classifier's behavior has drifted.

Third, check for **retrieval thrash** in agentic RAG paths. If the Corrective RAG loop is retrying more often (maybe because of stale embeddings or degraded retrieval quality), each query makes multiple retrieval and generation calls. The trace logs will show the average number of iterations per query.

Fourth, check the **embedding pipeline**. If documents are being re-embedded unnecessarily (duplicate ingestion, no deduplication), embedding costs can spike.

The fix depends on the root cause, but common interventions are: tune the semantic cache threshold, implement a cost ceiling per query to force cheaper fallback paths, fix embedding staleness to reduce corrective retrieval loops, and add deduplication to the ingestion pipeline.

---

## References

- Asai et al. "Self-RAG: Learning to Retrieve, Generate, and Critique" (2024)
- Yan et al. "Corrective Retrieval Augmented Generation (CRAG)" (2024)
- Shi et al. "RAGRouter: Learning to Route Queries to Multiple RAL Models" (2025)
- Redis. "RAG at Scale: How to Build Production AI Systems in 2026"
- Anthropic. "1M Token Context Window General Availability" (March 2026)
- RAGAS Framework. "Context Precision, Recall, Faithfulness, and Relevancy Metrics"
- AWS. "Multi-Tenant RAG with Amazon Bedrock Knowledge Bases" (2025)
- Microsoft. "Design a Secure Multitenant RAG Inferencing Solution" (2025)
- Microsoft. "What's new in Azure AI Search" (agentic retrieval, 2026-08-01-preview)
- Cloudflare. "AI Search is now generally available" and AI Search limits and pricing (October 2026)
- Pinecone. "Pinecone Nexus is generally available" (August 2026)
- Qdrant. "Qdrant 1.19" release notes (August 2026)
- pgvector. CHANGELOG, 0.8.7 (October 2026)

---

*Previous: [RAG Evaluation Patterns](13-rag-evaluation-patterns.md) | Next: [Data Engineering for AI](15-data-engineering-for-ai.md)*
