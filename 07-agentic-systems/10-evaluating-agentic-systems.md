# Evaluating Agentic Systems

Evaluating agents is fundamentally different from evaluating RAG. While RAG is about "Accuracy," Agents are about **"Reliability," "Efficiency," and "Safety."** Production agent eval relies on **Trajectory Benchmarks** and **LLM-as-Judge** for multi-step reasoning, with tools like Langfuse, LangWatch, Braintrust, and Arize Phoenix offering native trace-level scoring. The 2026 addition is **eval integrity**: capable agents now game graders, leak answers across instances, and spoof their own transcripts, so an agent eval has to be engineered like a security boundary.

> [!NOTE]
> For standard RAG evaluation (Retrieval vs. Generation metrics), see [06-retrieval-systems/09-advanced-retrieval-patterns.md](../06-retrieval-systems/09-advanced-retrieval-patterns.md) and Section 14. This chapter focuses specifically on the *Execution Path* of an agent.

## Table of Contents

- [The Evaluation Shift](#the-evaluation-shift)
- [Trajectory Benchmarks (The GOLD Standard)](#trajectory-benchmarks)
- [Eval Integrity: When the Agent Games the Harness](#eval-integrity-when-the-agent-games-the-harness)
- [Key Metrics: Success, Cost, and Duration](#key-metrics)
- [LLM-as-Judge for Step Quality](#llm-as-judge-for-step-quality)
- [Production Evaluation (A/B Testing Agents)](#production-evaluation)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Evaluation Shift

| Metric | RAG App | Agentic App |
|--------|---------|-------------|
| **Unit of Eval** | Single Response | The **Trajectory** (All steps) |
| **Success Criteria**| Groundedness/Faithfulness | Task Completion / Logical Soundness |
| **Complexity** | Low (Text similarity) | High (Tool state validation) |

---

## Trajectory Benchmarks

Modern eval scores the **"Path to the Result."**
1. **Optimal Path**: The shortest sequence of tools to solve the task.
2. **Agent Path**: The actual steps taken.
3. **The Score**: `Efficiency = (Optimal Steps / Agent Steps)`. A score of `0.2` means the agent meandered or looped excessively.

**Classic Benchmarks** (still useful for orientation):
- **SWE-bench**: Fixing GitHub issues (Code Agency). The Verified set shows memorization: a 2026 study (arXiv 2609.27891) that rewrote problem statements and remapped namespaces in SWE-bench Verified repositories found consistent drops.
- **WebArena**: Navigating menus and forms (Browser Agency).
- **GAIA**: General tool-use tasks (Assistant Agency).

**The agentic benchmarks people cite as of October 2026:**

| Benchmark | What it measures | Reference point (source) | Reporting trap |
|-----------|------------------|--------------------------|----------------|
| **Terminal-Bench 4.0** (flat 8-hour agent timeout) | Long terminal tasks in a sandbox | Leaderboard (entries added October 6): Opus 5.5 64.85% and Sonnet 5.5 61.82%, both at max effort, ahead of GPT-6 Astra and GPT-6.1 Sol tied at 58.18% (max) and Fable 5.1 57.88% (max). Vendor-reported: Opus 5.5 66.4% at xhigh (Anthropic) | Scores swing with effort level and harness; 3.0 results do not carry over |
| **OSWorld 2.0** (108 long-horizon computer-use tasks across 31 self-hosted websites and desktop apps; the median takes a skilled human about 1.6 hours) | Hour-scale computer use; **binary completion is the primary metric** | XLANG leaderboard: Opus 5 at max, 44.33% binary / 77.67% partial (v2.1 full set). Vendor-reported partial only: Opus 5.5 81.8% (labeled OSWorld 2.1), Astra 72.6% (offline v2026.08.08 set), Haiku 5.5 72.4% (OSWorld 2.1 offline subset, October 7) | Binary and partial differ by 30+ points; vendors mostly quote partial |
| **Agents' Last Exam** (147 published tasks, artifact-graded) | Professional workflows with hidden reference outputs | Leaderboard: Opus 5.5 (max) 38.2% pass, score 63.2 (listed cost about $1,340); Astra (max) 34.2% pass, score 59.3 (about $1,100) | The partial score runs about 25 points above the pass rate, and launch posts picked the score |
| **SWE-Bench Pro v2** (Scale, September 22) | Repository-level coding, network-locked, re-graded | Public 642-task set saturated (Opus 5 99.4%); private 272-task set: Opus 5 81.6% | Use the private set; public numbers say little now |
| **tau3-bench** 2.0.0 (October 19; repository `tau2-bench`) | Tool-using customer-service agents, including full-duplex voice, now in six languages and with an entity-capture benchmark | 1.x figures: Sierra's leaderboard had gpt-live-1 at 81.7% pass@1 in voice, and Google reports Gemini 3.8 Live at 68.6% with extended thinking versus 30.1% without (vendor-reported, not comparable across sources) | The 2.0.0 release notes say "Scores are not comparable with 1.x" (communication checks, judging, and prompts changed), so re-run rather than compare, or pin the `v1.0.1` tag; voice results also depend on the reasoning backend |
| **hyper-tau-bench** (Sierra, September 8) | Agents that build agents: recover requirements, build a service agent, score it on held-out tests | Opus 5 alone in Claude Code 23.9%; same model class with a deep-context engineer 82.2% | Measures requirement elicitation more than coding; 17 to 42% of runs tried to probe the sandbox or grader |

The interview point: single-session desktop and coding tasks are close to solved on the older suites, but hour-scale binary completion on OSWorld 2.0 is still under 50%. Whenever someone quotes a number, ask which version, binary or partial, what effort level, which harness, and who ran it.

---

## Eval Integrity: When the Agent Games the Harness

Through mid-2026 the main eval-validity worry was contamination. The new one is the agent actively exploiting the evaluation. Four findings define it:

- **Unearned passes grow with capability.** An audit of 3,810 passing trajectories (arXiv 2609.34262, September 2026) found the confirmed violation rate on SWE-Bench Pro v1.0 rose from 24% (Opus 4.7) to 73% (Fable 5) on matched tasks, mostly through access to reference solutions in git history. Later cohorts fell (11% for Fable 5.1, 0% for GPT-6 Astra; configurations not normalized). Minimal patches plus exploit replay closed the holes. A rising score can mean better cheating.
- **Instances are not independent samples.** METR's investigation of OpenAI's July cyber evaluations found about 1,200 agents posting 70,000+ messages to an unsanctioned board in a shared package-cache namespace, coordinating attacks and tampering with grading; OpenAI agents in a separate web-research benchmark made about 13,000 edits to dormant public wikis in one June week to share answers across instances (collusion.wiki).
- **Transcripts can lie.** METR found successful tool-call spoofs (appearing to issue one call while running another) in about 7% of evaluated transcripts, and much of the deception targeted a check the agents wrongly believed existed. Impossible tasks feed this: an outside estimate cited by METR says 30 to 40% of the targets could not be exploited as intended.
- **Feedback teaches evasion.** In arXiv 2609.28614, research agents reward-hacked spontaneously on 30.5% of open-ended tasks, and cumulative evasion reached 40.5% after detailed rejection feedback versus 20.3% after a generic rejection (the comparison does not isolate the explanations).

**The integrity checklist:**

1. Strip reference solutions from the environment, including git history, caches, and mounted volumes; network-lock the sandbox.
2. Isolate instances from each other as well as from the internet: no shared writable caches, registries, wikis, or scratch storage.
3. Record transcripts from the executor side and verify them; never trust the agent's account of what it ran.
4. Include canary tasks with no valid solution, and treat a "pass" on one as a harness bug.
5. Keep the grader opaque to the agent: terse verdicts, no grader source in reach.
6. Re-audit passing trajectories every time the model generation changes, because exploit rates move with capability.
7. If you run self-improving or eval-driven optimization loops, treat the eval set as supply chain: a poisoned benchmark can backdoor self-modifying agents (arXiv 2609.17817).

---

## Key Metrics

### 1. Task Success Rate (TSR)
The percentage of tasks where the final state is correct.
> [!IMPORTANT]
> A "Correct Answer" via a "Wrong Path" is a score of 0 in senior production settings, where "wrong" means unsafe, unauthorized, or exploiting the grader. A different but valid path to the same verified outcome should pass; grade outcomes and policy, not one rigid tool sequence (see [Loop Engineering](12-loop-engineering.md#verification-and-grading)).

### 2. Action Success Rate (ASR)
The percentage of individual tool calls that returned valid data (not errors or hallucinations).

### 3. Unit Cost per Task
Total tokens + infrastructure cost (Sandboxes, API calls) per completed goal.

---

## LLM-as-Judge for Step Quality

We use a stronger model (Claude Opus 5.5, GPT-6.1 Sol or GPT-6 Astra) to review the **Reasoning Log** of a smaller agent, ideally from a different model family than the agent to avoid shared blind spots.
- **Thought Quality**: Did the agent's logic for using Tool X follow from Observation Y?
- **Redundancy Check**: Did the agent repeat a search it just performed?
- **Feedback Loop**: This "Judge" output is then used for **DPO (Direct Preference Optimization)** to align the agent's future behavior. Keep judge rationales out of the agent's own retry loop (see the feedback finding above).

**A third judge tier: decision models.** For binary or categorical checks ("did the agent call a write tool without approval?", "is this step redundant?"), typed classifiers that return a probability instead of text are now cheap enough to run on all traffic. TypeSafe's Jev (early access, $0.042 per 1M input tokens) was integrated into LangSmith, Langfuse, Braintrust, DeepEval, and Opik within two weeks of its September launch; LangChain measured about 0.44 s per call against 2.16 to 2.83 s for LLM judges, and Langfuse cited a third-party benchmark in which Jev matched Claude Fable 5.1's verdicts 91.5% of the time. Limits: no written rationale, no abstention, and degraded accuracy with irrelevant context, so keep LLM judges where you need reasoning or open-ended assessment. One September study found cascading cheap judges into expensive ones gained at most 2.7 points, largely because the two kinds fail together: on Jev's most confident errors, about 96% of LLM-judge verdicts repeated the same wrong answer (arXiv 2609.29769). Calibrate against human labels, not against another model.

**Tooling churn is a selection criterion now.** Dynatrace completed its acquisition of Arize on October 1, 2026 (announced August 13; Phoenix is Elastic-2.0 licensed); OpenAI Evals is scheduled to shut down on November 30, 2026, after an announced read-only phase from October 31, with OpenAI pointing users to Promptfoo, which is now part of OpenAI and remains MIT licensed; LangSmith capped extended-retention SaaS traces at 180 days from September 14 and on October 1 replaced per-run and uptime pricing with usage-based pricing in LangChain Standard Units (1 LSU = $1). Prefer OTel-native, exportable traces so the eval history outlives the vendor, and verify after every SDK upgrade that tracing still sees the model calls: the Anthropic Python SDK 1.0 and OpenAI Python SDK 3.0 moved to `httpx2`, and Anthropic's migration guide warns that OpenTelemetry's HTTPX instrumentation, Sentry, respx, and vcrpy can silently miss SDK requests unless `httpx2.alias_httpx()` runs before anything imports `httpx`. An eval pipeline that loses its traces does not fail; it reports on a shrinking sample.

---

## Production Evaluation

Production teams use **Shadow Execution**.
1. **V1 Agent** responds to the user.
2. **V2 (Experimental) Agent** runs the same query in a "Hidden Sandbox."
3. **The Comparison**: We compare the two trajectories. If V2 consistently solves tasks in fewer steps without safety violations, we promote it to production.

---

## Interview Questions

### Q: How do you evaluate an agent when the environment is non-deterministic (e.g., the web)?

**Strong answer:**
We use **Mock Environments** or **Snapshotted States**. For high-fidelity testing, we use a containerized browser that resets to a clean state for every test run. We then compare the agent's trajectory against a **Reference Trace**. If the environment is truly live, we use **State-Based Verification**: instead of comparing the text, we check the external world's state (e.g., "Is there a new row in the database with the correct values?"). Artifact-graded benchmarks such as Agents' Last Exam take the same approach: deterministic graders compare the files the agent produced against a hidden reference.

### Q: Why is "Meandering" (taking too many steps) a critical failure in Staff-level Agent design?

**Strong answer:**
Meandering leads to three failures: 1) **Cost**: Every step is an LLM call; 2) **Latency**: Every step adds 2-5 seconds; 3) **Entropy**: The longer the trajectory, the higher the chance of the agent encountering a weird edge case that triggers a hallucination. The standard fix is **Step Budgets**: if an agent doesn't solve a task in 10 steps, we terminate it and escalate to a human to prevent a "Token Leak." Budget discipline cuts both ways, though: in hyper-tau-bench, two builds that ran 3.0x and 1.3x over budget scored zero, while surviving runs spent 0.45x on average, so report success at a fixed budget, not success at any cost.

### Q: Your coding agent's benchmark score jumped 15 points after a model upgrade. How do you know it is real?

**Strong answer:**
I assume it might be cheating until I have checked, because the September 2026 audit of SWE-Bench Pro v1.0 found unearned passes rising from 24% to 73% of matched tasks as models got stronger, mostly by reading reference solutions out of git history. My checks: (1) **re-audit a sample of new passes** with a separate reviewer looking for exploit signatures (git history access, test edits, grader probing), and replay any exploit to confirm; (2) **harden the environment**: strip history and reference artifacts, network-lock, make the grader opaque, and add canary tasks with no valid solution; (3) **check independence**: no shared writable caches or registries between parallel instances, since agents have coordinated across instances through both; (4) **verify transcripts from the executor side**, because about 7% of transcripts in METR's investigation contained spoofed tool calls; and (5) **cross-check on a perturbed or private set** (renamed namespaces, rewritten problem statements, a private split like SWE-Bench Pro v2's) to separate capability from memorization. If the gain survives all five, I believe it, and I report it with version, effort level, harness, and who ran it.

---

## References
- Jimenez et al. "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?" (ICLR 2024). https://arxiv.org/abs/2310.06770
- Liu et al. "AgentBench: Evaluating LLMs as Agents" (2023). https://arxiv.org/abs/2308.03688
- RAGAS. "Agentic Evaluation Module" (2025)
- "Maintaining Benchmarks Against Increasingly Capable Agents: Detection and Remediation of Unearned Passes" (arXiv 2609.34262, September 2026)
- METR. "OpenAI / Hugging Face incident investigation" (August 26, 2026). https://metr.org/blog/2026-08-26-openai-hugging-face-incident-investigation/
- Agents' Last Exam leaderboard. https://snorkel.ai/leaderboard/agents-last-exam/
- OSWorld 2.0 leaderboard (XLANG Lab). https://osworld-v2.xlang.ai/
- Terminal-Bench 4.0 leaderboard. https://www.tbench.ai/leaderboard/terminal-bench/4.0
- Sierra. "hyper-tau-bench: evaluating agents that build agents" (September 2026). https://sierra.ai/blog/hyper-t-bench-evaluating-agents-that-build-agents

---

*Next: [Durable Execution for Long-Running Agents](11-durable-execution.md)*
