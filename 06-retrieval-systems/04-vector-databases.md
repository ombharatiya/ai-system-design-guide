# Vector Databases

Vector databases are purpose-built systems for storing, indexing, and searching high-dimensional embeddings. The market has split into **Managed Serverless** and **Specialized High-Performance** engines. We no longer ask "Does it support vector search?" (Postgres, Redis, and Mongo all do). We ask **"Does it scale to 100M+ vectors with sub-100ms P99 and full metadata filtering?"**

## Table of Contents

- [What Is a Vector Database](#what-is-a-vector-database)
- [Vector Search Fundamentals](#vector-search-fundamentals)
- [Indexing Algorithms](#indexing-algorithms)
- [Competitive Landscape](#competitive-landscape)
- [Detailed Database Comparison](#detailed-database-comparison)
- [Metadata Filtering](#metadata-filtering)
- [Query Patterns](#query-patterns)
- [Production Operations](#production-operations)
- [Managed vs Self-Hosted (TCO Analysis)](#managed-vs-self-hosted-tco-analysis)
- [Selection Framework](#selection-framework)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## What Is a Vector Database

A vector database stores embeddings (dense vectors) and enables fast similarity search over them.

```
Traditional DB:      SELECT * FROM docs WHERE category = 'tech'
Vector DB:           SELECT * FROM docs ORDER BY similarity(embedding, query_embedding) LIMIT 10
```

### Core Capabilities

| Capability | Purpose |
|------------|---------|
| Vector storage | Persist high-dimensional embeddings |
| Similarity search | Find nearest neighbors quickly |
| Metadata filtering | Combine vector search with attribute filters |
| CRUD operations | Update embeddings as data changes |
| Scaling | Handle millions to billions of vectors |

### Why Not General Databases?

Traditional databases can store vectors but lack optimized search:

| Approach | Search Complexity | Practical at Scale |
|----------|-------------------|-------------------|
| Brute force (e.g., pgvector with no HNSW or IVFFlat index) | O(n * d) | OK to ~1M vectors |
| ANN index (dedicated vector DB) | O(log n) or O(1) | Yes, billions |

---

## Vector Search Fundamentals

### Exact vs Approximate Search

**Exact (brute force):**
- Compare query to every stored vector
- O(n * d) per query
- Perfect accuracy

**Approximate Nearest Neighbor (ANN):**
- Use index structure to prune search space
- Sub-linear complexity
- Slightly lower recall (typically 95-99%)

### Distance Metrics

| Metric | Formula | Range | Best For |
|--------|---------|-------|----------|
| Cosine | 1 - (a . b) / (norm(a) * norm(b)) | [0, 2] | Text embeddings |
| Euclidean (L2) | sqrt(sum((a - b)^2)) | [0, inf) | Image embeddings |
| Dot product | a . b | (-inf, inf) | Already normalized |

**For text embeddings:** Use cosine similarity (or dot product if pre-normalized).

### Recall vs Latency Tradeoff

```
                    ^ Recall
                    |
               100% | ------------------ Brute force
                    |         *          Well-tuned ANN
                    |      *
                    |   *
                95% |*                   Fast ANN
                    |
                    +-----+-------+------> Latency
                       1ms      10ms
```

ANN indices trade some accuracy for speed. Tune for your requirements.

---

## Indexing Algorithms

### HNSW (Hierarchical Navigable Small World)

The most popular algorithm for production **in-memory** vector search.

**How it works:**
1. Build a graph where nodes are vectors
2. Connect to nearby neighbors
3. Multiple layers of abstraction (hierarchical)
4. Search: navigate from top layer down, greedy nearest neighbor

```
Layer 2:   *--------*--------*
           |        |        |
Layer 1:   *--*--*--*--*--*--*
           |  |  |  |  |  |  |
Layer 0:   ********************  (all vectors)
```

**Pros:**
- Excellent recall/latency tradeoff
- No training required
- Supports updates natively

**Cons:**
- Memory-intensive: the full-precision vectors and the graph links both live in RAM
- The graph itself is a small share at high dimensions. OpenSearch's sizing rule is 1.1 x (4 x dims + 8 x M) bytes per vector, about 12-19% above the raw float32 vectors at 1,536 dims for M of 16 to 64; low-dimensional vectors pay proportionally more
- 10M vectors at 1,536 dims need ~70-75 GB of RAM per copy (61 GB of that is the float32 vectors), before replicas

**Key parameters:**
- `M`: Max connections per node (16-64)
- `ef_construction`: Build-time exploration (100-500)
- `ef_search`: Query-time exploration (50-200)

### DiskANN (SSD-based)

Built for **billion-scale** indexes that do not fit in RAM. The original paper served a billion SIFT points from one workstation with 64 GB of RAM and an SSD, at over 5,000 queries per second, under 3 ms mean latency and 95%+ 1-recall@1.

**How it works:**
- Keeps the full-precision vectors and the graph on SSD (NVMe), with only compressed (PQ) codes in RAM to steer the search
- Uses the Vamana algorithm for efficient disk-based graph traversal

**Pros:**
- 10x cheaper than HNSW for billion-scale datasets with <5ms latency penalty
- 90-95% reduction in RAM requirements vs HNSW

**Cons:**
- Slightly higher latency than pure in-memory HNSW
- Tail latency depends on NVMe IOPS, so size SSDs for query concurrency, not just capacity

**Example:** A 100-million-vector index with 1,536 dimensions needs roughly 700 GB of RAM per copy for HNSW (614 GB of float32 vectors plus graph and overhead). Using DiskANN, the RAM requirement drops by 90-95% while maintaining sub-10ms query times.

### IVF (Inverted File Index)

Partition vectors into clusters, search only relevant clusters.

**How it works:**
1. Use k-means to create centroids
2. Assign each vector to nearest centroid
3. At query time: find nearest centroids, search those clusters

**Pros:**
- Lower memory than HNSW
- Can use quantization (IVF-PQ)

**Cons:**
- Requires training
- Updates need re-clustering or hybrid approach

**Key parameters:**
- `nlist`: Number of clusters (sqrt(n) rule of thumb)
- `nprobe`: Clusters to search at query time

### Product Quantization (PQ)

Compress vectors to reduce memory and speed up comparison.

**How it works:**
1. Split vector into subvectors
2. Quantize each subvector to a codebook
3. Store codes instead of full vectors

**Memory reduction:** 4-32x typical

**Tradeoff:** Lower accuracy due to quantization loss

### Low-Bit and Rotation-Based Quantization

Between May and October 2026 several engines shipped or extended 1 to 8-bit vector storage, mostly as opt-in or preview options rather than defaults. The newest trick is **rotate first, then quantize**. Embedding dimensions have very uneven variance, so per-dimension scalar quantization wastes bits on quiet dimensions and clips loud ones. Multiplying by a random or Hadamard rotation spreads the energy evenly across dimensions, after which a uniform 4-bit or 1-bit grid loses far less information.

| Engine | Feature | Status | Footprint |
|---|---|---|---|
| Qdrant | TurboQuant with Hadamard rotation (1.18, May 2026); Turbo4 storage type keeps only the 4-bit form (1.19, August 2026); `turbo8` and `turbo16` vector datatypes (1.20, October 2026) | Opt-in | From 36 bits per coordinate in 1.18 (float32 original plus 4-bit copy) to 4 bits with Turbo4, a ninefold cut |
| Weaviate | 4-bit Rotational Quantization (1.39, August 2026), adding to the 8-bit and 1-bit RQ it already had; 1.40 (October 2026) adds "RQ4 centered quantization" | Preview in 1.39, HNSW only; the 1.40 notes give no status label, so confirm before relying on it | 1,536-dim vector from 6,144 bytes to 784 bytes (7.84x) |
| Elasticsearch | DiskBBQ (`bbq_disk`, GA since 9.2) gained a symmetric 1-bit OSQ scorer and merge-time calibration (9.5, August 2026) | GA; default index type for float vectors from 9.4 where the license includes it (Enterprise) | ~1 bit per dimension plus correction terms |
| turbopuffer | Plain `i8` vector type (June 2026), no rotation | Shipped | 75% less storage and query cost than f32 for quantization-aware models (vendor) |

Qdrant's own numbers show the tradeoff: 4-bit TurboQuant recall 0.9193 vs 0.9285 for 8-bit scalar quantization vs 0.9419 uncompressed on one dataset, at twice the compression of scalar quantization (vendor-reported). The design decision is whether to **keep the float originals for rescoring**. Keeping them on disk or object storage and rescoring the top few hundred candidates recovers most of the lost recall; dropping them (Turbo4) is the cheapest option and caps your maximum recall.

The RAM math changes accordingly: 100M vectors at 1,536 dims is ~614 GB as float32 but ~77 GB at 4 bits. At a billion vectors, that is the difference between a fleet of high-memory nodes and a few machines plus SSD.

### Flat Index (Brute Force)

No approximation, exact search.

**Use when:**
- Less than 100K vectors
- Accuracy is critical
- Latency budget is generous

### Algorithm Comparison

| Algorithm | Memory | Build Time | Query Speed | Recall | Updates |
|-----------|--------|------------|-------------|--------|---------|
| HNSW | High | Medium | Very fast | 95-99% | Good |
| DiskANN | Low (SSD) | Medium | Fast | 95-99% | Fair |
| IVF | Medium | Fast | Fast | 90-98% | Fair |
| IVF-PQ | Low | Fast | Fast | 85-95% | Fair |
| Flat | Low | None | Slow | 100% | Instant |

---

## Competitive Landscape

### Vector-Native (Dedicated)

| Database | Type | Best For | Pricing Model |
|----------|------|----------|---------------|
| **Pinecone** | Managed cloud (serverless standard); BYOC GA September 23, 2026 (data plane in your AWS, GCP or Azure account) | Easy start, scale, managed SLAs; BM25 full-text search GA September 9, 2026 | Storage plus read and write units |
| **Qdrant** | Open source / Cloud (Rust, high-perf) | Self-hosted control, strong filtered search; vendor benchmarks cite ~12ms p99 at 10M vectors; 1.20 (October 2026) adds BM25 scoring on the full-text index, a `code` tokenizer, and per-element nested payload indexes | Per GB (cloud) or free |
| **Weaviate** | Open source / Cloud | Native hybrid (BM25 + dense + metadata) in a single query, MMR diversity and Boost API (GA in 1.39), multimodal; 1.40 (October 2026) makes namespaces GA as a licensed feature, stops creating an implicit legacy vector index (auto-schema creates a `default` named vector), and defaults the vectorizer to none, so schemas that relied on the old defaults change on upgrade | Per dimension-hour |
| **Milvus** | Open source / Cloud (Zilliz) | Distributed scale; 3.0 (July 2026) adds External Collections over Parquet, Iceberg, Lance and Vortex in object storage, plus online column backfill | Free (self-host) or Zilliz Cloud |
| **turbopuffer** | Managed, object-storage-first | Many cold namespaces (per-tenant indexes) at low cost; native embedding GA September 29, 2026 | Usage-based |
| **Chroma** | Open source | Prototyping, local dev, embedded use | Free |

### General-Purpose (Plugin/Extension)

| Database | Type | Best For | Pricing Model |
|----------|------|----------|---------------|
| **pgvector (0.8.7)** | PostgreSQL extension | Small scale, existing PG (HNSW + IVFFlat). Patch to 0.8.7 (October 1, 2026): it fixes CVE-2026-103484, an IVFFlat build overflow that any role allowed to create an index can trigger | Compute only |
| **Elasticsearch (9.5)** | Search engine | Hybrid via the `rrf` and `linear` retrievers, DiskBBQ quantized vectors, `semantic_text` fields that embed automatically at ingest | License-based |
| **MongoDB Atlas Vector Search** | Document DB | Existing Mongo shops; Automated Embedding GA August 13, 2026 (Atlas embeds and re-embeds documents with Voyage models on write) | Atlas cluster pricing |

### Where the Market Moved in 2026

Three shifts change how you evaluate an engine:

1. **Embedding moved into the database.** MongoDB Atlas Automated Embedding (GA), turbopuffer native embedding (GA; turbopuffer cites Readwise cutting median embedding latency 8x by embedding in parallel with query execution), Elasticsearch `semantic_text`, and Milvus 3.0's Hugging Face Inference Providers integration all remove the separate embed-and-sync pipeline. The cost is coupling: the vendor now picks the model version, so ask how a model change triggers re-embedding before you adopt it.
2. **The index moved toward the lakehouse.** Milvus 3.0 External Collections query Parquet, Iceberg, Lance and Vortex data in place (zero-copy, read-only), and turbopuffer's September 30 "RIP, vector database" post describes a v3 storage design where the ANN index becomes a secondary index rather than the primary data layout (announced; it passes CI but is not in production yet, and the scale figures in the post are the vendor's). The argument shifts from "which vector DB" to "index in place over object storage, or copy into a serving store."
3. **Low-bit quantization spread across vendors** (see [above](#low-bit-and-rotation-based-quantization)), changing the RAM budget for HNSW sizing.

---

## Detailed Database Comparison

### Feature Matrix

| Feature | Pinecone | Qdrant | Weaviate | Milvus | pgvector |
|---------|----------|--------|----------|--------|----------|
| **Language** | Proprietary | Rust | Go | Go/C++ | C |
| Hosted option | Yes | Yes | Yes | Yes (Zilliz) | Via cloud PG |
| Self-hosted | BYOC only (data plane in your cloud) | Yes | Yes | Yes | Yes |
| **Serverless** | Yes (Best) | Yes | Yes | Yes (Zilliz) | No |
| **Cloud-Native** | Any | Any | Any | K8s for distributed; Docker standalone | Any |
| Metadata filtering | Good | Excellent | Good | Good | Via SQL |
| **Hybrid search** | Sparse-dense, or BM25 FTS with client-side fusion | Native | Native | Native | Multi-stage (limited) |
| Max vectors | Billions | Billions | Billions | Billions | ~10M |
| HNSW index | Yes | Yes | Yes | Yes | Yes |

---

## Metadata Filtering

Critical for multi-tenant and filtering use cases.

```python
# Pinecone
results = index.query(
    vector=query_embedding,
    top_k=10,
    filter={"tenant_id": "123", "category": {"$in": ["tech", "science"]}}
)

# Qdrant (Query API; the legacy /search endpoint is removed from the 1.19 REST schema)
results = client.query_points(
    collection_name="documents",
    query=query_embedding,
    limit=10,
    query_filter=Filter(
        must=[
            FieldCondition(key="tenant_id", match=MatchValue(value="123")),
            FieldCondition(key="category", match=MatchAny(any=["tech", "science"]))
        ]
    )
).points
```

**Performance impact:** Filtering happens during search, not after. Pre-filtered indices are faster but less flexible.

**Why metadata filtering is often the bottleneck:** In naive vector search, we find the "Top K" nearest neighbors and THEN filter by metadata. If the filter is very restrictive, we might find 0 results after filtering. Specialized databases now use **Pre-Filtering with HNSW**, traversing the graph but only considering nodes that satisfy the boolean metadata constraint. This requires specialized bitmasks or hardware acceleration (SIMD) to keep latencies low.

**Disk-Native Metadata:** Modern DBs tier data between RAM and disk per component. Qdrant 1.19 replaced the old `on_disk` and `always_ram` flags with one `memory` setting (pinned, cached or cold) that applies to vectors, HNSW graphs, sparse and payload indexes, so complex filters (e.g., full-text + geo + vector) run without holding every payload index in RAM.

---

## Query Patterns

### Pattern 1: Simple Semantic Search

```python
def semantic_search(query: str, top_k: int = 5) -> list[Document]:
    query_embedding = embed(query)
    results = vector_db.search(query_embedding, top_k=top_k)
    return [Document(id=r.id, text=r.payload["text"], score=r.score) for r in results]
```

### Pattern 2: Filtered Search

```python
def filtered_search(query: str, filters: dict, top_k: int = 5) -> list[Document]:
    query_embedding = embed(query)
    results = vector_db.search(
        query_embedding,
        top_k=top_k,
        filter=filters  # {"tenant_id": "abc", "created_after": "2025-01-01"}
    )
    return results
```

### Pattern 3: Hybrid Search (Dense + Sparse)

```python
def hybrid_search(query: str, alpha: float = 0.5, top_k: int = 5) -> list[Document]:
    # Dense (semantic)
    dense_embedding = embed(query)
    dense_results = vector_db.search(dense_embedding, top_k=top_k * 2)

    # Sparse (keyword)
    sparse_results = bm25_search(query, top_k=top_k * 2)

    # Combine with reciprocal rank fusion
    combined = reciprocal_rank_fusion(
        [dense_results, sparse_results],
        weights=[alpha, 1 - alpha]
    )

    return combined[:top_k]
```

Some databases (Weaviate, Qdrant, Milvus, Elasticsearch) fuse dense and keyword results natively. Pinecone's BM25 full-text search ranks by one scoring type per request, so you run the lexical and vector searches separately and fuse them client-side (e.g., with RRF).

```python
# Weaviate native hybrid (Python client v4)
docs = client.collections.use("Document")
results = docs.query.hybrid(
    query=query,
    alpha=0.5,  # 0 = BM25 only, 1 = vector only
    limit=5,
)
```

### Pattern 4: Multi-Vector Query

For parent-child or multi-aspect retrieval:

```python
def multi_vector_search(queries: list[str], top_k: int = 5) -> list[Document]:
    all_results = []

    for query in queries:
        embedding = embed(query)
        results = vector_db.search(embedding, top_k=top_k)
        all_results.extend(results)

    # Dedupe and rerank
    unique = dedupe_by_id(all_results)
    reranked = rerank(queries[0], unique)  # Use primary query for reranking

    return reranked[:top_k]
```

---

## Production Operations

### Capacity Planning

```python
def estimate_resources(
    num_vectors: int,
    dimensions: int,
    metadata_size_bytes: int = 500
) -> dict:
    # Vector storage
    vector_size = dimensions * 4  # float32
    total_vector_storage = num_vectors * vector_size

    # Index overhead (HNSW ~1.5x)
    index_overhead = total_vector_storage * 1.5

    # Metadata
    metadata_storage = num_vectors * metadata_size_bytes

    # Total
    total_gb = (total_vector_storage + index_overhead + metadata_storage) / 1e9

    # QPS estimate (rough)
    qps_per_gb = 50  # depends heavily on config
    estimated_qps = total_gb * qps_per_gb

    return {
        "storage_gb": total_gb,
        "estimated_qps": estimated_qps,
        "recommended_replicas": max(1, int(total_gb / 50))  # ~50GB per replica
    }
```

### Index Maintenance

```python
class VectorDBMaintenance:
    def __init__(self, client):
        self.client = client

    def add_documents(self, documents: list[Document]):
        """Upsert documents with batching."""
        batch_size = 100
        for i in range(0, len(documents), batch_size):
            batch = documents[i:i + batch_size]
            embeddings = embed_batch([d.text for d in batch])

            self.client.upsert([
                {
                    "id": doc.id,
                    "vector": embedding,
                    "payload": doc.metadata
                }
                for doc, embedding in zip(batch, embeddings)
            ])

    def delete_documents(self, doc_ids: list[str]):
        """Delete by document ID."""
        self.client.delete(ids=doc_ids)

    def update_metadata(self, doc_id: str, metadata: dict):
        """Update metadata without re-embedding."""
        self.client.set_payload(
            collection_name="documents",
            payload=metadata,
            points=[doc_id]
        )
```

### High Availability

```
+-------------------------------------------------------------+
|                    Load Balancer                              |
+----------------------------+--------------------------------+
                             |
            +----------------+----------------+
            v                v                v
     +--------------+ +--------------+ +--------------+
     |  Replica 1   | |  Replica 2   | |  Replica 3   |
     |   (Read)     | |   (Read)     | |   (Primary)  |
     +--------------+ +--------------+ +--------------+
                                             |
                                       (Replication)
                                             |
                                       +-----v-----+
                                       |  Storage   |
                                       +-----------+
```

**Key patterns:**
- Leader-follower for writes
- Read replicas for query scaling
- Async replication for HA

### Patching

Treat the vector store like any other database on the CVE feed. pgvector 0.8.7 (October 1, 2026) fixes CVE-2026-103484, a buffer overflow during IVFFlat index builds that can lead to code execution for any database role allowed to create an index; earlier 2026 patches fixed a parallel HNSW build overflow (0.8.2) and possible index corruption during HNSW vacuuming (0.8.3). On managed Postgres, confirm the provider has shipped 0.8.7, and do not grant `CREATE INDEX` to application roles in multi-tenant schemas.

For Milvus, the floor is **3.0.3** (October 14, 2026). It rejects malformed sparse vectors in Search and HybridSearch that could cause a heap out-of-bounds write on the query node, and it stops DescribeCollection from showing metadata of collections the caller cannot access (3.0.1 had already fixed unauthenticated access through streaming gRPC calls on the external proxy port). Budget for its breaking changes in the same upgrade: the legacy `/api/v1/*` routes on the metrics port are gone (point health probes at `/healthz` or `/livez`), and binlog imports now need the cluster-level `ImportBinlog` privilege. Qdrant 1.20 changes a security default in the other direction: `service.enforce_internal_auth` is now on whenever an API key is configured, so verify internal cluster traffic carries credentials before you roll it out.

### Monitoring

```python
VECTOR_DB_METRICS = [
    "query_latency_p50",
    "query_latency_p99",
    "queries_per_second",
    "index_size_gb",
    "vector_count",
    "filter_latency",
    "upsert_latency",
    "cache_hit_rate"
]

def alert_rules():
    return {
        "query_latency_p99_high": {
            "condition": "query_latency_p99 > 500ms",
            "severity": "warning"
        },
        "query_latency_p99_critical": {
            "condition": "query_latency_p99 > 2000ms",
            "severity": "critical"
        },
        "low_recall": {
            "condition": "bench_recall < 0.90",
            "severity": "warning"
        }
    }
```

---

## Managed vs Self-Hosted (TCO Analysis)

### Cost Comparison

| Aspect | Pinecone (Serverless) | Self-Hosted (Qdrant/Milvus) |
|--------|-----------------------|-----------------------------|
| **Ops Overhead** | Zero | High (Requires K8s + SRE) |
| **Scaling** | Instant (Scale to zero) | Manual (Node provisioning) |
| **Cost (Small)** | $0 - $100/mo | $50/mo (Minimum instance) |
| **Cost (Scale)** | High per token/vector | Low unit cost |

### Managed Service Pricing (indicative, always verify on provider pages)

| Provider | Model | Example: 10M vectors, 1536 dims |
|----------|-------|--------------------------------|
| Pinecone | Pod-based or Serverless | ~$70-150/month serverless |
| Qdrant Cloud | Per GB | ~$50/month (20GB) |
| Weaviate Cloud | Per dimensions | ~$100/month |
| Zilliz (Milvus) | Per CU | ~$75/month |

### Managed RAG Pipelines (Build vs. Buy)

A newer option skips the vector database decision entirely: a managed pipeline that parses, chunks, embeds, indexes and retrieves, billed by ingested tokens, storage and queries. Cloudflare AI Search went GA on October 1, 2026, with pricing effective November 1, 2026:

| Meter | Price | Included per month (Workers plans) |
|---|---|---|
| Ingestion | $0.75 per 1M tokens (+$0.50 per 1M for images) | 5M tokens |
| Storage | $2.00 per GB-month | 10 GB-month |
| Semantic, vector or hybrid queries | $0.75 per 1K | 1,000 |
| Full-text queries | $0.10 per 1K | 1,000 |

The per-query meter dominates at scale: 10M hybrid queries a month is ~$7,500 in query fees alone, versus a self-hosted index whose marginal cost per query is close to zero once provisioned. Managed pipelines win on time-to-first-answer and for low or spiky volume; at steady high QPS, compare the query meter against node costs plus embedding API spend. Pinecone Nexus and Cohere Compass Cloud take the same "buy the pipeline" approach further; see [Agentic RAG](08-agentic-rag.md#compile-once-knowledge).

### Self-Hosted Costs

```python
def estimate_self_hosted_cost(
    vectors: int,
    dimensions: int,
    cloud: str = "aws"
) -> dict:
    storage_gb = (vectors * dimensions * 4 * 2.5) / 1e9  # 2.5x for index

    # Instance sizing: an in-RAM HNSW index must fit in memory with headroom
    if storage_gb < 12:
        instance = "r6g.large"  # 16 GB RAM
    elif storage_gb < 25:
        instance = "r6g.xlarge"  # 32 GB RAM
    elif storage_gb < 50:
        instance = "r6g.2xlarge"  # 64 GB RAM
    else:
        instance = "r6g.4xlarge"  # 128 GB RAM; past ~100 GB, shard, quantize, or use DiskANN

    return {
        "storage_gb": storage_gb,
        "instance": instance,
        "monthly_compute": instance_pricing[instance],
        "monthly_storage": storage_gb * 0.10,  # EBS
        "total_monthly": instance_pricing[instance] + storage_gb * 0.10
    }
```

### Decision: Managed vs Self-Hosted

| Factor | Managed | Self-Hosted |
|--------|---------|-------------|
| Ops overhead | Low | High |
| Cost at small scale | Higher | Lower |
| Cost at large scale | Variable | Often lower |
| Control | Less | Full |
| Compliance | Depends | Full control |
| Vendor lock-in | Yes | No (if open source) |

**Verdict**: Start with Serverless. Only self-host if you have >500M vectors or strict **On-Prem/GPU-Local** requirements. If the requirement is "data stays in our cloud account" rather than "we run it," BYOC (Pinecone, GA September 2026) now covers that without taking on operations.

---

## Selection Framework

### Decision Tree

```
Need < 100K vectors?
+-- Yes -> pgvector (if already using PostgreSQL)
|          +-- Chroma (for prototyping)
|
+-- No -> Need managed service?
          +-- Yes -> Cloud-first?
          |          +-- Yes -> Pinecone (easiest)
          |          +-- No -> Qdrant Cloud or Zilliz
          |
          +-- No -> Need enterprise features?
                    +-- Yes -> Milvus on Kubernetes
                    +-- No -> Qdrant or Weaviate self-hosted
```

### Evaluation Criteria

| Criterion | Weight | Questions to Ask |
|-----------|--------|------------------|
| Scale | High | How many vectors now? In 1 year? |
| Latency | High | What are p99 requirements? |
| Ops capacity | High | Can we operate this? |
| Cost | Medium | Budget constraints? |
| Features | Medium | Hybrid search? Multimodal? |
| Lock-in risk | Low-Medium | Open source preferred? |

### Proof of Concept Checklist

Before committing to a vector database:

- [ ] Load representative data volume
- [ ] Benchmark query latency at target QPS, with your real filters and top-k depth (vendor numbers rarely include restrictive filters; Qdrant-FineWeb-10B, released September 2026 with exact top-1,000 ground truth over 10.07B vectors, is a public benchmark at production scale)
- [ ] Test metadata filtering performance
- [ ] Verify update/delete performance
- [ ] Test failure recovery
- [ ] Evaluate monitoring and observability
- [ ] Calculate total cost of ownership

---

## Interview Questions

### Q: How would you choose between Pinecone and a self-hosted solution?

**Strong answer:**
Decision depends on several factors:

**Choose Pinecone when:**
- Team lacks ops capacity for stateful infrastructure
- Need to move quickly (days not weeks)
- Scale is moderate (under 100M vectors)
- Budget allows managed service premium
- Compliance allows cloud-vendor dependency

**Choose self-hosted (Qdrant, Milvus) when:**
- Have Kubernetes and ops expertise
- Cost sensitivity at scale
- Need full control over the software, not just data location
- Specific compliance requirements
- Want to avoid vendor lock-in

Two 2026 changes narrow the gap: Pinecone BYOC (GA September 23) runs the data plane in your own cloud account with no inbound access from Pinecone, and Pinecone's BM25 full-text search (GA September 9) removes the "we need a separate keyword engine" argument, though you still fuse lexical and vector results client-side.

For most startups, I would start with Pinecone or Qdrant Cloud for velocity, then evaluate migration if costs become prohibitive at scale. The switching cost is the re-embed and re-index, not the API: query surfaces are similar, but in-database embedding features tie you to the vendor's model choice.

### Q: Explain how HNSW works and when you would not use it.

**Strong answer:**
HNSW builds a hierarchical graph of vectors:

**How it works:**
1. Insert vectors as nodes in a multi-layer graph
2. Higher layers have fewer nodes, larger jumps
3. Search: start at top layer, greedily navigate to nearest neighbor
4. Descend layers until bottom (all vectors)

**Why it is good:**
- O(log n) query complexity
- No training required
- Supports real-time updates
- Excellent recall/latency tradeoff

**When not to use:**
- Very small datasets (<10K): brute force is fine
- Extremely memory constrained: HNSW keeps every full-precision vector plus its graph links in RAM
- Need exact search: HNSW is approximate
- Heavy update workload with tight latency: updates can cause temporary degradation

Alternatives:
- IVF-PQ for memory constraints
- DiskANN for billion-scale with cost efficiency
- Flat index for exact search
- LSH for very high-dimensional sparse vectors

### Q: When would you use a Disk-based index (like DiskANN) over a RAM-based index (HNSW)?

**Strong answer:**
I would use a Disk-based index when the memory cost of the index exceeds the budget or the capacity of a single high-memory node. For example, a 100-million-vector index with 1,536 dimensions needs roughly 700 GB of RAM per replica for HNSW (614 GB is the float32 vectors alone). Using DiskANN, I can keep the full vectors and the graph on NVMe SSDs and only compressed codes in RAM, reducing the RAM requirement by 90-95% while maintaining sub-10ms query times. This represents a massive TCO (Total Cost of Ownership) reduction for any workload that can absorb a few extra milliseconds per query, which covers most RAG traffic, since the LLM call dominates end-to-end latency.

### Q: Why is metadata filtering often the bottleneck in vector databases?

**Strong answer:**
In naive vector search, we find the "Top K" nearest neighbors and THEN filter them by metadata (e.g., "only documents from 2024"). If the filter is very restrictive, we might find 0 results after filtering. Specialized databases now use **Pre-Filtering with HNSW**, traversing the graph but only considering nodes that satisfy the boolean metadata constraint. This is computationally expensive because it breaks the "short-circuit" logic of HNSW, requiring specialized bitmasks or hardware acceleration (SIMD) to keep latencies low.

### Q: How do you handle multi-tenancy in a vector database?

**Strong answer:**
Three main approaches:

**1. Metadata filtering (most common):**
```python
results = db.search(
    vector=query,
    filter={"tenant_id": current_tenant}
)
```
- Pros: Simple, single index
- Cons: All tenants share resources, potential for bugs exposing data

**2. Collection per tenant:**
```python
results = db.collection(f"tenant_{tenant_id}").search(vector=query)
```
- Pros: Strong isolation, per-tenant scaling
- Cons: Many collections, operational overhead

**3. Namespace per tenant (Pinecone):**
```python
results = index.query(vector=query, namespace=tenant_id)
```
- Pros: Isolation within single index
- Cons: Vendor-specific

**I would choose:**
- Metadata filtering for most cases (simple, cost-effective)
- Separate collections for high-security requirements
- Never post-filter (retrieve all, filter after) due to leakage risk

**The hybrid-search catch:** BM25 scores depend on corpus-wide IDF statistics. In a shared collection, one large tenant's vocabulary shifts every other tenant's keyword scores, and term statistics leak across the boundary. Use an engine with per-tenant IDF (Qdrant 1.19 added it for sparse and BM25 search) or give large tenants their own collection.

---

## References

- Malkov and Yashunin. "Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs" (HNSW, 2018)
- Subramanya et al. "DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node" (Microsoft Research, NeurIPS 2019)
- [OpenSearch. "k-NN index: HNSW memory estimation"](https://docs.opensearch.org/2.9/search-plugins/knn/knn-index/)
- Pinecone Documentation: https://docs.pinecone.io/
- Pinecone. "The Managed Architecture of Serverless Vector DBs" (2024)
- Qdrant Documentation: https://qdrant.tech/documentation/
- Weaviate Documentation: https://weaviate.io/developers/weaviate
- Milvus Documentation: https://milvus.io/docs
- pgvector: https://github.com/pgvector/pgvector
- [Milvus 3.0.0 release notes (July 2026)](https://github.com/milvus-io/milvus/releases/tag/v3.0.0)
- [Qdrant 1.19 release (Aug 2026)](https://qdrant.tech/blog/qdrant-1.19.x/)
- [Weaviate 1.39 release (Aug 2026)](https://weaviate.io/blog/weaviate-1-39-release)
- [Qdrant 1.20.0 release notes (Oct 2026)](https://github.com/qdrant/qdrant/releases/tag/v1.20.0), [Weaviate 1.40.0 release notes (Oct 2026)](https://github.com/weaviate/weaviate/releases/tag/v1.40.0), and [Milvus 3.0.3 release notes (Oct 2026)](https://github.com/milvus-io/milvus/releases/tag/v3.0.3)
- [Elasticsearch release notes](https://www.elastic.co/docs/release-notes/elasticsearch)
- [turbopuffer. "RIP, vector database" (Sep 2026)](https://turbopuffer.com/blog/rip-vector-database)
- [Qdrant-FineWeb-10B benchmark (Sep 2026)](https://qdrant.tech/blog/qdrant-fineweb-10b-release/)
- [Cloudflare AI Search pricing](https://developers.cloudflare.com/ai-search/platform/limits-pricing/)

---

*Previous: [Embedding Models](03-embedding-models.md) | Next: [Hybrid Search](05-hybrid-search.md)*
