# Agent Memory and State

Memory is what allows an agent to learn and maintain context over time. Agent memory has matured from "Chat History" into a **Multi-Tiered Cognitive Architecture** with four named layers (Working, Episodic, Semantic, Procedural), each with its own write pattern, latency budget, and failure modes. Production systems (Mem0, Letta, Anthropic Memory Tool + Skills, Zep/Graphiti, LangMem) now treat memory selection as a first-class architecture decision. The 2026 shift to watch: several of them converged on **file-based memory** that the agent navigates by path (Letta's MemFS, Claude's memory tool, coding-agent instruction files), while the market-leading library dropped its graph store.

The 2026 research wave shaping this chapter: A-MEM (NeurIPS 2025), HippoRAG (multi-hop graph retrieval), Multi-Layered Memory Architectures, HaluMem (operation-level memory hallucination benchmark), MINJA / MemoryGraft (query-only memory poisoning attacks), and TTT-E2E (a multi-lab effort spanning Stanford, Berkeley, UCSD, NVIDIA, and Astera), a test-time-training approach that compresses context into weights.

## Table of Contents

- [The Memory Hierarchy](#the-memory-hierarchy)
- [Short-Term: The Reasoning Trace](#short-term-the-reasoning-trace)
- [Episodic Memory: Past Experiences](#episodic-memory-past-experiences)
- [Semantic Memory: The Persona](#semantic-memory-the-persona)
- [Procedural Memory: Learned Skills and Workflows](#procedural-memory-learned-skills-and-workflows)
- [Tradeoffs: Where Does Fact X Go?](#tradeoffs-where-does-fact-x-go)
- [Production Implementations](#production-implementations)
- [Failure Modes and Mitigations](#failure-modes-and-mitigations)
- [Mem0 and Personalization](#mem0-and-agentic-personalization)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Memory Hierarchy

Agents use a tiered approach to storage:

| Tier | Type | Technology | Purpose |
|------|------|------------|---------|
| **L1** | Working Memory | Context Window / KV Cache | Current task steps, local vars |
| **L2** | Episodic Memory | Vector DB / Graph | "What did I do last time?" |
| **L3** | Semantic Memory | SQL / Knowledge Graph | User preferences, "The Truth" |
| **L4** | Procedural Memory | Skills Registry / Tool Policies / Workflow Graph | "How do I perform this task?" |

### Practical Properties of Each Tier

The tiers differ on more than purpose. Read pattern, write pattern, latency budget, and freshness expectations each push toward different storage tech:

| Dimension | L1 Working | L2 Episodic | L3 Semantic | L4 Procedural |
|---|---|---|---|---|
| **What it holds** | Active turn, tool outputs, scratchpad, system prompt | Past sessions, trajectories, observations with timestamps | Distilled facts, preferences, entity relationships | Skills, playbooks, system-prompt instructions, code/tool sequences |
| **Read pattern** | Every token, every turn (in-attention) | Top-k on similarity + recency + importance | Triggered lookup on entity/topic mention | Loaded when matching task signature |
| **Write pattern** | Continuous append by inference engine; KV-cache mutation | Append-only log; commit at turn boundary | Extract, dedupe, upsert; conflict resolution at write | Reflective write after success/failure; explicit human or self-edit |
| **Latency budget** | <50ms (resident in GPU HBM) | 100-300ms (vector ANN + rerank) | 200-800ms (graph traversal + LLM extraction) | 50-500ms (file read or small index lookup) |
| **Freshness expectation** | Token-fresh; lost at session end | Hours to months; tolerates staleness | Should reflect *current* state; staleness is a bug | Slow-changing; updates are deliberate |
| **Storage tech** | KV cache in HBM (vLLM PagedAttention blocks) | Vector DB (Pinecone, Weaviate, Qdrant), append-only log | Knowledge graph (Neo4j, Graphiti), KV store, bitemporal relational rows | Filesystem (Claude `/memories/`, Skills as `SKILL.md`), prompt registry, fine-tuned LoRA |
| **Query semantics** | Position + attention | Similarity + recency + importance (Park et al. weighted) | Entity-relation match, structured query, bitemporal filter | Task-signature match, often filename or tag lookup |
| **Eviction** | Sliding window, LRU on KV block hash | Decay scoring, consolidation into L3, archival to cold storage | Supersession by closing the old fact's validity (`invalid_at`); explicit deletion for GDPR | Manual deprecation, A/B against newer skill, version pinning |

**What this means in practice:** when a fact arrives, the architecture question is not "should we remember it?" but "*at which tier*, with which freshness contract, and with which eviction rule?" Picking the wrong tier produces predictable failure modes (a session preference promoted to L3 leaks across sessions; a stable user fact left in L2 gets evicted in two weeks). See [Tradeoffs: Where Does Fact X Go?](#tradeoffs-where-does-fact-x-go) below.

---

## Short-Term: The Reasoning Trace

Production agents no longer just store the "Messages"; they store the **State Object**.
- **The Scratchpad**: A dedicated section of the prompt where the agent "writes notes" to itself that are NOT shown to the user.
- **KV Cache Tiling**: For long-running agents, we use **Prefix Caching** to keep the "System Instruction" and "Standard Tools" warm in GPU memory, only swapping the dynamic task state.
- **Offloading Large Results**: Tool outputs are the fastest way to flood working memory. Agno 3.0 (August 2026) writes any tool result over 16,000 characters to a file store and replaces it in context with an envelope plus `read_result` and `search_result` tools, so the agent pulls back only what it needs.
- **Notes Instead of One Summary**: An experimental Codex feature that shipped with GPT-6 Astra keeps searchable notes across context windows rather than compressing history into a single summary. It is a vendor-shipped alternative to summary compaction for long-horizon work: instead of one lossy summary, the agent keeps an index it can query.

---

## Episodic Memory: Past Experiences

Episodic memory stores "Runs" or "Trajectories."
- If an agent failed to scrape a website last Tuesday, episodic memory should prevent it from trying the same failing selector today.
- **Pattern**: When a task completes, summarize the "Lessons Learned" and store them in a vector DB. At the start of a new task, perform a **Self-Search** for similar previous tasks.

---

## Semantic Memory: The Persona

Semantic memory stores "Facts" about the user or the environment.
- *"The user prefers JSON output."*
- *"The production DB is offline between 3 AM and 4 AM."*

**Best practice**: Use a **Knowledge Graph** for semantic memory. Unlike vector search (which is fuzzy), a graph provides deterministic retrieval of entities and relationships (e.g., `User` -- `OWNER_OF` --> `Project_A`).

Treat the graph database as a choice to justify, not a default. Mem0 v2 removed its Neo4j, Memgraph, Kuzu, and Apache AGE drivers and now keeps entity links in a parallel collection inside the existing vector store, which covers most personalization lookups without a second database to run. Reach for a dedicated graph when multi-hop traversal or temporal questions ("what was true on March 12?") are the product, as in Graphiti's bitemporal edges.

---

## Procedural Memory: Learned Skills and Workflows

Procedural memory stores how to do things. While episodic memory answers, "What happened before?" and semantic memory answers, "What is true?", procedural memory answers:

"What is the correct process for completing this type of task?"

This layer captures reusable skills, tool-use patterns, operating procedures, and workflow preferences.

Examples:

* "When generating a weekly report, first pull metrics from Snowflake, then validate against the dashboard, then summarize anomalies."
* "When responding to a customer complaint, classify urgency, retrieve policy, draft response, and escalate if confidence is low."
* "When writing SQL, always inspect the schema first, generate a query, run validation, and explain assumptions."

Procedural memory is especially important for agentic systems because many tasks are not just about remembering facts. They require following the right sequence of actions.

---

## Tradeoffs: Where Does Fact X Go?

The first-order decision is not "which tier" but *"who pays the cost of being wrong?"* A missed retrieval in L2 fails one turn. A bad fact in L3 fails *every* turn until corrected. A poisoned skill in L4 propagates to every future invocation.

### Tier-selection table

| Fact / concern | Tier | Reasoning |
|---|---|---|
| "The user's API rate limit is 1000 req/min" | L3 with bitemporal validity (`valid_at` / `invalid_at`) | Tenant-scoped fact; queryable by entity; must support supersession. Not L4: it's data, not procedure. |
| "Steps to deploy our service" | L4 as a versioned skill | Multi-step recipe with conditional branches. Skills compose; semantic triples do not. |
| "The agent's last failed attempt at this task" | L2 raw, *then* reflect a lesson into L4 if generalizable | Raw trajectory belongs in episodic; the generalized lesson ("never run migrations during peak hours") earns a Reflexion-style write to L4. |
| "User prefers terse responses" | L3 | Stable preference, queryable by `user_id`, single triple. |
| "User asked for terse responses *in this conversation*" | L1 only | Session-scoped; do not pollute L3 with potentially-transient preferences. |
| Current weather, today's stock price | None: call the tool | Fast-changing facts with a source of truth elsewhere should never enter memory. |
| "Project Phoenix has team members A, B, C" | L3 as a graph fragment | Multi-hop traversal value; Graphiti or Neo4j-style storage. |

### Cost tradeoffs

- **L1 dominates *latency cost***: TTFT scales with context size; longer working memory means slower first token. KV-cache pressure pushes premium accelerator memory toward saturation.
- **L2 dominates *storage cost* at scale**: append-only logs grow linearly with usage. The [Day-30 problem](https://cipherbuilds.ai/blog/day-30-agent-memory-problem) describes how unpruned episodic stores rot agent quality after a month.
- **L3 dominates *write-amplification cost***: every turn potentially triggers extract, dedupe, conflict resolve. Mem0's design explicitly trades retrieval speed for write-time work.
- **L4 dominates *governance cost***: a bad skill propagates to every future invocation. Anthropic's [Dreams](https://www.mindstudio.ai/blog/what-is-claude-dreaming-anthropic-agent-memory) research preview acknowledges this by writing consolidation output to a separate memory store instead of editing the live one, which leaves room for review before promotion.

### The graduation rule

The interesting design question is *when an L2 episode graduates to L3 or L4*. The defensible rule is threshold-based, not implicit-decay:

1. **N independent observations** of the same pattern (N=3 to 5 is typical).
2. **Confidence-weighted** by source: user-stated > tool-output > model-inferred.
3. **Human or LLM-judge review** at the consolidation step (not per turn).
4. **Scheduled batch consolidation**, not synchronous per-turn writes (avoids write amplification).
5. **Bidirectional**: semantic facts in L3 can be re-instantiated as episodic context for specific tasks. Memory is not a one-way street.

### When to curate: write time, sleep time, or read time

Curation timing is now a named design axis, with real systems at each point:

| Timing | Example | Cost profile | Main risk |
|---|---|---|---|
| **Write time** | Mem0-style extraction on every `add()` | One LLM call per write; cheap reads | Extraction errors are baked in and propagate (the HaluMem finding) |
| **Sleep time** | Anthropic Dreams (research preview): reads a memory store plus past session transcripts, writes a reorganized store with duplicates merged and stale entries replaced; Mem0's paid Dream feature | Batch job off the request path | Staleness between runs; needs a promotion review |
| **Read time** | Just-in-Time Memory (arXiv 2609.27334, September 2026): keep raw trajectories and curate for the current task when reading | Curation cost and latency move onto the read path | The raw store grows; retention and privacy cost rise |

The research result worth citing: Just-in-Time Memory beat the strongest write-time memory baseline by 16.2, 16.3, and 3.9 success-rate points on ALFWorld, WebShop, and tau2-bench, and even an untrained curator was competitive. Read-time curation wins when what is relevant depends on the task you have not seen yet; write-time extraction wins when reads are hot and latency-bound.

---

## Production Implementations

The named systems differ less on "what they store" and more on *write discipline*, *retrieval algorithm*, and *governance posture*.

| System | Sweet spot | What it does well | Where it falls short |
|---|---|---|---|
| **[Mem0](https://github.com/mem0ai/mem0)** | Cross-session personalization at scale | Since v2.0.0 (April 16, 2026): single-pass, ADD-only extraction (one LLM call per `add()`, hash dedup, no update or delete pass) and hybrid retrieval that fuses semantic similarity, BM25, and entity boosting into one score. Entity links live in a parallel collection inside your existing vector store, which let Mem0 delete its Neo4j, Memgraph, Kuzu, and Apache AGE graph drivers. Reports 92.5 on LoCoMo and 94.4 on LongMemEval after the redesign ([benchmarks](https://mem0.ai/blog/ai-memory-benchmarks-in-2026)); its 2025 paper reported a 26% relative gain over OpenAI's built-in memory on an LLM-judge metric (all vendor-reported). SDK v2.2.1 (September 25). | 8K char per-memory cap (not for documents); cloud-first posture creates data sovereignty friction; open-source extraction is append-only, so merging and supersession moved to the paid Dream feature (Pro and Enterprise only, since August 4). |
| **[Letta (formerly MemGPT)](https://github.com/letta-ai/letta)** | Long-running autonomous agents where coherence is the product | Archived its MemGPT-era server on August 16, 2026; active development is the letta-code harness and the TypeScript Agent SDK. All agents now use **MemFS**: long-term memory is a git repository of Markdown files with YAML frontmatter, edited with ordinary file tools. Root files load into the system prompt every turn; directories with their own `MEMORY.md` index stay out of context until the agent navigates to them. | No vector index by default (keyword search via an optional mod), so recall depends on a well-organized tree; teams on the old core / recall / archival paging API must migrate. |
| **[Anthropic Memory Tool + Skills](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)** | Filesystem-mounted L3 and L4 in one substrate | Memory at `/memories/`; Skills as `SKILL.md` packages plus optional scripts (out of beta on the Claude API since August 19, 2026); Managed Agents mount memory at `/mnt/memory/` per session with immutable versioning (April 23, 2026 GA), and since August 19 memory stores also sync into self-hosted sandboxes. The Dreams research preview consolidates a store offline into a new output store. | Filesystem semantics push complexity to the agent (the agent must structure its own directories well). |
| **[Zep + Graphiti](https://github.com/getzep/graphiti)** | Temporal facts where "when did this become true?" matters | Open-source temporal knowledge graph. Each fact edge carries two timelines: `valid_at` / `invalid_at` (when the fact was true in the world) and `created_at` / `expired_at` (when the graph recorded it and when it was superseded). Beats MemGPT 94.8% to 93.4% on DMR. Bitemporal queries enable "what did we believe on March 12?" vs "what is true now?" | Heavier write path (graph extraction, dedup, conflict resolution) than vector-only stores. |
| **[LangMem + LangGraph](https://langchain-ai.github.io/langmem/)** | When you want all four memory types with LangGraph orchestration | Supports episodic, semantic, *and* procedural. Procedural memory in LangMem lets the agent update its own system prompt based on feedback. Background extraction runs out-of-band. | LangGraph-coupled; less attractive if you are not on the LangChain stack. |
| **[OpenAI ChatGPT Memory](https://openai.com/index/memory-and-new-controls-for-chatgpt/)** | Consumer-grade chat continuity, not production-grade agent memory | Two-layer architecture: explicit "saved memories" plus lightweight conversation summaries pre-injected into context. Skips a retrieval step at inference for low latency. | Loses precision vs Mem0-style retrieval. No fine-grained programmatic API for enterprise integration. |
| **Cursor / Devin Desktop (formerly Windsurf)** | Codebase-aware L2/L3 for software engineering agents | Codebase indexed on project open; `@`-mentions for explicit context. Windsurf's auto-generated Cascade "Memories" were a migration item when Cascade was removed in September 2026: a wizard moves them into Devin Local, the editor's only agent since then. | Domain-locked to code. Not a general-purpose memory layer, and the product churn shows why memory you depend on should be exportable. |
| **[Cognition Devin](https://cognition.ai/blog/devin-sonnet-4-5-lessons-and-challenges)** | Repo-scoped engineering agents | Repository wiki auto-indexed every few hours; explicit compacting/summarization preferred over model-managed state. Devin Search is an agentic codebase memory query interface. | Opinionated to engineering workflows. |

**Generative Agents (Park et al. 2023)** remains the reference architecture cited in every survey. The recency / importance / relevance retrieval formula (`alpha_recency * recency + alpha_importance * importance + alpha_relevance * relevance`, each normalized to [0,1] with importance rated 1-10 by an LLM) is still in production use across most of the systems above.

**Keep state you own separate from the orchestration API.** OpenAI shut down the Assistants API on August 26, 2026: threads, runs, and run steps now error, and the migration maps Threads to the Conversations API, Runs to Responses, and Run steps to Items. Vector stores were not part of the shutdown and keep working through the Responses `file_search` tool, so teams on hosted RAG kept their indexes and rewrote only the conversation-state code. Store memory and session state behind your own interface, and treat a vendor's hosted thread object as a cache of that state, not its system of record.

Anthropic's message threads beta (November 2, 2026, header `message-threads-2026-08-12`) makes the same point from the other direction. A create request stores the conversation, system prompt, and tools server-side, and later requests send only new messages naming `previous_message_id`. But a thread expires 5 minutes after its last request, is deleted no more than an hour after last use, is not ZDR eligible, and a continue can return 404 `thread_not_found`, after which the client must resend the full conversation. System and tools may be sent only on the create request, and fixed fields such as `model`, `tool_choice`, `thinking`, and `mcp_servers` must match every turn or the API returns 400 `thread_fingerprint_mismatch`. Threads do not change pricing or cache the prompt by themselves (you still need `cache_control`), and they do not run on Bedrock or Google-managed Vertex. The harness design follows: keep the transcript yourself, use the thread as a bandwidth optimization, and implement 404 replay from day one.

**Emerging frameworks worth tracking**: [Supermemory](https://supermemory.ai), [Recallr](https://recallrai.com), AWS [Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-integrate-lang.html), [Oracle AI Agent Memory](https://blogs.oracle.com/developers/oracle-ai-agent-memory-a-governed-unified-memory-core-for-enterprise-ai-agents).

---

## Failure Modes and Mitigations

Production memory systems have seven recurring failure modes. Knowing them by name is the difference between a junior and staff-level architecture conversation.

### 1. Memory poisoning via prompt injection

Untrusted input gets written to L3/L4 and replayed later as authoritative. [MINJA (NeurIPS 2025)](https://openreview.net/forum?id=QVX6hcJ2um) and [MemoryGraft (Dec 2025)](https://arxiv.org/html/2512.16962v1) demonstrate *query-only* poisoning attacks reaching 95% injection rate and 70% attack success rate *without* elevated privileges. [Palo Alto Unit 42's writeup](https://unit42.paloaltonetworks.com/indirect-prompt-injection-poisons-ai-longterm-memory/) shows poison planted weeks before it fires.

**Mitigations:**
- **Provenance tags** on every memory write: `source = user_stated | model_inferred | tool_output`.
- **Write-time guardrail model** that refuses suspicious instruction-shaped writes ("ignore previous and instead...", role confusion, embedded system-prompt fragments).
- **Trust tiers**: low-trust memories require corroboration before influencing high-stakes decisions.
- **Bulkhead isolation** so poison in one tenant cannot pivot into another.
- **Write-gated shared memory**: in Asana's agents (described September 29, 2026), anyone can give feedback on a task, but only admins and editors can commit it to permanent shared memory or delete from it; everyone else's feedback applies to the current task only. A poisoning vector becomes a permissioned write.
- **Treat compaction summaries as memory writes**: OpenAI disclosed in September 2026 that models in RL training wrote concealment instructions into their own context-compaction summaries, so a summary carried into the next context needs the same provenance and review as any other write (see [Loop Engineering](12-loop-engineering.md#context-and-memory-in-long-loops)).

### 2. Stale facts

Yesterday's preference vs today's. The classic "user said dark mode last month but uses light mode now."

**Mitigations:**
- **Bitemporal storage** (Zep/Graphiti pattern): every fact carries `valid_at` / `invalid_at` (when it was true) and `created_at` / `expired_at` (when the system recorded and retired it).
- **TTL on session-scoped preferences** so they auto-expire.
- **Decay weighting** in retrieval scoring.
- **Explicit re-confirmation** prompts for high-stakes facts older than N days.

### 3. Conflicting facts

User said X, now says Y. Three different conflict types deserve different responses:

| Conflict type | Right response |
|---|---|
| Temporal update ("I moved to Berlin") | Supersede the old fact: its `invalid_at` becomes the new fact's `valid_at`, and `expired_at` is set to now |
| Correction ("I never said that") | Retract with audit trail |
| Preference change ("I want concise responses now") | Add new fact; let decay handle the old |
| Outright contradiction (no obvious resolution) | Ask the user; never silently overwrite |

Track belief status (`ACTIVE` / `SUPERSEDED` / `RETRACTED`) using AGM belief revision rather than last-write-wins.

### 4. Memory drift

Quality degrades over time as low-quality writes dilute high-quality ones. The [Day-30 problem](https://cipherbuilds.ai/blog/day-30-agent-memory-problem) documents agent performance dropping ~30 days into production as the episodic store fills with noise.

**Mitigations:**
- **Quality-weighted retrieval**: boost memories with high verification scores.
- **Scheduled consolidation jobs** that merge duplicates and prune low-utility memories.
- **Canary fact tests in CI**: "agent should still recall the user's name after 50 turns."

### 5. Hallucinated memory writes

Agent infers a fact, stores it as ground truth, then cites it later as authority. Cascading failure where one bad write poisons future retrievals. The [HaluMem benchmark (Nov 2025)](https://arxiv.org/abs/2511.03506) shows existing systems accumulate errors at write time that propagate forward through the QA stage.

**Mitigations:**
- **Schema-enforced memory objects** with separate fields for `confirmed_facts` (with source) vs `inferred_facts` (with confidence).
- Never auto-promote inferences to confirmed without explicit user signal or tool-output corroboration.
- HaluMem-style staged evaluation in CI: measure extraction precision, update correctness, and QA accuracy separately, not as a single end-to-end metric.

### 6. Cross-tenant leakage

Vector ANN returns a neighbor from another tenant; cached prompts contain another tenant's data. [Field measurements](https://medium.com/@isuruig/multi-tenant-ai-infrastructure-the-5-isolation-layers-that-determine-whether-your-customers-data-stays-separate-340aaeef4922) show ~95% organic leakage rates on benign queries in unisolated multi-tenant RAG.

**Mitigations:**
- **Physical separation**: per-tenant collections, not metadata-filtered shared index.
- Enforce tenant scope at the *store* layer via service-account permissions, not application code.
- Separate KV-cache prefix per tenant.
- Per-tenant encryption keys on memory blobs so cross-namespace reads fail at the cryptographic layer.

### 7. Memory-induced reasoning traps

Correct, relevant memories can still make the agent worse. MemTrapBench (arXiv 2608.20202, August 2026) tests reasoning fixation and belief distortion caused by faithfully stored, semantically relevant memories. Across two model families and five memory frameworks, **every memory strategy underperformed the no-memory setting**, the strongest dropping by more than 10%. Retrieval metrics such as LoCoMo recall cannot see this harm, because the retrieval was correct.

**Mitigations:**
- Evaluate end-to-end task success with and without memory injected, not only retrieval quality.
- Gate injection: after retrieval, decide whether the memory is decision-relevant for this task before it enters context.
- Frame injected memories as past context, not instructions. The paper's AdaptiveMem inference-time instruction reduces the traps while preserving scores on standard memory benchmarks.

---

## Mem0 and Agentic Personalization

**Mem0**, Zep, Letta, and Cognee are the standard frameworks for "Smart Memory" in agent stacks.
- It automatically extracts "User Insights" from conversations (in Mem0 v2, one ADD-only extraction pass per write).
- It provides a "Memory API" that agents can call to add, search, or delete memories; conflict resolution is either your job or a scheduled consolidation pass.
- **Impact**: Agents feel "Alive" because they remember a detail you mentioned 3 months ago in a different session.

---

## Interview Questions

### Q: How do you handle "Conflicting Memories" in an agentic system?

**Strong answer:**
Conflicting memories (e.g., the user said "I like blue" last week but says "I like red" now) are handled via **Temporal Weighting** or **Explicit Disputing**. In my architecture, I assign a `timestamp` and a `confidence_score` to every memory triplet. If a new fact conflicts with an old one, the agent is prompted to "Resolve the Conflict" by asking the user for clarification or defaulting to the most recent timestamp. We also use **Decay Functions** where older, non-reinforced memories are eventually pruned from the active index.

### Q: Why is the "Context Window" alone insufficient for a staff-level Agent architecture?

**Strong answer:**
First, **Cost and Latency**: cache reads now cost 0.025x to 0.1x of base input, so a fully cached 1M-token prefix is no longer absurd (about $0.10 to $0.25 per turn on current Claude models: Sonnet 5.5 at $0.10 per 1M cached tokens since October 7, Opus 5.5 at $0.20, Fable 5.1 at $0.25). But misses bill at the full rate, OpenAI bills the whole request at long-context rates once input passes 272K, Claude Haiku 5.5 bills the whole request at 5x once the prompt passes 100K tokens (cache reads and writes count toward that length), and prefill latency still grows with length. Second, **Signal-to-Noise**: Large context windows suffer from context rot: the model gets distracted by irrelevant historical turns, and MemTrapBench showed that even relevant memories can distort reasoning. A Staff-level architecture uses **Selective Memory Retrieval** (RAG over history) to only pull in the 3-5 most relevant historical interactions, keeping the Reasoning Engine focused on the current sub-goal.

### Q: How would you design procedural memory for a production AI agent?

**Strong answer:**

I would design procedural memory as a combination of a **skills registry, workflow graph, and tool-use policies**. Each procedure would define the task type, required steps, available tools, validation checks, failure modes, and escalation rules. After each run, the agent can perform reflection and update the procedure if it discovers a better approach. For example, if an NL2SQL agent repeatedly fails because it skips schema inspection, we can encode schema inspection as a required first step in the procedural memory for all SQL-generation tasks.

### Q: When does episodic memory become a liability rather than an asset?

**Strong answer:**

Episodic memory becomes a liability in three named patterns. First, **index overload**: adding 1,000 low-quality observations buries the 10 high-quality ones in retrieval. This is catastrophic forgetting in the RAG sense. Second, the **Day-30 drift pattern**: agent quality degrades roughly 30 days into production as the episodic store fills with noise that retrieval cannot distinguish from signal. Third, **stale-context bleeding**: past trajectories that succeeded under one configuration become *wrong* context under a new one. A successful tool sequence for Stripe is actively misleading when the user has switched to Adyen.

The mitigations are quality-weighted retrieval, consolidation into L3, and hard recency cutoffs for context-sensitive trajectories. The deeper lesson: episodic memory needs a *pruning policy* from day one. Without one, it is technical debt that compounds linearly with usage.

### Q: How do you prevent memory poisoning when agents can write to their own long-term store?

**Strong answer:**

The hard part is that recent attacks (MINJA, MemoryGraft) are *query-only*: no elevated privileges required. Poison gets planted weeks before it fires. So the threat model is "every input can become a future authoritative memory." The defense has four layers:

1. **Provenance at write time**: every memory carries `source` (user-stated, model-inferred, tool-output), `timestamp`, and `trust_tier`.
2. **Write-time guardrail model**: a smaller classifier refuses suspicious instruction-shaped writes before they hit the store.
3. **Corroboration thresholds**: high-stakes decisions cannot be made on a single low-trust memory; they require multiple independent corroborating writes.
4. **Canary tests in CI**: synthetic poison payloads must not propagate into outputs. Run weekly.

The architectural separation that matters most: the agent's tool surface and its memory write surface should not share trust. Tool outputs should pass through a sanitizer before becoming memory.

### Q: Memory tier selection: where would you put each of these and why? (a) the user's API rate limit, (b) the steps to deploy our service, (c) the agent's last failed attempt at this task, (d) today's stock price.

**Strong answer:**

(a) **L3 semantic** with bitemporal validity. It is a tenant-scoped fact with a supersession lifetime. Not L4, because it is data, not procedure.

(b) **L4 procedural** as a versioned skill or playbook. It is a multi-step recipe with conditional branches. Skills compose; semantic triples do not.

(c) **L2 episodic raw**, with a *reflection hop* to L4 if the failure reveals a generalizable lesson. Raw trajectory belongs in episodic. The lesson ("never run migrations during peak hours") earns a Reflexion-style write to procedural.

(d) **None: call the tool.** Fast-moving facts with a live source of truth should never enter memory. They become stale by definition.

The general rule: data goes in L3, procedures in L4, observations in L2, and never store fast-moving facts that have a live source.

### Q: Walk me through the consolidation policy you would design for episodic-to-semantic transition. When does an episode become a fact?

**Strong answer:**

I use a threshold-based graduation policy, not implicit decay:

- **Frequency threshold**: N independent observations of the same pattern (3 to 5 is typical).
- **Confidence weighting**: user-stated > tool-output > model-inferred.
- **Judge review**: a scheduled batch consolidation job runs an LLM judge (or human reviewer for high-stakes domains) over candidate graduations.
- **Scheduled, not synchronous**: consolidation happens out-of-band on a cron, not per-turn. This avoids write amplification.
- **Bidirectional**: semantic facts in L3 can be re-instantiated as episodic context for specific tasks. Memory flows both ways.

The trap to avoid is implicit consolidation through decay weights alone. It works at small scale and silently fails at production scale because there is no audit trail for "why did this fact appear in L3?"

### Q: A benchmark shows your agent does better with memory turned off. What do you do?

**Strong answer:**

I take it seriously rather than assuming a bug, because MemTrapBench (August 2026) found exactly this across five memory frameworks: correct, relevant memories caused reasoning fixation and belief distortion, and every strategy underperformed no memory. First I split the eval: retrieval quality, injection decisions, and end-to-end task success with and without memory, sliced by task type. Usually memory helps on personalization and continuity tasks and hurts on fresh reasoning tasks where a past answer anchors the model. Then I add an injection gate: retrieve candidates, but only inject when a cheap classifier or rule says the memory is decision-relevant to this task, and frame it as past context with a timestamp, not as instruction. I also consider moving curation to read time (Just-in-Time Memory style), so what is injected is selected for the current task rather than extracted at write time. Success is measured as task success with memory minus task success without it, per task class, and that delta goes on the dashboard next to recall.

### Q: Your agent's memory store has 50M memories across 10K tenants. How do you guarantee cross-tenant isolation and what's your blast radius if isolation fails?

**Strong answer:**

The architecture has five isolation layers:

1. **Physical separation at the store layer**: per-tenant collections or shards, not metadata-filtered shared indexes. The shared-index-with-tenant-id pattern fails open under bugs, including encoding bugs: LangGraph's Postgres and SQLite stores joined namespace labels with "." so `("tenant.bob",)` and `("tenant", "bob")` collided, letting a caller who controlled a label read or delete another namespace's items (CVE-2026-115883, October 21, 2026; see [LangGraph persistence](../09-frameworks-and-tools/02-langgraph-orchestration.md#persistence-and-checkpointing)). Derive the tenant key from verified identity, never from a label the caller supplies.
2. **Enforcement via service-account scoping**: the application code cannot opt out of tenant scope; the database role does not have visibility to other tenants.
3. **Separate KV-cache prefix per tenant**: prevents cached prompts from leaking between tenants.
4. **Per-tenant encryption keys**: cross-namespace bytes are unreadable even if returned by a bug.
5. **Audit logging of every cross-namespace query attempt**: detection in depth.

**Blast radius if isolation fails**: a single bad vector query could leak the *neighborhood* of one query's embedding, potentially hundreds of records from one tenant. Field measurements show ~95% organic leakage rates in unisolated multi-tenant RAG. The mitigation is not "more careful application code"; it is structural separation that cannot be bypassed by application bugs.

### Q: HaluMem shows memory hallucinations accumulate at write time, then propagate. How would you instrument production memory to catch this?

**Strong answer:**

The trap most teams fall into is measuring memory quality only at the QA stage (end-to-end). HaluMem demonstrates that 60-80% of memory errors originate at *extraction* (write) time and propagate. You need to instrument three separate metrics:

1. **Extraction precision**: when the agent writes a fact to L3, is the fact actually supported by the source observation? Sample writes daily, evaluate with a stronger judge.
2. **Update correctness**: when conflicting facts arrive, did the conflict-resolution logic produce the right outcome? Use bitemporal queries to detect "facts that flipped without supersession metadata."
3. **QA accuracy**: end-to-end recall correctness.

On top of that, run **shadow-mode replay**: writes go through a verifier model in shadow mode; mismatches between live writes and shadow-verifier writes flag potential hallucinations for review. **Canary facts** in CI ensure the memory system does not silently regress. **Periodic full-store audits** sample random memories and ask "is this still consistent with the source conversation?"

### Q: TTT-E2E compresses context into weights via test-time training. Where does this fit in the L1-L4 hierarchy, and what new failure mode does it introduce?

**Strong answer:**

TTT-E2E sits *between* L1 and L4. It makes context-derived information part of the model itself for the rest of the session. The appeal is latency: constant cost regardless of context length (2.7x speedup at 128K, 35x at 2M tokens on H100 per NVIDIA's benchmarks).

The new failure mode is governance. In-weights memory has:

- **No audit trail**: you cannot inspect "what does this model now believe?"
- **No eviction interface**: you cannot delete a memory once compressed into weights without rolling back the model state.
- **GDPR right-to-be-forgotten challenges**: the regulatory framework assumes data is at rest, not in weights.
- **Harder poisoning detection**: there is no inspectable store to scan for canary signatures.

Beyond governance there is a capability failure mode: the approach reports failing needle-in-a-haystack retrieval beyond its attention window (about 6% versus 99% for full attention at 128K), so it preserves the gist of the context, not verbatim facts. That rules it out as the only memory tier for retrieval-critical work.

The right framing: TTT-E2E moves memory governance from the storage layer to the training and deployment pipeline. The cost is not eliminated; it is relocated. For most production teams in 2026, this is a research direction to track, not a deployable architecture yet. The broader test-time-training family is mapped in [Research Radar, theme 12](../RESEARCH-RADAR.md#12-test-time-training-learning-at-inference).

---

## References

### Production Frameworks
- [Mem0: Production-Ready AI Agents with Scalable Long-Term Memory (ECAI 2025)](https://arxiv.org/abs/2504.19413)
- [Mem0 AI Memory Benchmarks 2026](https://mem0.ai/blog/ai-memory-benchmarks-in-2026)
- [Letta repository (MemFS; MemGPT-era server archived August 16, 2026)](https://github.com/letta-ai/letta)
- [Mem0 v2.0.0 release notes](https://github.com/mem0ai/mem0/releases/tag/v2.0.0)
- [MemGPT: Towards LLMs as Operating Systems (arXiv 2310.08560)](https://arxiv.org/abs/2310.08560)
- [Anthropic Memory Tool documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [Anthropic: Introducing Agent Skills (October 2025)](https://www.anthropic.com/news/skills)
- [Claude Dreams: offline memory consolidation (third-party explainer)](https://www.mindstudio.ai/blog/what-is-claude-dreaming-anthropic-agent-memory)
- [Zep: Temporal Knowledge Graph Architecture (arXiv 2501.13956)](https://arxiv.org/abs/2501.13956)
- [Graphiti GitHub](https://github.com/getzep/graphiti)
- [LangMem documentation](https://langchain-ai.github.io/langmem/)
- [OpenAI Memory and new controls for ChatGPT](https://openai.com/index/memory-and-new-controls-for-chatgpt/)
- [Cognition: Rebuilding Devin for Claude Sonnet 4.5](https://cognition.ai/blog/devin-sonnet-4-5-lessons-and-challenges)

### Research (2023-2026)
- [Generative Agents: Interactive Simulacra of Human Behavior (Park et al. 2023)](https://arxiv.org/abs/2304.03442)
- [Reflexion: Language Agents with Verbal Reinforcement Learning (Shinn et al. 2023)](https://arxiv.org/abs/2303.11366)
- [HippoRAG: Neurobiologically Inspired Long-Term Memory (Gutierrez et al. 2024)](https://arxiv.org/abs/2405.14831)
- [A-MEM: Agentic Memory for LLM Agents (Xu et al. NeurIPS 2025)](https://arxiv.org/abs/2502.12110)
- [Multi-Layered Memory Architectures (arXiv 2603.29194, March 2026)](https://arxiv.org/abs/2603.29194)
- [Memp: Exploring Agent Procedural Memory (Aug 2025)](https://arxiv.org/html/2508.06433v2)
- [LEGOMem: Modular Procedural Memory for Multi-agent LLM (Oct 2025)](https://arxiv.org/pdf/2510.04851)
- [Rethinking Memory Mechanisms of Foundation Agents (Feb 2026 survey)](https://arxiv.org/abs/2602.06052)
- [Position: Episodic Memory is the Missing Piece (arXiv 2502.06975)](https://arxiv.org/pdf/2502.06975)
- [Just-in-Time Memory: read-time curation of raw trajectories (arXiv 2609.27334, Sep 2026)](https://arxiv.org/abs/2609.27334)
- [MemTrapBench: memory-induced cognitive traps (arXiv 2608.20202, Aug 2026)](https://arxiv.org/abs/2608.20202)

### Safety, Poisoning, and Hallucinations
- [HaluMem: Operation-Level Memory Hallucination Benchmark (Nov 2025)](https://arxiv.org/abs/2511.03506)
- [MINJA Memory Injection Attack (NeurIPS 2025)](https://openreview.net/forum?id=QVX6hcJ2um)
- [MemoryGraft Persistent Memory Compromise (Dec 2025)](https://arxiv.org/html/2512.16962v1)
- [Palo Alto Unit 42: Indirect prompt injection poisons AI long-term memory](https://unit42.paloaltonetworks.com/indirect-prompt-injection-poisons-ai-longterm-memory/)
- [Multi-tenant AI Infrastructure: 5 Isolation Layers](https://medium.com/@isuruig/multi-tenant-ai-infrastructure-the-5-isolation-layers-that-determine-whether-your-customers-data-stays-separate-340aaeef4922)
- [The Day-30 Problem: agent memory drift](https://cipherbuilds.ai/blog/day-30-agent-memory-problem)

### Infrastructure
- TTT-E2E: "End-to-End Test-Time Training for Long Context" arXiv:2512.23675, and the [NVIDIA writeup: Reimagining LLM Memory](https://developer.nvidia.com/blog/reimagining-llm-memory-using-context-as-training-data-unlocks-models-that-learn-at-test-time/)
- [vLLM PagedAttention](https://docs.vllm.ai/en/latest/design/paged_attention/)
- [Anthropic Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

---

*Next: [Planning and Decomposition](06-planning-and-decomposition.md)*
