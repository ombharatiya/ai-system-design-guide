# Multi-Agent Orchestration

Complex systems are rarely one agent. They are teams of specialized agents. Orchestration has matured from "Blind Managers" to **Hierarchical Supervisors**, **Dynamic Swarms**, and **Cross-Vendor Agent Networks** enabled by interoperability protocols like A2A. Gartner projected in August 2025 that 40% of enterprise applications will feature task-specific AI agents by the end of 2026, up from less than 5% in 2025.

## Table of Contents

- [Why Multi-Agent?](#why-multi-agent)
- [The Supervisor Pattern](#the-supervisor-pattern-hierarchical)
- [Swarms (The OpenAI Pattern)](#swarms-the-openai-pattern)
- [The Pipeline Pattern](#the-pipeline-pattern-workflows-as-code)
- [Graph-Based Orchestration (2026 Dominant Pattern)](#graph-based-orchestration-2026-dominant-pattern)
- [Cross-Vendor Agent Orchestration via A2A](#cross-vendor-agent-orchestration-via-a2a)
- [The 2026 Framework Landscape for Multi-Agent](#the-2026-framework-landscape-for-multi-agent)
- [State Management in Agent Teams](#state-management)
- [Peer-to-Peer (P2P) Debate](#peer-to-peer-p2p-debate)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Why Multi-Agent?

A single agent with 50 tools experiences **Cognitive Load**.
1. **Specialization**: A "Code Agent" can use a model optimized for Python, while a "Search Agent" uses a model optimized for RAG.
2. **Parallelism**: Multiple agents can work on independent sub-tasks simultaneously.
3. **Decoupled Evaluation**: You can evaluate the "Writer Agent" separately from the "Researcher Agent."

---

## The Supervisor Pattern (Hierarchical)

The most common enterprise pattern as of 2026.

- **The Supervisor**: A high-reasoning model (Claude Opus 5.5 or Fable 5.1, GPT-6 Astra or GPT-6.1 Sol) that decomposes the user prompt and delegates to workers.
- **Workers**: Fast, cost-efficient models (Claude Sonnet 5.5 or Haiku 5.5, GPT-6 Luna or GPT-6.1 Terra, Gemini 3.8 Flash) that perform the work.
- **Reviewer**: A separate agent that validates the consolidated output against the supervisor's original plan.

**Architecture**: LangGraph remains the dominant framework for implementing these state-aware hierarchical loops. The Claude Agent SDK, Google ADK, and Microsoft Agent Framework all support this pattern natively as of 2026.

**Re-check the tier split before you pay for it.** The price gap that justified cheap workers under an expensive supervisor has narrowed: Claude Sonnet 5.5, GPT-6 Sol, and GPT-6.1 Sol all list at $2/$10 per 1M tokens, Opus 5.5 dropped to $4/$20, and Anthropic's own (vendor-reported) agentic evals put Sonnet 5.5 within a few points of Opus 5.5. If the worker tier is nearly as capable and only 2x cheaper, the coordination overhead of a supervisor tree can cost more than it saves. October reopened the gap at the bottom: Claude Haiku 5.5 lists at $0.10/$0.50 for prompts up to 100K tokens (5x on the whole request above that) and GPT-6.1 Terra at $1/$5, so narrow, short-context worker tasks are cheap again while long-context workers can cross a price cliff. Re-run the routing eval on each model generation.

**The supervisor is now also a product.** Coding products and managed runtimes ship the coordinator for you: Cursor's Projects beta (September 10, 2026) has a coordinator agent plan the work and delegate to as many parallel subagents as needed, and Cursor reports (vendor figure, internal use) that new Projects users merge 30% more PRs; OpenAI's Agents API caps fan-out with `max_concurrent_subagents` (default 6); Claude Managed Agents adds an `advisor` entry to the multiagent roster that the primary thread can consult mid-turn. A platform-level concurrency cap is the budget control the supervisor pattern always needed, so set it deliberately rather than inheriting the default.

---

## Swarms (The OpenAI Pattern)

Popularized by OpenAI's experimental Swarm library in late 2024 and carried into the OpenAI Agents SDK, **Swarms** focus on "Handoffs."

- One agent "Hands off" the conversation to another.
- **Key concept**: `Handoff(TargetAgent)`.
- **Benefit**: No central "Manager" bottleneck. The conversation flows naturally between specialized entities.

---

## The Pipeline Pattern (Workflows as Code)

A pipeline fixes the stages in code and lets agents fill them: stage 1 gathers, stage 2 drafts, stage 3 verifies, with structured results passed between stages and explicit checkpoints. The model decides *how* to do each stage; the code decides *which* stages run and in what order.

The distinction that matters is **who owns the control flow**:

| Approach | Control flow owned by | Strength | Weakness | Shipping example (2026) |
|----------|----------------------|----------|----------|-------------------------|
| **Workflow as code** | Your program | Deterministic, auditable, testable stages; human checkpoints at fixed points | Rigid when the task shape varies | GitHub Copilot dynamic workflows (public preview October 1): sequential or parallel stages, structured results between stages, subagents verifying each other, pauses for human review |
| **Model-driven delegation** | A coordinator model | Adapts to tasks whose shape is unknown up front | Harder to bound cost and to audit | GitHub's `/fleet`, Cursor Projects |
| **Cascade with a quality gate** | Code, with a model-scored gate | Cheap model drafts; escalate only when the gate fails | Gate calibration becomes the critical component | GitHub Copilot Fusion's Cascade workflow (GA October 28, grown out of the September 30 HydraFusion research preview) |

Use workflows-as-code when the stages are known and the audit trail matters; use model-driven delegation when they are not; and put a cascade in front of either when most inputs are easy.

---

## Graph-Based Orchestration (2026 Dominant Pattern)

The architectural momentum in 2026 has shifted decisively toward **graph-based orchestration**, where agent workflows are modeled as directed graphs with typed state.

### Why Graphs Won

- **Explicit control flow**: Nodes are agents or functions; edges define transitions, including conditional branches and loops
- **Visualizable**: Teams can inspect and debug the workflow as a diagram
- **State-aware**: Typed state objects pass through the graph, enabling checkpointing and resumption

### Framework Support

| Framework | Graph Model | Key Differentiator |
|-----------|-------------|-------------------|
| **LangGraph** (~44.3k stars) | Imperative DAG with typed state | Most mature, broadest community |
| **Google ADK** (~22.1k stars) | Graph workflows (2.x) with built-in A2A | Native Google Cloud integration; ADK TypeScript 2.0 deprecates Sequential, Parallel, and Loop agents in favor of graphs |
| **Microsoft Agent Framework** | Workflow graphs (sequential, concurrent, handoff) | Unified .NET + Python, enterprise governance |
| **Claude Agent SDK** | Supervisor-based hierarchical trees | Built-in tools (bash, editor), production-ready |

### The Paperclip Pattern (Hierarchical Agents at Scale)

A notable 2026 development is **Paperclip** (44,900 GitHub stars within three weeks of its March 2026 launch). It uses a hierarchical model where a CEO agent receives a top-level goal, decomposes it, and delegates to manager agents who spawn and coordinate worker agents. This pattern demonstrates how deeply hierarchical multi-agent trees can handle complex real-world tasks.

---

## Cross-Vendor Agent Orchestration via A2A

The **Agent-to-Agent (A2A) protocol** (see [Tool Use and MCP](03-tool-use-and-mcp.md#agent-to-agent-protocol-a2a)) enables a new multi-agent pattern: **cross-vendor orchestration**. Before A2A, multi-agent systems required all agents to share the same framework and runtime. Now:

1. **Agent Discovery**: An orchestrator finds specialist agents via their **Agent Cards** (JSON metadata describing capabilities, served at `/.well-known/agent-card.json`)
2. **Task Delegation**: The orchestrator sends a message (`SendMessage`, over JSON-RPC, gRPC, or REST `POST /message:send`), and the remote agent decides whether to answer directly or open a task
3. **Async Progress**: The remote agent streams status updates back; the orchestrator can delegate to other agents in parallel
4. **Result Collection**: Final artifacts are returned and integrated into the orchestrator's state

**Production example**: A procurement system where the orchestrator (LangGraph) delegates compliance checking to a specialized agent (Google ADK), inventory lookup to an MCP-connected tool, and contract generation to a CrewAI crew, all communicating via A2A and MCP respectively.

**When several clients steer one remote task**, lost updates become the failure mode. The unreleased A2A v1.1 work adds a monotonically increasing `Task.generation` so clients can detect missed events and do compare-and-set (`if_generation_match`, rejected with HTTP 412 on a stale generation). Until it ships, serialize steering through one owner. A2A v1.0.1 (May 28, 2026) is the current spec release; A2A joined the Agentic AI Foundation in August 2026.

---

## The 2026 Framework Landscape for Multi-Agent

Every major AI lab now ships an agent framework. The multi-agent orchestration landscape as of early November 2026 (versions as of November 6, star counts as of November 2):

| Framework | Provider | Multi-Agent Model | Status |
|-----------|----------|-------------------|--------|
| **LangGraph** | LangChain | Graph-based, most flexible | Production; LangGraph 1.2.20 (~44.3k stars) |
| **Claude Agent SDK** | Anthropic | Supervisor trees with built-in tools | GA (Python + TypeScript) |
| **Google ADK** | Google | Graph workflows with A2A native support | Python 2.13.0, TypeScript 2.4.0, Go 2.7.1, Java 1.14.0 (Go 2.7.0 changed telemetry and call-limit defaults; see [Navigating Framework Churn](../09-frameworks-and-tools/12-navigating-framework-churn.md)) |
| **Microsoft Agent Framework** | Microsoft | Workflows + group chat patterns, A2A and MCP interop | GA April 2, 2026; Python 1.24.0, .NET 1.27.0; AutoGen is in maintenance mode |
| **OpenAI Agents SDK** | OpenAI | Handoff-based swarms with guardrails | Python 0.24.0, TypeScript 0.21.0 (~30.4k stars); still pre-1.0 |
| **CrewAI** | CrewAI Inc. | Role-based crews with Flows | 1.15.35 (~60.4k stars; vendor claims 60%+ of Fortune 500) |
| **Smolagents** | HuggingFace | Lightweight, open-source | 1.26.0 (May 29, 2026); no release since |

**Key trend**: No single framework excels at all four multi-agent patterns (supervisor, swarm, pipeline, debate). Teams increasingly combine frameworks, e.g., LangGraph for complex orchestration with CrewAI for business-user-facing automations. The other trend is the managed runtime: OpenAI's Agents API (public beta), Claude Managed Agents (beta), and Amazon Bedrock Managed Agents (public preview) host the harness itself, which moves the build-versus-rent decision up a layer (see [Durable Execution](11-durable-execution.md)). None of the three is GA as of early November 2026, so check beta terms (API stability, data retention, residency) before you rent; AWS's own AgentCore managed harness is the GA exception (since June 17, 2026).

---

## State Management

The biggest challenge in multi-agent systems is the **Shared Blackboard**.

1. **Local State**: Context only visible to a specific agent.
2. **Global State**: Shared memory (e.g., the final draft) visible to all.
3. **Write Conflicts**: When two agents try to modify the same Global State.
   - **Best practice**: Use **Transactional Handoffs**. An agent can only write to the global state when it "Owns" the lock. Where locks are too coarse, use optimistic concurrency: version the state and reject writes against a stale version, the same compare-and-set idea A2A v1.1 applies to remote tasks.

---

## Peer-to-Peer (P2P) Debate

For high-accuracy tasks (e.g., Legal or Medical), we use **Agentic Debate**.
- **Agent A**: Proposes an answer.
- **Agent B**: Tries to find flaws in Agent A's answer.
- **Agent A**: Refines the answer based on B's critique.
- **Result**: Convergence on a higher-quality result than any single agent could produce.

**Use a different model family for the critic.** Same-family critics share blind spots. GitHub's HydraFusion "critique" mode (research preview, September 30, 2026) baked this in: a critic from a different model family reviewed the draft and the drafter revised once. It went GA on October 28 inside **Copilot Fusion**, whose Critique workflow has one model draft, a read-only critic review, and the original model revise (alongside Single and Cascade workflows at Standard, High, or Max effort). Capping the exchange at one revision is the other half of the lesson; open-ended debate is loopmaxxing with two agents.

---

## Interview Questions

### Q: What are the main failure modes of a "Supervisor" multi-agent architecture?

**Strong answer:**
The primary failure mode is **Decomposition Failure**. If the Supervisor agent breaks a task into sub-tasks that are logically inconsistent or have hidden dependencies, the workers will produce correct answers to the *wrong questions*. The standard fix is **Iterative Planning**: the Supervisor must get "Confirmation of sub-task feasibility" from the workers before they begin execution. Another failure is **Context Dilution**, where the global state becomes so bloated with worker logs that the Supervisor loses the "Big Picture."

### Q: How do you choose between a "Sequence of Chains" and a "Multi-Agent Graph"?

**Strong answer:**
I use a **Sequence of Chains** when the task is linear and deterministic (e.g., Extract -> Translate -> Summarize). I use a **Multi-Agent Graph** (like LangGraph) when the task is **Non-Linear** or requires **Conditional Loops**. For example, if the "Translate" step might fail and need to go back to "Extract" for more context, a static chain breaks, but a graph can self-correct by routing back to an earlier node.

### Q: When would you use A2A for multi-agent orchestration versus keeping all agents in a single framework?

**Strong answer:**
I keep agents in a single framework when the team owns all agents, they share the same runtime, and low latency between agent calls is critical. I introduce A2A when crossing **organizational or vendor boundaries**, for example when my orchestrator needs to delegate to a compliance agent maintained by a different team, or when integrating a third-party specialized agent (e.g., a legal review service). A2A adds HTTP overhead but provides **vendor neutrality**, **independent scaling**, and **capability discovery** via Agent Cards. The rule of thumb: same team, same framework; different team or vendor, use A2A.

### Q: When do you define a multi-agent workflow in code, and when do you let a coordinator model delegate?

**Strong answer:**
It depends on whether I know the shape of the work before it starts. If the stages are known (gather, draft, verify, approve), I write them as code: stages run in a fixed order, pass structured results, and stop at human checkpoints, which makes cost bounded and every run auditable. GitHub's dynamic workflows are this pattern as a product. If the task's shape is only discoverable by doing it (a large refactor, open-ended research), I let a coordinator model delegate, as in Cursor Projects or GitHub's `/fleet`, and I bound it from outside: a concurrency cap on subagents, a total budget for the fan-out, and a verifier that is not the coordinator. In practice I mix them: a code-defined outer pipeline whose "do the work" stage is a model-driven coordinator, so adaptivity lives where it pays and determinism everywhere else.

---

## References
- Significant Gravitas. "AutoGPT" (open-source project, 2023)
- Li et al. "CAMEL: Communicative Agents for 'Mind' Exploration of Large Language Model Society" (2023). https://arxiv.org/abs/2303.17760
- OpenAI. "Swarm" (experimental library, 2024), succeeded by the OpenAI Agents SDK (2025)
- A2A Project. "Agent2Agent Protocol Specification v1.0.1" (2026). https://github.com/a2aproject/A2A
- Gartner. "Gartner Predicts 40% of Enterprise Apps Will Feature Task-Specific AI Agents by 2026, Up from Less Than 5% in 2025" (press release, August 26, 2025). https://www.gartner.com/en/newsroom/press-releases/2025-08-26-gartner-predicts-40-percent-of-enterprise-apps-will-feature-task-specific-ai-agents-by-2026-up-from-less-than-5-percent-in-2025
- Andrew Ng. "Agentic Design Patterns" (The Batch, 2024)
- GitHub Changelog. "Dynamic workflows in Copilot CLI and the Copilot app" (October 1, 2026). https://github.blog/changelog/2026-10-01-dynamic-workflows-in-copilot-cli-and-the-copilot-app

---

*Next: [Agent Memory and State](05-agent-memory-and-state.md)*
