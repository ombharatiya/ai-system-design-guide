# Frequently Asked Questions: AI Engineering, RAG, and Agents

Short, direct answers to the questions people ask most about modern AI system design. Each answer points to the chapter where the topic is covered in depth.

## Table of Contents

- [General: AI Engineering Role](#general)
- [RAG and Retrieval](#rag)
- [Agents and Tool Use](#agents)
- [Models and Selection](#models)
- [Evaluation and Observability](#evaluation)
- [Inference and Cost](#inference)
- [Memory and State](#memory)
- [Security and Safety](#security)

---

## General

### What is an AI engineer?

An AI engineer builds production systems on top of large language models. The role sits between traditional software engineering and machine learning research: less model training, more system design around models that already exist. Day-to-day work covers prompt and context engineering, retrieval pipelines, agent loops, evaluation harnesses, and the infrastructure that keeps it all online. See [AI Job Market Trends](06-job-market-trends-2026.md).

### What is the difference between an AI engineer and an ML engineer?

ML engineers train, fine-tune, and ship models. AI engineers compose existing models (usually via APIs) into products. ML engineers spend their time on datasets and training loops. AI engineers spend their time on prompts, RAG, agents, evals, and latency. The boundary blurs at large companies that do both. See [Transition Guide](../TRANSITION_GUIDE.md).

### How do I become an AI engineer?

If you already write production code, the gap is small: learn how LLMs behave, learn RAG and agent patterns, learn evaluation, and learn one inference stack. The [Transition Guide](../TRANSITION_GUIDE.md) maps your current role (backend, frontend, QA, PM, data, DevOps) to the AI roles that fit and the specific skills to close.

### What programming language should I learn for AI engineering?

Python is the default for AI work. TypeScript is the most common second language because frontend and edge agent stacks live there. C# and Go show up in enterprise infrastructure roles. Most production AI code reads as ordinary application code: HTTP clients, queue consumers, database calls, plus calls to a model provider.

### Is AI engineering a good career?

For experienced engineers, yes: demand is strong and pay tracks senior software engineering, often far higher at frontier labs (self-reported levels.fyi medians on November 1, 2026: $882,500 for Anthropic software engineers and $640,000 at OpenAI, against $156,000 for US AI engineers overall). AI-related postings reached a record 7.3% of US job postings in October 2026 (Indeed Hiring Lab, a keyword count), even as overall software development postings stayed well below their 2020 level. The entry door is narrower. In the most AI-exposed occupations, the entry-level share of job postings fell from 29% in 2021 to 10% in 2026 (Indeed Hiring Lab), and starting earnings for new graduates of the most exposed majors, led by computer science, fell about 13% relative to the least-exposed majors after late 2022 (US Census Bureau working paper, September 2026), even though recent-graduate unemployment has not spiked. So the strongest path in is a lateral move that keeps your seniority. The other risk is churn: a framework that mattered last year may be in maintenance mode today. The skills that compound across releases are evaluation, system design, and grounded debugging. See [Job Market Trends](06-job-market-trends-2026.md).

### Where can I get a mock AI system design interview or mentorship?

Practice with a person, not only with a page. A mock interview surfaces the problems self-study hides: running out of time before evaluation, not stating tradeoffs, and freezing when the interviewer changes a constraint. Om, who maintains this guide, runs 1:1 mock AI system design interviews, answer and resume reviews, and ongoing mentorship for engineers moving into senior AI roles. Book on [EngineBogie](https://enginebogie.com/u/ombharatiya) or [Topmate](https://topmate.io/ombharatiya). Peers work too: trade mocks with someone preparing for a similar loop and use the [Answer Frameworks](02-answer-frameworks.md) as the scoring rubric.

---

## RAG

### What is RAG?

Retrieval-Augmented Generation is the pattern of fetching external context (documents, rows, code, images) at query time and putting it in the LLM's prompt so the model can ground its answer instead of hallucinating from training data. It is the most common production pattern for any LLM that needs to know things outside its training cutoff. See [RAG Fundamentals](../06-retrieval-systems/01-rag-fundamentals.md).

### How does RAG work?

A user query is converted into a search request, the system retrieves the most relevant chunks from a knowledge store (vector DB, keyword index, graph, or a mix), reranks them, and passes the top results to the LLM as context for generation. The two failure points are retrieval (the right chunk was not returned) and generation (the model ignored or misused the chunk). Most RAG failures are retrieval failures.

### Is RAG dead because of long context windows?

No. Even with roughly 1M-token context windows now standard well beyond the top end (Claude Fable 5.1, Opus 5.5, Sonnet 5.5, and Haiku 5.5; GPT-6 Astra, GPT-6 Sol, and GPT-6.1 Terra at 1.05M; Gemini 4 Argon and Gemini 3.8 Flash), RAG wins on cost, latency, freshness, and corpus scale. Long prompts are priced as a penalty, not a convenience: OpenAI bills the whole request at long-context rates once input passes 272K tokens (GPT-6 Sol goes from $2/$10 to $4/$15 per 1M), xAI doubles every token at 200K and above, and Claude Haiku 5.5 bills the whole request at 5x once the prompt passes 100K tokens (cache reads and writes count toward the threshold). Other Claude 4.6-and-later models, Opus 5.5 and Sonnet 5.5 included, still bill flat to 1M. Enterprise datasets (SharePoint, log archives, code monorepos) exceed any context window. RAG acts as the filter that finds the 0.01% of data worth putting into that high-value window. See [RAG vs Long Context](../06-retrieval-systems/14-production-rag-at-scale.md).

### What is the difference between RAG and fine-tuning?

RAG injects knowledge at query time through context. Fine-tuning bakes behavior into the weights. The rule of thumb: **RAG for facts, fine-tuning for form**. Use RAG when the knowledge changes, when you need citations, or when data must stay outside the model. Use fine-tuning when you want a consistent tone, a strict output format, or lower latency on repeated tasks. See [Fine-Tuning Strategies](../03-training-and-adaptation/02-fine-tuning-strategies.md).

### What is the best vector database?

There is no single best. Pinecone wins on managed scale and SLAs. Qdrant leads open-source speed (roughly 12ms p99 at 10M vectors). Weaviate has the strongest native hybrid (BM25 + dense + metadata in one query). Milvus is the choice once you need distributed scale beyond 50M vectors (run 3.0.3 or later, which fixes a heap out-of-bounds write from a malformed sparse vector and a metadata leak in DescribeCollection). pgvector is the right answer if you are already on Postgres and your index is under 10M vectors (run 0.8.7 or later, which fixes an IVFFlat overflow CVE). See [Vector Databases](../06-retrieval-systems/04-vector-databases.md).

### What is contextual retrieval?

Contextual retrieval is an Anthropic technique that prepends a short LLM-generated context summary to each chunk before embedding and indexing it, so the chunk carries its place in the document with it. Anthropic reports a 49% reduction in retrieval failures with hybrid search, and 67% when combined with a reranker. See [Contextual Retrieval](../06-retrieval-systems/10-contextual-retrieval.md).

### What is hybrid search?

Hybrid search combines sparse keyword retrieval (typically BM25) with dense vector retrieval and fuses the two ranked lists into one, usually with Reciprocal Rank Fusion. The sparse arm catches exact tokens (product codes, function names, rare nouns); the dense arm catches synonyms and intent. Weaviate, Qdrant, Milvus, and Elasticsearch fuse the two natively. Pinecone is the exception: its BM25 full-text search (GA September 9, 2026) ranks by one scoring type per request, so you run both searches and fuse the lists client-side. See [Hybrid Search](../06-retrieval-systems/05-hybrid-search.md).

### What is GraphRAG?

GraphRAG extracts entities and relationships from a corpus, builds a knowledge graph, and queries by traversal instead of (or alongside) vector similarity. It is the right pattern for **aggregative questions** ("summarize all legal risks across these 50 contracts") where vector RAG returns related-but-disconnected chunks. Microsoft's LazyGraphRAG defers expensive community summarization to query time, cutting ingestion cost. See [GraphRAG](../06-retrieval-systems/07-graph-rag.md).

### What is the best chunk size for RAG?

There is no universal answer. 300-500 token chunks with 50-token overlap is a reasonable default for prose. Code and structured data want larger chunks (1000-2000 tokens). The bigger wins come from **structure-aware chunking** (split at headers, paragraphs, code blocks), **contextual chunking** (prepend a summary), and **hierarchical chunking** (index small chunks, return the parent context). See [Chunking Strategies](../06-retrieval-systems/02-chunking-strategies.md).

---

## Agents

### What is an AI agent?

An AI agent is a system where an LLM decides what to do next, runs a tool, observes the result, and decides again, all in a loop. The simplest agent is the ReAct pattern: Thought → Action → Observation → repeat. Current frontier models bake the reasoning step into the model itself: Claude Opus 5.5 and Sonnet 5.5 do not let you switch thinking off, and GPT-6 Astra and Gemini 3.8 Flash expose a reasoning-effort or thinking-level setting instead of a separate Thought step. See [Agent Fundamentals](../07-agentic-systems/01-agent-fundamentals.md).

### What is the difference between an agent and a chatbot?

A chatbot responds to a message. An agent **takes actions** in the world: runs code, calls APIs, reads files, sends messages, books appointments. That distinction matters because actions are hard to roll back, so agents need very different guardrails, sandboxing, and human-in-the-loop patterns. See [Agentic Security](../07-agentic-systems/09-agentic-security-and-sandboxing.md).

### What is MCP (Model Context Protocol)?

MCP is an open protocol that lets LLM applications connect to tools and data sources through a standard interface. Anthropic launched it in November 2024. Governance moved to the Linux Foundation's Agentic AI Foundation (AAIF) in December 2025, and the AAIF now hosts 10 projects, including A2A, goose, AGENTS.md, and (since October 22, 2026) Agent Skills. Adoption is universal: Anthropic, OpenAI, Google, Microsoft, AWS all support it, and by May 2026 there were over 2,300 public MCP servers. The current spec revision, **2026-07-28**, made the protocol core stateless (no `initialize` handshake, no session header), and `pip install mcp` now installs the 2.x Python SDK, where FastMCP was renamed `MCPServer`. The next revision moved from December 2026 to a planned March 30, 2027, and every extension now sits at one of three maturity stages (Experimental, Beta, Stable): MCP Apps and enterprise auth are Stable, while Tasks, Skills, and Server Cards are Beta and can still break. Patch MCP servers and SDKs like any internet-facing dependency: 73 MCP-titled security advisories were published between August 15 and October 1, 2026, 9 of them critical, and some fixes need code changes, not just an upgrade. October's TypeScript SDK advisory CVE-2026-111770 (one client's responses leaking to another through a shared server or transport) is fixed in `@modelcontextprotocol/server` 2.3.0 only if you also create one server and transport per request or session. See [Tool Use and MCP](../07-agentic-systems/03-tool-use-and-mcp.md).

### What is the best agent framework?

The big three cover most production needs. **LangGraph** (1.2.x) is the default for stateful multi-agent control flow with checkpointing, and it leads on usage by a wide margin: about 43.7M PyPI downloads in the month to October 1, 2026, against about 2.4M for CrewAI. CrewAI still has more GitHub stars (about 59K vs 43K), a reminder that stars measure attention and downloads measure dependency. **CrewAI** (1.15.x) is the choice for role-based business automation, and since adding state checkpointing (1.14) and declarative Flows (1.15) it competes on durability too. **Microsoft Agent Framework** went GA (1.0) in April 2026 as the successor to AutoGen (now in maintenance mode) and Semantic Kernel, and is the default for enterprise .NET and Python shops. The lab SDKs (Strands Agents, Claude Agent SDK, OpenAI Agents SDK, Google ADK) now out-download every independent Python agent framework except LangChain and LangGraph, which supports a common split: a lab SDK at the leaf, a neutral orchestrator at the core. Download counts include CI installs, so read them as relative signals. See [Framework Selection Guide](../09-frameworks-and-tools/08-framework-selection-guide.md).

### What is agentic RAG?

Agentic RAG replaces the linear retrieve-then-generate pipeline with a loop where the agent decides what to retrieve, evaluates whether the results are good enough, and re-queries if not. Patterns include Self-RAG (model emits reflection tokens), Corrective RAG (separate grader), Adaptive RAG (classifier picks the depth), and multi-hop decomposition. Budget for 8-12 seconds per query at three to four iterations. See [Agentic RAG](../06-retrieval-systems/08-agentic-rag.md).

### How do computer-use agents work?

A computer-use agent takes screenshots of a desktop or browser, decides on a mouse and keyboard action, executes it, and takes another screenshot. The product has split into three shapes: an API tool you host (Anthropic's `computer_toolset_20260801`, GA on the Claude API since August 19, 2026; Gemini computer use, GA since October 30, 2026, with a built-in `require_confirmation` check), a runtime the vendor hosts (OpenAI's Agents API, still in public beta since September 10, 2026, added computer use in an OpenAI-hosted browser on September 29), and persistent agents with their own cloud computer and identity (Microsoft's Copilot Autopilot, in private preview). Read benchmark numbers carefully. OSWorld-Verified is saturated, and on the harder OSWorld 2.x leaderboard (XLANG, updated October 15, 2026) Claude Opus 5 at max effort still leads end-to-end models, fully completing 44.33% of v2.1 tasks while scoring 77.67% on partial credit; the leaderboard has no rows yet for the 5.5-generation Claude models, GPT-6, or Gemini 4. Anthropic's vendor-reported 81.8% for Opus 5.5 and 72.4% for Haiku 5.5 (on an offline subset) are partial-credit figures. Plan capacity on binary completion, and treat the GUI as your most injectable surface: Haiku 5.5's system card puts prompt-injection attack success within 15 attempts at 7.1% overall but 24.4% on GUI computer use. See [Computer-Use Agents](../17-tool-use-and-computer-agents/04-computer-use-agents.md).

### What is context engineering?

Context engineering is curating the full set of tokens an agent sees on every inference turn (system prompt, tools, retrieved data, prior tool results, message history), as opposed to prompt engineering, which writes one good instruction once. It matters because long-running agents suffer **context rot**: accuracy drops as the window fills with stale tool output. The core techniques are compaction (summarize and restart the loop), just-in-time loading (hold references, fetch on demand), structured note-taking (write progress outside the window), and sub-agent isolation (delegate detail-heavy sub-tasks to a clean window that returns a short summary). The goal is the smallest high-signal token set per turn. See [Context Engineering](../05-prompting-and-context/05-context-engineering.md).

### What are Agent Skills?

Agent Skills are folders of instructions, scripts, and resources that an agent loads on demand to specialize at a task. Each skill is a `SKILL.md` file with YAML metadata plus optional bundled files. They use **progressive disclosure**: the name and description sit in the system prompt, the full `SKILL.md` loads only when the agent judges it relevant, and referenced files load only as needed, which keeps context small. Skills complement MCP: MCP connects an agent to tools and data, Skills teach it the workflow for using them. Skills left beta on the Claude API on August 19, 2026, and MCP servers can now serve them through the official `io.modelcontextprotocol/skills` extension (Final September 13, 2026, rated Beta under MCP's extension maturity stages, so it can still break; the TypeScript SDK shipped support in 2.4.0 on October 23), which binds approval to a SHA-256 manifest of the skill's files: any changed, added, or removed file revokes it. Treat skills like dependencies: pin versions and review updates before an agent loads them. See [Building Tool-Use Agents](../17-tool-use-and-computer-agents/05-building-tool-agents.md).

---

## Models

### What is the best LLM right now?

There is no single best model; pick by task and price tier (prices per 1M input/output tokens; index scores are Artificial Analysis Intelligence Index v5.0 at max effort unless noted). **Claude Opus 5.5** (September 22, $4/$20) is Anthropic's recommended starting point and, at 58, the top publicly available model on the index (Google's Gemini 4 Carbon scores 59 but is limited to a group of cyber defenders). **Claude Sonnet 5.5** (September 28, $2/$10) scores 54 at half the price, **Claude Haiku 5.5** (October 7, $0.10/$0.50 for prompts up to 100K tokens) is the new fast tier at 36, and **Claude Fable 5.1** ($10/$50) is for work where Opus 5.5 at higher effort still falls short. (AA labels the Opus score "with fallback": when safeguards intervened, some tasks ran on older models.) OpenAI's lineup is **GPT-6 Astra** ($10/$50, its ceiling model, 57), **GPT-6.1 Sol** ($2/$10, 55, with 0.05x cache reads), **GPT-6.1 Terra** (October 26, $1/$5, 51), which brings back a middle tier, and **GPT-6 Luna** ($0.10/$0.50) for volume. Google's **Gemini 4 Argon** went GA on October 13 as `gemini-4-argon` (50 at high effort) at $2/$10 through November 30 and $4/$20 from December 1, so budget at the higher price, and **Gemini 3.8 Flash** is the cheap multimodal workhorse at an introductory $0.75/$3.75 that doubles to $1.50/$7.50 on January 1, 2027. AA's v5.0 (late October) rebuilt the index around four equal categories with mostly private or held-out sets, so v4.x scores do not carry over. Scores also do not carry across harnesses, so run your own evals before switching. See [Model Taxonomy](../02-model-landscape/01-model-taxonomy.md).

### How much does Claude / GPT / Gemini / DeepSeek cost?

Pricing changes monthly, so date every number. As of November 2, 2026 (Standard tier, per 1M input/output tokens): the ceiling tier is $10/$50 (Claude Fable 5.1, GPT-6 Astra); Claude Opus 5.5 is $4/$20; the mid tier sits at $2/$10 (Claude Sonnet 5.5, GPT-6 Sol, GPT-6.1 Sol, and Gemini 4 Argon until it moves to $4/$20 on December 1); GPT-6.1 Terra opened a $1/$5 tier on October 26; and the volume tier runs from $0.10/$0.50 (GPT-6 Luna, and Claude Haiku 5.5 for prompts up to 100K tokens) to $0.75/$3.75 (Gemini 3.8 Flash, introductory through December 31, 2026, then $1.50/$7.50). **DeepSeek now prices by time of day**: V4.1-Flash is $0.30/$1.20 at peak and half that off-peak, V4-Pro is $1.32/$3.96 at peak, and peak covers only two UTC windows on weekdays, so the old flat $0.14/$0.28 and $0.435/$0.87 figures are stale. Three modifiers matter as much as list price: cache reads (0.1x input on most models, 0.05x on Opus 5.5, Sonnet 5.5, GPT-6.1 Sol, and GPT-6.1 Terra, 0.025x on Fable 5.1), long-context surcharges (Haiku 5.5 bills the whole request at 5x above a 100K-token prompt, OpenAI reprices the whole request above 272K input, xAI doubles at 200K and above), and promotions (GPT-5.6 Sol's $4/$20 promo is guaranteed only through November 21, 2026, so budget at its $5/$30 list price). Always cross-check the provider pricing pages. See [Pricing and Costs](../02-model-landscape/03-pricing-and-costs.md).

### What is the difference between Claude Opus and Claude Sonnet?

Opus is Anthropic's flagship tier: Opus 5.5 ($4/$20 per 1M) is the model Anthropic tells you to start with. Sonnet is the production workhorse: Sonnet 5.5 ($2/$10) costs half as much and lands within a few points of Opus 5.5 on most of Anthropic's own vendor-reported agentic evals (about 8 points behind on FrontierCode 1.1, but above it on Terminal-Bench 4.0: 70.6% vs 66.4% for Opus 5.5 at xhigh effort, while the public tbench.ai board at max effort has the reverse order, 64.85% vs 61.82%), and since October 7 both read cache at 0.05x of input ($0.10 vs $0.20 per 1M). Haiku is the fast tier for routing, classification, extraction, and subagents. Haiku 5.5 (October 7) lists at $0.10/$0.50, about 90% below Haiku 4.5, but only for prompts up to 100K tokens: above that the whole request bills at $0.50/$2.50, and cache reads and writes count toward the threshold, so it is a short-prompt model by price even with a 1M window. It is not a drop-in swap for Haiku 4.5 either: `budget_tokens`, `temperature`, `top_p`, `top_k`, and assistant prefill return a 400, and the newer tokenizer counts about 30% more tokens for the same text. Above all three sits Claude Fable 5.1 ($10/$50), for demanding reasoning and long-horizon work where Opus 5.5 at higher effort still falls short. The routing pattern still holds (Haiku for easy queries, Sonnet for most traffic, Opus for hard ones, Fable only for ceiling-bound work), but the quality gap between tiers is now narrower than the price gap, so let your evals set the boundaries. Migration trap: Opus 5.5 and Sonnet 5.5 reject forced tool use (`tool_choice` of `any` or `tool` returns a 400; Haiku 5.5 accepts it but skips thinking) and do not let you disable thinking, and Opus 5.5 defaults to `medium` effort where Opus 5 defaulted to `high`. See [Model Selection](../02-model-landscape/04-model-selection-guide.md).

### Should I use an open-source model?

Yes for cost-per-query at high volume, data residency, or fine-tuning needs. On Artificial Analysis Intelligence Index v5.0 the best open-weight models score about 32 against 58 for Claude Opus 5.5, a gap of roughly 26 points. The older v4.x index put the gap near 12, but the scales are not comparable, and AA now counts the API-only Kimi K3.1 Pro as proprietary, so say which index and classification you are quoting. October added large open-weight releases from outside China: Meta's Muse Spark Open (816B total, 42B active) and Mistral Large 4 (about 1T total, 49B active, $0.70/$2.10 on Mistral's API). Read the license, not just the leaderboard. "Open" now ranges from plain MIT (MiMo-V2.6, DeepSeek V4.1-Flash) to MIT plus a security review for model-as-a-service operators above US$10B in revenue (GLM-5.3), to a separate agreement above US$20M in revenue (Kimi K3, Meta's Muse Model License), to a separate commercial license for any model-as-a-service business with no revenue threshold (Mistral's Ecosystem Open-Weight License); the Qwen Community License also requires a separate deal for some model-as-a-service businesses. The tradeoff is operational: you own the inference stack, the GPU bill, and the security patches. See [Model Landscape](../02-model-landscape/01-model-taxonomy.md).

### What is prompt caching?

Prompt caching keeps the KV cache for a fixed prompt prefix warm on the inference server so later calls with the same prefix pay a fraction of the input price for it. All major providers support it. Cache reads cost 0.1x the input rate on most current models (Claude Haiku 5.5 included), 0.05x on Claude Opus 5.5, Claude Sonnet 5.5, GPT-6.1 Sol, and GPT-6.1 Terra, and 0.025x on Claude Fable 5.1. Writes carry a premium: 1.25x for Anthropic's 5-minute cache and OpenAI's 30-minute cache (GPT-5.6 and later), 2x for Anthropic's 1-hour cache, and 2.5x for the 12-hour cache OpenAI added on October 20, 2026. Gemini instead charges hourly storage for explicit caches, and Gemini 4 Argon supports only implicit caching, so cache-control code written for explicit caches does nothing there. At those rates a 1.25x write pays for itself with a single cache hit and a 2x or 2.5x write needs two, so pick the TTL from the gap between requests and then watch hit rate: keep the prefix byte-stable (no timestamps or reordered tools at the top) and measure it, since OpenAI and Anthropic now both expose cache diagnostics. See [KV Cache and Context Caching](../04-inference-optimization/02-kv-cache-and-context-caching.md).

---

## Evaluation

### How do you evaluate an LLM?

LLM evaluation is layered: **reference-free metrics** (faithfulness, relevance, coherence via LLM-as-judge) for rapid iteration, **golden test sets** for regression, and **task-specific metrics** (exact match for QA, BLEU for translation, pass@k for code). For RAG specifically, use the RAG Triad: context relevance, faithfulness, answer relevance. See [LLM Evaluation](../14-evaluation-and-observability/01-llm-evaluation.md).

### What is LLM-as-judge?

LLM-as-judge uses one LLM to score the output of another against a rubric (correctness, helpfulness, safety). It scales where human evaluation cannot, but has known biases: position bias, verbosity bias, self-preference bias. The standard practice is to use a strong judge from a different model family than the generator (for example Claude Opus 5.5 judging GPT-6 Sol output, or the reverse) to limit self-preference, randomize positions, and validate against a small human-labeled sample. Do not assume a second judge catches the first one's mistakes: a September 2026 study (arXiv 2609.29769) found that on a cheap decision-model judge's most confident errors, about 96% of LLM-judge verdicts repeated the same wrong answer, so judge cascades mostly save money rather than add accuracy. The human-labeled set is your only real check on correlated judge error.

### What is the best LLM observability tool?

The leading platforms are **Langfuse** (best self-hosted open-source, part of ClickHouse since January 2026), **Braintrust** (best for eval-driven CI/CD with quality gates), **LangWatch** (best for agent simulation), **LangSmith** (LangChain-native; SaaS traces on extended retention are capped at 180 days since September 14, 2026), and **Arize Phoenix** (OTel-native, Elastic-2.0 licensed; Dynatrace, which agreed in August 2026 to buy Arize for $915M, completed the acquisition on October 1). LangSmith also moved to usage-based pricing in LangChain Standard Units (1 LSU = $1) on October 1, 2026, replacing per-run and uptime pricing, so rerun your cost model before renewing. Pick based on deployment model (SaaS vs self-hosted), whether you need CI/CD gating, and how heavily you use a specific framework. Ownership churn is now a selection criterion too: OpenAI's hosted Evals shuts down on November 30, 2026, with migration pointed at Promptfoo, which is now part of OpenAI and remains MIT-licensed. See [Observability](../14-evaluation-and-observability/02-observability.md).

### What is RAGAS?

RAGAS (Retrieval Augmented Generation Assessment) is a Python library for RAG evaluation. It provides reference-free metrics (faithfulness, answer relevance, context relevance) and reference-based metrics (context recall, context precision) computed with LLM-as-judge. It is the de facto starting point for any RAG eval pipeline. See [RAG Evaluation](../06-retrieval-systems/13-rag-evaluation-patterns.md).

### How do you detect and handle model drift in production?

Drift comes from two sides: the **provider** silently updates a model (or you migrate versions), and your **traffic** shifts away from what your prompts and evals were tuned on. Detection: pin model versions explicitly, run a canary eval suite daily against the pinned and latest versions, track output-distribution stats (length, refusal rate, format validity) and judge-scored quality on a sampled live slice, and alert on deltas. Response: a frozen golden set tells you whether the model changed or your traffic did; re-run prompt evals before adopting a new version, and keep the previous version warm for instant rollback. Silent changes are routine: on October 8, 2026 Google began routing `gemini-3.7-flash` requests to `gemini-3.8-flash`, so the model string kept working while the model behind it changed, and Claude Opus 5.5 ships with a `medium` default effort where Opus 5 used `high`, so a same-family upgrade can shift latency, cost, and quality with no code change unless you pin the setting. Treat an unexplained quality drop like an incident with an on-call path, not a curiosity.

---

## Inference

### What is vLLM?

vLLM is an open-source LLM inference engine that pioneered PagedAttention (virtual-memory-style allocation of the KV cache). It is the default open inference engine when the workload is "Llama, Mistral, Qwen, or DeepSeek under continuous batching." It is the easiest of the major engines to operate, but treat it like an internet-facing service and patch it on a schedule: run v0.32.0 (October 18, 2026) or later. It fixes CVE-2026-115915 (CVSS 8.1, which affects every release before 0.32.0 and was reproduced on 0.30.0) and CVE-2026-115912, on top of the 20+ advisories fixed by 0.30.0, which included a model-load remote code execution, a single-request engine kill, and a prefix-cache timing oracle. Test the upgrade rather than just bumping the pin: 0.32.0 changed the default CORS policy from `*` to none, made Python 3.11 the minimum, and made `max_tokens` an upper bound. See [Serving Infrastructure](../04-inference-optimization/06-serving-infrastructure.md).

### What is the difference between vLLM and SGLang?

Both are open-source inference engines. vLLM has broader model coverage and operational maturity. SGLang has roughly 29% higher throughput on structured-output and function-calling workloads thanks to async constrained decoding, and best-in-class prefix-cache reuse via RadixAttention. Security caveat: upgrading does not close SGLang's current critical exposure. CVE-2026-111391 (CVSS v4 9.2, an eval injection whose description covers 0.5.10 through 0.5.22) and CVE-2026-93034 (CVSS 9.8, in the ZMQ decoder) list no patched version, and v0.5.22 (October 19, 2026) is the latest release. Run a current release for the earlier fixes, keep the ZMQ, dumper, debug, and distributed-init endpoints off the network (inside the pod, never on a routable interface), and do not accept custom logit processors from untrusted callers.

### What is TensorRT-LLM?

NVIDIA's inference engine, tuned for peak throughput on NVIDIA GPUs. Its old reputation for weeks of setup is out of date: since 1.0 (September 2025) the PyTorch backend is the default, so you no longer compile a per-model engine before serving. What remains is hard NVIDIA lock-in and a second serving stack to operate. It is the right choice when you have committed NVIDIA capacity and your own benchmark, on your traffic shape, shows a throughput gain that pays for the lock-in.

### How do you optimize LLM inference cost?

Five high-leverage moves: **model cascading** (route easy queries to small models, hard ones to frontier), **prompt caching** (90-97.5% off cached prefixes on current models), **semantic caching** (skip the LLM call entirely for similar queries), **quantization** (FP8 or 4-bit weights to fit more on a GPU), and **continuous batching** (vLLM/SGLang batch at the iteration level). Together they routinely cut inference cost by 10x without losing quality. See [Cost Optimization](../04-inference-optimization/07-cost-optimization-playbook.md).

### What is speculative decoding?

Speculative decoding lets an LLM generate multiple tokens per forward pass by having a cheaper "draft" model (or a lightweight module trained alongside the main model) predict the next few tokens, then verifying them all in a single parallel pass on the target model. Medusa-style heads have given way to EAGLE-3, multi-token prediction (MTP) layers, and newer methods such as DFlash and DSpark, and some open-weight releases (DeepSeek-V4-Flash-DSpark) now ship the speculative module inside the checkpoint. The win is typically 2-3x faster wall-clock generation with no quality loss, because the target model verifies every token, though the gain shrinks as batch size grows. Built into vLLM, SGLang, and TensorRT-LLM. See [Speculative Decoding](../04-inference-optimization/03-speculative-decoding.md).

### What is a token budget and how do you enforce it?

A token budget is a hard ceiling on token consumption at a chosen scope: per request (max input plus `max_tokens` output), per user or tenant per day, per agent task (step budgets so a runaway loop cannot burn the month's spend), and per team per month for expensive tiers. Enforcement lives in the gateway, not in prompts: count tokens before dispatch, reject or downgrade requests over the per-request cap, decrement tenant quotas atomically, and terminate agent runs that exceed their step budget with a clean escalation. The practical pairing is budget plus routing: when a tenant nears quota, the router degrades them to a cheaper tier instead of cutting them off.

### How do you design fallbacks across multiple LLM providers?

Treat providers as unreliable dependencies with policy differences, not just uptime differences. The pattern: a primary with one or two fallbacks per task type, circuit breakers per provider on error rate, p95 latency, and rate-limit headroom, with pre-emptive traffic shifting before hard limits. At least one fallback must cross vendors: Anthropic logged at least 12 major or critical incidents between August 16 and September 29, 2026, and a September 29 OpenAI outage ran about 5 hours 20 minutes across the API, ChatGPT, and Codex. Four details separate production designs from whiteboard ones: **provider-paired prompts** (a prompt tuned for Claude underperforms on GPT-6 Sol, so prompt variants version with the provider), **cache economics** (failing over resets your prefix cache, so a warm primary can beat a nominally cheaper cold fallback), **state portability** (Claude Fable 5.1, Opus 5.5, and Sonnet 5.5 bind thinking blocks to the model and conversation that produced them, so a mid-conversation failover must strip prior thinking blocks), and **policy-aware fallback** (a provider can decline a content category your product needs, which is a failure class your health checks will not catch). Verify the whole chain monthly with a game-day drill.

---

## Memory

### What is the best AI agent memory framework?

The four mature options all changed shape in 2026. **Mem0** is the broadest standalone memory layer; since v2.0 (April 2026) it extracts with a single ADD-only pass, dropped its graph-database drivers in favor of entity links in the vector store, and moved conflict merging to Dream, a paid-plan feature. **Zep** (with its open-source Graphiti engine) is the temporal knowledge graph, where every fact carries validity times. **Letta** archived its MemGPT-era server in August 2026; memory is now a git-backed folder of Markdown files the agent edits with ordinary file tools, with no vector index by default. **Cognee** is knowledge-graph-first for RAG-heavy workflows. Pick by use case: cross-session personalization → Mem0; facts that change over time → Zep; long-horizon task agent → Letta; KG-grounded RAG → Cognee. Then test against a no-memory baseline: On MemTrapBench (August 2026) tasks built to trigger memory traps (reasoning fixation, belief distortion), every memory strategy it tested underperformed no memory, even though the stored memories were correct and relevant. See [Agentic Memory](../08-memory-and-state/04-agentic-memory-mem0.md).

### What is the difference between short-term and long-term memory in agents?

Short-term memory lives in the LLM's context window (the current turn, tool outputs, scratchpad). Long-term memory persists across sessions in a vector DB, graph, or relational store. Memory architectures split further: **episodic** (past trajectories), **semantic** (extracted facts about the user/world), **procedural** (learned skills, playbooks). Picking the right tier for a given fact matters: a session preference promoted to long-term memory leaks across sessions. See [Memory Architectures](../08-memory-and-state/01-memory-architectures.md).

### How does a knowledge graph help an AI agent?

A knowledge graph stores entities and relationships explicitly (User → OWNER_OF → Project_A). It gives the agent **deterministic** retrieval over structured relationships, which vector search cannot. The strongest pattern is hybrid: vector search finds the entry node by similarity, graph traversal expands relevant context. Used for compliance, multi-hop reasoning, and any domain where relationships are first-class (legal, biomedical, finance). See [Long-Term Memory](../08-memory-and-state/03-long-term-memory.md).

---

## Security

### What is prompt injection?

Prompt injection is the LLM-era version of SQL injection: malicious content in a user input or a retrieved document overrides the system instructions and makes the model do something it should not. **Direct injection** is in the user prompt. **Indirect injection** is hidden in a document the model reads (a webpage, an email, a PDF). The OWASP LLM Top 10 lists it as the #1 LLM risk. See [Prompt Injection Defense](../05-prompting-and-context/08-prompt-injection-defense.md).

### How do you prevent prompt injection?

There is no silver bullet: prompt injection cannot be fully "escaped" the way SQL can. The production defense stack combines: **input isolation** (XML tags marking untrusted content), **dual-LLM patterns** (small guard model classifies intent before the main model sees the input), **canary tokens** (detect if the model leaked its system prompt), **least-privilege tool scopes**, and **human-in-the-loop** on destructive tool calls, with the approval bound to the exact action that executes (2026 research on "approval laundering" showed the action a human approves can differ from the one that runs). See [Agentic Security](../07-agentic-systems/09-agentic-security-and-sandboxing.md).

### What is OWASP LLM Top 10?

The OWASP Top 10 for LLM Applications (2025 edition) is the canonical list of LLM security risks: prompt injection, sensitive information disclosure, supply chain, data and model poisoning, improper output handling, excessive agency, system prompt leakage, vector and embedding weaknesses, misinformation, and unbounded consumption. It replaced the 2023 list, folding insecure plugin design, overreliance, and model theft into broader entries. For agents, pair it with the separate OWASP Top 10 for Agentic Applications (2026), which covers risks such as agent goal hijacking, tool misuse, identity and privilege abuse, memory and context poisoning, and cascading failures. See [LLM Security](../12-security-and-access/01-llm-security.md).

### What is sandboxing in AI agents?

Sandboxing isolates the code an agent generates and runs from the host system. The standard pattern uses ephemeral sandboxes, either microVMs (Firecracker, which E2B builds on) or hardened containers (gVisor), that start in well under a second, run the code, and get destroyed. Without sandboxing, a prompt-injected agent can `rm -rf /` or exfiltrate secrets. With sandboxing, the worst case is a destroyed throwaway environment, but only if the sandbox also controls what crosses its boundary. In 2026 an OpenAI research agent reached an external service by tunneling through DNS, and the GitSpawn disclosure showed that a `.git` directory copied in intact (an archive or synced folder, not a clone) can use its `fsmonitor` setting to run commands outside the sandbox in seven coding-agent CLIs. Default-deny network egress (DNS included), keep credentials out of the sandbox, and treat repository config as untrusted input. See [Agentic Security](../07-agentic-systems/09-agentic-security-and-sandboxing.md).

---

## Related Reading

- [Question Bank (155 senior interview questions)](01-question-bank.md)
- [Answer Frameworks](02-answer-frameworks.md)
- [Common Pitfalls](03-common-pitfalls.md)
- [Whiteboard Exercises](04-whiteboard-exercises.md)
- [AI Job Market Trends](06-job-market-trends-2026.md)

---

*Have a question that should be here? Open an issue or PR on the repo.*
