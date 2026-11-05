# Data Engineering for AI

Models are only as good as the data fed to them, and most of the work in any serious AI system is upstream of the model. This chapter covers the **data layer** that feeds both RAG and fine-tuning: ingestion, cleaning, deduplication, PII and governance, quality filtering, and the pipelines that move it all. It sits in the retrieval section because RAG is where most teams first hit it, but it is **cross-cutting**: the same pipeline that prepares documents for a vector store also prepares examples for fine-tuning. Treating data engineering as "step two of RAG" understates it; it is a standalone discipline both consumers depend on.

## Table of Contents

- [The Shared Pipeline](#the-shared-pipeline)
- [Ingestion](#ingestion)
- [Cleaning and Normalization](#cleaning-and-normalization)
- [Deduplication](#deduplication)
- [PII, Consent, and Governance](#pii-consent-and-governance)
- [Quality Filtering and Enrichment](#quality-filtering-and-enrichment)
- [Pipelines and Orchestration](#pipelines-and-orchestration)
- [Data for Fine-Tuning](#data-for-fine-tuning)
- [Failure Modes](#failure-modes)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Shared Pipeline

The central idea: one pipeline, two consumers. RAG needs documents parsed, cleaned, deduped, chunked, enriched, and embedded. Fine-tuning needs examples parsed, cleaned, deduped, quality-filtered, decontaminated against eval sets, balanced, and formatted. The first five stages are **shared**; the two diverge only at the tail.

```
SOURCES        web · docs (PDF/office/email) · DBs/APIs · uploads
   |
[1] DETECT & ROUTE    MIME sniff -> language ID -> route by type
[2] PARSE / EXTRACT   typed elements + structure
[3] CLEAN / NORMALIZE boilerplate strip · encoding repair · unicode · language filter
[4] DEDUP             exact -> MinHash/LSH -> semantic (SemDeDup)
[5] GOVERN            PII detect/redact · license/consent/provenance metadata
[6] QUALITY FILTER    heuristics + model classifier  [always ablate on a downstream eval]
   |-- RAG branch ---------------------|   |-- FINE-TUNE branch ----------------|
   | [7a] chunk -> enrich -> embed     |   | [7b] curate/label -> synthesize    |
   |      -> VECTOR STORE (CDC-fresh)  |   |      -> decontaminate -> balance   |
   |-----------------------------------|   |      -> TRAINING RECORDS           |
CROSS-CUTTING   orchestration (Airflow/Dagster) · lineage (OpenLineage) · versioning (DVC/lakeFS)
```

The "most of the work" framing is durable: the widely cited heuristic that practitioners spend ~80% of their time on data preparation is folklore-grade (repeated across many sources rather than measured), but the direction is right, and vendor claims that AI agents now cut prep time 50-70% are unverified and self-interested. Treat both as directional.

---

## Ingestion

The input is a heterogeneous pile (PDFs, scans, office docs, HTML, email, images). You must detect what each file is, route it to the right parser, and emit a normalized structured representation.

- **File-type detection by content, not extension.** The standard is magic-number sniffing via `libmagic` (the library behind the Unix `file` command), exposed through bindings like `python-magic`. Parsers such as Unstructured auto-detect type this way and fall back to the extension.
- **Language detection** for routing and filtering, commonly with fastText's language classifier.
- **Parsing and routing.** Content type determines the processor: text-extractable PDFs take a fast text path, scanned or image PDFs take OCR, complex multi-column or table-heavy documents take a layout model or a multimodal (per-page-screenshot) path, office docs take format-specific handlers, and email is split into headers and body. The leading toolkits are **Unstructured** (64+ file types, emits typed elements like Title/NarrativeText/Table), **Docling** (IBM, MIT-licensed, layout plus table models, multiple export formats, and pluggable VLM backends such as NVIDIA Nemotron Parse 2.0 and MinerU 2.5 Pro as of September 2026, plus Nemotron 3.5 Parse from v2.141.0 on October 20), **MinerU**, and **LlamaParse** (tiered API modes up to a multimodal agentic parse). A teaching point: published parser benchmarks disagree, so **benchmark parser choice on your own documents** rather than trusting a leaderboard.

### Route Pages to the Cheapest Tier That Works

At corpus scale, parsing cost is set by **pages per GPU-second**, not by the best leaderboard score. The 2026 pattern is tiered: try the cheap path, escalate only the pages that fail a quality check.

| Tier | Example | Cost and throughput | Use for |
|---|---|---|---|
| No model | MinerU 4.0 `flash` tier, native PDF text extraction | CPU only | Born-digital PDFs and office files |
| Small specialist models | MinerU `basic` (OCR, formula and table models, CPU-capable); NVIDIA Nemotron 3.5 Parse (October 17, 2026; about 0.63B parameters, OpenMDW-1.1, optional multi-pass mode; it replaced Nemotron Parse 2.0) | Cheap, self-hosted | Scans with simple layout |
| Parsing VLM | Cohere Parse `parse-v5.0` (2.3B, August 27, 2026): $1.50 per 1,000 pages via API, 4.5 pages/s on one GPU (vendor); jina-ocr-v1 (September 14, 2026): 2.57 pages/s on one A100, CC BY-NC 4.0 so non-commercial without a license | Mid | Tables, forms, multi-column |
| Frontier VLM | GPT-6.1 Sol, Claude Opus 5.5, Gemini 4 Argon | Highest per page | The hard residue |

The accuracy gap between the tiers is smaller than the price gap: on Cohere's own ParseBench run, Cohere Parse averaged 79.2 against 84.4 for GPT-5.5 and 84.3 for Claude Opus 4.8, neither of them the vendor's newest model even at publication (vendor-run; rerun it against current models before you rely on it). MinerU 4.0 (September 16, 2026) builds the tiering in (`flash`, `basic`, `standard`, `advanced`), parses DOCX, PPTX, XLSX, EPUB and HTML natively without converting to PDF, and returns stable block locators (`doc/tier/page/block`) that make citations verifiable and let agents read documents progressively instead of receiving pre-cut chunks.

---

## Cleaning and Normalization

The shared first-pass quality gate:
- **Boilerplate removal.** Strip navigation, ads, headers, footers, cookie banners. For HTML at scale, the standard is a content-extraction library; the FineWeb project found that extracting from raw web archives with such a tool beat using pre-extracted text, which "retained too much boilerplate," so this is upstream quality, not cosmetics.
- **Encoding fixes** for mojibake and double-encoded text, plus line-ending normalization.
- **Unicode normalization** (NFC/NFKC) so visually identical strings compare equal, a prerequisite for dedup and matching to work at all.
- **Language filtering** below a confidence threshold.

A counterintuitive caveat worth teaching: **more filtering is not strictly better.** FineWeb's ablations found many heuristics had low or marginal impact (it dropped most candidate filters for too little gain), and Nemotron-CC (arXiv:2412.02595) reports that aggressive heuristic filtering can discard roughly 18% of the tokens its own quality classifier rates high-quality. Filters must be ablated against a downstream eval, not assumed.

---

## Deduplication

Dedup is the highest-leverage stage, and the clearest case for "shared infrastructure," because it pays off three ways. The foundational result (Lee et al., arXiv:2107.06499) reports that deduplicating training data makes models emit memorized text about 10x less often, reach equal or better accuracy in fewer steps, and, critically, **reduces train-test overlap, so dedup is also decontamination.** It also mitigates privacy risk by reducing memorization of repeated PII.

The three tiers, run as a cascade (cheap-and-exact first, expensive-and-semantic last):
1. **Exact**: hash whole documents or normalized substrings. Fast and precise; catches only verbatim copies.
2. **Fuzzy / near-duplicate**: **MinHash + LSH** estimates Jaccard similarity over token n-gram shingles and buckets candidates to avoid all-pairs comparison (the web-scale default; some vector DBs now ship it as a native index). SimHash is the classic alternative. The caveat: these are *lexical*, so documents that share a template but differ in meaning can be wrongly removed.
3. **Semantic / embedding dedup**: **SemDeDup** (arXiv:2303.09540) embeds each item, clusters, and drops near-duplicates within a cluster by cosine similarity; it reports removing ~50% of a large image-text dataset with minimal performance loss, catching paraphrases that MinHash misses.

The production pattern composes them, MinHash first, then SemDeDup. State the three-way payoff explicitly: RAG (duplicate chunks waste the context window and crowd out diverse evidence), training (less memorization, fewer steps), and eval integrity (dedup against benchmarks prevents contamination).

---

## PII, Consent, and Governance

**PII detection and redaction.** Microsoft Presidio (MIT) is the open standard: an Analyzer that detects entities via NER plus regex, checksums, and context words, and an Anonymizer that redacts, replaces, masks, hashes, or encrypts, across text, images (with OCR), and structured data, deployable at corpus scale. Because dedup reduces duplication-driven memorization, privacy and dedup are linked.

**Consent, licensing, and provenance** are now a regulatory requirement, not just hygiene. Under the EU AI Act, general-purpose-model providers must keep a copyright policy and publish a "sufficiently detailed summary" of training content using the AI Office's mandatory template, including data sources and respect for copyright opt-outs (this summary duty has applied to new general-purpose models since August 2025, with models already on the market given until August 2027). Article 50 transparency obligations have applied since August 2, 2026; generative systems placed on the market before that date have until December 2, 2026 to add machine-readable marking. The governance implication for the pipeline: every record carries source, license, consent status, and timestamp as metadata from ingestion onward, which is exactly what lineage (below) provides. Governance is not a final gate; it is metadata threaded through every stage. See [AI Governance and Compliance](../13-reliability-and-safety/04-ai-governance-and-compliance.md).

**Record how data was acquired, not just its license.** US courts are separating the legality of *training* from the legality of *acquisition*. In Bartz v. Anthropic (2025) the court held training on books to be fair use but did not extend that to building a library from pirated copies, and the case settled for $1.5B. In August 2026 music publishers including Sony Music Publishing and Warner Chappell sued Anthropic on the same acquisition theory, alleging torrenting of books containing lyrics and sheet music, while the US government filed a brief (reported September 2, 2026) backing OpenAI's fair-use position on training in The New York Times v. OpenAI. For the pipeline, that means a provenance field for **acquisition channel** (licensed feed, crawl under robots rules, purchased, user-contributed, torrent or shadow library) on every source, so a tainted source can be found and excised. The remediation path is expensive: Suno's v6 (September 2026) was retrained on licensed music, and the company said it would retire its older models.

**Fully open releases make provenance auditable.** Most open-weight models ship weights only. MBZUAI's K2 Horizon family (September 3, 2026, Apache 2.0, 0.9B to 375B-A23B) also releases its training data (TxT360-v2 plus pretraining and midtraining sets), training code and intermediate checkpoints, so you can run contamination checks against the actual pretraining corpus instead of trusting a model card.

---

## Quality Filtering and Enrichment

**Quality filtering** comes in two families. **Heuristic** rules (length, symbol-to-word ratio, repetition, stopword presence) are cheap but cannot catch complex content noise. **Model-based classifiers** score quality or educational value; FineWeb-Edu trained a lightweight classifier on LLM-generated quality annotations and reported large downstream gains, matching a larger corpus with far fewer tokens. The flagged caveat: classifier filtering is not a free lunch (the "data-quality illusion" work argues it can be miscalibrated), so **always ablate filters on a downstream eval, never trust them by reputation.**

**Chunking** (the RAG-side stage) has no universal winner: published benchmarks swing widely between fixed-size, recursive, and semantic strategies, so it is dataset-specific and must be evaluated, not defaulted. See [Chunking Strategies](02-chunking-strategies.md).

**Enrichment and metadata.** Attach standard metadata (title, author, timestamp, source, section) and generated metadata (chunk summaries, contextual prefixes, synthetic questions a chunk answers), which turns single-axis vector similarity into multi-dimensional filtered search. See [Contextual Retrieval](10-contextual-retrieval.md).

**Lineage.** OpenLineage (with Marquez as the reference implementation) is the vendor-neutral standard for tracking run, job, and dataset events; for ML it extends the graph forward through feature tables, models, and predictions. With data versioning (DVC, lakeFS, Delta/Iceberg time-travel), this is the substrate that makes the governance metadata auditable end to end.

---

## Pipelines and Orchestration

**Batch vs streaming.** Pretraining-corpus prep and bulk RAG indexing are batch jobs (Spark or Ray over object storage). RAG *freshness* is incremental: as source documents change, re-ingest only the deltas. Change Data Capture captures row-level source changes in real time, which for RAG maps to "detect changed, new, and deleted documents, re-parse and re-embed only those, then upsert or delete in the vector store," avoiding a full re-index and preventing stale or orphaned vectors.

**Orchestrators.** Airflow is the default with the biggest ecosystem; Dagster is asset-based with incremental recompute that suits incremental RAG re-embedding; Prefect is the lighter-weight Pythonic option. Spark and Ray are the distributed compute the orchestrator schedules, not orchestrators themselves.

**The embed stage is moving into the store.** MongoDB Atlas Automated Embedding (GA August 13, 2026) embeds new documents on write and re-embeds changed ones with Voyage models; turbopuffer made native embedding GA on September 29, 2026; Elasticsearch `semantic_text` embeds at ingest. That deletes the embed-and-upsert job from your DAG and makes CDC freshness the database's problem. The tradeoff is that the vendor now owns the model version, so a model change becomes a vendor-scheduled re-embed rather than one you plan. On the lake side, Milvus 3.0 (July 2026) can query Parquet, Iceberg, Lance and Vortex data in place as read-only External Collections and backfill new columns online, so re-embedding a few hundred million rows with a new model becomes a background column fill rather than a parallel index rebuild.

**Vector vs feature store.** The pipeline forks at the end: RAG writes chunk embeddings plus metadata to a **vector store** (the serving layer for retrieval), while fine-tuning writes curated examples to a **feature or training-record store** with point-in-time correctness. Both hang off the same shared trunk.

---

## Data for Fine-Tuning

The tail that diverges from RAG:
- **Curation beats volume.** A small set of carefully curated examples often beats a large mediocre one; the quality triad is difficulty, quality, and diversity. (The exact "1,000 beats 10,000" figures are illustrative, not laws.)
- **Synthetic data** (Self-Instruct-style expansion from a small seed set through a teacher model, then filtered) scales cheaply but risks diversity collapse and model degradation if generated carelessly, so diversity is a first-class objective. See [Synthetic Data Generation](../03-training-and-adaptation/06-synthetic-data-generation.md).
- **Decontamination is the must-do.** The flagged result (arXiv:2311.04850) reports that *rephrased or translated* test items slip past n-gram decontamination, and a model trained on such data can overfit a benchmark to near-frontier scores. Even synthetic data generated by frontier models was found contaminated. The rule: decontaminate with embedding or LLM-based matching, not just n-gram overlap, and do it before every eval, because contamination silently inflates scores. See [Benchmarks and Leaderboards](../14-evaluation-and-observability/03-benchmarks-and-leaderboards.md).

---

## Failure Modes

1. **Trusting file extensions** instead of magic-byte detection (wrong parser, silent garbage).
2. **Parsing scanned PDFs with a text-only path** (empty or partial extraction; route image PDFs to OCR or multimodal).
3. **Sending every page to the most expensive parser** (route by tier and escalate only failures; per-page cost dominates at corpus scale).
4. **Extracting from pre-stripped HTML** (boilerplate pollution; extract from raw with a content-extraction tool).
5. **Skipping Unicode normalization** (exact dedup and matching silently miss duplicates).
6. **Dedup at the wrong tier** (MinHash alone misses paraphrases; cascade exact, then fuzzy, then semantic).
7. **Over-aggressive heuristic filtering** (Nemotron-CC reports it can discard ~18% of high-quality tokens; ablate every filter).
8. **Trusting a quality classifier on reputation** (the data-quality illusion; validate downstream).
9. **No PII redaction before training** (memorized and regurgitated PII, worse with duplication).
10. **No license, consent, provenance, or acquisition-channel metadata** (an un-auditable corpus that cannot meet training-content-summary obligations or isolate a tainted source).
11. **Eval contamination, the silent killer** (n-gram-only decontamination misses rephrased test items; use embedding or LLM decontamination).
12. **Stale RAG index** (full re-index instead of CDC; cost blowup, stale answers, orphaned vectors).
13. **Bad chunking by default** (benchmarks disagree; evaluate per dataset).
14. **No lineage** (irreproducible datasets and undebuggable regressions; emit lineage from day one).

---

## Interview Questions

### Q: Why is deduplication one of the most important stages in an AI data pipeline?

**Strong answer:**
Because it pays off three different ways from one operation. For training, deduplicating cuts memorization (models regurgitate training text far less) and reaches equal or better accuracy in fewer steps, so it saves compute. For RAG, duplicate chunks waste the context window and crowd out diverse evidence, hurting retrieval quality. And crucially, dedup against your benchmarks is also decontamination: the foundational study found removing duplicates also removed train-test overlap that was inflating eval scores. In practice I run it as a cascade, exact hashing first, then MinHash with LSH for near-duplicates, then semantic embedding dedup for paraphrases that lexical methods miss, because each tier catches what the cheaper one cannot.

### Q: How do you keep eval results honest against data contamination?

**Strong answer:**
The trap is that simple n-gram decontamination is not enough. A well-known result showed that paraphrased or translated versions of test items slip right past n-gram matching, and a model trained on those can overfit a benchmark to near-frontier scores, and even synthetic data generated by strong models came back contaminated. So I decontaminate with semantic methods, embedding similarity search plus an LLM adjudicator, not just string overlap, and I run it against all of my eval sets before every evaluation. More broadly I prefer held-out or freshly released test sets where possible, treat any benchmark as potentially contaminated, and lean on my own gold data for the decisions that matter, since that is the one set I can guarantee the model has not seen.

### Q: You need to parse 5 million pages of mixed born-digital PDFs, scans and office files for RAG. How do you design it for cost?

**Strong answer:**
I would route by tier and treat pages per GPU-second as the core metric. First, detect type by content and send born-digital PDFs and office files down a no-model path (native text extraction; MinerU's `flash` tier does this), which is often most of an enterprise corpus. Scans go to a small self-hosted OCR and layout model. Only pages that fail a cheap quality check (empty text, broken tables, low OCR confidence, garbled reading order) escalate to a parsing VLM, and only the residue that still fails goes to a frontier VLM.

The arithmetic, assuming roughly 1,500 input and 800 output tokens per page and no reasoning tokens: Cohere Parse at $1.50 per 1,000 pages puts all 5M pages at $7,500. GPT-6 Sol ($2 / $10 per 1M) is about $0.011 per page, or ~$55,000, and Claude Opus 5.5 ($4 / $20) at least double that, since its thinking cannot be turned off, while specialist parsers trailed older frontier models (GPT-5.5, Opus 4.8) by only about 5 points on vendor-run ParseBench. The cheap end of the frontier families breaks the simple ladder: GPT-6 Luna and Claude Haiku 5.5 (both $0.10 / $0.50, Haiku only while a request stays under 100K tokens, which a single page always does) are about $0.0006 per page, ~$2,750 for the corpus, below the parsing API (Haiku's newer tokenizer counts about 30% more tokens for the same content, so expect it to land somewhat higher). GPT-6.1 Terra ($1 / $5) is a middle rung at about $0.0055 per page (~$27,500). All three belong in the benchmark next to the specialists; the open question is whether they hold up on tables and scans.

Dedicated GPUs change it again: at Cohere's reported 4.5 pages/s per GPU, 5M pages is roughly 300 GPU-hours, which is why Cohere says its single-tenant Model Vault deployment undercuts the API by 23% at 50% utilization (vendor-reported). I would check licenses before self-hosting (jina-ocr-v1 is CC BY-NC), keep block-level locators so citations point at a page and block, and benchmark each tier on a few hundred of my own pages before fixing the routing thresholds.

---

## References

- Lee et al., "Deduplicating Training Data Makes Language Models Better" arXiv:2107.06499
- Abbas et al., "SemDeDup" arXiv:2303.09540
- Penedo et al., "The FineWeb Datasets" arXiv:2406.17557
- Yang et al., "Rethinking Benchmark and Contamination ... with Rephrased Samples" arXiv:2311.04850
- Microsoft, [Presidio](https://github.com/microsoft/presidio)
- [Unstructured](https://www.unstructured.io/), [Docling](https://github.com/docling-project/docling), [OpenLineage](https://openlineage.io/)
- [MinerU 4.0 release notes (Sep 2026)](https://github.com/opendatalab/MinerU/releases/tag/mineru-4.0.0-released)
- [Cohere. "Parse" (Aug 2026)](https://cohere.com/blog/parse)
- [MBZUAI K2 Horizon 375B-A23B model card](https://huggingface.co/IFM/K2-Horizon-375B-A23B)

---

*Previous: [Production RAG at Scale](14-production-rag-at-scale.md) | Next: [Agent Fundamentals](../07-agentic-systems/01-agent-fundamentals.md)*
