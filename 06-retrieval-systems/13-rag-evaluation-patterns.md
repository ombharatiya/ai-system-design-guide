# RAG Evaluation Patterns

Evaluation is the hardest unsolved problem in RAG. You can build a retrieval pipeline in a day; knowing whether it actually works takes weeks. The industry has converged on a layered evaluation strategy: the RAG Triad for correctness, component-level metrics for debugging, and automated regression testing for production safety. Langfuse, LangWatch, Braintrust, and Arize Phoenix all ship native RAG eval recipes; pick by deployment model (self-hosted vs SaaS) and whether you need eval-gated CI/CD blocking.

## Table of Contents

- [The RAG Triad](#the-rag-triad)
- [RAGAS Framework and Metrics](#ragas-framework-and-metrics)
- [Component-Level Evaluation](#component-level-evaluation)
- [LLM-as-Judge for RAG](#llm-as-judge-for-rag)
- [Building Golden Test Sets](#building-golden-test-sets)
- [Automated Regression Testing](#automated-regression-testing)
- [Production Monitoring](#production-monitoring)
- [Cost of Evaluation at Scale](#cost-of-evaluation-at-scale)
- [Tools Comparison](#tools-comparison)
- [System Design Interview Angle](#system-design-interview-angle)
- [References](#references)

---

## The RAG Triad

The RAG Triad is the foundational framework for evaluating RAG systems. It decomposes correctness into three independent dimensions, each catching a different failure mode.

```
                          User Query
                              |
                              v
                    +-------------------+
                    |    RETRIEVER      |
                    +-------------------+
                              |
                   (1) Context Relevance
                    "Did we retrieve the
                     right documents?"
                              |
                              v
                    +-------------------+
                    |    GENERATOR      |
                    +-------------------+
                         /         \
            (2) Groundedness      (3) Answer Relevance
            "Is the answer         "Does the answer
             supported by           address the actual
             the context?"          question?"
                  |                       |
                  v                       v
             No hallucination       No tangential answers
```

### Dimension 1: Context Relevance

**Question**: Is each retrieved chunk actually relevant to the user query?

**What it catches**: Bad retrieval. The vector search returned documents about the wrong topic, or the query was ambiguous and the retriever guessed wrong.

**How to measure**:
- For each retrieved chunk, ask: "Is this chunk relevant to answering the query?"
- Score: (number of relevant chunks) / (total retrieved chunks)
- A score of 0.3 means 70% of retrieved context is noise, forcing the LLM to find a needle in irrelevant hay.

**Why it matters**: Low context relevance is the root cause of most RAG failures. Even a perfect generator cannot produce a good answer from irrelevant context.

### Dimension 2: Groundedness (Faithfulness)

**Question**: Is every claim in the generated answer supported by the retrieved context?

**What it catches**: Hallucination. The LLM generated claims that are plausible but not present in the retrieved documents.

**How to measure**:
- Decompose the answer into individual claims/statements.
- For each claim, search the retrieved context for supporting evidence.
- Score: (number of supported claims) / (total claims)
- A score of 0.7 means 30% of the answer is hallucinated.

**Why it matters**: This is the metric that enterprise customers care about most. An unfaithful RAG system is worse than no RAG at all because it produces confident-sounding wrong answers with fake citations.

### Dimension 3: Answer Relevance

**Question**: Does the final answer actually address what the user asked?

**What it catches**: Tangential answers. The retrieval was good, the answer is grounded, but it does not answer the question. Common when the retriever finds related-but-not-matching content.

**How to measure**:
- Generate N hypothetical questions that the answer would be a good response to.
- Measure semantic similarity between these hypothetical questions and the original query.
- High similarity means the answer is on-topic.

**Why it matters**: A system can retrieve relevant context and faithfully summarize it, yet still miss the point of the question. Answer relevance catches this.

### Triad Failure Modes

| Failure Pattern | Context Relevance | Groundedness | Answer Relevance | Root Cause |
|----------------|-------------------|-------------|-----------------|------------|
| Good RAG | High | High | High | System working correctly |
| Bad Retrieval | **Low** | High | Low | Embeddings or search misconfigured |
| Hallucination | High | **Low** | High | LLM ignoring context, prompt issue |
| Tangential Answer | High | High | **Low** | Query ambiguity, wrong index |
| Total Failure | **Low** | **Low** | **Low** | Fundamental pipeline issue |

---

## RAGAS Framework and Metrics

RAGAS (Retrieval Augmented Generation Assessment) is the most widely adopted open-source evaluation framework for RAG, providing reference-free metrics that do not require ground-truth answers.

### Core RAGAS Metrics

```
  RAGAS Metric Suite (v0.2+)
  |
  +-- Retrieval Metrics
  |     +-- Context Precision: Are relevant docs ranked higher?
  |     +-- Context Recall: Did we find all relevant docs?
  |     +-- Context Entities Recall: Did we capture key entities?
  |     +-- Context Relevance: Is retrieved context pertinent?
  |
  +-- Generation Metrics
  |     +-- Faithfulness: Are claims supported by context?
  |     +-- Answer Relevance: Does the answer address the query?
  |     +-- Answer Correctness: Does the answer match ground truth?
  |     +-- Answer Similarity: Semantic overlap with reference answer
  |
  +-- Noise & Robustness
  |     +-- Noise Sensitivity: How much does irrelevant context hurt?
  |
  +-- Multi-Modal (2025+)
        +-- Multimodal Faithfulness: Claims supported by images + text?
        +-- Multimodal Relevance: Are retrieved images relevant?
```

### How RAGAS Faithfulness Works (Under the Hood)

```
Step 1: Claim Extraction
  Answer: "Revenue grew 15% in Q3, driven by APAC expansion
           and the new enterprise tier launched in July."

  Claims:
    c1: "Revenue grew 15% in Q3"
    c2: "Growth was driven by APAC expansion"
    c3: "Growth was driven by the new enterprise tier"
    c4: "The enterprise tier was launched in July"

Step 2: Evidence Matching (per claim)
  c1: Found in Context chunk 3 --> SUPPORTED
  c2: Found in Context chunk 1 --> SUPPORTED
  c3: Not found in any context --> UNSUPPORTED
  c4: Context says "August" not "July" --> CONTRADICTED

Step 3: Score Calculation
  Faithfulness = supported / total = 2/4 = 0.50
```

### How RAGAS Context Precision Works

```
  Retrieved chunks ranked by retriever score:
    Rank 1: Chunk about Q3 revenue    --> Relevant (v_1 = 1)
    Rank 2: Chunk about company history --> Not relevant (v_2 = 0)
    Rank 3: Chunk about Q3 expenses   --> Relevant (v_3 = 1)
    Rank 4: Chunk about office locations --> Not relevant (v_4 = 0)

  Context Precision@K:
    Precision@1 = 1/1 = 1.0
    Precision@2 = 1/2 = 0.5
    Precision@3 = 2/3 = 0.67
    Precision@4 = 2/4 = 0.5

  Average Precision = (1.0*1 + 0.5*0 + 0.67*1 + 0.5*0) / 2
                    = (1.0 + 0.67) / 2 = 0.835
```

### RAGAS vs. Ground-Truth Metrics

| Metric | Needs Ground Truth? | What It Measures |
|--------|-------------------|------------------|
| Faithfulness | No | Claims supported by context |
| Context Relevance | No | Retrieved chunks relevance |
| Answer Relevance | No | Answer addresses query |
| Context Recall | **Yes** | Coverage of reference answer |
| Answer Correctness | **Yes** | Match against reference answer |
| Answer Similarity | **Yes** | Semantic overlap with reference |

**Insight**: Start with reference-free metrics (faithfulness, context relevance, answer relevance) for rapid iteration. Add ground-truth metrics once you have a golden test set for regression testing.

---

## Component-Level Evaluation

The RAG Triad evaluates the system end-to-end. Component-level evaluation isolates each stage to pinpoint failures.

### Retriever Evaluation

```
  Query Set (100+ queries with known relevant documents)
        |
        v
  Run Retriever --> Retrieved docs per query
        |
        v
  Compare against ground truth relevance labels
        |
        v
  Metrics:
    +-- Recall@K: What fraction of relevant docs are in the top K?
    +-- MRR (Mean Reciprocal Rank): How high is the first relevant doc?
    +-- NDCG@K: Quality-weighted ranking metric
    +-- Precision@K: What fraction of top K are relevant?
```

**Key Retriever Benchmarks**:

| Metric | Minimum Threshold | Good | Excellent |
|--------|------------------|------|-----------|
| Recall@10 | 0.70 | 0.85 | 0.95+ |
| MRR | 0.50 | 0.70 | 0.85+ |
| NDCG@10 | 0.50 | 0.70 | 0.85+ |
| Precision@5 | 0.40 | 0.60 | 0.80+ |

### Separate the Embedder from the Index

A retrieval miss has two possible owners, and they need different tests:

| Question | Metric | Ground truth | Fix if it fails |
|----------|--------|--------------|-----------------|
| Did the **embedder** rank the right chunk highly? | Recall@K with **exact** (brute-force) kNN | Human relevance labels | Model, chunking, contextualization |
| Did the **ANN index** return what exact kNN would? | ANN recall@K vs exact kNN | Exact neighbors, no labels needed | Index params (`ef`, probes), quantization, rescoring |

Run both on the same queries. If exact-kNN recall is fine and ANN recall is 0.85, no embedding upgrade will help. Quantization makes this test mandatory: the 1-to-4-bit rotation-based schemes that vector DBs added in 2026 (Qdrant TurboQuant and Turbo4, Weaviate 4-bit RQ preview) trade recall for memory, and are opt-in or preview rather than defaults.

For index behavior at a scale you cannot label, use an open benchmark with exact ground truth. Qdrant-FineWeb-10B (September 1, 2026) ships 10.07B dense plus 10.07B sparse vectors and 100,000 queries with exact top-1,000 neighbors; its companion PubMed-Multi-Vector set compares dense, sparse and ColBERT-style retrieval on the same corpus. Vendor latency claims measured at 10M vectors with shallow top-k and no filters tell you little about a filtered top-100 query at a billion.

**Public leaderboards are priors, not results.** MTEB scores can be inflated when models train on data that overlaps the public test sets. RTEB (from the MTEB team, October 2025) mixes open datasets with private held-out sets that the maintainers evaluate, which makes it the better shortlist filter; NVIDIA reports Nemotron 3 Embed 8B (open weights, OpenMDW-1.1) at #1 on RTEB Multilingual with 78.5 (vendor-reported, July 2026). Shortlist from RTEB, then decide on your own golden set.

### Generator Evaluation

Isolate the generator by fixing the retrieval context and varying only the generation.

```
  Fixed Context (known relevant chunks)
  + Query
        |
        v
  Run Generator --> Answer
        |
        v
  Metrics:
    +-- Faithfulness (RAGAS): Does it stay grounded?
    +-- Completeness: Does it cover all relevant info in context?
    +-- Conciseness: Is it appropriately brief?
    +-- Format Compliance: Does it follow the expected output format?
    +-- Citation Accuracy: Do citations point to the right chunks?
```

### Reranker Evaluation

```
  Query + Initial retrieval results (e.g., top 100 from BM25)
        |
        v
  Run Reranker --> Reranked results
        |
        v
  Metrics:
    +-- NDCG improvement: Did reranking move relevant docs up?
    +-- Recall preservation: Did reranking lose any relevant docs?
    +-- Latency: What did reranking add to query time?
```

---

## LLM-as-Judge for RAG

Using an LLM to evaluate another LLM's output is the dominant evaluation paradigm. It scales where human evaluation cannot, but has known biases.

### How It Works

```
  Evaluation Prompt Template:
  +------------------------------------------------------------------+
  | You are evaluating a RAG system. Given:                           |
  | - User Query: {query}                                             |
  | - Retrieved Context: {context}                                    |
  | - Generated Answer: {answer}                                      |
  |                                                                    |
  | Rate the following on a scale of 1-5:                             |
  | 1. Faithfulness: Are all claims in the answer supported by        |
  |    the context? (1=hallucinated, 5=fully grounded)                |
  | 2. Relevance: Does the answer address the user's question?        |
  |    (1=off-topic, 5=directly answers)                              |
  | 3. Completeness: Does the answer cover all relevant info?         |
  |    (1=missing key info, 5=comprehensive)                          |
  |                                                                    |
  | Provide scores and brief justifications in JSON.                  |
  +------------------------------------------------------------------+
```

### Known Biases and Mitigations

| Bias | Description | Mitigation |
|------|-------------|------------|
| **Verbosity** | LLM judges prefer longer answers | Normalize scores by answer length; add conciseness penalty |
| **Self-preference** | GPT-4 rates GPT-4 answers higher | Use a different judge model than the generator |
| **Position** | First option in A/B comparisons rated higher | Randomize presentation order |
| **Sycophancy** | Judge agrees with the system being evaluated | Use structured rubrics with specific criteria |
| **Leniency** | LLMs rarely give scores below 3/5 | Use binary (pass/fail) instead of Likert scales |

### Best Practices for LLM-as-Judge

1. **Use binary decisions over scales**: "Is this claim supported? YES/NO" is more reliable than "Rate support on 1-5."
2. **Decompose into atomic evaluations**: Evaluate one claim or one dimension at a time.
3. **Require evidence**: Force the judge to cite the specific context passage that supports/contradicts each claim.
4. **Calibrate with human agreement**: Run 100+ examples through both LLM and human judges. Measure Cohen's Kappa. Target > 0.7.
5. **Use a strong model for calibration, a cheap one for volume**: a frontier judge (Claude Opus 5.5, GPT-6 Sol) for the human-agreement calibration set and CI; a small model for sampled production traffic once it matches the frontier judge's verdicts on that set. Never use the same model that generated the answer.
6. **Pin and version the judge**: a judge swap is a metric change. Claude Sonnet 4.5 retires on November 30, 2026 (Claude API and Foundry), and Claude Haiku 4.5 retires on Foundry on November 15, so teams judging with either there must switch this quarter. The obvious small replacement, Claude Haiku 5.5 (GA October 7), is a metric change too: it rejects `temperature`, `top_p`, `top_k`, and assistant prefill with a 400, and its tokenizer counts about 30% more tokens for the same text. Re-run the human-agreement calibration and re-baseline thresholds before comparing scores across the switch.

### A Third Judge Tier: Typed Decision Models

Many RAG checks are binary ("Is this claim supported? YES/NO"), and a 2026 class of models returns exactly that without generating text. TypeSafe's **Jev** (early access since September 2026) returns typed decisions (yes/no, choice, or score) with probabilities at $0.042 per 1M input tokens, output unmetered, and was integrated into LangSmith, Langfuse, Braintrust, Opik and DeepEval within two weeks of launch. OpenAI's Decisions API (GA October 22, 2026, on `gpt-6-luna`) is the first-party equivalent: up to 400 typed questions per request at $0.10 per 1M uncached input tokens, output not charged.

| | Frontier LLM judge | Small LLM judge | Decision-model judge |
|---|---|---|---|
| **Example** | Claude Opus 5.5, GPT-6 Sol | GPT-6 Luna, Claude Haiku 5.5 | Jev, OpenAI Decisions API |
| **Output** | Verdict plus written rationale | Verdict plus rationale | Typed decision with probability; no rationale, no abstention |
| **Good for** | Calibration, CI, ambiguous cases | Sampled production traffic | Binary claim checks on 100% of traffic |
| **Cannot do** | Cheap volume | Hardest edge cases | Claim extraction, question generation, open-ended critique |

The independent check matters more than the price. A September 2026 study (Rao and Callison-Burch, arXiv 2609.29769) found LLM rubric judges cost 16 to 325x more than Jev with accuracy differing significantly in at most 8 of 27 comparisons, but on Jev's most confident errors about 96% of LLM verdicts repeated the same wrong answer, and no cascade beat the best single judge by more than 2.7 points. **Escalating from a cheap judge to an expensive one saves money; it does not catch correlated errors.** Only a human-labeled slice does. Two reported limitations hit RAG directly: accuracy degrades with irrelevant context (exactly what a long retrieved context contains), and an independent benchmark found weaknesses on graded relevance. Test it at your context lengths before trusting it for faithfulness or context-relevance scoring, and fit any probability threshold on your own labels: an October 2026 benchmark (TypedBench, arXiv 2610.11392) found the hosted decision model it tested systematically underconfident.

---

## Building Golden Test Sets

A golden test set is a curated, versioned collection of (query, expected_context, expected_answer) triples that serves as the ground truth for regression testing.

### Building Process

```
  Step 1: Seed Collection
  +-------------------------------------------------------+
  | Source production queries (logs, support tickets)       |
  | Target: 200-500 diverse queries                        |
  | Coverage: all topics, question types, difficulty levels |
  +-------------------------------------------------------+
            |
            v
  Step 2: Synthetic Augmentation
  +-------------------------------------------------------+
  | Use RAGAS or DataMorgana to generate additional queries |
  | from your corpus:                                       |
  |   - Simple factual questions (40%)                     |
  |   - Multi-hop reasoning questions (25%)                |
  |   - Conditional/comparative questions (20%)            |
  |   - Adversarial/edge cases (15%)                       |
  +-------------------------------------------------------+
            |
            v
  Step 3: Human Annotation
  +-------------------------------------------------------+
  | For each query, annotate:                               |
  |   - Expected relevant document IDs (for retrieval eval) |
  |   - Reference answer (for generation eval)              |
  |   - Difficulty label (easy / medium / hard)             |
  |   - Category tags (topic, question type)                |
  +-------------------------------------------------------+
            |
            v
  Step 4: Versioning and Freezing
  +-------------------------------------------------------+
  | Store in version control (golden_set_v3.json)           |
  | FREEZE the set for each evaluation cycle                |
  | Never modify a frozen set; create a new version         |
  +-------------------------------------------------------+
```

### Golden Set Composition Guidelines

| Question Type | Percentage | Purpose |
|--------------|-----------|---------|
| Simple factual | 40% | Baseline: should always pass |
| Multi-hop reasoning | 25% | Tests cross-document retrieval |
| Comparative | 15% | Tests retrieval of multiple relevant docs |
| Temporal | 10% | Tests handling of versioned/dated content |
| Adversarial | 10% | Tests robustness (unanswerable, out-of-scope) |

### Synthetic Test Generation with RAGAS

```python
# Generate synthetic test queries from your corpus (RAGAS 0.2+ API;
# the 0.1-era ragas.testset.generator / evolutions imports no longer exist)
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from ragas.llms import LangchainLLMWrapper
from ragas.embeddings import LangchainEmbeddingsWrapper
from ragas.testset import TestsetGenerator
from ragas.testset.synthesizers import (
    SingleHopSpecificQuerySynthesizer,
    MultiHopSpecificQuerySynthesizer,
    MultiHopAbstractQuerySynthesizer,
)

generator_llm = LangchainLLMWrapper(ChatOpenAI(model="gpt-4o"))
generator_embeddings = LangchainEmbeddingsWrapper(
    OpenAIEmbeddings(model="text-embedding-3-large")
)
generator = TestsetGenerator(llm=generator_llm, embedding_model=generator_embeddings)

query_distribution = [
    (SingleHopSpecificQuerySynthesizer(llm=generator_llm), 0.40),
    (MultiHopSpecificQuerySynthesizer(llm=generator_llm), 0.35),
    (MultiHopAbstractQuerySynthesizer(llm=generator_llm), 0.25),
]
testset = generator.generate_with_langchain_docs(
    load_documents("./knowledge_base/"),
    testset_size=200,
    query_distribution=query_distribution,
)
# CRITICAL: Always human-review synthetic data before using as ground truth
testset.to_pandas().to_csv("golden_set_draft_v4.csv")
```

**Warning**: Synthetic test sets are a starting point, not a destination. Always validate with human review to avoid testing against artifacts of the generation model.

---

## Automated Regression Testing

Every RAG pipeline change (new embeddings, chunk size, prompt edit, reranker swap) needs automated regression testing before deployment.

### CI/CD Integration

```
  PR (RAG change) --> CI: Load golden set --> Run pipeline --> Compute metrics
                          --> Compare vs. baseline --> FAIL if drop > 5%, WARN if > 2%
                          --> Post metrics table as PR comment
```

### Quality Gates

| Metric | Absolute Minimum | Regression Threshold |
|--------|-----------------|---------------------|
| Recall@10 | 0.85 | 5% drop from baseline |
| MRR | 0.70 | 5% drop |
| Faithfulness | 0.80 | 3% drop |
| Answer Relevance | 0.75 | 5% drop |
| Answer Correctness | 0.70 | 5% drop |

Any metric below its absolute minimum blocks the PR. Any regression beyond threshold triggers a warning and flags the specific queries that degraded.

Run the same suite on a schedule against production, not only on your own PRs. **A pinned model ID is not a pinned behavior**: on September 25, 2026 OpenAI fixed an image-encoding bug in GPT-6 Sol and GPT-6 Luna under unchanged model IDs and told customers to rerun image evals. Image-dependent RAG scores measured before the fix may reflect the defect rather than your pipeline. Aliases move too: since October 8, 2026, Google routes `gemini-3.7-flash` requests to `gemini-3.8-flash` and `gemini-3.5-flash` to `gemini-3.6-flash`, so a judge string can keep working while the model behind it changes. Log the model version the response reports, not just the one you requested.

---

## Production Monitoring

Offline evaluation is necessary but not sufficient. Production queries differ from test sets, and retrieval quality can degrade over time as the corpus changes.

### Key Production Signals

| Signal | What It Detects | How to Measure |
|--------|----------------|----------------|
| **Empty Retrieval Rate** | Queries with no relevant results | % of queries where top-1 similarity < threshold |
| **Similarity Score Drift** | Embedding or corpus degradation | Track avg similarity over time; alert on drop |
| **Faithfulness Sampling** | Hallucination rate in production | Run LLM-as-judge on 5-10% random sample |
| **User Feedback Correlation** | Whether metrics match real quality | Compare thumbs-up/down with automated scores |
| **Latency P99** | Performance degradation | Track retrieval + generation latency |
| **Token Usage** | Cost drift | Monitor avg context tokens per query |

### Retrieval Quality Drift

Drift happens when the corpus changes but embeddings, chunks, or prompts do not keep up. Four common scenarios: (1) new documents with different vocabulary cause embedding space mismatch (fix by re-embedding affected collections); (2) user query patterns shift to topics with no content (detect via empty retrieval rate monitoring); (3) stale content returns outdated answers (add freshness metadata and prefer recent docs); (4) embedding model updates change similarity distributions (re-calibrate all thresholds after model changes).

---

## Cost of Evaluation at Scale

LLM-as-judge evaluation is powerful but expensive. Understanding the cost structure is critical for budgeting.

### Cost per Query (Full RAG Triad)

List prices on November 2, 2026, assuming 90% of tokens are input and 10% output, before reasoning tokens:

| Metric | LLM Calls | Tokens | Claude Opus 5.5 ($4 / $20) | Claude Sonnet 5.5 or GPT-6 Sol ($2 / $10) | GPT-6 Luna or Claude Haiku 5.5 ($0.10 / $0.50) |
|--------|-----------|--------|----------------------------|-------------------------------------------|------------------------------------------------|
| Faithfulness | ~3 (extract + verify) | ~3k | $0.017 | $0.0084 | $0.00042 |
| Context Relevance | ~5 (per chunk) | ~2.5k | $0.014 | $0.0070 | $0.00035 |
| Answer Relevance | ~2 (question gen) | ~1.6k | $0.009 | $0.0045 | $0.00022 |
| **Full Triad** | **~10** | **~7k** | **~$0.04** | **~$0.02** | **~$0.001** |

Haiku 5.5's price holds only while a prompt (counting cache reads and writes) stays at or under 100K tokens. Judge prompts sit far below that, but its tokenizer counts about 30% more tokens than Haiku 4.5's for the same text, so measure its token counts instead of assuming parity with Luna.

Two caveats change the real bill. Claude Opus 5.5 cannot run with thinking disabled (default effort `medium`), so set `low` effort for judge calls and measure the reasoning tokens. And the binary verification calls can move to a decision-model judge (about $0.0003 per query at Jev's $0.042 per 1M input tokens), leaving only claim extraction and question generation on a generative model.

### Scaling Strategy

| Evaluation Type | Frequency | Volume | Judge Model | Monthly Cost (10k queries/day) |
|----------------|-----------|--------|-------------|-------------------------------|
| **CI Regression** | Per PR | Golden set (500 queries) | Frontier (Claude Opus 5.5 / GPT-6 Sol) | ~$20/run |
| **Nightly Batch** | Daily | Random 1k production queries | Small (GPT-6 Luna), Batch API | ~$15/month |
| **Production Sample** | Real-time | 5% of traffic (500/day) | Small (GPT-6 Luna) | ~$15/month |
| **Deep Audit** | Weekly | Full golden set + analysis | Frontier, Batch API | ~$45/month |

Frontier rows use Claude Opus 5.5 prices; GPT-6 Sol at $2 / $10 roughly halves them.

**Insight**: The judge bill is small next to the human-labeling bill; the expensive mistake is a cheap judge that disagrees with your annotators. Use a small model (GPT-6 Luna, Claude Haiku 5.5) for high-volume sampling only after it matches the frontier judge on the calibration set, and reserve Claude Opus 5.5 or GPT-6 Sol for CI and deep audits. Nightly and weekly jobs can wait, so run them through a Batch API at 50% off.

---

## Tools Comparison

### Framework Overview

| Tool | Best For | Open Source | Key Strength | Key Weakness |
|------|----------|------------|--------------|--------------|
| **RAGAS** | Quick RAG evaluation, synthetic data | Yes | Reference-free metrics, strong community | Metric results lack explanations |
| **DeepEval** | CI/CD integration, TDD for LLMs | Yes | pytest-compatible, self-explaining scores | Heavier setup |
| **TruLens** | RAG Triad evaluation, observability | Yes | Coined the RAG Triad, good tracing | Less active development |
| **UpTrain** | Production monitoring, drift detection | Yes | Hybrid eval (LLM + heuristic), drift alerts | Lower ranking accuracy |
| **Braintrust** | Team collaboration, experiment tracking | Commercial | Best UI/UX, experiment comparison | Paid for advanced features |
| **LangSmith** | Tracing, evals, trace-to-fine-tune loop | Commercial | Deepest LangChain/LangGraph integration; SDK and OTel ingestion for other stacks | Usage-based LSU pricing since Oct 1, 2026 (1 LSU = $1); SaaS extended-retention traces capped at 180 days (from Sep 14, 2026): not an audit log |
| **Langfuse** | Self-hosted tracing + evals | Yes | OTel-native, exportable; Python SDK 4.x (`@observe()`; the v2 `trace()` API is gone) | Self-hosting means operating ClickHouse, Redis and object storage alongside Postgres |
| **Arize Phoenix** | Self-hosted tracing + RAG evals | Source-available (Elastic-2.0) | OTel-native, strong retrieval views | Part of Dynatrace since October 1, 2026: watch the roadmap and license |
| **Promptfoo** | CI assertions, red-teaming | Yes (MIT) | Declarative test suites; OpenAI's migration target for OpenAI Evals (shuts down Nov 30, 2026) | Now part of OpenAI (still MIT): weigh vendor neutrality |

### When to Use What

```
  Starting a new RAG project?
    --> RAGAS for quick baseline metrics + synthetic test generation

  Adding RAG eval to CI/CD?
    --> DeepEval (pytest integration, quality gates as assertions)

  Need production monitoring?
    --> UpTrain or Braintrust (drift detection, alerting)

  Want end-to-end observability?
    --> LangSmith or Braintrust (SaaS), Langfuse or Phoenix (self-hosted, OTel)

  Still on OpenAI Evals?
    --> It shuts down November 30, 2026: export datasets, graders and
        results now, then move to Promptfoo or your own harness

  Building custom eval pipeline?
    --> Roll your own with LLM-as-judge + the RAG Triad structure
```

### Custom Evaluator Pattern

The core pattern for a custom evaluator is simple: for each triad dimension, use binary LLM-as-judge calls and aggregate.

```python
# Pseudocode: Core faithfulness evaluator (other dimensions follow the same pattern)

def evaluate_faithfulness(answer: str, context: str, judge) -> float:
    # Step 1: Extract atomic claims from the answer
    claims = judge.generate(f"List every factual claim as a JSON array:\n{answer}")

    # Step 2: Verify each claim against context (binary YES/NO)
    supported = sum(
        1 for claim in json.loads(claims)
        if "YES" in judge.generate(
            f"Is this claim supported by the context? YES or NO.\n"
            f"Claim: {claim}\nContext: {context}"
        ).upper()
    )
    return supported / max(len(json.loads(claims)), 1)
```

Apply the same decompose-then-judge pattern for context relevance (per-chunk: "Is this relevant to the query?") and answer relevance (generate hypothetical questions, measure similarity to original query).

---

## System Design Interview Angle

### Q: You deployed a RAG system and users report that answers are sometimes wrong. How do you systematically diagnose and fix the problem?

**Strong answer:**

I would use the RAG Triad to isolate the failure mode:

1. **Sample failing queries**: Collect 50-100 queries where users flagged bad answers. Categorize them by failure type.

2. **Run the triad**:
   - **Context Relevance low?** --> Retrieval problem. The system is fetching wrong documents. Fix: inspect embedding similarity scores, check if the query language matches document language, try hybrid search (BM25 + dense), add a reranker.
   - **Groundedness low?** --> Hallucination problem. The LLM is making things up despite having good context. Fix: strengthen the system prompt ("Only answer from the provided context"), require citations to chunk IDs and verify them, or switch to a more instruction-following model. Lowering temperature is no longer a universal lever: GPT-6 Astra rejects `temperature`, and the newest Claude models reject non-default sampling parameters.
   - **Answer Relevance low?** --> The system retrieves related content and faithfully summarizes it, but misses the actual question. Fix: improve query understanding (query rewriting, HyDE), add query classification to route to the correct index.

3. **Build a regression test**: Take the 50 failing queries, annotate the expected answers, and add them to the golden test set. Every future pipeline change must pass these cases.

4. **Set up ongoing monitoring**: Sample 5% of production traffic for automated evaluation. Alert when faithfulness drops below 0.80 or context relevance drops below 0.60.

The key insight is that "answers are wrong" is not a diagnosis; it is a symptom. The RAG Triad turns a vague complaint into a specific, actionable root cause.

### Q: How do you evaluate a RAG system when you do not have ground-truth answers?

**Strong answer:**

This is the most common real-world scenario. I use a three-layer approach:

**Layer 1: Reference-free metrics (Day 1)**. RAGAS faithfulness and context relevance require no ground truth. They tell you whether the system is hallucinating and whether retrieval is working. You can run these immediately on any query.

**Layer 2: Synthetic golden set (Week 1)**. Use RAGAS TestsetGenerator to create synthetic (query, answer) pairs from your corpus. This gives you approximate ground truth for answer correctness and context recall. Human-review a sample to validate quality.

**Layer 3: Production-derived golden set (Month 1)**. Mine production logs for queries with high user satisfaction (thumbs-up, no follow-up questions). Have annotators label these with reference answers. This creates a golden set that reflects real usage patterns, not synthetic distributions.

The trade-off is accuracy vs. speed. Layer 1 gives you signal in hours but is approximate. Layer 3 gives you ground truth but takes weeks. Run all three in parallel, starting with Layer 1 for immediate feedback.

### Q: Your RAG evaluation pipeline costs $500/day in LLM judge calls. How do you reduce it?

**Strong answer:**

Five strategies, in order of impact:

1. **Tiered judge models**: Use a small judge such as GPT-6 Luna (~$0.001/query for the full triad) for production sampling (90% of volume), after checking it agrees with the frontier judge on the calibration set. Reserve Claude Opus 5.5 (~$0.04/query) or GPT-6 Sol (~$0.02/query) for CI regression tests and weekly deep audits. With a 20-40x price gap, this alone cuts costs by well over 80%.

2. **Smart sampling**: Do not evaluate every query. Sample 5% of production traffic, stratified by query type and user segment. For CI, only run the golden set (500 queries), not the full synthetic set.

3. **Caching**: Many production queries are similar. Hash the (query, context, answer) tuple and cache evaluation results. Identical or near-identical inputs get the cached score.

4. **Heuristic pre-filters**: Before calling the LLM judge, run cheap heuristic checks. If the answer contains "I don't know" or has zero overlap with the context (ROUGE-L < 0.1), skip the expensive faithfulness evaluation and assign a score directly.

5. **Batch and decision models**: Move nightly and weekly runs to a Batch API (50% off), and move binary claim verification to a typed decision-model judge such as Jev or OpenAI's Decisions API. I would not expect the decision model to add accuracy: research in September 2026 found its confident errors are mostly shared by LLM judges, so cascading saves money but a human-labeled slice is still the real check.

The goal is to spend evaluation budget where it provides the most signal: on ambiguous, borderline cases where the LLM judge's nuanced reasoning matters, and on the human labels that keep every judge honest.

---

## References

- Es et al. "RAGAS: Automated Evaluation of Retrieval Augmented Generation" (2023, arXiv:2309.15217)
- TruLens. "The RAG Triad" (2024)
- DeepEval. "Using the RAG Triad for RAG Evaluation" (2025)
- Confident AI. "RAG Evaluation Metrics" (2025)
- Microsoft. "The Path to a Golden Dataset" (2025)
- Prem AI. "RAG Evaluation: Metrics, Frameworks & Testing" (2026)
- Rao and Callison-Burch. "JEV vs. LLMs as Rubric Judges: Cheaper, Faster, and Wrong in the Same Places" (arXiv 2609.29769, 2026)
- MTEB team. "RTEB: Retrieval Embedding Benchmark" (October 2025)
- Qdrant. "Qdrant-FineWeb-10B" (September 2026)
- RAGAS documentation. "Testset Generation for RAG" (docs.ragas.io)

---

*Previous: [Multi-Modal RAG](12-multimodal-rag.md) | Next: [Production RAG at Scale](14-production-rag-at-scale.md)*
