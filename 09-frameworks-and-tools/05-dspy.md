# DSPy: Programming Language Models

**DSPy** has become the industry reference for treating prompts as optimizable parameters. It represents a paradigm shift from "Prompt Engineering" (trial and error) to **Prompt Compilation** (automated optimization). Published results show large gains over hand-tuned prompts on some tasks, but the lift depends entirely on your metric and data, so measure it on your own eval set before promising a number.

The 2026 story is a second shift, from optimizing **prompts** to optimizing **program structure**: GEPA (reflective prompt evolution, DSPy 3.0) made optimization trace-driven, and Flex (DSPy 3.3.0, Aug 3, 2026) uses it to search over how modules are composed, not only over instruction text. Current release: DSPy 3.4.1 (Oct 12, 2026).

## Table of Contents

- [The Programming Paradigm](#the-programming-paradigm)
- [Signatures: Describing the Task](#signatures-describing-the-task)
- [Optimizers: MIPROv2, GEPA, and Flex](#optimizers-miprov2-gepa-and-flex)
- [Output Constraints: Refine and BestOfN](#output-constraints-refine-and-bestofn)
- [Managing Model Drift](#managing-model-drift)
- [DSPy 3.3 and 3.4 Changes](#dspy-33-and-34-changes)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Programming Paradigm

DSPy treats an LLM application like a **Neural Network**.
- **The Module**: A reusable block of logic (e.g., `ChainOfThought`).
- **The Signature**: A declarative specification of what the module does (Input -> Output).
- **The Optimizer**: A process that finds the best "Weights" (Prompts) for the module based on a metric.

---

## Signatures: Describing the Task

Instead of writing a 100-line prompt, you write a **Signature**:
```python
class ResearchAssistant(dspy.Signature):
    """Answer the question by synthesizing the provided web context."""
    context = dspy.InputField(desc="Scraped web content")
    question = dspy.InputField()
    answer = dspy.OutputField(desc="A technical summary with citations")
```
**Winning Nuance**: Signatures are **Model-Agnostic**. You can compile them for Claude Opus 5.5, Claude Sonnet 5.5, GPT-6.1 Sol, Gemini 3.8 Flash, or an open-weight model such as Qwen3.8-27B without changing a single line of program code. What changes per model is the compiled artifact (instructions and demos), which is why each model tier gets its own compile run.

---

## Optimizers: MIPROv2, GEPA, and Flex

**MIPROv2 (Multi-stage Instruction PRoposal Optimizer)** is the long-standing workhorse.
1. **Instruction Proposal**: An "Assistant Model" proposes 10-20 different ways to write the system prompt for the task.
2. **Bayesian Optimization**: DSPy runs the proposed prompts against a small training set and scores them using a metric.
3. **Selection**: It picks the prompt that maximizes your metric (e.g., Factuality score).

**GEPA** (reflective prompt evolution, `dspy.GEPA`, added in DSPy 3.0) replaces score-only search with reflection: an LLM reads execution traces and metric feedback, proposes targeted instruction edits, and keeps a Pareto frontier of candidates that win on different examples. It can use textual feedback, not just a scalar score, and its authors report beating both RL fine-tuning (GRPO) and MIPROv2 on their tasks with far fewer rollouts than RL needs.

**Flex** (DSPy 3.3.0, Aug 3, 2026) uses GEPA to optimize **program structure**: which modules exist and how they connect, not only what each prompt says. **ReAnchor** (3.4.0, Sep 25, 2026) calibrates decision thresholds, score cuts and choice weights against a program metric.

| Optimizer | Searches over | Reach for it when |
|---|---|---|
| MIPROv2 | Instructions and few-shot demos | A fixed pipeline needs better prompts and you have a few hundred labeled examples |
| GEPA | Instructions, guided by trace reflection and textual feedback | Rollouts are expensive, or your metric can explain why an output failed |
| Flex | Program structure, via GEPA | You suspect the decomposition itself is wrong (missing verify step, wrong module boundaries) |

---

## Output Constraints: Refine and BestOfN

Older tutorials teach `dspy.Assert` and `dspy.Suggest`. They were deprecated in DSPy 2.6 and are gone in 3.x; code that uses them fails on a current install. The replacements take a **reward function** instead of a boolean assertion:

```python
import dspy

def under_50_words(args, pred) -> float:
    return 1.0 if len(pred.answer.split()) <= 50 else 0.0

qa = dspy.ChainOfThought("question -> answer")

# Retry up to 3 times with feedback until the reward clears the threshold
refined = dspy.Refine(module=qa, N=3, reward_fn=under_50_words, threshold=1.0)

# Or sample 3 independently and keep the best
best = dspy.BestOfN(module=qa, N=3, reward_fn=under_50_words, threshold=1.0)
```

- `dspy.Refine` feeds the failed attempt back as a hint, which suits soft constraints such as length or format.
- `dspy.BestOfN` samples independently and keeps the highest reward, which suits noisy tasks.
- **Hard constraints** ("must not contain PII") should not rely on retries at all. Put a deterministic validator after the module and fail closed; a reward function makes compliance likely, not guaranteed.

---

## Managing Model Drift

When OpenAI or Anthropic releases a new model, hand-crafted prompts often break. In September 2026 alone, Anthropic shipped Fable 5.1, Opus 5.5 and Sonnet 5.5 and OpenAI shipped GPT-6 Astra, GPT-6 Sol and Luna, and GPT-6.1 Sol.
- **The DSPy answer**: **Re-compile**. The optimizer finds new instructions and demos for the new model against the same metric, which turns a model migration into a measured job rather than a rewrite.
- **The catch**: re-compilation is only as good as the eval set and metric behind it, and it costs real model calls. Budget it per model swap, and gate the swap on the compiled program beating the old one.

---

## DSPy 3.3 and 3.4 Changes

DSPy churns like every other framework, so pin it:

| Release | Date | What changed |
|---|---|---|
| 3.3.0 | Aug 3, 2026 | Flex structure optimization; **ReActV2** with native tool calling; start of a typed, provider-neutral LM boundary; NumPy optional |
| 3.3.1 | Aug 21, 2026 | GEPA 0.1.4; hardened `PythonInterpreter`; MCP Python SDK v2 compatibility |
| 3.4.0 | Sep 25, 2026 | Native LM engines with custom HTTP providers. **Breaking**: the experimental 3.3 typed LM API is removed. Also async ReActV2, `LocalInterpreter` (persistent CPython for trusted code, used with RLM and Flex), ReAnchor, and experimental typed decision outputs (`Noul`, `Choice`, `Score`) that integrate TypeSafe's Jev decision model |
| 3.4.1 | Oct 12, 2026 | GEPA's structured `InstructionProposer` becomes the default instruction proposer, so **optimization results can differ from 3.4.0 even with the same seed**; `dspy.RLM` can build DSPy sub-agents from inside its REPL; Python 3.15 support; fixes a 3.4.0 import crash |

Two deprecations to plan for: OpenAI-style `messages=` calls now warn and are slated for removal in 3.5, and custom `BaseLM.forward` integrations run with deprecation warnings until 3.5.

---

## Interview Questions

### Q: Why is DSPy considered "Anti-Prompt Engineering"?

**Strong answer:**
Because it replaces the **Manual trial-and-error loop** with an **Optimization Loop**. In prompt engineering, the human is the optimizer. In DSPy, the human is the **Teacher**. You define the *Goal* (Signature) and the *Evaluation* (Metric), and you provide a few *Examples*. The framework then uses mathematical optimization (like Bayesian search) to find the tokens that statistically perform the best. This makes the system far more **Portable** and **Scalable** than a library of hardcoded strings.

### Q: What is the biggest drawback of using DSPy in a production environment?

**Strong answer:**
**Compilation Latency and Cost**. To compile a complex DSPy pipeline, you might need to run 100-500 LLM calls to test different prompt variations, and you repeat that for every model you target. This is a significant upfront cost. However, for a Staff-level engineer, this is a **Tradeoff**: You pay more in development/compilation time to gain **measured reliability** and lower **Run-time Failure Rates** against a metric you control. Nothing about it is guaranteed: the compiled program is only as good as the eval set. Another challenge is the learning curve; it requires thinking like an ML researcher rather than a traditional developer. A third is framework churn: 3.4 removed an experimental API that 3.3 had introduced less than eight weeks earlier, and the 3.4.1 patch changed GEPA's default proposer so the same seed no longer reproduces 3.4.0 results. Pin the version, record it with every compiled artifact, and re-run the eval suite on upgrades.

### Q: When would you use GEPA or Flex instead of MIPROv2?

**Strong answer:**
MIPROv2 searches over instructions and demos for a fixed pipeline, and it needs enough rollouts to make Bayesian search work. GEPA reflects on traces and textual feedback, so it fits when each rollout is expensive (long agent runs) or when my metric can say *why* an output failed. Flex goes one level up and changes the program structure through GEPA, which I would try when error analysis says the decomposition is wrong, for example a missing verification step, rather than the wording. In every case the deciding input is the metric: a weak metric lets any optimizer overfit, so I keep a held-out set the optimizer never sees.

---

## References
- Khattab et al. "DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines" (arXiv 2310.03714, Oct 2023): https://arxiv.org/abs/2310.03714
- Opsahl-Ong et al. "Optimizing Instructions and Demonstrations for Multi-Stage Language Model Programs" (arXiv 2406.11695, Jun 2024; the MIPRO paper): https://arxiv.org/abs/2406.11695
- Agrawal et al. "GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning" (arXiv 2507.19457, Jul 2025): https://arxiv.org/abs/2507.19457
- DSPy release notes (3.4.0, Sep 25, 2026; 3.4.1, Oct 12, 2026): https://github.com/stanfordnlp/dspy/releases
- DSPy. "Output Refinement: BestOfN and Refine": https://dspy.ai/tutorials/output_refinement/best-of-n-and-refine/

---

*Next: [Semantic Kernel: Enterprise AI](06-semantic-kernel.md)*
