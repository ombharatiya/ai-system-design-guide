# Prompt Optimization (DSPy)

Prompting has moved from the "Hand-tuning" era to the "Programmatic" era. **DSPy (Declarative Self-improving Language Programs)** is the de-facto standard for building LLM pipelines where prompts are optimized automatically by algorithms. The current release is DSPy 3.4.0 (September 25, 2026), and the 3.x line has shifted the emphasis from optimizing prompt text (MIPROv2) to optimizing with reflection and program structure (GEPA, and Flex in 3.3). The framework deep dive is in [DSPy](../09-frameworks-and-tools/05-dspy.md); this chapter covers the prompting ideas.

## Table of Contents

- [The DSPy Philosophy: Programming vs. Prompting](#the-dspy-philosophy-programming-vs-prompting)
- [Signatures & Modules](#signatures--modules)
- [Teleprompters (Optimizers)](#teleprompters-optimizers)
- [The "Prompt as Weight" Analogy](#the-prompt-as-weight-analogy)
- [Metric-Driven Optimization](#metric-driven-optimization)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The DSPy Philosophy: Programming vs. Prompting

In traditional prompting, changing a model (e.g., from GPT-6.1 Sol to Claude Sonnet 5.5 or an open-weight model like Qwen3.8-27B) requires re-writing all your prompts.
**DSPy separates Logic from Formatting.**

- **Logic**: Defined by **Modules** (e.g., ChainOfThought, ReAct).
- **Optimization**: The system automatically finds the best prompt and examples for a *specific* model to fulfill that logic.

---

## Signatures & Modules

Instead of writing a prompt, you define a **Signature**: what the input is and what the output should be.

```python
# Signature pattern
class MultiHopQA(dspy.Signature):
    """Answer questions that require multiple context retrievals."""
    context = dspy.InputField()
    question = dspy.InputField()
    answer = dspy.OutputField(desc="A concise 1-sentence answer")

# Logic is handled by a Module
qa_system = dspy.ChainOfThought(MultiHopQA)
```

By default, `ChainOfThought` prepends a plain-text `reasoning` output field that the model fills before the answer. On reasoning models that is duplicate work, and on the newest Claude models (Fable 5.1, Opus 5.5, Sonnet 5.5) a `reasoning` field in the output is a listed trigger for `reasoning_extraction` refusals. Compare it against `dspy.Predict` with native thinking at a fixed effort level, and let the metric pick.

---

## Teleprompters (Optimizers)

Teleprompters (the original name; DSPy's docs now call them **optimizers**) are algorithms that iterate on your program to improve accuracy.
1. **BootstrapFewShot**: Automatically finds high-quality examples for your prompt.
2. **MIPROv2**: A Bayesian optimizer that tries different instruction phrasings and demos and selects the combination that maximizes your score. Still the workhorse when you have a few hundred labeled examples and a fixed pipeline.
3. **GEPA** (reflective prompt evolution): an LLM reads execution traces and metric feedback, proposes targeted instruction edits, and keeps a Pareto frontier of candidates. Its paper reports matching or beating RL-based tuning with far fewer rollouts, which matters when each rollout is a long agent run.
4. **Flex** (DSPy 3.3.0, August 3, 2026): uses GEPA to optimize **program structure** (which modules exist and how they connect), not only the text of each prompt.
5. **ReAnchor** (DSPy 3.4.0): calibrates decision thresholds, score cuts, and choice weights against the program metric, for programs whose outputs are typed decisions (yes/no, a choice, a score) rather than free text.

**Why it matters**: You no longer guess if "Be helpful" or "Think carefully" is better. The optimizer proves it with data.

---

## The "Prompt as Weight" Analogy

In DSPy, your prompt is like a weight in a neural network. You don't "hardcode" weights; you train them.
- If you change your model, you just **Re-compile** (re-train) your program. The optimizer will find new few-shot examples that the new model understands better.
- Treat the **effort level as part of the model**. Defaults differ (Claude Opus 5.5 defaults to `medium`, Opus 5 to `high`) and sampling parameters are disappearing from the newest APIs, so a program compiled at one effort level is not validated at another.
- Pin the DSPy version as well. 3.4.0 removed the experimental 3.3 typed LM API, and OpenAI-style `messages=` calls now emit deprecation warnings ahead of removal in 3.5.

---

## Metric-Driven Optimization

Optimization requires a **Metric** (a function that returns a score).
- **Exact Match**: `prediction.answer == target.answer`
- **LLM-as-Judge**: Use a stronger model (Claude Opus 5.5, GPT-6.1 Sol) to grade the output of a smaller one (Claude Haiku 5.5, GPT-6 Luna, Gemini 3.8 Flash). Calibrate the judge against human labels first: an optimizer will exploit any blind spot in the metric.
- **Judge cost**: an optimizer run scores hundreds to thousands of candidate outputs, so the judge often dominates the bill. For binary checklist criteria, typed-decision models are a cheaper option: one comparison (arXiv 2609.29769) found LLM rubric judges cost 16 to 325x as much as TypeSafe's Jev (early access), with accuracy differing significantly in at most 8 of 27 paired comparisons. They also failed together: on Jev's most confident errors, about 96% of LLM verdicts repeated the same wrong answer, so swapping judges does not replace the human calibration set.
- **Textual feedback**: GEPA works best when the metric returns *why* an output failed, not just a number.

---

## Interview Questions

### Q: How does DSPy solve the "fragility" of prompt engineering?

**Strong answer:**
DSPy moves the complexity of "formatting" and "grounding" away from the human and into the compiler. When we hand-write prompts, we are effectively "hard-coding" behavior that is specific to one model at one specific time (point-in-time tuning). If that model is updated or swapped, the prompt breaks. DSPy treats the prompt as a learnable parameter. By defining a clear **Signature** and a **Metric**, we allow the system to "search" for the most effective prompt over hundreds to thousands of evaluated rollouts, making the final system much more resilient to model changes. The fragility does not vanish; it moves into the metric, which is now the thing to get right.

### Q: What is a "Teleprompter" in the context of DSPy?

**Strong answer:**
A Teleprompter (now called an optimizer) is a programmatic optimizer. Its job is to take a DSPy program (which might be a complex chain of modules) and a small set of training examples, and then "compile" them into an optimized version. It does this by generating potential "thinking patterns" and examples, testing them against a metric, and selecting the most effective ones. In short, a Teleprompter is the "Gradient Descent" of the prompt engineering world. The 2026 extension is that the newer optimizers search over the program's structure as well as its prompts: Flex can add a verification step or change module boundaries, not just reword instructions.

---

## References
- Khattab et al. "DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines" (2023/2024)
- Agrawal et al. "GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning" (2025)
- [DSPy releases (3.3.0, 3.4.0)](https://github.com/stanfordnlp/dspy/releases)
- [Rao and Callison-Burch. "JEV vs. LLMs as Rubric Judges: Cheaper, Faster, and Wrong in the Same Places" (arXiv 2609.29769)](https://arxiv.org/abs/2609.29769)
- Stanford NLP. "DSPy Documentation and Tutorials" (2025)
- [Anthropic. "Refusals and fallback": keep reasoning in thinking blocks](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#keep-reasoning-in-thinking-blocks)

---

*Next: [Prompt Injection and Defense](08-prompt-injection-defense.md)*
