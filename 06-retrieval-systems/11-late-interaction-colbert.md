# Late Interaction & ColBERT

Late Interaction is a retrieval paradigm that sits between fast-but-imprecise **bi-encoders** and accurate-but-slow **cross-encoders**. ColBERT (Contextualized Late Interaction over BERT) is the defining model in this space, delivering cross-encoder-level accuracy at bi-encoder-level speed. The late-interaction family has matured into a production-ready alternative for high-precision search, with multimodal extensions (ColPali, ColQwen2.5, ColNomic, and unified retrievers like Wholembed v3) now in the same toolkit.

## Table of Contents

- [The Retrieval Architecture Spectrum](#the-retrieval-architecture-spectrum)
- [ColBERT Architecture](#colbert-architecture)
- [MaxSim: The Core Scoring Mechanism](#maxsim-the-core-scoring-mechanism)
- [ColBERTv2 and PLAID Indexing](#colbertv2-and-plaid-indexing)
- [Multi-Vector in General-Purpose Vector DBs (MUVERA)](#multi-vector-in-general-purpose-vector-dbs-muvera)
- [Late Interaction vs. Alternatives](#late-interaction-vs-alternatives)
- [Implementation with RAGatouille](#implementation-with-ragatouille)
- [Production Deployment Patterns](#production-deployment-patterns)
- [When to Choose ColBERT](#when-to-choose-colbert)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Retrieval Architecture Spectrum

There are three fundamental architectures for neural retrieval. Understanding where late interaction fits is the key to the entire chapter.

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   SPEED ◄──────────────────────────────────────────────► ACCURACY   │
│                                                                     │
│   Bi-Encoder          Late Interaction          Cross-Encoder       │
│   (Single Vector)     (Multi-Vector)            (Full Attention)    │
│                                                                     │
│   ● Fast (< 10ms)     ● Balanced (10-50ms)      ● Slow (100ms+)   │
│   ● Low accuracy       ● High accuracy           ● Highest accuracy│
│   ● Scales to 1B+     ● Scales to 100M+         ● Scales to 10K   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### How Each Architecture Processes a Query-Document Pair

```
BI-ENCODER (e.g., E5, BGE):
  Query  ──► Encoder ──► [1 vector]  ─┐
                                      ├──► dot product ──► score
  Doc    ──► Encoder ──► [1 vector]  ─┘

  Total interaction: 1 comparison

─────────────────────────────────────────────

LATE INTERACTION (ColBERT):
  Query  ──► Encoder ──► [N vectors] ─┐
                (one per token)       ├──► MaxSim ──► score
  Doc    ──► Encoder ──► [M vectors] ─┘
                (one per token)

  Total interaction: N x M comparisons (but decomposable)

─────────────────────────────────────────────

CROSS-ENCODER (e.g., ms-marco-MiniLM):
  [Query + Doc] ──► Encoder ──► score

  Total interaction: Full self-attention across
                     all query AND document tokens
```

**Insight**: The critical difference is *when* the query and document interact. Bi-encoders never interact (independent encoding). Cross-encoders interact fully (joint encoding). Late interaction is the middle ground: encode independently, then interact cheaply at the token level.

---

## ColBERT Architecture

ColBERT encodes queries and documents into **matrices of token-level embeddings** (not single vectors) and scores them using fine-grained token interactions.

### Encoding Phase

```
Query: "What is the price of Widget-X?"

Token Embeddings (each 128-dim):
  q1 = Embed("What")     = [0.12, -0.34, ..., 0.08]
  q2 = Embed("is")       = [0.05, -0.11, ..., 0.22]
  q3 = Embed("the")      = [0.01, -0.02, ..., 0.15]
  q4 = Embed("price")    = [0.45,  0.67, ..., 0.91]  ◄── high signal
  q5 = Embed("of")       = [0.03, -0.05, ..., 0.11]
  q6 = Embed("Widget-X") = [0.88,  0.21, ..., 0.73]  ◄── high signal

Document: "Widget-X costs $200 per month for the Standard plan"

Token Embeddings:
  d1 = Embed("Widget-X")  = [0.85,  0.19, ..., 0.71]
  d2 = Embed("costs")     = [0.42,  0.63, ..., 0.88]
  d3 = Embed("$200")      = [0.31,  0.55, ..., 0.79]
  d4 = Embed("per")       = [0.02, -0.01, ..., 0.09]
  d5 = Embed("month")     = [0.11,  0.08, ..., 0.14]
  d6 = Embed("Standard")  = [0.38,  0.44, ..., 0.62]
  d7 = Embed("plan")      = [0.29,  0.37, ..., 0.51]
```

**Key design choice**: ColBERT uses **128-dimensional** token embeddings (vs. 768-1024 for standard bi-encoders). This smaller dimensionality is critical for storage efficiency since we store N vectors per document instead of 1.

### Offline vs. Online Computation

| Component | When | Cost |
|-----------|------|------|
| Document encoding | Offline (indexing) | One-time, parallelizable |
| Query encoding | Online (per query) | Fast (~5-10ms on GPU) |
| MaxSim scoring | Online (per query) | Token-level ops, optimized by PLAID |

**This decomposition is what makes ColBERT fast**: documents are pre-encoded once. At query time, only the query needs encoding, and the scoring is simple arithmetic over pre-computed vectors.

---

## MaxSim: The Core Scoring Mechanism

MaxSim (Maximum Similarity) is the operator that makes late interaction work. It is conceptually simple but surprisingly powerful.

### How MaxSim Works

```
For each query token qi:
  1. Compute dot product with EVERY document token dj
  2. Keep only the MAXIMUM score

Score(Q, D) = SUM over all qi of MAX over all dj of (qi . dj)
```

### Worked Example

```
            d1        d2       d3       d4       d5
          Widget-X   costs    $200     per     month
  q4       0.41      0.89*    0.73     0.01     0.05
  price
  q6       0.95*     0.38     0.27     0.01     0.03
  Widget-X

  * = maximum for that query token

  MaxSim contribution from q4 ("price"): 0.89 (matched "costs")
  MaxSim contribution from q6 ("Widget-X"): 0.95 (matched "Widget-X")

  Total Score = sum of all max values across all query tokens
```

### Why MaxSim Outperforms Single-Vector Similarity

| Property | Single-Vector (Dot Product) | MaxSim (Late Interaction) |
|----------|----------------------------|--------------------------|
| **Granularity** | Document-level | Token-level |
| **Partial matching** | All-or-nothing | Tokens match independently |
| **Term importance** | Compressed into 1 vector | Each token contributes separately |
| **Rare terms** | Diluted by averaging | Preserved as individual vectors |

**The intuition**: In a bi-encoder, the meaning of "Widget-X" gets averaged with "costs," "$200," and every other token into a single vector. If "Widget-X" is rare, its signal gets diluted. In ColBERT, "Widget-X" keeps its own dedicated vector, so the MaxSim operator can find a strong match for it independently.

---

## ColBERTv2 and PLAID Indexing

The original ColBERT (2020) had a critical limitation: **storage**. Storing 128-dim vectors for every token in every document is expensive. A corpus of 10M documents with 200 tokens each is 2B token vectors: about 1 TB at float32, still 512 GB at float16.

### ColBERTv2 Improvements (2021)

ColBERTv2 introduced two key innovations:

**1. Residual Compression**:

```
Original ColBERT:
  Each token vector: 128 dims x 32-bit float = 512 bytes

ColBERTv2 Residual Compression:
  1. Cluster all token vectors into centroids (k-means)
  2. Store only the centroid ID + residual (difference)
  3. Quantize the residual to 1-2 bits per dimension

  Each token vector: ~16-32 bytes (16-32x compression)
```

**2. Denoised Supervision**:
- Trains on hard negatives mined from a cross-encoder teacher
- Cross-encoder labels "clean up" noisy training data
- Result: better quality embeddings despite compression

**ColBERTv2 Storage Comparison**:

| System | Per-Token Storage | 10M Docs (200 tokens each) |
|--------|------------------|---------------------------|
| ColBERT v1 | 512 bytes | ~1 TB |
| ColBERTv2 (compressed) | 32 bytes | ~64 GB |
| Bi-encoder (1 vector/doc) | 3 KB | ~30 GB |

### PLAID: The Indexing Engine

PLAID (Performance-optimized Late Interaction Driver) is the indexing and retrieval engine that makes ColBERT practical at scale.

```
┌─────────────────────────────────────────────────────────────────┐
│                    PLAID RETRIEVAL PIPELINE                     │
│                                                                 │
│  Stage 1: CANDIDATE GENERATION                                  │
│  ─────────────────────────────                                  │
│  For each query token, find nearest centroids                   │
│  Collect candidate passages that contain those centroids        │
│  Result: ~10,000 candidates from millions                       │
│                                                                 │
│  Stage 2: CENTROID INTERACTION (with pruning)                   │
│  ────────────────────────────────────────────                   │
│  Drop centroids whose query score is below a threshold          │
│  Approximate MaxSim using centroid-level scores only            │
│  Filter candidates to top ~1,000                                │
│                                                                 │
│  Stage 3: CENTROID INTERACTION (no pruning)                     │
│  ──────────────────────────────────────────                     │
│  Re-score survivors with all of their centroid IDs              │
│  (still no residuals decompressed)                              │
│  Filter to top ~100                                             │
│                                                                 │
│  Stage 4: FULL DECOMPRESSION                                    │
│  ────────────────────────────                                   │
│  Fully decompress token vectors for top candidates              │
│  Compute full MaxSim on the decompressed vectors                │
│  Return final ranked results                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight**: PLAID avoids decompressing all vectors for all documents. Each stage cheaply filters the candidate set so that expensive full scoring only happens on a tiny fraction of the corpus.

**PLAID Performance**:
- Retrieves from 10M+ documents in **50-100ms** on a single GPU
- Computes full MaxSim only for the final candidates, over decompressed (lossy) residual vectors. The candidate stages are approximate, so recall depends on how aggressively you prune (`ncells`, centroid score threshold); measure it against exhaustive MaxSim on a sample
- Uses centroid pruning to skip 99%+ of the corpus before full scoring

---

## Multi-Vector in General-Purpose Vector DBs (MUVERA)

PLAID is a purpose-built engine, and most teams do not want a second search system just for late interaction. **MUVERA** (Multi-Vector Retrieval via Fixed Dimensional Encodings, Google Research, NeurIPS 2024) closes that gap: it turns a document's set of token vectors into one long fixed-length vector whose inner product approximates the MaxSim (Chamfer) score, so an ordinary single-vector ANN index can generate candidates.

```
INDEX:  doc token vectors ──► FDE transform ──► 1 fixed-length vector ──► HNSW / DiskANN
                 └──────────────────────────────► keep the multi-vector set (compressed, on disk)

QUERY:  query token vectors ──► FDE ──► ANN top-k candidates
                                              └──► full MaxSim on stored multi-vectors ──► results
```

The shape is the same as PLAID (cheap approximate candidates, full MaxSim on a few hundred), but the candidate stage runs on infrastructure you already operate. FDEs are long (thousands of dimensions), so they are usually quantized.

**Engine support, October 2026:**

| Engine | Multi-vector support | Status |
|--------|----------------------|--------|
| **Milvus 3.0** | On-disk EmbList + DiskANN index for embedding lists, with MUVERA and LEMUR acceleration | GA July 29, 2026 (3.0.2 on Sep 20) |
| **Weaviate** | Native multi-vector embeddings (1.29+), with optional MUVERA encoding on HNSW since 1.31 (2025); MUVERA for the disk-backed HFresh index is new in 1.40 | Multi-vector and MUVERA on HNSW shipping; the HFresh MUVERA path shipped in 1.40.0 (October 7, 2026) with no GA or preview label in the notes, so check its status |
| **Qdrant** | Native multivector fields scored with MaxSim | Shipping; Qdrant's open PubMed-Multi-Vector dataset (8.37B BGE-M3 token vectors) benchmarks dense vs sparse vs ColBERT |
| **Vespa** | Native ColBERT tensor ranking | Shipping |

Model side: Answer.AI published ONNX exports of NeuML's ColBERT-MUVERA micro and small models on September 18, 2026.

### Compressing the Vectors Themselves

The other lever is storing fewer vectors per document. For page-image retrievers (ColPali-style), the GLIE paper (arXiv 2609.11808, September 2026) finds that visual-document token vectors sit near a manifold of intrinsic dimension five to six on the unit sphere. Two practical results: renormalizing k-means centroids back onto the sphere is a free correction worth up to +0.093 nDCG@5 over raw centroids, and keeping 4 learned vectors per page plus a 415K-parameter decoder that regenerates the full set for top candidates retains nearly 80% of uncompressed nDCG@5 on ViDoRe v1, against 70% for the best prior post-hoc method. It is a single research result, not an engine feature, but it points the same way: compact representation for candidates, full multi-vector scoring only at the top.

---

## Late Interaction vs. Alternatives

### Comprehensive Comparison

| Dimension | BM25 | Bi-Encoder | ColBERT (Late) | Cross-Encoder |
|-----------|------|-----------|----------------|---------------|
| **Encoding** | Term frequency | 1 vector/doc | N vectors/doc | Joint (no pre-compute) |
| **Query latency** | ~5ms | ~10ms | ~30-50ms | ~500ms+ per pair |
| **Scalability** | Billions | Billions | 100M+ | ~10K (reranking only) |
| **Storage (1M docs)** | ~2 GB | ~3 GB | ~6-12 GB | 0 (no index) |
| **Accuracy (NDCG@10)** | 0.30-0.35 | 0.35-0.40 | 0.39-0.44 | 0.42-0.46 |
| **Domain transfer** | Strong (lexical) | Weak (needs fine-tuning) | Strong (token-level) | Strongest |
| **Setup complexity** | Low | Medium | High | Low (no index) |

### When ColBERT Wins

```
                  ▲ Accuracy
                  │
             0.45 ┤                     ● Cross-Encoder
                  │                   ●
             0.40 ┤              ● ColBERT
                  │         ●
             0.35 ┤    ● Bi-Encoder
                  │ ●
             0.30 ┤ BM25
                  │
                  └────┬────┬────┬────┬────┬──► Throughput (QPS)
                      10   100  1K   10K  100K
```

**ColBERT occupies the sweet spot**: it typically adds a few NDCG points over a bi-encoder of similar size, with the largest gains on out-of-domain, term-heavy corpora (up to +13.8% mAP on specialized datasets), while being 10-50x faster than cross-encoders.

---

## Implementation with RAGatouille

RAGatouille (by Answer.AI) is the standard Python library for using ColBERT in RAG pipelines. It wraps the Stanford ColBERT codebase with a simple, high-level API.

### Basic Usage

```python
from ragatouille import RAGPretrainedModel

# Load a pretrained ColBERT model
RAG = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Index documents (one-time, creates PLAID index on disk)
documents = [
    "Widget-X costs $200 per month for the Standard plan.",
    "The Enterprise plan includes SSO and audit logs for $800/month.",
    "All plans include 99.9% uptime SLA and 24/7 email support.",
    "Widget-X was launched in 2023 and serves 10,000+ customers.",
]

index_path = RAG.index(
    index_name="products",
    collection=documents,
    split_documents=True  # auto-chunk long docs
)

# Search the index
results = RAG.search(
    query="How much does Widget-X cost?",
    k=3
)

for result in results:
    print(f"Score: {result['score']:.4f}")
    print(f"Text:  {result['content']}\n")
```

### Integration with LangChain

```python
from ragatouille import RAGPretrainedModel
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# Create ColBERT retriever
RAG = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
retriever = RAG.as_langchain_retriever(k=5)

# Build RAG chain
template = """Answer based on the following context:
{context}

Question: {question}"""

prompt = ChatPromptTemplate.from_template(template)
llm = ChatOpenAI(model="gpt-4o")

chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
)

response = chain.invoke("What features does the Enterprise plan include?")
```

### Other ColBERT Libraries and Integrations

| Library | Use Case | Notes |
|---------|----------|-------|
| **RAGatouille** | Python-first, simple API | Best for prototyping and small-medium scale |
| **colbert-ai** (Stanford) | Research, full control | Lower-level, more configuration options |
| **Vespa** | Production-scale deployment | Managed infrastructure with native ColBERT support |
| **PyLate** | Flexible training/fine-tuning | Built on Sentence Transformers, good for custom models |
| **Jina ColBERT v2** | Multilingual (89 languages) | Flexible output dimensions, production-ready |
| **Milvus 3.0 / Weaviate / Qdrant** | Multi-vector inside a general vector DB | MUVERA or native MaxSim; see the engine table above |

---

## Production Deployment Patterns

### Pattern 1: ColBERT as Primary Retriever

```
Query ──► ColBERT (PLAID) ──► Top 20 ──► LLM
```

Best for: medium-scale corpora (1M-50M docs) where accuracy is paramount and you can afford the storage overhead.

### Pattern 2: ColBERT as Reranker (Most Common)

```
Query ──► BM25 or Bi-Encoder ──► Top 1000 ──► ColBERT Rerank ──► Top 20 ──► LLM
```

Best for: large-scale systems where first-stage retrieval must be cheap, but you need high-quality reranking without the cost of a cross-encoder.

```
┌─────────────────────────────────────────────────────────────────┐
│              COLBERT-AS-RERANKER ARCHITECTURE                   │
│                                                                 │
│  User Query                                                     │
│      │                                                          │
│      ▼                                                          │
│  First Stage: BM25 / Bi-Encoder                                 │
│  (cheap, high recall, Top 1000)                                 │
│      │                                                          │
│      ▼                                                          │
│  Second Stage: ColBERT MaxSim Reranking                         │
│  (pre-computed doc tokens, score Top 1000)                      │
│  Cost: only query encoding + MaxSim arithmetic                  │
│      │                                                          │
│      ▼                                                          │
│  Top 20 Passages ──► LLM Generation                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Hybrid (ColBERT + BM25 + Dense)

```
Query ──┬──► BM25 (Top 50) ────────┐
        ├──► Dense Bi-Encoder (50) ─┼──► RRF ──► ColBERT Rerank ──► Top 10
        └──► ColBERT (Top 50) ─────┘
```

Best for: maximum accuracy at medium scale. Expensive but covers all retrieval modalities.

### Storage and Infrastructure Considerations

| Corpus Size | Bi-Encoder Storage | ColBERT Storage | GPU Requirement |
|------------|-------------------|-----------------|-----------------|
| 100K docs | ~300 MB | ~600 MB - 1.2 GB | CPU-only OK |
| 1M docs | ~3 GB | ~6-12 GB | 1 GPU recommended |
| 10M docs | ~30 GB | ~60-120 GB | 1-2 GPUs required |
| 100M docs | ~300 GB | ~600 GB - 1.2 TB | Multi-GPU / distributed |

**Reality check**: ColBERT's storage is 2-4x that of bi-encoders. For most RAG use cases (under 10M docs), this is manageable. For web-scale search (billions of pages), bi-encoders or learned sparse methods remain more practical for the first retrieval stage. MUVERA-style candidate generation on a disk-resident index (Milvus 3.0 DiskANN today, Weaviate HFresh once 1.40 is GA) moves the RAM ceiling up, because only the single-vector FDE index needs to be hot.

---

## When to Choose ColBERT

### Decision Framework

```
Is your corpus < 100M documents?
├── No  ──► Use Bi-Encoder for retrieval + ColBERT for reranking
└── Yes
    │
    Is accuracy more important than infrastructure simplicity?
    ├── No  ──► Use Bi-Encoder (simpler, cheaper)
    └── Yes
        │
        Can you afford 2-4x storage vs. bi-encoder?
        ├── No  ──► Use Bi-Encoder + Cross-Encoder reranker
        └── Yes
            │
            Do you want to run a separate retrieval engine?
            ├── No  ──► MUVERA / native multi-vector in your vector DB
            └── Yes ──► Use ColBERT (PLAID) as primary retriever
```

### ColBERT vs. Dense Retrieval vs. Hybrid Search

| Scenario | Best Choice | Why |
|----------|-------------|-----|
| General-purpose RAG (< 1M docs) | Hybrid (Dense + BM25) | Simplest, good enough accuracy |
| Domain-specific search (legal, medical) | ColBERT | Token-level matching preserves jargon |
| Multilingual corpus | Jina ColBERT v2 | Native 89-language support |
| Cost-sensitive, high-volume | Bi-Encoder + BM25 | Lowest storage and compute |
| Maximum accuracy, medium scale | ColBERT + Reranker | Best quality without cross-encoder latency |
| Web-scale (1B+ docs) | Bi-Encoder first stage + ColBERT rerank | ColBERT index too large for primary |

---

## Interview Questions

### Q: Explain the difference between bi-encoders, cross-encoders, and late interaction models. When would you choose each?

**Strong answer:**
The three architectures differ in *when* the query and document interact:

**Bi-encoders** encode query and document independently into single vectors. Interaction happens only via a dot product at the end. This is fast (pre-compute all document vectors, search in milliseconds) but loses fine-grained matching: the entire document meaning is compressed into one point in vector space.

**Cross-encoders** process the concatenated query + document through a single transformer. Full self-attention means every query token attends to every document token. This gives the highest accuracy but cannot pre-compute anything. Every query-document pair requires a full forward pass, making it infeasible for first-stage retrieval. Cross-encoders are used as rerankers on the top 10-100 candidates.

**Late interaction (ColBERT)** encodes query and document independently (like bi-encoders), but into *per-token* vector matrices instead of single vectors. Scoring uses MaxSim: for each query token, find its best-matching document token. This preserves token-level granularity while still allowing document pre-computation. The result is near-cross-encoder accuracy at near-bi-encoder speed.

I would choose bi-encoders for large-scale first-stage retrieval where simplicity matters, cross-encoders for high-stakes reranking of small candidate sets, and ColBERT when I need the accuracy of a cross-encoder but cannot afford its latency, particularly for domain-specific search where term-level matching matters (legal, medical, technical docs).

### Q: ColBERT stores one vector per token. How does it scale, and what are the storage tradeoffs?

**Strong answer:**
The naive storage cost of ColBERT is significant. A 200-token document requires 200 vectors of 128 dimensions each (25,600 floats), versus 1 vector of 768-1024 dimensions for a bi-encoder. That is roughly 25-33x the storage per document before compression.

ColBERTv2 addresses this with **residual compression**: token vectors are clustered into centroids, and only the centroid ID plus a quantized residual is stored. This achieves 16-32x compression per token vector, bringing practical storage to about 2-4x that of a float32 bi-encoder.

The PLAID indexing engine further improves efficiency at query time by using a multi-stage pipeline. It generates candidates from the centroids nearest each query token, scores them on centroid IDs alone (pruning low-scoring centroids first) to eliminate 99% of the corpus, and decompresses residuals only for the final candidates. The final full MaxSim, over decompressed residual vectors, is computed on fewer than 100 documents, keeping latency at 50-100ms even on 10M+ document corpora. Because the candidate stages are approximate, I would measure recall against exhaustive MaxSim on a sample before tuning pruning aggressively.

For scale beyond 100M documents, I would use ColBERT as a reranker rather than a primary retriever: let a bi-encoder or BM25 do the first-stage retrieval to narrow the candidate set to 1,000 documents, then apply ColBERT's MaxSim for high-quality reranking. If the team already runs Milvus, Qdrant or Weaviate, the middle path is MUVERA: index one fixed-dimensional encoding per document for ANN candidate generation and keep the compressed token vectors on disk for full MaxSim on the top few hundred.

### Q: You are designing a legal document search system with 5M documents. The team is debating between dense bi-encoder search with a cross-encoder reranker vs. ColBERT. What do you recommend?

**Strong answer:**
I would recommend ColBERT for this use case for three reasons:

First, **legal text is term-sensitive**. Contract clauses reference specific section numbers, defined terms (e.g., "Force Majeure"), and exact phrases. ColBERT's token-level MaxSim matching preserves these rare-but-critical terms that get diluted in a single-vector bi-encoder embedding.

Second, **5M documents is squarely in ColBERT's sweet spot**. With ColBERTv2 compression, the index would be roughly 30-60 GB, which fits on a single 80 GB GPU. This is small enough for primary retrieval, avoiding the need for a separate first-stage retriever.

Third, **cross-encoder reranking adds latency**. Each query-document pair requires a full transformer forward pass. Reranking 100 candidates with a cross-encoder might take 500ms-2s. ColBERT achieves comparable accuracy while keeping total latency under 100ms because document tokens are pre-computed.

The one area I would supplement ColBERT is with a parallel BM25 index for exact-match queries (statute numbers, case citations) where keyword precision matters. I would use RRF to combine ColBERT and BM25 results before passing to the LLM.

---

## References
- Khattab & Zaharia. "ColBERT: Efficient and Effective Passage Search" (SIGIR 2020)
- Santhanam et al. "ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction" (NAACL 2022)
- Santhanam et al. "PLAID: An Efficient Engine for Late Interaction Retrieval" (CIKM 2022)
- Answer.AI. "RAGatouille: State-of-the-art Late Interaction Retrieval" (GitHub, 2024)
- Jina AI. "Jina-ColBERT-v2: General-Purpose Multilingual Late Interaction Retriever" (2024)
- Dhulipala et al. "MUVERA: Multi-Vector Retrieval via Fixed Dimensional Encodings" (NeurIPS 2024)
- Weaviate. "An Overview of Late Interaction Retrieval Models" (2025)
- Milvus. "v3.0.0 release notes" (GitHub, July 2026)
- GLIE, arXiv 2609.11808: compressing visual-document multi-vector embeddings (September 2026)
- ECIR 2026. "Late Interaction Workshop" (2026)

---

*Previous: [Contextual Retrieval](10-contextual-retrieval.md) | Next: [Multi-Modal RAG](12-multimodal-rag.md)*
