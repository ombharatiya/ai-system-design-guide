# State Management Patterns

State management in AI systems has moved from simple "sessions" to **Stateful Agent Graphs**. Managing the flow and persistence of an agent's "mind" is as critical as the LLM itself: it is one of the main reasons LangGraph (1.2.19, released October 30, 2026) has become the default control-flow runtime for LangChain-built agents. In 2026 a second question joined "how do I structure state?": **who holds it**, your checkpointer, a durable workflow engine, or a managed agent platform.

## Table of Contents

- [The State Object](#the-state-object)
- [State Machines (LangGraph)](#state-machines-langgraph)
- [Checkpointing and Resume](#checkpointing-and-resume)
- [Where State Lives](#where-state-lives)
- [Parallel State (Fork/Join)](#parallel-state-forkjoin)
- [Time-Travel (State Rewriting)](#time-travel-state-rewriting)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The State Object

The "State" is the **Single Source of Truth** for an agent session.
```python
from typing import Annotated, Any, TypedDict

from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages


class AgentState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    plan: list[str]
    current_task: str
    tool_results: dict[str, Any]
    user_context: dict[str, Any]
    iteration_count: int
```
**Best practice**: State should be **Strictly Typed** and **Append-Only** whenever possible to prevent data loss during long execution loops. Append-only now matters for the model too: the newest Claude models invalidate thinking blocks when earlier messages change, so a state reducer that rewrites `messages` in place silently costs you reasoning continuity.

---

## State Machines (LangGraph)

Industry has converged on **Cyclic Graphs** (State Machines).
- **Nodes**: Functions that take the state and return an update.
- **Edges**: Conditional logic that determines the next node based on state values (e.g., `if state['error'] -> goto 'recovery_node'`).

LangGraph's lead has narrowed: CrewAI added checkpointing in 1.14, Microsoft Agent Framework ships workflow checkpointing and reconnectable background responses, and Google ADK 2.x added graph workflows. See [LangGraph Orchestration](../09-frameworks-and-tools/02-langgraph-orchestration.md).

---

## Checkpointing and Resume

In production, agents can run for minutes or hours.
- **Persistence Layer**: Every state update is saved to a DB (Postgres/Redis).
- **Resiliency**: If the server crashes, the orchestrator retrieves the last `checkpoint_id` and resumes exactly where it left off.
- **Side effects**: Resuming replays from the last checkpoint, so any tool call after it may run twice. Give side-effecting tools idempotency keys, or record their results before advancing the checkpoint.
- **UX**: This allows for **Asynchronous Agents** where the user gets an "I'm working on it" message and a notification 10 minutes later when the state is "Complete."

---

## Where State Lives

| Option | Who holds the state | Strengths | Costs |
|--------|---------------------|-----------|-------|
| **Graph checkpointer** (LangGraph + Postgres) | You, per super-step | Full control, inspectable state, time travel | You build retries, timeouts, and idempotency yourself |
| **Durable workflow engine** (Temporal, Durable Functions) | The engine's event history | Retries, timers, and exactly-once workflow logic; model and tool calls become activities with retry policies | Determinism rules for workflow code; another system to run |
| **Managed agent platform** (Claude Managed Agents, OpenAI Agents API, Bedrock Managed Agents in preview) | The provider's session | No loop or state store to operate; versioned agent configs; session budgets | Data residency and retention follow the vendor (the OpenAI Agents API, in public beta since September 10, 2026, is US residency only and has no ZDR); lock-in |

Frontier labs use durable execution too (Temporal reported OpenAI's usage up 60x in under a year), and managed runtimes are attacking cold starts: AWS announced AgentCore Runtime V2 on September 18, 2026, reporting a P75 cold start of about 2 seconds versus roughly 5.4 to 30 seconds on the original runtime, billed at a higher rate for far fewer GB-hours (vendor-reported). The tradeoffs and an interview question on build versus rent are in [Durable Execution](../07-agentic-systems/11-durable-execution.md#managed-agent-runtimes-renting-the-durable-loop).

---

## Parallel State (Fork/Join)

For complex tasks, we **Fork** the state.
1. **Fan-out**: Send the state to 3 sub-agents (e.g., Researcher A, B, and C).
2. **Fan-in (Join)**: A "Manager" agent receives the outputs of all three and merges them back into the main state object.

Define the merge with a reducer (append, union, or last-writer-wins per field) rather than letting branches overwrite shared keys, and cap fan-out: OpenAI's Agents API, for example, defaults `max_concurrent_subagents` to 6.

---

## Time-Travel (State Rewriting)

As covered in [Human-in-the-Loop Patterns](../07-agentic-systems/08-human-in-the-loop-patterns.md), state management allows for **Human Intervention**.
- A developer can browse the session history, find a "bad turn," edit the state object at that specific timestamp, and **Re-run** the graph from that point.
- **Treat a rewrite as a fork, not an edit.** On Claude Fable 5.1, Opus 5.5, and Sonnet 5.5, thinking blocks are bound to the exact prefix that produced them. Re-running from an edited checkpoint with the old thinking blocks still attached gets a 400 on accounts created on or after August 31, 2026 (or drops the blocks under the `drop_block` beta option). Replay from the edit point with fresh model calls, and keep the original branch for audit.

---

## Interview Questions

### Q: Why use a "Graph-based" State Machine (LangGraph) instead of a simple "While loop" for agents?

**Strong answer:**
A While loop is **Opaque and Brittle**. You can't easily visualize the logic, and error handling becomes a mess of nested if-statements. A Graph-based approach is **Observable and Modular**. You can visualize the Entire Flow (as a Mermaid diagram), unit-test individual nodes, and implement complex features like "Backtracking" or "Parallel execution" simply by adding new edges. It also makes **State Persistence** trivial because the framework handles the saving/loading between nodes. The honest counterpoint: for a single agent with one tool loop, a while loop plus a durable checkpoint is often enough, and the graph earns its complexity when you have branches, human interrupts, or parallel sub-agents.

### Q: How do you prevent "State Bloat" in long-running agent sessions?

**Strong answer:**
We use **State Pruning** and **Message Summarization**. Instead of carrying the entire `tool_results` dictionary through the whole graph, we trim it once a sub-task is complete, keeping a reference to the full output in storage. For the `messages` list, we compact history when it crosses a token threshold, preferably with the provider's compaction (which keeps prefix caching and thinking blocks valid), rather than a homemade summarizer that rewrites earlier turns. The summary is treated as untrusted output: we keep the raw transcript and check that the summary does not introduce instructions the original did not contain.

### Q: Your team edits a bad turn in LangGraph and replays from that checkpoint. After switching to Claude Opus 5.5, replays start failing with 400 errors. What happened?

**Strong answer:**
Opus 5.5 binds each thinking block to the model and to the exact conversation prefix that produced it. Editing an earlier turn changes the prefix, so when the replay sends the old thinking blocks back, the API detects the mismatch and, for accounts created on or after August 31, 2026, returns a 400. The fix is to treat time travel as a fork: strip the thinking blocks after the edit point and let the model reason afresh, or use the `drop_block` behavior under the binding-controls beta while migrating. I would also make the main path append-only, using mid-conversation system messages for corrections instead of edits, so only deliberate debugging sessions pay the cost.

---

## References
- [LangGraph persistence and checkpointing documentation](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Temporal. Amazon Bedrock AgentCore with Temporal Serverless Workers (September 2026)](https://temporal.io/blog/amazon-bedrock-agentcore-with-temporal-serverless-workers)
- [AWS. The new AgentCore Runtime (September 2026)](https://aws.amazon.com/blogs/machine-learning/the-new-agentcore-runtime-elastic-optimized-and-consistently-fast-starts/)
- [Anthropic. Preserved thinking (thinking-block binding)](https://platform.claude.com/docs/en/build-with-claude/preserved-thinking)

---

*Next: [Section 09: Frameworks and Tools](../09-frameworks-and-tools/01-langchain-deep-dive.md)*
