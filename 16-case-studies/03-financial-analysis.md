# Case Study: Financial Analysis with Ensemble Verification

This case study covers designing a high-reliability AI system for generating equity research reports where accuracy is critical.

## Table of Contents

- [Problem Statement](#problem-statement)
- [Requirements Analysis](#requirements-analysis)
- [Architecture Design](#architecture-design)
- [Ensemble Pipeline](#ensemble-pipeline)
- [Fact Verification](#stage-3b-fact-verification-with-multi-agent-debate)
- [Quality Gates](#quality-gates)
- [Results and Metrics](#results-and-metrics)
- [Interview Walkthrough](#interview-walkthrough)

---

## Problem Statement

**Company:** Investment firm generating equity research reports

**Challenge:**
- Reports influence multi-million dollar investment decisions
- Zero tolerance for hallucinated financial data
- Regulatory scrutiny on AI-generated analysis
- Current manual process: 8 hours per report, $500 cost

**Goal:**
- Reduce report generation time to < 30 minutes
- Maintain accuracy at 99.5%+
- Clear audit trail for compliance
- Cost target: < $50 per report

---

## Requirements Analysis

### Accuracy Requirements

| Data Type | Tolerance | Verification Method |
|-----------|-----------|---------------------|
| Financial metrics (EPS, PE) | 0% error | Source verification |
| Percentage changes | ±0.1% | Cross-validation |
| Date references | 100% accuracy | Source extraction |
| Company names | 100% accuracy | Entity matching |
| Analyst quotes | Verbatim or flagged | Quote extraction |

### Compliance Requirements

- All claims must cite source documents
- No forward-looking statements without disclaimers
- Clear AI-generated disclosure
- Full audit trail of generation process
- Human review for publication

---

## Architecture Design

### High-Level Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│               FINANCIAL ANALYSIS PIPELINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Stage 1: Data Extraction (Self-Consistency k=5)                │
│  └── Extract key metrics from filings with majority vote        │
│                                                                  │
│  Stage 2: Analysis Generation (Mixture of Agents)               │
│  ├── Model A: Quantitative analysis focus                       │
│  ├── Model B: Qualitative/narrative focus                       │
│  ├── Model C: Risk factor analysis                              │
│  └── Aggregator: Synthesize into coherent report                │
│                                                                  │
│  Stage 3: Fact Verification (Multi-Agent Debate)                │
│  └── 3 models debate each factual claim, flag disagreements     │
│                                                                  │
│  Stage 4: Final Review (Panel of Judges)                        │
│  └── Quality score determines auto-publish vs human review      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

The pipeline as a flow. Each stage uses a different model class on purpose: extraction wants multimodal (charts and tables), generation wants narrative quality, audit wants reasoning depth, panel wants cheap-but-many for diversity:

```mermaid
flowchart LR
    S1[Stage 1: Extraction<br/>Gemini 3.8 Flash<br/>Self-Consistency k=5] --> S2
    S2[Stage 2: Analysis<br/>Mixture of Agents<br/>Quant + Narrative + Risk] --> S3
    S3[Stage 3: Verification<br/>Multi-Agent Debate<br/>3 models per claim] --> S4
    S4[Stage 4: Final Review<br/>Panel of Judges<br/>Quality score] --> D{Auto-publish<br/>threshold met}
    D -->|yes| P[Publish]
    D -->|no| H[Human Review Queue]
```

### Data Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   10-K/Q    │     │  Earnings   │     │  Analyst    │
│   Filings   │     │  Calls      │     │  Reports    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                           ▼
                   ┌───────────────┐
                   │     Data      │
                   │   Ingestion   │
                   └───────┬───────┘
                           │
                           ▼
                   ┌───────────────┐
                   │   Extraction  │
                   │  (k=5 SC)     │
                   └───────┬───────┘
                           │
                           ▼
              ┌────────────┴────────────┐
              │    Structured Data      │
              │    (verified metrics)   │
              └────────────┬────────────┘
                           │
                           ▼
                   ┌───────────────┐
                   │    MoA        │
                   │  Generation   │
                   └───────┬───────┘
                           │
                           ▼
                   ┌───────────────┐
                   │    Debate     │
                   │  Verification │
                   └───────┬───────┘
                           │
                           ▼
                   ┌───────────────┐
                   │    Panel      │
                   │    Review     │
                   └───────┬───────┘
                           │
               ┌───────────┴───────────┐
               ▼                       ▼
        ┌─────────────┐         ┌─────────────┐
        │ Auto-Publish│         │Human Review │
        │ (high conf) │         │ (low conf)  │
        └─────────────┘         └─────────────┘
```

The data lineage in Mermaid, showing how three input sources converge into one verified output:

```mermaid
flowchart TD
    F1[10-K and 10-Q Filings] --> ING[Data Ingestion]
    F2[Earnings Calls] --> ING
    F3[Analyst Reports] --> ING
    ING --> EX[Extraction<br/>k=5 Self-Consistency]
    EX --> SD[(Structured Data<br/>verified metrics)]
    SD --> MOA[MoA Generation<br/>3 specialized agents]
    MOA --> DEB[Debate Verification<br/>flag disagreements]
    DEB --> PAN[Panel Review<br/>quality score]
    PAN --> AP[Auto-Publish<br/>high confidence]
    PAN --> HR[Human Review<br/>low confidence]
```

---

## Ensemble Pipeline

### Stage 1: Multimodal Data Extraction (Gemini 3.8 Flash)

```python
from google import genai
from google.genai import types

class FinancialDataExtractor:
    """
    Gemini 3.8 Flash (GA) reads 10-K tables and charts as page images plus text.
    Gemini 3.1 Pro is still a preview model, and preview models carry weaker
    lifecycle guarantees than a compliance pipeline should depend on.
    """
    def __init__(self):
        self.client = genai.Client()

    async def extract_metrics(self, doc_pages: list[bytes]) -> dict:
        response = await self.client.aio.models.generate_content(
            model="gemini-3.8-flash",
            contents=[
                "Extract all balance sheet items into JSON.",
                *[types.Part.from_bytes(data=p, mime_type="image/png") for p in doc_pages],
            ],
            config=types.GenerateContentConfig(
                response_mime_type="application/json",
                response_schema=BalanceSheet,  # Pydantic model: fixed field names and types
            ),
        )
        return json.loads(response.text)
```

### Stage 2: Analysis Generation (Claude Opus 5.5)

```python
class AnalysisEngine:
    """
    Claude Opus 5.5 for qualitative synthesis and narrative coherence.
    """
    async def generate_report(self, data: dict) -> str:
        response = await self.anthropic.messages.create(
            model="claude-opus-5-5",
            max_tokens=16000,
            output_config={"effort": "high"},  # Opus 5.5 defaults to medium; set it explicitly
            messages=[{"role": "user", "content": f"Analyze: {data}"}],
        )
        return "".join(b.text for b in response.content if b.type == "text")
```

### Stage 3a: Reasoning Audit (GPT-6.1 Sol)

```python
class AuditorAgent:
    """
    GPT-6.1 Sol at high reasoning effort audits claims for subtle accounting
    contradictions. A different model family from the generator is less
    likely to share the generator's blind spots.
    """
    async def audit_claim(self, claim: str, raw_data: str) -> dict:
        response = await self.openai.responses.create(
            model="gpt-6.1-sol",
            reasoning={"effort": "high"},
            input=f"Find any contradiction between the claim and the data.\n"
                  f"Claim: {claim}\nData: {raw_data}",
        )
        return self.parse_audit(response.output_text)
```

### Stage 3b: Fact Verification with Multi-Agent Debate

The debate stage is what catches the subtle hallucinations a single model misses. Three independent debaters verify each claim in parallel; consensus wins, dissent flags the claim for human review:

```mermaid
sequenceDiagram
    participant CE as Claim Extractor
    participant D1 as Debater A<br/>Claude Opus 5.5
    participant D2 as Debater B<br/>GPT-6.1 Sol
    participant D3 as Debater C<br/>Gemini 3.8 Flash
    participant CON as Consensus Logic
    participant OUT as Verification Result

    CE->>CE: extract factual claims<br/>from report
    Note over CE,D3: For each claim, debaters verify independently
    par Independent verification
        CE->>D1: claim + source docs
        D1-->>CON: verdict (supported/inferred/unsupported/contradicted)
    and
        CE->>D2: claim + source docs
        D2-->>CON: verdict
    and
        CE->>D3: claim + source docs
        D3-->>CON: verdict
    end
    CON->>CON: check consensus
    alt all agree supported
        CON->>OUT: verified
    else any contradiction
        CON->>OUT: flagged for human review
    else split verdicts
        CON->>OUT: low confidence
    end
```

```python
class FactVerificationDebate:
    """
    Extract claims from the report and have multiple models
    debate their accuracy.
    """
    
    def __init__(self, debaters: list, rounds: int = 2):
        self.debaters = debaters
        self.rounds = rounds
        self.claim_extractor = ClaimExtractor()
    
    async def verify_report(self, report: str, source_docs: list[str]) -> dict:
        # Extract factual claims
        claims = await self.claim_extractor.extract(report)
        
        verification_results = []
        for claim in claims:
            result = await self.debate_claim(claim, source_docs)
            verification_results.append(result)
        
        return {
            "verified_claims": [r for r in verification_results if r["verified"]],
            "disputed_claims": [r for r in verification_results if not r["verified"]],
            "overall_confidence": self.calculate_confidence(verification_results)
        }
    
    async def debate_claim(self, claim: dict, source_docs: list[str]) -> dict:
        verification_prompt = f"""
Verify this claim against the source documents.

Claim: {claim['text']}

Source documents:
{self.format_sources(source_docs)}

Is this claim:
1. Supported: Explicitly stated in sources
2. Inferred: Reasonably derived from sources
3. Unsupported: Not found in sources
4. Contradicted: Conflicts with sources

Provide your verdict with evidence.
"""
        
        # Each debater verifies independently
        verdicts = await asyncio.gather(*[
            debater.generate(verification_prompt)
            for debater in self.debaters
        ])
        
        # Check consensus
        parsed_verdicts = [self.parse_verdict(v) for v in verdicts]
        consensus = self.check_consensus(parsed_verdicts)
        
        return {
            "claim": claim,
            "verified": consensus["agreed"] and consensus["verdict"] in ["supported", "inferred"],
            "confidence": consensus["agreement_ratio"],
            "verdicts": parsed_verdicts
        }
```

---

## Quality Gates

### Automated Quality Checks

```python
class QualityGate:
    def __init__(self):
        self.thresholds = {
            "claim_verification_rate": 0.95,  # 95% claims verified
            "data_accuracy": 0.99,            # 99% metrics accurate
            "panel_score": 4.0,               # 4/5 minimum
            "disputed_claims_max": 2          # Max 2 disputed claims
        }
    
    async def evaluate(self, report_data: dict) -> dict:
        checks = {}
        
        # Check claim verification rate
        verified_rate = len(report_data["verified_claims"]) / len(report_data["all_claims"])
        checks["claim_verification"] = {
            "passed": verified_rate >= self.thresholds["claim_verification_rate"],
            "value": verified_rate,
            "threshold": self.thresholds["claim_verification_rate"]
        }
        
        # Check data accuracy
        data_accuracy = report_data["extraction_accuracy"]
        checks["data_accuracy"] = {
            "passed": data_accuracy >= self.thresholds["data_accuracy"],
            "value": data_accuracy,
            "threshold": self.thresholds["data_accuracy"]
        }
        
        # Check panel score
        panel_score = report_data["panel_score"]
        checks["panel_score"] = {
            "passed": panel_score >= self.thresholds["panel_score"],
            "value": panel_score,
            "threshold": self.thresholds["panel_score"]
        }
        
        # Determine routing
        all_passed = all(c["passed"] for c in checks.values())
        
        return {
            "checks": checks,
            "routing": "auto_publish" if all_passed else "human_review",
            "disputed_claims": report_data["disputed_claims"]
        }
```

### Human Review Interface

```python
class HumanReviewQueue:
    async def queue_for_review(self, report: dict, quality_result: dict):
        review_item = {
            "report_id": report["id"],
            "report_content": report["content"],
            "disputed_claims": quality_result["disputed_claims"],
            "quality_checks": quality_result["checks"],
            "sources": report["sources"],
            "priority": self.calculate_priority(quality_result),
            "queued_at": datetime.now()
        }
        
        await self.review_queue.enqueue(review_item)
        
        # Notify reviewers
        await self.notify_reviewers(review_item)
```

---

## Results and Metrics

### Performance Comparison

| Metric | Manual Process | AI Pipeline | Improvement |
|--------|---------------|-------------|-------------|
| Time per report | 8 hours | 25 minutes | 19x faster |
| Cost per report | $500 | $41 | 92% reduction |
| Factual error rate | 2.1% | 0.4% | 81% reduction |
| Human review load | 100% | 28% | 72% reduction |

### Quality Metrics

| Quality Dimension | Target | Achieved |
|-------------------|--------|----------|
| Data extraction accuracy | 99% | 99.3% |
| Claim verification rate | 95% | 96.8% |
| Panel quality score | 4.0/5.0 | 4.2/5.0 |
| Regulatory compliance | 100% | 100% |

### Cost Breakdown (November 2026 List Prices)

Assumes ~200K tokens of source material per report (10-K, two 10-Qs, an earnings call transcript) and ~150 extracted claims. Gemini 3.8 Flash is priced at its January 1, 2027 list rate ($1.50 / $7.50), not the introductory rate.

| Component | Assumption | Cost | Share |
|-----------|------------|------|-------|
| Extraction (Gemini 3.8 Flash, k=5) | 5 × 200K tokens in, ~75K out in total | $2.06 | 5% |
| Analysis (Claude Opus 5.5, Mixture of Agents) | 3 proposers × 220K in plus an aggregator; ~48K out including thinking | $3.78 | 9% |
| Reasoning audit (GPT-6.1 Sol, high effort) | ~300K in across per-claim calls (each well under the 272K long-context threshold), ~50K out | $1.10 | 3% |
| Debate verification (3 model families) | 150 claims × 3 debaters × 2 rounds × ~8K in / 1K out | $29.25 | 70% |
| Panel review | 3 mid-tier judges × ~40K in / 2K out | $0.28 | 1% |
| Infrastructure and vector ops | | $5.00 | 12% |
| **Total** | | **~$41** | 100% |

*Note: Generation is cheap; verification is the bill. Debate fan-out (claims × debaters × rounds) is about 70% of spend, and Opus 5.5 alone accounts for $15.60 of it. The lever is the claim count: deduplicate claims, and skip debate for numbers that already match the Stage 1 structured data exactly. Prompt caching the source excerpts between debate rounds cuts the input side further (caches are per model, so each debater keeps its own).*

---

## Interview Walkthrough

**Interviewer:** "Design an AI system for generating financial research reports with very high accuracy requirements."

**Strong response:**

1. **Clarify accuracy requirements** (1 min)
   - "What's the acceptable error rate for financial data?"
   - "What's the regulatory compliance requirement?"
   - "Is latency or accuracy the priority?"

2. **Acknowledge the core challenge** (1 min)
   - "The key challenge is that hallucinations are unacceptable for financial data. A single wrong number could mislead investment decisions. I need ensemble methods for reliability."

3. **High-level architecture** (3 min)
   - "I would use a multi-stage pipeline with different ensemble techniques at each stage:"
   - "Data extraction: Self-consistency with k=5 for unanimous agreement on numbers"
   - "Analysis: Mixture of Agents for diverse perspectives"
   - "Verification: Multi-agent debate to catch hallucinations"
   - "Quality gate: Panel of judges to score before publishing"

4. **Deep dive on fact verification** (3 min)
   - "For fact verification, I extract every factual claim from the report"
   - "Three diverse models debate whether each claim is supported by sources"
   - "If they disagree, the claim is flagged for human review"
   - "This catches subtle errors that single-model verification misses"

5. **Cost-quality tradeoff** (2 min)
   - "This pipeline is 10-20x more expensive than single-model generation"
   - "But for financial reports, the cost of errors (legal, reputational) far exceeds the cost of verification"
   - "I would implement confidence-based routing: auto-publish high-confidence reports, human-review low-confidence ones"

6. **Monitoring** (1 min)
   - "I would track extraction accuracy, claim verification rate, and panel scores continuously"
   - "Drift detection would alert if accuracy drops"
   - "Full audit trail for compliance"

---

## Key Learnings

1. **Self-consistency alone is insufficient** for numerical data extraction. Unanimous agreement (k/k votes) should be required.

2. **Multi-agent debate most effective** for catching subtle reasoning errors and hallucinations.

3. **Source attribution is critical** for both accuracy and compliance. Every claim must link to source documents.

4. **Confidence-based routing** is essential for cost management. Not every report needs full ensemble verification.

5. **Human-in-the-loop is still necessary** for disputed claims and edge cases. Design for graceful escalation.

---

## References

- Verga et al. "Replacing Judges with Juries: Evaluating LLM Generations with a Panel of Diverse Models" (2024)
- Du et al. "Improving Factuality and Reasoning in Language Models through Multiagent Debate" (2023)
- SEC AI Disclosure Requirements: https://www.sec.gov/

---

*Next: [Code Assistant Case Study](04-code-assistant.md)*
