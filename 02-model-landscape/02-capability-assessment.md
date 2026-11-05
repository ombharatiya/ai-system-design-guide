# Capability Assessment

This chapter covers how to evaluate and compare model capabilities for your specific use case. Generic benchmarks rarely tell the full story; this guide helps you conduct meaningful assessments.

## Table of Contents

- [Why Benchmarks Are Not Enough](#why-benchmarks-are-not-enough)
- [Reading a Vendor Benchmark Table](#reading-a-vendor-benchmark-table-november-2026)
- [Evaluation Dimensions](#evaluation-dimensions)
- [Reasoning Calibration & Efficiency](#reasoning-calibration)
- [Internal Elo-based Evaluation](#internal-elo-based-evaluation)
- [Building Custom Evaluations](#building-custom-evaluations)
- [Common Evaluation Pitfalls](#common-evaluation-pitfalls)
- [Practical Assessment Process](#practical-assessment-process)
- [A/B Testing Models](#ab-testing-models)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Why Benchmarks Are Not Enough

### The Benchmark Problem

Public benchmarks, from the saturated (MMLU, HumanEval, GPQA Diamond) to the current agentic suites (SWE-Bench Pro, Terminal-Bench 4.0, OSWorld 2.0), have limitations:

| Issue | Impact |
|-------|--------|
| Training data contamination | Models may have seen test questions. SWE-Bench Pro v2's public split is at 99.4% (Claude Opus 5) while its private 272-task set is at 81.6%; Scale attributes most of the gap to training exposure |
| Unearned passes | Agents find answers instead of solving tasks. An audit of SWE-Bench Pro v1.0 found 24% (Opus 4.7) to 73% (Fable 5) of passes unearned, mostly via reference solutions in git history (arXiv 2609.34262) |
| Harness dependence | Same model, same test set: GPT-6 Astra scored 62.7% on ARC-AGI-3 with ARC Prize's provider-neutral harness and 99.9% with OpenAI's adapter that preserves reasoning state between requests |
| Metric choice | OSWorld 2.0's primary metric is binary task completion, but launch posts quote partial credit. On the official leaderboard's v2.1 full set, Opus 5 (max) scores 44.33% binary and 77.67% partial on the same run |
| Safety fallback | Claude Opus 5.5 was benchmarked with safeguards on; when they intervened, Opus 4.8 or Opus 5 completed the task. Artificial Analysis labels these entries "with fallback" |
| Task mismatch | Benchmarks may not reflect your use case |
| Weak construct validity | Benchmarks assigned the same concept correlate about as strongly with each other as with unrelated ones (56 benchmarks across 53 models, arXiv 2609.08812) |
| Aggregate scores hide variance | Model A may beat B overall but lose on your domain |
| Gaming | Models optimized for benchmarks over real tasks |
| Saturation | Benchmarks lag behind model capabilities: GPQA Diamond sits around 96% at the frontier |

### What Benchmarks Tell You

```
Benchmark results tell you: "Model X scored 58% on Terminal-Bench 4.0"

What you need to know: "Will Model X, at the effort level I can afford,
in my harness, finish my team's real tasks without supervision?"
```

**Rule of thumb:** Use benchmarks for initial filtering, then conduct your own evaluation.

### Reading a Vendor Benchmark Table (November 2026)

Every score needs six labels before it means anything: **benchmark version, harness, effort level, who ran it (leaderboard or vendor), fallback state, and metric**. Add **cost per task** whenever two scores are close. September and October 2026 launches show why:

| Claim you will see | What was actually measured |
|--------------------|----------------------------|
| "Opus 5.5 scores 66.4% on Terminal-Bench 4.0" | Anthropic's own run at xhigh effort (vendor-reported, SE ±2.6). On the tbench.ai leaderboard (entries added October 6), Opus 5.5 is first at 64.85% (max) and Sonnet 5.5 second at 61.82% (max), ahead of GPT-6 Astra and GPT-6.1 Sol (tied at 58.18%). Anthropic's own runs reverse the Claude order (Sonnet 5.5 70.6%, Opus 5.5 66.4%) |
| "Astra 57.9%" vs "Astra 58.2%" on Terminal-Bench 4.0 | OpenAI's figure is high effort; 58.18% is the leaderboard's max-effort run |
| "Fable 5.1 55.8%" vs "Fable 5.1 57.9%" | Anthropic's own run vs the leaderboard max-effort entry (57.88%), which xAI quoted in its Grok 4.7 table |
| "GPT-6 Astra 99.9% on ARC-AGI-3" | ARC Prize's Provider Adapter harness, high effort, $18,817. Its Standard harness at max effort scored 62.7% for $26,098 |
| "Opus 5.5 81.8% on OSWorld 2.1" | Partial-credit metric, vendor run. On the official leaderboard's v2.1 full set (September 17), Opus 5 at max effort scores 44.33% binary and 77.67% partial; that update predates Opus 5.5 |
| HLE with tools: Astra ahead, or Opus 5.5 ahead? | Both. HLE-Diamond (1,000 cleaned questions): Astra 82.9%, Opus 5.5 73.9%. Anthropic's full-HLE table: Opus 5.5 67.7%, Astra 57.2%. Dataset and runner flip the order |
| "Three models tie at 74% on DeepSWE v1.1" | Same pass rate at $2.36 per task (Gemini 3.8 Flash, high), $4.43 (GPT-6 Astra, xhigh) and $11.84 (Claude Opus 5, max) |
| "Opus 5.5 scores 58 on the AA Intelligence Index" vs "59" | 58 is Index v5.0 (announced October 27), max effort, with safeguard fallback; 59 is v5.0.1 (November 3), which regraded every model on CritPt v2 and put Gemini 4 Carbon, which developers cannot call, at 60. v4.x scores are on a different scale and cannot be compared, even when a number happens to match |
| "Sonnet 5.5 scores 54" vs "Sonnet 5.5 scores 41" | Both AA v5.0: 54 at max effort, 41 at high. One effort setting moved it 13 points; GPT-6.1 Terra moves from 51 at max to 38 at its default `medium` |
| "Haiku 5.5 scores 72.4% on OSWorld 2.1" | Anthropic's offline subset with partial credit, vendor-reported. Not the leaderboard's binary metric, and not comparable with full-set scores |

The rule that follows: **only same-harness, same-effort, same-runner numbers are comparable.** A vendor number is a claim about the vendor's best configuration, not a prediction for yours. The [benchmarks chapter](../14-evaluation-and-observability/03-benchmarks-and-leaderboards.md#harness-and-scaffold-variance) covers harness variance in more depth.

---

## Evaluation Dimensions

### Dimension 1: Task Performance

| Task Type | Evaluation Approach | Key Metric |
|-----------|---------------------|------------|
| **Autonomous Coding** | SWE-Bench Pro v2 (private set), DeepSWE v1.1, Terminal-Bench 4.0, then your own repositories | % tasks resolved, cost per resolved task |
| **Long-Horizon Planning** | Agentic loop testing (tau3-bench v1.0.1, Agents' Last Exam) | Success rate on 10+ step plans |
| **Computer Use** | OSWorld 2.0 | Binary completion first, partial credit second |
| **Reasoning Depth** | Effort sweep (low to max) | Accuracy and tokens at each effort level |
| **Long Context RAG** | Multi-needle and reasoning recall (MRCR, AA-LCR) at up to 1M tokens | Recall and cross-document reasoning at depth |
| **Native Multimodal** | Interleaved Vision/Voice/Text | Sync accuracy across modalities |

### Dimension 2: Agentic Mastery

How well does the model use tools and follow multi-step instructions?

```python
def evaluate_agentic_flow(agent, task_environment):
    """
    Measure success on 'Autonomous Agent' tasks:
    1. Plan generation
    2. Tool selection accuracy
    3. Error recovery
    4. Feedback loop utilization
    """
    results = []
    for scenario in task_environment.scenarios:
        traj = agent.run(scenario.goal)
        results.append({
            "success": traj.reached_goal,
            "steps": len(traj.steps),
            "tool_errors": traj.count_invalid_tool_calls()
        })
    return aggregate(results)
```

Grade the final state with fresh, isolated checks, not the agent's own report, and lock the sandbox's network to the model endpoint where you can. SWE-Bench Pro v2 now does both (a network-locked sandbox plus a re-grade of every diff on a pristine image), and the re-grade caught a frontier model forging a Go module version.

### Dimension 3: Reasoning Reliability

Thinking is now a dial, not a switch. Claude Opus 5.5 cannot disable thinking, Sonnet 5.5's lowest setting is `between_tools`, and GPT-6 Astra has no `none` effort. The question is which **effort level** your task needs:

| Effort (illustrative) | Accuracy (Math) | Accuracy (Code) | Avg Latency | Tokens / Output |
|-----------------------|-----------------|-----------------|-------------|-----------------|
| **Low** | 78% | 74% | 2.1s | 600 |
| **Medium** | 88% | 84% | 5.8s | 1,500 |
| **High** | 93% | 88% | 12.5s | 2,800 |
| **Max** | 94% | 89% | 31s | 7,000 |

Defaults move under you: Opus 5.5 and Haiku 5.5 default to `medium` where Opus 5 used `high`, GPT-6.1 Sol and GPT-6.1 Terra default to `medium`, and Sonnet 5.5 defaults to `high` on the API but medium in Claude Code. Pin effort explicitly in both evals and production. A max-effort eval does not predict a medium-effort deployment.

### Reasoning Calibration

**The "Over-Thinking" Problem:**
Models often spend 2000+ "thinking" tokens on a question that could be answered with 10 tokens (e.g., "What is 2+2?").

**Principal-level Nuance:**
Evaluate models based on **Logic Efficiency**: `Accuracy / (Inference Tokens)`.
Production systems use **Model Arbitration**: a small, cheap model (GPT-6 Luna, Claude Haiku 5.5 on prompts under 100K tokens, Gemini 3.8 Flash at low thinking) classifies the query and picks the effort level, or the model, for the main call. On models where thinking cannot be turned off, the arbiter's job is choosing `low` versus `high`, which still avoids the 10x latency and cost penalty on simple queries.

Measure **tokens per task**, not just accuracy. Models differ several-fold here: Artificial Analysis measured GPT-6 Astra using about a third of GPT-5.6 Sol's tokens per coding task, while Muse Spark 1.3 became more verbose than 1.2, so its cost per task rose even though its per-token price did not change.

### Dimension 4: Context Recall

With 1M-token windows now standard (Claude 4.6 and later, GPT-6 at 1.05M, Gemini 3.8 Flash and Gemini 4 Argon at 1,048,576), simple "needle-in-a-haystack" is no longer enough. We now measure **Contextual Reasoning** across the window.

| Metric | Measurement | Target |
|--------|-------------|--------|
| **Window Recall** | Factual recall at 90% window depth | > 98% |
| **Cross-Doc Reasoning** | Logic linking Doc A (pos 10k) to Doc B (pos 900k) | > 90% |
| **Contextual Noise Resistance** | Accuracy when 90% of window is irrelevant "filler" | > 95% |
| **Long-Output Coherence** | Consistency across 64K-128K generated tokens (current output caps are 128K on Claude 5.x and GPT-6 models and 65,536 on Gemini 3.8 Flash; Gemini 4 Argon defaults to 65,536 and reaches about 1M only through its long-decode continuation protocol) | Task-specific |

Record cost alongside recall. On OpenAI models a prompt above 272K tokens bills the whole request at long-context rates, and on Claude Haiku 5.5 a prompt above 100K bills it at 5x, so a model that recalls well at 800K may still lose to RAG on price, and a cheap model can stop being cheap at 101K. The same applies to output: Argon's long decode lets one logical call replace a chunked generation loop, but only after you have measured coherence and cost at that length.

---

## Internal Elo-based Evaluation

**Moving beyond static rubrics.**
Rubrics (1-5 scales) are prone to "judge fatigue" and "score drifting." Modern systems use **Pairwise Elo** for internal golden sets.

**The Workflow:**
1. **Blind Side-by-Side:** Model A and Model B generate answers for the same query.
2. **The Judge:** A strong model (Claude Opus 5.5, GPT-6 Astra, or a human) selects the winner. Run each pair twice with positions swapped; you can no longer pin temperature on most frontier judges, so measure judge variance instead of assuming it away.
3. **Elo Update:** Update the internal leaderboard.

```python
def update_elo(winner_elo, loser_elo, k=32):
    expected_winner = 1 / (1 + 10 ** ((loser_elo - winner_elo) / 400))
    new_winner_elo = winner_elo + k * (1 - expected_winner)
    new_loser_elo = loser_elo + k * (0 - (1 - expected_winner))
    return new_winner_elo, new_loser_elo
```

**Why it wins:** It provides a **relative** ranking that is much more robust to changes in judge personality or model versioning.

**Judge cost is now a design choice.** Decision-model judges that return calibrated scores instead of text (TypeSafe's Jev) cost 16 to 325x less than LLM rubric judges, and accuracy differed significantly in at most 8 of 27 paired comparisons (arXiv 2609.29769). But on the decision model's most confident errors, about 96% of LLM verdicts repeated the same wrong answer, and judge cascades gained at most 2.7 points. Cheap judges are fine for binary checklist criteria; a second judge from the same family does not buy independence.

---

## Building Custom Evaluations

### Step 1: Define Evaluation Criteria

```python
evaluation_criteria = {
    "correctness": {
        "weight": 0.4,
        "description": "Is the answer factually correct?",
        "scale": [1, 2, 3, 4, 5],
        "rubric": {
            5: "Completely correct, no errors",
            4: "Mostly correct, minor issues",
            3: "Partially correct, some errors",
            2: "Mostly incorrect",
            1: "Completely wrong or nonsensical"
        }
    },
    "relevance": {
        "weight": 0.3,
        "description": "Does the answer address the question?",
        "scale": [1, 2, 3, 4, 5]
    },
    "completeness": {
        "weight": 0.2,
        "description": "Are all parts of the question addressed?",
        "scale": [1, 2, 3, 4, 5]
    },
    "conciseness": {
        "weight": 0.1,
        "description": "Is the answer appropriately concise?",
        "scale": [1, 2, 3, 4, 5]
    }
}
```

### Step 2: Create Test Set

```python
test_set = [
    {
        "id": "q001",
        "query": "What is the refund policy for subscription cancellation?",
        "context": "[relevant documentation]",
        "ground_truth": "Full refund within 30 days, prorated after",
        "difficulty": "easy",
        "category": "policy"
    },
    {
        "id": "q002",
        "query": "How do I integrate the API with a Python async application?",
        "context": "[API documentation]",
        "ground_truth": "[expected code pattern]",
        "difficulty": "medium",
        "category": "technical"
    },
    # ... 50-100+ test cases
]
```

**Test set guidelines:**
- Cover all major use cases
- Include easy, medium, hard examples
- Balance across categories
- Include edge cases
- Have clear ground truth answers
- Draw from real production traffic where possible, so it does not read like a test

### Step 3: Implement Evaluation

```python
class ModelEvaluator:
    def __init__(self, models: list[str], test_set: list[dict]):
        self.models = models
        self.test_set = test_set
        self.results = {}
    
    def evaluate_all(self):
        for model in self.models:
            self.results[model] = self.evaluate_model(model)
        return self.results
    
    def evaluate_model(self, model: str) -> dict:
        scores = []
        latencies = []
        
        for case in self.test_set:
            start = time.time()
            response = self.generate(model, case)
            latency = time.time() - start
            latencies.append(latency)
            
            # Score using LLM judge or human
            score = self.score_response(case, response)
            scores.append(score)
        
        return {
            "mean_score": mean(scores),
            "score_by_category": self.group_by_category(scores),
            "p50_latency": percentile(latencies, 50),
            "p99_latency": percentile(latencies, 99)
        }
    
    def score_response(self, case: dict, response: str) -> float:
        # Option 1: LLM-as-judge
        return self.llm_judge(case, response)
        
        # Option 2: Exact match
        # return exact_match(response, case["ground_truth"])
        
        # Option 3: Semantic similarity
        # return cosine_sim(embed(response), embed(case["ground_truth"]))
```

### Step 4: LLM-as-Judge

```python
def llm_judge(case: dict, response: str) -> dict:
    prompt = f"""Evaluate this response to a customer query.

Query: {case['query']}
Expected Answer: {case['ground_truth']}
Model Response: {response}

Rate the response on these criteria (1-5 scale):
1. Correctness: Is it factually accurate?
2. Relevance: Does it answer the question?
3. Completeness: Are all aspects covered?
4. Conciseness: Is it appropriately brief?

Output JSON:
{{"correctness": X, "relevance": X, "completeness": X, "conciseness": X, "reasoning": "..."}}
"""
    
    result = judge_model.generate(prompt)
    return parse_json(result)
```

---

## Common Evaluation Pitfalls

### Pitfall 1: Small Test Set

**Problem:** 20 test cases is not enough for reliable comparison.

**Solution:** Aim for 100+ cases, stratified by difficulty and category.

### Pitfall 2: Ambiguous Ground Truth

**Problem:** "Reasonable" answers get marked wrong.

```
Query: "What is the capital of Australia?"
Ground truth: "Canberra"
Model answer: "The capital of Australia is Canberra."
Exact match: FAIL (but clearly correct)
```

**Solution:** Use semantic matching or LLM judge, not exact match.

### Pitfall 3: Evaluation Set Leakage

**Problem:** Using same cases for development and evaluation.

**Solution:** Keep a held-out test set that you never use for prompt tuning.

### Pitfall 4: Ignoring Variance

**Problem:** Running each test once ignores model randomness.

**Solution:** Run each case at least three times at the production effort setting and report confidence intervals. Temperature is no longer a lever on most frontier models: Claude API version `2026-10-21`, which all eight of Anthropic's new SDK and CLI majors send by default, returns 400 on any `temperature`, `top_p`, or `top_k` on every model (earlier versions reject non-default values on Claude 4.7 and later); Google deprecated temperature in the Gemini API; and GPT-6 Astra does not accept it. Variance is something you measure, not something you tune away.

### Pitfall 5: Cost Blindness

**Problem:** Best model is 10x more expensive.

**Solution:** Always report quality-adjusted cost, per task rather than per token. The DeepSWE v1.1 three-way tie at 74% spans $2.36 to $11.84 per task.

```python
def quality_adjusted_cost(model_results):
    return {
        model: {
            "quality": results["mean_score"],
            "cost_per_1k": results["cost_per_1k_queries"],
            "quality_per_dollar": results["mean_score"] / results["cost_per_1k"]
        }
        for model, results in model_results.items()
    }
```

### Pitfall 6: Evaluating a Different System Than You Ship

**Problem:** The eval ran at max effort, in a vendor harness, with safeguard fallback to another model, or against a model that has since changed under the same ID.

**Solution:** Match effort, harness, tools and fallback settings to production, and re-run evals on provider-side changes, not only on model-ID changes. OpenAI fixed an image-understanding bug in GPT-6 Sol and Luna on September 25, 2026 without changing the model ID and told customers to rerun image evals. DeepSeek's legacy `deepseek-v4-flash` names now serve V4.1-Flash, and `deepseek-flash` moved to the V4.1-Flash-1017 build on October 17. Google has served `gemini-3.7-flash` requests with Gemini 3.8 Flash since October 8, so an eval pinned to 3.7 Flash now measures a different model.

### Pitfall 7: Eval Awareness

**Problem:** Models behave differently when they recognize a test. Anthropic reports Opus 5.5 showing evaluation awareness in 36% of audit transcripts against 0.4% of internal Claude Code transcripts.

**Solution:** Build eval sets from real (scrubbed) production traffic and shadow-mode runs rather than obviously synthetic prompts, and weight production signals over offline scores when they disagree.

---

## Practical Assessment Process

### Week 1: Setup and Initial Filtering

```
Day 1-2: Define evaluation criteria and create test set
Day 3-4: Benchmark 4-6 candidate models at the effort level you can afford to ship
Day 5: Analyze results, filter to top 2-3
```

### Week 2: Deep Evaluation

```
Day 1-2: Expand test set for top candidates
Day 3: Test edge cases and robustness
Day 4: Measure latency and throughput
Day 5: Calculate total cost of ownership
```

### Week 3: Production Validation

```
Day 1-2: Shadow mode deployment
Day 3-4: A/B test if traffic allows
Day 5: Final decision and documentation
```

### Decision Template

```markdown
## Model Evaluation Report

### Candidates Evaluated
- Model A: GPT-6 Sol (effort: medium)
- Model B: Claude Sonnet 5.5 (effort: high, API default)
- Model C: GLM-5.3-Flash (API)

### Evaluation Results

| Metric | Model A | Model B | Model C |
|--------|---------|---------|---------|
| Overall Score | 4.2/5 | 4.3/5 | 3.9/5 |
| Category 1 | ... | ... | ... |
| P50 Latency | 4.1s | 5.2s | 3.6s |
| Tokens per task (2,600 in + output incl. reasoning) | 3,500 | 3,700 | 4,000 |
| Cost/1K queries (list price) | $14.20 | $16.20 | $1.09 |

### Recommendation
Model B (Claude Sonnet 5.5) for quality-critical paths
Model C (GLM-5.3-Flash) for high-volume, cost-sensitive paths

### Rationale
[Detailed reasoning, including harness, effort and retirement dates]
```

---

## A/B Testing Models

### When to A/B Test

- High traffic (1000+ queries/day)
- Clear success metrics
- Acceptable risk of quality variation
- Need production validation

### A/B Test Design

```python
import hashlib

class ModelABTest:
    def __init__(self, model_a: str, model_b: str, traffic_split: float = 0.5):
        self.model_a = model_a
        self.model_b = model_b
        self.traffic_split = traffic_split
        self.results = {"a": [], "b": []}
    
    def route_request(self, request_id: str) -> str:
        # Deterministic routing for consistency. Python's built-in hash() is
        # salted per process for strings, so use a stable digest instead.
        digest = hashlib.sha256(request_id.encode()).digest()
        hash_val = int.from_bytes(digest[:8], "big") % 100
        if hash_val < self.traffic_split * 100:
            return self.model_a
        return self.model_b
    
    def record_outcome(self, request_id: str, metrics: dict):
        model = self.route_request(request_id)
        bucket = "a" if model == self.model_a else "b"
        self.results[bucket].append(metrics)
    
    def analyze(self):
        return {
            "model_a": {
                "name": self.model_a,
                "mean_score": mean([r["score"] for r in self.results["a"]]),
                "sample_size": len(self.results["a"])
            },
            "model_b": {
                "name": self.model_b,
                "mean_score": mean([r["score"] for r in self.results["b"]]),
                "sample_size": len(self.results["b"])
            },
            "p_value": self.calculate_significance()
        }
```

Route on a stable unit (user or conversation ID), not a request ID, when the models carry state across turns. On Claude Fable 5.1, Opus 5.5, Sonnet 5.5 and Haiku 5.5, thinking blocks are bound to the model that produced them (and on Sonnet 5.5 and Haiku 5.5 to the producing account), so flipping a conversation between arms mid-stream silently drops reasoning.

### Metrics to Track

| Metric Type | Examples |
|-------------|----------|
| Quality | User ratings, expert review, LLM judge |
| Engagement | Click-through, time on page, follow-up queries |
| Business | Conversion, support escalation, resolution rate |
| Operational | Latency, errors, cost per task, refusal rate |

---

## Interview Questions

### Q: How would you evaluate models for a customer support chatbot?

**Strong answer:**
I would structure evaluation in layers:

**1. Offline evaluation (80% of effort):**
- Create test set from real support tickets (200+ cases)
- Cover all categories: billing, technical, returns, general
- Include easy, medium, hard difficulty
- Measure: accuracy, helpfulness, safety

**2. Evaluation method:**
- Use LLM-as-judge for subjective metrics
- Human review for sample (20%)
- Track instruction following (format, length)

**3. Metrics:**
```python
metrics = {
    "resolution_accuracy": "Does answer solve the problem?",
    "safety": "No harmful/wrong advice?", 
    "tone": "Professional and empathetic?",
    "escalation_appropriate": "Knows when to involve human?"
}
```

**4. Production validation:**
- Shadow mode: run new model, compare outputs
- A/B test: 10% traffic to new model
- Monitor: CSAT, escalation rate, resolution time

### Q: What is wrong with using MMLU to compare models for your use case?

**Strong answer:**
MMLU has several problems for specific use cases:

**1. Domain mismatch:** MMLU tests academic knowledge. My customer support bot needs product knowledge.

**2. Format mismatch:** MMLU is multiple choice. My use case is free-form generation.

**3. Contamination:** Models may have trained on MMLU questions.

**4. Aggregation hides variance:** Model A might beat B on MMLU but lose on the specific categories I care about.

**5. No context testing:** MMLU does not test RAG or long-context abilities.

**6. Saturation:** Frontier models cluster at the top, so it no longer separates them. Artificial Analysis now lists MMLU-Pro, GPQA Diamond and similar evals as legacy, outside its current index.

**Better approach:** 
- Use benchmarks for initial filtering (saves time), preferring contamination-resistant ones with private splits
- Build custom evaluation for final decision
- Test on actual use case data
- Include operational metrics (latency, cost per task)

### Q: A vendor reports 99.9% on ARC-AGI-3 for its new model. ARC Prize's own harness shows 62.7% for the same model. Which number do you use?

**Strong answer:**
Both are real; they measure different systems. ARC Prize verified both for GPT-6 Astra. The 99.9% ran on a Provider Adapter harness that preserves the model's opaque reasoning state between requests, at high effort, for about $19K. The 62.7% ran on the Standard harness, a provider-neutral interface where the model only keeps notes it chooses to write, at max effort, for about $26K.

**Which one predicts my system:**
- If my agent runs on the vendor's stateful API and harness (for OpenAI, the Responses API with reasoning state carried across turns), the adapter number is closer to my setup.
- If I run a provider-neutral harness, route across vendors, or rebuild context each turn, the standard number is the honest one. Every cross-vendor comparison must use it.

**What I take from the gap:** about 37 points separate two runs of the same weights, and the scaffold is the difference (the lower score even ran at higher effort). So I evaluate in my own harness, at my own effort level, and I treat state handling (what persists between turns, and whether failover drops it) as part of the model choice. I also record cost per task: here the higher score was cheaper.

---

## References

- Zheng et al. "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (2023)
- Arena (formerly LMArena / LMSYS Chatbot Arena): https://arena.ai/
- Artificial Analysis Intelligence Index methodology: https://artificialanalysis.ai/methodology/intelligence-benchmarking
- ARC Prize, GPT-6 Astra on ARC-AGI-3: https://arcprize.org/blog/astra
- OSWorld 2.0 leaderboard (XLANG): https://osworld-v2.xlang.ai/
- Terminal-Bench 4.0 leaderboard: https://www.tbench.ai/leaderboard/terminal-bench/4.0
- "Maintaining Benchmarks Against Increasingly Capable Agents" (arXiv 2609.34262)
- "JEV vs. LLMs as Rubric Judges" (arXiv 2609.29769)
- HELM: https://crfm.stanford.edu/helm/
- LMSys Evaluation: https://github.com/lm-sys/FastChat/tree/main/fastchat/llm_judge
- OpenAI Evals (open-source framework; the hosted Evals platform shuts down November 30, 2026): https://github.com/openai/evals

---

*Previous: [Model Taxonomy](01-model-taxonomy.md) | Next: [Pricing and Costs](03-pricing-and-costs.md)*
