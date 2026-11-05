# Embedding Models

Embedding models convert text into high-dimensional vectors. The frontier has moved past static single-vector representations to **multi-resolution, late-interaction, and multimodal** embeddings, and in 2026 to **shared embedding spaces** that let you index with one model and query with another.

## Table of Contents

- [The Embedding Frontier (Matryoshka)](#the-embedding-frontier-matryoshka-embeddings)
- [Late Interaction (ColBERT v2)](#late-interaction-colbert-v2)
- [Binary and Int8 Quantization](#binary-and-int8-quantization)
- [Model Selection Criteria](#model-selection-criteria)
- [Shared and Asymmetric Embedding Spaces](#shared-and-asymmetric-embedding-spaces)
- [Multimodal Embeddings (Vision + Text)](#multimodal-embeddings)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Embedding Frontier: Matryoshka Embeddings

Traditionally, if you embedded text into 1,536 dimensions, you were stuck using all 1,536 dimensions for search. 

**Matryoshka Representation Learning (MRL)**
- Models are trained to "store" the most important info in the first few dimensions.
- **The Win**: You can embed at 1,536 dims, but index only a short prefix (64 to 256 dims) for a "fast search" pass, then refine the top results with the full 1,536 dims.
- **Efficiency**: An order-of-magnitude smaller first-pass index. The recall cost depends on the model and the truncation point (OpenAI reported text-embedding-3-large cut to 256 dims still beat ada-002 at 1,536), so measure it on your own queries before picking a prefix length.

Most current API models are Matryoshka-trained: Cohere Embed 5 (256 to 2,048), voyage-code-4 (256 to 2,048), gemini-embedding-2 (128 to 3,072), and OpenAI text-embedding-3 via the `dimensions` parameter.

---

## Late Interaction: ColBERT v2

Standard embeddings are "Bi-Encoders" (one vector per chunk). **ColBERT** (Contextualized Late Interaction over BERT) uses a "token-level" approach.

- **How**: Instead of 1 vector per chunk, ColBERT stores 1 vector **per token**.
- **Interaction**: At query time, the model compares every token in your query to every token in the documents (the "MaxSim" operation).
- **Status**: ColBERT v2 (and successors like ColPali, ColQwen2.5, ColNomic for documents and pages-as-images) is drastically compressed via PLAID indexing, making it feasible for production. It achieves much higher precision for "needle in a haystack" technical queries.
- **2026 serving path**: General-purpose engines now index multi-vector data directly, often through fixed-dimensional MUVERA encodings: Milvus 3.0 (EmbList plus DiskANN, July 2026) and Weaviate (MUVERA for multi-vector collections, extended to the disk-backed HFresh index in 1.40.0, released October 7, 2026, whose notes give HFresh no GA or preview label, so confirm its status before depending on it). You no longer need a dedicated ColBERT server to try late interaction. See [Late Interaction and ColBERT](11-late-interaction-colbert.md).

---

## Binary and Int8 Quantization

Storing `float32` vectors is expensive. Production indexes lean heavily on **quantization**, either emitted by the model or applied in the database.

- **Binary Embeddings**: Convert vectors to 1s and 0s. 
  - **Memory**: 32x reduction.
  - **Speed**: Hamming distance (XOR operations) is 10x faster than Cosine similarity on modern CPUs.
- **Int8 and binary from the model**: Cohere Embed 5 (float, int8, binary), voyage-code-4 (float32, int8, uint8, binary) and Perplexity's pplx-embed-v2 preview (native int8) return quantized vectors directly, designed for quantized use rather than post-hoc rounding. OpenAI's text-embedding-3 models return floats only, so you quantize them in the database.
- **Low-bit quantization in the database**: Qdrant TurboQuant and Turbo4 and Weaviate 4-bit Rotational Quantization apply a random or Hadamard rotation before scalar quantization; both shipped in 2026 as opt-in or preview options, not defaults. Elasticsearch's BBQ family is the exception: it stores roughly 1 bit per dimension, and DiskBBQ (`bbq_disk`) is the default index type for float vectors from 9.4 where the license includes it. The mechanics and RAM math are in [Vector Databases](04-vector-databases.md#low-bit-and-rotation-based-quantization).

---

## Model Selection Criteria

| Model | Provider | Features | Context |
|-------|----------|----------|---------|
| **gemini-embedding-2** | Google | Natively multimodal: text, up to 6 images, video up to 120 s, audio, PDFs up to 6 pages, interleaved inputs; 128 to 3,072 dims. Stable since April 2026; replaces text-only gemini-embedding-001 (shuts down May 14, 2028) | 8,192 text tokens |
| **Cohere Embed 5** (`embed-v5.0-pro` / `-fast`) | Cohere | September 30, 2026. Text, image and fused text+image; Pro and Fast share one space; float/int8/binary; $0.12 / $0.08 per 1M text tokens | 128K |
| **Voyage 4** (`voyage-4-large` / `voyage-4` / `voyage-4-lite`) | Voyage AI (MongoDB) | January 15, 2026. One shared space across sizes; $0.12 / $0.06 / $0.02 per 1M; open-weight `voyage-4-nano` | 32K |
| **voyage-code-4** | Voyage AI | August 13, 2026. Trained on natural-language-to-code pairs mined from pull requests, for agents that search from a bug symptom; $0.12 per 1M | 32K |
| **voyage-multimodal-3.5** | Voyage AI | January 15, 2026. Interleaved text, images and video; 256 to 2,048 dims; float, int8 and binary output | 32K |
| **EmbeddingGemma 2** | Google (open weights, Apache 2.0) | October 2026. 740M parameters (270M text, 170M vision and 300M audio encoders); text, code, images, video and audio in one 768-dim space, Matryoshka truncation to 512, 256 or 128 dims (validate 128 carefully on multimodal data); MTEB multilingual v2 61.36 and MTEB code 78.68 (vendor-reported; no RTEB score yet) | 8K |
| **Nemotron 3 Embed 8B / 1B** | NVIDIA (open weights, OpenMDW-1.1) | July 16, 2026. 8B: 4,096 sliceable dims, RTEB Multilingual #1 at 78.5 (vendor-reported); 1B: 2,048 dims, NVFP4 variant | 32,768 (8B) |
| **Qwen3-Embedding-8B** | Open weights | Instruction-tuned, long-doc strength; 70.58 MTEB Multilingual mean, but the top spot is now contested | 32K |
| **Llama-Embed-Nemotron-8B** | NVIDIA | October 2025. Strong multilingual scores, 4,096 dims; weights are licensed for non-commercial and research use only | 32K |
| **Cohere Embed v4** | Cohere | Multimodal (text + image), Matryoshka, binary quantization; still available, no deprecation announced | 128k |
| **OpenAI text-embedding-3-large** | OpenAI | Matryoshka via `dimensions`, 3,072 dims, $0.13 per 1M, float output only. OpenAI has shipped no newer embedding model; there is no "text-embedding-4" | 8,192 |
| **BGE-M3** | Open Source | Multilingual, multi-granularity (dense + sparse + late-interaction) | 8k |
| **Jina-Embeddings-v3** | Jina AI | Task-specific LoRA adapters, Matryoshka; pair with a Jina ColBERT model for late interaction | 8k |

Open-weight models (Nemotron 3 Embed, Qwen3, BGE) now match or beat the commercial APIs on public benchmarks. Pick commercial when you want managed infra and SLAs; pick open weights when cost-per-query at high volume matters more than latency floor.

**How to read the leaderboards:**
- **Prefer RTEB over MTEB-only rankings.** RTEB (from the MTEB team, October 2025) mixes open datasets with private held-out sets that the maintainers evaluate, which counters training on the test. MTEB Multilingual's top spot has changed hands several times (Qwen3-Embedding-8B, then llama-embed-nemotron-8b, then KaLM-Embedding-Gemma3-12B in claimed rankings), so check the live board rather than quoting a leader.
- **Do not replace the embedder with an LLM.** "The Embedder's Dilemma" (arXiv 2608.12875, August 2026) compared 10 LLMs with 26 embedding models on 37 MTEB-style tasks. They tie in aggregate (best LLM 77.6, best embedder 77.2), but parity costs up to 1,431x more per benchmark pass, and open LLMs ran 2.5x to 736x slower on the same GPU. LLMs lead on reasoning-heavy retrieval and embedders on classification. That points to where an LLM belongs: a late-stage reranker for hard queries, not stage one.
- **Code search for agents is its own category.** Agent queries look like vague bug reports, not identifiers. Voyage reports voyage-code-4 at +27.54% over voyage-code-3 on a 19-benchmark agentic code retrieval suite built from held-out repositories (vendor-reported; independent RTEB results pending).

---

## Shared and Asymmetric Embedding Spaces

The classic rule was "new embedding model means full re-index." Two 2026 patterns break it:

- **Model families in one shared space.** The Voyage 4 family and Cohere Embed 5 Pro/Fast are trained so their vectors are interchangeable. Index the corpus once with the large model (best document representations), then serve queries with the small one (lowest query latency and cost), or move query traffic between sizes without touching the index.
- **Swappable query encoders.** Qdrant's Constella research preview (September 29, 2026) keeps a fixed Stella (400M) document index and swaps query encoders. Constella Nano (34.5M parameters) keeps about 91% of Stella's BEIR-15 nDCG@10 (0.5081 vs 0.5614) at 12x lower CPU latency (3.1 ms vs 39.0 ms); Constella Zero, a token lookup with no transformer, scores 0.4572 at 0.081 ms (vendor-reported research preview).

The constraint: shared spaces are vendor-defined. You cannot mix a Voyage index with a Cohere query encoder, and preview models warn that their vectors will not be compatible with later releases (Perplexity's September 2026 pplx-embed-v2-context-9b-preview did; the non-preview 9B and 0.6B models that replaced it in October share one space with each other, so anything indexed on the preview must be re-embedded).

---

## Multimodal Embeddings

Text-only RAG silently throws away the charts, tables, diagrams, and layout signal that often hold the answer. Modern stacks treat pages, screenshots, and figures as first-class retrieval objects:

- **Unified vision-text embeddings**: Cohere Embed 5, voyage-multimodal-3.5 and gemini-embedding-2 each map images and text into a single vector space, so you can query "where is the emergency shutoff valve?" against schematics. On ViDoRe V3, Cohere reports Embed 5 Pro 85.8, Fast 84.5, voyage-4-large 83.7, Gemini Embedding 2 83.2 and text-embedding-3-large 75.5 (vendor-reported). voyage-multimodal-3.5 and gemini-embedding-2 also take video, and gemini-embedding-2 adds audio, in the same space. EmbeddingGemma 2 (October 2026) brings all four modalities to open weights under Apache 2.0 at 740M parameters, so a team that cannot send audio or video to an API can now self-host one shared space. Its 8K context and 768 dimensions are modest next to the hosted models; benchmark it on your own media before assuming parity.
- **Page-as-image with late interaction**: ColPali, ColQwen2.5, and ColNomic embed each page render directly, skipping fragile OCR and preserving visual hierarchy.
- **CLIP-family models**: Still useful for image-heavy catalogs (e-commerce, media) where text-image alignment is the core signal.

---

## Interview Questions

### Q: What is the "Vocabulary Mismatch" problem in embeddings?

**Strong answer:**
Embeddings rely on the semantic space learned during training. If a user query uses a newer term (e.g., a model name released after the embedding model's cutoff) that wasn't in the embedding model's training set, the model might assign it a generic "AI" vector, missing the specific nuances. The standard fix is **Hybrid Search** (using BM25 to catch the specific keyword) plus **Cross-Encoder Reranking**, which handles out-of-distribution vocabulary better by looking at query and document tokens simultaneously.

### Q: Why would you choose a Matryoshka model for a 1-billion-vector index?

**Strong answer:**
Scaling to 1 billion vectors with standard `float32` 1536-dim embeddings requires ~6TB of high-speed RAM for an HNSW index, which is prohibitively expensive. With a Matryoshka model, I can use the first 128 dimensions (Binary quantized) for the initial retrieval. This reduces the memory footprint by over 90%, allowing the "Top 1,000" candidates to be found on significantly cheaper hardware. I can then fetch the full-resolution vectors for just those 1,000 candidates to perform the final reranking.

### Q: You need to upgrade the embedding model behind a 500M-vector index. How do you do it without downtime?

**Strong answer:**
First, decide whether a full re-embed is actually required:
1. **Same shared-space family** (Voyage 4 sizes, Cohere Embed 5 Pro and Fast): no re-index. Move query traffic to the new size and A/B it.
2. **Different model**: a full re-embed. At 500M chunks of ~500 tokens that is 250B tokens, about $30,000 at $0.12 per 1M before batch discounts, plus the write load on the index. Budget it like a migration, not a config change.

For the re-embed itself I run **dual-write, backfill, shadow-read, cut over**: new writes go to both vector fields, a background job backfills the new field oldest-first, a shadow path queries the new field and logs recall@k against the golden set, and traffic flips per tenant once the new field matches or beats the old one. Engines now make this easier: Milvus 3.0 adds and backfills columns online, so the new embedding is just another field. I keep the old field until the rollback window closes.

If the database embeds for me (MongoDB Atlas Automated Embedding, turbopuffer native embedding, Elasticsearch `semantic_text`), I lose the sync pipeline but the vendor now controls the model version, so I check how they handle re-embedding on a model change before I commit.

---

## References
- Kusupati et al. "Matryoshka Representation Learning" (2022/2024 update)
- Khattab and Zaharia. "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT" (2020); Santhanam et al. "ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction" (2022)
- OpenAI. "New embedding models and API updates" (January 2024)
- [Google. Gemini Embedding 2 model page](https://ai.google.dev/gemini-api/docs/models/gemini-embedding-2)
- [Cohere. "Embed 5" (Sep 2026)](https://cohere.com/blog/embed-5)
- [Voyage AI. "voyage-4" (Jan 2026)](https://blog.voyageai.com/2026/01/15/voyage-4/) and ["voyage-code-4" (Aug 2026)](https://blog.voyageai.com/2026/08/13/voyage-code-4/)
- [NVIDIA. Nemotron 3 Embed 8B model card](https://huggingface.co/nvidia/Nemotron-3-Embed-8B-BF16)
- [Google. EmbeddingGemma 2 model card](https://huggingface.co/google/embeddinggemma-2)
- ["The Embedder's Dilemma" (arXiv 2608.12875, Aug 2026)](https://arxiv.org/abs/2608.12875)

---

*Previous: [Chunking Strategies](02-chunking-strategies.md) | Next: [Vector Databases](04-vector-databases.md)*
