# AI Design Patterns Quick Reference

Quick lookup for common patterns. See individual chapters for detailed implementation.

---

## Retrieval Patterns

| Pattern | Use Case | Key Tradeoff |
|---------|----------|--------------|
| **Basic RAG** | Simple Q&A over documents | Easy to implement, limited accuracy |
| **Hybrid Search** | Combining semantic + keyword | Better recall, more complexity |
| **Reranking** | High-precision retrieval | Accuracy vs latency |
| **Query Expansion** | Ambiguous queries | Better recall, more tokens |
| **HyDE** | No direct matches expected | Creative, but can hallucinate |
| **Parent-Child Chunking** | Need surrounding context | Memory overhead |

```
Query → Embed → Vector Search → Rerank → Top-K → Generate
              ↓
         BM25 Search ─────────┘ (hybrid)
```

---

## Generation Patterns

| Pattern | Use Case | Key Tradeoff |
|---------|----------|--------------|
| **Zero-Shot** | Simple tasks | Fast, less reliable |
| **Few-Shot** | Need format control | Token cost |
| **Chain-of-Thought** | Reasoning tasks | Latency, shows work |
| **Self-Consistency** | High-stakes answers | 3-5x cost |
| **Structured Output** | API responses | Constrained creativity |

---

## Agent Patterns

| Pattern | Use Case | Complexity |
|---------|----------|------------|
| **ReAct** | Tool-using agents | Medium |
| **Plan-and-Execute** | Multi-step tasks | High |
| **Multi-Agent Debate** | Verification | High |
| **Human-in-the-Loop** | High-stakes actions | Medium |
| **Swarm / Handoff** | Specialized sub-agents | High |
| **Advisor / Executor** | Cheap model runs the loop, strong model consulted at decision points | Medium |
| **Orchestrator + isolated subagents** | Parallel work without context contention | High |
| **Managed harness (rent the loop)** | Durable sessions and recovery you do not operate | Low to build; residency and retention limits decide |

**Advisor / executor**, added in 2026, is the cost-quality lever worth knowing: an inexpensive executor drives the agent loop and calls a stronger advisor model at decision points, passing the transcript and receiving a plan or correction. It is a cost claim rather than a quality claim: pairing a low-effort executor with a stronger advisor can beat the cost-quality line that executor traces by raising its own effort, while the top scores still belong to maximum effort at higher cost. Published figures are vendor-benchmarked, so measure on your own tasks. Consult rate is the metric to watch: if the executor consults on nearly every step you have bought an expensive model with extra latency.

**Orchestrator plus isolated subagents** is where the single-versus-multi-agent argument landed. Subagents get their own context and return summaries, with no peer-to-peer channel between them. Context is what is isolated, not state: managed platforms typically share one sandbox, filesystem, and credential set across the roster. Vendor platforms enforce the limits that make it work in practice, typically one level of delegation and a cap on roster size (OpenAI's Agents API defaults `max_concurrent_subagents` to 6), which is a good default even when your framework does not enforce it.

**Managed harness** options multiplied in September 2026, when OpenAI's Agents API (public beta) and Amazon Bedrock Managed Agents built on it (preview) joined Claude Managed Agents (beta). Amazon Bedrock AgentCore's managed agent harness has been generally available since June 17, 2026, with any model and an export path to Strands-based code, which answers the lock-in objection. All of them run durable sessions and crash recovery for you. The disqualifiers are usually data handling rather than features (the Agents API launched with US residency only and no zero data retention), and no managed runtime gives exactly-once semantics for your own side-effecting tools. See [Durable Execution](07-agentic-systems/11-durable-execution.md#managed-agent-runtimes-renting-the-durable-loop).

```
┌─────────────────────────────────────────┐
│              REACT LOOP                  │
│                                         │
│  Observe → Think → Act → Observe → ...  │
│              ↓                          │
│         [Tool Call]                     │
│              ↓                          │
│         [Result]                        │
└─────────────────────────────────────────┘
```

---

## Agentic Coding Patterns (2026)

| Pattern | Use Case | Key Tool |
|---------|----------|----------|
| **Scaffold → Implement → Verify** | Full feature development | Claude Code / OpenHands |
| **Read-Plan-Edit** | Refactoring existing code | Claude Code text_editor |
| **Test-Driven Agent** | High reliability code | Agent writes tests first |
| **Shadow Review** | PR quality gate | Agent reviews diff before merge |
| **AGENTS.md / CLAUDE.md Manifest** | Project context injection | AGENTS.md works across tools; Claude Code reads it when no CLAUDE.md exists |
| **Sub-Agent Parallelism** | Large codebase changes | Multiple agents per module |

```
┌────────────────────────────────────────────────────────┐
│              AGENTIC CODING LOOP                       │
│                                                        │
│  Understand → Plan → Implement → Run Tests → Fix       │
│      ↑             (bash + text_editor tools)    │     │
│      └──────────── Iterate until tests pass ────┘      │
│                                                        │
│  [AGENTS.md / CLAUDE.md injects: coding style,         │
│   test commands, forbidden patterns,                   │
│   architecture decisions]                              │
└────────────────────────────────────────────────────────┘
```

**When to use which tool:**
```
Need full autonomy + CLI → Claude Code / Codex CLI
Need open-source + any LLM → OpenHands / Cline / OpenCode
Need tight IDE integration → Cursor / Devin Desktop (formerly Windsurf)
Need reproducible pipelines → OpenHands in Docker CI
Need a harness you don't operate → OpenAI Agents API / Claude Managed Agents
```

---

## Reliability Patterns

| Pattern | Problem Solved | Implementation |
|---------|----------------|----------------|
| **Retry with Backoff** | Transient failures | Exponential backoff |
| **Circuit Breaker** | Cascading failures | Fail-fast after threshold |
| **Fallback Model** | Primary unavailable | Secondary model |
| **Timeout** | Slow responses | Cancel + fallback |
| **Bulkhead** | Resource isolation | Separate pools |
| **Refusal-Aware Routing** | Safety refusal returned as a success | Branch on `stop_reason`, not HTTP status |

```python
# Reliability stack
@circuit_breaker(failure_threshold=5)
@retry(max_attempts=3, backoff=exponential)
@timeout(seconds=30)
@fallback(model="claude-sonnet-5-5")  # a different vendor from the primary
async def generate(prompt):
    return await primary_model.generate(prompt)
```

Fallbacks have to cross vendors, because one incident can hit everything a vendor serves at once (OpenAI's September 29, 2026 incident degraded the API, ChatGPT, and Codex together for about 5 hours 20 minutes). Fallback also changes behavior, not just the endpoint: Claude thinking blocks are bound to the model that produced them and cannot be replayed into another, and a Claude safety refusal arrives as HTTP 200 with `stop_reason: "refusal"`, so a status-code check never sees it. See [Design Patterns](15-ai-design-patterns/01-design-patterns.md#pattern-retry-with-fallback).

---

## Caching Patterns

| Pattern | Hit Rate | Use Case |
|---------|----------|----------|
| **Exact Match** | Low | Identical queries |
| **Semantic Cache** | Medium | Similar queries |
| **Prompt (Prefix) Cache** | High for agent loops | Stable prefix (system prompt, tools, history); provider cache reads bill at 0.025x to 0.1x input, and self-hosted engines (vLLM, SGLang) reuse the KV cache directly |
| **Response Cache** | Varies | Deterministic outputs |

---

## Security Patterns

| Pattern | Threat | Implementation |
|---------|--------|----------------|
| **Input Validation** | Prompt injection | Sanitize, detect |
| **Output Filtering** | Data leakage | PII detection, blocklists |
| **Tenant Isolation** | Cross-tenant access | Filter at query time |
| **Rate Limiting** | Abuse | Per-user/tenant limits |
| **Egress Allowlist** | Exfiltration, eval breakouts | Default-deny network, including DNS, package proxies, and publish endpoints |
| **Bound Approvals** | Approval laundering | Approval hashed to the exact action and re-verified at execution |
| **Skill Pinning** | Silently changed instructions | Verify skill manifest digests; re-approve on any change |
| **Harness Isolation** | Repo config running code outside the sandbox (GitSpawn) | Run the agent harness itself inside the sandbox; treat `.git/config` and agent config files as code |
| **Monitor-and-Kill** | An alert nobody acts on in time | Wire the monitor to an automatic stop and test it like a failover; an OpenAI training run alarmed after about 12 minutes but was killed about 2.5 hours later when its auto-stop failed |

```
Input → Validate → Sanitize → LLM → Filter → Validate → Output
```

---

## Evaluation Patterns

| Pattern | Use Case | Metrics |
|---------|----------|---------|
| **Golden Set** | Regression testing | Pass rate |
| **LLM-as-Judge** | Quality scoring | Binary pass/fail per criterion, calibrated against human labels |
| **Human Eval** | Ground truth | Agreement rate |
| **A/B Testing** | Production comparison | User metrics |

---

## Cost Optimization Patterns

| Pattern | Savings | Tradeoff |
|---------|---------|----------|
| **Model Routing** | 50-70% | Complexity |
| **Prompt (Prefix) Caching** | 90-97.5% on cached input tokens | Write premium (1.25x; 2x for Anthropic's 1-hour cache); TTL management |
| **Response / Semantic Caching** | 20-40% | Staleness |
| **Prompt Compression** | 10-30% | Quality risk |
| **Batch / Flex Tier** | 50% | Latency (async or best-effort) |

```
Query → Classify → Route → [Small Model] or [Large Model]
                      ↓
              [Cheap: 80%]  [Expensive: 20%]
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| **Context Stuffing** | Token waste | Retrieve relevant only |
| **Retry Forever** | Resource exhaustion | Circuit breaker |
| **Trust All Output** | Hallucination | Verify, ground |
| **Single Model** | Single point of failure | Multi-provider |
| **No Observability** | Blind debugging | Trace everything |
| **Infinite Agentic Loop** | Agent spins without progress | Max turns + Critic agent |
| **Over-trusting Computer-Use** | Agent clicks wrong UI elements | Screenshot validation + HITL |
| **No AGENTS.md / Manifest** | Agent lacks project context | Always provide coding manifest |
| **Max Effort Everywhere** | 3-10x cost with no benefit | Pin effort per route; thinking can no longer be disabled on some models (Opus 5.5, Sonnet 5.5), so effort is the remaining lever |
| **Floating Model Defaults** | Silent behavior change when a tool or SDK swaps its default | Pin model ID and effort; re-run evals on every change |

---

## Pattern Selection Guide

**Starting a new project?**
1. Begin with Basic RAG
2. Add reranking when precision matters
3. Add hybrid search for keyword-heavy content

**Need reliability?**
1. Start with retry + timeout
2. Add circuit breaker for external calls
3. Add fallback models on a different vendor for critical paths

**Cost concerns?**
1. Turn on provider prompt caching for stable prefixes, then add semantic caching for repeated questions
2. Add model routing for query complexity, and pin reasoning effort per route
3. Batch where latency allows

---

*See [15-ai-design-patterns/](15-ai-design-patterns/) for detailed implementations*
