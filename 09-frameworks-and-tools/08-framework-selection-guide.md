# Framework Selection Guide

The landscape of AI frameworks has consolidated significantly over the past year. Every major AI lab now ships an agent SDK, Microsoft merged AutoGen and Semantic Kernel into a unified Agent Framework (GA April 2026), the labs now also rent the agent loop as a managed runtime, and interoperability protocols (MCP, A2A) are table stakes. This guide provides the **Decision Matrix** for choosing your stack based on production requirements, team expertise, and system scale.

## Table of Contents

- [The Framework Landscape](#the-framework-landscape)
- [The Decision Matrix](#the-decision-matrix)
- [Build vs. Buy vs. Framework](#build-vs-buy-vs-framework)
- [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
- [Staff-Level Recommendation](#staff-level-recommendation)
- [Interview Questions](#interview-questions)

---

## The Framework Landscape

### Orchestration & Agent Frameworks

| Framework | Tier | Primary Value | Key Weakness |
|-----------|------|---------------|--------------|
| **LangGraph** | L1 (Core) | Precise state control, graph-based, checkpoint time travel | Complexity, steep learning curve |
| **DSPy** | L1 (Core) | Reliability & Optimization (MIPROv2, GEPA, Flex) | Upfront cost (compile runs per model) |
| **LlamaIndex**| L2 (Data) | Advanced Retrieval (RAG), ingestion, parsing | Logic flexibility; vendor focus shifted to parsing products (Sep 2026) |
| **CrewAI** | L3 (App) | Business process speed, enterprise RBAC, checkpointing (1.14+) | Hides failures |
| **MS Agent Framework** | L1 (Enterprise) | Unified .NET + Python, replaces AutoGen + SK, GA with LTS (Apr 2026) | Azure-first; fast weekly release cadence |
| **Pydantic AI / Mastra** | L1 (Typed) | Typed agents for Python (Pydantic AI 2.x) and TypeScript (Mastra) | Smaller ecosystems; see [chapter 11](11-pydantic-ai-and-mastra.md) |
| **Vercel AI SDK 7** | L2 (TS runtime) | Provider-neutral TS calls, durable `WorkflowAgent`, experimental `HarnessAgent` | Agent abstractions newer than its core |
| **Haystack 3** | L2 (Data + Agent) | Agent with lifecycle hooks, skills, tool-result offloading (3.0, Jul 2026) | 3.0 removed `ToolInvoker` and moved 30 components to separate packages |
| **Agno 3** | L2 (Agent) | Tool-result offloading, CodeMode (one sandboxed interpreter instead of wide tool schemas) | 3.0 (Aug 2026) required a database migration |

### Agent SDKs (Lab-Specific)

| Framework | Tier | Primary Value | Key Weakness |
|-----------|------|---------------|--------------|
| **Claude Agent SDK** | L1 (Agent) | Built-in tools, production agent loop, budgets | Requires Claude models |
| **OpenAI Agents SDK** | L1 (Agent) | Lightweight handoffs, guardrails, sandbox agents | OpenAI-centric; 0.x with behavior-changing minors |
| **Google ADK** | L1 (Agent) | Graph workflows (2.0), multi-language, native A2A + Google Cloud | Google ecosystem bias; Java still on 1.x |
| **AWS Strands Agents** | L1 (Agent) | AWS-native, model-agnostic, most-downloaded lab SDK on PyPI | Harness packages change default models between releases |

### Managed Agent Runtimes

| Runtime | Tier | Primary Value | Key Weakness |
|-----------|------|---------------|--------------|
| **Claude Managed Agents** | L1 (Hosted) | Sandboxes, session budgets, server-evaluated permission policy, agents-as-code (`ant apply`); Claude API, Claude Platform on AWS and Claude on Google Cloud | Beta; Claude only; second cost meter ($0.08 per running session-hour) |
| **OpenAI Agents API** | L1 (Hosted) | Managed Codex harness: compaction, recovery, subagents; hosted sandboxes bill only during active turns (since Oct 9) | Public beta; US residency only, no ZDR |
| **Bedrock Managed Agents** | L1 (Hosted) | The Agents API inside AWS with IAM and CloudTrail | Public preview; three US regions |

### Coding Agents

| Framework | Tier | Primary Value | Key Weakness |
|-----------|------|---------------|--------------|
| **Claude Code** | L1 (Coding) | Autonomous CLI coding agent, plugins, enterprise managed settings | Requires Claude models; closed source |
| **Codex CLI** | L1 (Coding) | Open-source (Apache-2.0) CLI, cloud environments, code review | OpenAI-centric defaults |
| **Cursor / Devin Desktop** | L2 (IDE) | Tight IDE + agent integration, cloud agents | Closed-source infra; Cursor is part of SpaceX, and Devin Desktop (Windsurf until June 2026) belongs to Cognition |
| **GitHub Copilot** | L2 (IDE + platform) | GitHub-native coding agent and review; Copilot Fusion (GA Oct 28, 2026) routes across models with Standard, High and Max effort | Strongest inside GitHub; computer use and dynamic workflows launched as public previews (Oct 1) |
| **OpenCode / OpenHands** | L2 (Coding) | Open-source, any model | You own hosting, sandboxing and upgrades |

Semantic Kernel is not listed as a choice for new work: Microsoft Agent Framework is its successor, and existing SK users should plan the migration.

Adoption data backs the "lab SDK at the leaf, neutral orchestrator at the core" advice. In the 30 days to Nov 1, 2026 on PyPI: `langchain` 170.5M, `langgraph` 44.8M, `strands-agents` 37.1M, `claude-agent-sdk` 29.8M, `openai-agents` 12.5M, `google-adk` 10.1M, `llama-index-core` 6.6M, `pydantic-ai` 5.4M, `dspy` 5.3M, `crewai` 2.4M, `agno` 1.8M, `agent-framework-core` 1.0M, `haystack-ai` 0.57M, `autogen-agentchat` 0.38M, `semantic-kernel` 0.30M. On npm in October: `ai` (Vercel AI SDK) 115.4M, `@anthropic-ai/claude-agent-sdk` 46.4M, `@langchain/langgraph` 14.0M, `@mastra/core` 8.2M, `@openai/agents` 7.5M (Mastra passed OpenAI's TypeScript SDK this month). The ranking at the top has not moved in months. CI installs and transitive dependencies inflate all of these, so read them as relative signals.

---

## The Decision Matrix

**Use this logic to select your stack:**

### Core Orchestration
1. **Is it a pure RAG app?** → **LlamaIndex** (keep it behind your own retrieval interface).
2. **Does it require long-running state/Human-in-the-loop?** → **LangGraph**.
3. **Is high reliability (99%+) and cross-model portability critical?** → **DSPy**.
4. **Are you a C#/.NET enterprise shop?** → **Microsoft Agent Framework** (replaces Semantic Kernel + AutoGen).
5. **Are you building high-level automations for business users?** → **CrewAI + Flows**.
6. **Is the service TypeScript-first?** → **Vercel AI SDK** for calls, **Mastra** or **LangGraph TS** for agents and workflows.

### Agent SDKs (choose based on your primary model provider)
7. **Building agents on Claude / Anthropic API?** → **Claude Agent SDK** (Python/TS, built-in tools for file/code/command).
8. **Building agents on OpenAI API?** → **OpenAI Agents SDK** (lightweight handoffs, guardrails, MCP support).
9. **Building agents on Google Cloud / Gemini?** → **Google ADK** (graph workflows, native A2A, Vertex AI deployment, multi-language).
10. **Need cross-vendor agent communication?** → Use **A2A protocol** on top of any framework above.
11. **Want the vendor to run the loop (compaction, recovery, sandboxes)?** → A **managed runtime**, after checking data residency, ZDR, and the session-time meter.

### Coding Agents
12. **Are you doing autonomous file-system level coding tasks?** → **Claude Code** or **Codex CLI** (terminal), **Cline** (VS Code).
13. **Need an open-source coding agent that works with any LLM?** → **OpenCode** (MIT) or **OpenHands** (Docker).
14. **Want the best IDE experience with AI?** → **Cursor** (closed, SpaceX), **Devin Desktop** (Cognition, formerly Windsurf), or **GitHub Copilot** if you live in GitHub.

---

## Build vs. Buy vs. Framework

As a Staff Engineer, you must resist **Framework Bloat**.

- **Use a Framework** when it solves a **Non-Trivial Computer Science Problem** (e.g., State persistence, Bayesian prompt optimization, Vector-Graph linking).
- **Build Custom (Thin Wrapper)** when you are just making simple calls to an LLM. Frameworks add latency, update-churn, and debugging overhead that isn't worth it for a single-turn agent.

**Name the pattern before the framework.** Between late June and late September 2026, independent frameworks shipped the same three primitives:

| Pattern | What it does | Shipped as |
|---|---|---|
| **Harness-first APIs** | The agent is a configurable harness (tools, hooks, instructions, settings bundled as capabilities), not a bare loop | Pydantic AI 2 capabilities, AI SDK 7 `HarnessAgent` (experimental), MAF Harness Agent, OpenAI Agents API (managed Codex harness) |
| **Tool-result offloading** | Large tool outputs go to a file or store; the context gets a pointer and read/search tools | Haystack 3.0 `ToolResultOffloadHook`, Agno 3.0 (results over 16,000 characters) |
| **Code-mode tools** | The model writes code that calls tools inside a sandboxed interpreter instead of one JSON call per tool | Agno CodeMode, MAF `HyperlightCodeActProvider` (Microsoft reports more than 60% fewer tokens on its workload), DSPy 3.4 `LocalInterpreter` |

A strong design answer describes these mechanisms and treats the framework as an implementation detail. If your framework lacks one, it is usually a few hundred lines to add.

---

## Anti-Patterns to Avoid

1. **Framework Tunneling**: Trying to force a complex logic flow into a framework that doesn't support it (e.g., using a pure RAG library for a coding agent).
2. **The Golden Hammer**: Using LangChain just because it's popular, when a 50-line Python script would be faster and cheaper.
3. **Ignoring Observability**: Deploying any framework without an LLMOps layer (LangSmith, Langfuse, Phoenix), ideally one that ingests OpenTelemetry so you can export your traces.
4. **Trusting Defaults**: Letting the framework pick the model, or the agent's permissions. In August and September 2026, the OpenAI Agents SDK (0.20.0), Claude Code (2.1.280 and 2.1.284), Codex CLI (rust-v0.159.1) and the Strands harness packages all changed their default model, which silently changed cost, latency and behavior for every caller that never set one. In October, Claude Code 2.1.302 started headless and Agent SDK sessions with no permission mode in `default` mode on third-party providers, with telemetry off, or on a fresh install, so the same CI job could behave differently per environment.

---

## Staff-Level Recommendation

For a modern, production-grade agentic system:
- **Orchestration**: LangGraph (for state and loops) or Microsoft Agent Framework (for .NET shops).
- **Agent SDK**: Match to your model provider: Claude Agent SDK (Anthropic), Agents SDK (OpenAI), ADK (Google), Strands (AWS). All support MCP for tool access.
- **Managed runtime**: Only for leaf agents where compaction, recovery and sandboxing save more than the lock-in costs, and only where residency and ZDR terms fit.
- **Optimization**: DSPy (to compile prompts for different model tiers).
- **Retrieval**: LlamaIndex (for multi-stage RAG).
- **Observability**: An OpenTelemetry-friendly tracer you can export from (LangSmith, Langfuse, Phoenix).
- **Cross-vendor agents**: A2A protocol for agent-to-agent coordination across organizational boundaries.
- **Autonomous coding**: Claude Code or Codex CLI (terminal), Cline (VS Code) for file-level editing tasks.
- **Open coding agent**: OpenCode or OpenHands for self-hosted or CI pipeline integration.

**The 2026 insights**:
1. Agentic coding tools (Claude Code, Codex, Cursor, OpenHands) are not replacements for orchestration frameworks. They are a **new category** that operates at the file-system level, above the LLM API but below the application logic. The AI SDK 7 `HarnessAgent` goes one step further and treats those harnesses as pluggable dependencies.
2. The protocol layer has matured: **MCP for agent-to-tool** and **A2A for agent-to-agent** are becoming infrastructure standards, not optional add-ons. Design your architecture to support both.
3. Every lab shipping its own agent SDK, and now its own managed runtime, creates a **vendor lock-in risk**. Mitigate by using MCP for tool access (portable across SDKs), A2A for agent coordination (vendor-neutral), and agent definitions kept as code in your repo.
4. Frameworks are converging on the same runtime primitives (harness-first APIs, tool-result offloading, code-mode tools), so the durable skill is knowing the primitives, not the class names.

---

## Interview Questions

### Q: Why do we see a trend towards "Programming" (DSPy) instead of "Prompting"?

**Strong answer:**
**Industrialization**. Prompt engineering is "Alchemy": it is inconsistent and does not scale. Programming LLMs via Frameworks like DSPy allows us to treat AI as a **Software Engineering discipline**. We can apply CI/CD, unit testing (metrics), and automated optimization. This moves AI from "Nondeterministic Magic" to a **Predictable Component** of a larger distributed system, which is a requirement for any mission-critical production environment.

### Q: If you had to build a system that works across OpenAI, Anthropic, and local open-weight models, how would you architect it?

**Strong answer:**
I would use **DSPy** for the prompt layer and **LangGraph** for the orchestration layer. DSPy's **Signatures** allow me to decouple the task definition from the model's specific behavior. I would then use a **Universal Model Gateway** (like LiteLLM or an internal proxy) to handle the different API formats. The gateway holds every provider key, so it is security-critical infrastructure: LiteLLM alone had an unauthenticated RCE (CVE-2026-37004, CVSS 9.8, fixed in 1.83.7) and an auth bypass (CVE-2026-49468, fixed in 1.84.0) in 2026, two PyPI releases (1.82.7 and 1.82.8) were compromised, and on October 21 a leaked admin key that was never revoked ended in seven malicious PyPI wheels and container images, live for about two hours, that harvested cloud credentials, provider keys and the proxy's master and virtual keys. Its stored-key exfiltration bug through `api_base` (CVE-2026-84377) is fixed only in specific patch releases on each minor line (for example 1.96.2), so a simple version floor is not enough. I pin an exact version from the advisory's fixed list, verify artifact hashes rather than trusting a tag, set `allow_client_side_credentials=false`, and keep the gateway off the public internet. The gateway must also stop injecting sampling parameters: GPT-6 Astra rejects custom `temperature`, and under Anthropic API version `2026-10-21` every Claude model returns 400 for `temperature`, `top_p` or `top_k`, even at default values. For tool access, I would use **MCP**: it is model-agnostic, so the same MCP servers work regardless of which LLM backend is active. If I need cross-team agent coordination, I would use **A2A** at the boundary layer. This stack ensures that if I need to switch from GPT-6.1 Sol to Claude Sonnet 5.5 for cost or latency reasons, I do not have to rewrite 50 prompts; I re-compile, re-run the eval suite, and update the config.

### Q: With every AI lab shipping its own agent SDK (Claude Agent SDK, OpenAI Agents SDK, Google ADK), how do you avoid vendor lock-in?

**Strong answer:**
The key is to **separate the orchestration layer from the model layer**. I use a framework-agnostic orchestrator like LangGraph or a thin custom wrapper for the core workflow logic. Model-specific SDKs are useful for prototyping or when you are committed to a single provider, but for production multi-vendor systems, I keep the model interaction behind an abstraction (LiteLLM gateway or DSPy signatures). For tool access, **MCP** provides portability: the same MCP server works with any SDK. For agent coordination, **A2A** provides vendor-neutral agent-to-agent communication. The practical rule: use lab-specific SDKs and managed runtimes at the leaf nodes (individual agent implementations) but keep the orchestration graph, the agent definitions and the eval suite vendor-neutral and in my repo. Lock-in also shows up in lifecycle: the same model now retires on different dates on different clouds, so I track lifecycle per (model, platform) pair, not per model.

---

## References
- Gartner. "Gartner Predicts 40% of Enterprise Apps Will Feature Task-Specific AI Agents by 2026, Up from Less Than 5% in 2025" (press release, Aug 26, 2025): https://www.gartner.com/en/newsroom/press-releases/2025-08-26-gartner-predicts-40-percent-of-enterprise-apps-will-feature-task-specific-ai-agents-by-2026-up-from-less-than-5-percent-in-2025
- Thoughtworks. Technology Radar (Vol. 34, Apr 2026): https://www.thoughtworks.com/radar
- Microsoft. "Agent Framework Overview": https://learn.microsoft.com/en-us/agent-framework/overview/
- Anthropic. "Agent SDK reference: Python": https://code.claude.com/docs/en/agent-sdk/python
- Google. "Agent Development Kit": https://google.github.io/adk-docs
- OpenAI. Agents SDK releases: https://github.com/openai/openai-agents-python/releases
- OpenAI. "Agents API overview" (beta, Sep 2026): https://developers.openai.com/api/docs/guides/agents-api/overview
- GitHub. "Route and orchestrate models with GitHub Copilot Fusion" (Oct 28, 2026): https://github.blog/changelog/2026-10-28-route-and-orchestrate-models-with-github-copilot-fusion/
- Vercel. "AI SDK 7" (Jun 2026): https://vercel.com/blog/ai-sdk-7
- deepset. Haystack 3.0 release notes (Jul 2026): https://github.com/deepset-ai/haystack/releases/tag/v3.0.0
- Agno 3.0 release notes (Aug 2026): https://github.com/agno-agi/agno/releases/tag/v3.0.0

---

*Next: [Navigating Framework Churn](12-navigating-framework-churn.md)*
