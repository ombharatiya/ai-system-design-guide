# LangSmith Observability

In 2023, LLM observability was "logging strings." Now it is **Full Trajectory Debugging** and **Automated Evaluation Pipelines**, and in late 2026 the platforms are closing the loop from trace to eval to fine-tune. LangSmith is the LangChain-native option in a crowded "LLMOps" layer that also includes Langfuse (part of ClickHouse since January 2026; Python SDK 5.2.0, Nov 5, 2026, a new major since Oct 14 that needs Python 3.11+ and Langfuse server v4.59.0 or newer, so a plain `pip install langfuse` breaks v4-era samples and self-hosted v3 servers), LangWatch, Braintrust, and Arize Phoenix (licensed Elastic-2.0; Dynatrace completed its $915M acquisition of Arize on October 1, 2026). Phoenix 21.0.0 (Oct 30) turns authentication on by default, so a self-hosted upgrade needs auth configured first. Ownership changes are now a selection criterion: prefer tools that ingest OpenTelemetry and let you export your traces and datasets.

## Table of Contents

- [The Observability Pyramid](#the-observability-pyramid)
- [Tracing and Trajectories](#tracing-and-trajectories)
- [Unit Testing for LLMs (Datasets)](#unit-testing-for-llms-datasets)
- [Automated Evaluators (LLM-as-Judge)](#automated-evaluators)
- [Managing Deployment: A/B Testing](#ab-testing)
- [Platform Changes, September to October 2026](#platform-changes-september-to-october-2026)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Observability Pyramid

1. **Top (Value)**: Is the user task getting completed? (Success Rate)
2. **Middle (Flow)**: Which agent node is the bottleneck? (Latency/Cost per node)
3. **Bottom (Raw)**: What were the exact prompt/completion pairs? (Traces)

---

## Tracing and Trajectories

LangSmith automatically captures every node in a **LangGraph** or **Chain**.
- **Metadata Tagging**: Tag every trace with `user_id`, `model_tier`, and `is_canary`.
- **The Debugger**: You can "play back" a trace in the LangSmith UI, modifying the prompt and seeing how the response changes. This works without re-running the entire application.
- **Trajectories view**: a session-level view (September 2026) that shows a whole multi-turn agent run rather than one trace at a time.
- **Transport caveat**: OpenAI Python SDK 3.0 (Aug 12, 2026) and Anthropic Python SDK 1.0 (Aug 20, 2026) moved to `httpx2`. Instrumentation that hooks `httpx` (OpenTelemetry's HTTPX instrumentor, Sentry, `respx`, `vcrpy`) can silently miss SDK calls unless `httpx2.alias_httpx()` runs before anything imports `httpx`, per Anthropic's migration guide. After an SDK upgrade, check that spans still arrive before you trust a quiet dashboard.

---

## Unit Testing for LLMs (Datasets)

Building an LLM app without a **Dataset** is "vibe-based development."
- **Gold Standard Datasets**: A collection of `(Input, Expected_Output)` pairs.
- **Standard workflow**: Whenever a user provides negative feedback, that interaction is automatically pumped into a "Correction Dataset" for future testing.

---

## Automated Evaluators

You cannot manually check 1,000 log entries every morning.
- **LLM-as-Judge**: Using a strong model, ideally from a different family than the one under test (for example Claude Opus 5.5 judging a GPT-6.1 Sol agent), to score the production model on categories like **Tone**, **Accuracy**, and **Safe Action execution**.
- **Decision-model judges**: A third tier arrived in September 2026. TypeSafe's Jev (early access, $0.042 per 1M input tokens, output unmetered) returns a typed yes/no, choice or score with a probability instead of generated text; LangSmith added it as an evaluator on Sep 21. LangChain measured about $0.00035 per call and 0.44 s average latency against 2.16 to 2.83 s for LLM judges (vendor-run measurement). The catch: a paired study (arXiv 2609.29769) found that on Jev's most confident errors about 96% of LLM-judge verdicts repeated the same wrong answer, and judge cascades gained at most 2.7 points. Use decision models for cheap binary checks on 100% of traffic, keep LLM judges where you need a written rationale, and keep a human-labeled gold set because escalating between judges does not catch correlated errors.
- **Custom Evaluators**: Python functions that check for regex patterns, JSON schema validity, or Toxicity scores.

---

## A/B Testing

LangSmith supports **Experiment Comparison** offline and variant tagging online. The traffic split and the rollback live in your deployment layer (feature flags or the gateway), not in the tracing tool:
- Run 2% of traffic on a new "System Prompt" version, tagging each trace with the variant.
- Compare the **Success Rate** and **Token Cost** per variant in near real time, using online evaluators on sampled traces.
- Wire an alert on the failure-rate threshold to the flag system so it rolls back automatically.

---

## Platform Changes, September to October 2026

| Change | Date | Design consequence |
|---|---|---|
| **Usage-based pricing** in LangChain Standard Units (1 LSU = $1) replaced per-run and uptime pricing | Oct 1, 2026 | Deployment compute and traces are both priced in LSUs (the enterprise price list shows observability and evaluations at 0.0075 LSU per trace), so trace volume is a budget line. Sample high-volume, low-value traces, keep full tracing for errors and canaries, and re-run the cost model before turning on online evaluators across all traffic |
| SaaS traces with extended retention kept **at most 180 days** | From Sep 14, 2026 | A SaaS tracing store is not an audit log. If compliance needs longer retention, export to your own store or run self-hosted or BYOC (unchanged) |
| **LangSmith Fine-Tuning** public beta (`smithtune` CLI) | Sep 24, 2026 | Traced trajectories become SFT or LoRA runs on open-weight models through Fireworks or Baseten, then evaluate and deploy. Relevant because OpenAI is winding down self-serve fine-tuning (no new jobs for active customers from Jan 6, 2027) |
| **Engine v2** | Sep 24 to 25, 2026 | Red-team hypothesis generation, detection of error, latency and cost trends and of repetitive tool calls, and validation of proposed fixes against broader eval sets. LangChain says Engine now uses 40% fewer usage credits |
| SDK cadence | Aug to Oct 2026 | `langsmith` went from 0.11.0 (Aug 14) to 0.14.16 (Oct 29), 14 releases in October alone: pin it like any other dependency |

---

## Interview Questions

### Q: Why is "Trace Attribution" critical for Staff-level engineers?

**Strong answer:**
In complex multi-agent systems, the final output might be bad, but the error happened 10 steps ago in a "Researcher" node. Without **Trace Attribution**, you're just guessing where to fix the prompt. Attribution allows me to see the **Line of Reasoning**. I can see that the "Researcher" failed to find the right URL, which led to the "Summarizer" hallucinating. This allows for **Targeted Optimization** instead of broad "Prompt Engineering."

### Q: How do you justify the cost of an observability platform like LangSmith?

**Strong answer:**
The cost is offset by **Developer Productivity** and **Token Efficiency**, and since LangSmith moved to usage-based pricing on October 1, 2026 it scales with how much I trace, so I size it per trace rather than per seat. A single day of an engineer "guessing" why a model is failing costs significantly more than a monthly subscription. Moreover, by using LangSmith to find "Meandering" agents (those taking too many steps), I can optimize the graphs to reduce the average number of steps from 8 to 5. Because every step re-sends the growing context, that cuts input tokens by more than the 37.5% step reduction suggests, which matters when agent workloads run at hundreds of input tokens per output token. I would also budget for the judges: running an LLM judge on every trace can cost more than the observability seat, which is why cheap decision-model judges on 100% of traffic plus LLM judges on a sample is a sensible default.

---

## References
- LangChain. "LangSmith Observability" docs: https://docs.langchain.com/langsmith/home
- LangChain. LangSmith billing (usage-based pricing from Oct 1, 2026): https://docs.langchain.com/langsmith/billing
- LangChain. "LangSmith Engine, agents, fine-tuning and trajectories" (Sep 2026): https://www.langchain.com/blog/langsmith-engine-agents-fine-tuning-trajectories
- Rao and Callison-Burch. "JEV vs. LLMs as Rubric Judges: Cheaper, Faster, and Wrong in the Same Places" (arXiv 2609.29769, Sep 2026): https://arxiv.org/abs/2609.29769

---

*Next: [LlamaIndex and Data-Centric AI](04-llamaindex.md)*
