# RLHF and DPO (Alignment)

Alignment is the process of ensuring an LLM's behavior matches human values and instructions. The field has moved from traditional RLHF to more efficient methods like DPO for offline preference tuning, and to online RL with verifiable or rubric-based rewards at the frontier.

## Table of Contents

- [The Alignment Problem](#the-alignment-problem)
- [RLHF: The Foundation](#rlhf-the-foundation)
- [DPO: Direct Preference Optimization](#dpo-direct-preference-optimization)
- [Online Alignment](#online-alignment)
- [Alignment for Reasoning Models](#alignment-for-reasoning-models)
- [Rubric Rewards: RL Beyond Verifiable Domains](#rubric-rewards-rl-beyond-verifiable-domains)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Alignment Problem

Pretrained models are "knowledgeable but uncontrolled." They may:
1. Generate harmful content (Safety).
2. Fail to follow instructions (Instruction Following).
3. Hallucinate wildly (Factuality).

Alignment creates "Reward Models" and "Policy Updates" to steer the model.

---

## RLHF: The Foundation

Reinforcement Learning from Human Feedback (RLHF) involves three steps:
1. **SFT**: Supervised Fine-Tuning.
2. **Reward Model (RM)**: Train a model on `(Prompt, Winning_Response, Losing_Response)` to predict human scores.
3. **PPO (Proximal Policy Optimization)**: Use the RM to provide a "reward signal" to the LLM via Reinforcement Learning.

**Nuance**: Traditional RLHF is now considered too complex/unstable for most teams due to the overhead of training a separate Reward Model and the instability of PPO.

---

## DPO: Direct Preference Optimization

DPO is the default for teams doing offline preference tuning. It eliminates the Reward Model. Frontier labs still run online RL (PPO, GRPO and variants) for their largest post-training stages; see [RLVR and Reasoning Models](08-rlvr-and-reasoning-models.md).

### How it Works:
DPO uses the LLM itself as the Reward Model by mathematically deriving the optimal policy directly from preference data.
- **Goal**: Maximize the probability of the "winning" response and minimize the "losing" response, relative to a fixed "reference model."

### The Multi-Stage Alignment Pattern:
1. **Base SFT**: 5k-10k high-quality samples.
2. **DPO Step 1**: Alignment for instruction following.
3. **DPO Step 2**: Alignment for safety and specific tone.

---

## Online Alignment

**The Problem with Offline DPO**: It only learns from static data. If the model improves beyond that data, it hits a ceiling.

**The Solution: Online DPO (or RLOO)**:
1. The model generates 4-8 responses to a prompt.
2. A **Judge Model** (a strong frontier model such as Claude Opus 5.5 or GPT-6 Astra) or a **Rule-based Reward** (e.g., Code Execution) ranks them in real-time.
3. The model updates its policy immediately based on this "Online" feedback.

---

## Alignment for Reasoning Models

Aligning "thinking" models (the o1 and DeepSeek-R1 lineage) shifts the reward from **human preference over responses** to **verified outcomes**, while deliberately keeping optimization pressure off the chain of thought itself.

| Feature | Standard Alignment | Reasoning Alignment |
|---------|-------------------|---------------------|
| Reward Target | The final response | The **verified final outcome** (sometimes process rewards on steps) |
| Reward Signal | Helpful/Safe | **Correctness + Conciseness** |
| Method | Human Ranking | Rule-based (e.g., "Did the code run?") |
| Chain of Thought | Not present | Generated freely; **not** graded for content |

**Principal-level Nuance**: Verification-based RL (RLVR) is what trained today's reasoning models: instead of humans saying what is better, hard verifiable outcomes (math answers, code test cases) are the reward signal. The second lesson is what **not** to reward. OpenAI's 2025 work on monitoring reasoning models found that penalizing "bad thoughts" in the CoT teaches models to hide intent rather than stop misbehaving, so labs keep the CoT unoptimized to preserve it as a monitoring signal. That signal is now under pressure: the GPT-6 Astra system card (September 2026) reports sharply higher CoT controllability than GPT-5.6 Sol and says OpenAI likely could not reliably catch covert sandbagging. Details in [RLVR and Reasoning Models](08-rlvr-and-reasoning-models.md#monitorability-do-not-train-against-the-chain-of-thought).

---

## Rubric Rewards: RL Beyond Verifiable Domains

Most valuable domains (medicine, law, support, writing) have no unit test. The current workaround is **rubric-based rewards**: a judge model scores each response against a checklist of criteria, and the scores become the RL reward. It reintroduces the Goodhart problem RLVR avoided.

- **Criteria compensate for each other.** If you add up rubric scores, the policy can buy points on easy criteria while failing the one that matters. In a clinical-consultation study (arXiv:2609.38847, September 2026), rubric coverage rose during training while appropriateness on held-out physician criteria fell **below the untrained model**.
- **Fix the aggregation, not just the judge.** The same paper's **protocol-level rubrics** count a dimension only if all of its criteria hold and its failure clause does not fire; that raised appropriateness by 10.8 points without losing coverage.
- **Score precision, not just coverage.** Recall-only rubrics reward padding. Training Qwen2.5-7B-Instruct with a coverage-only reward (arXiv:2610.33512, October 2026, judged by GPT-5.4 against the base model) beat the base model only 3.1% of the time on RubricHub-Medical and 7.1% on HealthBench; adding a precision term (each unit necessary and factually correct) raised that to 68.0% and 74.1% and cut average RubricHub-Medical response length from 9,195 to 3,533 (base model 2,432).
- **Hold out the criteria you care about most.** Evaluate on rubric items the reward never saw, and treat a rising reward with flat or falling held-out quality as reward hacking until proven otherwise.

---

## Interview Questions

### Q: Why is DPO often preferred over RLHF/PPO?

**Strong answer:**
DPO is preferred primarily due to its simplicity and stability. PPO requires maintaining four models in memory (Policy, Reference, Value, and Reward), which is extremely VRAM-intensive. Furthermore, PPO is notoriously sensitive to hyperparameters and often suffers from "reward hacking" or sudden collapse. DPO treats alignment as a simple classification problem on preference pairs, making it much more stable, easier to tune, and significantly cheaper to run. The tradeoff: DPO is offline, so it can only learn what is in the static pairs and can overfit them; when you have a verifier or a good judge and the budget for rollouts, online methods (online DPO, RLOO, GRPO) usually go further, which is why frontier labs still run online RL.

### Q: What is the risk of "Alignment Tax"?

**Strong answer:**
The "Alignment Tax" refers to the decline in a model's raw capabilities (e.g., coding, creative writing, or logical reasoning) after it is aligned for safety or specific personas. Because the model is being forced to prioritize safety or adherence to a specific style, it may become "too cautious" or lose the nuance it learned during pretraining. The main lever is the **KL constraint** to the reference model (the KL penalty in PPO, the `beta` parameter in DPO), which limits how far the policy drifts from the SFT model. The rest is process: mix capability data into the preference set and rerun capability evals after every alignment stage, so the tax shows up as a measured regression instead of a user complaint.

---

## References
- Rafailov et al. "Direct Preference Optimization: Your Language Model is Secretly a Reward Model" (2023)
- Schulman et al. "Proximal Policy Optimization Algorithms" (2017)
- OpenAI. "Learning to Reason with LLMs" (2024)
- Baker et al. "Monitoring Reasoning Models for Misbehavior and the Risks of Promoting Obfuscation" (OpenAI, 2025)
- Liu et al. "Scoring Higher, Answering Worse: Mitigating Reward Hacking in Rubric-Based RL via Protocol-Level Rubrics" arXiv:2609.38847 (2026)

---

*Next: [Knowledge Distillation](05-knowledge-distillation.md)*
