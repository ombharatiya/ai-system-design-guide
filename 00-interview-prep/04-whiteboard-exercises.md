# Whiteboard Exercises for AI System Design

This chapter provides detailed walkthroughs of system design exercises commonly asked in AI-focused interviews. Each exercise includes the full problem statement, a structured solution approach, and discussion points that distinguish strong candidates.

## Table of Contents

- [Exercise 1: Enterprise RAG System](#exercise-1-enterprise-rag-system)
- [Exercise 2: Customer Support Chatbot](#exercise-2-customer-support-chatbot)
- [Exercise 3: Code Review Assistant](#exercise-3-code-review-assistant)
- [Exercise 4: Document Processing Pipeline](#exercise-4-document-processing-pipeline)
- [Exercise 5: Real-Time Content Moderation](#exercise-5-real-time-content-moderation)
- [Exercise 6: Multi-Tenant AI Platform](#exercise-6-multi-tenant-ai-platform)
- [Exercise 7: Semantic Search at Scale](#exercise-7-semantic-search-at-scale)
- [Exercise 8: Evaluation Pipeline for a Production LLM Product](#exercise-8-evaluation-pipeline-for-a-production-llm-product)
- [Exercise 9: Memory and State for a Long-Running Agent](#exercise-9-memory-and-state-for-a-long-running-agent)
- [Exercise 10: Inference Batching Service on a Single GPU](#exercise-10-inference-batching-service-on-a-single-gpu) ⭐ *NEW*
- [Tips for Whiteboard Exercises](#tips-for-whiteboard-exercises)

---

## Exercise 1: Enterprise RAG System

### Problem Statement

Design a RAG-based knowledge assistant for a large enterprise with the following requirements:

- 10 million documents from multiple sources (SharePoint, Confluence, Google Drive, internal wikis)
- 50,000 employees with role-based access
- Documents update continuously
- Must respect document permissions at query time
- Sub-3 second response time for 95% of queries
- Support for multiple languages (English, Spanish, Mandarin)

### Time Allocation (35 minutes)

| Phase | Time | Focus |
|-------|------|-------|
| Clarification | 3 min | Scope, priorities, constraints |
| High-level architecture | 7 min | Components and data flow |
| Data pipeline | 8 min | Ingestion, chunking, indexing |
| Query pipeline | 8 min | Retrieval, generation, permissions |
| Reliability and scale | 5 min | Failure handling, scaling |
| Evaluation | 4 min | Metrics and monitoring |

### Solution Walkthrough

#### Clarification Questions

```
1. What is the document size distribution? (PDFs, wikis, code?)
2. How often do permissions change? (Impacts caching strategy)
3. Is conversation history required or single-turn Q&A?
4. What is the accuracy bar? (Can we say "I don't know"?)
5. Are there compliance requirements? (Audit, data residency)
```

#### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          DATA PLANE                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────────┐│
│  │   Connectors │───▶│   Processor  │───▶│       Vector Database        ││
│  │ (SP,GD,Conf) │    │ (chunk,embed)│    │  (Pinecone/Qdrant/Weaviate)  ││
│  └──────────────┘    └──────────────┘    └──────────────────────────────┘│
│                                                      ▲                   │
│                                                      │ sync              │
│  ┌──────────────────────────────────────────────────┼───────────────────┐│
│  │                    Permission Service            │                   ││
│  └──────────────────────────────────────────────────┼───────────────────┘│
└─────────────────────────────────────────────────────┼───────────────────┘
                                                      │
┌─────────────────────────────────────────────────────┼───────────────────┐
│                          QUERY PLANE                │                   │
│  ┌──────────────┐    ┌──────────────┐    ┌─────────┴──────┐             │
│  │    User      │───▶│  Query API   │───▶│   Retriever    │             │
│  │  Interface   │    │              │    │ (+ perm filter)│             │
│  └──────────────┘    └──────────────┘    └────────┬───────┘             │
│                             │                      │                     │
│                             ▼                      ▼                     │
│                      ┌──────────────┐    ┌──────────────┐               │
│                      │   Reranker   │◀───│   Chunks     │               │
│                      └──────┬───────┘    └──────────────┘               │
│                             │                                            │
│                             ▼                                            │
│                      ┌──────────────┐    ┌──────────────┐               │
│                      │  Generator   │───▶│   Response   │               │
│                      │    (LLM)     │    │  + Citations │               │
│                      └──────────────┘    └──────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Data Pipeline Deep Dive

**1. Connectors:**
```
Each source has a dedicated connector:
- SharePoint: Graph API with delta sync
- Confluence: REST API with webhooks
- Google Drive: Drive API with push notifications

Connector responsibilities:
- Fetch document content and metadata
- Track change events (create, update, delete)
- Extract permissions from source system
- Normalize to common document schema
```

**2. Document Schema:**
```json
{
  "doc_id": "uuid",
  "source": "sharepoint|confluence|gdrive",
  "source_id": "original_id_in_source",
  "title": "string",
  "content": "string",
  "content_type": "pdf|html|docx|md",
  "language": "en|es|zh",
  "permissions": {
    "users": ["user_id_1", "user_id_2"],
    "groups": ["group_id_1"],
    "visibility": "private|internal|public"
  },
  "metadata": {
    "author": "string",
    "created_at": "timestamp",
    "updated_at": "timestamp",
    "path": "folder/path"
  }
}
```

**3. Chunking Strategy:**
```
Given mixed document types, use adaptive chunking:

- Markdown/HTML: Semantic chunking by headers
- PDFs: Layout-aware chunking using document AI
- Wiki pages: Section-based chunking

Chunk parameters:
- Target size: 512 tokens
- Overlap: 50 tokens
- Preserve: headers, tables, code blocks

Each chunk inherits parent document permissions.
```

**4. Embedding:**
```
Multilingual requirement suggests:
- Model: Cohere Embed 5 (embed-v5.0-pro: 100+ languages, 128K
  context, $0.12/1M text tokens)
- Alternatives: voyage-4-large ($0.12/1M) or gemini-embedding-2;
  self-hosted Nemotron 3 Embed 8B (NVIDIA reports #1 on RTEB
  Multilingual) if text cannot leave the VPC
- Decide with a bake-off on real Spanish and Mandarin queries,
  not leaderboard averages

Backfill cost: 50M chunks × ~512 tokens ≈ 26B tokens, about $3K
at $0.12/1M. Re-embedding is cheap; the re-index time is what
you plan around.

Batch embedding:
- Process in batches of 100 chunks
- Rate limit handling with exponential backoff
- Store embedding with chunk in vector DB
```

**5. Vector Database Choice:**
```
Pinecone or Qdrant for this scale.

Selection criteria:
- Metadata filtering: Critical for permissions
- Scale: 10M docs × 5 chunks = 50M vectors
- Hybrid search: Needed for keyword queries. Pinecone's full-text
  search (GA September 2026) ranks by one scoring type per query,
  so run BM25 and vector searches separately and fuse client-side
  with RRF

Schema:
- Vector: embedding
- Metadata: doc_id, chunk_id, language, permissions, source
```

#### Query Pipeline Deep Dive

**1. Permission Resolution:**
```python
def get_user_permissions(user_id: str) -> PermissionSet:
    """
    Resolve all documents user can access.
    Returns set of:
    - Direct user grants
    - Group memberships expanded
    - Public document access
    
    CACHED with 5-minute TTL since permissions change infrequently.
    """
    cache_key = f"permissions:{user_id}"
    if cached := cache.get(cache_key):
        return cached
    
    perms = permission_service.resolve(user_id)
    cache.set(cache_key, perms, ttl=300)
    return perms
```

**2. Retrieval with Filtering:**
```python
def retrieve(query: str, user_id: str, top_k: int = 20) -> List[Chunk]:
    perms = get_user_permissions(user_id)
    
    # Detect language for query
    lang = detect_language(query)
    
    # Build permission filter
    # User can see: public docs, their own, or groups they belong to
    filter = {
        "$or": [
            {"visibility": "public"},
            {"users": {"$in": [user_id]}},
            {"groups": {"$in": perms.groups}}
        ]
    }
    
    # Optional: boost same-language content
    if lang != "en":
        filter["language"] = lang
    
    results = vector_db.search(
        query_embedding=embed(query),
        top_k=top_k,
        filter=filter
    )
    return results
```

**3. Reranking:**
```
Rerank top-20 to get top-5 with a cross-encoder.
Model: a hosted 32K-context reranker, Cohere Rerank 5 (Pro $0.05,
Fast $0.02 per 1M tokens) or Voyage rerank-3, or self-hosted
bge-reranker-v2-m3 (multilingual, open weights); verify on Spanish
and Mandarin queries before choosing
Latency budget: ~150ms (matches the p95 budget below)
```

**4. Generation:**
```python
def generate(query: str, chunks: List[Chunk], user_id: str) -> Response:
    context = format_chunks_with_citations(chunks)
    
    prompt = f"""You are a knowledge assistant for [Company].
Answer the question using ONLY the provided context.
If the context does not contain the answer, say "I could not find information about that in our knowledge base."
Always cite sources using [1], [2] format.

CONTEXT:
{context}

QUESTION: {query}
"""
    
    response = llm.generate(
        prompt=prompt,
        model="claude-sonnet-5-5",  # pinned ID; the fallback route is another vendor
        effort="low",               # API default is "high"; unset, it eats the latency budget
    )
    
    return format_with_source_links(response, chunks)
```

No `temperature` here: Claude API version `2026-10-21` returns a 400 for any sampling parameter on every model, even at its default value, and GPT-6 Astra does not accept them. Grounding comes from the prompt, retrieval and citation checks, not from sampling settings.

#### Scaling and Reliability

**Latency Budget (p95 < 3s):**
```
Permission resolution:   50ms  (cached)
Embedding:              100ms
Vector search:          100ms
Reranking:              150ms
LLM generation:        1500ms
Network/overhead:       100ms
─────────────────────────────
Total:                 2000ms (buffer for P95)
```

**Scaling Considerations:**
```
- Vector DB: Sharded by source or hash
- Embedding service: Horizontal scale, stateless
- LLM calls: Multiple providers for redundancy
- Cache: Redis cluster for permissions and responses
```

**Failure Handling:**
```
- Vector DB down: Return cached results + degraded warning
- LLM down: Fallback to secondary provider
- Rate limiting: Queue with backpressure
- Embedding service: Batch retries with circuit breaker
```

#### Evaluation Approach

**Offline Metrics:**
```
- Retrieval: Precision@5, Recall@5, MRR
- Generation: RAGAS (faithfulness, relevance)
- End-to-end: Answer correctness on test set
```

**Online Metrics:**
```
- User feedback: Thumbs up/down
- Query reformulation rate: User rephrasing indicates failure
- Citation click-through: Are sources useful?
```

**Monitoring:**
```
- Latency dashboards by percentile
- Permission filter hit rate
- Empty result rate by source
- Cost per query
```

---

## Exercise 2: Customer Support Chatbot

### Problem Statement

Design an AI-powered customer support system for an e-commerce company:

- Handle 10,000 conversations per day
- Access to product catalog (1M products), order history, FAQs
- Goal: Resolve 70% of tickets without human handoff
- Support order lookup, returns, product questions
- Multilingual support (3 languages)
- Integration with existing Zendesk ticketing

### Solution Highlights

**Key Architectural Decisions:**

1. **Agent Architecture with Flow Control:**
```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   ┌─────────┐     ┌─────────────┐     ┌─────────────┐   │
│   │ Intake  │────▶│  Classify   │────▶│   Router    │   │
│   └─────────┘     └─────────────┘     └──────┬──────┘   │
│                                              │           │
│         ┌────────────────┬──────────────────┼───────┐   │
│         ▼                ▼                  ▼       ▼   │
│   ┌───────────┐   ┌───────────┐   ┌───────────┐ ┌─────┐ │
│   │Order Flow │   │Product Q&A│   │ Returns   │ │Human│ │
│   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘ └─────┘ │
│         │               │               │               │
│         └───────────────┴───────────────┘               │
│                         │                               │
│                   ┌─────▼─────┐                         │
│                   │  Response │                         │
│                   │ Generator │                         │
│                   └───────────┘                         │
└─────────────────────────────────────────────────────────┘
```

2. **Tool Design:**
```python
tools = [
    {
        "name": "lookup_order",
        "description": "Look up order details by order ID or customer email",
        "parameters": {
            "order_id": "optional string",
            "email": "optional string"
        }
    },
    {
        "name": "search_products",
        "description": "Search product catalog",
        "parameters": {
            "query": "string",
            "category": "optional string",
            "price_range": "optional tuple"
        }
    },
    {
        "name": "create_return",
        "description": "Initiate a return for an order",
        "parameters": {
            "order_id": "string",
            "reason": "string",
            "items": "list of item IDs"
        }
    },
    {
        "name": "escalate_to_human",
        "description": "Transfer to human agent",
        "parameters": {
            "reason": "string",
            "priority": "low|medium|high"
        }
    }
]
```

3. **Escalation Criteria:**
```
Escalate to human when:
- Customer explicitly requests human
- Sentiment is highly negative (detected by classifier)
- Issue involves payment disputes
- Agent confidence is low after 2 attempts
- Complex multi-order issues
- Refund above threshold amount
```

4. **Integration Pattern:**
```
Zendesk integration:
- Webhook receives new tickets
- AI handles via API
- Resolution → close ticket
- Escalation → assign to queue with context summary
- All interactions logged to ticket timeline
```

---

## Exercise 3: Code Review Assistant

### Problem Statement

Design a code review assistant for a development platform:

- Reviews pull requests automatically
- Provides specific, actionable feedback
- Respects repository style guides and conventions
- Can suggest code fixes
- Integration with GitHub/GitLab
- Handles 50,000 PRs per day

### Solution Highlights

**Key Technical Choices:**

1. **Context Assembly:**
```
For each changed file, assemble context:
- The diff (changed lines)
- Full file content (for understanding)
- Related files (imports, tests, types)
- Repository conventions (.eslintrc, .editorconfig)
- Previous review comments (learn from feedback)
```

2. **Review Categories:**
```python
review_types = [
    "bug_risk",           # Potential bugs
    "security",           # Security issues
    "performance",        # Performance concerns
    "maintainability",    # Code quality
    "style",              # Style guide violations
    "test_coverage"       # Missing tests
]
```

3. **Model Selection:**
```
Primary: Claude Sonnet 5.5 or GPT-6.1 Sol (both $2/$10 per 1M);
  pick with a bake-off on a few hundred past PRs whose real
  review outcomes you know
Escalation: Claude Opus 5.5 ($4/$20) for large or
  security-sensitive diffs
Fallback: the other vendor's mid-tier model, with its own
  tested prompt

Specialized models:
- Security scanning: CodeQL + LLM review
- Style: Linters + LLM explanation

Cost check (50K PRs/day, ~30K input tokens of assembled context
and ~2K output tokens per PR, $2/$10 model):
  50K × (30K × $2/1M + 2K × $10/1M) = 50K × $0.08
  = $4,000/day ≈ $120K/month before caching
Levers: cache repository conventions and shared files as a
prefix, skip generated and vendored files, set effort per route
(thinking tokens bill as output), escalate selectively.
```

4. **Output Format:**
````markdown
## Review Summary

### Critical Issues (must fix)
- **Line 45**: SQL injection vulnerability in user query
  ```python
  # Instead of:
  query = f"SELECT * FROM users WHERE id = {user_id}"
  # Use:
  query = "SELECT * FROM users WHERE id = ?"
  cursor.execute(query, (user_id,))
  ```

### Suggestions (consider fixing)
- **Line 78-82**: This loop could be simplified using list comprehension
...
````

5. **Latency Strategy:**
```
Target: Review ready within 2 minutes of PR creation

Strategy:
- Queue PR for processing
- Parallel processing of files
- Stream results as available
- Cache repository conventions
```

6. **Approval Authority:**
The assistant comments; by default it does not approve. Since October 28, 2026, Copilot code review approvals are generally available on GitHub: when an admin turns them on (they are off by default), a Copilot approval counts toward the repository's required approvals, admins choose which file paths Copilot may approve, and pushing new commits dismisses the approval. So "agent approves agent" is now a supported configuration, not a hypothetical. If you enable it or anything similar, limit it to low-risk paths, never let an agent approve a PR an agent wrote, and dismiss the approval on every new push. Expect this separation-of-duties follow-up.

---

## Exercise 4: Document Processing Pipeline

### Problem Statement

Design a document processing pipeline for financial services:

- Process 100,000 documents per day (invoices, contracts, forms)
- Extract structured data with 99% accuracy
- Handle PDFs, scanned documents, handwritten notes
- HIPAA/SOC2 compliance
- Human review for low-confidence extractions

### Solution Highlights

**Pipeline Architecture:**

```
┌────────┐   ┌───────────┐   ┌────────────┐   ┌────────────┐
│ Ingest │──▶│ Classify  │──▶│  Extract   │──▶│  Validate  │
└────────┘   └───────────┘   └────────────┘   └────────────┘
                                                     │
                                     ┌───────────────┼───────────────┐
                                     ▼               ▼               ▼
                              ┌──────────┐   ┌──────────┐   ┌──────────┐
                              │ Auto-pass│   │  Review  │   │  Reject  │
                              └──────────┘   └──────────┘   └──────────┘
```

**Key Components:**

1. **Document Classification:**
```
Fine-tuned classifier on document types:
- Invoice, Contract, Receipt, Form, ID, Other

Model: LayoutLMv3 or fine-tuned ViT
Confidence threshold: 0.95 for auto-routing
```

2. **Extraction Strategy:**
```
Tiered extraction based on document type:

Tier 1: Document AI (Textract, Azure Document Intelligence, or a
parsing API such as Cohere parse-v5.0 at $1.50 per 1,000 pages)
- Good for structured forms
- Fast and cheap
- Returns confidence scores

Tier 2: Vision LLM (Claude Opus 5.5, GPT-6.1 Sol, Gemini 3.8 Flash)
- Fallback for complex layouts
- Better for unstructured text
- More expensive

Combine outputs and cross-validate.

Pin model IDs and re-run the extraction eval on a schedule, not
only on ID changes: OpenAI fixed an image-encoding bug that had
degraded image understanding in GPT-6 Sol and GPT-6 Luna on
September 25, 2026, without changing either ID.
```

3. **Validation Rules:**
```python
validation_rules = {
    "invoice": [
        ("total", lambda x: x > 0, "Total must be positive"),
        ("date", lambda x: parse_date(x), "Invalid date format"),
        ("vendor_id", lambda x: regex_match(x, TAX_ID_PATTERN), "Invalid tax ID"),
        ("line_items", lambda x: sum(i.amount for i in x) == total, "Line items must sum to total")
    ],
    "contract": [
        ("parties", lambda x: len(x) >= 2, "Contract must have at least 2 parties"),
        ("effective_date", lambda x: parse_date(x), "Invalid date"),
        ("signature_present", lambda x: x == True, "Signature required")
    ]
}
```

4. **Human Review Interface:**
```
Reviewer sees:
- Original document image
- Extracted fields with confidence scores
- Validation errors highlighted
- Suggested corrections from LLM
- One-click approval or field-level corrections
```

5. **Compliance Measures:**
```
HIPAA/SOC2 requirements:
- All documents encrypted at rest (AES-256)
- TLS 1.3 in transit
- Audit log for all access and changes
- PHI detection and masking
- Retention policies enforced
- Access controls with MFA
- Model-provider retention checked per model, not per vendor:
  Claude Opus 5.5 and Sonnet 5.5 are available with zero data
  retention, while Fable 5.1 requires 30-day retention unless
  Anthropic expressly authorizes ZDR
```

---

## Exercise 5: Real-Time Content Moderation

### Problem Statement

Design a content moderation system for a social platform:

- 1 million posts per day (text, images, video)
- Latency requirement: under 500ms for posts to be visible
- Detect: hate speech, violence, adult content, spam
- Appeal workflow for false positives
- Support 10 languages

### Solution Highlights

**Architecture Pattern: Multi-Stage Cascade**

```
         ┌───────────────────────────────────────────┐
         │              Fast Filters                 │
         │   (regex, blocklist, hash matching)       │
         └─────────────────┬─────────────────────────┘
                           │ Pass 95%
                           ▼
         ┌───────────────────────────────────────────┐
         │            ML Classifiers                 │
         │   (text: BERT, image: CLIP, video: X3D)   │
         └─────────────────┬─────────────────────────┘
                           │ Uncertain 5%
                           ▼
         ┌───────────────────────────────────────────┐
         │            LLM Analysis                   │
         │   (context-aware, nuanced decisions)      │
         └─────────────────┬─────────────────────────┘
                           │ Still uncertain 0.5%
                           ▼
         ┌───────────────────────────────────────────┐
         │            Human Review                   │
         └───────────────────────────────────────────┘
```

**Key Design Decisions:**

1. **Latency Optimization:**
```
Target: 500ms total

Stage 1 (Fast): 20ms
- Regex patterns
- Known hash matching (PhotoDNA)
- Blocklist lookup

Stage 2 (ML): 80ms
- Batched inference on GPU
- Small specialized models
- Parallel text/image processing

Stage 3 (LLM): 400ms (async for borderline)
- Only 5% of content reaches here
- Used for nuanced decisions
```

2. **Threshold Strategy:**
```python
class ModerationDecision:
    BLOCK = "block"          # High confidence violation
    ALLOW = "allow"          # High confidence safe
    LIMIT = "limit"          # Reduce distribution
    REVIEW = "human_review"  # Queue for human

thresholds = {
    "hate_speech": {
        "block": 0.95,
        "limit": 0.80,
        "review": 0.60
    },
    "adult_content": {
        "block": 0.98,  # Higher threshold, legal implications
        "limit": 0.90,
        "review": 0.70
    }
}
```

3. **Appeal Workflow:**
```
1. User submits appeal
2. Content queued for human review
3. Different reviewer than original (blind review)
4. Decision logged with reasoning
5. If overturned:
   - Content restored
   - Original decision added to training data as negative
   - Model retrained periodically
```

---

## Exercise 6: Multi-Tenant AI Platform

### Problem Statement

Design a multi-tenant AI platform (AI-as-a-Service):

- Serve 500+ enterprise customers
- Each customer has their own documents and models
- Complete data isolation between tenants
- Per-tenant usage tracking and billing
- Different pricing tiers with different capabilities
- SOC2 compliance required

### Solution Highlights

**Tenant Isolation Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                         API Gateway                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │   Auth → Tenant Context → Rate Limit → Route             │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Tenant-Aware Service Layer                    │
│                                                                  │
│  All operations scoped to tenant_id from context                │
│  - Retrieval filters by tenant                                  │
│  - Cache keys prefixed by tenant                                │
│  - Audit logs include tenant                                    │
└─────────────────────────────────────────────────────────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  Shared Vector  │ │  Shared LLM     │ │  Shared Object  │
│  DB (filtered)  │ │  (no tenant     │ │  Storage        │
│                 │ │   data in prompt│ │  (tenant paths) │
│  tenant_id in   │ │   history)      │ │                 │
│  all metadata   │ │                 │ │  s3://bucket/   │
└─────────────────┘ └─────────────────┘ │  {tenant_id}/   │
                                        └─────────────────┘
```

**Critical Isolation Points:**

```python
class TenantContext:
    tenant_id: str
    user_id: str
    tier: str  # "starter" | "pro" | "enterprise"
    
    def __enter__(self):
        # Set tenant context for all downstream calls
        _tenant_context.set(self)
        
    def __exit__(self, *args):
        _tenant_context.set(None)

# Middleware ensures tenant context on every request
@middleware
def enforce_tenant_context(request, call_next):
    tenant_id = extract_tenant_from_token(request.headers["Authorization"])
    with TenantContext(tenant_id=tenant_id, ...):
        verify_tenant_access(tenant_id, request.path)
        response = call_next(request)
        add_tenant_to_audit_log(tenant_id, request, response)
    return response
```

**Billing and Usage Tracking:**

```python
usage_schema = {
    "tenant_id": "string",
    "timestamp": "datetime",
    "operation": "embed|retrieve|generate",
    "model": "string",
    "tokens_in": "int",
    "tokens_out": "int",
    "region": "string",       # residency route; priced higher, see below
    "latency_ms": "int",
    "cost_cents": "decimal"
}

# Real-time usage aggregation
async def track_usage(tenant_id: str, operation: Usage):
    # Append to time-series DB
    await timeseries.write("usage", {
        "tenant_id": tenant_id,
        **operation.dict()
    })
    
    # Update real-time counter for rate limiting
    await redis.incr(f"usage:{tenant_id}:{today()}", operation.tokens)
```

**Residency as a priced tier:**

Enterprise tenants that need in-region processing cost more to serve. Anthropic charges 1.1x for US-only inference (`inference_geo`) on Claude 4.6 and later models, OpenAI adds 10% on its data-residency endpoints for models released since March 5, 2026, and regional Claude endpoints on Bedrock and Google Cloud carry about a 10% premium over global ones (on Google-managed Vertex, current models such as Opus 5.5 and Sonnet 5.5 get only `us` or `eu` multi-region residency at that premium, not a single region; see [Pricing and Costs](../02-model-landscape/03-pricing-and-costs.md)). Route only residency-bound tenants to pinned regions, record the route in every usage row, and price it into the tier instead of paying the premium on all traffic.

---

## Exercise 7: Semantic Search at Scale

### Problem Statement

Design a semantic search system for an e-commerce site:

- 50 million products
- 100 million queries per day
- P99 latency under 100ms
- Support filters (price, category, brand, ratings)
- Personalization based on user history
- Real-time inventory updates

### Solution Highlights

**Key Challenge: 100ms at 100M queries/day**

```
100M queries/day = 1,157 QPS average
Peak: 5,000-10,000 QPS

At 100ms latency, need:
- Edge caching
- Pre-computed embeddings
- Optimized retrieval
- Minimal LLM involvement
```

**Architecture:**

```
┌────────────────────────────────────────────────────────────┐
│                         CDN/Edge                            │
│              (Cache popular queries: ~30% hit)              │
└─────────────────────────────┬──────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────┐
│                      Query Service                          │
│  1. Embed query (cached embeddings for common queries)      │
│  2. Retrieve candidates (ANN search)                        │
│  3. Apply filters (post-filter or hybrid)                   │
│  4. Personalize ranking                                     │
│  5. Return results                                          │
└─────────────────────────────┬──────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────┐
│                    Vector Database Cluster                  │
│  - Sharded by category (reduce search space)                │
│  - HNSW index with ef_search tuned for speed                │
│  - Metadata filtering with roaring bitmaps                  │
└────────────────────────────────────────────────────────────┘
```

**Latency Budget:**

```
Edge cache check:    5ms
Embedding lookup:   10ms (cached) or 30ms (compute)
Vector search:      30ms
Filtering:          10ms
Personalization:    10ms
Serialization:      10ms
Network overhead:   25ms
─────────────────────
Total:              100ms target (with cache hit)
```

**Hybrid Search Strategy:**

```python
def search(query: str, filters: dict, user_id: str) -> List[Product]:
    # Determine search strategy based on query
    if is_keyword_heavy(query):
        # "nike air max 90 size 10"
        sparse_weight = 0.7
        dense_weight = 0.3
    else:
        # "comfortable running shoes for flat feet"
        sparse_weight = 0.3
        dense_weight = 0.7
    
    # Parallel retrieval
    dense_results = vector_db.search(embed(query), top_k=100, filter=filters)
    sparse_results = elastic.search(query, top_k=100, filter=filters)
    
    # Reciprocal rank fusion
    combined = rrf_merge(
        [dense_results, sparse_results],
        weights=[dense_weight, sparse_weight]
    )
    
    # Personalization boost
    personalized = apply_user_preferences(combined, user_id)
    
    return personalized[:20]
```

**Real-Time Updates:**

```
Product updates (price, inventory) flow:
1. Change event published to Kafka
2. Consumer updates vector DB metadata
3. Search reflects change within seconds

Reindexing (description changes):
1. Full re-embed required
2. Run as async job
3. Swap index when complete
```

---

## Exercise 8: Evaluation Pipeline for a Production LLM Product

### Problem Statement

"Your company ships an AI assistant used by 50,000 daily users. Leadership wants to ship model and prompt changes weekly without quality regressions. Design the evaluation pipeline: offline evals, CI gating, judge calibration, and production monitoring. Budget: the eval system itself should cost under 2% of inference spend."

### Clarifying Questions to Ask

- What does "quality" mean for this product? (Task success, faithfulness, tone, safety?)
- Do we have any labeled data today, or are we starting from zero?
- What is the change velocity? (Prompts daily, models monthly?)
- What is the cost of a shipped regression? (Support tickets, churn, regulatory exposure?)
- Who consumes eval results? (Engineers gating PRs, PMs tracking quality, execs tracking trend lines?)

### Solution Walkthrough

**High-level architecture:**

```
                    ┌────────────────────────────────────────────┐
                    │              EVAL PIPELINE                  │
                    │                                             │
  Prompt/model PR ──► CI runner ── dev set (visible, ~200 cases) │
                    │     │                                       │
                    │     ├── held-out set (CI-only, ~300 cases,  │
                    │     │    rotated quarterly)                  │
                    │     │                                       │
                    │     └── gate: pass/fail vs baseline ──► merge│
                    │                                             │
  Production ──────► sampler (1-5% of traffic)                   │
                    │     │                                       │
                    │     ├── LLM-judge scoring (async, cheap)    │
                    │     ├── human-graded slice (weekly, ~100)   │
                    │     └── outcome metrics (thumbs, escalation)│
                    │                                             │
                    └──── dashboards + regression alerts ─────────┘
```

**1. Dataset strategy (the part most candidates skip):**

```
Golden dataset composition:
- 40% sampled from real production traces (stratified by intent cluster)
- 30% known-hard cases from past incidents and complaints
- 20% adversarial cases (injection attempts, edge formats)
- 10% canary cases that must never change behavior

Split: dev (iterate freely) / held-out (CI-only, never inspected)
Refresh: quarterly, sampled from recent traffic; archive old sets
```

**2. Scoring design:**

- Binary pass/fail per dimension (faithfulness, completeness, tone, safety), not 1-5 scales. Binary decisions are reproducible; Likert drifts.
- LLM-as-judge for scale: a cheap model (GPT-6 Luna, Gemini 3.8 Flash, Claude Haiku 5.5) with a rubric and few-shot anchors per dimension. Because the dimensions are binary, a decision model (such as OpenAI's Decisions API; see [Decision Models](../RESEARCH-RADAR.md#16-decision-models-typed-answers-with-probabilities)) is a cheaper option for the clear-cut ones. It gives no rationale, and its probabilities need calibrating against the human-graded slice like any judge.
- Judge calibration: monthly agreement check against the human-graded slice. Judge-human agreement is itself a dashboard metric; below 85% agreement, the judge prompt gets fixed before any product conclusions are drawn.

**3. CI gating:**

```python
def eval_gate(pr_results, baseline_results):
    # Hard gates: any safety regression blocks merge
    if pr_results.safety_pass_rate < baseline_results.safety_pass_rate:
        return Block(reason="safety regression")
    # Soft gates: quality within noise band
    delta = pr_results.task_success - baseline_results.task_success
    if delta < -0.02:                  # 2pt regression threshold
        return Block(reason=f"quality drop {delta:.1%}")
    if pr_results.held_out_success < pr_results.dev_success - 0.05:
        return Warn(reason="possible dev-set overfitting")
    return Pass()
```

**4. Production monitoring:**

- Sampled judge scoring on live traffic (1-5%), trended daily.
- Outcome linkage: complaint rate, thumbs-down rate, and escalation rate plotted against eval scores on the same dashboard. Divergence between eval trend and outcome trend triggers an eval review.
- Drift alarms: input distribution shift (intent cluster mix), score drop by segment, and judge agreement decay.

**5. Cost control:**

```
50K DAU, ~3 requests each = 150K requests/day
Sample 2% = 3K judged requests/day
Judge cost: ~1K tokens per judgment on a $1/1M model = ~$3/day
Weekly human slice: 100 cases x 3 min reviewer time
CI runs: 500 cases x ~2K tokens per PR = under $2 per PR
Total: well under the 2% budget; the human slice is the
dominant cost and it is what keeps the judge honest.
```

**6. Tooling and portability:**

Keep datasets, rubrics, judge prompts and gate thresholds in your own repo, and treat hosted eval platforms as replaceable runners. OpenAI's hosted Evals shuts down November 30, 2026 (OpenAI points users to Promptfoo, which is now part of OpenAI and remains MIT-licensed), and LangSmith SaaS keeps extended-retention traces for at most 180 days from September 14, 2026. Quarter-over-quarter trend lines need a store you control.

### What Distinguishes Strong Candidates

- They design the dataset before the scorer; a great judge on a stale dataset measures nothing.
- They treat the judge as a system component with its own eval (calibration vs humans), not as ground truth.
- They name eval gaming risks unprompted: dev-set overfitting, judge sycophancy, metric narrowing, silent exclusion of hard cases.
- They keep the held-out set and reference answers unreachable from the system under test. Agents with tool access find answer keys: a September 2026 audit (arXiv 2609.34262) judged 24% to 73% of passing SWE-Bench Pro v1.0 trajectories unearned, depending on the model, mostly through git history that exposed reference solutions.
- They link offline scores to production outcomes instead of celebrating green dashboards.
- They quantify the eval budget and place the expensive human grading where it has the most leverage.

---

## Exercise 9: Memory and State for a Long-Running Agent

### Problem Statement

"Design the memory system for a personal AI assistant that works with a user over months: it should remember preferences and facts, learn from past sessions, recall relevant history at the right moments, and forget what it should not keep. Sessions can run for hours. Support 1M users."

### Clarifying Questions to Ask

- What memory failures hurt most: forgetting something important, or recalling something wrong or stale?
- Privacy constraints? (GDPR deletion, per-user isolation, regulated data?)
- Is memory per-user only, or shared organizational knowledge too?
- Latency budget for recall on each turn?
- How long do individual sessions run? (Determines in-session vs cross-session design.)

### Solution Walkthrough

**The memory hierarchy:**

```
┌──────────────────────────────────────────────────────────┐
│ L1 Working memory: the context window                     │
│   Current session, tool results, scratchpad               │
│   Managed by compaction + just-in-time loading            │
├──────────────────────────────────────────────────────────┤
│ L2 Episodic memory: what happened                         │
│   Past session summaries, trajectories, outcomes          │
│   Store: vector DB, retrieved by similarity + recency     │
├──────────────────────────────────────────────────────────┤
│ L3 Semantic memory: what is true                          │
│   Extracted facts and preferences with provenance         │
│   Store: structured records or knowledge graph            │
├──────────────────────────────────────────────────────────┤
│ L4 Procedural memory: how to do things                    │
│   Learned workflows, per-user playbooks (skills)          │
│   Store: versioned files, loaded on demand                │
└──────────────────────────────────────────────────────────┘
```

**1. In-session (L1): context engineering, not storage.**
Compaction at a window threshold (summarize history, keep recent artifacts), just-in-time loading of large content by reference, and a structured scratch note the agent re-reads after compaction. A session that runs hours never holds its whole transcript in context.

**2. Write path (the hard part):**

```
Session ends (or hits checkpoint)
    │
    ├── Summarize episode → L2 (embedding + metadata:
    │     timestamp, topics, outcome, sentiment)
    │
    └── Fact extraction pass → candidate facts
          │
          ├── Deduplicate against existing L3
          ├── Conflict check: contradicts a stored fact?
          │     ├── Newer + higher confidence → supersede (keep old
          │     │     version with valid_to timestamp)
          │     └── Ambiguous → store as candidate, confirm with
          │           user at next natural opportunity
          └── Importance filter: discard chit-chat, keep
                preferences, commitments, corrections
```

The conflict path matters most: "user moved from Madrid to Lisbon" must supersede, not coexist. Bitemporal records (valid_from, valid_to) make supersession auditable and reversible.

**3. Read path:**

- Each turn builds a recall query from the current intent, retrieves top-k from L2 (similarity + recency + importance weighting) and matching facts from L3.
- A relevance gate drops weak matches rather than stuffing them in; wrong memories poison the response worse than missing ones.
- Recall budget: a few hundred tokens of memory per turn, never a transcript dump.
- Measure memory-on against memory-off. Even correct, relevant memories can hurt: MemTrapBench (arXiv 2608.20202, August 2026) found every memory strategy it tested underperformed the no-memory setting on its trap tasks, where correctly stored memories pulled the agent into reasoning fixation or distorted beliefs about the current task.

**4. Forgetting and privacy:**

- Decay: episodic entries lose retrieval weight over time unless reinforced by access.
- Consolidation: a periodic job merges related episodes into summaries (many "billing question" episodes become one pattern note). Vendors now ship this as an offline "dreaming" pass: Anthropic's Dreams (research preview) and Mem0's Dream (Pro and Enterprise plans).
- GDPR deletion: per-user partition keys everywhere; deletion removes L2/L3/L4 rows and invalidates caches. Hard isolation per user; memory is never shared across tenants by similarity.

**5. Scale sketch (1M users):**

```
Storage: ~thousands of L2 entries + hundreds of L3 facts per
  active user; vector DB partitioned by user_id (metadata
  filter or namespace per tenant tier)
Write path: async after session close; queue + worker pool
Read path: p95 < 150ms recall (ANN on a per-user slice is small)
Cost: extraction pass on session close is the main LLM cost;
  use a cheap model (GPT-6 Luna, Gemini 3.8 Flash, or DeepSeek
  V4.1-Flash) with a strict schema
```

### What Distinguishes Strong Candidates

- They separate in-session context management from cross-session memory instead of conflating them.
- They spend time on the write path (extraction, dedup, conflict supersession) rather than only on retrieval.
- They treat wrong recall as worse than no recall and design the relevance gate accordingly.
- They name memory poisoning as a security surface: untrusted content written today and trusted tomorrow needs provenance tags and review gates.
- They know the build-vs-buy options and how they have moved, while still being able to design from primitives. Mem0's open-source SDK has been ADD-only since v2.0 (April 2026), with merge and supersede moved to its paid Dream feature, so a conflict path like the one above stays in your code. Letta archived its MemGPT-era server in August 2026 and now keeps memory as git-backed Markdown files with no vector index by default. Zep remains a graph-based option.

---

## Exercise 10: Inference Batching Service on a Single GPU

### Problem Statement

"Design the batching layer for an LLM inference service on one GPU. Users wait synchronously for results, and a batch holds at most 100 sequences. Explain how requests are admitted, grouped, scheduled and returned, then how the design changes when traffic outgrows the GPU."

Interview-prep guides report variants of this prompt from Anthropic system design rounds (Aced, formerly Exponent, September 2026; candidate-reported, not confirmed by the company). It tests whether you understand serving from the inside rather than only calling an API.

### Clarifying Questions to Ask

- Is this autoregressive generation or a single forward pass (embeddings, classification, scoring)? The answer changes the whole design.
- What are the prompt and output length distributions? Prefill-heavy RAG prompts and decode-heavy chat stress different things.
- Which SLO matters: time to first token (TTFT), inter-token latency, or end-to-end, and at what percentile?
- Do users stream, or wait for the full response?
- Are there priority classes or tenants? Can clients cancel?

### Solution Walkthrough

**1. Two regimes, two batchers:**

```
Single forward pass (embed, classify, score):
  Dynamic batching. Flush when the batch reaches 100 OR the oldest
  request has waited max_wait (e.g. 5-10 ms). Added latency is
  bounded by max_wait; throughput comes from fuller batches.

Autoregressive generation:
  Static batching wastes the GPU: every slot waits for the longest
  sequence. Use continuous (iteration-level) batching: after each
  decode step, finished sequences leave and queued ones join.
```

**2. Architecture:**

```
Client ──► API (auth, validation, max_tokens cap)
              │
              ▼
        Admission control ── 429 + Retry-After when the queue is full
              │
              ▼
        Request queue (priority, deadline)
              │
              ▼
        Scheduler (runs every step)
          - admit while slots < 100 AND free KV blocks suffice
          - chunked prefill: split long prompts so they do not
            stall in-flight decodes
          - on KV pressure, preempt the lowest priority sequence
            (swap to host memory or recompute later)
              │
              ▼
        GPU worker: one forward pass per step over the mixed batch
              │
              ▼
        Detokenize and stream tokens back (SSE); free KV blocks on
        completion or client disconnect
```

**3. The binding constraint is KV memory, not the 100-slot cap:**

```
Llama 3.1 8B in BF16:
  2 × 32 layers × 8 KV heads × 128 head_dim × 2 bytes
  = 128 KiB of KV per token
80 GB GPU: ~16 GB of weights, leave ~50 GiB for KV ≈ 400K tokens

Average context 2K tokens → ~200 sequences fit: the 100 cap binds
Average context 8K tokens → ~50 sequences fit: memory binds first
```

So the scheduler admits by free KV blocks (PagedAttention-style paging), reserves headroom for output growth, and treats 100 per batch as a ceiling, not a target.

**4. Synchronous users and backpressure:**

- Stream tokens so the user's wait is TTFT, not full generation time.
- Bound the queue by request count and by queued tokens, and shed load early with a 429. An unbounded queue turns a traffic spike into timeouts for everyone. vLLM added admission-control flags for exactly this in v0.29 (`--max-num-queued-reqs`, `--max-num-queued-tokens`).
- Cancel on client disconnect so abandoned requests stop holding KV blocks.
- Cap `max_tokens` per request; one runaway generation can pin memory for minutes.

**5. Metrics:**

TTFT and inter-token latency at p50 and p99, queue wait, batch occupancy, KV utilization, preemption rate, and **goodput**: the request rate that meets both the TTFT and inter-token SLOs. Peak tokens per second at a full batch means little if p99 TTFT misses the SLO.

**6. Beyond one GPU:**

- Replicas behind a prefix- and KV-aware router (llm-d's Endpoint Picker, NVIDIA Dynamo's router), so requests sharing a system prompt land where that prefix is already cached.
- Disaggregated prefill and decode pools when prompts are long. The KV handoff is not free (about 3 GB for a 10K-token prompt on Llama 3.1 70B in BF16, roughly 65 ms at 400 Gb/s), so it pays only when it raises goodput.
- Speculative decoding (EAGLE-3, MTP, block-diffusion drafters such as DFlash) to cut inter-token latency; benchmark it at your real concurrency, because the gain shrinks as batches fill.

### What Distinguishes Strong Candidates

- They ask whether the workload is generation or a single forward pass before drawing anything.
- They size KV memory with numbers and admit by memory, not by slot count.
- They bound the queue and say what the client sees under overload.
- They report goodput at an SLO, not peak throughput.
- They say they would run vLLM or SGLang in production, since both already implement continuous batching and paged KV, and use the interview to show they understand what those engines do.
- They treat the engine as a shared-fate domain. Inputs get validated before they reach the GPU: one unauthenticated request with a negative token ID to vLLM's embeddings endpoint crashed the engine for every client until restart (CVE-2026-93592, fixed in v0.28.0). And a shared prefix cache is a cross-tenant side channel, so salt cache keys per tenant (vLLM's `cache_salt`) and keep the engine patched (v0.30.0 fixed a path that dropped the salt and reopened a prefix-cache membership oracle, and v0.32.0 is the current security floor, v0.33.0 for hosts that load model repositories they did not vet).

**Related prompts to practice** (candidate-reported from Anthropic and OpenAI loops via the prep vendor Aced, formerly Exponent; not confirmed by the companies): a GPU credit or quota accounting system, peer-to-peer distribution of a large file (think model weights) from one bandwidth-limited source to thousands of machines, and a token-generation service at 100,000 requests per second. The same moves apply: clarify the workload, find the binding resource, bound the queue, and measure at an SLO.

---

## Tips for Whiteboard Exercises

### Drawing Tips

1. **Start with boxes and labels** before connecting with arrows
2. **Use consistent notation**: rectangles for services, cylinders for databases, arrows for data flow
3. **Label data on arrows**: what flows between components
4. **Leave space** for additions as you discuss

### Common Patterns to Know

| Pattern | When to Use | Draw As |
|---------|-------------|---------|
| Load balancer + service fleet | Any scaled service | LB → multiple boxes |
| Queue + workers | Async processing | Queue → worker pool |
| Cache layer | Read-heavy, latency-sensitive | Diamond before service |
| CDC/streaming | Real-time updates | Kafka/stream icon |
| Sidecar | Cross-cutting concerns | Small box attached to service |

### Phrases That Signal Strong Candidates

- "Before I design this, let me understand the scale..."
- "The tradeoff here is..."
- "In production, we would also need..."
- "One failure mode to consider is..."
- "Let me walk you through the latency budget..."
- "For evaluation, I would measure..."

### Time Management

- Do not spend more than 5 minutes on clarification
- Draw the complete high-level picture before deep diving
- Leave time for reliability and evaluation
- Check in with the interviewer on focus areas

### Rehearse With Someone Else Drawing the Clock

Solo practice catches knowledge gaps; it does not catch pacing, rambling, or the habit of defending a choice instead of weighing it. Run at least one exercise end to end with another person playing the interviewer. If you want an experienced interviewer and written feedback, book a 1:1 mock with Om on [EngineBogie](https://enginebogie.com/u/ombharatiya) or [Topmate](https://topmate.io/ombharatiya).

---

*See also: [Question Bank](01-question-bank.md) | [Answer Frameworks](02-answer-frameworks.md) | [Common Pitfalls](03-common-pitfalls.md)*
