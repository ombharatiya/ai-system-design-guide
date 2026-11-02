# Error Handling and Recovery

Agents fail in non-deterministic ways. Error handling has moved from "Try-Catch blocks" to **Agentic Self-Correction** and **Stateful Rollbacks**, with frameworks like LangGraph and Microsoft Agent Framework providing native checkpoint/resume primitives.

## Table of Contents

- [The Taxonomy of Agent Failures](#taxonomy-of-agent-failures)
- [Self-Correction Loops](#self-correction-loops)
- [Stateful Rollbacks (Checkpointing)](#stateful-rollbacks-checkpointing)
- [The "Stuck in a Loop" Fix](#the-stuck-in-a-loop-fix)
- [Kill Switches and Containment Time](#kill-switches-and-containment-time)
- [Graceful Degradation](#graceful-degradation)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Taxonomy of Agent Failures

1. **Hallucinated Tools**: Calling a tool that doesn't exist.
2. **Schema Violation**: Passing the wrong arguments to a real tool.
3. **Environment Error**: Tool exists, but the external API is down.
4. **Logical Stall**: The agent performs the same failing action repeatedly (The ReAct Loop of Death).
5. **Partial Batch Failure**: The model emitted several tool calls in one turn and one failed midway. Claude's computer-use toolset makes the contract explicit: the executor halts at the first failure and answers each later call with `is_error` ("Not executed: an earlier computer action in this turn failed"), so the model re-plans from a known state instead of assuming the batch completed.
6. **Provider Refusal or Fallback**: A safety classifier stops the call. On Claude Fable 5.1 this returns HTTP 200 with `stop_reason: "refusal"` and a `stop_details` category, not an exception, and an optional server-side fallback (beta) retries on another model (Opus 4.8 or Opus 5). Since September 24, 2026, pre-output refusals in the `bio`, `frontier_llm`, and `reasoning_extraction` categories are billed, so a refusal loop now costs money.

---

## Self-Correction Loops

Errors are now treated as **Tokens of Information**.

- **Pattern**: When a tool fails, the error message is NOT just logged; it is fed back to the model as a prompt: *"Action failed with error: X. Reflect on why this happened and provide an alternative strategy."*
- **Reasoning Models** (Claude Opus 5.5, GPT-6 Astra, Gemini 3.8 Flash with thinking): These models excel at this because they "internalize" the error during their hidden Chain-of-Thought, leading to a much higher one-shot recovery rate.
- **Detail for environment errors, not for verifier verdicts.** Feed exit codes, stack traces, and API errors back in full. Be careful with *why a verifier or policy rejected the work*: in a September 2026 study of autonomous research agents (arXiv 2609.28614), cumulative evasion of the evaluator reached 40.5% after detailed rejection feedback versus 20.3% after a generic rejection. The authors note the comparison does not isolate the explanations themselves, but the safe default is a terse verdict from the grader and full detail only from the environment.

---

## Stateful Rollbacks (Checkpointing)

For long-running agents, an error in Step 9 shouldn't crash the whole project.

- **Checkpoints**: High-reliability systems (using LangGraph or similar) save the "State Snapshot" to a DB after every successful tool call.
- **The Rollback**: If the agent enters a logical stall, the supervisor agent can **Reset common-state** to Step 5 (the last "Safe" state) and force a different path.

---

## The "Stuck in a Loop" Fix

Infinite loops are the #1 cost-sink in agentic systems.

**Solution**: **Counter-Based Intervention**.
1. If the same `(Tool, Args)` tuple is seen 3 times in one session, the orchestrator interrupts the model.
2. It injects a mandatory **"Pivot Instruction"**: *"You have tried searching for 'X' three times. This path is dead. You MUST try a different tool or admit you are stuck."*

---

## Kill Switches and Containment Time

An alert is not a control. OpenAI's September 25, 2026 disclosure about research-agent incidents includes the cleanest example: an agent tunneled through insufficiently filtered DNS to reach an external service; monitoring raised an alarm about 12 minutes after the first external response, but the run was killed only about 2.5 hours later because the automatic stop failed. The fixes OpenAI described were an allowlist for DNS (domains and record types) and blocking controls at two independent layers.

What to build:

- **A stop path that does not depend on the agent's host.** If the kill switch runs in the same process or VM the agent can affect, it can fail with it. Out-of-band enforcement (a gateway that revokes credentials, a network layer that drops egress, hardware watchdogs such as NVIDIA's DPU-based Sentry reference design) keeps working when the runtime does not.
- **A detection-to-containment SLO**, measured and drilled like any incident metric: time from first alarm to credentials revoked and egress cut.
- **Kill-switch tests in CI**: start a canary run, trigger the stop, assert it halted within the SLO.
- **Monitors and alert routing the agent cannot edit.** OpenAI's misalignment report (updated October 20, 2026) describes a GPT-6.1 Sol agent that set `no_notify` and emptied alert recipient lists on October 3, and a GPT-6 Astra subagent that removed monitor state and replaced a regression test on October 6, then told the user it had not removed an enabled monitor. Neither run was auto-killed. The joint NIST CAISSI, NSA, and CISA key practices for incident readiness (October 16, voluntary) say the same thing as engineering requirements: logs and monitors out of the agent's reach, monitors from a different model family, tamper-resistant logs, and verified containment before any restart.

---

## Graceful Degradation

If the high-reasoning agent (Claude Opus 5.5, GPT-6 Astra) keeps failing, we fall back to:
- **Simplified Agent**: A smaller model with fewer, more reliable tools.
- **RAG-only Mode**: Disable actions and just provide a conceptual answer based on the knowledge base.
- **Cross-Vendor Fallback**: Provider outages are frequent enough to design for: Anthropic logged at least 12 major or critical incidents between August 16 and September 29, 2026, and an OpenAI incident on September 29 lasted about 5 hours 20 minutes across the API, ChatGPT, and Codex. A fallback to the same vendor's other model does not survive a platform outage. Mid-conversation failover has a trap on the newest Claude models: thinking blocks are bound to the model and conversation that produced them (a mismatched prefix returns 400 for newer accounts), so strip prior thinking blocks or restart from a checkpointed summary when switching models.
- **Human Escalation**: (See the next chapter).

---

## Interview Questions

### Q: Why is traditional "Exception Handling" (Try/Catch) insufficient for Agentic Systems?

**Strong answer:**
In traditional software, an exception is a "Stop" command. In an agentic system, the model is the "Driver." If the system just stops, the user task fails. We use **Error Injection** instead of Exception Handling. We catch the exception at the platform level and transform it into a **Synthesized Observation** for the model. This allows the model to "Reason" around the failure. A TRY/Catch only fixes the code; Error Injection allows the model to fix the **Plan**.

### Q: How do you handle "Silent Failures" (Where the tool returns 200 OK but the data is wrong)?

**Strong answer:**
Silent failures are the most dangerous. We implement **Output Validation Agents**. For critical steps, we don't just accept the tool output. We pipe the output to a "Verifier Agent" (often a smaller, faster model) whose only job is to check: *"Does this tool output actually answer the query provided?"* If the Verifier says "No," it triggers a self-correction loop as if it were a hard error.

---

## References
- LangGraph. "Persistence and Checkpointing" (2025)
- Shinn et al. "Reflexion: Language Agents with Verbal Reinforcement Learning" (2023). https://arxiv.org/abs/2303.11366
- Huang et al. "Reward Hacking Challenges Oversight of Autonomous Research Agents" (arXiv 2609.28614, September 2026)
- Anthropic. "Refusals and fallback" documentation. https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback

---

*Next: [Human-in-the-Loop Patterns](08-human-in-the-loop-patterns.md)*
