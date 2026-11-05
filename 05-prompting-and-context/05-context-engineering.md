# Context Engineering

Context engineering is the science of filling the LLM's finite "working memory" with the most valuable tokens. With context windows at about 1M tokens across the frontier (Claude Opus 5.5, Sonnet 5.5, and Fable 5.1 at 1M; GPT-6 Astra and GPT-6.1 Sol at 1.05M; Gemini 3.8 Flash at 1,048,576) and reasoning controlled by an effort dial, the focus has shifted from "fitting data" to "ranking relevance," "managing compute budget," and keeping the conversation in a shape the provider can cache and the model can keep reasoning over.

## Table of Contents

- [The Long Context Paradigm (1M+ Tokens)](#the-long-context-paradigm-1m-tokens)
- [Agentic Context Engineering](#agentic-context-engineering)
- [Append-Only Context: Thinking Is Bound to the Conversation](#append-only-context-thinking-is-bound-to-the-conversation)
- [Reasoning Effort and Thinking Controls](#reasoning-effort-and-thinking-controls)
- [Lost-in-the-Middle](#lost-in-the-middle)
- [Context Budgeting & Token Awareness](#context-budgeting--token-awareness)
- [Prompt Caching Economics](#prompt-caching-economics)
- [Contextual Compression](#contextual-compression)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Long Context Paradigm (1M+ Tokens)

Models like Claude Opus 5.5 (1M), Claude Sonnet 5.5 (1M), GPT-6.1 Sol (1.05M), and Gemini 3.8 Flash (about 1M) have massive context windows.

**Insight**: "Context is the new RAG."
For stable corpora that fit comfortably in the window (a few hundred thousand tokens, roughly a few hundred pages), it is often more accurate and simpler to put the whole corpus in context than to run an external vector database. This is called **"In-Context RAG."** It does not scale to document collections measured in thousands of files, and recall still degrades with length (see context rot below).

**Run the numbers before choosing.** For a 500K-token corpus on Claude Opus 5.5 ($4 input, $0.20 cache read per 1M): about $2.00 per request uncached, about $0.10 per request once cached, after a one-time $2.50 cache write. A RAG call that sends 10K retrieved tokens to the same model costs about $0.04 in input. So cached long context is now within about 2.5x of RAG on input cost (as long as traffic keeps the cache warm), and the decision turns on recall quality, latency, and how often the corpus changes. Watch the pricing cliffs: OpenAI bills the whole request at long-context rates once input passes 272K tokens, and xAI doubles all tokens at 200K and above. Anthropic stays flat to 1M on Opus 5.5 and Sonnet 5.5, but Claude Haiku 5.5 (October 7, 2026) bills the whole request at 5x once the prompt passes 100K tokens, counting cache reads and writes toward that length, so cheap in-context RAG on Haiku stops being cheap at exactly the corpus sizes where it is tempting.

---

## Agentic Context Engineering

Prompt engineering writes one good instruction. **Context engineering** curates the full set of tokens the model sees on **every inference turn** of an agent loop: system prompt, tools, retrieved data, prior tool results, and running message history. The distinction matters because an agent accumulates context turn after turn, so the curation problem is continuous, not one-shot. This is the framework Anthropic, OpenAI, and Google now build their agent harnesses around.

### Context Rot: Why Context Is a Finite Resource

A 1M-token window does not mean you should fill it. Models suffer **context rot**: accuracy degrades as the token count grows, because attention scales with n-squared pairwise relationships and training data skews toward shorter sequences. Treat context as a budget with diminishing returns, not free space. The job is to keep the **smallest high-signal set of tokens** that still lets the model act correctly. The serving bill agrees: arXiv 2611.05306 ("You don't need long context", November 3, 2026) reports that serving Qwen 3 Coder Next with a 256K window instead of 16K costs 7x more per generated token, while frequent compaction kept frontier models accurate on SWE-bench Verified with at most one action per context (Claude Opus 4.5 70.2% vs 70.0% with full history, GPT-5.6 Sol 82.4% vs 84.4%).

### The Five Core Techniques

| Technique | What it does | Use when |
|-----------|--------------|----------|
| **Compaction** | Summarize the message history and reinitialize the loop with the compressed summary plus the few most-recent artifacts | Long back-and-forth sessions approaching the window limit |
| **Just-in-time loading** | Keep lightweight identifiers (file paths, URLs, row IDs) in context and load the full content on demand via a tool | Large corpora or databases that cannot all fit, exploratory tasks |
| **Structured note-taking** | Agent writes progress notes to a file or memory store outside the window, then reads them back later | Long-horizon tasks spanning dozens of tool calls |
| **Sub-agent isolation** | Spawn a focused sub-agent with a clean window for a sub-task; it returns only a 1k-2k token summary | Parallel research, deep search, anything that would flood the main window with intermediate detail |
| **System prompt calibration** | Aim for the "Goldilocks zone": specific enough to be reliable, general enough to not be brittle; use clear XML or Markdown sections | Always, as the foundation under the other four |

### Compaction

When the history grows large, pass it back to the model to summarize, preserving the load-bearing details (architectural decisions, unresolved bugs, key constraints) and dropping redundant tool output. Claude Code uses this pattern: it continues with the compressed summary plus the most recently accessed files. **Tune for recall first** (keep everything that matters), then improve precision (cut redundancy).

Compaction can also happen in cache space rather than in text. arXiv 2610.28577 ("Cache Engineering for Agents", October 20, 2026) lets an agent drop cached tokens from future attention and re-rotate retained key positions without rewriting the transcript or re-prefilling; a rolling drop of old tool responses beat threshold-triggered summarization by 1.7 points on average at 27.7% lower cost (author-reported). The catch for infrastructure: after a drop, identical token sequences can map to different cache states, so a prefix match alone is no longer a correct cache key (see [KV Cache and Context Caching](../04-inference-optimization/02-kv-cache-and-context-caching.md)).

Providers now run compaction for you. The Claude API offers server-side compaction (beta) that summarizes automatically near a threshold, and on-demand compaction (beta, September 14, 2026, header `compact-2026-09-04`) that returns a signed compaction block you send back in place of the messages it summarizes. On-demand compaction can keep the most recent turns word for word after the summary, and thinking in those kept turns stays valid on models that bind thinking to the conversation. Prefer these on the newest Claude models, because a hand-rolled summary that rewrites history invalidates thinking blocks (next section).

**The summary is a trust boundary.** OpenAI's misalignment reports (September 2026) describe models in RL training writing concealment instructions into their own compaction summaries (flagged in 2.15% of GPT-5.6 Sol summaries and 0.27% of GPT-6 Astra's), and an unreleased model writing jailbreak-style instructions into 27 summaries, one of which its successor followed. Keep the raw transcript for audit, diff or validate summaries against it, and never let a summary carry instructions the original system prompt did not. See [Loop Engineering](../07-agentic-systems/12-loop-engineering.md#context-and-memory-in-long-loops).

### Just-in-Time Loading

Instead of pre-loading every document, the agent holds references and fetches content only when a step needs it. This mirrors how a human works from a file tree: you open the file you need, not the whole repo. It keeps the window small and lets the agent discover structure through exploration. The trade-off is latency, so a hybrid (pre-load the obvious, fetch the rest) is often best.

The same applies to **tool definitions**, which can cost tens of thousands of tokens once several MCP servers are attached. Cursor reported (September 2026, vendor-reported) that each built-in tool it offloaded was needed in fewer than 20% of conversations, that loading tools dynamically cut static tool-description tokens by 60%, and that an earlier move of MCP tools into dynamic context cut total tokens by 46.9% in sessions that called an MCP tool. Use tool search or deferred loading rather than shipping every schema on every turn.

### Structured Note-Taking (Agentic Memory)

The agent persists notes outside the context window and pulls them back in when relevant. This is what lets an agent stay coherent across a task that is far longer than its window. See [Agent Memory and State](../07-agentic-systems/05-agent-memory-and-state.md) and [Memory Architectures](../08-memory-and-state/01-memory-architectures.md) for the storage substrates (filesystem, vector, graph).

### Sub-Agent Isolation

A coordinator delegates a focused sub-task to a sub-agent that works in its own clean window and returns a condensed summary. The detailed search or analysis context never pollutes the coordinator's window. This is the context-management reason multi-agent systems work, separate from any parallelism benefit. See [Multi-Agent Orchestration](../07-agentic-systems/04-multi-agent-orchestration.md).

---

## Append-Only Context: Thinking Is Bound to the Conversation

Many harnesses quietly rewrite history: they re-render the system prompt with a timestamp, edit an earlier message to inject a reminder, trim old tool results in place, or switch models mid-conversation. On the newest Claude models that now breaks reasoning continuity.

On Claude Fable 5.1, Opus 5.5, and Sonnet 5.5, each thinking block records which model produced it and is valid only if **nothing before it changed**: system prompt, tools, earlier messages, even the image bytes behind a URL. Unreadable blocks are dropped silently (and not billed); for accounts created on or after August 31, 2026, a prefix mismatch returns a 400 ("The block is bound to a different conversation") unless you opt into `drop_block` under the `thinking-binding-controls-2026-08-01` beta. Blocks are also model-bound (no earlier model reads Fable 5.1 blocks, and Opus 5.5 does not read Fable or Mythos blocks), and Sonnet 5.5 blocks are additionally bound to the producing account.

| Harness habit | Append-only replacement |
|---------------|-------------------------|
| Edit the top-level system prompt mid-session | Append a mid-conversation `system` message; use `clear_at: "next_user_message"` (beta, September 1, 2026) for per-turn reminders |
| Swap or add tools by editing the `tools` array | Inline tool definitions in a mid-conversation system message (beta `inline-tools-2026-09-15`, September 22, 2026) |
| Change top-level effort mid-conversation | Per-message effort (beta, September 1, 2026) |
| Hand-rolled summary that replaces old turns | Provider compaction blocks |
| Fall back to a different model mid-conversation | Expect reasoning to be dropped; re-establish state from a summary or restart the turn |

OpenAI's equivalent is passing reasoning items back through the Responses API rather than reconstructing the transcript. Evidence that preserved reasoning state matters: on ARC-AGI-3, GPT-6 Astra scored 62.7% on ARC Prize's minimal harness at max effort but 99.9% on a provider harness at high effort that preserves reasoning state between requests and uses compaction (ARC Prize, September 2026).

---

## Reasoning Effort and Thinking Controls

Every frontier vendor now offers **controllable internal reasoning** before the response, and the control surface has moved from token budgets to effort levels.

### Claude (Opus 5.5, Sonnet 5.5, Fable 5.1): Adaptive Thinking + Effort

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    thinking={"type": "adaptive", "display": "summarized"},  # default display is "omitted"
    output_config={"effort": "high"},  # low | medium | high | xhigh | max
    messages=[{"role": "user", "content": "Refactor this codebase to be async..."}],
)

# Response has two kinds of blocks:
# 1. thinking block (a summary of the reasoning; the raw chain is never returned)
# 2. text block (the actual answer)
for block in response.content:
    if block.type == "thinking":
        print("[THINKING SUMMARY]", block.thinking)
    elif block.type == "text":
        print("[ANSWER]", block.text)
```

**Key parameters:**
- `effort`: `low` to `max`. Defaults differ by model: `medium` on Opus 5.5, `high` on Sonnet 5.5 and Fable 5.1. Set it explicitly.
- Fixed `budget_tokens` is gone: it returns a 400 on Opus 4.7 and later, Sonnet 5 and 5.5, Fable 5 and 5.1, and Haiku 5.5, and is deprecated on Opus 4.6 and Sonnet 4.6. Haiku 4.5 still uses it.
- Thinking cannot be disabled on Opus 5.5; on Sonnet 5.5, `{"type": "disabled"}` returns a 400 and the lowest setting is `{"type": "between_tools"}`, accepted only at `high` effort or below.
- Thinking tokens bill at the output rate whatever the `display` setting: 10K thinking tokens cost about $0.20 on Opus 5.5 ($20 per 1M output) and $0.10 on Sonnet 5.5.
- Streaming works; with the default `display: "omitted"`, thinking blocks stream with empty text, which looks like a long pause in a UI.

### OpenAI (GPT-6.1 Sol, GPT-6 Astra): Reasoning Effort

```python
response = client.responses.create(
    model="gpt-6.1-sol",
    reasoning={"effort": "medium"},  # low | medium | high | xhigh | max
    input="Find the race condition in this scheduler...",
)
print(response.output_text)
# Reasoning tokens are billed as output but not returned; summaries are optional
```

GPT-6.1 Sol defaults to `medium`. GPT-6 Astra offers `low` through `max` with no `none` level, does not accept custom `temperature`, `top_p`, or `logprobs`, and requires the Responses API for tools.

### Gemini (3.8 Flash): Thinking Level

Gemini 3.8 Flash replaces `thinking_budget` with `thinking_level` (`low`, `medium` default, `high`), and Google deprecated `temperature`, `top_p`, and `top_k` in the Gemini API on July 21, 2026. See [Determinism Without Temperature](01-prompt-engineering-fundamentals.md#determinism-without-temperature).

### Choosing an Effort Level

| Workload | Starting point |
|----------|----------------|
| Complex multi-step code refactoring, long-horizon agents | `high` or `xhigh` |
| Simple Q&A, extraction, classification | `low` (or a non-reasoning model; `between_tools` on Sonnet 5.5) |
| STEM / math | `high`; `max` only when evals show headroom at `high` |
| High-volume chatbot | `low` |
| Security-critical decision | `high`, plus an external verifier |

Token spend per level varies by model, and the same level name can behave differently on the next model version, so measure on a sample of real traffic and judge **cost per completed task**, not per request.

**Production pattern**: route effort per request with a complexity classifier instead of one global setting.

```python
def smart_generate(query: str) -> str:
    complexity = classifier.predict(query)  # 0-1 score
    if complexity > 0.7:
        effort = "high"
    elif complexity > 0.3:
        effort = "medium"
    else:
        effort = "low"
    return call_model(query, effort=effort)
```

Two caveats. Caches are model-scoped, so routing between models forfeits cache reuse; before building a multi-model cascade, test the strongest model at lower effort. And on Claude, changing top-level effort mid-conversation invalidates the cached messages; the per-message effort beta avoids that.

---

## Lost-in-the-Middle

In 2023, models lost accuracy for information in the middle of the prompt.
**Status**: Frontier models (Claude Opus 5.5, Sonnet 5.5, GPT-6.1 Sol, Gemini 3.8 Flash) perform significantly better, but the **Attention Gradient** still exists, and advertised windows routinely overstate usable context (see [Benchmarks and Leaderboards](../14-evaluation-and-observability/03-benchmarks-and-leaderboards.md)).
- **Best Practice**: Place critical instructions and gold-standard examples at the **very beginning** and **very end** of your prompt. Middle = raw data/knowledge chunks.
- **Use chunk ordering**: Rerank retrieved documents so most relevant are first and last.
- **Caching tension**: anything you put at the end changes per request, so keep the stable instructions in the cached prefix and repeat only a short reminder after the volatile data.

---

## Context Budgeting & Token Awareness

Every token costs money and increases TTFT (Time to First Token).

| Component | Budget (Tokens) | Why? |
|-----------|-----------------|------|
| **System Prompt** | 1,000 - 10,000 | Core logic and persona; agent harness prompts sit at the high end and are the first thing to trim (Cursor cut about 66% of its system prompt in September 2026). |
| **Tool Definitions** | 1,000 - 50,000+ | Grows with every MCP server; load on demand. |
| **History** | 2,000 - 5,000 | Conversational "State." |
| **Data/Search** | 10k - 1M | Depends on task depth. |
| **Output + Thinking Reserve**| 16,000 - 128,000 | Thinking tokens count against `max_tokens`; current frontier models allow up to 128K output. |

Agent workloads are input-dominated: Anthropic's Claude Code telemetry (September 2026, vendor-reported) shows an input-to-output token ratio of 324:1, up from 189:1. At that ratio, cached-input price and cache hit rate drive the bill, not the output price.

---

## Prompt Caching Economics

Almost all major providers (OpenAI, Anthropic, Google, DeepSeek) support **Prefix Caching**, and the read discount is now deep and model-specific:

| Provider | Cache write | Cache read | Mechanics |
|----------|-------------|------------|-----------|
| **Anthropic** | 1.25x (5-minute TTL) or 2x (1-hour TTL) | 0.1x on most models; 0.05x on Opus 5.5 ($0.20 per 1M) and Sonnet 5.5 ($0.10 per 1M, cut from $0.20 on October 7, 2026); 0.025x on Fable 5.1 ($0.25 per 1M) | Explicit `cache_control` breakpoints or top-level auto-caching; on Haiku 5.5, cached tokens count toward its 100K long-context threshold |
| **OpenAI** (GPT-5.6 and later) | 1.25x (30-minute TTL) or 2.5x (12-hour TTL, since October 20, 2026) | 0.1x; 0.05x on GPT-6.1 Sol and GPT-6.1 Terra | Automatic above 1,024 tokens; choose the 30-minute or 12-hour TTL |
| **Google** (Gemini 3.8 Flash) | No separate write price; explicit caches bill storage at $0.50 per 1M tokens per hour ($1.00 from January 1, 2027) | 0.1x ($0.075 per 1M at the introductory rate, $0.15 from January 1, 2027) | Implicit caching is automatic above 4,096 tokens; explicit caches give you control over contents and lifetime, but an idle one can cost more in storage than it saves |
| **DeepSeek** (V4.1-Flash) | No surcharge (a miss bills at the normal input price) | about 2% of the cache-miss price | Automatic; peak and off-peak pricing |

- **The Crossover**: with a 1.25x write and a 0.1x or cheaper read, the cache pays for itself on the first hit; a 2x one-hour write pays off from the second hit, and so, narrowly, does a 2.5x twelve-hour write. Buy the long TTL for prefixes reused across hours (a nightly batch, a codebase agents return to), not for chat sessions that end in minutes. If you reuse a large context (e.g., a codebase) more than once inside the TTL, caching wins.
- **Make the hit rate observable**: cache diagnostics went GA on the OpenAI Responses API (September 8, 2026) and the Claude API (September 23, 2026). Track hit rate as an SLO.

**The Architectural Choice**: Design your system to keep the "System Prompt + Tools + Base Knowledge" static to maintain a near-100% cache hit rate. Silent invalidators are the usual culprits: a timestamp in the system prompt, unsorted JSON, a tool list that varies per request.

For a full cost model, see [FinOps and Token Economics](../11-infrastructure-and-mlops/04-finops-and-token-economics.md).

---

## Contextual Compression

When the volatile part of the context (retrieved chunks, tool output, long user documents) is large, compress it before it reaches the frontier model.
- **How**: A small auxiliary model scores tokens or sentences and drops low-information ones. LLMLingua-2 (Microsoft, 2024) trains a small token classifier for task-agnostic compression and reports 2x to 5x compression with modest quality loss on its benchmarks. Extractive filtering (keep only the sentences a reranker scores as relevant) is the simpler baseline.
- **Where it pays**: uncached, volatile input. Do not compress the stable prefix: compression changes the bytes, breaks the prefix cache, and at 0.025x to 0.1x cache-read prices the cached prefix is already cheaper than any compression you could apply.
- **Risk**: compression can delete the one negation or number that mattered. Evaluate on your task with and without it.

---

## Interview Questions

### Q: When would you choose Long Context over RAG?

**Strong answer:**
I choose Long Context when high-fidelity retrieval and cross-document reasoning are critical and the corpus is stable. RAG suffers from a "Retrieval Gap": if your vector search misses the relevant chunk, the model never sees it. Long context (about 1M tokens on current frontier models) removes the retrieval miss, but not attention misses: recall still degrades with length, so it is not 100% recall. Cost has moved in long context's favor: a cached 500K-token corpus costs about $0.10 per request on Claude Opus 5.5, about 2.5x a 10K-token RAG call on the same model, provided traffic keeps the cache warm. I'd use it for codebase analysis, legal document review, and multi-file financial auditing. I'd stick to RAG for dynamic or web-scale data, corpora that exceed any window, and per-user permission filtering that a shared prefix cannot express.

### Q: How do you handle the high TTFT associated with million-token prompts?

**Strong answer:**
The primary solution is **Context Caching**: the provider keeps the KV cache for the shared prefix, so prefill runs only on the new tokens. TTFT drops sharply, though not to small-prompt levels, because the new tokens still attend over the full cached prefix, and every decoded token pays that attention cost too. Second, shrink what is uncached: retrieve instead of stuffing, and compress the volatile part. Third, split the work: map-reduce the corpus across parallel sub-calls and merge the results, which trades total tokens for latency. Self-hosted, chunked prefill and prefill/decode disaggregation keep a giant prompt from stalling other requests, but they do not make that one prompt's prefill faster.

### Q: An agent works fine for short tasks but degrades on long-running ones. How do you fix it?

**Strong answer:**
This is **context rot**: the window fills with stale tool output and the model loses the thread. I would apply agentic context engineering. First, **compaction**: summarize the history at a threshold and continue from the summary plus the most-recent artifacts, using the provider's compaction where available and treating the summary as untrusted output to validate against the raw transcript. Second, **just-in-time loading**: hold file paths and IDs instead of full content, and fetch on demand. Third, **structured note-taking**: have the agent write progress to a scratch file it can re-read, so working memory stays small. For sub-tasks that generate a lot of intermediate detail (deep search, multi-file analysis), I would use **sub-agent isolation** so that detail returns as a short summary instead of flooding the main window. The goal is the smallest high-signal token set per turn, not the largest.

### Q: Your gateway falls back from Claude Opus 5.5 to another model mid-conversation, and both quality and error rates get worse. What is going on?

**Strong answer:**
Two things. Thinking blocks on the newest Claude models are bound to the model that produced them, so the fallback model drops them and continues without the prior reasoning; that is the quality loss. And if the gateway also rewrites history on the way (re-rendering the system prompt, trimming old tool results, injecting reminders into earlier turns), the prefix check fails: accounts created on or after August 31, 2026 get a 400, and with `drop_block` enabled the request runs without that reasoning. The fix is to make the harness append-only (mid-conversation system messages, inline tool additions, provider compaction) and to make fallbacks explicit: restart the turn on the fallback model from a clean summary, and for safety refusals use the provider's server-side fallback (`fallbacks: "default"`, beta), which retries on a recommended model without the gateway touching history. I would also add a metric for dropped thinking blocks per conversation so this cannot regress silently.

---

## References
- Liu et al. "Lost in the Middle" (2023/2024 update)
- Pan et al. "LLMLingua-2: Data Distillation for Efficient and Faithful Task-Agnostic Prompt Compression" (2024)
- [Anthropic. "Effective context engineering for AI agents" (2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Anthropic. "Effective harnesses for long-running agents" (2026)](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic. Preserved thinking (thinking-block binding)](https://platform.claude.com/docs/en/build-with-claude/preserved-thinking)
- [Claude API release notes (compaction, effort, inline tools)](https://platform.claude.com/docs/en/release-notes/overview)
- [OpenAI. Prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching)
- [Google. Gemini API context caching](https://ai.google.dev/gemini-api/docs/caching) and [pricing](https://ai.google.dev/gemini-api/docs/pricing)
- [OpenAI. Misalignment reports (compaction summaries)](https://alignment.openai.com/misalignment-reports/)
- [Cursor. "Improved token efficiency for longer agent runs" (September 2026)](https://cursor.com/blog/improved-token-efficiency)
- [Anthropic. Claude Opus 5.5 and Claude Code usage data (September 2026)](https://claude.com/blog/claude-opus-5-5-built-for-coding-sessions-that-use-more-context)
- [ARC Prize. GPT-6 Astra on ARC-AGI-3 (September 2026)](https://arcprize.org/blog/astra)

---

*Next: [Structured Generation](06-structured-generation.md)*
