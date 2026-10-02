# Context Engineering

Context engineering is the science of filling the LLM's finite "working memory" with the most valuable tokens. With context windows now reaching 1M+ tokens (Claude Sonnet 4.6, Gemini 3.1 Pro, GPT-5.5) and models gaining Extended Thinking, the focus has shifted from "fitting data" to "ranking relevance" and "managing compute budget."

## Table of Contents

- [The Long Context Paradigm (1M+ Tokens)](#long-context)
- [Agentic Context Engineering](#agentic-context-engineering)
- [Extended Thinking & Budget Tokens](#extended-thinking)
- [Lost-in-the-Middle](#lost-in-the-middle)
- [Context Budgeting & Token Awareness](#budgeting)
- [Prompt Caching Economics](#prompt-caching)
- [Contextual Compression (RAD-L)](#compression)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Long Context Paradigm (1M+ Tokens)

Models like Gemini 3.1 Pro (1M), Claude Sonnet 4.6 (1M), Claude Opus 4.7 (1M), and GPT-5.5 (1M) have massive context windows.

**Insight**: "Context is the new RAG."
For datasets under 100,000 documents, it is often more accurate and faster to put the entire dataset in the context window than to use an external vector database. This is called **"In-Context RAG."**

---

## Agentic Context Engineering

Prompt engineering writes one good instruction. **Context engineering** curates the full set of tokens the model sees on **every inference turn** of an agent loop: system prompt, tools, retrieved data, prior tool results, and running message history. The distinction matters because an agent accumulates context turn after turn, so the curation problem is continuous, not one-shot. This is the framework Anthropic, OpenAI, and Google now build their agent harnesses around.

### Context Rot: Why Context Is a Finite Resource

A 1M-token window does not mean you should fill it. Models suffer **context rot**: accuracy degrades as the token count grows, because attention scales with n-squared pairwise relationships and training data skews toward shorter sequences. Treat context as a budget with diminishing returns, not free space. The job is to keep the **smallest high-signal set of tokens** that still lets the model act correctly.

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

### Just-in-Time Loading

Instead of pre-loading every document, the agent holds references and fetches content only when a step needs it. This mirrors how a human works from a file tree: you open the file you need, not the whole repo. It keeps the window small and lets the agent discover structure through exploration. The trade-off is latency, so a hybrid (pre-load the obvious, fetch the rest) is often best.

### Structured Note-Taking (Agentic Memory)

The agent persists notes outside the context window and pulls them back in when relevant. This is what lets an agent stay coherent across a task that is far longer than its window. See [Agent Memory and State](../07-agentic-systems/05-agent-memory-and-state.md) and [Memory Architectures](../08-memory-and-state/01-memory-architectures.md) for the storage substrates (filesystem, vector, graph).

### Sub-Agent Isolation

A coordinator delegates a focused sub-task to a sub-agent that works in its own clean window and returns a condensed summary. The detailed search or analysis context never pollutes the coordinator's window. This is the context-management reason multi-agent systems work, separate from any parallelism benefit. See [Multi-Agent Orchestration](../07-agentic-systems/04-multi-agent-orchestration.md).

---

## Extended Thinking & Budget Tokens

Several frontier models now offer **controllable internal reasoning** before generating a response:

### Claude (Sonnet 4.6, Opus 4.7): Extended Thinking

```python
response = client.messages.create(
    model="claude-3-7-sonnet-20250219",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000  # max internal reasoning tokens
    },
    messages=[{"role": "user", "content": "Refactor this codebase to be async..."}]
)

# Response has two blocks:
# 1. thinking block (visible for debug, not shown to user)
# 2. text block (the actual answer)
for block in response.content:
    if block.type == "thinking":
        print("[THINKING]", block.thinking)
    elif block.type == "text":
        print("[ANSWER]", block.text)
```

**Key parameters:**
- `budget_tokens`: 1,024 → 100,000. Higher = better accuracy, higher cost.
- Thinking tokens billed at standard rates. A 10K thinking budget = +$0.15 per request.
- Streaming works — thinking blocks stream before text.

### o3 (OpenAI) — Reasoning Effort

```python
response = client.chat.completions.create(
    model="o3",
    reasoning_effort="medium",  # "low" | "medium" | "high"
    messages=[{"role": "user", "content": "Prove P=NP or disprove it."}]
)
# Reasoning tokens are invisible — o3 never exposes its internal chain
```

**Effort levels vs cost (approx.):**
| Effort | Speed | Cost multiplier | Best for |
|--------|-------|-----------------|----------|
| low | Fast | 1x | Simple logic, quick lookups |
| medium | Medium | 3-5x | Coding, analysis |
| high | Slow | 8-20x | PhD-level problems, ARC-AGI |

### When to Enable Thinking / Reasoning

| Condition | Recommendation |
|-----------|----------------|
| Complex multi-step code refactoring | ✅ Enable (budget: 8K-20K) |
| Simple Q&A / extraction | ❌ Disable — adds cost & latency |
| STEM / math problems | ✅ Enable (o3-mini medium) |
| High-volume chatbot | ❌ Disable — use standard mode |
| Security-critical decision | ✅ Enable — extra reasoning catches edge cases |

**Production pattern**: Use a complexity classifier to gate Extended Thinking. If query complexity score < 0.5, skip thinking mode entirely (saves 60-80% on reasoning-heavy workloads).

```python
def smart_generate(query: str) -> str:
    complexity = classifier.predict(query)  # 0-1 score
    
    if complexity > 0.7:
        # Enable Extended Thinking for hard problems
        return claude_with_thinking(query, budget_tokens=8000)
    else:
        # Standard fast mode for simple tasks
        return claude_standard(query)
```

---

## Lost-in-the-Middle

In 2023, models lost accuracy for information in the middle of the prompt.
**Status**: Frontier models (Claude Sonnet 4.6, Claude Opus 4.7, Gemini 3.1 Pro, GPT-5.5) perform significantly better, but the **Attention Gradient** still exists.
- **Best Practice**: Place critical instructions and gold-standard examples at the **very beginning** and **very end** of your prompt. Middle = raw data/knowledge chunks.
- **Use chunk ordering**: Rerank retrieved documents so most relevant are first and last.

---

## Context Budgeting & Token Awareness

Every token costs money and increases TTFT (Time to First Token).

| Component | Budget (Tokens) | Why? |
|-----------|-----------------|------|
| **System Prompt** | 500 - 1,000 | Core logic and persona. |
| **History** | 2,000 - 5,000 | Conversational "State." |
| **Data/Search** | 10k - 1M | Depends on task depth. |
| **Output Reserve**| 1,000 - 4,000 | Must reserve space for reasoning. |

---

## Prompt Caching Economics

Almost all major providers (OpenAI, DeepSeek, Anthropic, Google) support **Prefix Caching**.

- **The Crossover**: If you reuse a 100k token context (e.g., a codebase) for more than 2 requests, the caching discount effectively makes it cheaper than RAG.
- **Cache Hits**: $0.05 / 1M tokens.
- **Cache Misses**: $5.00 / 1M tokens.

**The Architectural Choice**: Design your system to keep the "System Prompt + Base Knowledge" static to maintain a 100% cache hit rate.

---

## Contextual Compression (RAD-L)

For extremely long contexts (10M+), we use **Reasoning-Aware Deletion (RAD-L)**.
- **How**: A tiny auxiliary model (0.1B) scans the text and removes "filler" words, common linguistic patterns, and irrelevant sections *before* the prompt is sent to the giant frontier model.
- **Benefit**: Reduces prompt size by 20-50% with <1% drop in accuracy.

---

## Interview Questions

### Q: When would you choose Long Context over RAG?

**Strong answer:**
I choose Long Context when high-fidelity retrieval and cross-document reasoning are critical. RAG suffers from "Retrieval Gap"—if your vector search misses the relevant chunk, the model never sees it. Long Context (up to 2M tokens) provides 100% recall. Specifically, I'd use it for codebase analysis, legal document review, and multi-file financial auditing. I'd stick to RAG for dynamic web-scale data or billion-document datasets that exceed any context window.

### Q: How do you handle the high TTFT associated with million-token prompts?

**Strong answer:**
The primary solution is **Context Caching**. By caching the heavy document on the GPU cluster, the model doesn't have to "re-read" (prefill) the entire 1M tokens for every turn. The TTFT for a cached prompt is nearly the same as for a 1k token prompt. Additionally, for non-cached requests, I would use **Streaming Prefill**, where the model generates an initial summary or "Thought" while it is still processing the latter half of the massive context.

### Q: An agent works fine for short tasks but degrades on long-running ones. How do you fix it?

**Strong answer:**
This is **context rot**: the window fills with stale tool output and the model loses the thread. I would apply agentic context engineering. First, **compaction**: summarize the history at a threshold and continue from the summary plus the most-recent artifacts. Second, **just-in-time loading**: hold file paths and IDs instead of full content, and fetch on demand. Third, **structured note-taking**: have the agent write progress to a scratch file it can re-read, so working memory stays small. For sub-tasks that generate a lot of intermediate detail (deep search, multi-file analysis), I would use **sub-agent isolation** so that detail returns as a short summary instead of flooding the main window. The goal is the smallest high-signal token set per turn, not the largest.

---

## References
- Liu et al. "Lost in the Middle" (2023/2024 update)
- [Anthropic. "Effective context engineering for AI agents" (2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Anthropic. "Effective harnesses for long-running agents" (2026)](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Anthropic. "Extended Thinking: Technical Guide": https://docs.anthropic.com/
- OpenAI. "o3 and o3-mini System Card" (2025)
- [Continuum AI. OrcaPromptVault — system prompts and tool schemas of shipped agents](https://github.com/Continuum-AI-Corp/OrcaPromptVault) — dated wire captures showing how much of the window a production harness consumes before any user content, per model and per entry point

---

*Next: [Structured Generation](06-structured-generation.md)*
