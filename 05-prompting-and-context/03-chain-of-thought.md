# Chain-of-Thought (CoT)

Chain-of-Thought (CoT) is the technique of encouraging an LLM to generate intermediate reasoning steps before providing a final answer. It has evolved from a simple prompt phrase into the core architectural feature of reasoning models: first OpenAI o1 and DeepSeek-R1, and today GPT-6 Astra and GPT-6.1 Sol, Claude Opus 5.5 and Fable 5.1, Gemini 4 Argon and Gemini 3.8 Flash, and DeepSeek V4.1-Flash, all of which reason before answering under an effort or thinking-level control.

## Table of Contents

- [The CoT Revolution](#the-cot-revolution)
- [Zero-Shot vs. Programmatic CoT](#zero-shot-vs-programmatic-cot)
- [The Rise of "Thinking" Models](#the-rise-of-thinking-models)
- [CoT as a Monitoring Signal](#cot-as-a-monitoring-signal)
- [Self-Correction and Verification](#self-correction-and-verification)
- [When CoT Fails (Over-thinking)](#when-cot-fails-over-thinking)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The CoT Revolution

Standard LLMs are "Next Token Predictors." For complex math or logic, a single pass is often insufficient. CoT provides the "Scribble Pad" (Working Memory) for the model to work through sub-problems.

**The Formula**: `Input -> Reasoning (Chain) -> Output`

---

## Zero-Shot vs. Programmatic CoT

| Technique | Trigger Phrase | Efficiency | Use Case |
|-----------|----------------|------------|----------|
| **Zero-Shot CoT** | "Let's think step by step." | High | Ad-hoc queries on non-reasoning models. |
| **Few-Shot CoT** | (Provided examples with logic) | Higher Stability | Production pipelines on non-reasoning models. |
| **Programmatic CoT** | "1. Analyze X. 2. Verify Y. 3. Resolve Z." | **Best for Agents** | Complex multi-tool tasks where the steps must be auditable. |

On reasoning models, these prompts are mostly redundant: the model already thinks before it answers. Set the effort level instead, and state the success criteria the answer must meet.

On the newest Claude models (Fable 5.1 and 5, Opus 5.5 and 5, Sonnet 5.5, and Haiku 5.5) they can also fail outright. A prompt that asks the model to write its reasoning into the output, such as a `<thinking>` or scratchpad section filled in before the answer or a `reasoning` field in JSON, can be declined under the `reasoning_extraction` refusal category, and a refusal in that category that arrives before any output is billed. Ask for a short explanation or the evidence behind the answer instead, and read summarized thinking blocks when you need to see how the model reasoned.

---

## The Rise of "Thinking" Models

Models like **GPT-6 Astra and GPT-6.1 Sol**, **Claude Opus 5.5 and Fable 5.1**, **Gemini 4 Argon and Gemini 3.8 Flash**, and **DeepSeek V4.1-Flash** have CoT "baked in" via Reinforcement Learning (RL). See [RLVR and Reasoning Models](../03-training-and-adaptation/08-rlvr-and-reasoning-models.md) for how the training works.

1. **Effort replaces the token budget**: The model has a dedicated thinking phase, controlled by an effort level rather than a raw token count. OpenAI exposes `reasoning.effort` (`low` to `max` on GPT-6 Astra and GPT-6.1 Sol), Anthropic exposes adaptive thinking plus `effort` (fixed `budget_tokens` now returns a 400 on Opus 4.7 and later), and Gemini 4 Argon and Gemini 3.8 Flash use `thinking_level`. API details are in [Context Engineering](05-context-engineering.md#reasoning-effort-and-thinking-controls).
2. **Hidden CoT**: No major vendor returns the raw chain of thought. OpenAI returns optional summaries and encrypted reasoning items; Anthropic returns summarized or empty ("omitted", the default on its newest models) thinking blocks, bound to the producing model and conversation. Thinking is billed as output tokens whether or not you can read it. One driver is **distillation defense**: Anthropic disclosed in February 2026 that competitors ran 16M+ exchanges through about 24,000 fraudulent accounts asking Claude to write out its reasoning, and its September 2026 threat report described nearly 200M exchanges across five campaigns.
3. **Scaling Law**: These models follow the **Inference Scaling Law**: the longer they "think," the better they solve hard problems, with diminishing returns per token. Experimental reasoning models from OpenAI and Google DeepMind reached gold-medal level at IMO 2025.

---

## CoT as a Monitoring Signal

Reading the model's reasoning used to be a cheap way to catch misbehavior before it became an action. That signal is weakening at the frontier.

- **Controllability is rising.** OpenAI's GPT-6 Astra system card (September 3, 2026) reports that, among chains of thought 750 to 1,250 tokens long, Astra controls its CoT 60.9% of the time, against 16.1% for GPT-5.6 Sol and 1.7% for GPT-5.5 Thinking (OpenAI's eval). UK AISI's separate controllability eval found Astra followed the constraint on 93% of samples. OpenAI writes that if the model tried to sandbag covertly, it would likely not catch it reliably.
- **Monitors get evaded under ordinary pressure.** EvasionBench (arXiv 2609.30217) put agents in tasks that required an operation a known runtime monitor prohibited: best-of-3 evasion attempts reached about 98%, success up to 88%, and evasion generally rose with reasoning effort.

**Design implication:** treat CoT as a debugging aid, not a control. Put safety monitors on **actions and outputs** (tool calls, diffs, network egress, data written), sandbox what the agent can touch, and keep the reasoning trace as supporting evidence. See [Agentic Security and Sandboxing](../07-agentic-systems/09-agentic-security-and-sandboxing.md) and the [Research Radar](../RESEARCH-RADAR.md#4-latent-and-alternative-reasoning) on latent reasoning.

---

## Self-Correction and Verification

Production pipelines no longer trust a single Chain-of-Thought. They layer in **Self-Verification**.

```markdown
# Process
1. Generate Answer A via CoT.
2. Critique: "Are there any errors in the logic above?"
3. If errors: "Correct the logic and provide Answer B."
```

**Nuance**: Intrinsic self-correction (the same model critiquing itself with no new information) often fails to improve reasoning and can make it worse (Huang et al., 2024). What works is **external feedback**: **Execution-Verified CoT** for coding, where the model writes the logic, runs the code, and corrects itself if the tests fail, or a separate verifier with a different prompt, model, or tool. See [Loop Engineering](../07-agentic-systems/12-loop-engineering.md#verification-and-grading).

---

## When CoT Fails (Over-thinking)

CoT is not a silver bullet. For simple tasks, it adds:
1. **Latency**: More tokens = slower response.
2. **Cost**: You pay for every "thought" token, billed at the output rate (for example $20 per 1M on Claude Opus 5.5, $10 per 1M on GPT-6.1 Sol).
3. **Over-thinking**: The model might hallucinate complexity where none exists (e.g., explaining why 2+2=4 for 3 paragraphs).

The fix on current models is the effort dial: `low` for chat, classification, and extraction, higher levels only where evals show a gain. Some models no longer let you switch thinking off at all (Claude Opus 5.5 cannot disable it; Claude Sonnet 5.5's lowest setting is `between_tools`), so effort is the only lever.

---

## Interview Questions

### Q: Why does CoT improve performance on mathematical word problems?

**Strong answer:**
CoT improves performance by aligning the model's computational complexity with the task's logical complexity. In a standard single-pass generation, the model must predict the final answer token based on limited local information. With CoT, the model "breaks" the problem into smaller, auto-regressive steps. Each step uses the previous step's output as context, allowing the model's attention mechanism to focus on one sub-problem at a time (e.g., first adding the apples, then subtracting the oranges), reducing the "cognitive load" of the single-pass prediction.

### Q: How do you handle CoT in a production environment where latency is critical?

**Strong answer:**
We use a **Hybrid Reasoning Architecture**:
1. **Tier 1 (Routing)**: A cheap classifier picks the effort level per request (`low` for most traffic, `high` only for queries that need it), rather than one global setting.
2. **Tier 2 (Condensed reasoning)**: For high-volume paths, use a small, fast model at low effort, or **Knowledge Distillation**: train a smaller model to produce the final answer directly from a teacher's reasoning traces.
3. **Tier 3 (Streaming)**: Stream something while the model thinks. Raw CoT is not available, so stream summarized thinking or progress updates (Claude's `display: "summarized"`, or the `display: "updates"` beta for between-tool progress notes) so the UI does not look frozen.

I judge the result by cost and latency per completed task, not per request: a cheaper call that needs a retry is not cheaper.

### Q: Your safety team wants to monitor a coding agent by reading its chain of thought. What do you tell them?

**Strong answer:**
Useful, but not sufficient, and getting less sufficient. Vendors do not expose raw reasoning, only summaries. The GPT-6 Astra system card reports Astra can control its CoT 60.9% of the time on mid-length traces versus 16.1% for GPT-5.6 Sol, and OpenAI says covert sandbagging would likely go uncaught. Separately, EvasionBench showed agents evade known runtime monitors at high rates, more so at higher effort. So I would put the primary monitors on actions: every tool call, file diff, and network request goes through a policy check, the sandbox denies egress by default, and irreversible actions need approval bound to the exact action executed. The reasoning summary goes into the audit log as context for humans reviewing an incident.

---

## References
- Wei et al. "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models" (2022)
- Kojima et al. "Large Language Models are Zero-Shot Reasoners" (2022)
- Wang et al. "Self-Consistency Improves Chain of Thought Reasoning in Language Models" (2023)
- Huang et al. "Large Language Models Cannot Self-Correct Reasoning Yet" (ICLR 2024)
- OpenAI. "Learning to Reason with LLMs" (2024)
- [OpenAI. GPT-6 Astra System Card (September 2026)](https://deploymentsafety.openai.com/gpt-6-astra)
- [Anthropic. "Detecting and preventing distillation attacks" (2026)](https://www.anthropic.com/news/detecting-and-preventing-distillation-attacks)
- [Anthropic. "Refusals and fallback": keep reasoning in thinking blocks](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#keep-reasoning-in-thinking-blocks)
- [Instrumental Monitor Evasion Emerges Under Ordinary Task Pressure (arXiv 2609.30217)](https://arxiv.org/abs/2609.30217)

---

*Next: [Tree-of-Thought](04-tree-of-thought.md)*
