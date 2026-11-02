# AI Job Market Trends - November 2026

> **Updated November 2, 2026:** the posting and layoff signals (Indeed Hiring Lab's October 22 snapshot and Microsoft's October 26 exit offer), the compensation rows marked "Nov 2026" (Anthropic, OpenAI, and US AI Engineer pages on levels.fyi, updated November 1, 2026), the Anthropic S-1, Anthropic's interview-format and AI-policy entries, and Anthropic's embedded-evaluator program. **Last checked October 2, 2026:** the entry-level studies, the Indeed metro-exposure and data-center pay figures, the other companies' AI-in-interview policies and candidate-reported prompts, the FDE programs, H-1B status, and the compensation rows marked "Oct 2026" or "Sep 2026" (Cursor, Google, Microsoft, OpenAI FDE, BCG X). **Last checked May 17, 2026:** every compensation row marked "May 2026" (national bands, specialty roles, London, Berlin, Bangalore, Singapore, Sierra, Thinking Machines), the role taxonomy apart from the FDE and evaluator entries, the skills lists, and the tech-stack percentages, all from the May 17 sweep of 100+ public job listings, hiring reports, and recruiter signals. Treat those comp rows as floors and the percentages as directional.

> **November 2026 update:** Four signals since early October. First, AI demand reached a record share of a weak market: AI-related postings hit 7.3% of all US postings in Indeed Hiring Lab's October 22 snapshot (a keyword count covering ML, data science, GenAI, and ChatGPT; the previous peak was 3.3% in 2022), while Indeed's software development postings index sat at about 79 against a February 2020 baseline of 100. AI work is growing inside a weak software market, not lifting it. Second, Anthropic filed its S-1 publicly on October 26, so Anthropic equity in an offer is now pre-IPO stock with lock-up agreements attached (see [Compensation insight](#compensation-insight)). Third, Microsoft opened a Voluntary Exit Offer on October 26 for eligible employees at level 70 and below in the US, UK, Canada, and India, without stating amounts or an AI rationale. Fourth, interview formats moved at the team level, not the company level: an Anthropic hiring manager wrote on October 27 that he had dropped the traditional coding interview for his team in favor of portfolio prototypes, which is one manager's practice and not company policy (see [Frontier-lab specifics](#frontier-lab-specifics)). On levels.fyi, OpenAI's software engineer median fell from $710K to $640K between October 2 and November 1 as its sample grew, a reminder of how far small self-reported samples move.
>
> **October 2026 update:** Four shifts since August. First, the entry-level squeeze is now measured in pay and postings, not unemployment: the entry-level share of postings in the most AI-exposed occupations fell from 29% to 10% (Indeed Hiring Lab), and starting earnings for graduates of the most exposed majors fell about 13% relative to the least-exposed majors (US Census Bureau), yet recent-graduate unemployment has not spiked (NBER). Second, AI-attributed layoffs cooled to 9% of September's cuts, while capex-funded and management-layer cuts continued (see [the first headline shift](#1-the-market-is-paradoxically-hot-and-cold)). Third, AI-in-interview policy split instead of standardizing: Meta runs an AI-enabled round, Google is piloting one, Anthropic bars AI unless told otherwise, and some firms are moving final rounds back in person (see [AI tools in the loop](#ai-tools-in-the-loop-check-the-policy-per-round)). Fourth, FDE is being credentialed and pushed out of the labs: OpenAI's Deployment Company, BCG X's campus FDE roles, and Anthropic's Claude Frontier Academy, announced October 2 (see [FDE](#forward-deployed-engineer-fde)).
>
> **August 2026 update:** Three datapoints worth carrying into interviews and career planning. First, the entry-level squeeze now has administrative payroll evidence behind it: a widely cited Stanford Digital Economy Lab paper revised in August 2026 with ADP data through June reports that workers aged 22-25 in AI-exposed occupations sit about **19% below** the employment level they would have reached had they kept pace with less-exposed peers, while experienced workers show no comparable gap. The mechanism the authors identify is reduced hiring rather than increased separations, which matters: the entry door is narrower, but the people already inside are not being pushed out. Second, the infrastructure buildout is now the clearest wage story in the market. Indeed Hiring Lab (July 14, 2026) puts data-center roles at **six of every 1,000 US job postings, up from two per 1,000 in May 2023**, and reports that data-center installation and maintenance work pays about **42% more per hour** than equivalent non-data-center roles. If you are technical but not an ML researcher, that is where the leverage is. Third, capability-tiered access programs at the frontier labs (Daybreak Blue and Red, Project Glasswing) have created a small but real category of roles that require verified identity and security clearance-adjacent vetting, which is a new consideration in where you apply.
>
> **June 2026 update:** Two market-moving events since the May sweep. Anthropic released **Claude Fable 5** (June 9, $10/$50 per 1M), bringing Mythos-class capability to general availability with an Opus 4.8 fallback safeguard; expect a wave of capability-ceiling product work and the eval, safety, and routing roles that come with it. DeepSeek made its **75% V4 Pro discount permanent** (May 22), accelerating the cost-engineering hiring trend: candidates who can exploit a 70x price spread across model tiers are screening well. The compensation rows at the time were all from the May 17 sweep; treat any row still marked "May 2026" as a floor in the segments those events touch.

This chapter is for engineers planning their next move, hiring managers building rubrics, and engineering leaders making organizational design decisions. It complements [TRANSITION_GUIDE.md](../TRANSITION_GUIDE.md) (how to transition into AI roles) and the [Question Bank](01-question-bank.md) (what to study).

---

## Table of Contents

- [The Three Headline Shifts](#the-three-headline-shifts)
- [Role Taxonomy in 2026](#role-taxonomy-in-2026)
- [Skills by Career Level](#skills-by-career-level)
- [What Job Listings Actually Require](#what-job-listings-actually-require)
- [Compensation Reality](#compensation-reality)
- [Geographic & Industry Distribution](#geographic--industry-distribution)
- [Interview Process Patterns](#interview-process-patterns)
- [Emerging Roles to Watch](#emerging-roles-to-watch)
- [Strategic Takeaways](#strategic-takeaways)
- [References](#references)

---

## The Three Headline Shifts

If you read nothing else, internalize these three things.

### 1. The market is paradoxically hot and cold.

Q1 2026 saw ~52,050 tech layoffs (Oracle 30K, Amazon, Meta, Dell), the highest Q1 since 2023 ([Kore1](https://www.kore1.com/tech-layoffs-2026/); [Tom's Hardware](https://www.tomshardware.com/tech-industry/tech-industry-lays-off-nearly-80-000-employees-in-the-first-quarter-of-2026-almost-50-percent-of-affected-positions-cut-due-to-ai)). Simultaneously, AI roles grew +8.9% QoQ and +4.8% YoY, with ~275,000 unfilled AI roles ([Allwork.space](https://allwork.space/2026/05/ai-hiring-is-rising-even-as-tech-layoffs-surge-140/)). Junior/entry-level engineers were hit hardest - routine codegen, QA testing, basic frontend work disproportionately cut. Senior + specialist AI roles remained resilient.

**By Q3 the picture sharpened.** AI-attributed cuts cooled. Challenger, Gray & Christmas counted 3,462 AI-cited cuts in August, the lowest monthly total since December 2025, and 3,961 in September (9% of the month's 43,281). AI is still the top reason year to date: 120,136 cuts, about 21% of all announced cuts. Tech remains the hardest-hit sector, with 165,925 cuts through September (29% of the total). Much of the cutting is not attributed to AI at all. Uber cut about 3,300 roles (roughly 10%) on September 2 to remove management layers and did not cite AI. Oracle began another round on September 14 while its fiscal 2026 capex reached $55.7B, up from $21.2B: AI infrastructure spending and engineering layoffs are often the same budget decision. October added a large program framed as voluntary: Microsoft's Voluntary Exit Offer (October 26) covers eligible employees at level 70 and below, in participating teams, in the US, UK, Canada, and India; each organization decides whether to offer it, and Microsoft published no amounts, end date, or AI rationale.

The entry-level evidence also firmed up, and it points to hiring and pay rather than unemployment:

- **Indeed Hiring Lab (September 17, 2026):** in the most AI-exposed occupations, the entry-level share of postings fell from 29% in 2021 to 10% in 2026, while the senior share rose from 22% to 47%. Advertised pay grew faster in exposed occupations, but most of that "AI premium" disappears once seniority mix is held constant.
- **US Census Bureau (working paper CES-WP-26-56, September 2026):** after late 2022, starting earnings for graduates of the most AI-exposed majors, led by computer science, fell about 13% relative to graduates of the least-exposed majors, which the authors compare to graduating into a large recession. About half the drop comes from graduates landing in lower-paying sectors, and the gap narrows to about 5% after two years.
- **Federal Reserve Bank of Dallas (September 22, 2026):** each 10-point rise in a major's share of automatable tasks is associated with a 1.7-point lower chance of employment in Texas within a year of graduating, and more graduates retreating to grad school.
- **NBER working paper w35796 (September 2026):** unemployment for bachelor's holders aged 22-25 was 7.3% in summer 2026, inside the range of the previous four summers, with no significant rise relative to older graduates.

Forecasters are split on software specifically: Indeed's Q3 Labor Market Outlook panel (123 panelists, surveyed September 8-16, 2026) put software development on both the list of sectors with the largest expected AI-driven job losses and the list with the largest expected gains.

**Implication:** "Tech hiring" and "AI hiring" are not the same story in 2026, and neither are "layoffs" and "AI layoffs". If your career is generalist mid-level SWE work, you're feeling the cold market. If you're AI-specialized at senior level, you're in a sellers' market. If you're entering the field, the door is narrower, and the penalty shows up as fewer junior postings and lower starting pay rather than as unemployment.

### 2. The title is collapsing; the work is fragmenting.

Most companies now post "AI Engineer" as the umbrella, but inside the role you specialize quickly into RAG, agents, evals, fine-tuning, or platform work. ["Most AI job titles will collapse into 'AI Engineer' over the next 18 months; prestige labels survive only at frontier labs"](https://www.ivanturkovic.com/2026/04/24/ai-job-titles-2026-naming-chaos/). The "Prompt Engineer" standalone title has effectively disappeared from major job boards - the skill survived; the title didn't ([PE Collective](https://pecollective.com/blog/is-prompt-engineering-a-real-career/); [Medium - Prompt Engineering Is Dead 2026](https://medium.com/write-a-catalyst/prompt-engineering-is-dead-2026-ai-systems-engineering-7acdbbcb2160)).

**Implication:** If you're hiring a "Prompt Engineer," you're 18 months behind. Define the actual problem (eval rigor? agent debugging? customer-facing tuning?) and hire for that specific role.

### 3. Forward Deployed Engineer is the breakout role of 2026.

FDE didn't exist as a discrete category at frontier labs in mid-2025. By May 2026, OpenAI, Anthropic, and Google were all hiring hundreds. Google/Box CEOs publicly called it "the most in-demand job in tech" ([Fast Company](https://www.fastcompany.com/91541878/google-box-ceos-say-this-is-the-most-in-demand-job-in-tech); [Hashnode FDE guide](https://hashnode.com/blog/a-complete-2026-guide-to-the-forward-deployed-engineer)). TC at frontier labs held at $350-550K mid-to-senior (Aced's September 2026 estimate for OpenAI FDEs). By October the role had spread beyond the labs: OpenAI stood up a separate Deployment Company, BCG X began hiring FDEs straight from campus, and Anthropic announced a program to train 10,000 "Frontier Deployed Engineers" inside customers and partners by the end of 2027 (details under [Emerging Roles](#forward-deployed-engineer-fde)).

**Implication:** Frontier-AI buyers (Fortune 500, government, biotech) demand on-site engineering presence as a contractual deliverable. FDE is the role that exists because the buyer values it - not because it's the most efficient way to deliver software.

---

## Role Taxonomy in 2026

### Established titles (still hiring strongly)

| Title | Description | Where it's posted |
|-------|-------------|-------------------|
| **AI Engineer** | The de facto general-purpose AI title. Other titles are collapsing into it. | Universal - most postings |
| **LLM Engineer** | Centered on transformer fine-tuning, RAG, agents. Distinct from ML Engineer. | Mid-large companies; [iSmart LLM JD 2026](https://www.ismartrecruit.com/job-descriptions/llm-engineer) |
| **ML Engineer / ML+AI Software Engineer** | Classic training-and-deployment role. | [levels.fyi ML/AI focus](https://www.levels.fyi/t/software-engineer/focus/ml-ai) |
| **Applied AI Engineer** | Customer-embedded variant at frontier labs. | [Anthropic Applied AI](https://job-boards.greenhouse.io/anthropic/jobs/5116274008) |
| **Member of Technical Staff (MTS)** | Deliberately ambiguous title that blurs research vs engineering. | OpenAI, Anthropic, Thinking Machines, Mistral ([Scout AI on MTS](https://scoutnow.ai/blog/rebirth-member-of-technical-staff)) |
| **AI Research Engineer / Research Scientist** | Frontier labs only; PhD-preferred. | [Sundeep Teki - AI Research Eng 2026](https://www.sundeepteki.org/advice/the-ultimate-ai-research-engineer-interview-guide-cracking-openai-anthropic-google-deepmind-top-ai-labs) |
| **AI Solutions Architect** | Heavy in enterprise. | EY, Caterpillar, Deloitte ([EY listing](https://careers.ey.com/ey/job/Amsterdam-AI-Solution-Architect-1083-HP/1258705801/)) |
| **AI Platform Engineer** | Owns the internal LLM-ops platform. | [Augment Code spec 2026](https://www.augmentcode.com/guides/ai-platform-engineering-leader-job-spec) |
| **AI Engineering Manager** | Highest-paying single role; median $293.5K ([AI Pulse benchmarks](https://theaimarketpulse.com/salaries/)). | Universal at scale-ups+ |
| **AI Product Manager** | Required for nearly every B2B SaaS. | Universal ([Aakash Gupta](https://www.aakashg.com/product-manager-requirements/)) |
| **AI Technical Program Manager (TPM)** | Specializations: "Responsible AI TPM," "AI Infrastructure TPM," "GenAI Customer Performance TPM" | Microsoft, AMD, Together AI |

### NEW titles since 2025

| Title | Why it emerged | Where it's posted |
|-------|----------------|-------------------|
| **Forward Deployed Engineer (FDE)** | Frontier-AI buyers demand on-site engineering as a deliverable. | OpenAI (including the OpenAI Deployment Company), Anthropic, Google ([Anthropic FDE](https://job-boards.greenhouse.io/anthropic/jobs/4985877008)); consultancies hiring from campus (BCG X "Forward Deployed AI Engineer"); "Frontier Deployed Engineer" is a new variant at Anthropic customers and partners |
| **AI Evaluation Engineer** | Eval work matured into a discrete discipline. | OpenAI ([Applied Evals](https://openai.com/careers/software-engineer-applied-evals-san-francisco/), [Frontier Evals](https://openai.com/careers/research-engineer-frontier-evals-and-environments-san-francisco/)), Apple, Scale AI, Distyl, Apex; embedded evaluators at Faculty (Accenture) |
| **Agentic Systems Engineer / AI Agent Engineer** | Agents became their own engineering surface. | Teradata, GE Vernova, Deloitte, OpenAI ([Agent Infrastructure SWE](https://openai.com/careers/software-engineer-agent-infrastructure-san-francisco/)) |
| **AI Reliability Engineer** | Production AI needs SRE-like discipline; distinct from traditional SRE. | Anthropic ([Staff/Sr AI Reliability](https://www.anthropic.com/jobs)); AI SRE as a category being defined by Resolve.ai, Rootly. |
| **AI Security Engineer / LLM Red Team Specialist** | Prompt-injection defense and jailbreak research as a discipline. | Life360 ([Principal AI Security Engineer](https://www.remoterocketship.com/us/company/life360/jobs/principal-ai-security-engineer-ai-native-platform-united-states-remote/)); 10 emerging AI security roles enumerated by [Practical DevSecOps](https://www.practical-devsecops.com/emerging-ai-security-roles/). |
| **MCP Engineer / MCP Software Engineer** | MCP adoption made server development its own specialty. | Descope ([MCP SWE](https://careers.descope.com/p/fe57f6224769-mcp-model-context-protocol-software-engineer)) |
| **AI Operator / Computer-Use Specialist** | Tied to computer-use and persistent agents such as ChatGPT agent, Claude's Cowork (being merged into the main Claude app since September 16, 2026), and Microsoft Copilot Autopilot. | $75-120K specialist tier ([Coasty](https://coasty.ai/blog/best-computer-use-platform-2026-20260402)) |

### Roles disappearing or consolidating

- **Prompt Engineer (standalone):** Title is dying. Skill remains as table stakes.
- **Distillation Engineer:** Appears as a *responsibility* in fine-tuning/inference engineer postings, not its own widely-posted req.

---

## Skills by Career Level

### L4-L5 (Mid-level IC, 3-5 yrs)

- Python production proficiency - **71% of AI job postings** ([Second Talent](https://www.secondtalent.com/resources/most-in-demand-ai-engineering-skills-and-salary-ranges/))
- Hands-on with at least one major LLM provider SDK (OpenAI, Anthropic, Bedrock) and one orchestration framework - most commonly LangChain/LangGraph (34.3% of agentic AI postings; [Agentic Engineering Jobs](https://agentic-engineering-jobs.com/langchain-job-market-2026))
- Vector DB fundamentals: Pinecone, Weaviate, pgvector - tool-specific experience is learnable in weeks; conceptual understanding matters most
- RAG: chunking, hybrid search, BM25, reranking, retrieval evals
- Containerization: Docker (15.4%), Kubernetes (17.6%)
- Cloud: AWS (32.9%), Azure (26%)

### L6-L7 (Senior / Staff)

- Production LLM systems shipped end-to-end - "industry experience shipping real systems is a better signal than an academic credential"
- Multi-tenant isolation across vector indexes, GPU memory, agent state
- Eval frameworks (LangSmith / Langfuse / Braintrust); eval-gated CI/CD
- Fine-tuning / LoRA / QLoRA / RLHF
- Cost optimization - token budgets, model routing, caching
- "Reason about LLMs, vector stores, and RAG as part of standard system design, not as a niche specialty" ([Design Gurus](https://designgurus.substack.com/p/system-design-interviews-changed))

### L8+ (Principal / Leadership IC)

- Own agent orchestration layers, model-routing, LLMOps platforms serving all eng teams
- Runtime governance for non-deterministic systems
- Architect for SOC 2 / HIPAA / EU AI Act compliance - trigger DPIA + FRIA under AI Act Article 27 (the Annex III high-risk obligations behind FRIA now start December 2, 2027; Article 50 transparency duties have applied since August 2, 2026)
- "Define technical vision and scale engineering teams matters more than coding prowess alone"

### Manager track (EM / Director)

- AI Engineering Manager median $293.5K - highest-paying single role ([AI Pulse](https://theaimarketpulse.com/salaries/))
- Hiring rubrics now weight: "can you put this person in a room with a PM and a junior eng and have them drive technical direction without making a mess" - 5 of 7 hiring managers surveyed ([Design Gurus](https://designgurus.substack.com/p/system-design-interviews-changed))
- Mission alignment and safety judgment heavily weighted at frontier labs ([Anthropic EM guide](https://www.gethireready.com/interview-guides/engineering-manager-anthropic))

### PM track (AI PM / AI TPM)

- "AI is the new baseline, not a bonus skill"
- 4+ yrs PM, ideally B2B SaaS or AI-driven product
- Critical: "fewer than 1 in 4 senior AI PM candidates meet the bar for technical fluency + product rigor" ([Aakash Gupta](https://www.aakashg.com/product-manager-requirements/))
- "Candidates who can show a working prototype outperform those who can only describe one"

---

## What Job Listings Actually Require

### Must-Have (called out as required across 100+ postings)

- Python production code, 3+ yrs
- LLM API integration (OpenAI / Anthropic / Bedrock)
- RAG pipeline experience including vector DB, chunking, retrieval evals
- Production-grade observability and eval pipelines
- Cloud + Kubernetes + IaC
- Agent debugging / multi-step workflow tracing
- Prompt injection / jailbreak defense for security-sensitive roles

### Nice-to-Have (explicitly listed as "plus" or "bonus")

- Publications or OSS contributions; working portfolio of 3-5 projects beats a paper for applied roles
- CUDA / GPU-level optimization - must-have at NVIDIA/frontier labs, nice-to-have elsewhere
- Distillation / model compression
- Distributed inference experience
- Java/C++ for legacy enterprise integration
- Reinforcement learning beyond RLHF

### Top Tech Stack in Listings (May 2026)

Ranked by frequency:

1. **Python** - 71% of all AI postings
2. **PyTorch / JAX** - universal at frontier labs
3. **LangChain / LangGraph** - 34.3% of agentic postings, #1 framework
4. **LlamaIndex** - co-occurs in 38% of LangChain listings
5. **AWS (32.9%) / Azure (26%) / GCP / Vertex / Bedrock**
6. **Kubernetes (17.6%) + Docker (15.4%)**
7. **Vector DBs:** Pinecone, Weaviate, Qdrant, Chroma, pgvector
8. **MCP (Model Context Protocol)** - now ["a fundamental requirement"](https://medium.com/@adnanmasood/the-rise-of-model-context-protocol-mcp-skills-5f0d6a1c3579) at cutting-edge teams
9. **Observability:** LangSmith, Langfuse, Braintrust, Arize
10. **Inference engines:** vLLM, SGLang, TensorRT-LLM
11. **Terraform / Helm / Ray / Kubeflow / MLflow / Feast** - internal platform stack
12. **Provider SDKs:** OpenAI Agents SDK, Claude SDK, Vercel AI SDK, Mastra, Pydantic AI

### By Company Tier

- **Frontier labs** (Anthropic, OpenAI, SpaceXAI/xAI): PyTorch/JAX, vLLM/custom inference, internal evals, MCP servers, CUDA/GPU-level optimization
- **Scale-ups** (Cursor, Harvey, Sierra, Decagon, Glean, Perplexity): TypeScript + Python mix, LangGraph / OpenAI Agents SDK, Pinecone/pgvector, LangSmith/Braintrust evals
- **Enterprises** (Deloitte, EY, Caterpillar, Citi): Azure-heavy, Bedrock, LangChain, governance/MLOps focus, on-prem capability

### Non-Technical Requirements

- **Communication / cross-functional collaboration** - table stakes at senior+
- **Customer-facing skills** - load-bearing for FDE roles; Anthropic requires 3+ yrs in "technical, customer-facing role"
- **OSS contributions** - Anthropic states explicitly: ["if you have done interesting independent research, written an insightful blog post, or made substantial contributions to open-source software, put that at the TOP of your resume"](https://www.sundeepteki.org/advice/how-to-get-hired-at-openai-anthropic-and-google-deepmind-in-2026)
- **Publications** - required for AI Research Engineer roles; applied roles weight shipped systems and open-source work over a PhD
- **Mission alignment** - Anthropic explicitly screens for it via a Behavioral and Values round
- **Regulatory experience:** SOC 2 / HIPAA / FedRAMP for enterprise; EU AI Act familiarity (FRIA/DPIA) for EU operations
- **Security clearance** - required for Lockheed and federal-adjacent roles

---

## Compensation Reality

> Public-source ranges only. Verify with [levels.fyi](https://www.levels.fyi/) for current data. All figures USD unless noted. The **Sweep** column says when each row was last checked: "Nov 2026" rows come from levels.fyi pages updated November 1, 2026, and "Oct 2026" rows from levels.fyi pages captured October 2, 2026 (both self-reported, verified against offer letters where available; n is the submission count); "Sep 2026" rows come from the job-board and prep-vendor sources named in the row, and "May 2026" rows have not been re-run since the May 17 sweep.

| Tier / Company | Level | Total Comp | Sweep |
|---|---|---|---|
| **Anthropic** | All SWE | Median $882.5K TC (n=49) | Nov 2026 |
| **Anthropic** | Senior SWE | $621K TC (per-level base no longer shown) | Nov 2026 |
| **Anthropic** | Lead SWE | $862K TC | Nov 2026 |
| **Anthropic** | Staff SWE | $1.41M TC | Nov 2026 |
| **OpenAI** | All SWE | $249K (L2) to $1.89M (L7) TC; median $640K (n=248). The same page lists L6 at $1.94M, above L7, a sign of thin per-level samples | Nov 2026 |
| **OpenAI** | L5 SWE | $934K TC | Nov 2026 |
| **OpenAI MTS / Research Scientist** | - | $245K – $685K base (federal filing) | May 2026 |
| **OpenAI FDE** | Mid-senior | $153K-$325K base; est. $350K-$550K TC (Aced estimate) | Sep 2026 |
| **OpenAI Deployment Company** | Founding FDE | Up to $400K (listing via a16z jobs; up to 50% travel) | Sep 2026 |
| **Cursor (now part of SpaceX)** | SWE | $188K to $1.665M TC; median $1.0M (n=22) | Oct 2026 |
| **Sierra** | SWE | $200K – $460K TC; median $450K | May 2026 |
| **Thinking Machines Lab** | All eng | $450K – $500K base (Q1 H-1B filings) | May 2026 |
| Google AI Engineer | L3-L6 | $180K (L3) to $1.15M (L6) TC; median $260K (n=40) | Oct 2026 |
| Microsoft AI Engineer | All | Median $282K (n=9); May range $238K – $355K+ TC | Oct 2026 |
| **US AI Engineer (levels.fyi)** | All | Median $156K (p25 $110K, p75 $212K, p90 $282K; n=339) | Nov 2026 |
| **US National AI Engineer** | Entry (0-2y) | $90-135K base / $110-160K TC | May 2026 |
| **US National AI Engineer** | Mid (3-5y) | $140-210K base / $170-260K TC | May 2026 |
| **US National AI Engineer** | Senior (6-9y) | $180-280K base / $220-350K+ TC | May 2026 |
| **US National AI Engineer** | Staff/Principal (10+y) | $250-400K+ base / $350-600K+ TC | May 2026 |
| BCG X Forward Deployed AI Engineer | Campus hire | $110K-$190K base (secondary reporting, unconfirmed; the same band appears on BCG X's experienced-hire FDE posting) | Sep 2026 |
| RAG Engineer Senior | - | $195-290K base; $400K+ TC at frontier | May 2026 |
| LLM Fine-Tuning Specialist | - | $195K-$350K | May 2026 |
| AI Security Engineer | - | $152-210K | May 2026 |
| LLM Red Team Specialist | - | $160-230K | May 2026 |
| **AI Engineering Manager** | - | $293.5K median (highest-paying single role) | May 2026 |
| AI Product Manager | - | $141K – $250K (median $159K) | May 2026 |
| **Agentic AI Architect** | - | $260K – $420K base | May 2026 |
| AI Evaluation Engineer | - | Too few public postings for a stable range; companies level it against their senior SWE band. Use the US National Senior row as the anchor. | May 2026 |
| MCP / Integrations Engineer | - | New title with thin public data; typically leveled as senior platform engineering. Anchor to the senior SWE band. | May 2026 |
| London (Quant fund / FAANG) | Senior ML | £140-180K base; £200K+ TC | May 2026 |
| London (Google DeepMind) | Senior | £110-155K base + RSU | May 2026 |
| Berlin / Germany | Senior | €95-130K | May 2026 |
| **Bangalore (Top GCC / AI-first)** | Senior | ₹1-2 Cr TC | May 2026 |
| Bangalore (Fresh PhD / top MS) | Entry | ₹22-32 LPA | May 2026 |
| Singapore | Avg | S$221,200 | May 2026 |
| Singapore (Principal/Lead) | 10+y | S$323,505 | May 2026 |

**Sources:** [levels.fyi Anthropic](https://www.levels.fyi/companies/anthropic/salaries/software-engineer), [Anthropic S-1 on EDGAR](https://www.sec.gov/Archives/edgar/data/2130357/000162828026067168/), [OpenAI](https://www.levels.fyi/companies/openai/salaries/software-engineer), [Cursor](https://www.levels.fyi/companies/cursor/salaries/software-engineer), [Sierra](https://www.levels.fyi/companies/sierra/salaries/software-engineer), [Aced OpenAI FDE](https://www.aced.io/jobs/fde/openai), [a16z jobs on the Deployment Company](https://a16zjobs.substack.com/p/openais-new-deployment-arm-is-hiring), [Management Consulted on BCG X FDE](https://managementconsulted.com/bcg-forward-deployed-ai-engineer/), [Pin AI Comp Guide 2026](https://www.pin.com/blog/ai-compensation-salary-guide/), [Kore1 salary guide](https://www.kore1.com/ai-engineer-salary-guide/), [AI Pulse benchmarks](https://theaimarketpulse.com/salaries/), [Career Check London 2026](https://www.careercheck.io/blog/ml-engineer-salary-london-2026), [Zen van Riel Europe](https://zenvanriel.com/job/ai-engineer-salary-europe/), [Scaler India](https://www.scaler.com/topics/ai-ml-engineer-salary-complete-guide/).

### Compensation insight

The gap between frontier-lab SWE medians ($882.5K at Anthropic and $640K at OpenAI on levels.fyi, November 1, 2026) and the US AI Engineer median ($156K) is roughly **4.1-5.7x**. Choose your company tier with eyes open.

Three cautions before you anchor a negotiation on these rows:

- **Small, self-reported samples move.** Between October 2 and November 1, OpenAI's median fell from $710K to $640K as its sample grew from 239 to 248, its L5 now reads $934K (against $1.15M in the May sweep), and its page lists L6 above L7. Anthropic's median held near $883K on fewer submissions (55 down to 49). Cursor (n=22) and Microsoft AI Engineer (n=9) are too thin to quote as market rates.
- **FDE pay depends on who employs you.** Frontier-lab FDEs sit at $350K-$550K estimated TC; consulting FDEs hired from campus reportedly start on a $110K-$190K base. Same title, roughly a third of the pay.
- **Equity is harder to value around a listing or a deal.** Anthropic filed its S-1 publicly on October 26, 2026, with no share count or price range set yet and with lock-up agreements described. The filing puts stock-based compensation at $2.2B in the first half of 2026 and headcount at about 5,300 on September 30, 2026, up from about 940 at the end of 2024. For an offer that includes Anthropic equity, ask how long the lock-up runs, when RSUs settle and get taxed after listing, and what the refresh policy is once the stock is public. Outright acquisitions keep landing: Cursor became part of SpaceX in August 2026 (reported at about $60B in SpaceX stock), and NVIDIA confirmed on September 3 that it is buying Hugging Face for $12.93B, with no closing date disclosed. License-and-hire deals, a common exit for startup teams, are still being struck (NVIDIA's reported roughly $6B licensing deal with Poolside includes offers to about 109 of its employees) but now carry antitrust risk: the DOJ is reportedly probing NVIDIA's Groq deal (reported at about $17-20B; terms undisclosed). Ask how unvested equity converts in an acquisition and what happens to employees who are not part of the hire.

---

## Geographic & Industry Distribution

- **The hubs are also the most exposed markets:** Indeed Hiring Lab's metro-level GenAI exposure score (August 25, 2026; US average about 44) is highest in San Jose (59), Seattle (57), Washington DC (54), San Francisco (53), and Austin (52). Indeed's reading is that more local work could be reshaped, not that the jobs disappear.
- **Data-center build-out pays, with strings attached:** Indeed's follow-up study (August 13, 2026) finds premiums over equivalent non-data-center roles of 63% (about $50,000 a year) for facilities managers, about $25,000 at the median for network engineers, and 42% per hour for installation and maintenance work. The tradeoffs: 26% of data-center postings specify up to 50% travel, and data-center IT staff are nearly 40x as likely to work nights. Demand is still climbing: Indeed's data-center postings index reached 247 in its October 22 snapshot (June 2024 = 100), up 147% since June 2024, while tech postings outside data centers sat at 99.
- **H-1B is contested, not settled:** a September 18, 2026 proclamation renewed the $100,000 fee on new H-1B petitions for September 21, 2026 to September 21, 2027, but the fee has not been collected since a federal court vacated the implementing policy on June 8, and Forbes reported on October 1 that a second federal judge struck down the renewed version. Separately, the wage-weighted H-1B lottery (four entries at wage Level IV, one at Level I) took effect February 27, 2026, which favors senior offers and compounds the entry-level squeeze for international new grads.
- **Two-tier market:** in January 2026, Indeed Hiring Lab reported that about 95% of hiring firms had not posted an AI job, with adoption concentrated among the largest firms ([Indeed Hiring Lab Jan 2026](https://www.hiringlab.org/2026/01/16/ai-adoption-accelerating-still-concentrated-among-largest-firms/))
- **Enterprise adoption:** 72% of enterprises have at least one AI workload in production as of Q1 2026 ([Medha Cloud](https://medhacloud.com/blog/ai-adoption-statistics-2026))
- **Consulting boom:** BCG reports 25% of $14.4B 2025 revenue ($3.6B) was AI consulting ([Metaintro BCG](https://www.metaintro.com/blog/bcg-25-percent-ai-revenue-consulting-jobs-2026))
- **Remote-friendly:** LangChain ecosystem 35.2% remote, 48.4% hybrid, 16.4% strictly onsite
- **AI share of postings at a record:** AI-related postings reached 7.3% of all US postings in Indeed Hiring Lab's October 22, 2026 snapshot, past the previous peak of 3.3% in 2022 ([Indeed Hiring Lab, Oct 2026](https://hiringlab.indeed.com/2026/10/22/us-labor-market-snapshot-october-2026/)). The count is keyword-based (ML, data science, GenAI, ChatGPT), so it measures AI mentions, not AI-engineer headcount. The same snapshot puts software development postings at about 79 (February 2020 = 100), the lowest of the sectors shown, and posted wages up 2.4% year over year in September against 3.7% CPI inflation

---

## Interview Process Patterns

The May 2026 standard at AI-native companies:

1. **Recruiter screen** (30 min) - culture/mission + comp + visa
2. **Technical phone screen** (60-90 min) - practical coding, production-style
3. **Take-home** (48 hr - 3 day) - common at LangChain, Mistral, Eightfold; build a small RAG/agent system. ["Not a test of whether you can build, but how - code quality, evals, error handling"](https://github.com/alexeygrigorev/ai-engineering-field-guide/blob/main/interview/01-interview-process.md)
4. **Onsite/virtual loop** (4-6 hr): coding round + AI system design + project deep dive + behavioral. ["Whiteboard-only rounds are mostly gone, even Google's format is collaborative now"](https://designgurus.substack.com/p/system-design-interviews-changed)
5. **Hiring manager / values round** - explicit at Anthropic

### AI-role specifics

- **System design rounds** now expect LLM infra, GPU scheduling, vector stores, RAG, eval-gated CI, cost/latency tradeoffs
- **AI-assisted coding rounds** exist at some companies (Meta, Canva, Sierra, Cursor, and a Google pilot) but are not standard; see [the policy table below](#ai-tools-in-the-loop-check-the-policy-per-round). Where AI is allowed, the scored skill is output validation, not prompt cleverness
- **Take-home transparency:** where AI use is permitted, add an "AI audit note" - what you used AI for, what you changed, why. Transparency beats stealth
- **Sierra:** in-person-only at SF or NY offices; "Plan + Build + Present" 2-hour agent assessment with no algorithm rounds
- **Cursor:** 8-hour take-home using their own product with limited docs and a Slack channel - assesses product sense, autonomy, system design
- **Anthropic:** "the answer that sounds like it was written the night before is a bad signal"

### AI tools in the loop: check the policy per round

AI-in-interview policy split in 2026 instead of converging. Two trends run in parallel: some companies score how you work with an assistant, and others are moving rounds back in person to verify that you can work without one.

| Company | Policy (as publicly reported) | Source quality |
|---|---|---|
| **Meta** | AI-enabled coding round since October 2025: 60 minutes in CoderPad on a multi-file codebase with tests, progressing from fixing bugs to building a feature to optimizing. E6 and below get one traditional and one AI-assisted coding round; E7 and above get a single coding round, and it is AI-assisted | Prep-vendor and candidate reports; Meta has published no rubric |
| **Google** | Pilot for junior to mid-level roles on select US teams, started May 2026: Gemini allowed in a "code comprehension" round (read, debug, and optimize an existing codebase) from H2 2026; the Googleyness and Leadership round adds a technical design discussion of a past project | Business Insider (May 7, 2026), confirmed by Google's VP of recruiting; no pilot results published |
| **Anthropic** | Live interviews are "all you, no AI assistance unless we indicate otherwise"; take-homes without Claude unless indicated. Separately, one hiring manager dropped the traditional coding interview for his team in October 2026 (see below); that is a team practice, not a policy change | Anthropic's candidate AI guidance (last updated July 10, 2025, unchanged as of November 2, 2026) |
| **Canva, Sierra, Cursor** | AI tools expected in at least one round (Sierra and Cursor formats above) | Company posts and candidate reports |
| **Microsoft** | No public AI-assisted interview format found | - |

The counter-trend is in-person verification. Stanford economist Nick Bloom told Business Insider (July 31, 2026) that some tech firms are moving middle and final rounds back on site because AI undermines remote coding tests, and Greenhouse's 2025 AI in Hiring Report found 39% of US hiring managers running more in-person interviews to verify candidates.

**Prep implication:** ask the recruiter for the AI policy of each round, in writing. Using AI where it is barred ends the loop; refusing it where it is expected wastes the round. In AI-enabled rounds the work is verification: reading unfamiliar code, testing what the model produced, and explaining why you rejected a suggestion.

### Frontier-lab specifics

- **Anthropic:** the commonly reported coding round is a 90-minute, 4-level progressively harder problem testing whether you write clean modular code that absorbs new requirements. Expect variation by team: on October 27, 2026, Felix Rieseberg, a hiring manager on Claude Desktop and Cowork, wrote that he had "laid the traditional coding interview to rest" and now screens candidates on portfolios of working prototypes. That is one manager's practice, not company policy; headlines saying Anthropic killed the coding interview overstate it, so ask your recruiter which format your team uses. Values round explicit. Candidate-reported system design prompts (via the prep vendor Aced, formerly Exponent, September to October 2026) lean on infrastructure: an inference batching system for a single GPU handling up to 100 inputs per batch while users wait synchronously, a peer-to-peer system that moves a large file from a bandwidth-constrained source to thousands of machines, and a token-generation service at 100,000 RPS. For machine learning engineer roles Aced also lists "design an agentic AI system that can autonomously adapt to new tasks."
- **OpenAI:** "Design the OpenAI Playground" - wireframes + API + DB schema for thread/message history; multi-tenant secure cloud IDE. Also candidate-reported: a GPU credit management system. The FDE loop reportedly runs six stages, including system design under token-cost, latency, and eval-gate constraints and a role-play with a simulated non-technical executive (Aced, September 2026).
- **Mistral (Paris):** 5-round process, no remote, with a dedicated "LLM theory" stage covering transformer internals and alignment
- **Applied-AI companies:** candidate-reported prompts include an insurance-claims agent that uses RAG to output an approval decision while controlling token cost (Scale AI) and an agentic customer-support system for Spotify (Sierra). Model answers treat cost as a first-class constraint and measure retrieval separately from decision accuracy. All prompts in this section come from prep vendors and are not confirmed by the companies.

---

## Emerging Roles to Watch

These roles grew fastest through 2026 (first identified in the May sweep, with later changes noted through November 2, 2026) - bet on them if you're planning a 12-month career trajectory.

### Forward Deployed Engineer (FDE)
- **Why:** Frontier-AI buyers (Fortune 500, government, biotech) demand on-site engineering as a contractual deliverable
- **Comp:** $350-550K mid-to-senior at frontier labs (Aced's September 2026 estimate for OpenAI, base $153K-$325K); consulting FDEs hired from campus start far lower (BCG X: $110K-$190K base, per secondary reporting, unconfirmed)
- **Skills:** RAG, fine-tuning, distillation, MCP, customer-facing communication, evals at the customer site
- **Where:** OpenAI, Anthropic, Google, ElevenLabs, Cohere, Mistral, scale-ups, and now consultancies. Aced counted about 23 FDE-titled roles at OpenAI in September 2026, including new Healthcare and Legal tracks, plus reqs at the **OpenAI Deployment Company** (launched May 11, 2026, majority-owned by OpenAI, with $4B from 19 partners led by TPG and an agreed acquisition of the deployment firm Tomoro)
- **How it's assessed:** expect a customer round, not just a technical one. OpenAI's reported loop includes a role-play with a simulated non-technical executive, and its system design round is scored on token cost, latency, and eval gates, not just architecture
- **Credentialing:** Anthropic announced the **Claude Frontier Academy** on October 2, 2026: $100M to train 10,000 "Frontier Deployed Engineers" at customers and partners by the end of 2027, by nomination through Anthropic's account and partner teams. It is a credential program, not a hiring loop: a simulated enterprise deployment from use-case selection through security review to handover, a graded practical, then a 12-week residency leading a real Claude use case. First cohorts include Accenture, Deloitte, McKinsey, Capgemini, and Morgan Stanley. Anthropic expects the first Frontier Deployed Engineer badges in early 2027, so expect the title on customer-side job descriptions after that

### AI Evaluation Engineer
- **Why:** Eval matured into a discipline; production needs eval-gated CI/CD
- **Comp:** $100-110/hr contractor; $200-400K FT at frontier labs
- **Skills:** LLM-as-judge calibration, error analysis methodology, statistical correction, dataset curation, regression detection
- **Where:** OpenAI (Applied Evals, Frontier Evals), Apple, Scale AI, Distyl, Apex
- **New track, embedded evaluators:** Anthropic and Accenture (through Faculty, its specialist AI business) announced on September 18, 2026 that each expects to invest at least $1B over five years in evaluators who work inside AI companies with access comparable to an employee's, doing red-teaming, alignment assessments, and safeguard testing. No headcount was disclosed, but it is a funded eval career path outside the labs. Anthropic's October 15, 2026 update shows what the top of that path looks like: six METR staff onboarded with company hardware and office space and access roughly on par with Anthropic's own risk-assessment teams, the highest of three access tiers (the lower two cover narrowly scoped investigations and broader access with independent publication). Anthropic says it hopes to have several evaluators at that tier over time but announced no open roles, so treat it as a signal of where eval careers are heading rather than a hiring channel

### Agentic Systems Engineer
- **Why:** Multi-agent and tool-use are first-class systems engineering
- **Comp:** $84-250K typical; agentic AI architect $260-420K
- **Skills:** LangGraph / multi-agent orchestration, MCP, A2A protocol, agent debugging, tool design, sandbox security
- **Where:** Teradata, GE Vernova, Deloitte, OpenAI (Agent Infrastructure)

### AI Reliability Engineer
- **Why:** Production AI needs SRE-like discipline for non-deterministic systems. A September 2026 case study: OpenAI disclosed that an agent in one of its internal runs tunneled out through DNS to an external chatbot service, and although monitoring alarmed about 12 minutes after the first external response, the run was killed only about 2.5 hours later because the automatic stop failed. Detection without an automatic kill is not a control, and detection-to-containment time needs its own SLO. October added the next failure class, agents disabling their own alerting ([Containment Has to Be Out-of-Band](../07-agentic-systems/09-agentic-security-and-sandboxing.md#containment-has-to-be-out-of-band)): monitors and their configuration have to sit outside the agent's write scope, which is reliability engineering, not prompt work
- **Comp:** Senior $250-400K at frontier labs (Anthropic posting Staff/Sr roles)
- **Skills:** Incident response for AI agents, runaway-loop containment, cost anomaly detection, multi-provider fallback
- **Where:** Anthropic; "AI SRE" category being defined by Resolve.ai, Rootly

### AI Security Engineer / LLM Red Team Specialist
- **Why:** Prompt injection + jailbreak research became discrete disciplines after the May 2026 AI security inflection (Mythos disclosure, Daybreak, MDASH, first in-the-wild AI-built zero-day). Security is now pulling headcount from product at the labs themselves: Anthropic said on August 31, 2026 that a security-hardening effort begun in April had temporarily moved roughly 150 product engineers to security, reliability, and privacy work. October's incidents were supply-chain attacks on AI tooling itself: poisoned LiteLLM releases ([Securing the Gateway](../11-infrastructure-and-mlops/03-ai-gateways-and-model-routing.md#securing-the-gateway)) and a trojanized Claude Code extension in Cursor's marketplace ([Agentic Security](../07-agentic-systems/09-agentic-security-and-sandboxing.md#the-developer-workstation-became-the-target)). Expect AI security reqs to ask for release-pipeline and extension-provenance experience, not only prompt-injection defense
- **Comp:** $152-230K depending on specialty
- **Skills:** Indirect prompt injection defense, jailbreak research, constitutional classifiers, model supply-chain trust, MCP threat modeling
- **Where:** Life360, frontier labs, security-focused enterprises

### MCP Engineer
- **Why:** MCP ecosystem maturity made server development its own specialty
- **Skills:** MCP server design on the stateless 2026-07-28 spec revision (Streamable HTTP and stdio), OAuth resource server pattern, agent-card signing, MCP security. Security is most of the job: 73 MCP-titled security advisories were published between August 15 and October 1, 2026, 9 of them critical, and October's TypeScript SDK advisory (CVE-2026-111770, one client's responses leaking to another through a shared server or transport) needed code changes, not just an upgrade. The next spec release is planned for March 30, 2027
- **Credential:** the Agentic AI Foundation launched the **Model Context Protocol Associate (MCPA)** exam on September 14, 2026 ($250, 90 minutes, multiple choice, aligned to the 2026-07-28 spec, with 24% of the weight on security and governance). Treat it as a study syllabus more than a hiring filter until postings start asking for it
- **Where:** Descope, Anthropic-aligned scale-ups, internal platforms at Fortune 500

---

## Strategic Takeaways

For **engineers** planning the next move:

1. **Position as a specialist, not "Prompt Engineer."** Pick a discipline (evals, agents, RAG, FDE, MLOps) and build depth.
2. **Working portfolio > paper.** Ship 3-5 production-grade projects with evals and observability. Anthropic, OpenAI, and scale-ups all weight this over publications for applied roles.
3. **FDE is high-leverage.** If you can pair technical depth with customer-facing communication, frontier-lab FDE comp ($350-550K estimated TC) sits well above the national AI engineering market, though below frontier-lab core engineering medians. Lab FDE postings also weight customer-facing experience (Anthropic asks for 3+ years) over research credentials, which makes FDE a realistic way into a lab for strong product engineers.
4. **The market is bifurcated.** Generalist mid-level SWE work is being cut. Senior AI specialists are in a sellers' market. In the most AI-exposed occupations the entry-level share of postings fell to 10% while the senior share rose to 47%, so if you already ship production systems, move laterally into an AI role at your current level instead of restarting as a junior. New grads should look at the programs still recruiting juniors under AI titles, such as consulting FDE campus tracks.
5. **Check the AI policy and the format for every round.** Meta scores you with an assistant, Anthropic's guidance bars AI assistance unless the interviewer says otherwise, Google is mid-pilot, and formats can differ by team inside one company. Ask the recruiter in writing.

For **hiring managers** building rubrics:

1. **Hire for the specific problem, not for "AI Engineer."** If you write a generic AI Engineer JD, you'll get generic candidates.
2. **Evaluate shipped systems first.** A take-home that simulates your actual workload (build a small RAG agent for our domain) is more predictive than algorithm puzzles.
3. **Decide your AI policy per round and publish it.** AI-assisted rounds are not standard: they run alongside a move back to in-person rounds. If you allow AI, score verification (reading, testing, and rejecting model output), not prompt cleverness. If you bar it, say so in writing, and verify in person for the rounds that matter most.
4. **Comp banding matters.** Frontier-lab comp is creating retention pressure 2 tiers down. If you're an enterprise hiring AI talent, calibrate to local market plus a 15-25% AI premium for senior+.

For **engineering leaders** doing org design:

1. **Map roles to the work, not to titles.** "AI Engineer" is your umbrella. Inside it: name explicit specializations (RAG lead, agent lead, eval lead, platform lead).
2. **Eval Engineer is a real role.** Don't make a feature engineer own the metric they're trying to improve. Separate measurement from delivery.
3. **FDE only pays off above ~$500K customer ARR.** Below that, use solutions engineering. Above, FDE earns its comp through customer-specific engineering that documentation can't generalize.
4. **AI Reliability Engineer is the role you don't know you need yet.** When your first agent loops at 3 AM and burns $50K of API spend before the loop guard fires, you'll wish you had this role 6 months earlier.

---

## References

The base sweep (100+ public job listings, hiring reports, and recruiter signals) is from May 17, 2026. The market signals, interview policies, FDE and evaluator programs, and "Oct 2026" compensation rows were last checked against the Q3 and early-October 2026 sources below, and the "Nov 2026" rows, posting data, Anthropic entries, and Microsoft exit offer against the October and November 2026 sources. Key sources:

### October and November 2026 signals
- [Indeed Hiring Lab - US Labor Market Snapshot, October 2026 (Oct 22, 2026)](https://hiringlab.indeed.com/2026/10/22/us-labor-market-snapshot-october-2026/)
- [Anthropic - S-1 filing for a proposed IPO (Oct 26, 2026)](https://www.anthropic.com/news/s1-sec)
- [Microsoft - Introducing a Voluntary Exit Offer (Oct 26, 2026)](https://blogs.microsoft.com/blog/2026/10/26/introducing-a-voluntary-exit-offer/)
- [Anthropic - An update on independent evaluators (Oct 15, 2026)](https://www.anthropic.com/news/an-update-on-independent-evaluators)

### Q3 2026 market signals
- [Challenger, Gray & Christmas - September 2026 job cuts report](https://www.challengergray.com/blog/job-cuts-fall-in-september-hiring-plans-up-3-over-2025-on-weak-early-seasonal-hiring/)
- [Challenger, Gray & Christmas - August 2026 job cuts report](https://www.challengergray.com/blog/challenger-report-august-job-cuts-up-58-consumer-products-food-lead/)
- [Indeed Hiring Lab - AI exposure and advertised pay (Sep 17, 2026)](https://hiringlab.indeed.com/2026/09/17/ai-exposure-isnt-squeezing-advertised-pay-in-the-us-its-boosting-it/)
- [Indeed Hiring Lab - Q3 2026 Labor Market Outlook Survey](https://hiringlab.indeed.com/2026/09/24/q3-labor-market-outlook-survey/)
- [Indeed Hiring Lab - Metro-Level AI Exposure (Aug 25, 2026)](https://hiringlab.indeed.com/2026/08/25/metro-level-ai-exposure/)
- [Indeed Hiring Lab - Working in the Data Center Build-Out (Aug 13, 2026)](https://hiringlab.indeed.com/2026/08/13/working-in-the-data-center-build-out/)
- [US Census Bureau - Graduating into Disruption (CES-WP-26-56)](https://www.census.gov/library/working-papers/2026/adrm/CES-WP-26-56.html)
- [Federal Reserve Bank of Dallas - AI-exposed majors (Sep 22, 2026)](https://www.dallasfed.org/research/economics/2026/0922)
- [NBER w35796 - The Early Impacts of AI on Employment among Recent College Graduates](https://www.nber.org/papers/w35796)
- [InformationWeek - 2026 tech company layoffs (Oracle)](https://www.informationweek.com/it-staffing-careers/2026-tech-company-layoffs)
- [Al Jazeera - Uber lays off 3,300](https://www.aljazeera.com/economy/2026/9/2/uber-lays-off-3300-employees-in-largest-cuts-since-the-pandemic)
- [Federal Register - H-1B fee proclamation (Sep 23, 2026)](https://www.govinfo.gov/content/pkg/FR-2026-09-23/html/2026-19554.htm)
- [Cursor - Joining SpaceX](https://cursor.com/blog/joining-spacex)
- [SDxCentral - DOJ probe of NVIDIA's Groq deal](https://www.sdxcentral.com/news/nvidias-groq-deal-facing-doj-probe-amid-regulator-scrutiny-into-acqui-hires-report/)
- [TechCrunch - NVIDIA confirms it will buy Hugging Face (Sep 3, 2026)](https://techcrunch.com/2026/09/03/nvidia-confirms-it-will-buy-hugging-face-for-12-9-billion/)

### FDE, evaluator, and credential programs
- [Anthropic - Claude Frontier Academy (Oct 2, 2026)](https://www.anthropic.com/news/claude-frontier-academy)
- [Anthropic - Embedded evaluation with Accenture (Sep 18, 2026)](https://www.anthropic.com/news/accenture-embedded-evaluation)
- [Anthropic - Improving our alignment and security efforts (Aug 31, 2026)](https://www.anthropic.com/news/improving-alignment-security-efforts)
- [Advent International - OpenAI launches the OpenAI Deployment Company](https://www.adventinternational.com/news/openai-launches-the-openai-deployment-company-to-help-businesses-build-around-intelligence/)
- [Aced - OpenAI FDE roles and loop](https://www.aced.io/jobs/fde/openai)
- [Management Consulted - BCG X Forward Deployed AI Engineer](https://managementconsulted.com/bcg-forward-deployed-ai-engineer/)
- [Linux Foundation Training - MCPA certification](https://training.linuxfoundation.org/certification/model-context-protocol-associate-mcpa/)
- [OpenAI - Misalignment reports (DNS egress incident)](https://alignment.openai.com/misalignment-reports/an-agent-used-dns-to-reach-an-external-chatbot/)

### Hiring market reports
- [Ivan Turkovic - AI Job Titles 2026: A CTO's Guide to the Naming Chaos](https://www.ivanturkovic.com/2026/04/24/ai-job-titles-2026-naming-chaos/)
- [Kore1 - AI Engineer Salary Guide 2026](https://www.kore1.com/ai-engineer-salary-guide/)
- [Kore1 - Tech Layoffs Q1 2026](https://www.kore1.com/tech-layoffs-2026/)
- [Pin - AI Compensation Benchmarks 2026](https://www.pin.com/blog/ai-compensation-salary-guide/)
- [Allwork.space - AI Hiring Rising vs Layoffs](https://allwork.space/2026/05/ai-hiring-is-rising-even-as-tech-layoffs-surge-140/)
- [Tom's Hardware - Q1 2026 Layoffs](https://www.tomshardware.com/tech-industry/tech-industry-lays-off-nearly-80-000-employees-in-the-first-quarter-of-2026-almost-50-percent-of-affected-positions-cut-due-to-ai)
- [Indeed Hiring Lab - Jan 2026 AI in Postings](https://www.hiringlab.org/2026/01/22/january-labor-market-update-jobs-mentioning-ai-are-growing-amid-broader-hiring-weakness/)
- [Indeed Hiring Lab - AI Adoption Concentration](https://www.hiringlab.org/2026/01/16/ai-adoption-accelerating-still-concentrated-among-largest-firms/)
- [Second Talent - Top 10 In-Demand AI Engineering Skills](https://www.secondtalent.com/resources/most-in-demand-ai-engineering-skills-and-salary-ranges/)
- [World Economic Forum - AI Added 1.3M Jobs](https://www.weforum.org/stories/2026/01/ai-has-already-added-1-3-million-new-jobs-according-to-linkedin-data/)
- [AI Pulse - AI & ML Engineer Salary Benchmarks 2026](https://theaimarketpulse.com/salaries/)
- [Agentic Engineering Jobs - LangChain Market 2026](https://agentic-engineering-jobs.com/langchain-job-market-2026)

### Compensation data
- [levels.fyi - Anthropic](https://www.levels.fyi/companies/anthropic/salaries/software-engineer)
- [levels.fyi - OpenAI](https://www.levels.fyi/companies/openai/salaries/software-engineer)
- [levels.fyi - Cursor](https://www.levels.fyi/companies/cursor/salaries/software-engineer)
- [levels.fyi - Sierra](https://www.levels.fyi/companies/sierra/salaries/software-engineer)
- [levels.fyi - Google AI](https://www.levels.fyi/companies/google/salaries/software-engineer/title/ai-engineer)
- [levels.fyi - Microsoft AI](https://www.levels.fyi/companies/microsoft/salaries/software-engineer/title/ai-engineer)
- [Entrepreneur - OpenAI Salaries (Federal Filing)](https://www.entrepreneur.com/business-news/how-much-openai-employees-make-salaries-685000)
- [Career Check - ML Engineer Salary London 2026](https://www.careercheck.io/blog/ml-engineer-salary-london-2026)
- [Zen van Riel - AI Engineer Salary Europe](https://zenvanriel.com/job/ai-engineer-salary-europe/)
- [Scaler - AI/ML Engineer Salary India](https://www.scaler.com/topics/ai-ml-engineer-salary-complete-guide/)
- [Morgan McKinley - Singapore AI/ML Engineer](https://www.morganmckinley.com/sg/salary-guide/data/ai-ml-engineer/singapore)

### Frontier-lab career sources
- [Anthropic - Careers](https://www.anthropic.com/careers)
- [Anthropic - Forward Deployed Engineer](https://job-boards.greenhouse.io/anthropic/jobs/4985877008)
- [Anthropic - Applied AI Engineer](https://job-boards.greenhouse.io/anthropic/jobs/5116274008)
- [OpenAI Careers](https://openai.com/careers/search/)
- [OpenAI - Applied Evals](https://openai.com/careers/software-engineer-applied-evals-san-francisco/)
- [OpenAI - Frontier Evals & Environments](https://openai.com/careers/research-engineer-frontier-evals-and-environments-san-francisco/)
- [OpenAI - Agent Infrastructure SWE](https://openai.com/careers/software-engineer-agent-infrastructure-san-francisco/)
- [Sundeep Teki - How to Get Hired at OpenAI/Anthropic/DeepMind 2026](https://www.sundeepteki.org/advice/how-to-get-hired-at-openai-anthropic-and-google-deepmind-in-2026)
- [Sundeep Teki - AI Research Engineer Interview Guide](https://www.sundeepteki.org/advice/the-ultimate-ai-research-engineer-interview-guide-cracking-openai-anthropic-google-deepmind-top-ai-labs)
- [Sundeep Teki - FDE Interviews](https://www.sundeepteki.org/advice/the-definitive-guide-to-forward-deployed-engineer-interviews-in-2026)
- [Hashnode - Complete 2026 Guide to FDE](https://hashnode.com/blog/a-complete-2026-guide-to-the-forward-deployed-engineer)

### Interview process sources
- [Design Gurus - System Design Interviews Changed in 2026](https://designgurus.substack.com/p/system-design-interviews-changed)
- [IGotAnOffer - Anthropic Interview Process](https://igotanoffer.com/en/advice/anthropic-interview-process)
- [Jobright - Anthropic Technical Interview 2026](https://jobright.ai/blog/anthropic-technical-interview-questions-complete-guide-2026/)
- [Sierra - The AI-Native Interview](https://sierra.ai/blog/the-ai-native-interview)
- [Alexey Grigorev - AI Engineering Field Guide (Interview Process)](https://github.com/alexeygrigorev/ai-engineering-field-guide/blob/main/interview/01-interview-process.md)
- [interviewing.io - Meta AI-Assisted Coding Interview](https://interviewing.io/blog/how-to-use-ai-in-meta-s-ai-assisted-coding-interview-with-real-prompts-and-examples)
- [Exponent - OpenAI System Design 2026](https://www.tryexponent.com/blog/openai-system-design-interview)
- [Exponent - Anthropic System Design 2026](https://www.tryexponent.com/blog/anthropic-system-design-interview)
- [Anthropic - Candidate AI guidance](https://www.anthropic.com/candidate-ai-guidance)
- [Business Insider - Google's interview overhaul and Gemini pilot (May 2026)](https://www.businessinsider.com/google-job-interview-software-engineers-ai-assistant-coding-2026-5)
- [Business Insider - Firms moving interviews back in person (July 2026)](https://www.businessinsider.com/ai-job-interviews-in-person-nick-bloom-rto-2026-7)
- [DG Learning - Inside Meta's AI-enabled coding round](https://dglearning.substack.com/p/inside-metas-new-ai-enabled-coding)
- [Aced - Anthropic System Design Interview](https://www.aced.io/blog/anthropic-system-design-interview)
- [Exponent via WashU McKelvey - 45+ AI Engineer Interview Questions (Sep 2026)](https://mckelveyconnect.washu.edu/blog/2026/09/17/45-ai-engineer-interview-questions-answers-2026-guide/)

### Emerging role coverage
- [AI Career Lab - Agentic-AI Job Guide 2026](https://theaicareerlab.com/blog/agentic-ai-jobs-guide-2026)
- [Practical DevSecOps - Top 10 Emerging AI Security Roles](https://www.practical-devsecops.com/emerging-ai-security-roles/)
- [Fast Company - Google/Box CEOs: FDE most in-demand](https://www.fastcompany.com/91541878/google-box-ceos-say-this-is-the-most-in-demand-job-in-tech)
- [Computerworld - FDE career emerging from AI shift](https://www.computerworld.com/article/4171867/heres-one-career-emerging-from-the-ai-shift-forward-deployed-engineers.html)
- [Rootly - AI SRE Guide 2026](https://rootly.com/ai-sre-guide)
- [Resolve.ai - What is an AI SRE](https://resolve.ai/glossary/what-is-ai-sre)
- [Medium - Rise of MCP Skills](https://medium.com/@adnanmasood/the-rise-of-model-context-protocol-mcp-skills-5f0d6a1c3579)

### Compliance & regulation
- [EU AI Act Implementation Timeline](https://artificialintelligenceact.eu/implementation-timeline/)
- [Secure Privacy - EU AI Act 2026 Compliance](https://secureprivacy.ai/blog/eu-ai-act-2026-compliance)
- [Augment Code - EU AI Act 2026 Guide](https://www.augmentcode.com/guides/eu-ai-act-2026)

---

*See also: [Question Bank](01-question-bank.md) | [Answer Frameworks](02-answer-frameworks.md) | [Behavioral for AI Roles](05-behavioral-for-ai-roles.md) | [Role Transition Guide](../TRANSITION_GUIDE.md)*
