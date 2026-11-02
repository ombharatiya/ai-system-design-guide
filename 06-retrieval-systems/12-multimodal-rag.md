# Multi-Modal RAG

Multi-modal RAG extends retrieval-augmented generation beyond plain text to handle images, tables, charts, audio, and mixed-layout documents. Production systems now routinely ingest PDFs with diagrams, slide decks, scanned invoices, and research papers where the visual layout *is* the meaning. Three architectures dominate: caption-and-index, unified vision-text embeddings (Cohere Embed 5, Voyage-Multimodal-3.5, Gemini Embedding 2), and page-as-image with late interaction (ColPali, ColQwen2.5, ColNomic). Video is the newest modality to get its own cost model, now that models can choose which segments to watch instead of sampling every frame.

## Table of Contents

- [Why Text-Only RAG Fails](#why-text-only-rag-fails)
- [Architecture Patterns](#architecture-patterns)
- [Multi-Modal Embedding Strategies](#multi-modal-embedding-strategies)
- [Vision-Language Models for Document Understanding](#vision-language-models-for-document-understanding)
- [ColPali and Vision-Based Retrieval](#colpali-and-vision-based-retrieval)
- [Table Extraction and Structured Data Retrieval](#table-extraction-and-structured-data-retrieval)
- [Chart and Diagram Understanding](#chart-and-diagram-understanding)
- [Video: Fixed Sampling vs Agentic Processing](#video-fixed-sampling-vs-agentic-processing)
- [Production Architecture](#production-architecture)
- [Implementation Example](#implementation-example)
- [System Design Interview Angle](#system-design-interview-angle)
- [References](#references)

---

## Why Text-Only RAG Fails

Traditional RAG pipelines parse documents into text chunks, embed them, and retrieve against a text query. This breaks on real-world documents:

| Document Element | Text-Only RAG Behavior | Actual Information Lost |
|-----------------|----------------------|------------------------|
| **Bar Chart** | Extracts axis labels only | Trends, comparisons, magnitudes |
| **Architecture Diagram** | Misses entirely | Component relationships, data flow |
| **Table** | Flattened rows lose structure | Row-column associations, headers |
| **Infographic** | Captures scattered text fragments | Visual hierarchy, spatial groupings |
| **Photo with Caption** | Gets caption, loses image | Visual evidence, spatial context |

**Reality**: Enterprise documents are 40-60% non-textual content. A financial report's value is in its charts. A medical paper's key finding is in its figures. Ignoring visual content means ignoring most of the knowledge.

---

## Architecture Patterns

There are three dominant patterns for multi-modal RAG, each with distinct trade-offs:

### Pattern 1: Unified Embedding Space

```
                     Shared Vector Space
                    +-------------------+
  Text  --> Encoder |  [0.2, 0.8, ...] |
  Image --> Encoder |  [0.3, 0.7, ...] |  --> Single Index --> Retrieve
  Table --> Encoder |  [0.1, 0.9, ...] |
                    +-------------------+

  Query "show revenue trends" --> encode --> nearest neighbors across ALL modalities
```

- **How**: Use a model like CLIP or SigLIP for natural images, or a document-grade multimodal embedder (Cohere Embed 5, Gemini Embedding 2, Voyage Multimodal) for pages, slides and screenshots, to project text and images into the same vector space.
- **Pros**: Single index, single query, simple retrieval logic.
- **Cons**: Embedding quality varies across modalities; tables need serialization.

### Pattern 2: Modality-Specific Retrieval with Fusion

```
  Query --> +----> Text Index    --> Top-K text chunks
            |
            +----> Image Index   --> Top-K images
            |
            +----> Table Index   --> Top-K tables
            |
            v
        Fusion / Reranking Layer --> Combined Top-K --> VLM Generator
```

- **How**: Separate embeddings and indices per modality. A reranker or reciprocal rank fusion (RRF) merges results.
- **Pros**: Best-in-class embeddings per modality; can tune each retriever independently.
- **Cons**: More infra complexity; fusion logic is non-trivial.

### Pattern 3: Vision-First (Page-as-Image)

```
  Document Page --> Screenshot/Render --> Vision Encoder --> Multi-vector Index
                                              |
  Query ---------> Text Encoder --------------+---> Late Interaction Score
                                                    --> Retrieve top pages
```

- **How**: Treat every document page as an image. Use a vision-language model (e.g., ColPali) to create patch-level embeddings. Score via late interaction (MaxSim).
- **Pros**: No OCR, no layout parsing, no table extraction pipeline. End-to-end trainable.
- **Cons**: Higher compute at indexing; loses fine-grained text search.

**Recommendation**: Pattern 3 (vision-first) is gaining ground fast for document-heavy use cases. Pattern 2 remains the production workhorse when you need precise text search alongside visual retrieval.

---

## Multi-Modal Embedding Strategies

### CLIP (Contrastive Language-Image Pretraining)

The original dual-encoder that maps text and images to a shared 512/768-dim space.

- **Strengths**: Huge ecosystem, well-understood, many fine-tuned variants.
- **Weaknesses**: Weaker on document-style images (charts, tables) vs. natural photos. Contrastive loss requires large batch sizes.

### SigLIP / SigLIP 2

Replaces CLIP's softmax cross-entropy with a sigmoid loss, allowing each image-text pair to be evaluated independently.

- **SigLIP 2 (2025)**: Adds captioning decoders, self-distillation, and masked prediction. Trained on 10B+ images across 109 languages.
- **Key Win**: Outperforms CLIP at small batch sizes (4-8k) and provides denser, more robust features.
- **Production Use**: National Library of Norway, e-commerce visual search, AI art curation.

### Comparison for RAG

| Model | Best For | Embedding Dim | Document Quality | Natural Image Quality |
|-------|----------|--------------|-----------------|----------------------|
| CLIP ViT-L/14 | General purpose | 768 | Medium | High |
| SigLIP 2 So400m | Multi-lingual docs | 1152 | High | High |
| Nomic Embed Vision | Text-heavy docs | 768 | High | Medium |
| Voyage Multimodal 3.5 | Mixed documents; text, images and video | 256-2048 (1024 default) | High | High |
| Cohere embed-v5.0-pro / -fast | Visually rich enterprise docs; text, image, or fused text+image input | 256-2048 (Matryoshka) | High: ViDoRe V3 85.8 / 84.5 (vendor-reported) | Not reported |
| Gemini Embedding 2 | One space for text, images, video, audio and PDFs | 128-3072 (768/1536/3072 recommended) | High: ViDoRe V3 83.2 (Cohere's vendor-run table) | Not reported |
| EmbeddingGemma 2 (open weights, Apache 2.0) | Self-hosted single space for text, code, images, video and audio; 740M parameters, 8K context | 128-768 (Matryoshka) | Not reported on ViDoRe | Not reported |

**What changed in 2026**: two hosted multimodal embedders now cover the "unified space" pattern without a separate image model, and in October an open-weight one joined them.
- **Gemini Embedding 2** (`gemini-embedding-2`, announced March 10, stable since April 2026) is Google's multimodal model: text up to 8,192 tokens, up to 6 images, video up to 120 seconds, native audio, and PDFs up to 6 pages per input, including interleaved inputs. `gemini-embedding-001` is text-only and shuts down May 14, 2028; Google names Gemini Embedding 2 as its replacement.
- **Cohere Embed 5** (September 30, 2026): `embed-v5.0-pro` at $0.12 and `embed-v5.0-fast` at $0.08 per 1M text tokens, $0.40 per 1M image tokens, 128K context. Pro and Fast share one embedding space, so you can index with Pro and query with Fast without re-indexing.
- **EmbeddingGemma 2** (October 2026, Apache 2.0): 740M parameters split across a 270M text model, a 170M vision encoder and a 300M audio encoder, mapping text, images, video and audio into one 768-dim space. It is the self-hosted answer when audio or video cannot leave your network, but Google publishes MTEB text and code scores (vendor-reported), not ViDoRe, so run your own page-level eval before choosing it for documents.

Cohere's own ViDoRe V3 table puts Embed 5 Pro at 85.8, Fast 84.5, voyage-4-large 83.7, Gemini Embedding 2 83.2 and OpenAI text-embedding-3-large 75.5. The spread among the top four is under three points and vendor-run, so it does not settle a choice; your own page-level eval set does.

### Embedding Strategy Decision

```
Is your content mostly natural images (photos, products)?
  YES --> CLIP or SigLIP fine-tuned on your domain
  NO
    |
    v
Is your content document pages (PDFs, slides, reports)?
  YES --> Baseline: single-vector multimodal embedder (Embed 5,
          Gemini Embedding 2) + reranker. Move to ColPali / ColQwen
          (multi-vector, no OCR) if page-level recall falls short
  NO
    |
    v
Is it a mix of text, images, and structured data?
  YES --> Modality-specific encoders + fusion (Pattern 2)
          or one hosted multimodal embedder (Embed 5, Gemini Embedding 2)
          if a single index matters more than per-modality tuning

Does it include audio or video clips?
  YES --> Gemini Embedding 2 for retrieval (clips up to 120 s per input),
          or EmbeddingGemma 2 if the media must stay self-hosted,
          plus transcripts in the text index for exact-term search
```

---

## Vision-Language Models for Document Understanding

VLMs serve two roles in multi-modal RAG: (1) as the **generator** that synthesizes answers from retrieved multi-modal context, and (2) as the **indexing engine** that extracts structured information at ingestion time.

### VLM Options for Document Understanding

At the frontier, chart reading, table extraction and diagram understanding are close enough that the ranking on *your* documents depends on your page mix. The differences that drive design are price, long-context billing and API behavior:

| | Claude Opus 5.5 / Sonnet 5.5 | GPT-6 Astra / GPT-6 Sol | Gemini 3.8 Flash |
|---|---|---|---|
| **Price per 1M (in / out)** | $4 / $20; $2 / $10 | $10 / $50; $2 / $10 | $0.75 / $3.75 intro, $1.50 / $7.50 from January 1, 2027 |
| **Context** | 1M, flat price to 1M | 1.05M; above 272K input the whole request bills at long-context rates | 1,048,576 in / 65,536 out |
| **Structured output** | Native (`output_config.format`) | Native | Native |
| **Watch-outs** | Thinking is always on (Opus 5.5) or on by default (Sonnet 5.5); parse the text block, not `content[0]` | OpenAI fixed an image-encoding bug that degraded image understanding in GPT-6 Sol and Luna on September 25, 2026, under the same model IDs: rerun image evals run before the fix | Intro price ends December 31; budget on the 2027 list price |

Specialist document parsers (Cohere Parse, MinerU, jina-ocr-v1, Nemotron Parse 2.0) now return Markdown with tables, bounding boxes and reading order at a fraction of frontier-VLM cost per page. For the extraction pass at volume, compare them first; see [OCR and Layout Analysis](../10-document-processing/01-ocr-and-layout.md).

### VLM-Augmented Ingestion Pipeline

```
  Raw PDF
    |
    v
  Page Renderer (pdf2image, 300 DPI)
    |
    v
  VLM Extraction Pass:
    +-- "Extract all tables as markdown"
    +-- "Describe this chart: axes, trends, key data points"
    +-- "Summarize the diagram: components and relationships"
    |
    v
  Structured Output (JSON)
    |
    +---> Text chunks     --> Text embedding index
    +---> Table markdown  --> Text embedding index (with metadata: "type=table")
    +---> Chart summaries --> Text embedding index (with metadata: "type=chart")
    +---> Page images     --> Image embedding index (CLIP/SigLIP)
```

This "describe-then-embed" approach converts visual content into searchable text while preserving the original image for the generation step.

---

## ColPali and Vision-Based Retrieval

ColPali represents a paradigm shift: instead of building complex OCR + layout + table extraction pipelines, treat each document page as a single image and let a vision-language model handle everything.

### How ColPali Works

```
  Document Page Image
        |
        v
  SigLIP Vision Encoder (So400m)
        |
  Splits image into patches (e.g., 32x32 grid = 1024 patches)
        |
        v
  Gemma 2B Language Model (contextualizes patch embeddings)
        |
        v
  Linear Projection --> 128-dim patch embeddings
        |
  Result: 1024 vectors of dim 128 per page
        |
        v
  Stored in Multi-Vector Index

  At query time:
  Query --> Tokenize --> Embed --> 128-dim token embeddings
        |
        v
  Late Interaction (MaxSim):
    Score = Sum over query tokens of Max similarity to any patch
```

### ColPali vs. Traditional Pipeline

| Aspect | Traditional Pipeline | ColPali |
|--------|---------------------|---------|
| **OCR** | Required (Tesseract, Azure OCR) | Not needed |
| **Layout Detection** | Required (Detectron2, LayoutLM) | Not needed |
| **Table Parser** | Required (Camelot, Tabula) | Not needed |
| **Chart Extractor** | Required (ChartOCR) | Not needed |
| **Indexing Speed** | Slow (multi-stage) | Fast (single forward pass) |
| **Retrieval Quality** | High on text, poor on visuals | High across all modalities |
| **Storage** | Text index (~small) | Multi-vector index (~larger) |

### ColPali Family

- **ColPali (v1)**: PaliGemma-3B backbone. The original.
- **ColQwen2 / ColQwen2.5**: Qwen2-VL and Qwen2.5-VL backbones. Better multilingual support, improved on Asian-language documents.
- **ColSmol**: Smaller variants for edge deployment, under 1B parameters.

### ViDoRe Benchmark Results

ColPali excels on visually complex benchmarks like InfographicVQA, ArxivQA, and TabFQuAD, which test infographics, figures, and tables respectively. It outperforms traditional text-based pipelines even on text-centric documents.

ViDoRe has since moved to V3, which is what vendors now quote for single-vector multimodal embedders (see the table above). That matters for the architecture choice: a single-vector page embedding scoring in the mid-80s on ViDoRe V3 (vendor-reported) is far cheaper to serve than 1,024 patch vectors per page, so test a single-vector multimodal embedder plus a reranker before committing to a multi-vector index.

---

## Table Extraction and Structured Data Retrieval

Tables are the hardest modality for traditional RAG. Flattening a table row-by-row destroys the column-header relationships that give each cell meaning.

### Strategy 1: VLM-Based Extraction

```python
# Pseudocode: Extract tables using a VLM
def extract_tables_from_page(page_image: bytes) -> list[dict]:
    prompt = """
    Extract ALL tables from this document page.
    For each table, return:
    {
      "title": "table title or caption",
      "headers": ["col1", "col2", ...],
      "rows": [["val1", "val2", ...], ...],
      "markdown": "| col1 | col2 |\\n|---|---|\\n| val1 | val2 |"
    }
    Return JSON array. If no tables, return [].
    """
    response = vlm.generate(image=page_image, prompt=prompt)
    return json.loads(response)
```

### Strategy 2: Specialized Table Parsers

- **Tabula / Camelot**: Rule-based PDF table extraction. Fast but brittle on complex layouts.
- **Table Transformer (DETR-based)**: Detects table boundaries and cell structure from images.
- **Unstructured.io**: Combines heuristics with ML models for layout-aware parsing.
- **Document-parsing VLMs** (Cohere Parse, MinerU 4.0, Docling with pluggable VLM backends): emit tables inline as HTML or Markdown together with bounding boxes and reading order, which keeps table-to-page citations intact. See [OCR and Layout Analysis](../10-document-processing/01-ocr-and-layout.md).

### Strategy 3: Table-Aware Chunking

```
  Original Table (20 rows x 8 columns)
        |
        v
  Chunk as complete unit (do NOT split tables across chunks)
        |
        v
  Embed the full markdown table as a single chunk
        |
        v
  Add metadata: {"type": "table", "page": 14, "caption": "Q3 Revenue by Region"}
        |
        v
  At generation time: pass the FULL table to the LLM, not a fragment
```

**Key Principle**: Tables must be atomic retrieval units. Never split a table across chunk boundaries.

---

## Chart and Diagram Understanding

### Chart Types and Extraction Approaches

| Chart Type | What to Extract | Best Approach |
|-----------|----------------|---------------|
| **Bar/Line/Pie** | Data values, trends, comparisons | VLM description + data table extraction |
| **Flow Diagram** | Steps, decisions, connections | VLM structured extraction (nodes + edges) |
| **Architecture Diagram** | Components, relationships, data flow | VLM description + entity extraction |
| **Scatter Plot** | Correlations, outliers, clusters | VLM trend description + raw data if available |
| **Gantt Chart** | Timeline, dependencies, milestones | VLM structured extraction |

### Dual-Representation Strategy

For each chart or diagram, store TWO representations:

```
  Chart Image
    |
    +---> (1) Text Description (for text-based retrieval)
    |         "This bar chart shows Q3 revenue by region.
    |          North America: $4.2M, Europe: $3.1M, APAC: $2.8M.
    |          NA grew 15% QoQ while APAC declined 3%."
    |
    +---> (2) Original Image (for visual retrieval + generation context)
              Stored with CLIP/SigLIP embedding for image-based queries
```

This ensures the chart is retrievable by both text queries ("what was APAC revenue?") and visual queries ("show me the revenue chart").

---

## Video: Fixed Sampling vs Agentic Processing

Video RAG cost models used to be simple arithmetic: duration x frame rate x tokens per frame. Gemini's default static processing samples at 1 FPS and uses roughly 100 tokens per second of video at default (low) media resolution, so an hour is about 360K tokens, or about $0.27 of input on Gemini 3.8 Flash at its intro price ($0.54 at the 2027 list price). A 1M-context model takes up to about 3 hours at low resolution, or 1 hour at high resolution.

On September 1, 2026 Google added **agentic video processing** to Gemini 3.7 Flash, 3.6 Flash and 3.5 Flash-Lite, and the docs now list Gemini 3.8 Flash as well: set `"processing": "agentic"` on the video input and the model uses internal video tools to decide which segments to inspect, at what speed, and through which modality (frames, audio or transcript). It is opt-in (static 1 FPS stays the default), works for uploads and YouTube URLs, and bills at standard token prices with no feature fee. Google reports up to 88% fewer tokens, up to 66% lower cost and up to 7% higher accuracy on standard video benchmarks, with the largest gains on long-form video (vendor-reported).

```
STATIC (default):   video ──► every frame at 1 FPS + audio ──► context ──► answer
                    cost scales with duration

AGENTIC (opt-in):   video + question ──► model picks segments, speed, modality
                                         (frames / audio / transcript) ──► answer
                    cost scales with how much of the video the question needs
```

Design implications:
- **Budget per question type, not per hour.** "What happens at the end?" touches far less video than "list every safety violation". Meter actual tokens per query class before setting quotas, and expect the up-to-88% saving mostly on long videos.
- **Retrieval moved inside the model for single-video QA.** Agentic processing replaces your own scene detection, keyframe sampling and transcript chunking when the question targets one video. Searching a library still needs an index: transcripts and scene captions in the text index, short clips (up to 120 seconds per input) embedded with Gemini Embedding 2, then the top videos handed to the model.
- **Evaluate coverage, not just accuracy.** Because the model chooses what to watch, ask for timestamped citations and spot-check them against the source on a sample of queries; a confident answer built from the wrong segment is the video version of a retrieval miss.

---

## Production Architecture

### Full Multi-Modal RAG Pipeline

```
  INGESTION:
  Raw Docs --> Doc Classifier --+--> Text-Heavy  --> chunking + text embeddings
                                +--> Visual-Heavy --> page render + ColPali
                                +--> Mixed        --> VLM extraction + hybrid
                                         |
                                         v
                          [Text Index] [Image Index] [Table Index]

  RETRIEVAL:
  Query --> Query Analyzer --+--> Text:  BM25 + dense search
                             +--> Image: CLIP/ColPali search
                             +--> Table: metadata-filtered dense
                                    |
                                    v
                             Cross-Modal Reranker --> Context Assembly --> VLM --> Response
```

### Scaling Considerations

| Concern | Solution |
|---------|----------|
| **Index Size** | ColPali stores ~1024 vectors/page. For 1M pages = ~1B vectors. Use quantization (binary, PQ), MUVERA-style single-vector encodings for candidate generation (Milvus 3.0; Weaviate 1.31+, with the disk-backed HFresh path added in 1.40), or fewer vectors per page (see [Late Interaction & ColBERT](11-late-interaction-colbert.md#compressing-the-vectors-themselves)). |
| **Ingestion Latency** | Frontier VLM extraction is slow (~2-5s/page). Specialist parsers are faster: jina-ocr-v1 runs 2.57 pages/s on one A100 at concurrency 32; Cohere reports 4.5 pages/s per GPU for Parse (both vendor-reported). Use async workers. |
| **Query Latency** | Multi-index fan-out adds latency. Use parallel retrieval + aggressive top-k pruning. |
| **Cost** | Ingestion is one-time; amortize over query volume. At October 2026 list prices, assuming ~1.7K input and ~800 output tokens per page: ~$0.004/page on Gemini 3.8 Flash (intro price), ~$0.011/page on Claude Sonnet 5.5 or GPT-6 Sol, before reasoning tokens; Batch APIs halve both. Cohere Parse lists $1.50 per 1,000 pages. |
| **Storage** | Store page images in object storage (S3). Store embeddings in vector DB. Store text in search index. |

---

## Implementation Example

### End-to-End Multi-Modal RAG with ColPali + VLM

```python
# Pseudocode: Production multi-modal RAG pipeline

import uuid
from colpali_engine import ColPali, ColPaliProcessor
from qdrant_client import QdrantClient
from qdrant_client.models import PointStruct, Filter, FieldCondition, MatchValue
import anthropic

# --- INDEXING ---

def index_document(pdf_path: str, collection: str):
    """Index a PDF document using ColPali for visual retrieval
    and VLM extraction for text-based retrieval."""

    pages = render_pdf_to_images(pdf_path, dpi=300)

    colpali_model = ColPali.from_pretrained("vidore/colpali-v1.3")
    processor = ColPaliProcessor.from_pretrained("vidore/colpali-v1.3")
    vlm_client = anthropic.Anthropic()

    for page_num, page_image in enumerate(pages):
        # 1. Generate ColPali multi-vector embeddings
        inputs = processor(images=[page_image])
        patch_embeddings = colpali_model(**inputs)  # shape: [1, 1024, 128]

        # 2. Extract structured content via VLM
        extraction = vlm_client.messages.create(
            model="claude-sonnet-5-5",
            max_tokens=16000,                  # thinking is on by default; leave headroom
            output_config={"effort": "low"},   # transcription, not reasoning
            messages=[{
                "role": "user",
                "content": [
                    {"type": "image", "source": encode_image(page_image)},
                    {"type": "text", "text": """Extract from this page:
                    1. All text content (preserve structure)
                    2. Tables as markdown
                    3. Chart descriptions with data points
                    Return as JSON with keys: text, tables, charts"""}
                ]
            }]
        )

        # Current Claude models return thinking blocks before the text block,
        # so content[0] is not the answer. Read the text block explicitly.
        text = next(b.text for b in extraction.content if b.type == "text")
        structured = json.loads(text)

        # 3. Store in vector DB
        qdrant.upsert(collection, points=[
            # ColPali multi-vector for visual retrieval
            PointStruct(
                id=str(uuid.uuid5(uuid.NAMESPACE_URL, f"{pdf_path}#page={page_num}")),
                vector={"colpali": patch_embeddings[0].tolist()},
                payload={
                    "source": pdf_path,
                    "page": page_num,
                    "type": "page_image",
                    "text_preview": structured["text"][:500]
                }
            ),
            # Text embeddings for each extracted element
            *create_text_chunks(structured, pdf_path, page_num)
        ])


# --- RETRIEVAL ---

def retrieve(query: str, collection: str, top_k: int = 5):
    """Hybrid retrieval: ColPali visual + text semantic search."""

    # Visual retrieval via ColPali
    query_inputs = processor(text=[query])
    query_embeddings = colpali_model(**query_inputs)

    # query_points (Query API) replaces the legacy search endpoints,
    # which Qdrant 1.19 removed from its OpenAPI schema
    visual_results = qdrant.query_points(
        collection,
        query=query_embeddings[0].tolist(),  # multi-vector query, MaxSim scoring
        using="colpali",
        query_filter=Filter(must=[
            FieldCondition(key="type", match=MatchValue(value="page_image"))
        ]),
        limit=top_k,
    ).points

    # Text retrieval via dense embeddings
    text_embedding = text_encoder.encode(query)
    text_results = qdrant.query_points(
        collection,
        query=text_embedding.tolist(),
        using="text",
        limit=top_k,
    ).points

    # Fuse results using reciprocal rank fusion
    fused = reciprocal_rank_fusion(visual_results, text_results, k=60)
    return fused[:top_k]


# --- GENERATION ---

def generate_answer(query: str, retrieved_context: list) -> str:
    """Generate answer using VLM with multi-modal context."""

    content_blocks = [{"type": "text", "text": f"Question: {query}\n\nContext:"}]

    for ctx in retrieved_context:
        if ctx.payload["type"] == "page_image":
            # Include the actual page image
            content_blocks.append({
                "type": "image",
                "source": load_page_image(ctx.payload["source"], ctx.payload["page"])
            })
        else:
            # Include text/table content
            content_blocks.append({
                "type": "text",
                "text": f"[{ctx.payload['type']}] {ctx.payload['content']}"
            })

    content_blocks.append({
        "type": "text",
        "text": "Answer the question using ONLY the provided context. Cite sources."
    })

    response = vlm_client.messages.create(
        model="claude-sonnet-5-5",
        max_tokens=16000,
        output_config={"effort": "medium"},
        messages=[{"role": "user", "content": content_blocks}]
    )
    return next(b.text for b in response.content if b.type == "text")
```

---

## System Design Interview Angle

### Q: Design a RAG system for a financial research platform that needs to answer questions about earnings reports containing text, tables, and charts.

**Strong answer:**

The core challenge is that 60%+ of the information in earnings reports lives in tables and charts, not prose. A text-only RAG pipeline would miss revenue breakdowns, trend lines, and comparative data.

**Architecture**: I would use a hybrid approach (Pattern 2 + elements of Pattern 3):

1. **Ingestion**: Render each PDF page at 300 DPI. Run a VLM extraction pass to convert tables to markdown and charts to structured descriptions. Simultaneously generate ColPali multi-vector embeddings for each page image.

2. **Storage**: Three indices: (a) text chunks with dense embeddings (financial text), (b) table markdown with dense embeddings plus metadata filters for table type, (c) ColPali multi-vector index for page-level visual retrieval.

3. **Retrieval**: Query analyzer classifies the query type. "What was Q3 revenue?" triggers text + table search. "Show me the revenue trend" triggers visual (ColPali) search. Results are fused via RRF and reranked by a multimodal reranker (Qwen3-VL-Reranker-2B/8B is the open-weight option: Apache 2.0, 32K context) so page images and table text compete on one scale.

4. **Generation**: A VLM (Claude or Gemini) receives the fused context (text chunks, table markdown, and relevant page images). It generates a grounded answer with citations to specific pages and tables.

**Key trade-offs**: ColPali gives excellent recall on visual content but stores ~1024 vectors per page, so for 100k documents (500k pages), that is ~500M vectors. I would use binary quantization to reduce storage by 32x, accepting a small recall hit, or MUVERA-style candidate generation if the vector DB supports it. Before committing to multi-vector at all, I would A/B a single-vector multimodal embedder (Cohere Embed 5 or Gemini Embedding 2) plus the reranker on my eval set: 500k vectors instead of 500M. For the text path, BM25 + dense hybrid search handles financial terminology well.

**Ingestion cost** (October 2026 list prices, ~1.7K tokens in and ~800 out per page): about $2,000 for 500k pages on Gemini 3.8 Flash at its intro price, about $5,700 on Claude Sonnet 5.5 or GPT-6 Sol before reasoning tokens, or $750 through Cohere Parse at $1.50 per 1,000 pages. Batch APIs halve the VLM numbers. I would route pages: a specialist parser for text-and-table pages, a frontier VLM only for chart-heavy pages where descriptions matter.

### Q: How would you handle a query that requires information from BOTH a chart and a table on different pages?

**Strong answer:**

This is the cross-modal, cross-page retrieval problem. The solution has three parts:

1. **Retrieval diversity**: Ensure the retriever returns results from multiple modalities. Set minimum quotas: at least 2 text results, 2 table results, and 1 visual result in every retrieval set, regardless of which modality scores highest.

2. **Context assembly**: When assembling the VLM prompt, include all retrieved content with explicit provenance: "[Table from page 14: Q3 Revenue by Region]" and "[Chart from page 22: Revenue Trend 2024-2026]". The VLM can then reason across both.

3. **Agentic fallback**: If the initial retrieval does not surface enough cross-modal context, an agentic layer can issue follow-up retrievals: "The table shows revenue numbers but the user asked about trends, so let me also search for charts related to revenue."

The key insight is that cross-modal questions are inherently multi-hop. The system needs to retrieve from one modality, recognize the gap, and retrieve from another.

### Q: Users want to ask questions across a 10,000-hour library of training videos. How do you design and cost it?

**Strong answer:**

Separate library search from single-video understanding, because only the second one can use the model's own video tools.

1. **Index once.** Processing every video per question is out: at roughly 100 tokens per second, the library is about 3.6B tokens per full pass. Instead, transcribe once (about $2,700 for 600,000 minutes at `gpt-transcribe`'s $0.0045 per minute), put transcripts and scene captions into a hybrid BM25 + dense index, and embed short clips (up to 120 seconds per input) with Gemini Embedding 2 for visual queries.
2. **Retrieve, then watch.** Retrieval returns the top videos with timestamps. Only those go to Gemini 3.8 Flash. A full hour at static 1 FPS is about 360K tokens (~$0.27 of input at the intro price); with `"processing": "agentic"` the model decides which segments to inspect, and Google reports up to 88% fewer tokens on long-form video (vendor-reported).
3. **Cost by query class.** Agentic token use depends on the question, so I would meter it per query type during a pilot and set quotas from the measured distribution, not from duration.
4. **Evaluate grounding.** Require timestamped citations and spot-check them, because the failure mode is a fluent answer from the wrong segment.

---

## References

- Faysse et al. "ColPali: Efficient Document Retrieval with Vision Language Models" (ICLR 2025)
- Google. "SigLIP 2: Multilingual Vision-Language Encoders" (2025)
- NVIDIA. "An Easy Introduction to Multimodal Retrieval-Augmented Generation" (2025)
- HKUDS. "RAG-Anything: All-in-One Multimodal RAG Framework" (2025)
- Vespa Blog. "PDF Retrieval with Vision Language Models" (2024)
- Google. "Gemini Embedding 2" (March 2026) and "Introducing agentic video in Gemini" (September 2026)
- Google. Gemini API video understanding documentation (accessed October 2026)
- Cohere. "Embed 5" (September 2026)

---

*Previous: [Late Interaction & ColBERT](11-late-interaction-colbert.md) | Next: [RAG Evaluation Patterns](13-rag-evaluation-patterns.md)*
