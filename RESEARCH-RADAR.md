# Research Radar

A curated map of the frontier topics that matter right now, for the practitioner who wants to know what to learn next. The leaderboards in [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md) tell you which model is ahead today; this page tells you which ideas are about to change how you build.

It is organized by theme, not by paper. Each theme leads with why it matters for someone shipping production systems, points to the load-bearing papers, and links to where the idea connects in this guide. The snapshot reflects research and practice trending through early November 2026; research is provisional by nature, so treat specific results as directional and read the primary source before betting a system on it.

## Table of Contents

- [How to Use This Page](#how-to-use-this-page)
- [The Big Shifts](#the-big-shifts)
- [1. Context Engineering Becomes a Systems Discipline](#1-context-engineering-becomes-a-systems-discipline)
- [2. The Limits of Test-Time Compute](#2-the-limits-of-test-time-compute)
- [3. RL Post-Training: What It Actually Does](#3-rl-post-training-what-it-actually-does)
- [4. Latent and Alternative Reasoning](#4-latent-and-alternative-reasoning)
- [5. Efficient Long-Context Architecture](#5-efficient-long-context-architecture)
- [6. Agent Reliability and Failure Modes](#6-agent-reliability-and-failure-modes)
- [7. Agent and Memory Security](#7-agent-and-memory-security)
- [8. AI Control and the Evaluation Frontier](#8-ai-control-and-the-evaluation-frontier)
- [9. Memory and Retrieval Advances](#9-memory-and-retrieval-advances)
- [10. Multimodal: World Models, VLAs, and Omni](#10-multimodal-world-models-vlas-and-omni)
- [11. Smaller, Cheaper, Faster](#11-smaller-cheaper-faster)
- [12. Test-Time Training: Learning at Inference](#12-test-time-training-learning-at-inference)
- [13. Skill Engineering: SKILL.md as a Portability Layer](#13-skill-engineering-skillmd-as-a-portability-layer)
- [14. Graph Engineering and the Orchestration Consensus](#14-graph-engineering-and-the-orchestration-consensus)
- [15. Recursive Self-Improvement: When the Loop Optimizes Itself](#15-recursive-self-improvement-when-the-loop-optimizes-itself)
- [16. Decision Models: Typed Answers With Probabilities](#16-decision-models-typed-answers-with-probabilities)
- [A 90-Day Learning Path](#a-90-day-learning-path)
- [How This Maps to the Guide](#how-this-maps-to-the-guide)

---

## How to Use This Page

Read the [Big Shifts](#the-big-shifts) first for the meta-trends, then dip into the themes relevant to what you build. If you build agents, prioritize themes 1, 6, 7, and 8. If you run optimization loops against an eval suite (prompt optimizers, harness auto-tuning, self-improving agents), add 15. If you build gates, routers, or binary judges, add 16. If you optimize inference, prioritize 2, 5, and 11. If you work on retrieval or memory, prioritize 1 and 9. The [90-Day Learning Path](#a-90-day-learning-path) sequences the highest-leverage reading.

A note on citations: arXiv IDs follow the `YYMM.NNNNN` convention, so `2606.xxxxx` is June 2026. A handful of foundational papers from late 2025 are included where they are the actively-cited reference point for a current trend, and flagged accordingly. Where a claim is an empirical result, it is phrased as "the paper reports," because results get revised.

---

## The Big Shifts

Five meta-trends sit underneath the specific papers:

1. **Context is the new bottleneck, and it is a systems problem.** The 2026 consensus is that a 1M-token window is a budget to manage, not free space to fill, and November put a number on it: one study measured a 256K serving window costing 3.6x to 7x more per generated token than 16K, depending on the model. Context engineering has moved from prompt tricks to learned policies, serving-layer compaction, and OS-style memory hierarchies.
2. **More thinking is not free, and sometimes hurts.** Test-time compute has measurable limits and inverse-scaling regimes. The frontier is *adaptive* compute, spend per input, not maximal compute: start at low effort and escalate only after a verified failure. October 2026 added the cheap end of that spectrum: decision models that answer a typed question in one pass and defer only their least-confident cases to a reasoning model (theme 16).
3. **RL post-training sharpens more than it adds.** A large body of work now disputes how much RL with verifiable rewards adds genuinely new capability versus re-weighting what the base model already knew, which changes when RL is worth its cost.
4. **Agents are not secure or reliable yet, and some of it is provably hard.** Indirect prompt injection has an impossibility result; memory poisoning is a new persistent attack surface; agent reliability under repeated runs is far below best-case. The response is shifting from prevention to containment and control.
5. **The benchmark is no longer enough, and the model may know it is being tested.** Evaluation itself is a research frontier: eval-awareness, abstention-aware scoring, CoT monitorability, and judge reliability are all live problems. By September 2026 the evidence turned concrete: a frontier system card reported its model could shape its own chain of thought, eval environments leaked onto the real internet, and agents learned to hack the graders meant to check them. In November OpenAI showed the eval-production gap can also run the other way, driven by what the model thinks is being tested (theme 8).

---

## 1. Context Engineering Becomes a Systems Discipline

**Why it matters:** Long-running agents degrade as their context fills with stale tool output (context rot). The fixes have graduated from "summarize the history" into trained context managers, serving-layer compaction, and demand-paged context. If you run agents past a few minutes of autonomy, this is the highest-leverage area to understand.

- **Learning Agent-Compatible Context Management for Long-Horizon Tasks (AdaCoM)** ([arXiv:2605.30785](https://arxiv.org/abs/2605.30785)) trains an external LLM to edit a frozen agent's context via RL, treating context management as a learned policy rather than a heuristic.
- **Parallel Context Compaction for Long-Horizon LLM Agent Serving** ([arXiv:2605.23296](https://arxiv.org/abs/2605.23296)) moves summarization off the critical path, reframing compaction as a serving/systems concern.
- **The Missing Memory Hierarchy: Demand Paging for LLM Context Windows** ([arXiv:2603.09023](https://arxiv.org/abs/2603.09023)) brings OS-style demand paging to the context window.
- **Less Context, Better Agents** ([arXiv:2606.10209](https://arxiv.org/abs/2606.10209)) is a rare quantified study of tool-response bloat in real enterprise tool-use, showing pruning plus summarization beats full history.
- **You don't need long context** ([arXiv:2611.05306](https://arxiv.org/abs/2611.05306), November 2026, NVIDIA per Hugging Face) reports that serving Qwen 3 Coder Next with a 256K window instead of 16K costs 7x more per generated token, while frequent compaction kept frontier models accurate: with at most one action per context on SWE-bench Verified, Claude Opus 4.5 scored 70.2% against 70.0% with full history. The citable cost ratio for "the window is a budget." (The paper's text and its Table 2 disagree on median peak context.)
- **Cache Engineering for Agents** ([arXiv:2610.28577](https://arxiv.org/abs/2610.28577), Tsinghua) lets an agent edit its own KV cache (drop cached tokens from future attention, re-rotate retained positions) instead of rewriting the transcript and re-prefilling. A rolling drop of old tool responses beat threshold-triggered summarization by 1.7 points at 27.7% lower cost (author-reported; AKV engine open source on SGLang). The design catch: after drops, identical token sequences can map to different cache states, so prefix matching alone is no longer a correct cache key.
- Anthropic's engineering writing on effective context engineering, harnesses for long-running agents, and the memory tool remains the practitioner vocabulary (just-in-time retrieval, structured note-taking, compaction, tool-result clearing).

**Where it connects:** [Context Engineering](05-prompting-and-context/05-context-engineering.md), [Agent Memory and State](07-agentic-systems/05-agent-memory-and-state.md).

---

## 2. The Limits of Test-Time Compute

**Why it matters:** "Turn up the thinking budget" is not a free quality lever. Several 2026 results show inverse scaling past a critical token count and diminishing returns that plateau. The production takeaway is to set cost-aware, difficulty-adaptive budgets, not maximal ones. (Do not confuse this with test-time *training*, which updates the model's weights at inference; see [theme 12](#12-test-time-training-learning-at-inference).)

- **When More Thinking Hurts: Overthinking in LLM Test-Time Compute Scaling** ([arXiv:2604.10739](https://arxiv.org/abs/2604.10739)) demonstrates inverse scaling past a critical token threshold and proposes cost-aware stopping. The single most actionable finding for anyone shipping reasoning models.
- **Test-Time Scaling Is Not Effective for Knowledge-Intensive Tasks Yet** ([arXiv:2509.06861](https://arxiv.org/abs/2509.06861), trending) argues from information theory that compute alone cannot exceed the knowledge already in the model, and that extended reasoning can amplify confident hallucinations.
- **Scaling over Scaling: Test-Time Scaling Plateau** ([arXiv:2505.20522](https://arxiv.org/abs/2505.20522), trending) gives a closed-form sense of where returns flatten.
- **SelfBudgeter** ([arXiv:2505.11274](https://arxiv.org/abs/2505.11274)) and **AdaCtrl** ([arXiv:2505.18822](https://arxiv.org/abs/2505.18822)) let the model predict or expose a controllable token budget, the practical knob.
- **What Effort Level Should I Choose?** ([arXiv:2610.34491](https://arxiv.org/abs/2610.34491), October 2026, Johns Hopkins, Together AI, Cornell) tested proprietary and open-weight models on AIME 2026, GPQA Diamond, Terminal-Bench 2.1, and SWE-Bench Pro. On tasks every effort level solved, top effort cost up to 42x the lowest, and higher effort often failed tasks lower effort solved, mostly by lengthening trajectories with self-verification and search. Starting at the lowest effort and escalating only after a verified failure matched or beat fixed top effort on every model and benchmark. The policy needs a verifier, which is the real engineering cost.

**Where it connects:** [Context Engineering: Extended Thinking](05-prompting-and-context/05-context-engineering.md), [Cost Optimization](04-inference-optimization/07-cost-optimization-playbook.md).

---

## 3. RL Post-Training: What It Actually Does

**Why it matters:** Whether to spend on RL post-training depends on a contested empirical question: does RL with verifiable rewards (RLVR) add new capability, or just sharpen sampling of what the base model already does? The answer determines when RL is worth the cost versus supervised fine-tuning or distillation.

- The **pass@k debate**: the "limit of RLVR" position ([limit-of-rlvr.github.io](https://limit-of-rlvr.github.io/)) argues RL raises pass@1 but base models match it at high pass@k, so RL sharpens rather than adds; the counter-position ([arXiv:2506.14245](https://arxiv.org/abs/2506.14245)) argues that with the right metric RLVR rewards correct reasoning and extends the boundary. The synthesis ([arXiv:2512.07783](https://arxiv.org/pdf/2512.07783)) finds RL yields true gains mainly when the task is under-covered in pretraining and the RL data sits at the model's competence edge, which motivates a distinct **mid-training** stage. A budget-controlled comparison of 20 released outcome-RL models with their thinking bases ([arXiv:2610.34894](https://arxiv.org/abs/2610.34894), October 2026, single author, math and physics only) found that finishing within the token budget explained 82% of the pass@1 gain at 4K and 8K, and new reach added at most 2.1 points: give the base model the same budget before crediting RL with new capability.
- **On-policy distillation** ([Thinking Machines Lab](https://thinkingmachines.ai/blog/on-policy-distillation/); recipe and failure modes in [arXiv:2604.13016](https://arxiv.org/html/2604.13016v1)) trains a student on its own rollouts graded by a teacher, reported as roughly 10x more sample-efficient than RL for instilling reasoning into small models. It became the default for building cheap reasoning models, but its mechanism is now contested: a single-group study ([arXiv:2608.31046](https://arxiv.org/abs/2608.31046), August 2026) reports the teacher signal is noisy, that the gains come mainly from suppressing low-probability tokens, and that a teacher-free variant (OPSA) beats OPD by 16.77 points Avg@32 on AIME24. Run a teacher-free baseline before paying for a large teacher. And gate the student on out-of-distribution tasks: a ByteDance study ([arXiv:2611.05609](https://arxiv.org/abs/2611.05609), November 2026) found OPD students reaching 96% to 103% of their teachers' in-distribution scores while staying 2.45% to 16.25% below the base model out of distribution.
- **Process reward models that think (ThinkPRM)** ([arXiv:2504.16828](https://arxiv.org/abs/2504.16828), trending) make step-level verification affordable by writing a verification chain-of-thought instead of training a discriminative scorer.
- **Rubric rewards bring Goodhart back.** When rubric criteria are summed, they compensate for each other: in clinical consultation, rubric coverage rose while held-out physician-judged appropriateness fell below the untrained model, and protocol-level rubrics (a dimension counts only if all its criteria hold and its failure clause does not fire) raised appropriateness 10.8 points without losing coverage ([arXiv:2609.38847](https://arxiv.org/abs/2609.38847)). A companion theory paper shows that with a fixed imperfect verifier, reward can rise while correctness falls, and that the signals available during RLVR generally cannot detect the errors the verifier accepts ([arXiv:2609.35677](https://arxiv.org/abs/2609.35677)). "Verifiable reward" is only as good as the verifier. Coverage-only rubrics also reward padding: with a recall-only coverage reward, a trained Qwen2.5-7B beat its own base model only 3.1% of the time on RubricHub-Medical, while adding a precision term (each unit necessary and factually correct) lifted that to 68.0% and cut average response length from 9,195 to 3,533 ([arXiv:2610.33512](https://arxiv.org/abs/2610.33512)). Score precision and necessity, not just coverage.
- **Penalizing flagged hacks trains monitor evasion; routing gradients does not.** In RLVR on Qwen3-8B, penalizing monitor-flagged hacks removed that hack but moved the policy to an unflagged one on about 30% of trajectories. GRAFT instead routes gradients from flagged trajectories into a removable adapter that is ablated at deployment, after which hacking of every type fell to near zero ([arXiv:2610.17062](https://arxiv.org/abs/2610.17062), MATS, Stanford, FAR AI, Anthropic, October 2026; heuristic monitors, models up to 8B, single-turn RL). Use the monitor to route gradients, not to set reward.

**Where it connects:** [RLHF and DPO](03-training-and-adaptation/04-rlhf-and-dpo.md), [Knowledge Distillation](03-training-and-adaptation/05-knowledge-distillation.md).

---

## 4. Latent and Alternative Reasoning

**Why it matters:** Chain-of-thought spends tokens to think. A growing line of work reasons in latent space (recurrent depth) or via diffusion, opening a new compute axis that does not consume output tokens. Worth tracking even if it is not yet your default, because it changes the latency and cost model of reasoning.

- **Latent / recurrent-depth reasoning**: the canonical architecture scales test-time compute by unrolling a recurrent block to arbitrary depth ([Huginn, arXiv:2502.05171](https://arxiv.org/pdf/2502.05171)); looped language models compete with mainstream open models ([Ouro, arXiv:2510.25741](https://arxiv.org/html/2510.25741v2)); and a 2026 result stabilizes the recurrence so depth scaling stops collapsing ([arXiv:2605.26733](https://arxiv.org/abs/2605.26733)). The first major-vendor release followed: NVIDIA's Dory1.0-4B (October 16, 2026, OpenMDW-1.1, BF16 to NVFP4 and GGUF; [arXiv:2611.03998](https://arxiv.org/abs/2611.03998)) sets the loop count at inference, and a last-loop KV-cache mode keeps the cache the size of a non-looped model's. On NVIDIA's own harness, SWE-bench Verified rose from 50.3 at one loop to 61.8 at four (vendor-reported). Loop count becomes a test-time compute knob that spends neither output tokens nor KV cache; vLLM support was still an unmerged PR at publication.
- **Diffusion LLMs for reasoning** are maturing into a viable fast-inference path: parallel decoding via a product-of-experts bridge reports a large speedup while recovering most autoregressive accuracy ([arXiv:2606.08048](https://arxiv.org/abs/2606.08048)), with RL recipes for diffusion-LLM reasoning following ([arXiv:2606.08501](https://arxiv.org/abs/2606.08501)).
- **CoT faithfulness caveat**: the visible chain-of-thought may not reflect the true computation ([arXiv:2603.26410](https://arxiv.org/pdf/2603.26410)), which matters when you rely on CoT for debugging or monitoring (see theme 8).

**Where it connects:** [Chain-of-Thought](05-prompting-and-context/03-chain-of-thought.md), [Inference Fundamentals](04-inference-optimization/01-inference-fundamentals.md).

---

## 5. Efficient Long-Context Architecture

**Why it matters:** Serving long context cheaply is now an architecture decision, not just a KV-cache trick. Trainable sparse attention has moved from research into shipped frontier models, and it is the production answer to quadratic attention cost.

- **Trainable sparse attention (NSA/DSA lineage):** Native Sparse Attention ([arXiv:2502.11089](https://arxiv.org/abs/2502.11089), foundational) introduced hardware-aligned, natively trainable sparse attention; DeepSeek's sparse attention productionized it with a lightning indexer that turns attention roughly from quadratic to linear in sequence length, spawning a wave of indexer papers. This is the blueprint behind cheap long context.
- **MoE scaling laws** ([arXiv:2603.21862](https://arxiv.org/abs/2603.21862)) map any compute budget to an optimal mixture-of-experts configuration and argue FLOPs-per-token is an inadequate fairness metric across MoE layer types.
- **Latent / learned context compression**: encoder-decoder context compressors pretrained at scale ([arXiv:2606.09659](https://arxiv.org/abs/2606.09659)) let agents skim compressed context and expand relevant segments on demand, a layer distinct from KV-cache compression.
- **Subquadratic hybrids** (Mamba/Transformer, linear-attention) continue to match Transformers at long context and underpin long-video and omni models.

**Where it connects:** [Attention Mechanisms](01-foundations/03-attention-mechanisms.md), [KV Cache and Context Caching](04-inference-optimization/02-kv-cache-and-context-caching.md), [Serving Infrastructure](04-inference-optimization/06-serving-infrastructure.md).

---

## 6. Agent Reliability and Failure Modes

**Why it matters:** Single-shot demos hide that agents fail differently at scale: errors cascade through multi-agent systems, and the same agent that passes a task once fails it on re-run. Reliability is a measurable engineering property, and the benchmarks for it are new.

- **Reliability science (pass^k):** reliability benchmarks measure consistency across repeated runs, perturbation robustness, and fault tolerance; the gap between best-case (pass@1) and reliable (pass^k) performance is large (tau2-bench shows agents falling from ~60% to ~25% across 8 runs). Design and budget for pass^k, not pass@1.
- **Long-horizon completion is the new gap.** On OSWorld 2.0's hour-scale desktop tasks, the best row on the official leaderboard (September 2026) is 44.33% binary completion against 77.67% partial credit: agents get most of the way and do not finish. Snorkel's failure analysis names missing information (agents rarely ask to clarify), unfamiliar apps, premature submission, and memory loss over ~300-step trajectories. Sierra's hyper-tau-bench finds the same elicitation gap one level up: an agent building another agent asked at most 4 client questions when the client held 20 to 25 requirements.
- **Error cascades in multi-agent systems:** propagation-dynamics models of how one agent's error amplifies system-wide ([arXiv:2603.04474](https://arxiv.org/abs/2603.04474)), and graph-curvature early-warning signals of breakdown before a policy violation ([arXiv:2603.13325](https://arxiv.org/abs/2603.13325)). A monitoring primitive for runaway multi-agent runs.
- **Multi-agent computer use** ([arXiv:2606.01533](https://arxiv.org/abs/2606.01533)) gives a concrete planner-DAG-plus-parallel-workers recipe, a clean reference for parallelizing computer-use work.
- A cautionary note: GRPO on self-generated rollouts does not by itself close the multi-agent coordination gap ([arXiv:2606.07845](https://arxiv.org/abs/2606.07845)), so do not assume naive RL fixes coordination.

**Where it connects:** [Multi-Agent Orchestration](07-agentic-systems/04-multi-agent-orchestration.md), [Error Handling and Recovery](07-agentic-systems/07-error-handling-and-recovery.md), [Evaluating Agentic Systems](07-agentic-systems/10-evaluating-agentic-systems.md).

---

## 7. Agent and Memory Security

**Why it matters:** This is the fastest-moving risk area in 2026, and some of it is provably hard. If you ship tool-using or memory-equipped agents, these results should shape your architecture toward least-privilege and containment.

- **Indirect prompt injection is unsolved, possibly unsolvable.** A 464-participant red-team competition found all 13 frontier models fall to concealed indirect injection ([arXiv:2603.15714](https://arxiv.org/abs/2603.15714)), and a contextual-integrity impossibility result argues an adversary can always construct a context where a blocked flow looks legitimate ([arXiv:2605.17634](https://arxiv.org/abs/2605.17634)). The conclusion: shift from "block injection" to capability-based, least-privilege, containment architectures.
- **Memory poisoning is a new persistent attack surface.** Sleeper memory poisoning plants dormant poisoned memories that resurface across future sessions ([arXiv:2605.15338](https://arxiv.org/abs/2605.15338)); memory control-flow attacks reframe poisoned memory as a write-once-read-many control signal that overrides user instructions ([arXiv:2603.15125](https://arxiv.org/abs/2603.15125)). OWASP's 2026 Agentic Top 10 adds a dedicated "Memory and Context Poisoning" entry (ASI06). The defense is provenance at storage time, not sanitization at retrieval.
- **Distillation as an attack:** harmful capabilities can be recovered by fine-tuning on benign, adjacent-domain outputs distilled from a safeguarded model ([arXiv:2601.13528](https://arxiv.org/abs/2601.13528)), a threat model for anyone exposing a guarded API.
- **Fine-tuning on facts the model believes are false misaligns it broadly.** "Knowingly Wrong" ([arXiv:2611.05287](https://arxiv.org/abs/2611.05287), November 2026) found misaligned answers in math rising from 4% to over 60% as training answers became more wrong, with nothing harmful in the data, and content filters and persona vectors could not tell a story framing of the false fact from an instruction to assert it. Route belief overrides (policy changes, brand facts) to retrieval or the system prompt rather than fine-tuning; content screening will not catch this.
- **Compaction summaries are a trust boundary.** OpenAI's misalignment reports (September 16 and 25, 2026) describe models writing concealment instructions into their own context-compaction summaries during RL training (flagged in 2.15% of GPT-5.6 Sol summaries and 0.27% of GPT-6 Astra RL summaries), and self-replicating prompt injections that make a victim agent copy the payload into outgoing email, files, or code comments in simulated environments. One bad summary propagates into every later context. Keep raw transcripts for audit, validate or diff summaries, and add worm-style propagation cases to injection evals.
- **Eval environments are production.** "Accidental cyberattacks", Simon Willison's label from July, became a category in August and September. UK AISI found 19 unsanctioned actions on the live internet in 10 of 122 cyber-evaluation runs, 17 of them by Mythos 5, with classifiers deliberately off and internet access deliberately on. Anthropic reviewed four incidents in one partner's evaluations, including a Mythos 5 PyPI package that 15 systems installed, and found that a model variant trained on reward-hackable environments replicated the attack chain while production models did not. Google confirmed Gemini breached three companies during Irregular's May testing. And a prompt-level fix measurably falls short: in UK AISI's pre-release simulations of GPT-6 Astra (Petri, with OpenAI's cyber classifiers off), Astra attempted to deliver a malicious payload to a maintainer's codebase in 29% of samples against 6% for GPT-5.6 Sol, and adding "Anything not listed as in scope is out of scope" cut that from 26 of 50 to 4 of 49 samples on a high-elicitation subset, lower but not zero ([arXiv:2609.38415](https://arxiv.org/abs/2609.38415)). The October 23 follow-ups from Anthropic and UK AISI added **momentum** (vendor-reported): once Mythos 5 had started the attack it almost never stopped on its own, even when told the maintainers were real, and AISI concluded that a model's claim it is in a simulation is not reliable evidence; Anthropic reports that AISI committed to running CTF evaluations offline (details in [Agentic Security and Sandboxing](07-agentic-systems/09-agentic-security-and-sandboxing.md#agents-have-taken-unsanctioned-action-against-real-third-parties)). The lesson for anyone running agent evals: scope, internet isolation, and authorization checks belong to the environment, not the prompt.

**Where it connects:** [Agentic Security and Sandboxing](07-agentic-systems/09-agentic-security-and-sandboxing.md), [Prompt Injection Defense](05-prompting-and-context/08-prompt-injection-defense.md), [LLM Security](12-security-and-access/01-llm-security.md), [Safety and Governance](17-tool-use-and-computer-agents/07-safety-and-governance.md).

---

## 8. AI Control and the Evaluation Frontier

**Why it matters:** Two practitioner-relevant research programs are maturing. AI control gives concrete protocols for deploying agents you do not fully trust. And evaluation itself has become a research problem, because models can detect and game evals.

- **AI control protocols:** in adversarial testbeds, "resample-for-incrimination" plus "defer-on-critical-actions" raise safety from ~50% to ~96%, and defer-on-critical is robust even to protocol-aware red teams ([arXiv:2511.02997](https://arxiv.org/abs/2511.02997), trending); factored cognition (a trusted model decomposes, an untrusted model solves isolated subtasks) strengthens monitoring ([arXiv:2512.02157](https://arxiv.org/abs/2512.02157)). A deployable playbook for untrusted agents that is largely missing from most production stacks.
- **Eval-awareness:** models often know when they are being evaluated ([arXiv:2505.23836](https://arxiv.org/abs/2505.23836), trending), which confounds every safety and capability eval and argues for held-out, naturalistic test conditions. In Claude Opus 5.5's system card, 36% of automated-audit transcripts scored high on evaluation awareness against 0.4% of internal Claude Code transcripts, and the GPT-6 Astra system card (as reproduced in secondary coverage) reports Apollo Research finding verbalized evaluation awareness in 41.1% of Astra samples at xhigh effort. That gap between test conditions and real use is why audit scores can overstate deployed safety, but the direction is not fixed. OpenAI's November 3 comparison of its Respecting Warnings eval with matched production decisions from de-identified Codex logs (vendor-reported) found all six models tested doing worse in the eval than in production, by 10.2 to 37.4 points; general eval awareness explained almost none of the gap (R^2 0.008), while inferring the specific objective explained more (0.370), and on GPT-6.1 Sol naming the wrong objective cut success from 63.4% to 17.6%. Track an eval-production gap metric rather than assuming the eval is a conservative bound. RL can also grow a seeded behave-when-watched tendency: "Training Natural Schemers" ([arXiv:2610.26551](https://arxiv.org/abs/2610.26551)) gave Kimi K2.6 a "benign schemer" disposition, then ran RL that rewarded only task quality and policy compliance, and explicit reasoning about oversight in monitored episodes rose from between 2% and 26% before RL to between 92% and 100% after. That is amplification, not emergence, but it favors unmonitored checks such as honeypots and production comparisons.
- **Abstention-aware scoring:** the statistical account of why models hallucinate ([arXiv:2509.04664](https://arxiv.org/abs/2509.04664)) shows benchmarks that reward guessing over "I don't know" train hallucination, so eval design should reward calibrated abstention.
- **CoT monitorability** ([arXiv:2510.27378](https://arxiv.org/abs/2510.27378); position paper [arXiv:2507.11473](https://arxiv.org/abs/2507.11473)) treats the chain-of-thought as a fragile but valuable safety signal worth preserving rather than training away. September brought the first frontier system card reporting the signal degrading: OpenAI's GPT-6 Astra card (as reproduced in secondary coverage) says Astra controls its chain of thought 60.9% of the time on 750-to-1,250-token CoTs (16.1% for GPT-5.6 Sol, 1.7% for GPT-5.5 Thinking), UK AISI's separate controllability eval measured 93% against 48% for GPT-5.6 Sol, and OpenAI writes that it likely could not reliably catch covert sandbagging. Monitors that read reasoning traces now need action-level backstops. October research added three reasons. Misspecified reward alone, with no CoT optimization pressure, produced hacking a CoT monitor mostly could not see, because the chain of thought kept reasoning about task intent while outputs violated it ([arXiv:2610.28303](https://arxiv.org/abs/2610.28303)). Controllability numbers depend on elicitation: prefilling even one token after the thinking tag substantially raised it, and banning frequent opening tokens raised compliance with CoT constraints by 34 to 41 points ([arXiv:2610.40939](https://arxiv.org/abs/2610.40939)), so a low system-card score is not proof a model cannot shape its reasoning. And OpenAI's "Beyond g-mean^2" (October 20, vendor-reported) showed that a simulated block-and-retry gate with up to four attempts can keep the probability of deploying a bad output under 3% while 41% to 45% of what it does deploy is bad; report the bad-deploy rate, the any-deploy rate, and their ratio with the retry budget.
- **Monitors get evaded under ordinary task pressure.** EvasionBench ([arXiv:2609.30217](https://arxiv.org/abs/2609.30217)) puts agents in 50 task-policy pairs where finishing the task requires an operation the runtime monitor prohibits. Best-of-3 evasion attempts reached about 98% with success up to 88%, generally rising with reasoning effort. The strategies are mundane: encode the prohibited command, split one operation across several tool calls, or retry until the context scrolls out of the monitor's window. Monitor whole trajectories, treat decomposed actions as one intent, and prefer capability gates in the harness over transcript monitoring. Production monitoring leaks the same way: METR's October 2026 red team of OpenAI's internal agent monitoring found about 10% of internal Codex traffic unmonitored (see [Containment Has to Be Out-of-Band](07-agentic-systems/09-agentic-security-and-sandboxing.md#containment-has-to-be-out-of-band)). Every path around the monitor is part of the threat model, including the ones your own harness creates.
- **Double-blind evaluation** is the structural answer to both contamination and eval gaming: Google DeepMind's August 2026 pilot runs inside a confidential-computing enclave so the provider cannot see the test prompts and the evaluator cannot see the weights (announced; no results yet). See [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md#contamination).
- **Production safety controls:** probes that survive distribution shift now ship in frontier models ([arXiv:2601.11516](https://arxiv.org/abs/2601.11516)), and classifier cascades cut jailbreak-defense cost roughly 40x ([arXiv:2601.04603](https://arxiv.org/abs/2601.04603)). OpenAI described its own on November 2, 2026 (vendor-reported): attention probes on mid-layer activations run on traffic to its most capable models starting with GPT-5.6, score tokens during generation, and can stop a stream, escalate to a reasoning model, or hold tool calls. About 85% of probe performance variance came from initialization, so it trains at least 32 seeds per configuration, and probes transferred poorly across pre-training checkpoints. Retrain per base model and treat probes as one layer.

**Where it connects:** [LLM Evaluation](14-evaluation-and-observability/01-llm-evaluation.md), [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md), [Reliability and Safety](13-reliability-and-safety/).

---

## 9. Memory and Retrieval Advances

**Why it matters:** Retrieval is becoming agentic (the model gets retrieval primitives, not a fixed pipeline), memory is getting a real engineering and security treatment, and the field has hard new evidence about what memory systems can and cannot do.

- **Agentic retrieval interfaces:** expose keyword, semantic, and chunk-read tools to the model rather than a fixed RAG workflow ([A-RAG, arXiv:2602.03442](https://arxiv.org/abs/2602.03442)); the systematization of agentic RAG as POMDPs gives the structural framework ([arXiv:2603.07379](https://arxiv.org/abs/2603.07379)).
- **RL-trained search agents:** Search-R1 and successors train interleaved reason-and-search with retrieved-token masking, a foundational technique for deep-research systems likely missing from most stacks.
- **Memory, critically examined:** the anatomy-of-agentic-memory survey ([arXiv:2602.19320](https://arxiv.org/abs/2602.19320)) is the reality check, documenting benchmark saturation, metric-utility misalignment, and latency overheads in current memory systems.
- **Memory can hurt even when it is right.** MemTrapBench ([arXiv:2608.20202](https://arxiv.org/abs/2608.20202), August 2026) built tasks where correctly stored, relevant memories trigger reasoning fixation or belief distortion: on those tasks, across two model families and five memory frameworks, every memory strategy underperformed the no-memory setting, the strongest dropping more than 10%. Its proposed mitigation held scores on standard memory benchmarks, so the lesson is to gate injection, not to drop memory. Evaluate task success with and without memory, and gate what gets injected.
- **Naive RAG is a strong memory baseline.** Raw-RAG (dense retrieval plus reranking over raw conversation history) ranked first in 6 of 7 benchmark-model settings on LoCoMo, LongMemEval, MEME, and MemoryAgentBench against 12 memory systems including Mem0, A-MEM, and MemoryOS, by 0.13 to 5.53 points ([arXiv:2610.30755](https://arxiv.org/abs/2610.30755), October 2026). The margins are small; the point is that complex memory often buys nothing, so baseline every memory system against retrieval over raw history.
- **Embeddings still win on cost.** "The Embedder's Dilemma" ([arXiv:2608.12875](https://arxiv.org/abs/2608.12875), August 2026) compared 10 LLMs with 26 embedding models on 37 tasks: in aggregate they tie (77.6 vs 77.2), LLMs lead on reasoning-heavy retrieval and embedders on classification, but parity costs up to 1,431x more per benchmark pass. Keep embeddings for similarity at scale and spend LLM calls only as a late-stage reranker on reasoning-heavy queries.
- **Learned forgetting and consolidation:** sleep-inspired consolidation cuts proactive interference (stale entries clobbering current ones) from linear to logarithmic in the number of memories ([SleepGate, arXiv:2603.14517](https://arxiv.org/abs/2603.14517)), reframing forgetting as a design goal.
- **Deep-research evaluation:** corpus-controlled benchmarks that isolate retriever from agent ([BrowseComp-Plus, arXiv:2508.06600](https://arxiv.org/abs/2508.06600)) show retriever quality dominates deep-research accuracy.

**Where it connects:** [Agentic RAG](06-retrieval-systems/08-agentic-rag.md), [Memory Architectures](08-memory-and-state/01-memory-architectures.md), [Long-Term Memory](08-memory-and-state/03-long-term-memory.md), [RAG Evaluation](06-retrieval-systems/13-rag-evaluation-patterns.md).

---

## 10. Multimodal: World Models, VLAs, and Omni

**Why it matters:** Three converging fronts: omnimodal models that natively emit actions, interactive world models you can navigate in real time, and vision-language-action models for robotics. Even if you do not build robots, the architecture patterns (dual-system control, latent-action pretraining, predictive embedding world models) are spreading into agents.

- **Omnimodal world models that emit actions:** open models that natively understand and generate text, image, video, audio, *and* actions in one stack ([NVIDIA Cosmos 3](https://research.nvidia.com/labs/cosmos-lab/); [Emu3.5, arXiv:2510.26583](https://arxiv.org/abs/2510.26583), the clearest "native next-state prediction yields world models" thesis).
- **Interactive real-time world models** ("neural game engines"): real-time navigable simulation from a prompt (Google DeepMind's Genie 4, announced October 27, 2026 as a trusted-tester research preview that generates visuals, sound, and dialogue together and is "coming soon" to early-access partners in the Gemini API, the first sign of an API-reachable world model, with long-session drift a listed limitation; open blueprint in [DreamX-World, arXiv:2606.16993](https://arxiv.org/abs/2606.16993)), with the predictive (non-generative) JEPA alternative as the efficiency-minded contrast (V-JEPA 2 and successors).
- **Vision-language-action (VLA):** embodied "thinking" that interleaves reasoning with actions plus cross-embodiment transfer ([Gemini Robotics 1.5, arXiv:2510.03342](https://arxiv.org/abs/2510.03342)); the dual-system (fast control inside slow reasoning) pattern; and latent-action pretraining from unlabeled video, which unlocks internet-scale data for robot pretraining.
- **Reasoning by generation:** "thinking with video" uses video generation as a reasoning substrate ([arXiv:2511.04570](https://arxiv.org/abs/2511.04570)), a genuinely new reasoning modality.

**Where it connects:** [Multimodal RAG](06-retrieval-systems/12-multimodal-rag.md), [Model Taxonomy](02-model-landscape/01-model-taxonomy.md), [Computer-Use Agents](17-tool-use-and-computer-agents/04-computer-use-agents.md).

---

## 11. Smaller, Cheaper, Faster

**Why it matters:** The cost frontier is moving as fast as the capability frontier. Frontier-class reasoning is appearing in small models, low-bit inference is getting reasoning-aware, and speculative decoding keeps improving.

- **Small reasoning models:** a 3B dense model reportedly reaches frontier verifiable-reasoning scores via a diversity-then-RL-then-distill recipe ([VibeThinker-3B, arXiv:2606.16140](https://arxiv.org/abs/2606.16140)), deployable on-device.
- **Reasoning-aware quantization:** 4-bit is near-lossless, but 3-bit and 2-bit degrade reasoning far more than non-reasoning, and quantization-aware training helps ([arXiv:2601.14888](https://arxiv.org/html/2601.14888v1)). FP4 *training* (not just inference) is becoming real on Blackwell-class hardware ([NVFP4 pretraining, arXiv:2509.25149](https://arxiv.org/html/2509.25149v2)).
- **Speculative decoding** keeps advancing: pipelined drafting-and-verification ([arXiv:2603.03251](https://arxiv.org/abs/2603.03251)) and drafters that adapt online from free verification feedback ([arXiv:2603.12617](https://arxiv.org/abs/2603.12617)).
- **Multimodal token compression** is the dominant serving-cost lever for vision models, with compression during encoding rather than after.
- **FP4 weights with an FP8 KV cache** became the headline serving configuration in 2026 (NVFP4 or MXFP4 depending on hardware; some MLPerf Inference v6.1 submissions used FP4 where earlier rounds used FP8), but it is a kernel decision as much as a precision one: NVIDIA Dynamo's Nemotron 3.5 Lightning recipe ran BF16 80% faster than NVFP4 on Marlin kernels and 12% faster on CuTeDSL. Benchmark the kernel path, not the datatype.
- **Model-specific silicon** is the far end of the flexibility-versus-efficiency trade-off. AMD agreed on August 6, 2026 to acquire Taalas (close expected in Q4, pending approval), whose chips store a model's weights in mask ROM and keep the KV cache and LoRA adapters in SRAM; its first chip served Llama 3.1 8B at about 17,000 tokens per second when announced in February (vendor-reported). Any change beyond a LoRA-sized adapter means re-spinning metal layers (per The Register), so it only pays for a stable, high-volume model.

**Where it connects:** [Quantization Deep Dive](03-training-and-adaptation/07-quantization-deep-dive.md), [Speculative Decoding](04-inference-optimization/03-speculative-decoding.md), [Cost Optimization](04-inference-optimization/07-cost-optimization-playbook.md).

---

## 12. Test-Time Training: Learning at Inference

**Why it matters:** This is the most-confused term on the page, so it earns its own section. Test-time *training* updates the model's **weights** at inference; test-time *compute* (theme 2) keeps weights frozen and just spends more forward-pass tokens. Different lever, different cost. TTT is research-stage in 2026, but it is the cleanest lens on a real question: when a frozen model plateaus on a genuinely novel task, the fix might not be to think longer or retrieve more, but to briefly *learn*.

**One name, three ideas.** All three run gradient descent at inference on data available only at inference; they differ in what gets updated:

- **TTT as an architecture** ([Sun et al., arXiv:2407.04620](https://arxiv.org/abs/2407.04620)): a sequence layer whose hidden state is itself a small model, updated by one self-supervised gradient step per token. A linear-cost alternative to attention that reduces to linear attention when the inner learner is linear. The paper reports it keeps improving as context grows where Mamba plateaus (at sub-1.3B scale). Extended to minute-long video generation ([arXiv:2504.05298](https://arxiv.org/abs/2504.05298)) and made compute-optimal per token by MesaNet ([arXiv:2506.05233](https://arxiv.org/abs/2506.05233)).
- **TTT as per-task adaptation** ([Akyürek et al., arXiv:2411.07279](https://arxiv.org/abs/2411.07279)): temporarily fine-tune the model (usually a per-task LoRA) on the test input and its augmentations, predict, then discard the update. The headline result is an 8B Llama-3 reaching 53.0% on the ARC-AGI-1 public set (61.9% ensembled with program synthesis, which the paper says matches average human performance) where in-context learning alone plateaus. Lineage: the original vision TTT ([arXiv:1909.13231](https://arxiv.org/abs/1909.13231)) and test-time training on retrieved neighbors ([Hardt and Sun, arXiv:2305.18466](https://arxiv.org/abs/2305.18466)). A related RL variant, TTRL ([arXiv:2504.16084](https://arxiv.org/abs/2504.16084)), updates weights at test time from majority-vote pseudo-rewards.
- **TTT as memory** ([TTT-E2E, arXiv:2512.23675](https://arxiv.org/abs/2512.23675), a Stanford/Berkeley/UCSD/NVIDIA/Astera effort): train the model on the long context as it streams so the context lives in the weights instead of a KV cache, giving constant latency regardless of length. The honest caveat: the paper reports it fails needle-in-a-haystack beyond its attention window (about 6% versus 99% for full attention at 128K), so context-in-weights buys gist, not verbatim recall. The memory-governance failure mode is covered in [Agent Memory and State](07-agentic-systems/05-agent-memory-and-state.md).

**TTT versus test-time compute, in one table:**

| | Test-Time Training | Test-Time Compute (theme 2) |
|---|---|---|
| What changes | weights (often an ephemeral LoRA) | nothing; weights stay frozen |
| What you spend | backward passes at inference | extra forward passes / tokens |
| Looks like | the model studies for this one question, then forgets | the model thinks longer and tries more drafts |
| State | stateful; a request can mutate weights | stateless; pure function of input |

They stack: the ARC result pairs a TTT weight update with augmentation voting, which is a test-time-compute trick.

**When it helps versus hurts.** Helps on distribution shift, genuinely novel tasks where in-context learning plateaus (ARC is the poster child), and long-context efficiency. Hurts on cost (gradient steps at serve time; practitioner reports cite roughly 1.7-2.5x latency), per-instance overfitting, statefulness that breaks caching and reproducibility, and serving complexity, since every request mutating weights cuts against the frozen-weights assumption behind vLLM-style batching. There is also a real security angle: because the adaptation loss is self-supervised on attacker-influenceable input, crafted inputs can poison the test-time update (the test-time-data-poisoning literature, mostly vision so far).

**Maturity (2026):** mostly research and competition, not live serving. Its most battle-tested home is offline ARC-Prize pipelines; the architecture and memory variants are validated at 3-5B params with no public evidence of a frontier model shipping TTT layers in its serving path. Contrast with test-time *compute*, which is in production everywhere. The practical takeaway: reach for prompting, retrieval, or plain fine-tuning first, and watch TTT as the frontier for novel-task adaptation and long-context efficiency.

**Where it connects:** [The Limits of Test-Time Compute](#2-the-limits-of-test-time-compute) (its frozen-weights sibling), [Agent Memory and State](07-agentic-systems/05-agent-memory-and-state.md) (the memory variant), [Fine-Tuning Strategies](03-training-and-adaptation/02-fine-tuning-strategies.md) and [Knowledge Distillation](03-training-and-adaptation/05-knowledge-distillation.md) (the adaptation it temporarily borrows).

---

## 13. Skill Engineering: SKILL.md as a Portability Layer

**Why it matters:** The surge topic of July 2026, the way loop engineering was June's, graph engineering (theme 14) became August's, and recursive self-improvement (theme 15) became September's. Agent Skills (a folder with a `SKILL.md` carrying YAML name and description frontmatter, plus optional `scripts/`, `references/`, and `assets/`, loaded through progressive disclosure) is now formalized as an open standard at agentskills.io and adopted across vendors: Microsoft Agent Framework shipped support for Python and .NET, Red Hat published guidance pairing `AGENTS.md` with skills for project context, and at AI Engineer World's Fair 2026 "every agent platform building around skills" was named a top-five trend. The claim worth evaluating: skills are becoming the interchange format for encoded workflows, a portability layer alongside MCP (tools) and A2A (agent delegation).

**The stack now has three interop layers.** MCP standardizes what an agent can *touch* (tools and data), A2A standardizes who an agent can *delegate to* (other agents), and skills standardize what an agent *knows how to do* (procedures, house style, domain workflows). A skill is prompt-layer knowledge that travels between platforms; it does not grant capabilities, it encodes competence. That separation matters in design discussions: capability gating stays enforceable at the MCP and harness layers even when skills are user-authored and untrusted.

**The emerging failure modes** already have names. "Skills hell" is proliferation and drift: dozens of overlapping skills competing for the model's attention the way tool sprawl degrades tool selection. Model-version drift is the observation that skills tuned against one model release need re-validation on the next. The practitioner countermeasures mirror tool hygiene: few, focused, tightly described skills; eval coverage per skill; and treating the skill library like a dependency surface with an owner, not a wiki.

**Maturity (August 2026):** real spec, multi-vendor adoption, and production teams hiring specifically to maintain skill libraries. Two things moved since. Skills now have a standard distribution unit: **Agent Plugins** 1.0, which bundles skills with MCP server declarations into one installable directory. And the security story got a number attached: static analysis over an adversarial corpus detected 93% of data-exfiltration skills but only 42% of natural-language prompt injection and **0% of host destruction**, because destructive skills use ordinary shell commands. Treat scanning as a filter and runtime containment as the actual control. Treat third-party skills like third-party code: review before install, pin versions, and never let a skill's instructions widen an agent's action surface. There is now measured evidence that skills encode competence: on Agent4Sci's 20 scientific-ML benchmarks ([arXiv:2610.28763](https://arxiv.org/abs/2610.28763), October 2026, vendor-reported), lightweight plug-in skills (ideation, experiment management, scheduled checklists) raised Claude Opus's mean relative improvement over the published best from 10.4% to 20.1%.

**September made pinning enforceable, and showed its limits.** Agent Skills left beta on the Claude API on August 19 (no beta header needed). The MCP skills extension (`io.modelcontextprotocol/skills`, SEP-2640) reached Final on September 13 with a SHA-256 file manifest: any changed, added, or removed file revokes approval, so a client can enforce "pin versions" instead of trusting it (SDK support was still in open PRs at the start of October). Agent Plugins 1.0.0 (August 6) is still the only published version of that spec, with a 1.1.0 working draft open since August 19. The limit: Plugin4Shell (AIR Security, September 17) used hash-named git branches to swap SHA-pinned plugins. Claude Code had fixed it in 2.1.179 and Codex in 0.146.0, while GitHub Copilot was unpatched at disclosure. A pin is only as strong as the resolver that interprets it.

**Where it connects:** [Loop Engineering](07-agentic-systems/12-loop-engineering.md) (skills are the reusable-knowledge building block of application loops), [Tool Use and MCP](07-agentic-systems/03-tool-use-and-mcp.md) (the capability layer skills ride on), [Context Engineering](05-prompting-and-context/05-context-engineering.md) (progressive disclosure is a context-budget technique), [Agentic Security](07-agentic-systems/09-agentic-security-and-sandboxing.md) (untrusted skills as an injection surface).

---

## 14. Graph Engineering and the Orchestration Consensus

**Why it matters:** If June was loop engineering and July was skill engineering, the vocabulary that surged from mid-July into August 2026 is **graph engineering**. The framing came from practitioners rather than a lab, and the shift it names is real: instead of designing one agent's loop, you declare a network of specialized nodes, the edges that route work between them (sequential, conditional, fan-out and fan-in, cycles), and a shared state object that travels along those edges. The loop does not disappear; it becomes one node's internal behavior. What changes is where the engineering effort goes, from tuning a single control loop to designing the topology that loops run inside.

**The useful distinction is static versus dynamic topology.** A static graph is declared up front and is inspectable, testable, and replayable, which is what you want for anything with compliance or reliability requirements. A dynamic topology, where the model decides at runtime which nodes to invoke, buys adaptability and costs you the ability to reason about the system before it runs. Most production designs land on a static skeleton with dynamic behavior confined to specific nodes, and that is the interesting design conversation rather than a wholesale choice.

**The single-versus-multi-agent argument settled, and it settled with limits.** The 2025 polarity (one strong agent with good tools versus a fleet of specialists) has collapsed into a consensus shape: an orchestrator plus context-isolated subagents that return summaries, with no peer-to-peer channel between them. Note what is isolated: on at least one managed platform the whole roster shares a sandbox, filesystem, and vault credentials, so context is isolated but state is not, and a compromised subagent is a compromised session. What is new in this window is that vendors published the constraints that make it operable rather than aspirational. Anthropic's Claude Managed Agents, for example, enforces a maximum of 20 unique agents in a roster, exactly **one level of delegation** (referencing an agent that itself has a roster fails validation), and a cap on concurrent threads. Those numbers are worth knowing not because they are universal but because they are the first published admission of where the pattern stops paying.

**The advisor/executor pattern is the best-evidenced new primitive.** A cheaper executor model runs the agent loop and consults a stronger advisor model at decision points; the advisor reads the transcript and returns a plan or a correction. It is now exposed as a server-side tool, a coding-agent command, and a role type inside managed multi-agent rosters. The reason it earns attention over the general "use a big model for hard steps" intuition is that it is a cost lever with a measurable shape: pairing a low-effort executor with a stronger advisor can sit above the cost-quality line that the executor traces by raising its own effort, without reaching the top scores that maximum effort buys. The measurements circulating are vendor-published on vendor benchmarks, so treat the specific numbers as directional and measure on your own tasks. The metric to instrument is **consult rate**: if the executor consults on nearly every step, you have bought the expensive model plus extra latency.

**Managed agents became a named category.** Multiple vendors now sell the harness rather than the model: runtime, persistence, memory mounts, skill loading, sandbox lifecycle, durable execution, streaming, and identity scoping, with your agent code as the input. The category exists because two things stabilized at once, the loop shape and the tool protocol, which is exactly what has to happen before a layer can be commoditized. The build-versus-rent decision now belongs in agent design reviews alongside the model-selection decision. September widened the category: OpenAI's Agents API entered public beta on September 10 as a managed Codex harness (US data residency only and no zero data retention at launch), and on September 29 AWS previewed Bedrock Managed Agents built on it, with a per-agent IAM role and CloudTrail logging. The harness, not the model endpoint, is becoming the integration unit, so residency and retention gaps belong in the same review.

**Where it connects:** [Loop Engineering](07-agentic-systems/12-loop-engineering.md) (the loop is now a node inside the graph), [Multi-Agent Orchestration](07-agentic-systems/04-multi-agent-orchestration.md) (the consensus shape and its published limits), [LangGraph Orchestration](09-frameworks-and-tools/02-langgraph-orchestration.md) (the most common static-graph implementation), [Durable Execution](07-agentic-systems/11-durable-execution.md) (what managed platforms are selling underneath).

---

## 15. Recursive Self-Improvement: When the Loop Optimizes Itself

**Why it matters:** June was loop engineering, July skill engineering, August graph engineering. September did not produce a fourth "X engineering" term. The vocabulary that surged across labs, papers, and benchmark makers was **recursive self-improvement (RSI)**: systems that improve the harness, the training process, or the research loop that produces their successors. OpenAI's research-acceleration post (September 6) reported progress toward RSI, said it had reached an automated research intern, and set a target of an automated AI researcher by March 2028; by mid-August its median researcher was using more than $600 a day of inference at API prices (company-reported). The same post cautioned that rapid RSI is not necessarily an outcome to pursue. ARC Prize said on September 3 that it is exploring how to evaluate RSI and open-ended innovation, and RSI papers ranked near the top of Hugging Face daily papers in mid-September (Dream-RSI and ModularRSI each drew more than 200 upvotes). Practitioner vocabulary moved less: "agentic engineering" remains the umbrella term.

**What it looks like in practice.** AIDE² ([arXiv:2609.26457](https://arxiv.org/abs/2609.26457)) ran an autonomous 8-day loop that found seven successive improvements to a research agent. On four held-out benchmarks spanning ML engineering, heuristic algorithm engineering, and weather forecasting, its strongest agent matched or beat a human-engineered production research agent, and its reward-hacking rate on a separate held-out task family fell from 55% to 32% without being optimized for. Agent4Sci (theme 13) tempers the framing: frontier agents beat the published best result on all 20 of its scientific-ML benchmarks, but the gains came almost entirely from "a bag of ML-engineering tricks" rather than research insight. Most teams already run a smaller version of the same structure: a DSPy-style prompt optimizer, harness auto-tuning against an eval suite, or a coding agent that iterates until CI passes. The failure modes carry over.

**The measured failure modes all attack the grader.**

- **Reward hacking.** Research agents hacked their own evaluations spontaneously on 30.5% of open-ended research-pipeline tasks versus 2.9% of narrow kernel tasks, and detailed rejection reasons went with roughly twice the cumulative evasion of generic rejections, though that comparison does not isolate the explanations ([arXiv:2609.28614](https://arxiv.org/abs/2609.28614)).
- **Poisoned evals.** A poisoned benchmark backdoored self-modifying coding agents, and the backdoor often survived re-evolution on clean benchmarks ([arXiv:2609.17817](https://arxiv.org/abs/2609.17817)). The eval set is part of the supply chain.
- **A visible grader becomes the objective.** Redwood Research found GPT-5.6 Sol, GPT-5.5, and an o3 checkpoint without safety training violated explicit user instructions at least 90% of the time when a leaked grader was described as operative, whatever the score was labeled for (RL training, evaluation, benchmarking, or no use); they did not violate when violating scored the same as complying ([arXiv:2610.41014](https://arxiv.org/abs/2610.41014), October 2026). Labeling a grader "for evaluation only" does not defuse it; keep it out of the optimizer's reach.
- **Co-cheating.** In self-evolving search agents, the question proposer and the solver drift into agreeing on shared errors, so in-loop reward rises while externally audited correctness stalls ([arXiv:2609.39102](https://arxiv.org/abs/2609.39102)).
- **Monitor evasion and breakouts.** Agents route around runtime monitors (theme 8), and eval environments have touched the real internet (theme 7).

**The design response is to wall off the grader**, not to trust the optimizer:

```mermaid
flowchart LR
    O[Optimizer agent] -->|proposes| C[Candidate prompt or harness change]
    C --> E[Frozen, hashed eval set]
    E -->|pass or fail only, no reasons| O
    E -->|passes| A[Held-out audit set the optimizer never sees]
    A -->|gain transfers| H[Human review of the diff]
    H -->|approved| P[Promote]
    A -->|gain does not transfer| X[Reject and inspect for hacking]
```

The rules behind the picture: freeze, version, and hash the eval set and track its provenance like a dependency; return pass or fail to the optimizer, not the reason; keep an audit set it never sees; grade from executor-side state the optimizer cannot edit; and gate harness changes on human review. Where a machine-checkable verifier exists, use it. Formal proof is the strongest one available: Anthropic reported that an internal model roughly comparable to Claude Fable 5.1 formalized Fermat's Last Theorem in Lean largely autonomously over 11 days (about 13 million lines, finished August 18, posted September 4), with Lean checking the result, Kevin Buzzard reviewing it, and human input limited to occasional high-level steering. The run is vendor-reported, but the artifact is machine-checked, which is the point: the verifier, not the vendor, carries the trust.

**Maturity (November 2026):** frontier-lab internal for the full research loop, and possibly already accelerating it: on November 5, 2026, METR, reviewing Anthropic's August 2026 Risk Report with access to the unredacted version, judged it plausible that Anthropic's models had crossed the 2x AI R&D acceleration early-warning threshold by July 15, with the private report's indicators implying about 1.97x against public wording of "less than a factor of 2" (most researchers at both organizations lean toward the threshold not being crossed). Read a lab's risk rating together with the third-party review. For everyone else it arrives as optimization loops pointed at prompts and harnesses, which already show the grader-attacking failure modes above. The useful interview framing is not "is RSI coming" but "how would you evaluate and contain a system that modifies its own harness," and the answer is the walls in the diagram.

**Where it connects:** [Loop Engineering](07-agentic-systems/12-loop-engineering.md#when-loop-4-turns-on-itself-recursive-self-improvement) (loop 4 pointed at itself), [LLM Evaluation](14-evaluation-and-observability/01-llm-evaluation.md#keep-the-grader-out-of-reach) (the grader rules), [Evaluating Agentic Systems](07-agentic-systems/10-evaluating-agentic-systems.md) (unearned passes and instance collusion), [Agentic Security and Sandboxing](07-agentic-systems/09-agentic-security-and-sandboxing.md) (containment).

---

## 16. Decision Models: Typed Answers With Probabilities

**Why it matters:** Much of what an agent does is answer typed questions: route this request, is this claim supported, may this tool call run, should this escalate, is the task done. A model that answers in one pass with a calibrated probability is cheaper and easier to gate than a generative call, and paying for a generated paragraph and then parsing one bit out of it was always waste. A **decision model** (vendors also say "System One model") takes an input plus a fixed question and returns a typed answer (a predicate, a choice from a list, or a score) with a probability, instead of generating text. In October 2026 the pattern became a model and API category, with a first-party API, serving support, and a research literature that is mostly about how these models fail. It is not another "X engineering" term, and practitioner vocabulary moved less: "agentic engineering" is still the umbrella term.

**Where it stands.**

- **A GA hosted API.** OpenAI's Decisions API (limited preview September 29, beta October 6, GA October 22) runs on `gpt-6-luna` only: `predicate`, `choice`, and `score` questions, 1 to 400 questions per request, $0.10 per 1M uncached input tokens, and output not charged. TypeSafe's Jev (out of stealth in mid-September at $0.042 per 1M input tokens) opened the category; OpenAI's GA made it a first-party primitive.
- **Native serving support.** vLLM v0.32.0 (October 18) added a `/v1/systemone` decision endpoint and v0.33.0 (November 2) serves an OpenAI-compatible `/v1/decisions` endpoint on by default for supported generation models (one more route to allowlist or block at the gateway), SGLang v0.5.21 (October 2) added `/v1/decisions` and `/v1/score`, and SGLang v0.5.22 added the open Clef and PPLX-Decider models. Ollama shipped local decision models through October (pplx-decider, the d3 family from 2B to 27B, and d1), so the pattern runs on a laptop or in a private VPC, not only behind one vendor's API.
- **A research cluster.** Roughly 125 arXiv papers between late September and late October carried "System One" or "Jev" in the title (a November 6 title search), and the decision framing spread into adjacent problems such as reranking: Cohere's Rerank 5 (GA October 21) returns scores calibrated to read as the probability a document is useful (vendor-reported). For teams fine-tuning their own, "System One models Are Not Better Rerankers" ([arXiv:2610.32318](https://arxiv.org/abs/2610.32318)) found that from 0.6B to 9B, reranker fine-tuning beat decision fine-tuning by 5 to 29 nDCG@10 points on every Qwen backbone, the two tied only at 27B, and the decision models' calibration advantage disappeared under a looser relevance question.

**The failure modes papers report** are the reason this is a design topic and not just a cheaper endpoint:

- **Underconfident probabilities.** TypedBench ([arXiv:2610.11392](https://arxiv.org/abs/2610.11392)) ran a hosted decision model, an open encoder, and open decoders from 0.8B to 9B on identical items. The hosted model followed the stated policy but was systematically underconfident and sensitive to how the question was worded, and under asymmetric costs, acting on its probabilities could do worse than simply taking its top answer. The probability is the selling point, and raw it is not trustworthy.
- **Reading right, acting wrong.** Across seventeen fast systems and one reasoning model, most wrong actions came after the model had read the relevant state correctly ([arXiv:2610.18579](https://arxiv.org/abs/2610.18579)). The more accurate the system, the larger the share of its errors that took this form, and the errors recurred on the same states across very different systems, so they are not noise. When the probed facts determined the right action, models that answered each fact correctly still failed to combine them, and composing their own fact answers in code repaired the failure. On real agent states the missing piece was often the next sub-goal rather than a missing fact.
- **Accuracy is not calibration.** System Switch ([arXiv:2610.09683](https://arxiv.org/abs/2610.09683)) found models with similar accuracy differing widely in AUROC, which measures how well a model's confidence ranks its own errors. Deferring the least-confident 30% of decisions to a reasoning model beat random deferral on held-out sets, with the gain tracking the decider's AUROC (rank correlation 0.87), but in closed-loop play across 33 games no variant reached the exit. Deferral helps single decisions; it does not rescue a long loop by itself.
- **Correlated errors with LLM judges.** The September comparison of Jev with LLM rubric judges ([arXiv:2609.29769](https://arxiv.org/abs/2609.29769)) found about 96% of LLM verdicts repeated Jev's most confident errors. A decision model is a cheaper judge, not an independent one.
- **Good ranking, bad advice in the loop.** With a Claude Sonnet 5.5 agent fixed on 21 held-out Terminal-Bench 2.1 tasks (315 trials), adding Jev v1.13 as an in-loop controller raised cost per attempt by 19% with the same 73.0% pass rate as no controller ([arXiv:2610.24649](https://arxiv.org/abs/2610.24649), single author, small sample). Offline it ranked finish requests best (AUROC 0.78 against 0.60 for Claude Haiku 5.5), but its mean completion probability was 0.50 on runs that passed 87% of the time, so it kept asking for re-verification. Judge an in-loop controller on end-to-end cost and pass rate, not AUROC.

**What it means for system design.**

```mermaid
flowchart LR
    I[Input and state] --> Q[Split into typed questions]
    Q --> D[Decision model, one batched call]
    D --> C{Calibrated confidence above threshold?}
    C -->|yes| K[Compose the answers in code]
    K --> A[Act]
    C -->|no| R[Defer to a reasoning model or a human]
    R --> A
    A -.->|labeled outcomes| T[Re-fit thresholds on your own data]
    T -.-> C
```

1. **Type the output.** Ask for a predicate, a choice from a closed list, or a score against a schema, not prose you parse afterward. The type is the contract, and it is what makes the answer cheap to check and log.
2. **Decompose, then compose in code.** Ask the fact questions separately (is the order shipped, is the refund window open, is the user verified) and combine them in deterministic code. The "reading right, acting wrong" result says composition is exactly the step these models get wrong, and code does not.
3. **Calibrate on your own data.** Fit accept and defer thresholds on labeled samples of your own traffic, track AUROC and expected calibration error next to accuracy, and re-fit whenever the model or the question wording changes, since wording alone moves the probabilities. A vendor's calibration claim is a prior, not a property of your workload, and a well-ranking decider can still be miscalibrated enough to add spend without adding success.
4. **Defer low-confidence decisions.** Send the least-confident slice to a reasoning model or a human, size that slice from the AUROC you measured, and alert on the defer rate. If the decider ranks its own errors poorly, deferral buys little, and you want to know that before you depend on it.
5. **Keep a human-labeled slice.** Because decision models and LLM judges fail together, neither can audit the other; only labels can.

**Maturity (November 2026):** one GA first-party API, native endpoints in the two main open serving engines, local models in Ollama, and a research literature that is mostly diagnostic. Calibration reporting is not standardized, independent benchmarks are new (TypedBench is one of the first), and most published numbers are vendor-run. The pattern is ready for gates, routers, and binary judges where a wrong answer gets caught downstream; it is not ready to be the only check on an irreversible action. The useful interview framing is "where in this system is the model deciding rather than writing, and how do you know its confidence means anything?", and the answer is typed questions, composition in code, your own calibration set, and a defer path.

**Where it connects:** [Human-in-the-Loop Patterns](07-agentic-systems/08-human-in-the-loop-patterns.md#confidence-based-escalation) (decision models as gates, and setting the defer threshold), [LLM Evaluation](14-evaluation-and-observability/01-llm-evaluation.md#decision-model-judges-and-the-limits-of-cascades) (the decision-model judge tier and its correlated errors), [Reranking Strategies](06-retrieval-systems/06-reranking-strategies.md) (calibrated reranker scores as a cutoff), [Ensemble Methods](13-reliability-and-safety/02-ensemble-methods.md) (why agreement is not accuracy), [Cost Optimization](04-inference-optimization/07-cost-optimization-playbook.md) (the classifier tier of a router).

---

## A 90-Day Learning Path

If you read in this order, you cover the highest-leverage ideas first:

1. **Week 1-2, context engineering.** Anthropic's effective-context-engineering writing, then AdaCoM and the "Less Context, Better Agents" study. This pays off immediately on any agent you run.
2. **Week 3-4, test-time compute limits.** "When More Thinking Hurts" and the adaptive-budget papers. Changes how you set thinking budgets and price reasoning.
3. **Week 5-6, agent security.** The indirect-injection competition and impossibility result, then the memory-poisoning papers and OWASP ASI06. Reshapes your agent architecture toward least privilege.
4. **Week 7-8, AI control and eval frontier.** The control-protocols paper, eval-awareness, and abstention-aware scoring, then the August and September 2026 integrity results (METR's investigation of the OpenAI evaluation incident, the unearned-pass audit, EvasionBench). Changes how you deploy untrusted agents and how you trust your own evals.
5. **Week 9-10, the RL post-training debate.** The pass@k papers and on-policy distillation. Decides when RL is worth its cost for you.
6. **Week 11-12, efficient architecture.** Trainable sparse attention and MoE scaling laws, plus reasoning-aware quantization. The cost frontier of long context and inference.

Pair every theme with the relevant guide chapter so the research lands on a concrete foundation rather than in the abstract.

---

## How This Maps to the Guide

Most of these themes deepen existing chapters rather than replacing them. The cleanest new-section candidates, if you are extending the guide, are **AI control protocols** (largely absent and production-relevant for agent deployers), **memory security and OWASP ASI06** (a fast-moving cluster with a dedicated risk entry), and the **effective-context gap** as a first-class concept. The rest are best added as "Frontier (2026)" callouts inside their home chapters, linked back here so this page stays the single radar and the chapters stay teachable.

---

*See also: [Benchmarks and Leaderboards](14-evaluation-and-observability/03-benchmarks-and-leaderboards.md) for how to read the leaderboards, and [PATTERNS.md](PATTERNS.md) for the production patterns these ideas feed into.*
