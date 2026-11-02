# Agent Fundamentals

Agents are LLM-powered systems that move beyond "chat" into "autonomous problem solving." The definition has shifted from simple ReAct loops to **Closed-Loop Reasoning Systems** that use built-in "System 2" thinking: adaptive thinking with an effort dial on Claude Opus 5.5 and Fable 5.1, five reasoning-effort levels (low to max) on GPT-6 Astra, and thinking levels on Gemini 3.8 Flash.

## Table of Contents

- [The Agent Formula](#the-agent-formula)
- [System 1 (LLM) vs. System 2 (Reasoning Model)](#system-1-vs-system-2-thinking)
- [Agency Levels (Autonomous Spectrum)](#agency-levels)
- [Core Components](#core-components)
- [The Agent Lifecycle](#the-agent-lifecycle)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Agent Formula

Modern agency is often described as:
`Agent = Reasoning Model + Tool Use + Persistent Memory + Environment Feedback`

**Nuance**: In 2023, agents were "wrappers" around chat models. Today, agents are increasingly **Integrated**. Frontier models (Claude Opus 5.5, GPT-6 Astra, Gemini 3.8 Flash) have the "Thinking" process trained in through reinforcement-learning post-training and interleave it between tool calls, making the agent loop more stable and less prone to "stalling."

---

## System 1 vs. System 2 Thinking

Architecting an agent requires choosing the right "Thinking Mode":

| Mode | Cognitive Type | Analogy | Current stack (November 2026) |
|------|----------------|---------|---------------|
| **System 1** | Fast, intuitive, reactive | Reflexes | Claude Haiku 5.5 / GPT-6 Luna / Gemini 3.8 Flash at `low` thinking |
| **System 2** | Slow, logical, planning | Deliberation | Claude Opus 5.5 or Fable 5.1 at high effort / GPT-6 Astra or GPT-6.1 Sol at high effort or above |

**The Design Pattern**: Use System 1 models for "Fast UI" and "Routing." Use System 2 models for "Decision Gates" and "Complex Planning."

A third shape arrived in October 2026: **decision models** (sometimes marketed as "System One models") that return a typed answer, such as a predicate, a choice, or a score, with a probability instead of text. OpenAI's Decisions API went GA on October 22 on `gpt-6-luna` (1 to 400 questions per request, $0.10 per 1M uncached input, output not charged), vLLM and SGLang added native endpoints, and Ollama added local decision models. They fit routers and gates well because the output needs no parsing, but papers report underconfident probabilities, so calibrate thresholds on your own labeled traffic before trusting them.

**The line is now a dial, not a model swap.** On the newest frontier models thinking cannot be fully switched off: Opus 5.5 rejects disabling it, Sonnet 5.5's lowest setting is `between_tools`, and GPT-6 Astra has no `none` effort. You set depth per call with the effort parameter, which means effort is part of your agent's configuration and must be pinned. Defaults move under you: Opus 5.5 defaults to `medium` effort where Opus 5 defaulted to `high`, so an unpinned migration silently changes latency and quality.

**Harness defaults move too, and they swap the model itself.** OpenAI Agents SDK 0.20.0 made `gpt-5.6-luna` its implicit default model, Codex CLI switched its default to GPT-6.1 Sol on September 29, and Claude Code moved its default Opus and Sonnet to the 5.5 models within days of each launch (and its default Haiku to Haiku 5.5 in 2.1.293, October 2026). Small model swaps carry API changes, too: Haiku 5.5 rejects `temperature`, `top_p`, `top_k`, and assistant prefill, and bills the whole request at 5x once a prompt passes 100,000 tokens. Pin model IDs as well as effort, and re-run evals on every SDK or harness upgrade.

---

## Agency Levels

Not every autonomous system is an "Agent." We categorize them by the **Level of Agency**:

1. **L0: Scripted Chains**: Fixed sequence (e.g., standard LangChain).
2. **L1: Tool-Enabled**: Model picks a tool but doesn't plan.
3. **L2: ReAct Agent**: Simple loop of "Thought -> Action -> Observation."
4. **L3: Autonomous Planner**: Decomposes a goal into a graph of sub-tasks.
5. **L4: Ambient Agent**: Runs in the background, intervenes only when necessary.

L4 became a product category in September 2026: OpenAI's dots (announced September 29) run always-on on their own cloud computer and do only read-only research when idle, Meta's Muse runs in a dedicated cloud VM, and Microsoft's Copilot Autopilot (private preview) gets its own identity, memory, and computer inside the customer tenant. The design questions at L4 shift from "can it finish the task" to tenancy, agent identity, approval policy, and billing.

---

## Core Components

### 1. The Reasoning Model (The Executive)
The CPU of the agent. It determines the "Path to Success."

### 2. Tools (The Limbs)
Interfaces (APIs, Browsers, DBs) that allow the agent to affect the world.
> [!Note]
> The **Model Context Protocol (MCP)** is now the industry standard for tool interoperability, with adoption from Anthropic, OpenAI, Google, Microsoft, and AWS. Governance moved to the Linux Foundation's Agentic AI Foundation in December 2025; the foundation has also hosted the A2A agent-to-agent protocol since August 2026.

### 3. Memory (The Experience)
- **Short-term**: Context window (KV Cache).
- **Long-term**: Vector DBs, persistent state (e.g., Mem0), or file-based memory (Letta's git-backed MemFS, Claude's memory tool).

---

## The Agent Lifecycle

1. **Intake**: Receive user goal.
2. **Decomposition**: Break goal into sub-steps.
3. **Execution**: Call tools and handle results.
4. **Reflection**: Evaluate if the observation got the agent closer to the goal.
5. **Completion**: Synthesize final proof for the user.

---

## Interview Questions

### Q: Why is a "Reasoning Model" (like Claude Opus 5.5 or GPT-6 Astra) better for agency than a standard LLM?

**Strong answer:**
Standard LLMs (System 1) predict the *very next token* based on pattern matching. When they encounter an error in a tool call, they often hallucinate a fix instead of admitting the failure. Reasoning Models use **Chain-of-Thought (CoT)** during inference, and the current generation thinks between tool calls, not only before the first one. For an agent, this means higher **Path Reliability**: the model is less likely to enter an infinite loop or retry the same failing action because it reasons about the failure before choosing the next step. It is not a substitute for harness guards. Reasoning models still loop, so the harness keeps repetition detectors and budgets regardless (see [Loop Engineering](12-loop-engineering.md)).

### Q: How do you prevent "Agentic Drift" in long-running tasks?

**Strong answer:**
Agentic Drift occurs when the sub-steps take the agent so far from the original goal that it loses context. The standard solution is **Goal Anchoring**: include the "Original Objective" as a pinned system message and use a **Secondary Observer Model** (a smaller, cheaper model) to score every agent action against the original objective. If the score drops below a threshold, the agent is forced to "re-plan" from the root.

---

## References
- Kahneman, D. "Thinking, Fast and Slow" (2011), the source of the System 1 / System 2 framing
- OpenAI. "Learning to Reason with LLMs" (2024)
- DeepSeek-AI. "DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning" (2025)

---

*Next: [Reasoning Loops: ReAct and Beyond](02-reasoning-loops-react-and-beyond.md)*
