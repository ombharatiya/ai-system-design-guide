# Synthetic Data Generation

The industry has hit the "Data Wall", the exhaustion of high-quality human text on the web. Synthetic data is now the primary engine for model improvement, sitting at the core of every modern frontier-model recipe.

## Table of Contents

- [The Data Wall and the Synthetic Shift](#after-the-data-wall-the-synthetic-shift)
- [Evol-Instruct Pattern](#evol-instruct-pattern)
- [Constitutional AI & AI Feedback (RLAIF)](#constitutional-ai--ai-feedback-rlaif)
- [Verifiable Synthetic Data (Math/Code)](#verifiable-synthetic-data)
- [De-biasing and Diversity](#de-biasing-and-diversity)
- [Provenance and Licensing](#provenance-and-licensing)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## After the "Data Wall": The Synthetic Shift

Disclosed pretraining budgets for open models already reach 15T to 45T tokens (Llama 4's mixture was more than 30T, and Thinking Machines Lab's Inkling disclosed 45T multimodal tokens in July 2026), and the supply of high-quality, legally clean human text is not growing at that rate. Closed labs do not disclose their totals.
**The reality:** Post-training sets are now mostly model-generated (reasoning traces, rewrites, verified solutions), and synthetic shares of pretraining are rising: Microsoft's Phi-4 report allocates **40% of pretraining tokens** to synthetic data. Most labs do not publish their mix, so treat any single industry-wide percentage as a guess.

| Source | Human Data | Synthetic Data |
|--------|------------|----------------|
| **Volume** | Fixed (Finite) | Infinite |
| **Quality** | Variable (Noisy) | Controllable (Purified) |
| **Cost** | High (Human Labelers) | Cheap (Inference/GPU) |
| **Bias** | Mirror of internet | Can be manually balanced |

---

## Evol-Instruct Pattern

Evol-Instruct is a recursive process where an LLM takes a simple instruction and evolves it into a more complex one.

**The Evolution Directions (from the WizardLM paper):**
1. **In-depth**: Add constraints, deepen the question, make it more concrete, or require more reasoning steps.
2. **In-breadth**: Mutate the instruction into a new, rarer one in the same domain to widen topic coverage.
3. **Elimination**: Discard failed evolutions (no new information, unanswerable, or just a copy of the evolving prompt) before they reach the training set.

```python
# Simple Instruction: "Write a function to add two numbers."
# Evolved Instruction: "Write a thread-safe Python class that performs 
# matrix addition with error handling and unit tests, adhering to PEP8."
```

---

## Constitutional AI & AI Feedback (RLAIF)

Developed by Anthropic and widely adopted across the industry, RLAIF uses a "Constitution" (a set of rules) to guide a model in evaluating and improving its own data.

**The Loop:**
1. **Propose**: Model A generates a response.
2. **Critique**: The same model, or a separate judge, identifies flaws against the constitution's principles.
3. **Revise**: Model A produces a better version based on the critique.
4. **Train**: The final (Prompt, Revise) pair is added to the SFT set.

---

## Verifiable Synthetic Data

The biggest risk of synthetic data is **Model Collapse** (the model learning its own mistakes).
**The standard fix**: Focus on domains where the "Truth" is verifiable without an LLM.

- **Math**: Use Formal Verification (Lean/Isabelle) or Python execution to verify answers. Formal checking now scales to very long machine-generated outputs: Anthropic reports an internal model formalized Fermat's Last Theorem in Lean in 11 days, about 13M lines, with Lean verifying the result (vendor-reported, reviewed by Kevin Buzzard).
- **Code**: Run generated code against test cases (Unit Tests).
- **RAG**: Use "Gold Context" to generate questions where the answer is explicitly in the text.

---

## De-biasing and Diversity

Synthetic data is used to "fill the gaps" in human data.
- **Languages**: Generating high-quality text in low-resource languages (e.g., Swahili, Marathi) by translating conceptual templates.
- **Logic**: Creating 1,000,000 variations of a specific logical fallacy to "harden" the model against it.

---

## Provenance and Licensing

Every synthetic row inherits obligations from the model that generated it.

- **Teacher terms**: Closed-model terms usually forbid training competing models on outputs, and vendors now detect bulk CoT-harvesting traffic (see [Knowledge Distillation](05-knowledge-distillation.md#distillation-as-a-threat-extraction-via-api)). Open teachers vary: MIT and Apache 2.0 are clean; custom licenses (Kimi K3, Qwen Community 1.0) add conditions for hosted products.
- **Lineage crosses borders**: NVIDIA's competitive-coding research model (above the IOI 2026 gold threshold in an unofficial run) was trained on 477,642 traces distilled from Z.ai's GLM-5.2. GLM-5.2's MIT license permits that, but your compliance team will still ask which models touched the data.
- **Record per row**: generator model and version, its license, the prompt template, the verifier that accepted it, and the seed corpus it was derived from. Courts are separating lawful training from unlawful acquisition of source material, so the seed data's acquisition record matters as much as the generator's license.

---

## Interview Questions

### Q: What is the risk of "Model Collapse" when training on synthetic data?

**Strong answer:**
Model Collapse occurs when a model is trained on data generated by an earlier version of itself. Because the model's distribution is narrower than the real world (it has preferences/biases for certain words and patterns), the training loop becomes a "positive feedback loop" of errors and blandness. The standard mitigations:
1. Mixing in 5-20% "Golden" human-authenticated data.
2. Using "Verifiable" rewards (Math/Code) so mistakes are never learned.
3. Using more powerful "Teacher" models to generate data for "Student" models.

### Q: How do you ensure the *quality* of a synthetic dataset of 10 million rows?

**Strong answer:**
We use a **Multi-Stage Filtering Pipeline**:
1. **Semantic Deduplication**: Using embeddings to remove near-identical clusters.
2. **LLM-as-Judge**: Sampling 1% of the data and having a stronger model score it for logic and safety, with the judge calibrated against a small human-labeled set first.
3. **Perplexity Filtering**: Using a small model to calculate the perplexity of the text. If it's too high (nonsense) or too low (repetitive/simple), it's discarded.
4. **Verifiable Execution**: If the data contains code or math, it must pass a local compiler/interpreter check.

---

## References
- Xu et al. "WizardLM: Empowering Large Language Models to Follow Complex Instructions" (2023)
- Bai et al. "Constitutional AI: Harmlessness from AI Feedback" (2022)
- OpenAI. "Weak-to-Strong Generalization" (2023)
- Abdin et al. "Phi-4 Technical Report" arXiv:2412.08905 (2024)

---

*Next: [Quantization Deep Dive](07-quantization-deep-dive.md)*
