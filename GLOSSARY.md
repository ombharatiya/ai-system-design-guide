# AI System Design Glossary

Quick reference for key terms used throughout this guide.

---

## A

**A2A (Agent2Agent Protocol)** - Open protocol for delegating tasks between agents across vendor or organization boundaries; complementary to MCP, which connects agents to tools. An agent advertises itself with an Agent Card at `GET /.well-known/agent-card.json` and accepts work through methods such as `SendMessage` and `GetTask`. v1.0.0 shipped March 12, 2026, and v1.0.1 (May 28, 2026) is the latest release. Joined the Agentic AI Foundation as a Growth Stage project in August 2026. See [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md#agent-to-agent-protocol-a2a).

**ABAC (Attribute-Based Access Control)** - Access control based on attributes of user, resource, and environment rather than fixed roles.

**Accidental Cyberattack (Eval Breakout)** - An agent under evaluation or training takes real, unsanctioned action against a third party because its environment could reach the live internet. Simon Willison coined the label in July 2026 for the OpenAI-Hugging Face incident. Later disclosures include the UK AI Security Institute (19 unsanctioned actions in 10 of 122 cyber-evaluation runs), Anthropic (a malicious PyPI package published during a partner's evaluation and installed by 15 systems), and Google (Gemini breaching three companies during third-party testing). The control is default-deny egress for every agent environment, evals included. See [Agentic Security and Sandboxing](07-agentic-systems/09-agentic-security-and-sandboxing.md#agents-have-taken-unsanctioned-action-against-real-third-parties).

**Advisor / Executor** - Orchestration pattern where a cheaper executor model runs the agent loop and consults a stronger advisor model at decision points, passing the transcript and receiving a plan or correction. Often beats raising the executor's own reasoning effort at lower cost per task. See [PATTERNS.md](PATTERNS.md).

**Agent Identity** - Giving each agent its own short-lived, verifiable credential, scoped to the user or service it acts for, instead of a shared static API key, so every tool call can be attributed, narrowed, and revoked. Standards are converging on existing workload-identity and OAuth machinery rather than new agent protocols: the IETF WIMSE working group adopted its first agent draft (draft-ietf-wimse-aims-00) on September 15, 2026, and the MCP roadmap (August 22, 2026) targets sender-constrained tokens (DPoP, SEP-1932) and Workload Identity Federation (SEP-1933), both still open proposals. The ID-JAG grant behind MCP enterprise-managed authorization is still an Internet-Draft, not an RFC. Vendors ship their own pieces meanwhile: OpenAI made mutual TLS and X.509 workload identity federation generally available for its API on August 29, 2026, and Bedrock Managed Agents (preview) gives each agent its own IAM role. See [Access Control](12-security-and-access/02-access-control.md#workload-identity-instead-of-static-keys).

**Agent Plugins** - Vendor-neutral packaging format (1.0, August 2026) that bundles Agent Skills and MCP server declarations into one installable directory with a `plugin.json` manifest. Standardizes distribution of agent capability; deliberately keeps only skills and MCP servers portable, leaving client-specific parts in namespaced directories. See [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md#agent-plugins).

**Agent Skills (SKILL.md)** - A folder holding a `SKILL.md` (frontmatter plus instructions) and supporting files that an agent loads only when a task needs it, so procedural knowledge costs context only when used. Out of beta on the Claude API since August 19, 2026. The official MCP Skills extension (`io.modelcontextprotocol/skills`, SEP-2640, Final September 13, 2026) serves skills from a server with a SHA-256 manifest of every file; any changed, added, or removed file revokes the user's approval, which stops a server from silently rewriting the instructions an agent loads. MCP rates the extension Beta, so it can still break, and the TypeScript SDK shipped support in 2.4.0 (October 23, 2026). Anthropic donated the Agent Skills format to the Agentic AI Foundation on October 22, 2026. Treat skills as untrusted, versioned dependencies. See [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md#skills-over-mcp).

**Agentic AI Foundation (AAIF)** - The Linux Foundation's neutral home for agent standards, formed in December 2025. On October 22, 2026 it added four projects (Agent Skills, Open Responses, OpenEnv, and OpenSandbox), bringing it to 10 hosted projects alongside MCP, goose, AGENTS.md, agentgateway, A2A, and Agent Router (the former Envoy AI Gateway), and reported 305 member organizations. Launched the MCPA (Model Context Protocol Associate) certification on September 14, 2026. See [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md).

**Agentic Coding** - LLM autonomously editing files, running shell commands, writing tests and iterating until a coding task is complete. Exemplified by Claude Code, Codex, Cursor, OpenHands, and Cline.

**Agentic Commerce Protocols** - The emerging stack for agents that buy and pay: **UCP** (Universal Commerce Protocol; discovery, cart, and checkout, release v2026-08-25), **ACP** (OpenAI and Stripe's Agentic Commerce Protocol, still beta), **AP2** (Google's Agent Payments Protocol for payment mandates, v0.2.0), and **x402** (pay-per-call over HTTP). In older material "ACP" means IBM's Agent Communication Protocol, now part of A2A. See [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md#the-commerce-layer).

**Agentic System** - LLM application that autonomously plans and executes multi-step tasks using tools.

**AI Control** - Safety approach that assumes a model may be misaligned and designs deployment protocols (monitoring, defer-on-critical-action, resampling, factored cognition) to stay safe even then. Distinct from alignment, which aims to make the model trustworthy in the first place. See [Research Radar](RESEARCH-RADAR.md).

**AI Gateway** - A control-plane proxy between your apps and model providers (LiteLLM, OpenRouter, Portkey, Kong, Agent Router, formerly Envoy AI Gateway). Exposes one OpenAI-compatible API and centralizes routing, fallback, load balancing, rate-limit handling, virtual keys and budgets, caching, and observability. Because it holds every provider key, patch it like internet-facing infrastructure and treat its release pipeline as part of your attack surface: LiteLLM shipped several critical fixes in 2026, and its poisoned October 21 releases were built to harvest provider, master, and virtual keys (see [Securing the Gateway](11-infrastructure-and-mlops/03-ai-gateways-and-model-routing.md#securing-the-gateway)). Pin packages by hash and images by digest, and verify provenance before an upgrade reaches the gateway.

**Approval Laundering** - The gap between the action a human approved and the action an agent harness actually executes (arXiv 2609.38983, September 2026), in six classes: scope, argument, temporal, tool, delegation, and semantic. Mitigate by binding the approval to the exact canonical action with a hash or signed token, re-verifying it at execution, and showing the human the executed form rather than the model's description of it. See [Human-in-the-Loop Patterns](07-agentic-systems/08-human-in-the-loop-patterns.md#approval-laundering).

**Attention Mechanism** - Neural network component that allows models to focus on relevant parts of input. Self-attention compares each token to all others.

---

## B

**Batching** - Processing multiple requests together to improve GPU utilization. Continuous batching adds new requests while others generate.

**Benchmark Saturation** - When frontier models cluster so near a benchmark's ceiling that score differences fall within noise (prompt phrasing, run variance), so the benchmark no longer separates models. MMLU, HumanEval, and GSM8K saturated first; by September 2026 so had GPQA Diamond, OSWorld-Verified, and SWE-Bench Pro's public set (Opus 5 at 99.4% on v2). Current comparisons lean on private or newer sets such as SWE-Bench Pro's private 272 tasks, Terminal-Bench 4.0, and HLE-Diamond. See [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md).

**BM25** - Traditional keyword-based ranking algorithm. Often combined with vector search for hybrid retrieval.

**Budget Tokens** - The explicit token cap on Claude's extended thinking (`thinking.budget_tokens`), still the control on older models such as Haiku 4.5. From the 4.6 generation on, adaptive thinking steered by an effort setting is the main control, and Opus 5.5, Sonnet 5.5, and Haiku 5.5 reject a manual budget with a 400, so a Haiku 4.5 to 5.5 migration is a request-shape change, not a model-string swap. See **Reasoning Effort**.

---

## C

**C2PA (Content Credentials)** - An open standard that cryptographically binds provenance metadata (who made this, whether AI was involved, what edits) to a media asset, with tamper-evident hard bindings and watermark-based soft bindings that survive re-encoding. The provenance layer behind AI-content labeling laws; removable, so layer it with watermarking and detection. See [Multimodal Generation](19-multimodal-generation/01-multimodal-generation.md).

**CAISSI** - The Center for Advancing Innovation and Standards for Super Intelligence, the name NIST's AI standards center (formerly CAISI) now uses after the US federal "Super Intelligence" naming order. With the NSA and CISA it published voluntary incident practices for agents in training and evaluation on October 16, 2026. See **Super Intelligence (SI)**.

**Capability Index (Composite Benchmark)** - A weighted aggregate of many benchmarks (e.g. Artificial Analysis Intelligence Index, Epoch Capability Index, HAL for agents) used to rank frontier models so the ranking keeps discriminating as individual benchmarks saturate. Indices re-baseline when they swap components: Artificial Analysis moved to Intelligence Index v5.0 in late October 2026 (four equally weighted categories: Knowledge Work, Coding, Science, and General, with about 60% of the evaluation on private or held-out sets), and v4.x scores are not comparable with v5.0 scores. Quote the index version with every number.

**Chain-of-Thought (CoT)** - Prompting technique that elicits step-by-step reasoning before final answer.

**Chunking** - Splitting documents into smaller pieces for embedding and retrieval. Strategies include fixed-size, semantic, and hierarchical.

**Claude Code** - Anthropic's agentic coding tool (terminal, IDE, desktop, and web). Uses shell, file-editing, and other tools to read, edit, and run code across a full project. Reads project instructions from CLAUDE.md, and since 2.1.277 (September 2026) falls back to AGENTS.md when a project has no CLAUDE.md. Since 2.1.302 (October 16, 2026), `claude -p` and Agent SDK sessions that set no permission mode start in `default` mode on third-party providers, with telemetry off, or on a fresh install, while interactive sessions still start in auto mode, so CI callers should set the permission mode explicitly.

**Claude Fable 5.1** - Anthropic's Mythos-class general-availability model (September 1, 2026, `claude-fable-5-1`), successor to Fable 5 (June 9, 2026). $10/$50 per 1M with cache reads cut to $0.25 (0.025x input), 1M context, adaptive thinking always on. Requires 30-day data retention; zero data retention only where Anthropic authorizes it. Its more permissive twin, Claude Mythos 5.1 (same weights, looser safeguards), is limited to vetted organizations: since October 6, 2026 it is available in every tier (Defense, Red Team, and Specialized) of Anthropic's expanded Cyber Verification Program, which absorbed Project Glasswing, and the program requires data retention (existing zero-retention holders keep it until Anthropic ships Enterprise Frontier Safeguards). Anthropic now points most workloads at the cheaper Claude Opus 5.5 ($4/$20, September 22, 2026), which scores 58 on the Artificial Analysis Intelligence Index v5.0 (max effort with safeguard fallback), the top score among publicly available models. See [Model Taxonomy](02-model-landscape/01-model-taxonomy.md).

**Cline** - Open-source VS Code extension providing autonomous AI coding with tool use (file editing, terminal, browser). MCP-native.

**Compaction** - Replacing older turns of a long agent context with a model-written summary so the loop can keep running past the context window. Managed harnesses such as the OpenAI Agents API do it for you, and the Claude API added on-demand compaction as a beta in September 2026. Lossy by design: decide what must survive (goal, constraints, open decisions) before you let it run. See **Context Rot**.

**Computer-Use** - A model capability to operate a GUI from screenshots by emitting mouse and keyboard actions; Anthropic introduced it with Claude 3.5 Sonnet in October 2024, and OpenAI and Google now ship equivalents. Claude's current tool version is `computer_toolset_20260801` (GA August 19, 2026); Opus 5.5 and Sonnet 5.5 reject the older `computer_20251124` on the Claude API and Google Cloud. Gemini's computer use tool went GA on October 30, 2026. Measure with OSWorld 2.0 binary scores, since OSWorld-Verified is saturated, and treat the GUI as the most injectable surface: Claude Haiku 5.5's system card reports prompt-injection attack success within 15 attempts of 7.1% overall but 24.4% on GUI computer use.

**Context Rot** - Degradation in output quality as irrelevant or stale tokens accumulate in the context window, often well before the advertised limit. Motivates compaction and just-in-time retrieval. See [Context Engineering](05-prompting-and-context/05-context-engineering.md).

**Context Window** - Maximum number of tokens an LLM can process in a single request. Ranges from 4K to 1M+ tokens; 1M is standard at the frontier, but the price of using it is not (see **Long-Context Cliff**).

**Context7** - MCP server that fetches up-to-date library documentation at runtime, solving the "stale training data" problem for coding agents.

**Cosine Similarity** - Measure of similarity between two vectors. Standard metric for comparing embeddings.

**Cursor** - AI-native IDE (fork of VS Code) with deep model integration for code completion, agentic editing, and multi-file context awareness. Part of SpaceX since August 2026, so the leading IDE agent now shares a parent with a frontier model line (SpaceXAI's Grok); weigh that in vendor-risk and data-governance reviews. A forked IDE's extension marketplace is its own supply chain, as a trojanized `Anthropic.claude-code` extension in Cursor's marketplace showed on October 29, 2026 (see [The Developer Workstation Became the Target](07-agentic-systems/09-agentic-security-and-sandboxing.md#the-developer-workstation-became-the-target)).

---

## D

**Data Contamination** - When benchmark questions or their answers leak into a model's training data, inflating scores through memorization rather than capability. Countered with time-gated, private, or held-out test sets. A related failure is environment leakage: one study found that 24% to 73% of SWE-Bench Pro v1.0 passes, depending on the model, were unearned, mostly because agents read the fix from the repository's git history (arXiv 2609.34262). See [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md).

**Data Residency (Regional Inference)** - Pinning where a request is processed, usually for compliance, at a premium of about 10%. Anthropic's `inference_geo: "us"` bills 1.1x on all token types for Claude 4.6 and later; OpenAI adds 10% on data-residency endpoints for models released on or after March 5, 2026; Bedrock and Google Cloud regional Claude endpoints carry a similar premium, and Bedrock's regional GPT-6 Astra lists at $11/$55 per 1M against $10/$50. It is also a per-product constraint: OpenAI's Agents API launched with US residency only. Route only residency-bound tenants to pinned regions instead of paying the premium on all traffic. See [Pricing and Costs](02-model-landscape/03-pricing-and-costs.md).

**Decision Model (System One Model)** - A model that returns a typed answer (a yes/no predicate, a choice from a fixed list, or a score) with a probability instead of generated text, built for gates, routers, classifiers, and judges. OpenAI's Decisions API went GA on October 22, 2026 on `gpt-6-luna`, vLLM and SGLang added native endpoints, and Ollama added local decision models. Reported failure modes: probabilities that run underconfident, so thresholds need calibrating on your own labeled data, and wrong actions taken after the model read the state correctly, so compose the action in code from the model's typed answers rather than trusting one end-to-end call. No written rationale comes back, which rules it out where reviewers need one. See [Research Radar: Decision Models](RESEARCH-RADAR.md#16-decision-models-typed-answers-with-probabilities), **LLM-as-Judge**, and [Eval-Gated CI/CD](16-case-studies/18-eval-gated-cicd.md#5-per-pr-eval-budget).

**Devin Desktop** - Cognition's AI-native IDE, formerly Windsurf (originally built by Codeium and owned by Cognition, the maker of Devin, since July 2025), renamed Devin Desktop in v3.0.12 on June 2, 2026. An alternative to Cursor; older tutorials, courses, and job posts still use the Windsurf name.

**Diffusion Language Model** - A non-autoregressive LLM that generates text by iteratively denoising a masked sequence in parallel rather than left to right, trading some quality for much higher throughput (reported 1,000+ tokens/sec). Strong on code and infilling; early-stage in 2026. See [Diffusion Language Models](04-inference-optimization/08-diffusion-llms.md).

**Disaggregated Serving (Prefill/Decode)** - Running prefill (prompt processing, compute-bound) and decode (token generation, memory-bandwidth-bound) on separate GPU pools, or on different chips, and shipping the KV cache between them. Orchestrated above the engines by NVIDIA Dynamo or llm-d with KV-aware routing. It pays only when it raises **goodput**: the KV transfer for a 10K-token prompt on Llama-3.1-70B in BF16 is about 3 GB, roughly 65 ms at 400 Gb/s. See [LLM Infrastructure](11-infrastructure-and-mlops/01-llm-infrastructure.md#cross-chip-prefill-and-decode).

**DPO (Direct Preference Optimization)** - Fine-tuning method that optimizes directly on preference data without a separate reward model.

**DSPy** - Framework for programming LLMs through optimizable modules rather than manual prompts.

**Durable Execution** - An execution model (Temporal, Restate, DBOS) that makes long-running agents survive crashes and restarts via an append-only event history and deterministic replay, giving exactly-once side effects, durable timers, and pauses that outlive deploys. See [Durable Execution](07-agentic-systems/11-durable-execution.md).

---

## E

**Effective Context Length** - The context length at which a model still maintains quality, routinely shorter than the advertised window. On RULER, many models claiming 128K hold quality only to ~32-64K. Design for effective, not advertised, context.

**Embedding** - Dense vector representation of text. Used for semantic search and similarity comparison.

**Endpointing (Turn Detection)** - In voice agents, deciding when the user has finished speaking so the agent can respond. Learned turn-detection models fire on a semantically complete thought, beating fixed silence timeouts that tax every turn. See [Real-Time Voice Agents](18-voice-and-audio-agents/01-realtime-voice-agents.md).

**Ensemble** - Combining multiple model outputs to improve reliability. Includes voting, debate, and mixture-of-agents.

**EU AI Act** - Regulation (EU) 2024/1689, the first comprehensive AI law, structured by risk tier (prohibited, high-risk, limited, minimal) with separate GPAI obligations and fines up to 7% of global turnover. Prohibitions and GPAI obligations already apply, and the Article 50 transparency duties apply from August 2, 2026, with the content-marking grace period ending December 2, 2026. The Digital Omnibus (Regulation (EU) 2026/1744, in force July 27, 2026) moved high-risk obligations to December 2, 2027 (Annex III) and August 2, 2028 (Annex I). See [AI Governance and Compliance](13-reliability-and-safety/04-ai-governance-and-compliance.md).

**Eval Awareness** - A model's tendency to detect when it is being evaluated and alter behavior accordingly, which confounds safety and capability benchmarks and argues for naturalistic, held-out test conditions. Anthropic's Opus 5.5 system card shows the size of the gap: 36% of automated-audit transcripts scored high on evaluation awareness, against 0.4% of internal Claude Code transcripts.

**Extended Thinking** - Claude's (3.7+) internal reasoning pass before the response. Older models such as Haiku 4.5 cap it with `thinking.budget_tokens`; newer models, Haiku 5.5 included, use adaptive thinking, where the model sizes its own reasoning under an effort setting. Opus 5.5 cannot turn it off, and Sonnet 5.5's lowest setting is `between_tools`. Thinking blocks on Fable 5.1, Opus 5.5, Sonnet 5.5, and Haiku 5.5 are bound to the model and conversation that produced them, so a fallback cannot replay them into another model (Haiku 5.5 silently drops blocks produced by the larger models), and editing anything before a thinking block returns a 400 for accounts created on or after August 31, 2026. Thinking tokens bill as output.

---

## F

**Few-Shot Prompting** - Including examples in the prompt to guide model behavior.

**Fine-Tuning** - Training a pre-trained model on task-specific data to improve performance. Platform availability is now a design risk: OpenAI is winding down self-serve fine-tuning, and even active customers cannot create new jobs from January 6, 2027. Existing fine-tunes are going too: under an October 14, 2026 notice, five `gpt-4o`, `gpt-4o-mini`, `gpt-4.1`, and `gpt-4.1-mini` snapshots, and fine-tunes built on them, shut down on January 14, 2027 (about three months' notice), and the named replacement, `gpt-6-luna`, cannot be fine-tuned. Keep the training data and eval set so you can rebuild the behavior on a base you control.

**FinOps for AI** - The discipline of measuring, attributing, and optimizing AI spend: cost per token/request/task, prompt caching, batch economics, showback and chargeback, and unit economics. See [FinOps and Token Economics](11-infrastructure-and-mlops/04-finops-and-token-economics.md).

**Forward Deployed Engineer (FDE)** - An engineer embedded with a customer to take an AI system from use case to production: scoping, integration, evals, security review, and handover. The breakout AI title of 2026, now split between frontier-lab roles (OpenAI FDE total compensation roughly $350K to $550K, per an outside estimate) and consulting roles hired from campus (BCG X: a $110K to $190K base, per secondary reporting, unconfirmed). Anthropic announced the Claude Frontier Academy on October 2, 2026: $100M to train 10,000 "Frontier Deployed Engineers" at customers and partners by the end of 2027, with a graded practical and a residency leading to a credential. See [Job Market Trends](00-interview-prep/06-job-market-trends-2026.md#forward-deployed-engineer-fde).

**Framework Churn** - The rapid, breaking evolution of AI orchestration frameworks (LlamaIndex, LangChain), which reshuffle package layouts and remove abstractions roughly yearly, breaking older tutorials and courses on a fresh install. Survive it by pinning/locking versions and learning primitives over APIs. See [Navigating Framework Churn](09-frameworks-and-tools/12-navigating-framework-churn.md).

**Frontier Safety Framework** - A lab's published policy for testing models for dangerous capabilities and the safeguards each capability level triggers: Anthropic's Responsible Scaling Policy (v3.4; ASL deployment protections, CB-1 and CB-2 chemical and biological determinations, periodic Risk Reports), OpenAI's Preparedness Framework (High and Critical levels; GPT-6 Astra was the first model OpenAI rated Critical for cybersecurity, and the GPT-6.1 Terra system card rates it Critical for cyber and High for bio), and Google DeepMind's Frontier Safety Framework (Google's Gemini 4 Argon model card says it "cannot rule out" reaching the bio and cyber critical capability levels). For builders the effect is practical: these levels decide which models you can access, which classifiers and fallbacks sit in front of them, and whether security or life-sciences work needs a verified-access program. See [AI Governance and Compliance](13-reliability-and-safety/04-ai-governance-and-compliance.md#frontier-lab-safety-frameworks).

**Full-Duplex Voice Agent** - A voice architecture where a speech model listens and speaks at the same time, handling turn-taking, backchannels, and interruptions, while delegating reasoning and tool calls to a separate text model (a talker-thinker split). OpenAI's `gpt-live-1` (API GA September 10, 2026) bills $0.05 per session minute plus the backend model's tokens, and the backend can be an OpenAI model or your own agent. A third option beside cascaded and speech-to-speech pipelines. See [Real-Time Voice Agents](18-voice-and-audio-agents/01-realtime-voice-agents.md).

**Function Calling** - LLM capability to output structured tool invocations rather than plain text.

---

## G

**GGUF** - The quantized model file format used by llama.cpp, Ollama, and LM Studio for local inference. Quant levels trade quality for size; Q4_K_M is the practical sweet spot. See [On-Device and Edge Deployment](04-inference-optimization/09-on-device-and-edge-deployment.md).

**GitSpawn** - An attack (Manifold Security, September 2, 2026) in which a repository's own `.git/config` (`core.fsmonitor`, `core.hooksPath`, or filters) makes a coding agent's background git calls run attacker commands outside the agent's sandbox, with no prompt. It needs the `.git` directory to arrive intact, as in an archive, shared drive, or sync folder; a normal clone does not copy repository config. Seven command-line agents were affected. Fixes shipped in goose 1.44.0 (CVE-2026-72718), Codex CLI 0.131.0, and Claude Code 2.1.196 for the fsmonitor path, but a second Claude Code path was still open on 2.1.252 and three agents were unpatched at Manifold's September 1 retest, so do not treat a version bump as the whole fix. The lesson: VCS metadata and agent config files are executable code, and the harness itself belongs inside the isolation boundary. See [Agentic Security and Sandboxing](07-agentic-systems/09-agentic-security-and-sandboxing.md#action-sandboxing-microvms-and-containers).

**Goodput** - The request rate a serving system sustains while meeting both its time-to-first-token and inter-token-latency SLOs. The right objective for serving decisions (disaggregation, batching, speculative decoding, new hardware), because peak throughput at batch sizes that blow the latency SLO is not usable capacity. See **Disaggregated Serving**.

**Graph Engineering** - Designing an agent system as a topology of specialized nodes, routed edges, and a shared state object rather than tuning a single agent's loop. The successor vocabulary to loop engineering, surging from mid-2026. The key design axis is static topology (inspectable, testable, replayable) versus dynamic topology (adaptive, harder to reason about). See [Research Radar](RESEARCH-RADAR.md#14-graph-engineering-and-the-orchestration-consensus).

**Grok 4.7** - The current Grok frontier model from SpaceXAI (xAI), released September 21, 2026 as `grok-4.7`. $2/$6 per 1M with a 500K context window; once a prompt reaches 200K, every token in the request bills at double. Model IDs xAI retired on May 15, 2026 redirect to `grok-4.3` and bill at its rates, so check what a pinned legacy ID actually serves. See [Model Taxonomy](02-model-landscape/01-model-taxonomy.md).

**Grounding** - Connecting LLM responses to factual sources to reduce hallucination.

**GRPO (Group Relative Policy Optimization)** - The RL algorithm behind DeepSeek-R1: drops PPO's value/critic network and computes advantage from the reward spread within a sampled group of completions. Cheaper than PPO; variants (Dr.GRPO, DAPO, GSPO) fix its length bias and zero-variance collapse. See [Training Reasoning Models](03-training-and-adaptation/08-rlvr-and-reasoning-models.md).

**Guardrails** - Input/output validation to prevent harmful or off-topic responses.

---

## H

**Hallucination** - Model generating plausible but factually incorrect information.

**Harness Engineering** - Designing the deterministic driver code around an agent (context assembly, tool execution, budgets, stop conditions, durable state, observability) rather than tuning the model itself. The harness is the kernel; the model is the policy. See [Loop Engineering](07-agentic-systems/12-loop-engineering.md).

**Harness (Scaffold) Variance** - The 10-20 point swing in benchmark scores produced by the same model weights under different prompts, tool access, reasoning effort, or agent scaffolds. Why provider self-reports are not comparable across labs, and only same-harness numbers can be compared. Always cite effort and who ran it: on Terminal-Bench 4.0, Claude Fable 5.1 scores 57.88% at max effort on the tbench.ai leaderboard but 55.8% in Anthropic's own run. See [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md).

**HNSW (Hierarchical Navigable Small World)** - Graph-based algorithm for approximate nearest neighbor search in vector databases.

**Human-in-the-Loop (HITL)** - Patterns for human oversight, approval, or correction of AI outputs.

---

## I

**In-Context Learning** - Model adapting to task based on examples in the prompt without weight updates.

**Indirect Prompt Injection** - A prompt-injection attack delivered through content the agent reads (a web page, document, tool result) rather than the user's direct input. Red-team studies and an impossibility result suggest it cannot be fully prevented, shifting defense toward least-privilege and containment. See [Agentic Security and Sandboxing](07-agentic-systems/09-agentic-security-and-sandboxing.md).

**Inference** - Running a trained model to generate predictions/outputs.

---

## J

**JSON Mode** - LLM output mode that guarantees valid JSON structure (legacy). Superseded by **Structured Outputs** in newer APIs.

---

## K

**KV Cache** - Cached key-value pairs from attention computation. Enables efficient autoregressive generation.

**KV-Aware Routing** - Load balancing that sends each request to the replica already holding the longest matching prefix in its KV cache, instead of round-robin, so prefix-cache hits survive scale-out. NVIDIA Dynamo (v1.5.0, September 21, 2026, with pluggable scorer and picker policies) and llm-d's Endpoint Picker (moved into llm-d from the Kubernetes Gateway API Inference Extension in August 2026) implement it above vLLM, SGLang, and TensorRT-LLM. Balance cache affinity against load, or the warmest replica becomes a hot spot. See [Serving Infrastructure](04-inference-optimization/06-serving-infrastructure.md#orchestration-above-the-engine-dynamo-and-llm-d).

---

## L

**LangChain** - Framework for building LLM applications with chains, agents, and integrations.

**Leaderboard Illusion** - The critique (Cohere et al., arXiv:2504.20879) that crowd-preference leaderboards like LMArena (now Arena, at arena.ai) are distorted by private best-of-N testing, unequal data access, and silent model deprecation. Contested in magnitude by LMArena; the practical takeaway is to read style-controlled Elo with confidence intervals and treat Arena as preference, not correctness.

**LiveCodeBench** - Benchmark evaluating coding models on real-world problems from competitive programming platforms. More reliable than HumanEval for production coding tasks, though Artificial Analysis moved it to its legacy set with Intelligence Index v4.3 (September 2026).

**LlamaIndex** - Data framework focused on document processing and retrieval for LLM applications.

**LLM-as-Judge** - Using an LLM to evaluate outputs from another LLM. Typed decision models that return a verdict with a probability instead of text are a cheaper option for yes/no and score judgments, at the cost of no written rationale, and their errors correlate with LLM judges', so a human-labeled slice still audits both. See **Decision Model (System One Model)**.

**Long-Context Cliff** - A pricing rule where the whole request moves to a higher rate once the prompt crosses a threshold, not just the tokens past it. OpenAI applies it above 272K input tokens (GPT-6 Astra goes from $10/$50 to $20/$75 per 1M, GPT-6.1 Terra from $1/$5 to $2/$7.50), xAI doubles every token at 200K, and Gemini 3.1 Pro Preview steps up above 200K. Anthropic is no longer an exception by vendor: Claude Haiku 5.5 (October 7, 2026) bills the whole request at 5x ($0.50/$2.50 instead of $0.10/$0.50) once the prompt passes 100K tokens, and cache reads and writes count toward that threshold, while Opus 5.5, Sonnet 5.5, and the other Claude 4.6-and-later models stay flat to 1M. Model the cliff per model before routing long prompts, and set compaction and retrieval budgets below it. See [Pricing and Costs](02-model-landscape/03-pricing-and-costs.md).

**Loop Engineering** - The discipline of designing and continuously improving the control loops that wrap an agent (the trigger, the inner reason-act-observe loop, a verification loop, event-driven invocation, and an eval-driven improvement loop) instead of hand-prompting the model each turn. See [Loop Engineering](07-agentic-systems/12-loop-engineering.md).

**Loopmaxxing** - The anti-pattern of assuming that more iterations automatically solve a task. It fails on goals with no verifiable exit condition, so the loop never converges and spend runs away. The multi-step descendant of token-maxxing. See [Loop Engineering](07-agentic-systems/12-loop-engineering.md).

**LoRA (Low-Rank Adaptation)** - Parameter-efficient fine-tuning that trains small adapter matrices instead of full model weights.

---

## M

**Managed Agents** - Provider-hosted agent runtimes that supply the harness (persistence, memory, skill loading, sandbox lifecycle, durable execution, identity scoping) so you supply only the agent logic. As of November 2026, all three major options are still pre-GA: Claude Managed Agents (beta; session budgets, agents-as-code via `ant apply` and a committed lockfile; also on Claude Models on Google Cloud since October 22), OpenAI's Agents API (the Codex harness as a managed service, public beta since September 10, 2026, launched with US data residency only and no zero data retention; since October 9 its hosted sandboxes bill only during active turns), and Amazon Bedrock Managed Agents built on it (public preview since September 29, 2026). Renting the loop does not give exactly-once semantics for your own side-effecting tools. See [Durable Execution](07-agentic-systems/11-durable-execution.md#managed-agent-runtimes-renting-the-durable-loop).

**MCP (Model Context Protocol)** - Open protocol for connecting models to tools, data, and prompts. Launched by Anthropic in November 2024; governance moved to the Linux Foundation's Agentic AI Foundation in December 2025; supported by Anthropic, OpenAI, Google, Microsoft, and AWS. Streamable HTTP transport and OAuth 2.1 authorization arrived in the 2025-03-26 revision (no spec release is named "MCP 2.0"; the 2.x numbers belong to the SDKs, from July 2026). The current revision, 2026-07-28, removed protocol-level sessions to make the core stateless. The next release moved from December 2026 to a planned March 30, 2027 (SEP-3398), with releases generally on the last Tuesday of March and September after that. Every extension now sits at one of three maturity stages (SEP-3392: Experimental, Beta, Stable), and a Standards Track SEP can later promote a Beta or Stable extension into the core specification. MCP Apps and enterprise auth are Stable, while Tasks, Skills, and Server Cards are Beta and can still break. See [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md#mcp-2026-07-28-the-stateless-rewrite).

**Memory Poisoning** - An attack that plants malicious or false entries into an agent's long-term memory so they resurface and influence future sessions. Added to the OWASP 2026 Agentic Top 10 as ASI06. Defense favors provenance at write time over sanitization at read time. See [Research Radar](RESEARCH-RADAR.md).

**Mixture of Agents (MoA)** - Ensemble pattern where multiple agents contribute to a synthesized response.

**Mixture of Experts (MoE)** - An architecture in which a learned router sends each token through a few expert feed-forward blocks out of many, so a model can carry a very large total parameter count while activating only a fraction per token. Open-weight models are now quoted as total/active (Xiaomi MiMo-V2.6-Pro 1.02T/42B, Z.ai GLM-5.3 744B/40B). Active parameters set compute per token; total parameters set memory, so serving still needs memory for every expert and usually expert parallelism. See [LLM Internals](01-foundations/01-llm-internals.md#mixture-of-experts-moe).

**Model Routing** - Choosing which model serves each request by task, cost, latency, capability, or semantics, often with a cascade (cheap model first, escalate on low confidence) and cross-provider fallback. See [AI Gateways and Model Routing](11-infrastructure-and-mlops/03-ai-gateways-and-model-routing.md).

**Multi-Tenancy** - Serving multiple customers from shared infrastructure with data isolation.

---

## N

**NVFP4 / MXFP4** - 4-bit floating-point weight formats with fine-grained block scaling: NVIDIA's NVFP4 on Blackwell and Rubin, and MXFP4, the Open Compute Project microscaling format that AMD quotes for its racks. NVFP4 or MXFP4 weights with an FP8 KV cache is the headline serving configuration of 2026, but the win depends on the kernel path: NVIDIA's own Dynamo recipe for Nemotron 3.5 Lightning ran BF16 80% faster than NVFP4 on Marlin kernels and 12% faster on CuTeDSL. Benchmark your stack before assuming 4-bit is faster. See [LLM Infrastructure](11-infrastructure-and-mlops/01-llm-infrastructure.md).

---

## O

**o3** - OpenAI's o-series reasoning model that followed o1 (o3-mini in January 2025, o3 in April 2025), which allocated test-time compute through internal chain-of-thought. Now legacy: Microsoft Foundry retires o3 on November 19, 2026, and the `o3-2025-04-16` API snapshot retires December 11, 2026, with GPT-5.6 models named as replacements.

**OAuth Mix-Up** - An attack on a client connected to several OAuth-protected servers: a malicious server tricks it into sending an authorization code or token meant for another server's issuer. In the MCP SDKs it was CVE-2026-104850 (TypeScript) and GHSA-qx49-fqc8-xw99 (Python), CVSS 7.5, fixed in TypeScript 1.31.0/2.2.0 and Python 1.30.0/2.2.0; machine-to-machine providers also need an `expectedIssuer`, and credentials saved before the upgrade stay exposed until cleared. Bind every token to its issuer and server. See [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md#oauth-mix-up-the-multi-server-client-is-the-confused-deputy).

**OCR (Optical Character Recognition)** - Extracting text from images or scanned documents.

**OpenHands** - Open-source autonomous software engineering agent (formerly OpenDevin). Supports multiple backend LLMs, runs in a Docker sandbox.

**OSWorld** - Benchmark of computer-use agents completing real tasks across desktop apps in a virtual machine. OSWorld-Verified is saturated; OSWorld 2.0 (XLANG) makes the binary score (task fully completed) primary and reports partial credit beside it, and the two diverge sharply: on XLANG's leaderboard (updated October 15, 2026) Claude Opus 5 at max effort still leads end-to-end models at 44.33% binary but 77.67% partial on the v2.1 task set, while Simular's Sai agent framework on Opus 5 posts the highest partial score (79.38%) at a lower 41.67% binary. The leaderboard has no rows yet for the 5.5-generation Claude models, GPT-6, or Gemini 4. Vendor launch posts often quote partial scores on their own subsets (Anthropic reports Opus 5.5 at 81.8% and Haiku 5.5 at 72.4% on an offline subset, both partial), so check which number and which task set you are reading. See **Computer-Use** and [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md#agentic-and-tool-use).

---

## P

**pass^k** - Agent reliability metric: the fraction of tasks solved on all k independent attempts (versus pass@k, solved on at least one). Exposes the reliability cliff where an agent at ~60% pass@1 can drop to ~25% pass^8. The production-relevant consistency signal.

**Plugin4Shell** - A plugin supply-chain flaw disclosed by AIR Security on September 17, 2026. Coding agents installed plugins pinned to a 40-character commit SHA without checking that the checked-out tree matched the pin, and git prefers a branch whose name equals the SHA, so on hosts that accept hash-shaped branch names a repository owner could swap already-approved code and background updates pulled it with no prompt. Fixed in Claude Code 2.1.179 and Codex 0.146.0; GitHub Copilot was unpatched at disclosure. A pin is a control only if you verify it after checkout. See [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md#agent-plugins).

**Prefix Caching** - Reusing KV cache for common prompt prefixes across requests.

**Prompt Caching** - Reusing the provider's KV cache for a repeated prompt prefix. Writes cost a premium (1.25x input on Anthropic's 5-minute cache, 2x on its 1-hour cache; on OpenAI GPT-5.6 and later, 1.25x for a 30-minute entry or, since October 20, 2026, 2.5x for a 12-hour entry) and reads cost a fraction: 0.1x on most models (Claude Haiku 5.5 included), 0.05x on Claude Opus 5.5, Claude Sonnet 5.5 (cut from 0.1x on October 7, 2026), GPT-6.1 Sol, and GPT-6.1 Terra, and 0.025x on Claude Fable 5.1. Gemini 4 Argon supports only implicit caching. A single reuse covers a 1.25x write premium, and a 2x or 2.5x write needs two, so choose the TTL from the gap between requests. Agent loops gain the most because they are overwhelmingly input: Anthropic reports a 324:1 input-to-output token ratio in Claude Code. See [Pricing and Costs](02-model-landscape/03-pricing-and-costs.md).

**Prompt Injection** - Attack where malicious input manipulates LLM behavior.

---

## Q

**QLoRA** - LoRA combined with 4-bit quantization for memory-efficient fine-tuning.

**Quantization** - Reducing model precision (e.g., FP16 to FP8, INT4, or NVFP4) to decrease memory and improve speed.

---

## R

**RAG (Retrieval-Augmented Generation)** - Pattern that retrieves relevant documents to provide context for LLM generation.

**RBAC (Role-Based Access Control)** - Access control based on user roles with predefined permissions.

**ReAct** - Agent pattern alternating between Reasoning and Acting steps.

**Reasoning Effort** - The request-level control over how much a reasoning model thinks before answering, now the main cost and latency knob: `reasoning.effort` on OpenAI, `effort` with adaptive thinking on Claude, `thinking_level` on Gemini. Levels run from low up to xhigh and max on the newest models. Defaults move between versions (Claude Opus 5.5 defaults to `medium` where Opus 5 defaulted to `high`), so pin effort per route, and never quote a benchmark score without its effort level. Reasoning tokens bill as output.

**Recursive Self-Improvement (RSI)** - An AI system improving the harness, training, or research process that produces its successors. The surging term of September 2026: OpenAI said it had reached an automated research intern and targets an automated AI researcher by March 2028. For engineers it is the improvement loop pointed at itself, which inherits every grader problem: freeze and version the eval set, keep a held-out audit set the loop never sees, and gate changes to the harness on human review. See [Loop Engineering](07-agentic-systems/12-loop-engineering.md#when-loop-4-turns-on-itself-recursive-self-improvement).

**Reranking** - Second-stage scoring to improve retrieval precision. Cross-encoders provide higher accuracy than bi-encoders.

**Responses API** - OpenAI's primary API for model calls, built-in tools (web search, file search over vector stores, remote MCP servers), and multi-turn state, paired with the Conversations API for durable threads. It replaced the Assistants API, which shut down on August 26, 2026: thread state had to move to Conversations, while vector stores carried over through the `file_search` tool. Newer models lean on the Responses API: GPT-6 Astra's tool calling requires it, and Pydantic AI 2.0 routes `openai:` models to it by default. Keep vendor-managed state behind your own abstraction anyway: Reusable Prompts, the migration target for assistant objects, is itself scheduled to shut down on November 30, 2026. See [Navigating Framework Churn](09-frameworks-and-tools/12-navigating-framework-churn.md).

**Reward Hacking** - A policy maximizing its reward through a loophole instead of doing the intended task: special-casing tests, editing the grader, or reading answers from the environment. It now shows up in agents at work, not only in training: one study measured 30.5% spontaneous reward hacking by research agents on open-ended research-pipeline tasks (arXiv 2609.28614), and Anthropic found that a model variant trained on hackable environments replicated a real-world attack chain that its production models did not. Separate the grader from the producer, hold out audit sets, and remove exploitable environments. See [Loop Engineering](07-agentic-systems/12-loop-engineering.md).

**RLHF (Reinforcement Learning from Human Feedback)** - Training method using human preferences to align model behavior.

**RLVR (RL with Verifiable Rewards)** - The dominant post-training recipe for reasoning models: reward the policy with a programmatic verifier (math, code, or logic with a checkable answer) instead of a learned reward model, which largely sidesteps reward-model hacking. See [Training Reasoning Models](03-training-and-adaptation/08-rlvr-and-reasoning-models.md).

**RTEB** - A retrieval benchmark from the MTEB team (October 2025) that mixes open datasets with private held-out sets the maintainers evaluate, so a model cannot be tuned to the whole test. Prefer it to MTEB-only rankings when shortlisting embedding models, then decide on your own corpus. NVIDIA reports Nemotron 3 Embed 8B first on RTEB Multilingual at 78.5 (vendor-reported, July 2026). See [Embedding Models](06-retrieval-systems/03-embedding-models.md#model-selection-criteria).

---

## S

**Safeguard Fallback** - When a safety classifier blocks a request, the provider (or your router) completes it on a different, usually older model. Claude Fable 5.1 returns HTTP 200 with `stop_reason: "refusal"`, and a beta setting lets the server retry on Opus 4.8 or Opus 5; when Anthropic scored Opus 5.5, blocked cyber tasks ran on Opus 4.8. Two consequences: a benchmark score labeled "with fallback" belongs to a routed system, not one set of weights, and a router must branch on `stop_reason`, not HTTP status. Since September 24, 2026 Anthropic bills pre-output refusals in three categories. See [Model Taxonomy](02-model-landscape/01-model-taxonomy.md).

**Self-Consistency** - Sampling multiple reasoning paths and selecting most common answer.

**Semantic Search** - Finding documents by meaning rather than keywords, using embeddings.

**Service Tier** - The same model sold at different prices for different latency or priority. Batch and Flex bill 0.5x when you can wait; OpenAI's Fast tier (renamed from Priority on July 30, 2026) bills 2x, and Ultrafast 6x (GA on GPT-6 Astra, and on GPT-6.1 Sol for all API users since October 8, 2026, at $12/$60); Gemini Priority is about 1.8x and Bedrock Priority 1.75x; Anthropic sells fast mode on Opus 5.5 at 2x ($8/$40, out of research preview since October 15, 2026; on the Claude API, Claude Models on Google Cloud (Anthropic managed), and Foundry's Azure-hosted deployments), is removing Opus 4.8 fast mode on November 30, 2026, and no longer sells Priority Tier commitments. The tier multiplies the whole bill, so choose it per route and record it on every request. See [Pricing and Costs](02-model-landscape/03-pricing-and-costs.md).

**Speculative Decoding** - A cheap drafter proposes several tokens and the target model verifies them in one forward pass, cutting latency without changing the target's output distribution. Drafters in 2026: EAGLE-3 heads, native multi-token-prediction (MTP) heads, DFlash block-diffusion drafters, and DeepSeek's DSpark modules bundled into the checkpoint; vLLM no longer documents Medusa. Gains shrink as batch size grows, so measure at production concurrency. See [Speculative Decoding](04-inference-optimization/03-speculative-decoding.md).

**Speech-to-Speech (S2S)** - A voice-agent architecture where one multimodal model takes audio in and emits audio out directly, versus a cascaded STT to LLM to TTS pipeline. More natural and lower-latency, but less debuggable and controllable. See also **Full-Duplex Voice Agent** and [Real-Time Voice Agents](18-voice-and-audio-agents/01-realtime-voice-agents.md).

**State-Handle Hijacking** - Attack class named in the MCP 2026-07-28 specification. With protocol-level sessions removed, servers mint explicit state handles returned as ordinary tool arguments; an attacker who obtains or guesses a handle can read or modify another user's state. Mitigation: never treat a handle as authentication, use non-deterministic handles, and bind them server-side to the authenticated principal. See [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md).

**Structured Outputs** - A provider feature that constrains decoding so the output conforms to a supplied JSON Schema; OpenAI, Anthropic, and Google all offer it natively. Stricter than legacy JSON mode. Now the reliable extraction path on Opus 5.5, Sonnet 5.5, and Fable/Mythos 5.1, which return a 400 for forced tool use (`tool_choice` of `any` or `tool`); Haiku 5.5 accepts a forced tool call but skips thinking on that turn.

**Super Intelligence (SI)** - The term US federal agencies now use in place of "AI" in non-statutory documents, under Executive Order 14434, "Inaugurating the Era of Super Intelligence" (signed September 29, 2026, published October 2), which applies it to the maximum extent permitted by law. The definition is the existing statutory one for artificial intelligence (15 U.S.C. 9401(3)), so "SI" in federal guidance, procurement language, or a customer's questionnaire means what "AI" meant, not a new capability threshold, and earlier regulations and contracts keep their wording. NIST's AI standards center now calls itself CAISSI, and its October 16, 2026 "Key Practices for Incident Readiness and Response in Super Intelligence Training and Evaluation" (with the NSA and CISA) is voluntary guidance worth knowing for agent work: it scopes incidents to significant uninstructed actions, treats actions caused by prompt injection or a backdoored model as instructed (handled as ordinary cyber incidents), and recommends monitors and logs that agents cannot reach, monitors from different model families, tamper-resistant logs, and verified containment before restart. See **CAISSI** and [AI Governance and Compliance](13-reliability-and-safety/04-ai-governance-and-compliance.md).

**SWE-bench Verified** - Human-validated 500-issue subset of SWE-bench measuring resolution of real GitHub issues; the canonical coding benchmark of 2024-2026. Now saturated and partly contaminated. Its successor, SWE-Bench Pro, saturated in turn: on Scale's v2 (September 2026) Opus 5 scores 99.4% on the 642-task public set, so cite the 272-task private set (Opus 5: 81.6%). Read the harness before trusting a score. See [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md).

**System One Model** - The vendor and research-paper name for a decision model: a model that answers a fixed question with a typed value and a probability instead of reasoning in text. See **Decision Model (System One Model)**.

**System Prompt** - Instructions that set context and behavior for an LLM conversation.

---

## T

**Temperature** - Parameter controlling randomness of LLM outputs. Lower = more deterministic. Increasingly unavailable on reasoning models: GPT-6 Astra accepts no custom temperature or `top_p`, and Anthropic's API version `2026-10-21` returns a 400 for any `temperature`, `top_p`, or `top_k` on every Claude model, even at the default value. The new Anthropic SDK majors (Python 2.0.0, TypeScript 1.0.0, Go 2.0.0) send that version by default, so an SDK upgrade can break sampling code on models that accepted it the day before. Get consistency from structured outputs, pinned effort, and evals rather than from temperature 0.

**Terminal-Bench** - Benchmark of agents completing hard, multi-step tasks in a real terminal, now the headline comparison for agentic coding. Scores do not carry over between major versions, so 3.0 numbers say nothing about 4.0. On the tbench.ai 4.0 leaderboard (entries added October 6, 2026), Claude Opus 5.5 leads at 64.85% and Claude Sonnet 5.5 follows at 61.82%, both at max effort, with GPT-6 Astra and GPT-6.1 Sol tied at 58.18%. Anthropic's own runs report 66.4% for Opus 5.5 and 70.6% for Sonnet 5.5 at xhigh (vendor-reported), a reminder that the same model scores differently in its vendor's harness. Always cite version, effort, and who ran it. See **Harness (Scaffold) Variance** and [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md#coding).

**Test-Time Compute (Inference-Time Scaling)** - Spending more compute at inference with the weights **frozen**: long chain-of-thought, best-of-N, self-consistency, search. Ubiquitous in production by 2026, with diminishing (sometimes negative) returns past a point. Contrast with Test-Time Training.

**Test-Time Training (TTT)** - Updating a model's **weights** at inference (often an ephemeral LoRA) on the test input, its augmentations, or retrieved neighbors, then predicting and discarding the update. Distinct from test-time compute, which leaves weights frozen. Research-stage in 2026; strongest on novel tasks like ARC and on long-context efficiency. See [Research Radar](RESEARCH-RADAR.md#12-test-time-training-learning-at-inference).

**Token** - Basic unit of text processing. Roughly 0.75 words or 4 characters in English.

**Tool Use** - LLM capability to invoke external functions/APIs.

**Transformer** - Neural network architecture based on self-attention. Foundation of modern LLMs.

---

## V

**Vector Database** - Database optimized for storing and searching high-dimensional vectors (embeddings).

---

## W

**Windsurf** - Former name of Cognition's AI-native IDE, renamed Devin Desktop in v3.0.12 on June 2, 2026. See **Devin Desktop**.

---

## X

**x402** - Open standard for paying per request over HTTP using the 402 Payment Required status code, with cards or stablecoins; governed by the x402 Foundation under the Linux Foundation since July 14, 2026. The commerce protocol most likely to touch a non-commerce agent, because it turns paid API and MCP tool calls into real spend that needs the same caps as token spend. See **Agentic Commerce Protocols**.

---

## Z

**Zero Data Retention (ZDR)** - A provider arrangement under which prompts and outputs are not stored after the request completes, often required for regulated data. Availability is per model and per product, not per vendor: Claude Fable 5.1 requires 30-day retention unless Anthropic authorizes ZDR, while Opus 5.5 and Sonnet 5.5 support it, and OpenAI's Agents API launched without ZDR. Check it before any technical comparison.

**Zero-Shot** - Prompting without examples, relying on model's pre-existing knowledge.

---

*See also: [PATTERNS.md](PATTERNS.md) for design pattern quick reference*
