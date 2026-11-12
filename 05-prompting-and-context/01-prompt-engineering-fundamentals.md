# Prompt Engineering Fundamentals

Prompt engineering is the design of inputs to steer LLM behavior. It has evolved from "trial and error" to a disciplined architectural practice, with frameworks like DSPy treating it as a compilation problem rather than a writing exercise.

## Table of Contents

- [The Core Philosophy: Intent + Constraint](#the-core-philosophy-intent--constraint)
- [The Instruction Hierarchy](#the-instruction-hierarchy)
- [Role Prompting](#role-prompting)
- [Instruction Clarity and Delimiters](#instruction-clarity-and-delimiters)
- [Zero-Shot vs. Few-Shot Efficiency](#zero-shot-vs-few-shot-efficiency)
- [Determinism Without Temperature](#determinism-without-temperature)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Core Philosophy: Intent + Constraint

Effective prompting is about maximizing **Intent Disclosure** while minimizing **Output Variance**.

1. **Intent**: Precisely what the model should do.
2. **Constraint**: Exactly what the model should *avoid* (Safety, Tone, Format).

**Principle**: "Prompting is Programming in Natural Language." Treat your prompts like code (Version control, Unit tests).

---

## The Instruction Hierarchy

Production systems use a tiered message structure, and models are trained to resolve conflicts in favor of the higher tier:

| Role | Responsibility | Nuance |
|------|----------------|--------|
| **System / Developer** | Operator rules, persona, safety, output contract. | Highest priority the application controls. OpenAI's reasoning models call this the `developer` message. The newest Claude models also accept `system` messages mid-conversation, so operators can add instructions without rewriting history. |
| **User** | The specific, dynamic query. | Trusted for intent, not for authority over operator rules. A document the user pastes is still untrusted data. |
| **Assistant** | History of previous turns. | Source of recency and self-consistency bias. On the newest Claude models, earlier turns must stay unedited or their thinking blocks are invalidated (see [Context Engineering](05-context-engineering.md#append-only-context-thinking-is-bound-to-the-conversation)). |
| **Tool / Retrieved content** | Tool results, RAG chunks, web pages. | Lowest trust: data, never instructions. The main channel for indirect prompt injection. |

The hierarchy is a **trained preference, not a security boundary**. OpenAI formalized it as instruction-hierarchy training (Wallace et al., 2024), and every frontier lab now trains for it, but injections still succeed at measurable rates. See [Prompt Injection and Defense](08-prompt-injection-defense.md).

---

## Role Prompting

Assigning a persona is no longer just "You are a teacher." It is a **Capabilities Anchor**.

- **Weak**: "You are a coder."
- **Strong**: "You are a Staff Software Engineer at a Tier-1 tech company specializing in high-concurrency Rust systems. You prioritize memory safety and zero-cost abstractions."

**Why it works**: the strong version encodes **priorities and constraints** (memory safety, zero-cost abstractions), which change what the model optimizes for. The job title itself does much less than people assume: a controlled study of 162 personas across several model families found that adding a persona to the system prompt did not reliably improve accuracy on factual questions (Zheng et al., 2024). Use roles for tone, vocabulary, and priorities; put correctness in explicit constraints and checks.

---

## Instruction Clarity and Delimiters

Current frontier models process 1M-token contexts. Delimiters help the model distinguish between instructions and data.

```markdown
# Instructions
Analyze the following text for PII.

# Data to Analyze
--- START OF USER DATA ---
$USER_INPUT_HERE
--- END OF USER DATA ---

# Output Schema
{ "pii_found": boolean, "types": [] }
```

**Delimiters to use**: XML tags (`<context>`, `</context>`), Markdown headers (`#`), or triple quotes (`"""`). For the output schema, prefer native structured outputs over a schema written in prose (see [Structured Generation](06-structured-generation.md)).

---

## Zero-Shot vs. Few-Shot Efficiency

| Aspect | Zero-Shot | Few-Shot |
|--------|-----------|----------|
| **Latency** | Lowest (Short prompt) | Higher (Example tokens) |
| **Accuracy**| Variable | High (Format stability) |
| **Use Case**| Simple chat, Summarization | Specific formatting, Subtle logic |

**Strategy**: For reasoning models (Claude Opus 5.5, GPT-6.1 Sol, Gemini 3.8 Flash at higher thinking levels), start **zero-shot with a clear goal, constraints, and success criteria**, and skip "think step by step" or prescriptive reasoning scripts: the model already reasons internally. Anthropic's current guidance is that a general instruction ("think thoroughly") often beats a hand-written step-by-step plan, and that the emphatic language older prompts needed ("CRITICAL: you MUST...") now causes overtriggering and should be dialed back. Add examples when you need a specific output format or a subtle labeling convention. For small models (8B and below), use **few-shot** to ground them.

---

## Determinism Without Temperature

The classic advice "set temperature to 0 for consistency" no longer works on the newest APIs. Sampling parameters are being removed in favor of reasoning effort as the control surface:

| Vendor | Status of `temperature` / `top_p` |
|--------|-----------------------------------|
| **Anthropic** | From API version `2026-10-21` (October 21, 2026), `temperature`, `top_p`, and `top_k` return a 400 at any value, even the default, on every model, and all eight new SDK and CLI majors released that day send that version by default: Python 2.0.0, TypeScript 1.0.0, C# 13.0.0, Go 2.0.0, Java 3.0.0, PHP 1.0.0, Ruby 2.0.0, and the ant CLI 2.0.0 (all now Apache 2.0 instead of MIT), so a routine dependency bump in any language can start returning 400s. Earlier API versions reject non-default values on Claude 4.7 and later models (including Sonnet 5, Sonnet 5.5, Haiku 5.5 and Fable 5.1). The Python SDK 1.0 (August 20, 2026) had already removed the three parameters from the Messages method signatures, so passing them raises a `TypeError`. |
| **OpenAI** | GPT-6 Astra (September 3, 2026) does not support custom `temperature`, `top_p`, or `logprobs`. |
| **Google** | `temperature`, `top_p`, and `top_k` deprecated in the Gemini API on July 21, 2026; Gemini 3.8 Flash uses `thinking_level` instead of `thinking_budget`. |

How to get consistency now:

1. **Pin the model snapshot and the effort level.** Effort changes behavior as much as temperature did, and defaults differ (Claude Opus 5.5 defaults to `medium`, Opus 5 to `high`).
2. **Constrain the output** with structured outputs or an enum, so variance cannot reach the format.
3. **Cache by input hash** for identical requests, and log request and response pairs for replay.
4. **Evaluate distributions, not single samples.** Run each eval case several times and track the pass rate.
5. **Re-run evals on vendor changelog events**, not only on model-ID changes: OpenAI fixed an image bug in GPT-6 Sol and Luna on September 25, 2026 under the same model ID.

---

## Interview Questions

### Q: Why do system prompts carry more weight than user prompts in modern LLMs?

**Strong answer:**
Because the model is trained to obey an **instruction hierarchy**: during post-training, labs construct conflicts between system, developer, user, and tool messages and reward the model for following the higher-privilege one (OpenAI published this as instruction-hierarchy training in 2024, and its Model Spec calls it the chain of command). So if a user asks for something the system prompt forbids, a well-aligned model sides with the operator. The important caveat is that this is a learned preference, not an architectural guarantee: vendor-reported indirect-injection success rates on current frontier models range from about 1% (Claude Opus 5.5 on Gray Swan's IPI benchmark at 15 attempts per scenario) to 8.5% (GPT-6 Astra on the IPI Arena set). Critical rules therefore also need enforcement outside the model: tool permissions, output validation, and egress controls.

### Q: What is the "Step-by-Step" prompt optimization?

**Strong answer:**
In 2022, "Think step by step" was a magic phrase to trigger Chain-of-Thought (CoT). For non-reasoning models, the better version is **Programmatic CoT**: explicit milestones such as "1. Identify the core problem. 2. List the constraints. 3. Propose 3 solutions. 4. Select the best one and justify." That makes the output structured and auditable. For reasoning models, I do not script the thinking. I state the goal, the constraints, and what a correct answer must satisfy, and I control depth with the effort setting. If I need auditable steps, I ask for a short explanation or the evidence behind the answer, not a transcript of the thinking. On the newest Claude models that distinction is enforced: a prompt that asks the model to write its reasoning into the output (a `<thinking>` section, a `reasoning` field) can be declined under the `reasoning_extraction` refusal category, and the supported way to inspect reasoning is summarized thinking blocks.

### Q: The API rejects `temperature` now. How do you make outputs reproducible enough to test?

**Strong answer:**
I stop treating reproducibility as a sampling setting and treat it as a system property. First, pin everything that changes behavior: the dated model ID, the effort level, the system prompt version, and the tool definitions. Second, remove variance where it does not belong: structured outputs for format, enums for labels, and an input-hash cache for repeated requests. Third, test statistically: each eval case runs several times, and the gate is a pass rate with a confidence interval, not a single golden string. Finally, I re-run the suite whenever the vendor changelog says something changed, because providers sometimes fix behavior under a stable model ID.

---

## References
- OpenAI. "Prompt Engineering Guide" and "Reasoning Best Practices" (2024-2026)
- [Anthropic. "Prompting best practices" (2026)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Anthropic. "Refusals and fallback": keep reasoning in thinking blocks](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#keep-reasoning-in-thinking-blocks)
- Wallace et al. "The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions" (2024)
- Zheng et al. "When 'A Helpful Assistant' Is Not Really Helpful: Personas in System Prompts Do Not Improve Performances of Large Language Models" (EMNLP Findings 2024)
- Microsoft Research. "The Power of Prompting" (Medprompt, 2023)
- [Claude API release notes (SDK 1.0, sampling parameters)](https://platform.claude.com/docs/en/release-notes/overview)

---

*Next: [Few-Shot and In-Context Learning](02-few-shot-and-icl.md)*
