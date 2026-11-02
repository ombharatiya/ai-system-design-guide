# LangGraph Orchestration

LangGraph is the **de facto standard** for building stateful, multi-agent systems. It reached v1.0 in October 2025 and is at 1.2.19 (Oct 30, 2026; seven patch releases in October); LangGraph 0.4 stays in maintenance mode until December 2026. Unlike simple chains, LangGraph allows for **Cycles**, **State Persistence**, and **Human-in-the-Loop** interventions.

Adoption depends on which number you read. On GitHub stars CrewAI still leads (about 60.4K versus 44.3K for `langchain-ai/langgraph` on Nov 2, 2026). On downloads LangGraph leads by an order of magnitude: about 44.8M PyPI downloads in the 30 days to Nov 1 versus about 2.4M for `crewai`, plus about 14.0M npm downloads of `@langchain/langgraph` in October. Stars measure attention; downloads measure dependency, inflated by CI and transitive installs. Quote the metric that matches the claim you are making.

## Table of Contents

- [The Graph Philosophy](#the-graph-philosophy)
- [Cyclic vs. Acyclic Workflows](#cyclic-vs-acyclic)
- [State Management in LangGraph](#state-management)
- [Persistence and Checkpointing](#persistence-and-checkpointing)
- [Multi-Agent Orchestration Patterns](#multi-agent-patterns)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Graph Philosophy

In 2023, agents were "Black Boxes."
Today, agents are **Graphs**.
A graph consists of:
- **Nodes**: Python functions (The LLM, a tool, or data processing).
- **Edges**: Paths between nodes.
- **Conditional Edges**: Logic that determines the path based on the **State**.

---

## Cyclic vs. Acyclic

Standard LangChain is **Acyclic** (Sequential).
LangGraph is **Cyclic**.
- **The Power of the Loop**: An agent can try a tool, see the error, and **cycle back** to the "Thinking" node to try again. This is the foundation of the **ReAct** pattern.

---

## State Management

The **State Schema** is the "Mind" of the graph.
```python
class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    plan: list[str]
    is_secure: bool
```
**Nuance**: Using `Annotated` with `add_messages` allows the graph to **Append** to history rather than overwriting it, preserving the full reasoning trajectory.

---

## Persistence and Checkpointing

Current LangGraph uses **Thread-based Persistence**.
- **The Concept**: Every session has a `thread_id`, and a checkpointer (Postgres, SQLite, Redis, or in-memory for tests) saves state after every super-step.
- **The Win**: If a user comes back after 2 days, the agent remembers the exact point it was at in a multi-step workflow.
- **Interrupts**: `interrupt()` pauses a node for human approval and resumes from the checkpoint, which is how durable human-in-the-loop works without a polling loop.
- **Time-Travel**: Developers can "re-run" a specific thread from a previous state to debug a failure.
- **Checkpoint storage is a tuning knob, and defaults move**: LangGraph 1.2.18 (Oct 26, 2026) changed `DeltaChannel` to write a full snapshot every 100 updates instead of every 1,000, which makes reads on long threads faster and grows storage (`snapshot_frequency=1000` restores the old behavior). Watch checkpoint-store size after upgrades, not just latency.

The moat has narrowed. CrewAI added checkpointing in 1.14 (Apr 2026), Microsoft Agent Framework ships workflow checkpointing, and Google ADK 2.0 added graph workflows with pause and resume. LangGraph's remaining edge is maturity: branch-from-any-checkpoint debugging and the LangSmith trace integration.

---

## Multi-Agent Patterns

| Pattern | Description | Case Study |
|---------|-------------|------------|
| **Supervisor** | One "Manager" directs specialized workers. | Research Team |
| **Peer-to-Peer**| Agents hand off tasks to each other directly. | Customer Support |
| **Hierarchical**| Graphs within Graphs (Nested graphs). | Enterprise Engineering |

Tool access in these patterns increasingly comes through MCP: since `langchain` 1.4.0 (Sep 3, 2026) the MCP adapter is part of the core `langchain` package, so a LangGraph node can mount MCP servers without a separate adapter dependency.

---

## Interview Questions

### Q: Why use LangGraph instead of a managed agent runtime such as the OpenAI Agents API or Claude Managed Agents?

**Strong answer:**
**Control and Portability versus operational offload.** The question used to be framed against OpenAI's Assistants API, which shut down on August 26, 2026 and forced every Threads-based app to rebuild on Responses plus Conversations. That is the first point: vendor-managed state can disappear on the vendor's schedule. The managed runtimes that replaced it are much better (the OpenAI Agents API, public beta Sep 10, 2026, runs the Codex harness with durable sessions, compaction and recovery; Claude Managed Agents adds session budgets, a server-evaluated permission policy and `ant apply` agents-as-code), and they remove real work. But the loop runs on their side. LangGraph is a **White Box framework**: I can use any model (OpenAI, Claude, open-weight models), control exactly when a tool is called, inject validation between steps, and run it on-prem. For regulated workloads the deciding facts are concrete: the Agents API beta is US-residency only with no Zero Data Retention. My default is LangGraph (or a thin custom loop) for the core workflow, with managed runtimes for leaf agents where their compaction and sandboxing save more than the lock-in costs.

### Q: How do you handle "State Overload" in a graph with 20+ nodes?

**Strong answer:**
We use **State Narrowing**. Instead of passing the entire global state to every node, we define specialized sub-states for sub-graphs. We also use `trim_messages` (or a summarization node) to prune the message history before it hits the LLM, ensuring we don't waste tokens while keeping the "Truth" preserved in the persistence layer. Large tool results go to a store with a pointer left in state, the same tool-result offloading pattern Haystack 3.0 and Agno 3.0 now ship as built-ins.

---

## References
- LangChain. "LangGraph: Multi-Agent Workflows" (Jan 2024): https://www.langchain.com/blog/langgraph-multi-agent-workflows
- LangGraph persistence docs: https://docs.langchain.com/oss/python/langgraph/persistence
- LangChain release policy: https://docs.langchain.com/oss/python/release-policy
- Anthropic. "Building Effective AI Agents" (Dec 2024; workflows vs agents, orchestrator-workers, evaluator-optimizer): https://www.anthropic.com/engineering/building-effective-agents

---

*Next: [LangSmith Observability](03-langsmith-observability.md)*
