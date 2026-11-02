# Few-Shot and In-Context Learning (ICL)

In-Context Learning (ICL) is the ability of an LLM to learn a new task simply by seeing examples in the prompt, without any weight updates. Maximizing ICL efficiency is a key lever for prompt stability.

## Table of Contents

- [The Anatomy of a Few-Shot Example](#the-anatomy-of-a-few-shot-example)
- [How many examples?](#how-many-examples)
- [Many-Shot ICL vs. Fine-Tuning](#many-shot-icl-vs-fine-tuning)
- [Dynamic Example Selection](#dynamic-example-selection)
- [The Importance of Labeling Nuance](#the-importance-of-labeling-nuance)
- [Advanced ICL: Analogy and "Few-Shot CoT"](#advanced-icl-analogy-and-few-shot-cot)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Anatomy of a Few-Shot Example

A high-quality example consists of three parts:
1. **Input**: A realistic sample of potential user data.
2. **Reasoning (Optional)**: A short explanation of *why* the output is what it is.
3. **Output**: The "Gold Standard" result.

```markdown
User: "The weather is okay, but the flight was late."
Reasoning: The user is neutral about the weather but negative about the service.
Sentiment: Mixed
```

---

## How many examples?

| Model Size | Sweet Spot | Scaling Behavior |
|------------|------------|------------------|
| **Small (8B)** | 5 - 10 | Gains continue until ~20 examples. |
| **Medium (70B-class dense, or ~30B-active MoE)**| 3 - 5 | Plateaus early; more examples increase latency. |
| **Frontier (closed, or 1T-class MoE)**| 0 - 2 | Highly capable; "Instruction Following" usually suffices. Examples mainly pin the output format. |

**Rule of thumb**: If you need more than 20 examples to get a stable output on a short-context budget, your task is likely too complex for the model. The classic next step was **fine-tuning**; with 1M-token windows and cheap cache reads, **many-shot ICL** is now a real alternative.

---

## Many-Shot ICL vs. Fine-Tuning

Many-shot ICL puts hundreds or thousands of examples in the prompt. Google DeepMind's study (Agarwal et al., 2024) found gains continuing well past the few-shot regime on many tasks, and in some cases many-shot prompts overrode pretraining biases that few-shot prompts could not.

The economics changed in 2026 because the example block is a **stable prefix** and cache reads are cheap:

| Setup (per request, list prices) | Input cost of a 50K-token example block |
|----------------------------------|------------------------------------------|
| Claude Sonnet 5.5, uncached ($2/1M) | $0.10 |
| Claude Opus 5.5, cache read ($0.20/1M) | $0.01 |
| Claude Sonnet 5.5 or GPT-6.1 Sol, cache read ($0.10/1M) | $0.005 |

At those prices, many-shot ICL beats fine-tuning on iteration speed (change an example, not a training run) and portability (it moves to the next model unchanged). Fine-tuning still wins on per-request latency, on tasks where the examples would not fit, and when you need the behavior without shipping the examples. Platform risk matters too: OpenAI is winding down self-serve fine-tuning, and active customers can no longer create new fine-tuning jobs from January 6, 2027. See [Fine-Tuning Strategies](../03-training-and-adaptation/02-fine-tuning-strategies.md).

---

## Dynamic Example Selection

In production RAG or Classification, don't use the same static examples for every user.
**The Dynamic Pattern:**
1. User provides a query.
2. Search a "Vector DB of Gold Examples" for the 3 most **semantically similar** cases.
3. Inject those 3 specific cases into the prompt.

**Result**: Drastically higher accuracy because the model sees "local" patterns relevant to the current user.

**Tradeoff**: dynamic examples change on every request, so they cannot sit in the cached prefix. Put them after the last cache breakpoint, or use a static many-shot block when cache savings outweigh the accuracy gain from per-query selection.

---

## The Importance of Labeling Nuance

Frontier models are sensitive to **Distribution Bias** in examples.
- If you provide 5 "Positive" examples and 1 "Negative," the model will bias toward "Positive."
- **Fix**: Always use **Label Balancing**. Ensure your few-shot examples roughly mirror the expected output distribution or are perfectly balanced (1:1).

---

## Advanced ICL: Analogy and "Few-Shot CoT"

**Analogy Prompting**: Instead of saying "Do X," provide an analogy.
"Translate this code like a translator would move a poem from French to English: preserve the soul (logic) but change the syntax."

**Few-Shot CoT**: Providing 2 examples where the reasoning is explicit. This "primes" the model's attention to focus on logic rather than just mimicking the output string. On reasoning models this matters less, since they reason before answering anyway; use worked examples there to show the *decision criteria* you want applied, not a thinking script. Keep the reasoning in the examples as a short rationale: on the newest Claude models, asking for the model's full reasoning in the output can be refused (see [Chain-of-Thought](03-chain-of-thought.md#zero-shot-vs-programmatic-cot)).

---

## Interview Questions

### Q: Why not just provide all 50 examples we have in the prompt?

**Strong answer:**
There are three primary reasons:
1. **Context Window Latency**: Every example adds tokens, increasing the "Prefill" time and the cost per request.
2. **Attention Dilution**: Even with 1M-token windows, models can "lose" specific constraints if buried under too much irrelevant data (the "lost-in-the-middle" effect and context rot).
3. **Overfitting**: Providing too many narrow examples can cause the model to mimic the *format* of the examples too strictly, losing its general capability to handle edge cases outside that set.

The counterpoint I would raise: if the 50 examples are stable, they become a cached prefix, and at $0.10 to $0.20 per 1M cached tokens the cost argument mostly disappears. Then the question becomes empirical: run the eval with 3 dynamic examples, with all 50, and with many-shot, and keep the cheapest configuration that meets the quality bar.

### Q: What is "Label Bias" in In-Context Learning?

**Strong answer:**
Label bias occurs when the model predicts a specific label more frequently simply because it appeared more often in the few-shot examples or because it appeared at the end of the list (majority-label and recency bias, Zhao et al., 2021). The standard mitigations are:
1. Shuffling the order of examples for different requests. Per-request shuffling breaks prefix caching, so for a cached example block, pick a balanced order once per prompt version and verify it with permutation testing instead.
2. Ensuring an equal number of positive/negative/neutral samples.
3. Using "Permutation Testing" during prompt development to ensure the model responds to the content, not the order.
4. Calibrating with a content-free input ("N/A") to measure and subtract the prior.

---

## References
- Brown et al. "Language Models are Few-Shot Learners" (2020)
- Zhao et al. "Calibrate Before Use: Improving Few-Shot Performance of Language Models" (2021)
- Min et al. "Rethinking the Role of Demonstrations: What Makes In-Context Learning Work?" (2022)
- Agarwal et al. "Many-Shot In-Context Learning" (NeurIPS 2024)
- [OpenAI deprecations (fine-tuning wind-down)](https://developers.openai.com/api/docs/deprecations)

---

*Next: [Chain-of-Thought](03-chain-of-thought.md)*
