# 🔄 Transitioning to AI Engineering Roles

> A concrete, role-specific guide for engineers, PMs, QAs, and managers moving into AI-focused positions.
> **No generic advice. Every path maps to real skills, real repo sections, and real courses.**

---

## Who This Guide Is For

You currently work as a software engineer, QA, PM, EM, or data engineer, and you want to move into an AI-focused role. This guide maps your **existing skills** to specific AI roles, tells you **exactly what gaps to close**, and points you to the right sections of this repo and courses to fill them.

---

## The AI Role Landscape

Before picking a path, understand what the target roles actually are:

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI ROLE LANDSCAPE                            │
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────────────┐  │
│  │   APPLICATION LAYER  │    │     INFRASTRUCTURE LAYER     │  │
│  │                      │    │                              │  │
│  │  LLM App Engineer    │    │  MLOps / AI Infra Engineer   │  │
│  │  AI Product Engineer │    │  AI Platform Engineer        │  │
│  │  Agentic Systems Eng │    │  AI Reliability Engineer     │  │
│  └──────────────────────┘    └──────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────────────┐  │
│  │    QUALITY LAYER     │    │     LEADERSHIP LAYER         │  │
│  │                      │    │                              │  │
│  │  AI Eval Engineer    │    │  AI Product Manager          │  │
│  │  AI Quality Engineer │    │  AI Engineering Manager      │  │
│  │  Red Team Analyst    │    │  AI Program Manager          │  │
│  └──────────────────────┘    └──────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────────────┐  │
│  │    RESEARCH LAYER    │    │     SPECIALIST LAYER         │  │
│  │                      │    │                              │  │
│  │  Applied AI Scientist│    │  Agentic Coding Specialist   │  │
│  │  Fine-tuning Engineer│    │  RAG Architect               │  │
│  │  Alignment Researcher│    │  AI Safety Engineer          │  │
│  └──────────────────────┘    └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Transition Paths by Current Role

---

### 1. 🖥️ Backend Engineer → AI Engineering

**Why backend is the best starting point:** You already understand APIs, latency, databases, distributed systems, and production reliability. AI applications need all of these. The gap is mostly domain knowledge, not engineering fundamentals.

#### Target Roles

```
Backend Engineer
      │
      ├──► LLM Application Engineer       (most common transition, 3–6 months)
      ├──► Agentic Systems Engineer        (3–9 months)
      ├──► AI Infrastructure / MLOps Eng  (6–12 months, needs GPU/serving knowledge)
      └──► RAG Architect                  (4–8 months)
```

#### Skill Gap Analysis

| You already have | Gap to close | Priority |
|-----------------|--------------|----------|
| REST API design | LLM API integration patterns | 🔴 High |
| Database design | Vector databases (Qdrant, Pinecone, Weaviate) | 🔴 High |
| Async/streaming | Streaming LLM responses, token streaming | 🔴 High |
| Auth & multi-tenancy | Multi-tenant RAG isolation | 🔴 High |
| Caching (Redis, CDN) | Prompt caching, semantic caching | 🟡 Medium |
| Monitoring (Prometheus) | LLM observability (traces, evals) | 🟡 Medium |
| CI/CD | LLMOps pipelines, model version management | 🟡 Medium |
| N/A | Embedding models and vector math | 🟡 Medium |
| N/A | Prompt engineering fundamentals | 🟡 Medium |
| N/A | RAG pipeline architecture | 🟡 Medium |
| N/A | Agent frameworks (LangGraph, CrewAI) | 🟢 Lower |
| N/A | Fine-tuning concepts (LoRA, RLHF) | 🟢 Lower |

#### Your 90-Day Plan

**Month 1: LLM Integration**
- Learn OpenAI / Anthropic API (streaming, function calling, structured output). On OpenAI, use the Responses API: the Assistants API shut down on August 26, 2026, so skip tutorials built on it
- Build a simple RAG system: PDF ingestion → Qdrant → LLM response
- Read this repo: [01-foundations](01-foundations/), [02-model-landscape](02-model-landscape/), [05-prompting-and-context](05-prompting-and-context/)
- Course: *ChatGPT Prompt Engineering for Developers* (DeepLearning.AI, free)

**Month 2: Production Patterns**
- Add multi-tenant isolation to your RAG system
- Add LangSmith or Langfuse tracing  
- Implement prompt caching for cost savings
- Read this repo: [06-retrieval-systems](06-retrieval-systems/), [12-security-and-access](12-security-and-access/), [08-memory-and-state](08-memory-and-state/)
- Course: *Building and Evaluating Advanced RAG* (DeepLearning.AI, free)

**Month 3: Agentic Systems**
- Build a LangGraph agent with tools (web search, code execution)
- Add evaluation pipeline with RAGAS or Phoenix
- Deploy with proper cost controls and rate limiting
- Read this repo: [07-agentic-systems](07-agentic-systems/), [09-frameworks-and-tools](09-frameworks-and-tools/), [14-evaluation-and-observability](14-evaluation-and-observability/)
- Course: *AI Agents in LangGraph* (DeepLearning.AI, free)

#### Portfolio Project Ideas
- Multi-tenant document Q&A service with access control
- Agentic code reviewer that posts GitHub PR comments
- RAG-powered internal knowledge base with eval pipeline

---

### 2. 🎨 Frontend Engineer → AI Product Engineering

**Why this transition works:** Frontend engineers understand UX, real-time UI updates, and user behavior. AI products live or die on UX: streaming responses, progressive rendering, loading states, feedback collection. Your skills are more valuable than you think.

#### Target Roles

```
Frontend Engineer
      │
      ├──► AI Product Engineer         (3–6 months, highest demand)
      ├──► AI UX Engineer              (3–6 months, UX focus)
      └──► Full-Stack LLM Engineer     (6–9 months, add backend LLM skills)
```

#### Skill Gap Analysis

| You already have | Gap to close | Priority |
|-----------------|--------------|----------|
| Streaming UI (SSE, WebSocket) | LLM token streaming integration | 🔴 High |
| State management | Conversation state, session memory | 🔴 High |
| User feedback patterns | AI feedback collection (thumbs, ratings) | 🔴 High |
| Form validation | Prompt input validation and sanitization | 🟡 Medium |
| Error handling for async | LLM timeout, fallback, retry patterns | 🟡 Medium |
| A/B testing | LLM A/B testing and variant tracking | 🟡 Medium |
| N/A | Basic prompt engineering | 🟡 Medium |
| N/A | LLM API integration (at least one provider) | 🟡 Medium |
| N/A | Understanding of context windows | 🟡 Medium |
| N/A | Basic RAG concepts (what it is and why) | 🟢 Lower |

#### Your 90-Day Plan

**Month 1: Integrate LLMs into UI**
- Build a streaming chat interface (Next.js + Vercel AI SDK)
- Implement proper loading states, token-by-token rendering, error boundaries
- Add a feedback widget (thumbs up/down, regenerate button)
- Course: *ChatGPT Prompt Engineering for Developers* (DeepLearning.AI, free)

**Month 2: UX Patterns for AI**
- Implement conversation memory with session state
- Add citation rendering for RAG responses
- Build a prompt playground UI for your team
- Read this repo: [08-memory-and-state](08-memory-and-state/), [05-prompting-and-context](05-prompting-and-context/)

**Month 3: Eval Integration**
- Instrument your UI to collect feedback signals
- Connect feedback to a Langfuse or LangSmith project
- Run a basic A/B test between two prompt variants
- Read this repo: [14-evaluation-and-observability](14-evaluation-and-observability/)
- Course: *Evaluating and Debugging Generative AI* (DeepLearning.AI + W&B, free)

#### Portfolio Project Ideas
- Streaming document editor with AI suggestions and inline citations
- Multi-step AI form wizard with persistent context
- AI feedback dashboard showing per-feature quality metrics

---

### 3. 🧪 QA Engineer → AI Eval Engineer

**Why QA is the most underrated path:** AI evaluation is essentially a new form of QA. Manual test case design, edge case thinking, regression prevention: these are exactly what AI systems need. But the tools are different, and the mindset around non-deterministic outputs needs to shift.

#### Target Roles

```
QA Engineer
      │
      ├──► AI Eval Engineer            (3–6 months, best fit, fast transition)
      ├──► AI Quality Engineer         (3–6 months)
      └──► Red Team Analyst            (6–9 months, security focus)
```

#### Skill Gap Analysis

| You already have | Gap to close | Priority |
|-----------------|--------------|----------|
| Test case design | Eval dataset creation (dimensional sampling) | 🔴 High |
| Regression testing mindset | Eval suites as CI quality gates | 🔴 High |
| Bug reporting | Error analysis methodology (open/axial coding) | 🔴 High |
| Test automation | LLM-as-judge evaluator automation | 🔴 High |
| Non-functional testing | Hallucination, bias, toxicity detection | 🟡 Medium |
| User acceptance testing | Human annotation workflows | 🟡 Medium |
| N/A | Tracing and observability setup | 🟡 Medium |
| N/A | RAGAS metrics (faithfulness, relevance, recall) | 🟡 Medium |
| N/A | Basic prompt engineering | 🟢 Lower |
| N/A | Python scripting for eval pipelines | 🟢 Lower |

#### Your 90-Day Plan

**Month 1: Error Analysis Foundation**
- Set up Langfuse or Phoenix tracing on any LLM application (your own or open source)
- Do 3 rounds of manual error analysis: review 50 traces, write notes, categorize
- Read the repo's evals companion guides:
  - [AI Evals: Comprehensive Study Guide](ai_evals_comprehensive_study_guide.md)
  - [AI Evals: LangWatch + Langfuse Guide](ai_evals_complete_guide_langwatch_langfuse.md)
- Course chapter to read: *Error Analysis: The Secret Sauce* (inside evals guides, Chapter 3)

**Month 2: Build Evaluators**
- Write 3 code-based evaluators (JSON schema check, format validator, regex-based)
- Write 1 LLM-as-judge evaluator with Train/Dev/Test calibration
- Introduce `judgy` for statistical bias correction
- Read this repo: [14-evaluation-and-observability](14-evaluation-and-observability/)
- Course: *Quality and Safety for LLM Applications* (DeepLearning.AI + WhyLabs, free)

**Month 3: CI/CD Integration**
- Wire evaluators into a GitHub Actions workflow so evals run on every PR
- Define quality gates (faithfulness > 0.85, format pass rate > 0.99)
- Create a weekly eval report dashboard
- Course: *AI Evals For Engineers & PMs* (Maven, Hamel + Shreya; paid, worth it for career transition)

#### Portfolio Project Ideas
- Open-source eval suite for a public LLM application
- Blog post: "How I applied QA methodology to catch LLM failures"
- Eval pipeline template repo with LangSmith + GitHub Actions

---

### 4. 📋 Product Manager → AI Product Manager

**Why PMs are uniquely positioned:** AI products fail not because of bad models but because of bad product decisions (wrong problem, wrong eval criteria, wrong success metrics). PMs who understand AI failure modes are extremely rare and highly valued.

#### Target Roles

```
Product Manager
      │
      ├──► AI Product Manager           (3–6 months, direct analog)
      ├──► AI Program Manager           (3–6 months, coordination focus)
      └──► Head of AI Product           (9–18 months, leadership path)
```

#### Skill Gap Analysis

| You already have | Gap to close | Priority |
|-----------------|--------------|----------|
| User research | Error analysis as voice-of-customer | 🔴 High |
| Success metrics definition | AI-specific metrics (faithfulness, completion rate) | 🔴 High |
| Roadmap prioritization | Failure mode prioritization from eval data | 🔴 High |
| A/B testing | LLM A/B testing design (prompt variants, models) | 🔴 High |
| Stakeholder communication | Explaining AI limitations to partners | 🟡 Medium |
| PRD writing | AI system capability docs and constraint documentation | 🟡 Medium |
| N/A | How LLMs work at a high level (no code required) | 🟡 Medium |
| N/A | RAG pipeline concepts | 🟡 Medium |
| N/A | Tracing / observability tools (Langfuse UI) | 🟡 Medium |
| N/A | Prompt engineering basics | 🟢 Lower |

#### Your 90-Day Plan

**Month 1: Build Technical Vocabulary**
- Read this repo's foundations, WITHOUT skipping to code:
  - [01-foundations](01-foundations/): understand transformers conceptually
  - [02-model-landscape](02-model-landscape/): know which models exist and what they cost
  - [GLOSSARY.md](GLOSSARY.md): learn the vocabulary
- Course: *AI for Everyone* (Coursera, Andrew Ng, free), designed for non-technical roles

**Month 2: Own Error Analysis**
- Ask your engineering team to set up Langfuse or LangSmith
- Personally review 100+ traces from your product; take notes and find patterns
- Run an error analysis session with your team; lead the failure mode categorization
- Read this repo: [14-evaluation-and-observability](14-evaluation-and-observability/)
- Read: Chapter 3 (Error Analysis) in [AI Evals Comprehensive Study Guide](ai_evals_comprehensive_study_guide.md)

**Month 3: Define Your Eval Strategy**
- Write an "AI Quality Spec" for your product: define what good looks like for each feature
- Work with engineers to instrument evals for those criteria
- Set success metrics for your next quarter that include AI quality gates (not just user growth)
- Course: *AI Evals For Engineers & PMs* (Maven, Hamel + Shreya; explicitly designed for PMs)

#### Skills That Make You Stand Out as an AI PM
- You've personally reviewed traces (most PMs delegate this)
- You can define failure modes quantitatively, not just qualitatively
- You can communicate the cost of quality improvements (prompt changes vs. model upgrades vs. fine-tuning)
- You understand the difference between RAG, fine-tuning, and prompt engineering, and when each is appropriate

---

### 5. 👨‍💼 Engineering Manager → AI Engineering Manager

**The EM transition is about leadership evolution:** Technical literacy in AI is necessary but not sufficient. The key shift is managing non-deterministic systems, teams evaluating quality without ground truth, and a field that changes every 3–6 months.

#### Target Roles

```
Engineering Manager
      │
      ├──► AI Engineering Manager       (6–12 months)
      ├──► Director of AI Engineering   (12–24 months)
      └──► VP of AI / Head of AI        (18–36 months)
```

#### What Changes as an AI EM

| Traditional EM | AI EM additions |
|----------------|-----------------|
| Sprint planning | Eval-driven iteration cycles |
| PR review standards | Eval suite as the new "tests pass" bar |
| Hiring for backend/frontend | Hiring for LLM, vector search, evals expertise |
| Incident response for outages | Incident response for quality regressions |
| Roadmap with feature flags | Roadmap with model upgrade risks |
| Performance reviews based on delivery | Performance reviews including AI quality ownership |

#### Your 90-Day Plan

**Month 1: Technical Depth**
- Read all of [09-frameworks-and-tools](09-frameworks-and-tools/) to understand the tooling landscape
- Read [09-claude-code.md](09-frameworks-and-tools/09-claude-code.md) and [10-opencoderguide.md](09-frameworks-and-tools/10-opencoderguide.md); you'll manage teams using these
- Understand costs: read [02-model-landscape/03-pricing-and-costs.md](02-model-landscape/03-pricing-and-costs.md)
- Course: *Generative AI with LLMs* (Coursera, DeepLearning.AI), which gives you enough depth to lead technical discussions

**Month 2: Process and Team Design**
- Redesign your team's definition of "done" to include eval gates
- Build an eval culture: weekly trace reviews, quality metrics in retros
- Define your AI incident runbook: what happens when hallucination rate spikes?
- Read this repo: [13-reliability-and-safety](13-reliability-and-safety/), [14-evaluation-and-observability](14-evaluation-and-observability/)

**Month 3: Strategy and Hiring**
- Define the AI skills matrix for your team: who has what, what's missing
- Build an interview rubric for AI engineers (use [00-interview-prep](00-interview-prep/) as your source)
- Set team-level AI quality OKRs for next quarter
- Course: *CS294 LLM Agents* (Berkeley, free), which gives you the depth for strategy conversations

---

### 6. 🛠️ DevOps / Platform Engineer → MLOps / AI Infrastructure Engineer

**Why platform engineers thrive here:** Kubernetes, CI/CD, observability, cost management, SLAs: you've done all of this. The AI-specific additions are GPU scheduling, model serving, and LLMOps pipelines.

#### Target Roles

```
DevOps / Platform Engineer
      │
      ├──► MLOps Engineer               (3–6 months)
      ├──► AI Infrastructure Engineer   (6–9 months)
      └──► AI Platform Engineer         (9–12 months)
```

#### Skill Gap Analysis

| You already have | Gap to close | Priority |
|-----------------|--------------|----------|
| Container orchestration (K8s) | GPU node pools, NVIDIA device plugins | 🔴 High |
| CI/CD pipelines | LLMOps pipelines (model eval, deployment gates) | 🔴 High |
| Observability stacks | LLM-specific metrics (token throughput, TTFT) | 🔴 High |
| Cost management | GPU cost optimization, spot instances for training | 🔴 High |
| Secret management | API key rotation for multiple LLM providers | 🟡 Medium |
| N/A | vLLM / SGLang for self-hosted model serving (Hugging Face TGI is in maintenance mode) | 🟡 Medium |
| N/A | Model versioning and registry | 🟡 Medium |
| N/A | Quantization basics (GPTQ, AWQ, GGUF, FP8, NVFP4) | 🟡 Medium |
| N/A | Basic prompt engineering to understand what you're serving | 🟢 Lower |

#### Your 90-Day Plan

**Month 1: LLM Serving**
- Deploy vLLM 0.32.0 or later (the security floor since October 18, 2026: 0.30.0 closed more than 20 advisories from August and September, including a model-load remote-code-execution bug, and 0.32.0 fixes two October CVEs that affect it; note that 0.32.0 requires Python 3.11 and no longer allows all CORS origins by default) serving a small Apache-2.0 model such as IBM Granite 4.2 8B
- Add Prometheus metrics: tokens/sec, latency P50/P95/P99, queue depth
- Set up auto-scaling based on request queue
- Read this repo: [04-inference-optimization](04-inference-optimization/), [11-infrastructure-and-mlops](11-infrastructure-and-mlops/)
- Course: *Efficiently Serving LLMs* (DeepLearning.AI + Predibase, free)

**Month 2: LLMOps Pipeline**
- Set up LangSmith or Langfuse for trace collection
- Build a CI/CD quality gate: eval suite runs before model deploy
- Implement prompt version control (Langfuse prompt registry or DSPy)
- Read this repo: [14-evaluation-and-observability](14-evaluation-and-observability/)

**Month 3: Scale and Cost**
- Compare self-hosted vs. API cost at target volume (use pricing guide in repo)
- Set up cost dashboards per model, per team, per feature
- Implement graceful multi-provider failover
- Course: *Machine Learning in Production* (Coursera, DeepLearning.AI)

---

### 7. 📊 Data Engineer → AI Data / Feature Engineer

**Why data engineers are essential:** Training data is the competitive moat of AI systems. Data pipelines, quality, and freshness determine model performance more than architecture. Your skills are immediately applicable.

#### Target Roles

```
Data Engineer
      │
      ├──► AI Data Engineer             (2–4 months, fastest transition)
      ├──► Embedding Pipeline Engineer  (3–6 months)
      └──► Fine-tuning Data Specialist  (4–8 months)
```

#### Skill Gap Analysis

| You already have | Gap to close | Priority |
|-----------------|--------------|----------|
| ETL pipelines | Document ingestion pipelines for RAG | 🔴 High |
| Data quality checks | Eval dataset quality validation | 🔴 High |
| Schema design | Metadata schema for vector databases | 🔴 High |
| Streaming pipelines | Real-time embedding and index update | 🟡 Medium |
| N/A | Embedding model selection and batching | 🟡 Medium |
| N/A | Vector database operations (upsert, filter, ANN search) | 🟡 Medium |
| N/A | Chunking strategies for document types | 🟡 Medium |
| N/A | Annotation pipeline design for fine-tuning | 🟢 Lower |
| N/A | RLHF preference data format | 🟢 Lower |

#### Your 90-Day Plan

**Month 1: RAG Data Pipeline**
- Build an ingestion pipeline: PDF/HTML/DOCX → chunked → embedded → Qdrant
- Add data quality gates: minimum chunk size, deduplication, language detection
- Implement incremental sync: only re-embed changed documents
- Read this repo: [06-retrieval-systems/02-chunking-strategies.md](06-retrieval-systems/02-chunking-strategies.md), [10-document-processing](10-document-processing/)

**Month 2: Eval Dataset Engineering**
- Build a test dataset using dimensional sampling (see evals guides)
- Set up human annotation pipeline using Label Studio or Argilla
- Track inter-annotator agreement; reject low-quality labels
- Read: [AI Evals Comprehensive Study Guide](ai_evals_comprehensive_study_guide.md), Chapter 12 (Human Annotation)
- Course: *Finetuning Large Language Models* (DeepLearning.AI, free)

**Month 3: Advanced Data Engineering**
- Build a pipeline that turns production traces into fine-tuning examples
- Implement embedding drift detection: alert when document distribution shifts
- Benchmark 3 embedding models on your domain data
- Read this repo: [03-training-and-adaptation](03-training-and-adaptation/)

---

## 📊 Role Comparison Overview

```
Role            Months to   Avg Salary    Best Suited For
                First Role   (US, 2026)
────────────────────────────────────────────────────────
Backend         3–6 mo       $170–220K    LLM App / Agentic Engineering
Frontend        3–6 mo       $150–190K    AI Product / UX Engineering
QA              3–6 mo       $140–180K    AI Eval / Quality Engineering
PM              3–6 mo       $160–200K    AI Product Management
DevOps          3–6 mo       $170–220K    MLOps / AI Platform
Data Eng        2–4 mo       $165–210K    RAG Data, Fine-tuning Data
EM              6–12 mo      $200–280K    AI Engineering Manager
```

*Salaries are US market estimates based on Levels.fyi and LinkedIn data, May 2026. Ranges vary significantly by company, location, and experience level. For a later anchor, levels.fyi's self-reported US median for AI Engineer was $160K on October 2, 2026.*

---

## 🗺️ Which Repo Sections Map to What

Use this when you're ready to go deep:

| Topic | Repo Section | Why |
|-------|-------------|-----|
| How LLMs work | [01-foundations](01-foundations/) | Foundation for everything else |
| Which model to use | [02-model-landscape](02-model-landscape/) | Model selection is a daily decision |
| Fine-tuning | [03-training-and-adaptation](03-training-and-adaptation/) | For fine-tuning data specialist path |
| GPU serving / vLLM | [04-inference-optimization](04-inference-optimization/) | MLOps / Platform path |
| Prompt engineering | [05-prompting-and-context](05-prompting-and-context/) | Everyone needs this |
| RAG pipeline | [06-retrieval-systems](06-retrieval-systems/) | Backend / Data Eng path |
| Agentic systems | [07-agentic-systems](07-agentic-systems/) | Backend / Senior AI Eng path |
| Memory & state | [08-memory-and-state](08-memory-and-state/) | All engineers building agents |
| LangGraph, CrewAI, Claude Code | [09-frameworks-and-tools](09-frameworks-and-tools/) | Practical tool selection |
| Document parsing | [10-document-processing](10-document-processing/) | Data Eng / RAG path |
| GPU infra, LLMOps | [11-infrastructure-and-mlops](11-infrastructure-and-mlops/) | DevOps / Platform path |
| Multi-tenant security | [12-security-and-access](12-security-and-access/) | Backend / PM path |
| Guardrails, red teaming | [13-reliability-and-safety](13-reliability-and-safety/) | QA / Red Team path |
| RAGAS, LangSmith, evals | [14-evaluation-and-observability](14-evaluation-and-observability/) | QA / PM / all roles |
| Design patterns | [15-ai-design-patterns](15-ai-design-patterns/) | Senior level preparation |
| Case studies | [16-case-studies](16-case-studies/) | Interview prep, reference designs |
| Evals deep dive | [AI Evals Comprehensive Guide](ai_evals_comprehensive_study_guide.md) | QA / PM path |
| AI Evals (LangWatch) | [AI Evals LangWatch Guide](ai_evals_complete_guide_langwatch_langfuse.md) | QA / Eval Eng path |
| Interview prep | [00-interview-prep](00-interview-prep/) | All roles |
| Courses | [COURSES.md](COURSES.md) | All roles |

---

## 📚 Recommended Starter Courses by Role

> Full details in [COURSES.md](COURSES.md)

| Your Role | First Course | Second Course | Third Course |
|-----------|-------------|---------------|--------------|
| **Backend** | ChatGPT Prompt Engineering for Devs (DL.AI, free) | Building & Evaluating RAG (DL.AI, free) | AI Agents in LangGraph (DL.AI, free) |
| **Frontend** | ChatGPT Prompt Engineering for Devs (DL.AI, free) | Building Systems with ChatGPT API (DL.AI, free) | Evaluating & Debugging GenAI (DL.AI + W&B, free) |
| **QA** | AI Evals Guide in this repo (free) | Quality & Safety for LLM Apps (DL.AI, free) | AI Evals For Engineers & PMs (Maven, Hamel + Shreya, paid) |
| **PM** | AI for Everyone (Coursera, free) | AI Evals Guide Chapter 3 (free) | AI Evals For Engineers & PMs (Maven, Hamel + Shreya, paid) |
| **DevOps** | Efficiently Serving LLMs (DL.AI, free) | Evaluating & Debugging GenAI (DL.AI + W&B, free) | Machine Learning in Production (Coursera) |
| **Data Eng** | Building & Evaluating RAG (DL.AI, free) | Finetuning LLMs (DL.AI, free) | AI Evals Guide in this repo (free) |
| **EM** | Generative AI with LLMs (Coursera) | AI Agents in LangGraph (DL.AI, free) | CS294 LLM Agents (Berkeley, free) |

*DL.AI = DeepLearning.AI*

---

## Common Mistakes to Avoid

1. **Skipping fundamentals.** Jumping to LangChain before understanding what an embedding is leads to cargo-cult code you can't debug.

2. **Building before evaluating.** Ship nothing without a way to measure quality. Define your eval criteria before writing the first prompt.

3. **Copying prompts without understanding them.** Prompts are engineering decisions. Understand why each element is there.

4. **Ignoring costs until it's too late.** Every API call has a price. Build cost tracking from day one. See [02-model-landscape/03-pricing-and-costs.md](02-model-landscape/03-pricing-and-costs.md).

5. **Assuming the model is the bottleneck.** In most production AI systems, the bottleneck is retrieval quality, prompt design, or data quality. The model is rarely the problem.

6. **Using "latest" in model version strings in production.** Pin exact versions, and pin reasoning effort too. Silent model updates will break your product: Claude Opus 5.5 changed its default effort from `high` to `medium`, and coding tools such as Codex CLI and Claude Code swapped their default models with ordinary releases in September 2026.

7. **Over-agenting.** Starting with a 5-agent system when a single well-prompted call would work. Start simple, add complexity only when needed.

---

## The 2026 Market: Why Experience Is Your Advantage

The entry-level door into AI-exposed work has narrowed sharply since 2022, and by 2026 the evidence comes from large postings and administrative datasets, not anecdotes:

- **Indeed Hiring Lab (September 17, 2026):** in the most AI-exposed occupations, the entry-level share of job postings fell from 29% in 2021 to 10% in 2026, while the senior share rose from 22% to 47%. Most of the apparent AI pay premium is that shift in seniority mix.
- **US Census Bureau (working paper CES-WP-26-56, September 2026):** after late 2022, initial earnings for graduates of the most AI-exposed majors, led by computer science, fell about 13% relative to the least-exposed majors, an effect the authors compare to graduating into a large recession.
- **Federal Reserve Bank of Dallas (September 22, 2026):** each 10-point increase in a major's share of automatable tasks was associated with a 1.7-point lower chance of employment in Texas within a year of graduating.

For a career switcher this cuts in your favor. Hiring has tilted toward people who already know how production systems fail, so **move laterally into an AI role at your current level instead of restarting as a junior**: lead with the systems you have shipped and add the AI-specific skills on top.

The breakout hybrid role of 2026 is the **Forward Deployed Engineer (FDE)**, an engineer embedded with customers to take an AI system from use case through integration, evals, and security review to handover. It now comes in two flavors: frontier-lab FDE roles (OpenAI total compensation roughly $350K to $550K, per an outside estimate) and consulting FDE roles that BCG X hires straight from campus (a $110K to $190K base, per secondary reporting, unconfirmed). Anthropic's Claude Frontier Academy, announced October 2, 2026, plans to train 10,000 "Frontier Deployed Engineers" at customers and partners by the end of 2027, by nomination through Anthropic's account and partner teams. It is a credential program, not a hiring pipeline, but expect the title on more job descriptions. Backend engineers with customer-facing experience and solutions engineers are the natural fit. See [Job Market Trends](00-interview-prep/06-job-market-trends-2026.md) for the full role taxonomy.

---

## How to Get Hired

**Build in public.** The AI engineering job market rewards demonstrated work:

1. **GitHub portfolio**: One polished end-to-end project beats 10 toy projects
2. **Write a blog post**: Describing one real problem you solved and how (error analysis, eval pipeline, RAG latency fix)
3. **Contribute to open source**: OpenHands, LlamaIndex, DSPy, RAGAS. Even documentation PRs get you noticed.
4. **Use this repo's interview prep**: [00-interview-prep/01-question-bank.md](00-interview-prep/01-question-bank.md) has 155 questions with strong answers
5. **Get a mock interview or a mentor**: a career switch goes faster with someone who has sat on the other side of the table. Om offers 1:1 mock AI system design interviews and transition mentorship on [EngineBogie](https://enginebogie.com/u/ombharatiya) and [Topmate](https://topmate.io/ombharatiya)

**What to say in interviews:**
- Name specific decisions: "I chose Qdrant over Pinecone because of X" (not "I built a RAG system")
- Cite failure modes you've encountered and how you fixed them
- Know at least one benchmark by heart (Terminal-Bench 4.0 or SWE-Bench Pro's private set, RAGAS scores, TTFT for your serving setup), and say which effort level and harness produced the number
- Show you think about eval and cost, not just features

**Check the AI-tool policy for every round, in writing.** Policies split: Meta runs an AI-assisted coding round, Anthropic's candidate guidance rules out AI assistance in live interviews and take-homes unless it says otherwise, and Google is piloting Gemini in one round for some junior to mid-level roles. Some companies are also moving final rounds back on site. See [AI tools in the loop](00-interview-prep/06-job-market-trends-2026.md#ai-tools-in-the-loop-check-the-policy-per-round).

---

*Part of the [AI System Design Guide](README.md), maintained by [ombharatiya](https://github.com/ombharatiya)*
