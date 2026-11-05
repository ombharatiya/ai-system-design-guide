# Chunking Strategies

Chunking is the process of splitting a document into discrete segments for retrieval. Production pipelines have moved beyond blind fixed-size splits to **structure-aware and semantic segments**, with newer techniques like late chunking and contextual prepending now in the mainstream toolkit.

## Table of Contents

- [The Retrieval-Context Tension](#the-retrieval-context-tension)
- [Recursive Structure Splitting](#recursive-structure-splitting)
- [Semantic Chunking](#semantic-chunking)
- [Hierarchical (Parent-Child) Chunking](#hierarchical-parent-child-chunking)
- [Late Chunking and Contextualized Chunk Embeddings](#late-chunking-and-contextualized-chunk-embeddings)
- [Content-Specific Strategies (Code, PDF, Tables)](#content-specific-strategies)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Retrieval-Context Tension

| Aspect | Small Chunks (100t) | Large Chunks (1000t) |
|--------|---------------------|----------------------|
| **Precision** | High (Exact match) | Low (Diluted) |
| **Context** | Poor (Broken sentences) | Rich (Surrounding info) |
| **Storage** | High (More vectors) | Low (Fewer vectors) |
| **Latency** | Low (Fast search) | High (Heavy retrieval) |

**Rule**: Smaller is better for *finding*, but larger is better for *thinking*. Use **Hierarchical Chunking** to get both.

---

## Recursive Structure Splitting

Instead of splitting at every 500 characters, we split at logical boundaries:
`[Double Newline] > [Single Newline] > [Period] > [Space]`.

**Best practice**: Use **Markdown-Aware Splitting**. If a document has `#` headers, ensure the header is prepended to *every* child chunk to preserve context (Contextual Chunking).

---

## Semantic Chunking

Semantic chunking uses an embedding model to detect "topic shifts."

1. Split text into individual sentences.
2. Group sentences as long as their embedding similarity stays above a threshold (e.g., 0.82).
3. If similarity drops, start a new chunk.

**Nuance**: Some pipelines replace the threshold with a **learned segmenter**: a small model scans the text and predicts a separator at every semantic break. It avoids hand-tuning a cosine cutoff per corpus, but published chunking benchmarks swing widely between fixed-size, recursive and semantic strategies, so measure recall on your own documents before paying for the extra model.

---

## Hierarchical (Parent-Child) Chunking

This is the industry standard for production RAG.

- **Process**: 
  1. Create "Parent" chunks of 1,500 tokens.
  2. Sub-divide each parent into 5 "Child" chunks of 300 tokens.
  3. **Index only the children**.
  4. At retrieval, if a child matches, **return the full parent context** to the LLM.
- **Why?**: The child is small and easy for the vector DB to match. The parent provides enough context for the LLM to actually reason correctly without "Broken Context" hallucinations.

---

## Late Chunking and Contextualized Chunk Embeddings

Hierarchical chunking fixes context at *read* time. The alternative is to fix it at *embed* time, so each chunk vector already knows which document it came from.

- **Late chunking** (Jina, 2024): run a long-context embedding model over the whole document, then pool token embeddings per chunk span. Every chunk vector is conditioned on the full document, with no extra LLM call.
- **Contextualized chunk embedding models** package this as an API. **voyage-context-4** (GA June 29, 2026; $0.12 per 1M tokens, 32K window, built-in auto-chunking) returns one document-aware vector per chunk; Voyage reports +2.08% chunk-level and +1.4% document-level retrieval over voyage-context-3 (vendor-reported). Perplexity's **pplx-embed-v2-context** family (9B and 0.6B, MIT, technical report October 28, 2026) encodes a document's chunks together, and the two sizes share a 1024-dimension space, so you can index with 9B and query with 0.6B. It replaced the September 25 preview, whose card says its vectors should not be mixed with later releases, so re-embed anything you built on the preview.

The tradeoff against Anthropic-style contextual prefixes: embedder-side context costs no per-chunk LLM call, but it only helps the dense side. BM25 still sees the bare chunk text, so keep header prepending for the sparse index. The full cost comparison is in [Contextual Retrieval](10-contextual-retrieval.md).

---

## Content-Specific Strategies

### 1. Code Chunking
- **Strategy**: Use AST (Abstract Syntax Tree) parsing.
- **Rule**: Never split a function mid-body. Keep imports and class declarations with their methods.

### 2. Table Chunking
- **Strategy**: Use Markdown formatting for tables.
- **Modern pattern**: "Summarized Tables." Store a natural language summary of the table in the vector DB, but return the full Markdown table to the LLM.

### 3. PDF/Layout Chunking
- **Strategy A, parse then chunk**: Use a layout-aware parser (Docling, MinerU, Cohere Parse, or an open parsing VLM) that emits reading-order blocks, with tables, figures and sidebars as separate typed elements. Chunk on those block boundaries so charts and sidebars never get mixed into body text.
- **Strategy B, skip text chunking**: Embed each page image directly with a late-interaction model (ColPali family). Layout survives because the page *is* the unit. See [Multi-Modal RAG](12-multimodal-rag.md).
- **Agent-facing pattern**: For agentic RAG, consider giving the agent stable block locators instead of pre-cut chunks. MinerU 4.0 (September 16, 2026) returns locators of the form `doc:{id}/tier:{tier}/page:{page}/block:{block}` and reads long documents progressively (the first 10 pages plus a continuation), so the agent navigates the document and citations point at a verifiable block.

---

## Interview Questions

### Q: Why is fixed-size chunking with overlap problematic for production systems?

**Strong answer:**
Fixed-size chunking is "content-blind." It frequently splits sentences mid-thought, breaks mathematical equations, and separates headers from their descriptive text. While "Overlap" (e.g., 10%) mitigates this by duplicating 10% of text across chunks, it doesn't solve the core issue: the model's attention is forced to reconstruct meaning from fragmented strings. Modern pipelines prefer **Semantic or Logical Chunking** because it ensures each vector represents a "Complete Semantic Unit," leading to significantly higher retrieval precision.

### Q: What is "Contextual Retrieval" (the Anthropic pattern)?

**Strong answer:**
Contextual Retrieval involves prepending a short, LLM-written context to every chunk before embedding and BM25 indexing it. For example, if a chunk is about "battery life," but it's from a manual for a "2025 Model X Drone," a line like `This chunk is from the Model X drone manual, battery section:` is added to the chunk. This ensures that the vector for "battery life" is influenced by the "Drone" context, preventing it from being accidentally retrieved for "phone battery" queries. Anthropic measured a 35% drop in top-20 retrieval failures from contextual embeddings alone, 49% when contextual BM25 is added, and 67% with reranking on top.

The cost is one LLM call per chunk at ingestion (cheap with prompt caching of the source document). The 2026 alternative is a contextualized chunk embedding model such as voyage-context-4, which gets document-aware vectors with no LLM call but does nothing for BM25. My default: header prepending for both indexes, an embedder-side context model for dense, and LLM contextualization only for the document types that still fail on the golden set.

---

## References
- [Anthropic. "Introducing Contextual Retrieval" (Sep 2024)](https://www.anthropic.com/news/contextual-retrieval)
- [Günther et al. "Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding Models" (2024)](https://arxiv.org/abs/2409.04701)
- [Voyage AI. "voyage-context-4" (June 2026)](https://blog.voyageai.com/2026/06/29/voyage-context-4/)
- [MinerU 4.0 release notes (Sep 2026)](https://github.com/opendatalab/MinerU/releases/tag/mineru-4.0.0-released)
- [Smith and Troynikov. "Evaluating Chunking Strategies for Retrieval" (Chroma, July 2024)](https://www.trychroma.com/research/evaluating-chunking)
- [LangChain. "Text splitter integrations" (docs)](https://docs.langchain.com/oss/python/integrations/splitters)
- [LlamaIndex. "Node Parser Modules" (docs, semantic and hierarchical parsers)](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/)

---

*Previous: [RAG Fundamentals](01-rag-fundamentals.md) | Next: [Embedding Models](03-embedding-models.md)*
