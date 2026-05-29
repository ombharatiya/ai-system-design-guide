# Prompt Optimization (DSPy)

Prompting has moved from the "Hand-tuning" era to the "Programmatic" era. **DSPy (Declarative Self-improving Language Programs)** is the de-facto standard for building robust LLM pipelines where prompts are optimized automatically by algorithms. The 3.x line (DSPy 3.1.3 shipped Feb 5, 2026, with point releases through May 2026) introduces tighter integration with reasoning-native models and a cleaner async runtime.

## Table of Contents

- [The DSPy Philosophy: Programming vs. Prompting](#philosophy)
- [Signatures & Modules](#signatures-modules)
- [Teleprompters (Optimizers)](#optimizers)
- [The "Prompt as Weight" Analogy](#prompt-as-weight)
- [Metric-Driven Optimization](#metrics)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The DSPy Philosophy: Programming vs. Prompting

In traditional prompting, changing a model (e.g., from GPT-5.5 to Claude Sonnet 4.6 or Llama 4) requires re-writing all your prompts.
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

---

## Teleprompters (Optimizers)

Teleprompters are algorithms that iterate on your program to improve accuracy.
1. **BootstrapFewShot**: Automatically finds high-quality examples for your prompt.
2. **MIPROv2**: A Bayesian optimizer that tries different instruction phrasings and selects the one that maximizes your score. Still the flagship optimizer in the 3.x line.

**Why it matters**: You no longer guess if "Be helpful" or "Think carefully" is better. The optimizer proves it with data.

---

## The "Prompt as Weight" Analogy

In DSPy, your prompt is like a weight in a neural network. You don't "hardcode" weights; you train them.
- If you change your model, you just **Re-compile** (re-train) your program. The optimizer will find new few-shot examples that the new model understands better.

---

## Metric-Driven Optimization

Optimization requires a **Metric** (a function that returns a score).
- **Exact Match**: `prediction.answer == target.answer`
- **LLM-as-Judge**: Use a larger model (Claude Opus 4.7, GPT-5.5 reasoning) to grade the output of a smaller model (Llama 4 8B, Claude Haiku 4.5).

---

## Interview Questions

### Q: How does DSPy solve the "fragility" of prompt engineering?

**Strong answer:**
DSPy moves the complexity of "formatting" and "grounding" away from the human and into the compiler. When we hand-write prompts, we are effectively "hard-coding" behavior that is specific to one model at one specific time (point-in-time tuning). If that model is updated or swapped, the prompt breaks. DSPy treats the prompt as a learnable parameter. By defining a clear **Signature** and a **Metric**, we allow the system to "search" for the most effective prompt through thousands of simulated iterations, making the final system much more resilient to model changes.

### Q: What is a "Teleprompter" in the context of DSPy?

**Strong answer:**
A Teleprompter is a programmatic optimizer. Its job is to take a DSPy program (which might be a complex chain of modules) and a small set of training examples, and then "compile" them into an optimized version. It does this by generating potential "thinking patterns" and examples, testing them against a metric, and selecting the most effective ones. In short, a Teleprompter is the "Gradient Descent" of the prompt engineering world.

---

## References
- Khattab et al. "DSPy: Compiling Declarative Language Models" (2023/2024)
- Stanford NLP. "DSPy Documentation and Tutorials" (2025)

---

*Next: [Prompt Injection and Defense](08-prompt-injection-defense.md)*
