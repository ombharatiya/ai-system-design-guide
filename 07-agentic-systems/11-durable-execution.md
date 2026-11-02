# Durable Execution for Long-Running Agents

An agent run is not a request/response handler. It calls tools, reads documents, triggers actions, waits for approvals, and carries state across steps that may run for minutes, hours, or days. That collides with ordinary infrastructure: processes get killed, nodes get recycled, and deploys roll pods. A naive agent loop holding state in memory loses everything on any of those events, and a naive retry re-runs side effects. **Durable execution** is the discipline that makes long-running agents survive all of it. This chapter covers the model, the tools, how it maps onto agent loops, and when it is worth the complexity.

## Table of Contents

- [Why Agents Break the Normal Failure Model](#why-agents-break-the-normal-failure-model)
- [The Durable-Execution Model](#the-durable-execution-model)
- [Tools](#tools)
- [Mapping Durable Execution onto Agent Loops](#mapping-durable-execution-onto-agent-loops)
- [Managed Agent Runtimes: Renting the Durable Loop](#managed-agent-runtimes-renting-the-durable-loop)
- [When You Need It](#when-you-need-it)
- [Do You Need Durable Execution?](#do-you-need-durable-execution)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Why Agents Break the Normal Failure Model

Agents are long-running, stateful, side-effecting processes, which breaks three assumptions:

- **Exactly-once side effects.** If a tool call succeeds but the agent crashes before recording it, a resumed run may retry the call, meaning duplicate payments, tickets, or deploys. The core ambiguity is that after a mid-activity crash you cannot tell whether the side effect committed or only its acknowledgment was lost, so a naive retry sends it twice.
- **Human-in-the-loop pauses that survive restarts.** An agent may need to block on an approval for hours or a day without losing progress or burning compute. A pause held in process memory dies on the next deploy.
- **Recorded nondeterminism.** You cannot replay an LLM call and pretend it is the same event; the same prompt can produce a different response. The output must be recorded the first time and reused during recovery.

Naive retries re-run side effects; naive checkpoints that save state only between steps still leave an unsafe window between executing a side effect and recording its result. The distinction to teach: **checkpoints capture state, but durable execution captures the journal of steps**, and you cannot safely resume mid-side-effect from a state snapshot alone.

---

## The Durable-Execution Model

The core pattern is **workflows-as-code plus an append-only event history plus deterministic replay.** Systems like Temporal record an immutable event history for each workflow; if a worker crashes at step 5 of 10, another worker replays the history to reconstruct in-memory state and resumes at step 6.

Where the determinism constraints come from is the load-bearing concept: recovery works *by replay*, so during a replay the steps in the execution must match the steps in the log, or the system cannot guarantee recovery. That means **workflow code itself must be deterministic**: no direct calls to the current time, no random numbers or UUIDs, no direct network calls, no nondeterministic thread interleaving inside workflow code. Nondeterminism and side effects are pushed into **activities** (steps) whose results are recorded once and replayed from the log thereafter.

The building blocks:
- **Activities** are the only place side effects and nondeterminism live; each is independently retried.
- **Exactly-once activities** use idempotency keys, often derived from the workflow and step IDs, so a retried tool call does not double-execute.
- **Durable timers** are persisted and survive worker restarts and deploys, so a workflow can wait days without holding a process open.
- **Signals** push external events (an approval, a cancellation) into a running workflow; paired **queries** read its current state without mutating it (status, monitoring). While a workflow awaits a signal or timer, the worker idles and consumes no compute until the event arrives, when it replays the history and resumes.

The cost of replay-based determinism is **versioning**: because long-running workflows replay old histories, changing workflow code can break replay and cause incidents unless versioned carefully. This is the single most-cited operational hazard.

---

## Tools

| Tool | Where state lives | Footprint | Notes |
|------|-------------------|-----------|-------|
| **Temporal** (reference) | A separate cluster (or Temporal Cloud) | High | Event history plus deterministic replay; many languages; proven at large scale; deepest agent-framework integrations. |
| **Restate** | Lightweight engine, sidecar or embedded | Low | Journals each step; adds Virtual Objects (stateful sessions keyed by user/session with automatic concurrency control). |
| **DBOS** | Postgres rows | Lowest | A library you import; workflow state and transactional side effects can share one Postgres transaction for exactly-once on DB steps; no separate cluster. |
| **Inngest** | Managed, event-driven | Low | Independently retried steps with AI-specific primitives and built-in concurrency and throttling for LLM rate limits. |
| **AWS Step Functions** | AWS-managed | Managed | Workflow as a declarative state machine (not general code); recently added agent-runtime integrations. |

The approaches differ in where they put the durability boundary. Temporal trades operational overhead for scale; DBOS collapses durability into your existing database; Restate is HTTP/gRPC-native with durable sessions; Inngest is event-driven and TS-first; Step Functions is AWS-native but declarative rather than general-purpose code.

---

## Mapping Durable Execution onto Agent Loops

The core mapping: the **agent loop becomes a workflow, and each model call and tool call becomes a durable activity.** On a crash, completed model calls and tool invocations replay from the log rather than re-execute, so you do not re-pay tokens or re-fire side effects, and you can even fix a bug and resume a running app.

The integration landscape in 2026:
- **Temporal + OpenAI Agents SDK** reached general availability in early 2026, wrapping each agent invocation and tool call as a durable activity.
- **Temporal + Google ADK** is experimental, rerouting LLM calls through activities, and notable for requiring minimal code change (the wrappers detect whether they are running inside a workflow and fall back to direct execution otherwise).
- **LangGraph** provides lighter-weight, framework-native durability through **checkpointers** that save graph state at each super-step to persistent storage, with selectable durability modes (checkpoint at exit, asynchronously, or synchronously before each step). Its guidance mirrors the determinism rule: keep the workflow deterministic and idempotent and wrap side effects in tasks.
- **DBOS and Restate** integrate at the library level with agent frameworks, wrapping agent runs and sub-agent calls as durable workflows and child workflows.
- **Temporal + Amazon Bedrock AgentCore Runtime** entered prerelease on September 21, 2026: a Temporal workflow runs the durable agent loop, model and tool calls are activities with retry policies, Strands Agents supplies the programming model, and AgentCore Runtime is the compute behind Temporal Serverless Workers.

The honest tension to teach: **framework-native checkpointing recovers *state*; a full durable-execution engine additionally gives exactly-once side effects, durable timers, signals, and replay semantics across deploys.** The gap matters most when tool calls have irreversible external effects. For agents that are mostly LLM reasoning with recoverable, idempotent tools, framework checkpointing plus idempotency keys on the few non-idempotent tools is often enough.

The canonical pattern that ties it together: when a proposed action is risky, the workflow pauses and waits for human approval via a signal, consuming no compute while it waits, then resumes durably, exactly the [human-in-the-loop](08-human-in-the-loop-patterns.md) approval gate, made crash-proof.

---

## Managed Agent Runtimes: Renting the Durable Loop

As of September 2026 both Anthropic and OpenAI sell the agent harness itself as a managed runtime with durable sessions built in (OpenAI's Agents API joined Claude Managed Agents on September 10), and AWS offers OpenAI's inside its own cloud. That adds a third option next to "framework checkpointing" and "run a durable-execution engine": rent the loop. None of the three agent runtimes below had reached GA by early November 2026, so treat their APIs, prices, and compliance terms as beta terms.

| Platform | Status (November 2, 2026) | Durability and control features | Constraints to check |
|----------|--------------------------|---------------------------------|----------------------|
| **Claude Managed Agents** | Beta (header `managed-agents-2026-04-01`) on the Claude API, Claude Platform on AWS, and, since October 22, Claude on Google Cloud | Durable sessions; hard session budgets that pause with `budget_reached` (August 7); cron-scheduled deployments (since June 9); agents-as-code via `ant apply` with a committed `claude-lock.json` (ant CLI 1.30.0, September 3); server-evaluated auto permission policy and live terminal attach (September 10) | Bills tokens plus $0.08 per session-hour of running time (idle time free); no Batch discount |
| **OpenAI Agents API** | Public beta since September 10; no GA announced | Managed Codex harness: durable sessions, context compaction, recovery, subagent delegation, MCP servers over HTTP, mid-run steering; OpenAI-hosted or self-hosted sandboxes; hosted-browser computer use (September 29) | US data residency only and no Zero Data Retention at launch; billed at model, tool, and container rates (hosted containers $0.03 for 1 GB to $1.92 for 64 GB per 20-minute session, billed by the minute with a 5-minute minimum). Since October 9, OpenAI-hosted sandbox compute bills only during active turns, so idle time between turns, such as a wait for human approval, no longer costs sandbox time, matching Claude's running-time meter |
| **Amazon Bedrock Managed Agents** (powered by OpenAI) | Public preview since September 29 | A customized OpenAI Agents API running inside AWS: per-agent IAM role, CloudTrail logging, durable sessions, human approval before consequential actions | us-east-1, us-east-2, and us-west-2 only; no extra charge during preview |
| **AWS AgentCore Runtime V2** | Announced September 18 | Restores each instance from a snapshot of an initialized agent; P75 cold start about 2 s versus roughly 5.4 s to 30 s on the original runtime (AWS-reported) | Opt in with `platformVersion` V2; higher rate on far fewer GB-hours |
| **Microsoft Foundry Agent Service** | Routines GA (September 24) | Recurring, one-shot timer, and event-based triggers for published agents | Network egress controls still in preview |

What renting buys: crash recovery, compaction, triggers, budgets, and approvals without operating a cluster. What it does not buy: **exactly-once semantics for your own side-effecting tools**. A managed runtime can resume a session, but it cannot know whether your payment API committed before the crash, so non-idempotent tools still need idempotency keys and, for multi-step side effects, your own durable workflow behind the tool. The other costs are lock-in at the harness layer (agents-as-code files, session semantics, and event formats are vendor-specific) and residency or retention gaps, which can rule a platform out for regulated data before any technical comparison starts. Bedrock Managed Agents running OpenAI's harness inside AWS is the clearest sign that the harness, not the model endpoint, is becoming the integration unit.

---

## When You Need It

Durable execution is the emerging answer to *production* agent reliability, and the 2026 traction is real: Temporal followed its earlier Series D with a $550M Series E at a $12.55B valuation (September 14, 2026), reporting 1.9 trillion billable actions in August (up more than 350% year over year), more than 4,300 paying customers, and OpenAI's Temporal usage up 60x in under a year (company-reported figures). A widely cited vendor case study describes a deep-research agent that **migrated from a framework prototype to a durable-execution engine** after hitting race conditions, fragile custom retry logic, and stale-state bugs that became costly to support (a vendor-published account, so read the direction as real and the framing as theirs).

But it is a deliberate complexity trade. The constraints (determinism, versioning hazards, a new testing and monitoring model) are real, and for agents that are mostly read-only, short-lived, or single-shot, **framework-native checkpointing or a queue plus idempotency keys is often enough** and far cheaper to operate. DBOS and Restate lower the entry cost materially versus a full cluster, so if the objection is operational overhead, the library-and-Postgres approach may get most of the value.

---

## Do You Need Durable Execution?

Walk these in order:

1. **Does any tool call have an irreversible external side effect** (payment, email, deploy, ticket, cross-system write)? No: framework checkpointing or a retry/queue likely suffices. Yes: continue.
2. **Can a single run outlast your process or deploy cycle, or must it pause for human approval across restarts?** No: in-memory plus a checkpoint on completion is probably fine. Yes: you need durable timers and durable pauses.
3. **Would re-running the whole agent on a crash be unacceptable** in cost, duplicate effects, or lost multi-hour progress? Yes: you need replay and exactly-once, so durable execution is justified.
4. **Pick the weight class:** DBOS if side effects are mostly writes to your own Postgres and you want one deploy; Restate for low-ops, HTTP-native, stateful sessions; Inngest for event-driven, TS-first, AI-native rate-limit control; Step Functions if you are all-in on AWS and fine with a declarative state machine; Temporal for large scale, complex long-running processes, and the deepest agent-framework integrations; a managed agent runtime (Claude Managed Agents, OpenAI Agents API, Bedrock Managed Agents) when its residency, retention, and pricing fit and you would rather rent the loop than run it; or stay with framework-native durability (LangGraph checkpointers) plus idempotency keys when the agent is mostly reasoning with recoverable tools.

It is overkill for simple CRUD, sub-millisecond hot paths, pure high-throughput streaming, or a tiny team whose needs a queue with a dead-letter handler already covers.

---

## Interview Questions

### Q: Why are naive retries and checkpoints insufficient for a production agent with side effects?

**Strong answer:**
Because an agent crash creates an ambiguity a retry cannot resolve safely. If the agent calls a tool that charges a card and then crashes, you cannot tell from a state snapshot whether the charge committed before the crash or whether only the acknowledgment was lost, so a naive retry risks charging twice. A plain checkpoint that saves state between steps still leaves an unsafe window between executing the side effect and recording that it happened. Durable execution closes this by capturing the journal of steps, not just the latest state: each side-effecting step is a recorded activity with an idempotency key, so on replay a completed activity returns its recorded result instead of re-running. That gives exactly-once semantics for the side effect and lets the workflow resume from the exact point it failed rather than from the beginning.

### Q: When is durable execution overkill, and what would you use instead?

**Strong answer:**
It is overkill when the agent has no irreversible side effects, runs short enough to fit inside a process and deploy cycle, and would be fine to simply re-run on failure, for example a read-only research or summarization agent with idempotent tools. There I would use framework-native durability like a LangGraph checkpointer to recover state, plus idempotency keys on the few non-idempotent calls, and a retry queue with a dead-letter handler. The determinism constraints and versioning hazards of a full engine like Temporal are a real cost, so I would only take them on once the agent has irreversible effects, must pause for human approval across restarts, or is long enough that re-running on a crash is unacceptable. If operational overhead is the blocker but I still need durability, a library approach like DBOS that uses my existing Postgres gets much of the value without running a cluster.

### Q: Would you build your agent on Temporal or rent a managed agent runtime like the OpenAI Agents API or Claude Managed Agents?

**Strong answer:**
I separate the agent loop from the side effects. Renting the loop is attractive: durable sessions, compaction, triggers, session spend caps, and policy-evaluated approvals arrive without a cluster, and the vendor tunes the harness to its models. But I check three things first. Compliance: the OpenAI Agents API launched with US-only data residency and no Zero Data Retention, which can end the conversation for regulated data. Semantics: a managed runtime resumes the session, but it cannot give exactly-once guarantees for my payment or ticketing tools, so those still need idempotency keys or a durable workflow of their own behind the tool interface. Lock-in: agents-as-code files, event formats, and session semantics are vendor-specific, so moving later is a rewrite, and model choice gets coupled to harness choice. My usual answer is a hybrid: rent the loop for agents that are mostly reasoning plus idempotent tools, and keep irreversible multi-step operations (refunds, provisioning) in Temporal or DBOS workflows exposed to the agent as single tools, so the part that must be exactly-once lives where I control it.

---

## References

- Resonate, ["From where do deterministic constraints come?"](https://journal.resonatehq.io/p/from-where-do-deterministic-constraints)
- Restate, ["What is durable execution?"](https://www.restate.dev/what-is-durable-execution)
- Temporal, [OpenAI Agents SDK integration](https://temporal.io/blog/announcing-openai-agents-sdk-integration), [Series D announcement](https://temporal.io/news/temporal-raises-300M-to-make-agentic-ai-real-for-companies), and [Series E announcement](https://temporal.io/news/temporal-raises-550m-at-a-12-55b-valuation)
- OpenAI, [Agents API overview](https://developers.openai.com/api/docs/guides/agents-api/overview)
- Anthropic, [Claude Platform release notes (Managed Agents)](https://platform.claude.com/docs/en/release-notes/overview)
- Temporal, [prototype to production-ready agentic AI: a Grid Dynamics case study](https://temporal.io/blog/prototype-to-prod-ready-agentic-ai-grid-dynamics)
- Google ADK, [Temporal integration](https://adk.dev/integrations/temporal/)
- LangChain, [durable execution in LangGraph](https://docs.langchain.com/oss/python/langgraph/durable-execution)
- Diagrid, ["Checkpoints are not durable execution"](https://www.diagrid.io/blog/checkpoints-are-not-durable-execution-why-langgraph-crewai-google-adk-and-others-fall-short-for-production-agent-workflows)

---

*Next: [Loop Engineering](12-loop-engineering.md)*
