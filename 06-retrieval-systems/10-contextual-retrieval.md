# Contextual Retrieval

Contextual Retrieval is an ingestion-time technique that solves the #1 cause of RAG failure: **chunks that lose meaning when separated from their source document**. Pioneered by Anthropic in late 2024, it is now a production standard for high-precision retrieval. Anthropic's own measurements show a 49% reduction in top-20 retrieval failures from contextual embeddings plus contextual BM25, and 67% when reranking is added.

## Table of Contents

- [The Problem: Context Dilution](#the-problem-context-dilution)
- [How Contextual Retrieval Works](#how-contextual-retrieval-works)
- [Contextual Embeddings](#contextual-embeddings)
- [Contextual BM25](#contextual-bm25)
- [The Full Pipeline: Hybrid + Reranking](#the-full-pipeline-hybrid--reranking)
- [Implementation Patterns](#implementation-patterns)
- [Cost Considerations](#cost-considerations)
- [Contextual Retrieval vs. Other Approaches](#contextual-retrieval-vs-other-approaches)
- [Production Architecture](#production-architecture)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Problem: Context Dilution

When we chunk documents for RAG, individual chunks lose the surrounding context that gives them meaning.

**Example of Context Dilution:**

```
Original Document: "Acme Corp Q3 2025 Financial Report"
  Section 4: Product Pricing

  "The Standard plan costs $200/month. The Enterprise
   plan includes SSO and audit logs for $800/month."

-------- After Chunking --------

Chunk 17: "It costs $200/month."
Chunk 18: "The Enterprise plan includes SSO and audit
           logs for $800/month."
```

**The problem with Chunk 17**: A user searching "How much does Acme Standard plan cost?" will likely miss this chunk because it contains no mention of "Acme," "Standard," or "plan." The embedding of "It costs $200/month" is semantically distant from the query.

**Insight**: Anthropic's research showed that traditional chunking causes a **5.7% retrieval failure rate** on the top-20 retrieved chunks. That means roughly 1 in 18 queries fails to retrieve the relevant information, even when it exists in the knowledge base.

---

## How Contextual Retrieval Works

The core idea is simple: **before embedding a chunk, prepend a short context string that explains what the chunk is about within the full document**.

```
┌──────────────────────────────────────────────────┐
│              TRADITIONAL CHUNKING                │
│                                                  │
│  Document ──► Split ──► Chunks ──► Embed ──► DB  │
│                                                  │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│              CONTEXTUAL RETRIEVAL                            │
│                                                              │
│  Document ──► Split ──► Chunks ──┐                           │
│                                  ├──► Contextualize ──►      │
│  Document (full) ───────────────┘    (LLM call per chunk)    │
│                                                              │
│  ──► Contextual Chunks ──► Embed ──► DB                      │
│                            + BM25 Index                      │
└──────────────────────────────────────────────────────────────┘
```

**The contextualization step** sends the full document + individual chunk to an LLM with this prompt:

```
<document>
{{WHOLE_DOCUMENT}}
</document>

Here is the chunk we want to situate within the whole document:
<chunk>
{{CHUNK_CONTENT}}
</chunk>

Please give a short succinct context to situate this chunk
within the overall document for the purposes of improving
search retrieval of the chunk. Answer only with the succinct
context and nothing else.
```

**Result for Chunk 17**:

```
Before: "It costs $200/month."

After:  "This chunk is from the Acme Corp Q3 2025 Financial
         Report, Section 4 on Product Pricing. It describes
         the cost of the Standard plan.
         It costs $200/month."
```

Now the embedding of this chunk contains "Acme," "Standard plan," and "Product Pricing": all the terms a user would naturally search for.

---

## Contextual Embeddings

Contextual Embeddings is the first sub-technique: embedding the contextualized chunk instead of the raw chunk.

### How It Improves Retrieval

| Scenario | Raw Chunk Embedding | Contextual Embedding |
|----------|--------------------|-----------------------|
| User asks about "Acme pricing" | Misses "It costs $200" | Matches "Acme...Standard plan...costs $200" |
| User asks about "SSO features" | Matches "SSO and audit logs" | Matches with added context of "Enterprise plan" |
| User asks about "Q3 financials" | No match (no mention of Q3) | Matches via prepended "Q3 2025 Financial Report" |

**Performance**: Contextual Embeddings alone reduce top-20 retrieval failure from **5.7% to 3.7%**, a **35% reduction** in retrieval failures.

### The Vector Space Shift

```
                    ▲ Dimension 2
                    │
                    │    ● "Acme pricing" (query)
                    │         \
                    │          \  close (contextual)
                    │           \
                    │            ● Contextualized chunk
                    │
                    │                          ● Raw chunk "It costs $200"
                    │                            (far from query)
                    │
                    └─────────────────────────────► Dimension 1
```

---

## Contextual BM25

The second sub-technique applies the same contextualization to create a **BM25 keyword index** over the enriched chunks.

### Why BM25 Still Matters

Dense embeddings excel at semantic similarity but fail on:
- **Exact terms**: Product IDs, version numbers, acronyms
- **Rare tokens**: Domain-specific jargon that embedding models under-represent
- **Proper nouns**: Company names, people, places

**Example**: A user searching "Widget-X pricing" would get zero BM25 matches on the raw chunk "It costs $200/month" because "Widget-X" never appears. With contextual BM25, the prepended context includes "Widget-X" as a keyword, enabling the BM25 match.

### Performance Gains (Cumulative)

| Configuration | Failure Rate | Reduction vs. Baseline |
|---------------|-------------|----------------------|
| Traditional embeddings (baseline) | 5.7% | -- |
| Contextual Embeddings only | 3.7% | 35% |
| Contextual Embeddings + Contextual BM25 | 2.9% | **49%** |
| Contextual Embeddings + Contextual BM25 + Reranking | 1.9% | **67%** |

**Takeaway**: The combination of contextual embeddings + contextual BM25 is the highest-leverage single change you can make to a RAG pipeline. Adding a reranker on top gets you to 67% fewer failures.

---

## The Full Pipeline: Hybrid + Reranking

The production-grade Contextual Retrieval pipeline has four stages:

```
┌─────────────────────────────────────────────────────────────────┐
│                     INGESTION PIPELINE                          │
│                                                                 │
│  1. Chunk documents (recursive, 300-500 tokens)                 │
│  2. For each chunk:                                             │
│     a. Send (full_doc + chunk) to LLM                           │
│     b. Get context string (50-100 tokens)                       │
│     c. Prepend context to chunk                                 │
│  3. Embed contextualized chunks ──► Vector DB                   │
│  4. Index contextualized chunks ──► BM25 Index                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     QUERY PIPELINE                              │
│                                                                 │
│  User Query                                                     │
│      │                                                          │
│      ├──► Vector Search (Top 50) ──┐                            │
│      │                             ├──► RRF Fusion (Top 25)     │
│      └──► BM25 Search (Top 50)  ──┘         │                   │
│                                             ▼                   │
│                                      Reranker (Top 5)           │
│                                             │                   │
│                                             ▼                   │
│                                     LLM Generation              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Reciprocal Rank Fusion (RRF) for Combining Results

The same RRF technique used in standard hybrid search applies here:

```
RRF_Score(doc) = sum( 1 / (k + rank_in_list) )
                 for each list where doc appears

k = 60 (standard smoothing constant)
```

---

## Implementation Patterns

### Pattern 1: Basic Contextual Retrieval (Python)

```python
import anthropic
from typing import List

client = anthropic.Anthropic()

CONTEXT_PROMPT = """<document>
{document}
</document>

Here is the chunk we want to situate within the whole document:
<chunk>
{chunk}
</chunk>

Please give a short succinct context to situate this chunk
within the overall document for the purposes of improving
search retrieval of the chunk. Answer only with the succinct
context and nothing else."""


def contextualize_chunk(
    full_document: str,
    chunk: str,
    model: str = "claude-haiku-5-5"  # small model: context strings are short and factual
) -> str:
    """Generate context for a single chunk."""
    response = client.messages.create(
        model=model,
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": CONTEXT_PROMPT.format(
                document=full_document,
                chunk=chunk
            )
        }]
    )
    # Read the text block, not content[0]: on models that think by default
    # (Haiku 5.5, Sonnet 5.5, Opus 5.5), a thinking block can come first.
    context = next(b.text for b in response.content if b.type == "text")
    return f"{context}\n\n{chunk}"


def process_document(document: str, chunks: List[str]) -> List[str]:
    """Contextualize all chunks in a document."""
    contextualized = []
    for chunk in chunks:
        ctx_chunk = contextualize_chunk(document, chunk)
        contextualized.append(ctx_chunk)
    return contextualized
```

### Pattern 2: Cost-Optimized with Prompt Caching

The biggest cost driver is sending the full document with every chunk. **Prompt Caching** solves this:

```python
def contextualize_with_caching(
    full_document: str,
    chunks: List[str],
    model: str = "claude-haiku-5-5"
) -> List[str]:
    """
    Use prompt caching so the full document is only
    processed once across all chunks.
    """
    results = []

    for chunk in chunks:
        response = client.messages.create(
            model=model,
            max_tokens=200,
            messages=[{
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": f"<document>\n{full_document}\n</document>",
                        "cache_control": {"type": "ephemeral"}
                    },
                    {
                        "type": "text",
                        "text": (
                            f"<chunk>\n{chunk}\n</chunk>\n\n"
                            "Please give a short succinct context to "
                            "situate this chunk within the overall "
                            "document for the purposes of improving "
                            "search retrieval of the chunk. Answer "
                            "only with the succinct context and "
                            "nothing else."
                        )
                    }
                ]
            }]
        )
        context = next(b.text for b in response.content if b.type == "text")
        results.append(f"{context}\n\n{chunk}")

    return results
```

**Cost Impact of Prompt Caching**: For a 10,000-token document split into 30 chunks, prompt caching cuts the document-input cost by roughly **85%**: one cache write at 1.25x the input price, then 29 reads at 0.1x, instead of 30 full-price reads.

Three operational traps erase that saving:
- **Interleaving and cold fan-out.** The default cache TTL is 5 minutes. Schedule all chunks of one document inside that window; a queue that spreads a document's chunks over hours turns every call into a cache write. Send the first call alone and fan out the rest after it returns: parallel first calls all miss and all pay the write.
- **Short documents.** The minimum cacheable prefix depends on the model: 512 tokens on Claude Haiku 5.5, Sonnet 5.5, Opus 5.5 and Fable 5.1, but 4,096 on Claude Haiku 4.5. A 3,000-token document silently never caches on Haiku 4.5; check `usage.cache_read_input_tokens` instead of assuming.
- **Long documents on Haiku 5.5.** Its $0.10/$0.50 price holds only while the whole prompt, cached document included, stays at or under 100K tokens. Above that the entire request bills $0.50/$2.50, so a 150K-token contract reprices every one of its chunk calls at 5x. Split very long documents into sections under the line, or budget the higher tier (still half of Haiku 4.5's list price).
- **Retired model IDs.** Dated model IDs retire: `claude-sonnet-4-20250514` left the Claude API on June 15, 2026, breaking every pipeline that hard-coded it. Keep the model ID in config, not code, and store it next to every generated context string.

### Pattern 3: Contextual Chunk Headers (Lightweight Alternative)

If LLM-based contextualization is too expensive, use **Contextual Chunk Headers (CCH)** as a deterministic alternative:

```python
def add_chunk_headers(
    document_title: str,
    section_hierarchy: List[str],
    chunk: str
) -> str:
    """
    Prepend document and section metadata to the chunk.
    No LLM call required: purely structural.
    """
    header_parts = [f"Document: {document_title}"]

    for i, section in enumerate(section_hierarchy):
        prefix = "  " * i
        header_parts.append(f"{prefix}Section: {section}")

    header = "\n".join(header_parts)
    return f"{header}\n\n{chunk}"


# Example usage:
contextualized = add_chunk_headers(
    document_title="Acme Corp Q3 2025 Financial Report",
    section_hierarchy=["Finance", "Product Pricing", "Standard Plan"],
    chunk="It costs $200/month."
)

# Result:
# Document: Acme Corp Q3 2025 Financial Report
#   Section: Finance
#     Section: Product Pricing
#       Section: Standard Plan
#
# It costs $200/month.
```

**When to use CCH vs. LLM Contextualization:**

| Factor | Chunk Headers (CCH) | LLM Contextualization |
|--------|--------------------|-----------------------|
| **Cost** | Free (no LLM calls) | About $0.50 (GPT-6 Luna or Claude Haiku 5.5, before thinking tokens) to $5 (Claude Haiku 4.5) per 1M document tokens with caching |
| **Quality** | Good for structured docs | Excellent for all docs |
| **Speed** | Instant | 0.5-2 s per call; throughput comes from concurrency |
| **Best for** | Markdown, HTML, PDFs with clear headers | Unstructured text, legal, medical |

---

## Cost Considerations

### Contextualization Costs

For a knowledge base of 10,000 chunks (400 tokens each) drawn from 500 documents of 8,000 tokens, with the document cached and about 500 uncached prompt tokens and 75 output tokens per call, at list prices on November 2, 2026:

| Option | Price per 1M (in / cache read / out) | Cost per Chunk | Total Cost | Notes |
|--------|--------------------------------------|----------------|------------|-------|
| **GPT-6 Luna** | $0.10 / $0.01 / $0.50 | ~$0.0002 | ~$2 | OpenAI caches automatically; 1.25x writes on the 30-minute cache, 2.5x on the 12-hour option |
| **Claude Haiku 5.5** | $0.10 / $0.01 / $0.50 (prompts up to 100K tokens) | ~$0.0002 | ~$2-3 + thinking | Adaptive thinking on by default (effort `medium`), so set low effort; its tokenizer counts ~30% more tokens than Haiku 4.5 for the same text (the ~$3 end of the range); 512-token minimum cacheable prefix |
| **Claude Haiku 4.5** | $1 / $0.10 / $5 | ~$0.002 | ~$21 | No thinking by default; 4,096-token minimum cacheable prefix |
| **Claude Sonnet 5.5** | $2 / $0.10 / $10 | ~$0.003 | ~$36 + thinking | Thinking is on by default; send `thinking: {type: "between_tools"}` or low effort for this job |
| **Claude Opus 5.5** | $4 / $0.20 / $20 | ~$0.007 | ~$70 + thinking | Thinking cannot be disabled; overkill for this task |
| **voyage-context-4** (no LLM call) | $0.12 embedding | ~$0.00005 | ~$0.50 | Replaces the embedding step; helps dense retrieval only (see below) |

**Best practice**: Use a small model for contextualization. The context strings are short and factual, so a frontier model adds cost and, on the newest Claude models, reasoning tokens (on by default, and impossible to disable on Opus 5.5) without a measurable quality gain. Combine with prompt caching, and run backfills through a Batch API (50% off list at both Anthropic and OpenAI) when they can wait.

Claude Haiku 5.5 (GA October 7, 2026) is the default pick for this job on Claude: a tenth of Haiku 4.5's price for prompts up to 100K tokens, and the same list price as GPT-6 Luna. It is not a drop-in swap. `temperature`, `top_p`, `top_k`, `budget_tokens` and assistant prefill all return a 400, the tokenizer change shifts every token count you budgeted, and the context strings it writes will differ from Haiku 4.5's, so re-run the retrieval golden set before re-contextualizing a corpus. Haiku 4.5 is still Active on the Claude API with no deprecation notice (Anthropic gives at least 60 days, so not before about January 2027), while Microsoft Foundry lists November 15, 2026 for it. Treat the contextualizer as a swappable dependency and store the model ID next to every context string.

### When to Use Contextual Retrieval

**Use it when:**
- Your corpus has fragmented documents where chunks lose meaning in isolation
- You have domain-specific jargon that embedding models struggle with
- Your retrieval failure rate exceeds 3-5%
- You can afford the one-time ingestion cost

**Skip it when:**
- Your chunks are already self-contained (e.g., FAQ pairs, product descriptions)
- Your corpus is tiny (< 100 chunks); just use long context instead
- You need real-time ingestion (< 1s per document) and cannot batch

---

## Contextual Retrieval vs. Other Approaches

| Approach | How It Works | Retrieval Improvement | Cost | Complexity |
|----------|-------------|----------------------|------|------------|
| **Naive Chunking** | Fixed-size splits, embed raw | Baseline | None | Low |
| **Chunk Headers (CCH)** | Prepend doc/section titles | 10-20% | None | Low |
| **Contextual Retrieval** | LLM-generated context per chunk | 35-49% | ~$2-45 per 10k chunks (small to mid model) | Medium |
| **Contextual + Reranking** | Above + cross-encoder rerank | 67% | Above, plus a per-query rerank fee | Medium-High |
| **Contextual chunk embeddings** | Embedder sees the whole document, emits one vector per chunk | Dense side only; not measured on Anthropic's benchmark | ~$0.50 per 10k chunks | Low |
| **HyDE** | Hypothetical doc generation at query time | 20-40% | Per-query LLM cost | Medium |
| **Parent-Child Chunking** | Embed children, retrieve parents | 15-30% | None | Medium |

**Key Distinction**: Contextual Retrieval is an **ingestion-time** technique (pay once), while HyDE is a **query-time** technique (pay per query). For high-volume systems, Contextual Retrieval amortizes much better.

### Contextual Retrieval vs. Late Chunking

**Late Chunking** (Jina, 2024) is a related but distinct approach:

```
Contextual Retrieval:
  Chunk ──► LLM adds context ──► Embed enriched chunk

Late Chunking:
  Full doc ──► Long-context embed model ──► Token embeddings
  ──► THEN chunk the token embeddings (preserving context)
```

Late Chunking requires a long-context embedding model (e.g., Jina v3) and avoids LLM calls entirely. It preserves context through the embedding model's attention mechanism rather than explicit text prepending. The tradeoff is that Late Chunking does not help BM25 search, only dense retrieval.

### Contextualized Chunk Embedding Models

The same idea now ships as a hosted embedding API, which cuts the ingestion cost in the table above by one to two orders of magnitude against a Claude contextualizer:

- **voyage-context-4** (Voyage AI, GA June 29, 2026): one vector per chunk that encodes the full document's context, built-in auto-chunking (`enable_auto_chunking=True`), native overlapping chunks, and transparent handling of documents longer than its 32K-token window. $0.12 per 1M tokens, down from $0.18 for voyage-context-3. Voyage reports gains of 2.08% at chunk level and 1.4% at document level over context-3 across 39 datasets (vendor-reported). Also served through MongoDB Atlas.
- **pplx-embed-v2-context** (Perplexity, Hugging Face, 9B and 0.6B, MIT, technical report October 28, 2026): encodes a document's chunks together, and the two sizes share a 1024-dimension space, so you can index with 9B and query with 0.6B. It replaced the September 25 preview (2,048 dimensions, Matryoshka to 1,024, native int8), whose card says its vectors should not be mixed with later releases, so re-embed anything you built on the preview.

| | LLM contextualization | Contextual chunk embeddings |
|---|---|---|
| **Ingestion cost (10k chunks)** | ~$2-45 plus embedding | ~$0.50, replaces embedding |
| **Helps BM25** | Yes (context text is indexed) | No |
| **Debuggable** | Yes, you can read the context string | No, context lives inside the vector |
| **Lock-in** | Context strings survive an embedder swap | Index tied to one vendor model; re-embed to switch |
| **Document edits** | Re-contextualize changed chunks | Re-embed the whole document |

**A sensible 2026 default**: contextual chunk embeddings for the dense side, deterministic chunk headers for the BM25 side, and LLM contextualization only for the document classes where both still miss (measure on your golden set before paying for it).

---

## Production Architecture

### Reference Architecture: Contextual RAG at Scale

```
┌─────────────────────────────────────────────────────────────────────┐
│                     INGESTION SERVICE                               │
│                                                                     │
│  Document Store ──► Chunker ──► Contextualization Queue             │
│                       │              │                              │
│                       │         ┌────┴────┐                         │
│                       │         │ Workers  │ (N parallel LLM calls) │
│                       │         │ + Cache  │                        │
│                       │         └────┬────┘                         │
│                       │              │                              │
│                       ▼              ▼                              │
│                  Raw Chunks    Contextualized Chunks                 │
│                       │              │                              │
│                       │         ┌────┴────┐                         │
│                       │         │ Embed + │                         │
│                       │         │ BM25    │                         │
│                       │         └────┬────┘                         │
│                       │              │                              │
│                       ▼              ▼                              │
│                  Metadata DB    Vector DB + BM25 Index               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     QUERY SERVICE                                   │
│                                                                     │
│  Query ──► [Vector Search] + [BM25 Search]                          │
│                    │               │                                │
│                    └───── RRF ─────┘                                │
│                           │                                         │
│                      Top 25 chunks                                  │
│                           │                                         │
│                      Reranker (Cohere Rerank 5, Voyage rerank-3,    │
│                      or a self-hosted cross-encoder)                │
│                           │                                         │
│                      Top 5 chunks                                   │
│                           │                                         │
│                      LLM Generation                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Scaling Considerations

| Concern | Solution |
|---------|----------|
| **Ingestion throughput** | Parallelize LLM calls (50-100 concurrent) with async workers |
| **Document updates** | Re-contextualize only changed chunks; store raw + context separately |
| **Cost at scale** | Small model + prompt caching, a document's chunks scheduled together; Batch API for backfills; or a contextual embedding model |
| **Quality monitoring** | Sample 1% of chunks and human-evaluate context quality |
| **Index consistency** | Update vector DB + BM25 index atomically per document |
| **Model churn** | Store the contextualizer model ID and prompt version with each context string. Existing strings stay valid when that model retires; sample the successor's output before mixing styles in one index |

---

## Interview Questions

### Q: Explain Anthropic's Contextual Retrieval. When would you use it and when would you skip it?

**Strong answer:**
Contextual Retrieval solves the "context dilution" problem in RAG. When documents are chunked, individual chunks lose the surrounding context that gives them meaning: a chunk saying "It costs $200" is useless without knowing *what* costs $200. The technique uses an LLM at ingestion time to generate a short context string (50-100 tokens) per chunk, explaining what that chunk is about within the document. This context is prepended to the chunk before embedding and BM25 indexing.

The key results: Contextual Embeddings alone reduce retrieval failures by 35%. Adding Contextual BM25 achieves 49% reduction. Adding a reranker reaches 67% reduction.

I would use it when chunks regularly lose meaning in isolation: legal contracts, financial reports, technical manuals. I would skip it when chunks are already self-contained (FAQs, product cards) or when the corpus is small enough for long-context RAG.

### Q: A knowledge base of 50,000 documents needs Contextual Retrieval. How do you manage the ingestion cost?

**Strong answer:**
Four strategies:
1. **Model selection**: Use a small, fast model (Claude Haiku 5.5 or GPT-6 Luna class, both $0.10 / $0.50) for contextualization. The output is short factual text, not creative writing. A frontier model adds cost without quality gain, and the newest Claude Opus and Sonnet models add reasoning tokens you cannot fully turn off; even on Haiku 5.5, set a low effort. Keep documents under Haiku 5.5's 100K-token price line or split them.
2. **Prompt caching**: Cache the full document text across all chunk contextualization calls, and schedule each document's chunks together so they land inside the cache TTL. For a 10,000-token document with 30 chunks, this cuts document-input cost by roughly 85%.
3. **Tiered approach**: Not every document needs LLM contextualization. For well-structured documents (Markdown, HTML with headers), use deterministic Contextual Chunk Headers (prepending doc title + section hierarchy) which is free. Reserve LLM contextualization for unstructured or ambiguous documents.
4. **Push context into the embedder**: Contextualized chunk embedding models such as voyage-context-4 ($0.12 per 1M tokens) produce document-aware chunk vectors with no per-chunk LLM call. At 50,000 documents of about 8,000 tokens (400M tokens), that is roughly $50 against about $200 to $300 with Haiku 5.5 or Luna and caching (about $2,000 on Haiku 4.5). The gap is smaller than it was, so the deciding factor is quality on your golden set more than price. The catch is that it only helps the dense side, so I would pair it with chunk headers for BM25 and keep LLM contextualization for the documents that still fail on the golden set.

### Q: How does Contextual Retrieval compare to HyDE for improving retrieval quality?

**Strong answer:**
They solve different sides of the same problem. Contextual Retrieval enriches **documents** at ingestion time (pay once), while HyDE enriches **queries** at search time (pay per query). For a system handling 10,000 queries/day against a 50,000-chunk corpus, Contextual Retrieval is dramatically cheaper because the ingestion cost is amortized. HyDE also has a hallucination risk: the hypothetical document might pull in wrong data. In practice, the strongest systems use both: Contextual Retrieval for ingestion enrichment and HyDE (or multi-query expansion) for complex queries that need query-side help.

---

## References
- Anthropic. "Contextual Retrieval" (September 2024)
- Jina AI. "Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding Models" (2024)
- Voyage AI. "voyage-context-3: Contextualized Chunk Embeddings" (2025)
- Voyage AI. "voyage-context-4" (June 2026)
- Perplexity. "pplx-embed-v2-context-9b" model card (Hugging Face, October 2026) and technical report (arXiv 2610.38854, October 28, 2026)
- NirDiamant. "RAG Techniques: Contextual Chunk Headers" (GitHub, 2024)

---

*Previous: [Advanced Retrieval Patterns](09-advanced-retrieval-patterns.md) | Next: [Late Interaction & ColBERT](11-late-interaction-colbert.md)*
