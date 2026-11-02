# Embeddings and Vector Spaces

Embeddings are dense vector representations of text that capture semantic meaning. They are foundational to RAG systems, semantic search, and many AI applications.

## Table of Contents

- [What Are Embeddings](#what-are-embeddings)
- [Embedding Model Architectures](#embedding-model-architectures)
- [Training Objectives](#training-objectives)
- [Distance Metrics](#distance-metrics)
- [Embedding Model Comparison](#embedding-model-comparison)
- [Matryoshka and Adaptive Dimensions](#matryoshka-and-adaptive-dimensions)
- [Late Chunking and Late Interaction](#late-chunking-and-late-interaction)
- [Quantization for Scale](#quantization-for-scale)
- [Practical Considerations (Batching, Caching)](#practical-considerations)
- [Embedding Drift and Versioning](#embedding-drift-and-versioning)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## What Are Embeddings

Embeddings map discrete text (words, sentences, documents) to continuous vector spaces where semantic similarity corresponds to geometric proximity.

**Key properties:**
- Similar meanings are close together
- Relationships can be encoded as vector operations (king - man + woman = queen)
- Enable efficient similarity search through approximate nearest neighbor algorithms

**Mental model:**
Think of embeddings as coordinates in a very high-dimensional space. Dimensionality (512 to 4096) provides expressiveness. Each dimension captures some aspect of meaning, though individual dimensions are not interpretable.

---

## Embedding Model Architectures

### Word Embeddings (Historical)

Early approaches embedded individual words:

| Model | Year | Approach | Limitation |
|-------|------|----------|------------|
| Word2Vec | 2013 | Skip-gram, CBOW | Static: "bank" same in all contexts |
| GloVe | 2014 | Co-occurrence matrix | Static |
| FastText | 2017 | Subword embeddings | Static, but handles OOV |

**Key limitation:** Same word gets same embedding regardless of context.

### Contextual Embeddings

Transformer-based models produce context-dependent embeddings:

```python
# Static embedding (Word2Vec)
embed("bank") = [0.1, 0.3, ...]  # Same vector always

# Contextual embedding (BERT)
embed("river bank") = [0.1, 0.3, ...]   # Geography sense
embed("bank account") = [0.5, 0.2, ...]  # Finance sense
```

### Sentence/Document Embeddings

For retrieval, we need to embed entire texts:

| Approach | Method | Pros | Cons |
|----------|--------|------|------|
| Mean pooling | Average token embeddings | Simple | Loses information |
| CLS token | Use [CLS] token embedding | Standard for BERT | May not capture full text |
| Last token | Use final token | Works for decoder models | Position bias |
| Trained pooling | Learn pooling weights | Better quality | Requires training |

Modern embedding models are trained specifically for sentence/document embedding, not just adapted from language models.

### Bi-Encoder Architecture

Standard retrieval embedding architecture:

```
Document -> Encoder -> Document Embedding
Query    -> Encoder -> Query Embedding

Similarity = cosine(doc_embedding, query_embedding)
```

**Properties:**
- Documents can be pre-computed and indexed
- Query embedding computed at query time
- O(1) similarity computation per document (with ANN)

### Cross-Encoder Architecture

Alternative that processes query and document together:

```
[Query, Document] -> Encoder -> Relevance Score
```

**Properties:**
- More accurate (sees both together)
- Cannot pre-compute: O(n) inference for n documents
- Used for reranking, not retrieval

Current rerankers take 32K-token inputs, so a candidate can be a long section or a short document instead of a 512-token passage: Cohere Rerank 5 (Pro $0.05, Fast $0.02 per 1M tokens), Voyage rerank-3 and rerank-3-lite (same $0.05 and $0.02), and open-weight options such as Qwen3-Reranker-8B and Qwen3-VL-Reranker (Apache 2.0, multimodal). See [Reranking Strategies](../06-retrieval-systems/06-reranking-strategies.md).

---

## Training Objectives

### Contrastive Learning

Most modern embedding models use contrastive learning:

```python
# Simplified contrastive loss
def contrastive_loss(anchor, positive, negatives):
    pos_sim = cosine_similarity(anchor, positive)
    neg_sims = [cosine_similarity(anchor, neg) for neg in negatives]
    
    # Push positive close, negatives far
    loss = -log(exp(pos_sim / tau) / 
                (exp(pos_sim / tau) + sum(exp(neg_sim / tau) for neg_sim in neg_sims)))
    return loss
```

**Key factors:**
- **Positive pairs:** Semantically similar texts (parallel sentences, query-document pairs)
- **Hard negatives:** Similar but not matching texts (BM25 retrieved non-relevant)
- **In-batch negatives:** Other batch items as negatives (efficient)

### Training Data Sources

| Source | Positive Pairs | Quality | Scale |
|--------|---------------|---------|-------|
| Parallel sentences | Translation pairs | High | Medium |
| Query-document | Search logs | High | Medium |
| Title-body | Document structure | Medium | Large |
| Paraphrase | NLI datasets | High | Small |
| Generated | LLM creates pairs | Variable | Large |

### Instruction-Tuned Embeddings

Recent models accept task instructions:

```python
# Instruction-tuned (e.g., E5, BGE)
query_embedding = embed("Represent this query for retrieval: What is RAG?")
doc_embedding = embed("Represent this document for retrieval: RAG combines...")
```

This improves performance by specifying the intended use.

---

## Distance Metrics

### Cosine Similarity

Most common for text embeddings:

```python
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

**Properties:**
- Range: [-1, 1] ([0, 1] when all components are non-negative)
- Measures angle, not magnitude
- Invariant to vector length

**When to use:** Default choice for text embeddings.

### Dot Product

```python
def dot_product(a, b):
    return np.dot(a, b)
```

**Properties:**
- Magnitude matters
- Unbounded range
- Equivalent to cosine for normalized vectors

**When to use:** When embeddings are already normalized, or magnitude is meaningful.

### Euclidean Distance

```python
def euclidean_distance(a, b):
    return np.linalg.norm(a - b)
```

**Properties:**
- Measures absolute difference
- Affected by magnitude
- For normalized vectors: sqrt(2 - 2 * cosine)

**When to use:** Rarely for text; more common for image embeddings.

### Metric Selection

| Metric | Vector Databases | Common Use |
|--------|------------------|------------|
| Cosine | Pinecone, Qdrant, Weaviate | Text embeddings |
| Dot Product | All major DBs | Normalized embeddings |
| Euclidean | All major DBs | Image, multimodal |

---

## Embedding Model Comparison

### Hosted Models (October 2026)

| Model | Dimensions | Max Input | Modalities | Cost / 1M tokens | Notes |
|-------|------------|-----------|------------|------------------|-------|
| OpenAI text-embedding-3-large | 3072 (Matryoshka) | 8,192 | Text | $0.13 | OpenAI has shipped no newer embedding model |
| OpenAI text-embedding-3-small | 1536 (Matryoshka) | 8,192 | Text | $0.02 | Cheap baseline |
| Google gemini-embedding-2 | 128-3,072 (768 / 1,536 / 3,072 recommended) | 8,192 text tokens; up to 6 images, 120 s video, audio, 6-page PDFs | Multimodal, interleaved | $0.20 text (Google pricing page) | Stable since April 2026; replaces text-embedding-004 (shut down) and text-only gemini-embedding-001 (shuts down May 14, 2028) |
| Cohere embed-v5.0-pro / -fast | 256-2,048 (Matryoshka) | 128K | Text, image, fused text+image | $0.12 / $0.08 text; $0.40 image | September 30, 2026; Pro and Fast share one embedding space |
| Voyage voyage-4-large / voyage-4 / voyage-4-lite | 1024 default (256-2,048) | 32K | Text | $0.12 / $0.06 / $0.02 | January 2026; the family shares one embedding space |
| Voyage voyage-code-4 | 1024 default (256-2,048) | 32K | Code | $0.12 | August 13, 2026; trained on natural-language queries mined from pull requests, for agent-style code search |
| Voyage voyage-context-4 | - | 32K window; longer documents handled | Text, contextualized chunks | $0.12 | One vector per chunk that encodes full-document context |

**Reading benchmark claims:**
- **Prefer RTEB** (from the MTEB team, October 2025) over MTEB-only rankings. RTEB mixes open datasets with private held-out sets that the maintainers evaluate, so models cannot be tuned toward all of it. NVIDIA reports Nemotron 3 Embed 8B at #1 on RTEB Multilingual with 78.5 (as of July 15, 2026).
- **MTEB leadership is contested.** Qwen3-Embedding-8B (70.58 Multilingual mean) has since been passed in claimed rankings by llama-embed-nemotron-8b and KaLM-Embedding-Gemma3-12B-2511 (72.32). Check the live leaderboard.
- **Vendor numbers are marketing until reproduced.** For visually rich documents, Cohere reports ViDoRe V3 scores of 85.8 (embed-v5.0-pro), 84.5 (fast), 83.7 (voyage-4-large), 83.2 (gemini-embedding-2) and 75.5 (text-embedding-3-large). On its own agentic code-retrieval suite, Voyage reports voyage-code-4 27.54% ahead of voyage-code-3, with independent verification pending. Both are vendor-reported.

### Open Source Models

| Model | Dimensions | Max Tokens | License | Notes |
|-------|------------|------------|---------|-------|
| Nemotron 3 Embed 8B | 4096 (sliceable) | 32,768 | OpenMDW-1.1 | July 16, 2026; RTEB Multilingual #1 at 78.5 (NVIDIA-reported); `query:` / `passage:` prefixes |
| Nemotron 3 Embed 1B | 2048 | - | OpenMDW-1.1 | 72.4 RTEB Multilingual (NVIDIA-reported); NVFP4 variant for cheap serving |
| Qwen3-Embedding-8B | Up to 4096 (Matryoshka) | 32K | Apache 2.0 | 70.58 MTEB Multilingual mean |
| BGE-large-en-v1.5 | 1024 | 512 | MIT | Older strong English baseline |
| E5-large-v2 | 1024 | 512 | MIT | Instruction-tuned |
| Nomic-embed-text-v1.5 | 768 (Matryoshka) | 8192 | Apache 2.0 | Small, long context |

### Selection Criteria

| Factor | Considerations |
|--------|----------------|
| Quality (RTEB, MTEB) | Higher is better, but evaluation on your own queries matters more; held-out benchmarks beat public ones |
| Dimensions | Higher = more expressive but more storage/compute |
| Max tokens | Must accommodate your document sizes |
| Cost | API vs self-hosting tradeoffs |
| Latency | Embedding generation time |
| Multilingual | If serving non-English content |
| Modalities | Images, PDFs, video or code need a model trained for them (gemini-embedding-2, Cohere Embed 5, voyage-code-4) |
| Shared space | A family that shares one space lets you change the query-side model without re-indexing |

---

## Matryoshka and Adaptive Dimensions

### The Idea

Matryoshka Representation Learning (MRL) trains embeddings such that prefixes of the full embedding are also meaningful:

```python
full_embedding = model.encode(text)  # 1024 dimensions

# All these are valid embeddings with decreasing quality
dim_512 = full_embedding[:512]  
dim_256 = full_embedding[:256]
dim_128 = full_embedding[:128]
dim_64 = full_embedding[:64]
```

### Why It Matters

| Use Case | Dimension | Tradeoff |
|----------|-----------|----------|
| Full Retrieval | 1024-3072 | Peak Accuracy |
| **Two-Stage Retrieval**| 128 -> 1024 | **Production standard**: retrieve 1000 with 128-d, refine top 100 with 1024-d. |
| Cost-sensitive | 256 | 12x storage savings vs 3,072-d; OpenAI reported text-embedding-3-large at 256-d still beat full-size ada-002 on MTEB |
| Edge / Mobile | 64 | Maximum speed, handles simple intent |

### Models with Matryoshka Support

- OpenAI text-embedding-3-* (native)
- Google gemini-embedding-2 (128 to 3,072)
- Cohere Embed 5 (256 to 2,048) and the Voyage 4 family, including voyage-code-4 (256 to 2,048)
- Open weights: Nemotron 3 Embed, Qwen3-Embedding, Nomic-embed-text-v1.5
- Several fine-tuned models

### Using Matryoshka Embeddings

```python
from openai import OpenAI
client = OpenAI()

# Request smaller dimensions
response = client.embeddings.create(
    model="text-embedding-3-large",
    input="Your text here",
    dimensions=256  # Request 256 instead of full 3072
)
```

---

## Late Chunking and Late Interaction

### Late Chunking

**Traditional Chunking:**
`Document -> Split into chunks -> Embed chunks individually`
- **Issue**: Chunk 2 loses the context from Chunk 1.

**Late Chunking (introduced by Jina AI in 2024):**
`Full Document -> Model Encoder -> Token-level Embeddings -> Pool into chunk boundaries`
- **Benefit**: Each chunk's embedding contains information from the **entire document** because the transformer's self-attention was applied to the full sequence before pooling.
- **Requirement**: A model with long-context support (at least 8k+ tokens).

**Contextualized chunk embedding models** now ship the idea as a product, so you get document-aware chunk vectors without an LLM call per chunk:
- **voyage-context-4** (GA June 29, 2026, $0.12 per 1M tokens): one vector per chunk that encodes full-document context, built-in auto-chunking, and transparent handling of documents beyond the 32K window. Voyage reports a 2.08% chunk-level gain over voyage-context-3 across 39 datasets.
- **pplx-embed-v2-context-9b-preview** (Perplexity, September 25, 2026): encodes a document's chunks together at 2048 dimensions. It is a preview, and its embeddings must not be mixed with later releases.

This changes the cost comparison against contextual-prefix approaches that call an LLM to write a context blurb for every chunk.

### Late Interaction (ColBERT)

ColBERT keeps one embedding per token instead of one per document and scores with MaxSim (each query token matches its best document token) at query time.

**When to use ColBERT:**
- Retrieval precision is critical
- Can afford storage overhead
- Query latency budget > 50ms

**Implementation:**

```python
# Using RAGatouille
from ragatouille import RAGPretrainedModel

model = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Index documents
model.index(
    collection=documents,
    index_name="my_index"
)

# Search
results = model.search(query="What is RAG?", k=10)
```

---

## Quantization for Scale

To handle billions of vectors, **Binary** and **Scalar (Int8)** quantization are standard, and rotation-based 1 to 4-bit schemes are the 2026 addition.

| Type | Data Size | Memory Savings | Quality Loss | Where You Get It |
|------|-----------|----------------|--------------|------------------|
| Float32 | 4 bytes/dim | Baseline | 0% | All |
| Int8 | 1 byte/dim | 4x | <1% | Model outputs (Cohere Embed 4/5, Voyage 4 family), most vector DBs |
| 4-bit rotated (RaBitQ, TurboQuant, RQ) | 0.5 byte/dim | 8x | Small with rescoring (vendor claims) | Qdrant Turbo4 (opt-in), Weaviate 4-bit RQ (preview), LanceDB RaBitQ |
| **Binary** | **1 bit/dim** | **32x** | ~5-10% | Model outputs (Cohere, Voyage), most vector DBs |

**Why rotation helps:** A random or Hadamard rotation spreads each vector's energy evenly across dimensions, so a uniform per-dimension quantizer stops wasting bits on a few outlier dimensions. That is what makes 1 to 4 bits per dimension usable. As of October 2026 these modes are opt-in or preview, not defaults: Qdrant added Hadamard-rotated TurboQuant in 1.18, an opt-in 4-bit Turbo4 storage type in 1.19 and `turbo8` and `turbo16` datatypes in 1.20 (October 22), Weaviate previewed 4-bit Rotational Quantization in 1.39 (7.84x compression), and LanceDB reports multi-bit RaBitQ at 96% recall without a refine step (vendor claim). Elasticsearch is the exception: BBQ (about 1 bit per dimension) has been the default for float `dense_vector` fields of 384+ dims since 9.1, and DiskBBQ (`bbq_disk`) is the default from 9.4 where the license includes it; 9.5 added a symmetric 1-bit OSQ scorer.

**Binary Quantization Pattern** (applies to any low-bit scheme):
1. Retrieve top 1000 using Binary embeddings (extreme speed).
2. Rerank top 50 using Float32 (or int8 originals kept on disk) or a Cross-Encoder (peak accuracy).

---

## Practical Considerations

### Batch Processing

```python
# Inefficient: one API call per document
embeddings = [embed(doc) for doc in documents]

# Efficient: batch API calls
batch_size = 100
embeddings = []
for i in range(0, len(documents), batch_size):
    batch = documents[i:i + batch_size]
    batch_embeddings = embed_batch(batch)
    embeddings.extend(batch_embeddings)
```

### Chunking for Embeddings

Long documents must be chunked before embedding:

```python
def embed_document(document: str, max_tokens: int = 512) -> list[np.array]:
    chunks = chunk_document(document, max_tokens=max_tokens)
    embeddings = []
    for chunk in chunks:
        embedding = embed(chunk)
        embeddings.append(embedding)
    return embeddings
```

**Considerations:**
- Chunk size should be less than model max tokens
- Overlap helps preserve context across chunk boundaries
- Store chunk-to-document mapping for retrieval

### Normalization

Many systems expect normalized embeddings:

```python
def normalize(embedding):
    norm = np.linalg.norm(embedding)
    return embedding / norm

# Cosine similarity of normalized vectors = dot product
similarity = np.dot(normalize(a), normalize(b))
```

Most vector databases and embedding APIs handle normalization, but verify.

### Caching

Embedding computation is expensive. Cache aggressively:

```python
import hashlib

def get_embedding(text: str, cache: dict) -> np.array:
    key = hashlib.sha256(text.encode()).hexdigest()
    
    if key in cache:
        return cache[key]
    
    embedding = compute_embedding(text)
    cache[key] = embedding
    return embedding
```

---

## Embedding Drift and Versioning

### The Problem

Embeddings are not comparable across:
- Different models
- Different versions of the same model
- Sometimes different API calls (some APIs have non-determinism)

### Consequences

If you update your embedding model:
- All existing embeddings become incompatible
- Must re-embed entire corpus
- Search results will be inconsistent during migration

### Mitigation Strategies

**1. Version your embeddings:**
```python
embedding_metadata = {
    "model": "text-embedding-3-large",
    "model_version": "2024-01",
    "dimensions": 3072,
    "created_at": "2025-12-16"
}
```

**2. Plan for re-embedding:**
- Estimate cost and time for full re-embed
- Build pipelines that can run in background
- Test new embeddings before switching

**3. Blue-green deployment:**
```
Index A: Current embeddings
Index B: New embeddings (building)

Query -> Both indexes -> Merge or switch
```

**4. Track embedding quality:**
- Monitor retrieval metrics continuously
- Detect drift in embedding distributions
- Alert on quality degradation

**5. Use shared embedding spaces where they exist:**
The Voyage 4 family (large, base, lite) shares one space, and Cohere Embed 5 Pro and Fast share one, so you can index with the large model and query with the small one, or swap the query model later, without re-indexing. Qdrant's Constella research preview (September 29, 2026) pushes this further: a fixed Stella document index with swappable tiny query encoders, where Qdrant reports Constella Nano (34.5M parameters) keeping about 91% of Stella's BEIR nDCG@10 at 12x faster CPU query encoding. A shared space only helps within the family; moving to another vendor is still a full rebuild.

**6. Know who owns the embedder:**
In-database embedding (MongoDB Atlas Automated Embedding, GA August 13, 2026; turbopuffer native embedding, GA September 29, 2026; Elasticsearch `semantic_text`) removes the sync pipeline but couples your index to a vendor-chosen model version. Pin the model, and find out how the vendor handles model upgrades and who pays for re-embedding.

---

## Interview Questions

### Q: How do embedding models learn semantic similarity?

**Strong answer:**
Embedding models are trained with contrastive learning. The objective is to make embeddings of semantically similar texts close together and dissimilar texts far apart.

Training process:
1. Positive pairs: Texts that should be similar (query-document pairs, paraphrases, translations)
2. Negative pairs: Texts that should be dissimilar (often from same batch or hard negatives from BM25)
3. Loss function: Pushes positive pairs close, negative pairs far

The model learns to place texts in a high-dimensional space where distance correlates with semantic similarity. This enables retrieval: embed the query, find nearest neighbors in the document embedding space.

Modern models like E5, BGE and Qwen3-Embedding are also instruction-tuned, where you prefix with task instructions to specialize the embedding; others, such as Nemotron 3 Embed, use fixed `query:` and `passage:` prefixes. Using the wrong prefix silently degrades retrieval.

### Q: When would you use ColBERT over a bi-encoder?

**Strong answer:**
ColBERT uses late interaction: instead of one embedding per document, it keeps per-token embeddings. At query time, it computes token-level similarity.

Choose ColBERT when:
- Retrieval precision is critical (legal, medical, high-stakes)
- You can afford 10-100x storage overhead per document
- Query latency budget is 50ms+ (slightly slower than bi-encoder)
- Your queries benefit from lexical matching (technical terms)

Choose bi-encoder when:
- Storage is constrained
- Need sub-20ms latency
- Retrieval precision from bi-encoder is sufficient
- Frequent re-indexing (ColBERT reindex is expensive)

In practice, a common pattern is: bi-encoder for first-stage retrieval (top 100), then cross-encoder or ColBERT for reranking.

### Q: How do you handle embedding drift when updating models?

**Strong answer:**
Embedding models produce vectors that are only meaningful relative to the same model. If you update the model, all old embeddings become incompatible.

My approach:
1. **Never update in place.** Create a parallel index with new embeddings.
2. **Test before switching.** Compare retrieval quality on a test set with both old and new embeddings.
3. **Background rebuild.** Re-embed the entire corpus with the new model in the background.
4. **Atomic switch.** Once the new index is complete and validated, switch traffic atomically.
5. **Rollback plan.** Keep the old index available for quick rollback.

For cost estimation: if you have 10M documents at 500 tokens average, and text-embedding-3-large costs $0.13/1M tokens, re-embedding costs about $650. Plan for this cost when considering model updates. The API bill is usually the small part; index rebuild time, dual-index storage and the evaluation run cost more.

One exception worth naming: within a shared-space family (Voyage 4, Cohere Embed 5 Pro and Fast) I can change the query-side model without touching the document index. Changing the document-side model still needs the full rebuild.

### Q: How do you choose dimensions for embeddings?

**Strong answer:**
Higher dimensions capture more information but cost more storage and computation.

Considerations:
- **Storage:** 1024-d float32 = 4 KB per embedding. At 10M docs = 40 GB just for embeddings.
- **Search speed:** Higher dimensions = slower nearest neighbor search.
- **Quality:** Diminishing returns above certain dimensions for most tasks.

Practical approach:
1. Start with the model's recommended dimensions.
2. If using Matryoshka models (like text-embedding-3), experiment with lower dimensions on your task.
3. Benchmark quality at each dimension on your own queries. OpenAI's published figures set a floor: text-embedding-3-large cut to 256-d still beats full-size ada-002 (61.0 MTEB average) against 64.6 at 3,072-d, so it keeps more than 94% of the full score on that benchmark. A narrow domain can lose more.
4. For two-stage retrieval: use low dimensions for first stage, full dimensions for reranking.

For most applications, 768-1024 dimensions provide good balance. The exception is very high-precision requirements where 2048-4096 may help.

### Q: Why not just ask an LLM to judge relevance instead of using embeddings?

**Strong answer:**
Because parity costs orders of magnitude more. "The Embedder's Dilemma" (arXiv 2608.12875, August 2026) compared 10 LLMs with 26 embedding models on 37 tasks. In aggregate they tie: the best LLM (Gemini 3.1 Pro) scored 77.6 against 77.2 for the best embedder. But matching the embedder cost up to 1,431x more (USD 154 versus 0.11 per benchmark pass), open LLMs ran 2.5x to 736x slower on the same GPU, and reasoning tokens were 28% to 81% of LLM cost.

The split that follows:
- **Embeddings** for first-stage similarity over the whole corpus: precomputed, cheap, millisecond lookup.
- **Cross-encoder or listwise rerankers** on the top 50-100.
- **An LLM judge only at the end**, on a handful of candidates, and only for reasoning-heavy queries where the paper found LLMs actually lead. Lower reasoning budgets usually kept retrieval quality, so run that judge at low effort.

---

## References

- Reimers and Gurevych. "Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks" (2019)
- Khattab and Zaharia. "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT" (2020)
- Wang et al. "Text Embeddings by Weakly-Supervised Contrastive Pre-training" (E5, 2022)
- Xiao et al. "C-Pack: Packaged Resources To Advance General Chinese Embedding" (BGE, 2023)
- Kusupati et al. "Matryoshka Representation Learning" (MRL, 2022)
- Günther et al. "Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding Models" (2024)
- "The Embedder's Dilemma" (arXiv 2608.12875, 2026)
- MTEB Leaderboard (includes RTEB): https://huggingface.co/spaces/mteb/leaderboard
- OpenAI Embeddings Guide: https://platform.openai.com/docs/guides/embeddings

---

*Previous: [Transformer Architecture](04-transformer-architecture.md) | Next: [Inference Pipeline](06-inference-pipeline.md)*
