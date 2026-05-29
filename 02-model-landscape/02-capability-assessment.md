# Capability Assessment

This chapter covers how to evaluate and compare model capabilities for your specific use case. Generic benchmarks rarely tell the full story; this guide helps you conduct meaningful assessments.

## Table of Contents

- [Why Benchmarks Are Not Enough](#why-benchmarks-are-not-enough)
- [Evaluation Dimensions](#evaluation-dimensions)
- [Building Custom Evaluations](#building-custom-evaluations)
- [Common Evaluation Pitfalls](#common-evaluation-pitfalls)
- [Practical Assessment Process](#practical-assessment-process)
- [Internal Elo-based Evaluation](#internal-elo-based-evaluation)
- [Reasoning Calibration & Efficiency](#reasoning-calibration)
- [A/B Testing Models](#ab-testing-models)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Why Benchmarks Are Not Enough

### The Benchmark Problem

Public benchmarks (MMLU, HumanEval, GSM8K) have limitations:

| Issue | Impact |
|-------|--------|
| Training data contamination | Models may have seen test questions |
| Task mismatch | Benchmarks may not reflect your use case |
| Aggregate scores hide variance | Model A may beat B overall but lose on your domain |
| Gaming | Models optimized for benchmarks over real tasks |
| Outdated | Benchmarks lag behind model capabilities |

### What Benchmarks Tell You

```
Benchmark results tell you: "Model X scored 88% on MMLU"

What you need to know: "Will Model X correctly answer my 
customers' questions about our product documentation?"
```

**Rule of thumb:** Use benchmarks for initial filtering, then conduct your own evaluation.

---

## Evaluation Dimensions

### Dimension 1: Task Performance

| Task Type | Evaluation Approach | Key Metric |
|-----------|---------------------|------------|
| **Autonomous Coding** | CWE/SWE-bench (Verified) | % issues resolved autonomously |
| **Long-Horizon Planning** | Agentic Loop testing | Success rate on 10+ step plans |
| **Reasoning Depth** | Thinking mode analysis | Logic consistency across CoT steps |
| **Long Context RAG** | Needle-in-a-Haystack (2M+) | Recall efficiency at scale |
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

### Dimension 3: Reasoning Reliability

Does the "Thinking" mode improve output accuracy vs standard generation?

| Mode | Accuracy (Math) | Accuracy (Code) | Avg Latency | Tokens / Output |
|------|-----------------|-----------------|-------------|-----------------|
| **Standard** | 72% | 68% | 1.2s | 400 |
| **Thinking** | 94% | 89% | 12.5s | 2400 |
| **Hybrid** | Variable | Variable | User-defined | Configurable |

### Reasoning Calibration

**The "Over-Thinking" Problem:**
Models often spend 2000+ "thinking" tokens on a question that could be answered with 10 tokens (e.g., "What is 2+2?").

**Principal-level Nuance:**
Evaluate models based on **Logic Efficiency**: `Accuracy / (Inference Tokens)`.
Production systems use **Model Arbitration**: a small model (Gemini 3.1 Flash, Claude Haiku 4.5, GPT-5.5-mini) detects whether a query needs "Thinking" mode. This avoids the 10x latency and cost penalty for simple queries.

---

## Internal Elo-based Evaluation

**Moving beyond static rubrics.**
Rubrics (1-5 scales) are prone to "judge fatigue" and "score drifting." Modern systems use **Pairwise Elo** for internal golden sets.

**The Workflow:**
1. **Blind Side-by-Side:** Model A and Model B generate answers for the same query.
2. **The Judge:** An "Ultra" model (Claude Opus 4.7, GPT-5.5 reasoning, or human) selects the winner.
3. **Elo Update:** Update the internal leaderboard.

```python
def update_elo(winner_elo, loser_elo, k=32):
    expected_winner = 1 / (1 + 10 ** ((loser_elo - winner_elo) / 400))
    new_winner_elo = winner_elo + k * (1 - expected_winner)
    new_loser_elo = loser_elo + k * (0 - (1 - expected_winner))
    return new_winner_elo, new_loser_elo
```

**Why it wins:** It provides a **relative** ranking that is much more robust to changes in judge personality or model versioning.

### Dimension 4: Context Recall

With 2M+ context windows, simple "needle-in-a-haystack" is no longer enough. We now measure **Contextual Reasoning** across the window.

| Metric | Measurement | Target |
|--------|-------------|--------|
| **Window Recall** | Factual recall at 90% window depth | > 98% |
| **Cross-Doc Reasoning** | Logic linking Doc A (pos 10k) to Doc B (pos 1M) | > 90% |
| **Contextual Noise Resistance** | Accuracy when 90% of window is irrelevant "filler" | > 95% |

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

**Solution:** Run multiple times with temperature > 0, report confidence intervals.

### Pitfall 5: Cost Blindness

**Problem:** Best model is 10x more expensive.

**Solution:** Always report quality-adjusted cost.

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

---

## Practical Assessment Process

### Week 1: Setup and Initial Filtering

```
Day 1-2: Define evaluation criteria and create test set
Day 3-4: Benchmark 4-6 candidate models
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
- Model A: GPT-4o
- Model B: Claude 3.5 Sonnet
- Model C: Llama 3.1 70B

### Evaluation Results

| Metric | Model A | Model B | Model C |
|--------|---------|---------|---------|
| Overall Score | 4.2/5 | 4.3/5 | 3.9/5 |
| Category 1 | ... | ... | ... |
| P50 Latency | 450ms | 520ms | 180ms |
| Cost/1K queries | $0.85 | $1.10 | $0.25 |

### Recommendation
Model B (Claude 3.5 Sonnet) for quality-critical paths
Model C (Llama 3.1 70B) for high-volume, cost-sensitive paths

### Rationale
[Detailed reasoning]
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
class ModelABTest:
    def __init__(self, model_a: str, model_b: str, traffic_split: float = 0.5):
        self.model_a = model_a
        self.model_b = model_b
        self.traffic_split = traffic_split
        self.results = {"a": [], "b": []}
    
    def route_request(self, request_id: str) -> str:
        # Deterministic routing for consistency
        hash_val = hash(request_id) % 100
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

### Metrics to Track

| Metric Type | Examples |
|-------------|----------|
| Quality | User ratings, expert review, LLM judge |
| Engagement | Click-through, time on page, follow-up queries |
| Business | Conversion, support escalation, resolution rate |
| Operational | Latency, errors, cost |

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

**Better approach:** 
- Use MMLU for initial filtering (saves time)
- Build custom evaluation for final decision
- Test on actual use case data
- Include operational metrics (latency, cost)

---

## References

- Zheng et al. "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (2023)
- LMSYS Chatbot Arena: https://chat.lmsys.org/
- HELM: https://crfm.stanford.edu/helm/
- LMSys Evaluation: https://github.com/lm-sys/FastChat/tree/main/fastchat/llm_judge
- OpenAI Evals: https://github.com/openai/evals

---

*Previous: [Model Taxonomy](01-model-taxonomy.md) | Next: [Pricing and Costs](03-pricing-and-costs.md)*
