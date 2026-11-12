# Human-in-the-Loop Patterns

No agent is 100% reliable. **Human-in-the-Loop (HITL)** is the bridge that ensures safety and accuracy in high-stakes environments. Production stacks have moved beyond "Approval Buttons" to **Co-Reasoning** and **Interrupt-Based Steering**, exposed natively in frameworks like LangGraph (interrupt+resume) and Microsoft Agent Framework. The bigger 2026 shift is **policy-gated autonomy**: a classifier or rule engine decides which actions run, which are denied, and which pause for a human, so people review the few calls that matter instead of clicking through all of them.

## Table of Contents

- [The HITL Spectrum](#the-hitl-spectrum)
- [Interrupts and Breakpoints](#interrupts-and-breakpoints)
- [Policy-Gated Autonomy](#policy-gated-autonomy)
- [Approval Laundering](#approval-laundering)
- [Time-Travel Debugging (State Editing)](#time-travel-debugging-state-editing)
- [Co-Reasoning (Shared Scratchpads)](#co-reasoning-shared-scratchpads)
- [Confidence-Based Escalation](#confidence-based-escalation)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The HITL Spectrum

| Pattern | Agent Autonomy | Human Role | Best For |
|---------|---------------|------------|----------|
| **Human-in-command** | Low | Drives every step | High-risk Legal/Medical |
| **Human-as-filter** | Medium | Approves/Edits final output | Content Generation |
| **Human-as-backup** | High | Only intervenes on error | Customer Support |
| **Human-on-the-loop** | Max | Audits logs after completion | High-volume analysis |

---

## Interrupts and Breakpoints

Modern architectures (LangGraph, Microsoft Agent Framework) use **Deterministic Breakpoints**.

- **The Pattern**: The system is hardcoded to "Pause" before a specific sensitive tool is called (e.g., `execute_purchase` or `delete_user`).
- **The Decision**: The environment waits for a user to send an `approve` or `reject` signal.
- **State Preservation**: The agent's reasoning state is "Frozen" in the DB until the human acts. To survive restarts and deploys, hold the pause in a durable workflow (see [Durable Execution](11-durable-execution.md)); for approvals inside an MCP tool call, the stateless 2026-07-28 revision returns `input_required` and resumes on a client retry (see [Tool Use and MCP](03-tool-use-and-mcp.md#multi-round-trip-requests-mrtr)).
- **Gate the action, not the turn**: models now emit several actions per turn (Claude's computer-use toolset runs a batch in order and halts at the first failure), so one model round trip can complete a multistep consequential action, and approving the batch as a whole means approving steps nobody looked at. Inspect every action in the batch on arrival and run the confirmation before each consequential action executes, including one in the middle of a batch; Anthropic's computer-use docs place the check before each block runs. A rejection ends the batch: answer the remaining actions as not executed so the model replans from a known state.

---

## Policy-Gated Autonomy

Per-action approval does not scale, and the evidence says humans are a weak filter when they do review: in Anthropic's blind study of 1,053 paid testers (cited when it made classifier-based auto mode the Claude Code default for Pro, Max, and Team plans on August 14, 2026), humans caught a swapped-in dangerous command 13.6% of the time, while the auto-mode classifier blocked 89%. Humans are not an independent check on judgment calls either: in Anthropic's randomized study of 50 professionals with causal-inference experience analyzing a dataset with Claude Code and Claude Opus 4.8 (October 27, 2026, vendor-reported), a hidden instruction set the agent's lean on four analytic choices by coin flip, and analysts matched the lean 86% of the time against about 50% by chance; analysts with a prior preference switched 60% of the time when the agent disagreed versus 6% when it agreed, and expertise did not predict deferral. The design consequence: ask for the human's view before showing the agent's, and measure deferral rates. Vendors have moved the default from "ask every time" to "policy decides, human sees the exceptions":

| Product (2026) | How the gate works |
|---|---|
| **Claude Managed Agents** auto permission policy (September 10) | The server evaluates each agent or MCP tool call and runs it, denies it, or pauses for approval; the result appears in an `evaluation` field on events, and `ant beta:sessions connect` attaches a terminal to approve live |
| **Claude Code** auto mode | Server-side classifier; since 2.1.284 (September 28) interactive sessions with no configured permission mode start in auto mode. Headless runs differ: since 2.1.302 (October 16), `claude -p` and Agent SDK sessions with no permission mode start in `default` mode on third-party providers, with telemetry off, or on a fresh install, so CI callers should set the mode explicitly |
| **OpenAI dots** (announced September 29) | Built-in rules plus Custom Rules (allow, block, or require approval per action); actions that could affect accounts or share information are auto-reviewed; when idle, a dot may only do read-only research |
| **OpenAI Agents API** computer use (September 29) | Each new website origin needs an explicit `browser_origin_access` approval (enabling network access does not count); sign-in goes through a separate UI so credentials never enter model input |
| **GitHub Copilot** assisted permissions (GA October 26 in Copilot CLI, the Copilot app, and VS Code) | An LLM evaluates command, file-path, and URL requests; clearly safe ones proceed automatically and the rest need approval. Admins set the default mode for new sessions (falling back to manual approval if unavailable) and can block Allow All while keeping Assisted |
| **Claude Duet** (research preview, October 21; US Pro, Max, and Ultra waitlist) | A proactive agent working across connected apps from its own cloud computer and browser on Claude Opus 5.5. It asks before sending messages, making payments (each purchase uses a one-time card), signing documents, or deleting anything, and blocks trading securities, moving money between accounts, and accessing someone else's account even with approval: a clean ask-list versus block-list split |
| **GitHub Copilot** computer use (public preview, October 1) | Approval per application, with reviewable and resettable permanent grants and an org-level kill switch |

A related primitive is the **decision model**: a model that scores a fixed set of answers instead of generating text, which is exactly what a gate needs. OpenAI's Decisions API (limited preview September 29, beta October 6, GA October 22, on `gpt-6-luna`) classifies inputs, routes requests, or chooses an agent action; Cloudflare's Clef (October 1, Apache 2.0) returns probabilities over predefined answers so a system can act or defer, at a vendor-reported 209 ms median on a Qwen3.8-27B base and 39 ms for Clef-flash. Cloudflare's framing that humans no longer need to be in the loop is a vendor claim; the defensible use is as the cheap first gate in front of a human. Research on decision models reports two failure modes worth testing for, underconfident probabilities and agents that read the state correctly and still take the wrong action (see [Research Radar, theme 16](../RESEARCH-RADAR.md#16-decision-models-typed-answers-with-probabilities) for the papers and pricing), so calibrate the threshold on labeled traffic and verify the action, not just the classification.

**An AI approval can now satisfy a human gate.** On October 28, 2026 GitHub made Copilot code review approvals generally available after a public preview: when an admin turns them on, a Copilot approval counts toward a repository's required-approvals rule (off by default, repository admins choose which file paths Copilot may approve, and new commits dismiss it like a human approval). The same post added a Max review effort in public preview that runs several independent review agents on models from different providers plus an aggregation agent, at more AI credits than Lite or Balanced. When an agent wrote the change, letting another model's approval satisfy branch protection turns a two-person rule into a zero-person rule. Keep at least one human approval on paths that touch auth, payments, infrastructure, and the agent's own configuration, and allow the AI approval only where a wrong merge is cheap to revert.

**The caveat to say out loud:** a permission classifier is a convenience layer, not containment. When Johann Rehberger showed an auto-mode bypass in August 2026 (a "confused environment" chain that succeeded in 3 or 4 of 5 small-sample runs), Anthropic's response, per his report, was that auto mode is "a convenience feature backed by a best-effort classifier, not a security guarantee" and that the real boundary is OS isolation and network egress control. Anthropic's own red-teaming ("Red-teaming Auto Mode," October 22, 2026, vendor-reported) found production monitors failed to block over 55% of attacks (the attack classes are in [Agentic Security](09-agentic-security-and-sandboxing.md#containment-has-to-be-out-of-band)). Pair every policy gate with that chapter's sandboxing.

---

## Approval Laundering

The approval a human gives is only as good as the binding between what they saw and what runs. A September 2026 paper (arXiv 2609.38983) defines **approval laundering**: the action a coding-agent harness executes differs from the action the human approved. It names six classes (scope, argument, temporal, tool, delegation, and semantic) and measured them at Claude Code's `PreToolUse` approval point. A cryptographic approval token, checked by paired replay of 118 runs, eliminated delegation laundering and the paper's seeded temporal case, but by design left scope laundering untouched and did not significantly reduce argument laundering, because those diverge one process level below what a field-level verifier can observe.

What to build:

- **Bind approval to the exact canonical action** (tool, arguments, target, session) with a hash or signed token, and re-verify at execution time.
- **Show the human the executed form**, not the model's description of it.
- **Expire approvals** quickly and scope them to one session and one agent identity, so they cannot be replayed by a subagent.
- **Keep the sandbox.** Even signed approvals leave gaps, so approval reduces risk; it does not replace isolation.

---

## Time-Travel Debugging (State Editing)

Standard agents are "One-way." If they make a mistake in Step 3, the session is usually ruined.
- **Innovation**: **State Injection**. A human reviewer can "Go back" to the state at Step 3, edit the agent's observation or thought, and then "Resume" execution.
- **Impact**: It allows humans to "Steer" the agent off a bad path without starting from zero.
- **Edit observations and plans, not thinking.** On the newest Claude models, thinking blocks are bound to the model and conversation that produced them, and a request whose prior thinking no longer matches its prefix can be rejected. Rewind to the checkpoint, edit the observation or the written plan, and drop any thinking produced after the edit point rather than rewriting it.

---

## Co-Reasoning (Shared Scratchpads)

Instead of the human being a "Judge," they become a **"Partner."**
- The agent shows its **Scratchpad** (Internal Thinking) to the human.
- Characterized as: *"I am planning to use Tool A because of Fact B. Does that seem right to you?"*
- **Benefit**: Catching reasoning errors *before* they translate into actions.
- **Share a plan, not raw thinking.** Frontier APIs no longer hand back raw chain-of-thought: OpenAI returns reasoning summaries, and on the newest Claude models thinking blocks arrive empty at the default `display: "omitted"` unless you opt into a display mode that returns text. Have the agent write an explicit plan and rationale for the human to review, and treat it as the agent's account of its reasoning, not a transcript of it.
- **Put the plan where the team already works.** Asana's agents (described September 29, 2026) post their research plan and the steps they took into the shared task, so any reviewer can comment and steer mid-run, while only admins and editors can commit that feedback to the agent's permanent shared memory.

---

## Confidence-Based Escalation

We calculate an **Uncertainty Score** for each consequential step.

- If the score exceeds a threshold, the agent **Automatically Pauses** and sends a notification to a human operator.
- **Where the score comes from has changed.** Token logprobs are disappearing from frontier APIs (GPT-6 Astra exposes no logprobs, temperature, or top_p, and Anthropic's API version `2026-10-21` rejects `temperature`, `top_p`, and `top_k` on every Claude model), and verbalized confidence is poorly calibrated. The current pattern is a separate, calibrated scorer: a decision model or classifier that returns a probability over a fixed set of outcomes (proceed, ask, escalate), with a defer threshold tuned on labeled cases.
- **Example**: An agent trying to resolve a complex billing dispute realizes the user's intent is ambiguous. It stops and says: *"I'm not 100% sure how to handle this specific refund case. One moment while I get a human expert to look at this."*
- **Escalations are product data.** Anthropic's inbound-sales agent on Claude Managed Agents (described September 30, 2026) ends each conversation in checkout, a quick answer, or a hand-off to a rep with the full conversation *and an explanation of why*. Anthropic reports (vendor figures) more than twice the lead-to-opportunity rate, deals closing about five days faster, and the share of conversations needing a human roughly halved. The "why" field is what makes the escalation queue a feedback loop rather than a dumping ground.

---

## Interview Questions

### Q: How do you design an HITL system that doesn't "Fatigue" the human operator?

**Strong answer:**
We use **Threshold Tuning**. We don't ask for approval on every action. We only trigger HITL for: 1) High-risk "Writing" tools, 2) Low-confidence reasoning steps, or 3) Actions that violate a "Policy" set by the business. Additionally, we provide the human with a **Contextual Summary**: instead of the whole log, we show them a 1-sentence "Diff" of what the agent wants to do, generated from the canonical action rather than the model's description of it. This reduces the "Review cognitive load" from minutes to seconds. In 2026 the bigger lever is moving the routine decisions to a policy gate (Claude Managed Agents' auto permission policy, OpenAI dots' Custom Rules) so humans only see the exceptions.

### Q: What is the "Over-Reliance" risk in HITL, and how do you mitigate it?

**Strong answer:**
Over-reliance happens when humans start clicking "Approve" without reading the logs. It is measurable: in Anthropic's blind study, human reviewers caught a swapped-in dangerous command only 13.6% of the time. We mitigate this with **Forced Review Checkpoints** (e.g., the human MUST edit at least one word in the proposed plan) or **Synthetic Error Injections** (intentionally showing the human a "wrong" plan 1% of the time to see if they catch it). If they pass the "Trap," they continue; if they fail, they are flagged for additional training. Because human catch rates are this low, I do not let approval be the only control on an irreversible action: it sits on top of a sandbox and a policy gate.

### Q: You gate agent actions with a classifier that can run, deny, or defer to a human. How do you set and monitor the defer threshold?

**Strong answer:**
I treat it as a calibrated decision problem with asymmetric costs. Offline, I label a few hundred real actions with the decision a careful reviewer would make, check calibration (does 0.9 mean right 90% of the time?), and pick the threshold where the expected cost of a wrong auto-approve, weighted by blast radius, equals the cost of a human review. The threshold differs by action class: read-only actions get a permissive threshold, writes a strict one, and irreversible or external-facing actions always defer regardless of score. Online, I monitor four signals: defer rate (a sudden rise means drift or an attack), human override rate on deferred items (if humans approve 99% of deferrals, the threshold is too tight and reviewers will stop reading), sampled audits of auto-approved actions (the only way to see false approvals), and latency added by the gate. I re-calibrate on every model or prompt change, because a new agent model shifts the action distribution under the classifier. And I never present the classifier as the security boundary; Anthropic has described its own auto-mode classifier as a best-effort convenience layer, not a security guarantee.

---

## References
- LangChain. "Human-in-the-loop in LangGraph" (2024/2025)
- Wang, Y. Approval laundering in coding-agent harnesses, arXiv 2609.38983 (September 2026). https://arxiv.org/abs/2609.38983
- Claude Platform release notes (Managed Agents auto permission policy, September 10, 2026). https://platform.claude.com/docs/en/release-notes/overview
- Rehberger, J. "Breaking Claude Code Opus 5 Auto Mode" (Embrace The Red, August 2026). https://embracethered.com/blog/posts/2026/breaking-claude-code-opus-5-and-automode/
- Anthropic. "How Anthropic's sales team rebuilt inbound with Claude Managed Agents" (September 2026). https://claude.com/blog/how-anthropics-sales-team-rebuilt-inbound-with-claude-managed-agents
- Anthropic. "Red-teaming Auto Mode" (October 22, 2026). https://alignment.anthropic.com/2026/red-teaming-auto-mode/
- OpenAI. "Decisions API" guide. https://developers.openai.com/api/docs/guides/decisions
- Claude Code changelog (2.1.302 headless permission-mode default, October 16, 2026). https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md

---

*Next: [Agentic Security and Sandboxing](09-agentic-security-and-sandboxing.md)*
