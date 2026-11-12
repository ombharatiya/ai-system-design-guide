# Advanced Retrieval Patterns

Beyond the basics, production RAG systems use specialized patterns to handle complex query-document gaps. These patterns are the "secret sauce" of high-precision search and are increasingly bundled into managed RAG offerings.

## Table of Contents

- [Query Decomposition (Multi-Query)](#query-decomposition-multi-query)
- [Hypothetical Document Embeddings (HyDE)](#hypothetical-document-embeddings-hyde)
- [Contextual Retrieval (The Anthropic Pattern)](#contextual-retrieval-the-anthropic-pattern)
- [Iterative Document Enrichment](#iterative-document-enrichment)
- [In-Context Reranking](#in-context-reranking)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Query Decomposition (Multi-Query)

Complex user queries are often "Compound Queries."
- **User**: "Compare our Q3 vs Q4 revenue and explain the drop."
- **Decomposition**:
  1. "Q3 Revenue"
  2. "Q4 Revenue"
  3. "Reasons for Q4 revenue variance"
- **Implementation**: Use an LLM to generate these 3 sub-queries, search the DB for ALL of them, and aggregate the context.

---

## Hypothetical Document Embeddings (HyDE)

Queries are short; documents are long. This "Asymmetry" causes retrieval failure.
- **Pattern**: 
  1. Take the user query.
  2. Ask the LLM: "Write a 1-paragraph hypothetical answer to this."
  3. **Embed the hypothetical answer** instead of the query.
- **Why?**: The hypothetical answer is in the same "Vector neighborhood" as the real documents, leading to much higher recall.

---

## Contextual Retrieval (The Anthropic Pattern)

Published by Anthropic in September 2024, this pattern solves **Context Dilution**.

- **The Problem**: A chunk might say "It costs $200," but without the header, we don't know "It" is a "Widget-X."
- **The Pattern**: During ingestion, for every 300-token chunk, have an LLM write a 50-100 token context string (e.g., "This chunk is about the pricing for Widget-X in the North American market") and prepend it before both embedding and BM25 indexing.
- **Benefit**: Anthropic measured a 35% drop in top-20 retrieval failures from contextual embeddings alone, 49% with contextual BM25 added, and 67% with a reranker on top.
- **2026 alternative**: Contextualized chunk embedding models (voyage-context-4) produce document-aware chunk vectors with no per-chunk LLM call, but only help the dense side. Full treatment in [Contextual Retrieval](10-contextual-retrieval.md).

---

## Iterative Document Enrichment

Instead of just storing the raw document, we store "Enriched" meta-data.
- **Summary**: Store a 1-paragraph summary of the document.
- **Q&A Generation**: Generate 5 questions this document answers and embed those *with* the document.
- **Status**: Many high-end RAG systems embed generated **"Questions"** alongside the chunk text, so a short user question matches a stored question instead of a long answer passage.

---

## In-Context Reranking

With 1M-token windows now standard on current models (Claude Opus 5.5 and Sonnet 5.5, the GPT-6 and GPT-6.1 families at 1.05M, Gemini 4 Argon, and even the fast tier with Gemini 3.8 Flash and Claude Haiku 5.5, though Haiku bills 5x once a prompt passes 100K tokens), **Rank-by-Context** is a viable pattern.
1. Retrieve Top 100 docs.
2. Put all 100 in the context window.
3. Ask the model: "Read these 100 docs and identify the 5 most relevant. Then, use those 5 to answer."
- **Win**: This utilizes the model's **Long Context Reasoning** to perform reranking without needing a separate Cross-Encoder model.
- **Cost check**: 100 docs of 500 tokens is 50K input tokens per query: about $0.10 on a $2-per-1M model such as Claude Sonnet 5.5 or GPT-6 Sol, versus about $0.0025 for the same tokens on Voyage rerank-3 ($0.05 per 1M), a 40x gap before output tokens. Reserve it for low-volume, reasoning-heavy queries, and keep the prompt under long-context price cliffs (100K on Claude Haiku 5.5, cached tokens included; 272K input on OpenAI; 200K on xAI), or use a flat-priced model such as Claude Sonnet 5.5.

---

## Interview Questions

### Q: Why is HyDE (Hypothetical Document Embedding) risky for some applications?

**Strong answer:**
HyDE relies on "Hallucinating" a baseline answer to find real data. If the user's query describes something non-existent or logically impossible, the LLM will still generate a hypothetical answer. This can pull in "Incorrect but Semantically Similar" data from the database, reinforcing the model's initial hallucination. The standard mitigation is a **Hybrid approach**: retrieve once with the real query (Keyword) and once with the HyDE query, then use **RRF** to combine them.

### Q: What is the "Asymmetric Retrieval" problem?

**Strong answer:**
Asymmetric retrieval refers to the fact that user queries are usually short (3-10 words) while document chunks are long (300-500 words). These inhabit different statistical distributions in the vector space, leading to "Distance Bias." High-performance systems solve this using **Asymmetric Encoders** (separate query and document prefixes or models) or **Query Expansion** (HyDE) to "inflate" the query into a document-like distribution.

The asymmetry is now also a cost lever. Voyage 4 and Cohere Embed 5 ship model families in one shared space, so you can index documents with the large model and encode queries with the small one. Qdrant's Constella research preview goes further: a fixed 400M-parameter Stella document index with a 34.5M query encoder that keeps about 91% of BEIR-15 nDCG@10 at 12x lower CPU latency (vendor-reported). At high QPS, query encoding is on the hot path and document encoding is not, so spend the model budget on the document side.

---

## References
- Gao et al. "Precise Zero-Shot Dense Retrieval without Relevance Labels" (HyDE, 2023/2024)
- [Anthropic. "Introducing Contextual Retrieval" (Sep 2024)](https://www.anthropic.com/news/contextual-retrieval)
- LlamaIndex. "Query Transformation Cookbook" (2025)

---

*Previous: [Agentic RAG](08-agentic-rag.md) | Next: [Contextual Retrieval](10-contextual-retrieval.md)*
