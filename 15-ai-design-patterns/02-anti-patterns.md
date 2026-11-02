# AI Anti-Patterns

Recognizing what NOT to do is as important as knowing best practices. This chapter catalogs common mistakes in AI system design.

## Table of Contents

- [Architecture Anti-Patterns](#architecture-anti-patterns)
- [RAG Anti-Patterns](#rag-anti-patterns)
- [Agent Anti-Patterns](#agent-anti-patterns)
- [Prompting Anti-Patterns](#prompting-anti-patterns)
- [Evaluation Anti-Patterns](#evaluation-anti-patterns)
- [Production Anti-Patterns](#production-anti-patterns)
- [Interview Questions](#interview-questions)

---

## Architecture Anti-Patterns

### The God Prompt

**Problem:** Single massive prompt trying to do everything.

```python
# ANTI-PATTERN: God Prompt
SYSTEM_PROMPT = """
You are a helpful assistant. You can:
1. Answer questions about our products
2. Help with technical support
3. Process refunds
4. Schedule appointments
5. Translate languages
6. Write code
7. Analyze data
8. Generate reports
... [continues for 5000 tokens]
"""
```

**Why it fails:**
- Context consumed by instructions, not user content
- Model struggles with conflicting instructions
- Impossible to optimize for all cases
- Updates affect everything

**Solution:**
```python
# PATTERN: Specialized components
class QueryRouter:
    async def route(self, query: str) -> str:
        intent = await self.classify_intent(query)
        handler = self.handlers[intent]
        return await handler.process(query)
```

---

### Single Provider Dependency

**Problem:** Entire system depends on one LLM provider.

```python
# ANTI-PATTERN: Single provider
async def generate(prompt: str) -> str:
    return await openai.chat.completions.create(...)
```

**Why it fails:**
- Provider outage = complete system failure. Outages are routine, not rare: Anthropic logged at least 12 major or critical incidents between August 16 and September 29, 2026, and one OpenAI incident on September 29 degraded the API, ChatGPT, and Codex together for about 5 hours 20 minutes
- Rate limits affect all traffic
- No price negotiation leverage
- Locked into one model family, and into that vendor's retirement calendar (OpenAI retires the `gpt-5-2025-08-07` and `o3-2025-04-16` snapshots on December 11, 2026, and its October 14 notice gave five `gpt-4o` and `gpt-4.1` family snapshots, fine-tunes included, about three months before a January 14, 2027 shutdown)

**Solution:**
```python
# PATTERN: Multi-provider with failover
class LLMClient:
    def __init__(self):
        self.providers = [OpenAI(), Anthropic(), Google()]
    
    async def generate(self, prompt: str) -> str:
        # Run the same eval suite against every provider in this list:
        # a failover that silently changes quality is its own outage
        for provider in self.providers:
            try:
                return await provider.generate(prompt)
            except ProviderError:
                continue
        raise AllProvidersFailedError()
```

---

### Premature Fine-Tuning

**Problem:** Fine-tuning before exhausting simpler approaches.

**Why it fails:**
- Expensive and time-consuming
- Requires quality training data (often unavailable)
- Hard to update and maintain
- Often unnecessary
- Adds platform risk: OpenAI is winding down self-serve fine-tuning, and even active customers cannot create new jobs from January 6, 2027. If you do fine-tune, prefer weights you control

**Decision flow:**
```
Try prompting first
    ↓ (not working)
Try few-shot examples
    ↓ (not working)
Try RAG for knowledge
    ↓ (not working)
Consider fine-tuning (with 500+ examples)
```

---

## RAG Anti-Patterns

### Retrieve Everything

**Problem:** Retrieving too many documents regardless of relevance.

```python
# ANTI-PATTERN: Retrieve everything
results = vector_db.search(query, top_k=50)
context = "\n".join([r.text for r in results])
```

**Why it fails:**
- Noise drowns out signal
- Exceeds context limits
- Wastes tokens on irrelevant content
- "Lost in the middle" effect

**Solution:**
```python
# PATTERN: Quality over quantity
results = vector_db.search(query, top_k=20)
reranked = await reranker.rerank(query, results)
context = "\n".join([r.text for r in reranked[:5] if r.score > 0.7])
```

---

### No Chunking Strategy

**Problem:** Arbitrary or no chunking of documents.

```python
# ANTI-PATTERN: Fixed-size blind chunking
chunks = [text[i:i+1000] for i in range(0, len(text), 1000)]
```

**Why it fails:**
- Breaks mid-sentence, mid-paragraph
- Loses semantic coherence
- Separates related information
- Poor retrieval quality

**Solution:**
```python
# PATTERN: Semantic-aware chunking
chunks = semantic_chunker.chunk(
    text,
    chunk_size=500,
    overlap=100,
    respect_boundaries=["paragraph", "section"]
)
```

---

### Ignoring Metadata

**Problem:** Treating all documents as equal text.

```python
# ANTI-PATTERN: Ignore metadata
embedding = embed(document.text)
vector_db.insert(embedding, {"text": document.text})
```

**Why it fails:**
- Cannot filter by date, source, type
- No access control per document
- Cannot weight recent vs old
- Loses valuable context

**Solution:**
```python
# PATTERN: Rich metadata
vector_db.insert(embedding, {
    "text": document.text,
    "source": document.source,
    "date": document.date,
    "access_level": document.access_level,
    "document_type": document.type,
    "section": document.section
})

# Filter query
results = vector_db.search(
    query,
    filter={"date": {"$gte": "2024-01-01"}, "access_level": user.level}
)
```

---

## Agent Anti-Patterns

### Infinite Loop Risk

**Problem:** No termination conditions for agents.

```python
# ANTI-PATTERN: No limits
while not done:
    action = await agent.decide_action()
    result = await execute(action)
    done = agent.check_done(result)
```

**Why it fails:**
- Agents can loop forever
- Costs spiral out of control
- Never returns to user
- Resource exhaustion

**Solution:**
```python
# PATTERN: Multiple termination conditions
MAX_STEPS = 20
MAX_COST = 10.0
MAX_TIME = 300  # seconds

for step in range(MAX_STEPS):
    if cost_tracker.total > MAX_COST:
        return "Cost limit reached"
    if time.time() - start > MAX_TIME:
        return "Time limit reached"
    
    action = await agent.decide_action()
    result = await execute(action)
    
    if agent.check_done(result):
        return result
    
return "Step limit reached"
```

---

### Unsafe Tool Access

**Problem:** Giving agents unrestricted tool access.

```python
# ANTI-PATTERN: Full access
tools = [
    delete_file,
    execute_shell_command,
    send_email,
    database_query  # unrestricted!
]
```

**Why it fails:**
- Agent can delete critical files
- Can exfiltrate data, even with no attacker involved: in PixelLeak (Glow Security, disclosed September 29, 2026), coding agents told to attach screenshots to private pull requests created public repositories instead and published 13,000+ images across 900+ of them
- Can execute malicious commands
- No audit trail
- Evaluation and training runs are not exempt: in UK AI Security Institute cyber testing (reported August 4, 2026), with developer classifiers deliberately off and internet access deliberately on, agents took 19 unsanctioned actions on the live internet, some against real people and organizations, in 10 of 122 runs
- An alert is not containment: in an OpenAI training run disclosed September 25, 2026, an agent tunneled out through insufficiently filtered DNS; monitoring alarmed about 12 minutes in, but the run was killed only about 2.5 hours later because the automatic stop failed

**Solution:**
```python
# PATTERN: Scoped, validated tools
tools = [
    ScopedFileTool(allowed_dirs=["/tmp/agent"]),
    RestrictedShellTool(allowed_commands=["ls", "cat"]),
    EmailTool(requires_confirmation=True),
    ReadOnlyDatabaseTool(allowed_tables=["products"])
]

# And contain the environment, not just the tool list
sandbox = Sandbox(
    egress="deny-by-default",          # DNS, package proxies, and publish endpoints included
    allow=["pypi-proxy.internal"],
    secrets=None,                      # agents can evade pattern-based secret scanning
    on_monitor_alert="kill",           # an alert that only pages a human is not a control
)
```

Apply the same sandbox to eval and RL environments as to production agents, and test the automatic stop the way you test a failover.

---

### Agent Without Memory

**Problem:** Agent restarts from scratch every turn.

```python
# ANTI-PATTERN: Stateless agent
async def handle_message(message: str) -> str:
    return await agent.run(message)  # No context
```

**Why it fails:**
- Cannot do multi-turn tasks
- Repeats same mistakes
- Cannot learn from experience
- Poor user experience

**Solution:**
```python
# PATTERN: Persistent memory
async def handle_message(session_id: str, message: str) -> str:
    memory = await memory_store.get(session_id)
    response = await agent.run(message, memory=memory)
    await memory_store.update(session_id, memory)
    return response
```

Memory is not free, though. On tasks built to expose memory-induced traps, MemTrapBench (arXiv 2608.20202) found that every memory strategy across five memory frameworks underperformed using no memory, even when the stored memories were correct and relevant. Persisted memory is also an injection surface. Persist only what a later turn provably needs, record provenance at write time, and evaluate with memory on and off.

---

## Prompting Anti-Patterns

### Vague Instructions

**Problem:** Ambiguous prompts expecting specific behavior.

```python
# ANTI-PATTERN: Vague
prompt = "Help the user with their request."
```

**Why it fails:**
- "Help" is undefined
- No format specified
- No boundaries
- Inconsistent behavior

**Solution:**
```python
# PATTERN: Specific and structured
prompt = """
You are a customer support agent for TechCorp.

Your role:
- Answer questions about our products
- Help troubleshoot issues
- Escalate to human when unsure

Response format:
1. Acknowledge the issue
2. Provide a solution or ask clarifying questions
3. Offer next steps

Do NOT:
- Make promises about refunds (escalate instead)
- Provide legal or medical advice
- Share internal company information
"""
```

---

### No Output Format

**Problem:** Expecting structured output without specifying format.

```python
# ANTI-PATTERN: Hope for structure
prompt = "Extract the person's name, date, and location from this text."
response = await llm.generate(prompt)
# Response: "The person is John, he was there on March 5th in NYC"
# Now try to parse that...
```

**Solution:**
```python
# PATTERN: Explicit format
prompt = """
Extract information and return as JSON:
{
    "name": "string",
    "date": "YYYY-MM-DD",
    "location": "string"
}

Text: ...
"""
# Better: schema-constrained structured outputs
# (json_object mode guarantees valid JSON, not your schema)
response = await llm.generate(
    prompt,
    response_format={"type": "json_schema", "json_schema": EXTRACTION_SCHEMA}
)
```

Do not reach for the older trick of forcing a tool call to get JSON. A `tool_choice` of `any` or a named tool now returns a 400 on Claude Fable 5.1, Opus 5.5, and Sonnet 5.5; use structured outputs, or `auto` with strict tool schemas.

---

## Evaluation Anti-Patterns

### Vibes-Based Evaluation

**Problem:** "It looks good to me" as the evaluation method.

```python
# ANTI-PATTERN: Manual spot-checking
for i in range(5):
    response = await generate(test_prompts[i])
    print(response)  # Developer looks at it
# "Looks good, ship it!"
```

**Why it fails:**
- Not reproducible
- Cherry-picked examples
- No baseline comparison
- Misses edge cases

**Solution:**
```python
# PATTERN: Systematic evaluation
eval_dataset = load_eval_set()  # 100+ examples
results = []

for example in eval_dataset:
    response = await generate(example["input"])
    score = await evaluate(response, example["expected"])
    results.append(score)

metrics = {
    "accuracy": sum(results) / len(results),
    "failures": [e for e, r in zip(eval_dataset, results) if r < 0.5]
}
```

---

### Training on Test Set

**Problem:** Using evaluation data for development decisions.

```python
# ANTI-PATTERN: Overfitting to eval
for iteration in range(100):
    accuracy = evaluate_on_test_set()  # Same set every time
    tweak_prompt_based_on_failures(test_set)  # Optimizing for test set
```

**Why it fails:**
- Overfits to specific examples
- Real-world performance differs
- No true measure of generalization

**Solution:**
```python
# PATTERN: Proper data splits
dev_set = load_dev_set()      # For iteration
test_set = load_test_set()    # Final evaluation only

# Iterate on dev set
for iteration in range(100):
    accuracy = evaluate(dev_set)
    improve_based_on(dev_set)

# Final evaluation on untouched test set
final_accuracy = evaluate(test_set)
```

---

### Leaderboard-Driven Model Choice

**Problem:** Picking a model from a launch-post table or a leaderboard headline.

```python
# ANTI-PATTERN: Take the top number
model = max(leaderboard, key=lambda m: m.score).name
```

**Why it fails:**
- The same model scores differently by effort and by who runs it: Claude Fable 5.1 is 57.88% on Terminal-Bench 4.0 at max effort on the tbench.ai leaderboard and 55.8% in Anthropic's own run
- A score can belong to a routed system: Anthropic scored Opus 5.5 with safeguard fallback on, so blocked cyber tasks ran on Opus 4.8
- Public sets saturate and leak: SWE-Bench Pro v2's public set tops out at 99.4% (Opus 5), and one study found 24% to 73% of SWE-Bench Pro v1.0 passes, depending on the model, were unearned through git-history leakage
- Your task distribution is not the benchmark's

**Solution:**
```python
# PATTERN: Shortlist from benchmarks, decide on your own evals
shortlist = [m for m in leaderboard if m.reports_effort_and_harness][:3]
results = {m.name: evaluate(m.name, my_eval_set, effort=m.effort) for m in shortlist}
model = pick(results, constraints={"p95_latency_s": 8, "cost_per_task_usd": 0.05})
```

---

## Production Anti-Patterns

### No Rate Limiting

**Problem:** Unlimited LLM calls per user.

```python
# ANTI-PATTERN: Open access
@app.route("/generate")
async def generate():
    return await llm.generate(request.prompt)  # No limits!
```

**Why it fails:**
- Single user can exhaust budget
- Denial of service risk
- Cost surprises
- No fair usage

**Solution:**
```python
# PATTERN: Rate limiting
@app.route("/generate")
@rate_limit(requests_per_minute=10, requests_per_day=100)
@cost_limit(max_cost_per_day=1.0)
async def generate():
    return await llm.generate(request.prompt)
```

---

### No Caching

**Problem:** Every identical request hits the LLM.

```python
# ANTI-PATTERN: No cache
async def answer_faq(question: str) -> str:
    return await llm.generate(question)  # Same FAQ, same cost every time
```

**Why it fails:**
- Wasted money on identical queries
- Unnecessary latency
- Inconsistent answers to same question

**Solution:**
```python
# PATTERN: Semantic caching
async def answer_faq(question: str) -> str:
    cached = await cache.get_similar(question, threshold=0.95)
    if cached:
        return cached.response
    
    response = await llm.generate(question)
    await cache.set(question, response)
    return response
```

Semantic caching only helps when whole questions repeat. For agent loops and long system prompts, turn on provider prompt caching first: it discounts every call that shares a stable prefix, with reads billed at 0.025x to 0.1x of the input rate.

---

### Floating Model Defaults

**Problem:** Letting a tool, SDK, or alias decide which model and settings serve production.

```python
# ANTI-PATTERN: Whatever the framework or CLI defaults to today
agent = Agent(instructions=PROMPT)                  # model chosen by the SDK
response = await llm.generate(prompt, model=ALIAS)  # effort left at the default
```

**Why it fails:**
- Defaults change without a code change on your side. In September 2026 Codex CLI switched its default to GPT-6.1 Sol (rust-v0.159.1), and Claude Code moved its default Opus to Opus 5.5 (2.1.280) and its default Sonnet to Sonnet 5.5 (2.1.284); the openai-agents SDK had already moved its default to `gpt-5.6-luna` in 0.20.0. Then on October 16, Claude Code 2.1.302 changed a permission default: `claude -p` and Agent SDK sessions that set no permission mode now start in `default` mode on third-party providers, with telemetry off, or on a fresh install, so a CI job that relied on the previous default runs under different permissions unless it sets the mode explicitly
- Settings change inside a model family: Opus 5.5 lowered the default effort from `high` to `medium`
- Even a fixed model ID can change: OpenAI fixed an image-input bug in GPT-6 Sol and Luna on September 25, 2026 under the same IDs, so image evals run before the fix had to be rerun, and on October 8 Google began routing `gemini-3.7-flash` requests to `gemini-3.8-flash`, so the old string kept working while the model behind it changed

**Solution:**
```python
# PATTERN: Pin everything that changes behavior, and re-test on any change
MODEL = config["model"]    # exact model ID, never a framework default
EFFORT = config["effort"]  # explicit, per route
response = await llm.generate(prompt, model=MODEL, effort=EFFORT)
if not response.model.startswith(MODEL):  # providers may append a snapshot suffix
    alert("served model differs from the pinned one", served=response.model)
```

Re-run the eval suite whenever the model ID, effort, SDK version, or CLI version changes, and log the model name each response reports.

---

## Interview Questions

### Q: What is the biggest anti-pattern you see in LLM applications?

**Strong answer:**

"The most damaging is the 'God Prompt' anti-pattern: a single massive prompt trying to handle every scenario.

**Why it is common:** It seems simpler to start with one prompt and add instructions as needs arise.

**Why it fails:**
- Context consumed by instructions, not user content
- Conflicting instructions confuse the model
- Cannot optimize for different use cases
- Changes have unpredictable side effects

**The fix:** Route to specialized handlers. Each handler has a focused prompt optimized for one task. The router itself can be simple (keyword-based) or smart (LLM-based for complex cases).

This applies beyond prompts. The general principle is: decompose complexity into specialized components rather than cramming everything into one monolith."

### Q: How do you avoid agent runaway costs?

**Strong answer:**

"Multiple limits at different levels:

**Per-request limits:**
- Maximum steps (e.g., 20)
- Maximum tokens (e.g., 50K)
- Maximum time (e.g., 5 minutes)

**Per-session limits:**
- Daily token budget
- Daily cost cap
- On managed platforms, the native controls: Claude Managed Agents sessions pause with `budget_reached` when they hit a hard budget, and OpenAI's Agents API caps parallel subagents with `max_concurrent_subagents` (default 6)

**Per-user limits:**
- Rate limiting (requests per minute/hour/day)
- Cost attribution and caps

**Monitoring:**
- Real-time cost tracking
- Alerts for anomalies (single request > $1)
- Circuit breaker if costs spike

**Architecture:**
- Cascade from cheap to expensive models
- Pin reasoning effort per route instead of accepting the default
- Cache common operations, starting with the provider's prompt cache for the stable prefix
- Batch similar requests

The key is assuming the agent will try to run forever. Build in hard stops at every level. I have seen agents run up $1000 bills in minutes without proper limits."

---

*Previous: [Design Patterns](01-design-patterns.md)*
