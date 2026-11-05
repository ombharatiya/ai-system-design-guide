# Memory Architectures

LLM memory has evolved from "history buffers" to a **Three-Tiered Cognitive Architecture**. This hierarchy mimics human cognitive systems (L1-L3) to balance speed, cost, and recall capacity. Production agent stacks now lean on dedicated memory layers (Mem0, Zep, Letta, Cognee) or on the memory features of managed agent platforms rather than rolling their own, and in 2026 a third substrate joined vectors and graphs: **plain files**, versioned in git and navigated by path. For the agent-centric four-layer view (adding procedural memory), see [Agent Memory and State](../07-agentic-systems/05-agent-memory-and-state.md).

## Table of Contents

- [The Three-Tiered Hierarchy](#the-three-tiered-hierarchy)
- [Tier 1: Working Memory (L1)](#tier-1-working-memory-l1)
- [Tier 2: Episodic Memory (L2)](#tier-2-episodic-memory-l2)
- [Tier 3: Semantic Memory (L3)](#tier-3-semantic-memory-l3)
- [File-Based Memory: Retrieval by Path](#file-based-memory-retrieval-by-path)
- [Memory Consolidation Patterns](#memory-consolidation-patterns)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Three-Tiered Hierarchy

| Tier | Type | Human Analogy | Technology | Latency |
|------|------|---------------|------------|---------|
| **L1** | Working Memory | Immediate focus | Context Window / KV Cache | <50ms |
| **L2** | Episodic Memory | Past experiences | Vector DB / raw trajectory logs | 100-300ms |
| **L3** | Semantic Memory | General knowledge | Fact store (Mem0), temporal graph (Zep/Graphiti), SQL, memory files | >500ms |

---

## Tier 1: Working Memory (L1)

L1 is the **active focus** of the model.
- **Context Window**: about 1M tokens on current frontier models (Claude Opus 5.5 and Sonnet 5.5 at 1M, GPT-6.1 Sol at 1.05M, Gemini 4 Argon at 1M shared with output); fast tiers match it (Gemini 3.8 Flash at 1,048,576, GPT-6 Luna at 1.05M); 128K to 1M on open-weight models.
- **KV Cache**: The GPU "RAM" that stores pre-computed keys and values.
- **Management Strategy**: **Sliding Windows**, **Compaction**, and **Prefix Caching** (vLLM/PagedAttention, SGLang RadixAttention).
- **Redundancy Note**: We only keep the most recent turns and critical system instructions in L1.

---

## Tier 2: Episodic Memory (L2)

L2 stores "What happened previously" in this session or past sessions with this user.
- **Storage**: Vector databases (Pinecone, Weaviate, Qdrant), with raw logs in object storage.
- **Retrieval**: Semantic search. If the user asks "What did we talk about last Tuesday?", L2 provides the answer.
- **Pattern**: **Experience Replay**. Agents retrieve successful past trajectories to guide current decisions.

---

## Tier 3: Semantic Memory (L3)

L3 stores **Durable Facts** and **Learned Rules** that should hold until something supersedes them.
- **Knowledge Graphs**: Store relationships (e.g., `User` -- `WORKS_FOR` --> `Company_X`), with validity intervals when "when did this become true?" matters (Zep/Graphiti).
- **Mem0**: An open-source library and managed platform that extracts facts (e.g., "User likes Dark Mode") and makes them available across sessions. Since v2.0.0 (April 2026) it stores them in your vector store with a parallel entity index rather than a graph database.
- **Truth Anchoring**: L3 acts as the "Ground Truth" when L1 and L2 provide conflicting information, which only works if conflicts are resolved somewhere (at write time, in a scheduled consolidation job, or at read time).

---

## File-Based Memory: Retrieval by Path

The biggest architectural shift of 2026 is memory stored as **files the agent reads and edits with ordinary tools**, instead of rows retrieved by embedding similarity.

| System | How it works |
|--------|--------------|
| **Letta MemFS** | Letta archived its MemGPT-era server on August 16, 2026. All agents now use MemFS: long-term memory is a git repository of Markdown files with YAML frontmatter. Files at the memory root load into the system prompt every turn; directories with their own `MEMORY.md` index stay out of context and are signposted. No vector index by default. |
| **Claude memory tool and Managed Agents memory stores** | A `/memories` directory the model reads and writes through a tool; Managed Agents mount stores per session, and self-hosted sandboxes gained memory stores on August 19, 2026. |
| **Coding agents** | `CLAUDE.md`, `AGENTS.md`, and skill files: project memory that humans can review in a pull request. |

**Why it is winning for agents**: humans can read, diff, and review it; git gives versioning and rollback for free; and the agent navigates by structure (index files, directories) rather than hoping the right chunk scores highest. **Where it loses**: large or high-churn fact sets, cross-user aggregation, and fuzzy recall over thousands of episodes, which is still vector territory.

---

## Memory Consolidation Patterns

Memories move between tiers via **Consolidation**:
1. **Extraction**: An LLM "Reviewer" extracts facts from L1 at the end of a session.
2. **Indexing**: Facts are stored in L2 (as vectors) and L3 (as facts, graph edges, or memory files).
3. **Decay**: Old, non-reinforced episodic memories are archived to cold storage or deleted; promoted facts in L3 are superseded, not silently decayed.

The design axis interviewers now probe is **when** consolidation runs:

| Timing | Example | Tradeoff |
|--------|---------|----------|
| **Write time** | Mem0 v2: one ADD-only extraction call per `add()` | Cheap reads; extraction errors are baked in |
| **Sleep time** | Anthropic Dreams (research preview), Mem0's Dream (paid plans) | Off the request path; staleness between runs |
| **Read time** | Just-in-Time Memory (arXiv 2609.27334): keep raw trajectories, curate per task | Beat the strongest write-time baseline by 3.9 to 16.3 success-rate points; curation cost moves onto reads |

More memory is not automatically better: on MemTrapBench (arXiv 2608.20202) tasks built to trigger memory traps, every memory strategy it tested underperformed the no-memory setting because correct, relevant memories still caused reasoning fixation. Gate memory injection, and evaluate task success with and without it.

The baseline to beat is plain retrieval over raw history. arXiv 2610.30755 ("Is Complex Agentic Memory Always Needed?", October 22, 2026) ran Raw-RAG (dense retrieval plus reranking over the raw conversation log) against 12 memory systems, including Mem0, A-MEM, MemoryOS and MemMachine, on LoCoMo, LongMemEval, MEME and MemoryAgentBench. Raw-RAG ranked first in 6 of 7 benchmark-model settings, by 0.13 to 5.53 points (LoCoMo with GPT-4o-mini: 89.07 vs MemMachine's 88.12), and readers given full access to the transformed memory stores still trailed the raw evidence. The margins are small; the point is that extraction, summarization and graphs often buy nothing, so ship a Raw-RAG baseline first and make every memory layer prove it beats that.

---

## Interview Questions

### Q: Why not just use a 1M-token context window for all memory (L1-L3)?

**Strong answer:**
While possible, it is **Economically and Cognitively inefficient**.
1. **Cost**: A 1M-token call to Claude Opus 5.5 costs $4.00 of input uncached, or $0.20 even fully cached, on every turn; a 10K-token call with retrieved context costs $0.04. Caching narrowed the gap but did not close it, and OpenAI bills the whole request at long-context rates once input passes 272K tokens.
2. **Attention Dilution**: Even with "Long Context" models, "Lost in the Middle" and context rot remain factors. If the context is cluttered with irrelevant historical turns, the model's reasoning on the *current* task degrades.
3. **Latency**: TTFT (Time to First Token) scales with uncached context size, and decoding over a long context is slower per token.
A staff-level architecture uses **Strategic Retrieval** to keep the context window lean and focused.

### Q: How do you handle "Privacy Leakage" in Tier 3 (Global Semantic Memory)?

**Strong answer:**
Tier 3 (Semantic Memory) must be **Sharded by Namespace**. Each user or organization gets a unique `namespace_id` in the vector DB and Knowledge Graph, derived from the authenticated identity, never from a caller-supplied label. LangGraph's Postgres and SQLite stores showed why: they joined namespace labels with ".", so `("tenant.bob",)` resolved to the same namespace as `("tenant", "bob")`, and only `put` rejected dotted labels (CVE-2026-115883, October 21, 2026; see [LangGraph Orchestration](../09-frameworks-and-tools/02-langgraph-orchestration.md) for fixed versions). We implement **RLS (Row Level Security)** at the database layer underneath, so a key-construction bug cannot become a cross-tenant read. Additionally, we use a **PII-Scrubbing Layer** during the Consolidation step to ensure that sensitive data (passwords, PII) never moves from the transient L1 context into the persistent L3 knowledge store. If the requirement is that even the operator cannot read the memory, the reference design is client-held keys plus processing in a hardware enclave: Google described (September 23, 2026, as a plan with no product dates) server-side memory for its Private AI Compute platform where storage keys live only on the user's devices, an attested enclave decrypts per request and re-encrypts, and the server software is published to a tamper-proof public record so devices can verify it before sending data. That is the same pattern as Apple's Private Cloud Compute.

### Q: When would you store agent memory as files instead of in a vector database?

**Strong answer:**
Files when the memory is **small, curated, and needs human oversight**: project conventions, user preferences, procedures, and decisions. A few hundred Markdown files with an index fit the agent's navigation style, every change is a reviewable git diff, and rollback is a revert. That is why Letta moved all agents to git-backed MemFS and why coding agents converged on `CLAUDE.md` and `AGENTS.md`. Vectors when the memory is **large, append-heavy, and recalled fuzzily**: thousands of past conversations or trajectories where you cannot know in advance which ones matter. Most production systems end up with both: files for the semantic and procedural layer, vectors (or raw logs with read-time curation) for the episodic layer. One operational catch with files: deleting a user's data under GDPR means purging git history, not just the current commit.

---

## References
- Park et al. "Generative Agents: Interactive Simulacra of Human Behavior" (2023)
- Packer et al. "MemGPT: Towards LLMs as Operating Systems" (2023)
- [Mem0 v2.0.0 release notes](https://github.com/mem0ai/mem0/releases/tag/v2.0.0)
- [Letta Code repository (MemFS, git-tracked memory)](https://github.com/letta-ai/letta-code); the retired V1 server lives on the archive branch of [letta-ai/letta](https://github.com/letta-ai/letta).
- [Just-in-Time Memory (arXiv 2609.27334)](https://arxiv.org/abs/2609.27334)
- [MemTrapBench (arXiv 2608.20202)](https://arxiv.org/abs/2608.20202)
- [Is Complex Agentic Memory Always Needed? Naive RAG Is Stronger Than We Thought (arXiv 2610.30755)](https://arxiv.org/abs/2610.30755)
- [Google DeepMind. "Advancing Private AI Compute with secure server-side memory" (September 2026)](https://deepmind.google/blog/advancing-private-ai-compute-with-secure-server-side-memory/)

---

*Next: [Short-Term Context Management](02-short-term-context.md)*
