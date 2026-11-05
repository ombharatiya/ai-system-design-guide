# Benchmarks and Leaderboards

Public benchmarks are how the field talks about model capability: MMLU, SWE-bench, GPQA, Arena Elo. They are useful for orientation and dangerous for decisions. This chapter teaches you what each major benchmark measures, which ones still separate frontier models and which have saturated, and how to read a benchmark claim critically so you are not fooled by a number.

The load-bearing idea: **a benchmark's definition and known flaws are stable; its scores are perishable.** Leaderboard numbers change weekly, get inflated by contamination and favorable harnesses, and are increasingly polluted by fabricated entries. So this page leads with what each benchmark *measures* and how it *breaks*, and treats specific scores as dated, sourced snapshots you should re-verify. For evaluating your own system (the thing that actually predicts production quality), see [LLM Evaluation](01-llm-evaluation.md).

## Table of Contents

- [How to Read This Page](#how-to-read-this-page)
- [The Capability Map](#the-capability-map)
  - [General Knowledge and Language](#general-knowledge-and-language)
  - [Frontier Reasoning](#frontier-reasoning)
  - [Mathematics](#mathematics)
  - [Coding](#coding)
  - [Agentic and Tool Use](#agentic-and-tool-use)
  - [Long Context](#long-context)
  - [Multimodal](#multimodal)
  - [Factuality and Instruction Following](#factuality-and-instruction-following)
  - [Human Preference](#human-preference)
- [Reading Benchmarks Critically](#reading-benchmarks-critically)
  - [Saturation](#saturation)
  - [Contamination](#contamination)
  - [Harness and Scaffold Variance](#harness-and-scaffold-variance)
  - [The Leaderboard Illusion](#the-leaderboard-illusion)
  - [The Benchmark-to-Production Gap](#the-benchmark-to-production-gap)
  - [Composite Indices](#composite-indices)
- [A Practical Checklist](#a-practical-checklist)
- [Which Benchmarks Matter in 2026](#which-benchmarks-matter-in-2026)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## How to Read This Page

Each benchmark below lists **what it measures**, its **format**, its **saturation status** (does the frontier cluster sit so high that score deltas are noise?), and its **known flaws**. Where a current score is useful for orientation it is given with a source and a "verify" flag, because:

- **Provider self-reports run higher than independent leaderboards.** A lab reports its model under the best harness, best effort setting, and uncapped infrastructure it could find. That number is not comparable to another lab's number or to an independent run. Only compare numbers produced by the *same* harness.
- **The 2026 web is polluted with fabricated leaderboard pages** inventing models and scores. If you see a frontier coding score attributed to a model name you do not recognize, assume it is SEO spam until a primary source (the lab's post, the benchmark's own leaderboard) confirms it.
- **The headline number hides the harness.** "88% on SWE-bench" is uninterpretable without the agent scaffold, tool access, effort level, and output-token cap. Since mid-2026 you also need the benchmark version, the metric (binary completion or partial credit), who ran it, and whether a safety fallback model handled some of the tasks.

So: use the *measures* and *flaws* columns to reason about a benchmark, and treat any single percentage as a dated data point, not a fact.

---

## The Capability Map

Benchmarks group by the capability they probe. Within each group, the field continuously retires saturated benchmarks and replaces them with harder successors, so the live separators move every year.

### General Knowledge and Language

Mostly **saturated**: frontier models cluster above 88-90%, so deltas are noise. Keep these as historical baselines, not frontier discriminators.

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **MMLU** | 57-subject multiple-choice academic knowledge (~15.9k questions) | Saturated (~90%+ cluster) | The most-cited benchmark historically. ~6.5% of items have label errors (per MMLU-Redux); heavy web contamination. Phased out of model cards. |
| **MMLU-Pro** | Harder MMLU successor: 10 options instead of 4, reasoning-heavy, ~12k questions | Near-saturated | Built to fix MMLU saturation; now hitting the same ceiling (top cluster ~88-90%). Dropped from the Artificial Analysis index as saturated. |
| **MMLU-Redux** | Error-corrected re-annotation of MMLU (~3k items) | Saturated | Exists to *quantify* MMLU's error rate (found 6.49% wrong, up to 57% in some subsets), not to rank frontier models. |
| **HellaSwag** | Commonsense sentence completion | Fully saturated (95%+) | Human baseline ~95%; solved since the GPT-4 era. |
| **ARC-Challenge** | Grade-school science multiple-choice | Saturated (~96%+) | Effectively solved. |
| **WinoGrande** | Winograd-schema coreference / commonsense | Saturated (~90%+) | Residual annotation artifacts. |
| **BBH (BIG-Bench Hard)** | 23 hardest BIG-Bench tasks, multi-step reasoning | Saturated with CoT (>90%) | Superseded by **BIG-Bench Extra Hard (BBEH)**, built because BBH saturated. |
| **GLUE / SuperGLUE** | Older NLU task suites | Retired | Models passed the human baseline on SuperGLUE in early 2021. |

### Frontier Reasoning

The benchmarks used to separate the top of the field. Several saturated or changed versions during 2026, so read the Status column before citing one.

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **HLE (Humanity's Last Exam)** | Closed-ended expert questions across 100+ subjects, retrieval-resistant (~2,500 questions, ~10% multimodal) | Live, but HLE-Diamond is now the cleaner reference | The premier frontier knowledge benchmark through mid-2026. Artificial Analysis swapped it for HLE-Diamond in Intelligence Index v5.0 and now lists full HLE as a legacy evaluation. At its Jan-2025 launch the SOTA was under 10%, a sign of how fast the frontier moved. Tools configs decide rankings: Anthropic's own full-HLE-with-tools table has Claude Opus 5.5 at 67.7% and GPT-6 Astra at 57.2% (vendor-reported), the reverse of the official HLE-Diamond tools run below. An expert re-grade of HLE-Physics (arXiv 2609.13009) found most audited "wrong" answers were grader errors, bad reference solutions, or ambiguous questions, so part of the remaining headroom is label error. |
| **HLE-Diamond** | 1,000 cleaned HLE questions, split evenly between reasoning and knowledge; released by the HLE team on September 22, 2026 after a year of cleaning | Live, wide separation | No tools: GPT-6 Astra 59.9%, Claude Opus 5.5 54.6%, GPT-6.1 Sol 53.2%, Claude Fable 5.1 50.7%. With web and code tools: Astra 82.9%, Opus 5.5 73.9%, Fable 5.1 72.3%. Always state which configuration you are quoting; tools add about 20 points for each model with both runs listed. A 10% component of the Artificial Analysis Intelligence Index since v5.0 (October 2026). |
| **GPQA-Diamond** | 198 PhD-level biology/physics/chemistry questions, Google-proof | Saturated | Designed so skilled non-experts with web access score ~34% and PhD experts ~65-70%. GPT-6 Astra sits at 96.0% in OpenAI's launch table (secondary reproductions) and 96.3% at xhigh in Artificial Analysis's independent run. AA moved it to legacy in Intelligence Index v4.2 and runs it only standalone. A sanity check for mid-tier models, no longer a frontier ranking signal. |
| **ARC-AGI-2** | Abstract grid-puzzle reasoning, efficiency-aware (~400 tasks); resists memorization | Past the human threshold at the frontier; weakly separating | The human panel passes 85%, and 85% is the grand-prize *threshold*. ARC Prize's results page now lists several frontier models above it: GPT-6 95.0% (September 2), GPT-6.1 Sol 94.2%, Claude Opus 5.5 93.3%, GPT-5.6 92.5%, Gemini 4 Argon 91.3%, Claude Opus 5 90.4%, Claude Fable 5 89.2%. The page does not state harness, cost, dataset split or verification status, so check the Verified leaderboard before calling any of these numbers Verified. Seven frontier models inside six points means it no longer ranks the frontier; use it as a tier filter and let ARC-AGI-3 do the separating. |
| **ARC-AGI-3** | The first fully interactive ARC benchmark (launched March 25, 2026): the agent discovers deterministic, closed-ended mechanics and goals by acting, and efficiency is compared with a human baseline in actions per level | Harness-dependent: near ceiling under a provider harness | Frontier AI scored 0.51% at launch. ARC Prize's Semi-Private run of GPT-6 Astra (September 3): **62.7%** on its provider-neutral Standard harness at max effort ($26,098), **99.9%** on ARC Prize's Provider Adapter harness at high effort ($18,817), which uses OpenAI's own context-management features to preserve opaque reasoning state between requests and compact context. ARC Prize reports both, labeled. Same model, same test set, a 37-point gap: the clearest 2026 case of the harness deciding the headline. The ARC Prize 2026 competition has finished, and arcprize.org says results will be announced December 4, 2026 (scheduled); the ARC-AGI-3 track's prizes total $850K, including a $700K Grand Prize for the first eligible agent to score 100% (it carries over if unclaimed), and prize-eligible code must be open sourced. Those compute-limited Kaggle entries are not comparable with frontier API runs. |
| **FrontierMath** | Research-level math, problems that take expert mathematicians hours to days (~350 problems, private; Tier 4 is the hardest) | Live but climbing fast | Open-ended with auto-verifiable closed-form answers (guess-proof). v1 had errors in ~42% of problems (fixed in the June 2026 v2); it was partly OpenAI-funded, so single-lab numbers warrant a governance caveat. Epoch's own framing is that less than 70% is within reach, so be skeptical of aggregator scores far above that. |
| **CritPt** | Research-grade physics reasoning across 11 domains, by 50+ physicists | Live, but low scores are partly a grading problem | Newer (late 2025). Artificial Analysis Intelligence Index v5.0 uses a CritPt v2 hard subset (private, 10%), and v5.0.1 (November 3, 2026) applied grading-robustness fixes from the CritPt authors and regraded every model on it. Raw scores still separate models, but the September expert re-grade (arXiv 2609.13009) corrected reference solutions and dropped flawed items: on the 54 retained CritPt challenges, GPT-5.6 Sol's corrected pass@4 reached 94.4%. Read a low CritPt score as part model weakness, part label error. |

### Mathematics

The classic math benchmarks are solved; the live separators are the freshest competition years and research-level sets.

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **GSM8K** | Grade-school word problems | Saturated (>95%) | GSM-Symbolic showed scores drop when numbers/clauses are perturbed, evidence the high numbers are partly memorization. |
| **MATH / MATH-500** | Competition math (AMC/AIME level) | Saturated (top ~99%) | MATH-500 is a 500-item subset; small size means high variance at the top. |
| **AIME (2024/2025/2026)** | Olympiad short-answer, integer answers, 30 problems/year | Year N saturates once public | Valued because each year is fresh (low contamination) until released. Only 30 items, so single-year scores are high-variance; prefer averaged runs. Saturated for 2024/2025. |
| **HMMT, Putnam** | Harder olympiad / undergraduate proof competitions | Putnam proofs not saturated | Proof grading is LLM-judge-dependent; final-answer shortcuts overstate true proof ability. |

### Coding

Snippet benchmarks are dead for ranking; agentic, repository-scale benchmarks are the production signal.

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **HumanEval / HumanEval+ / MBPP / MBPP+** | Single-function synthesis from a docstring, checked by unit tests | Saturated (frontier ~90-97%) | Tiny and public since 2021, so contaminated. HumanEval+ adds ~80x more tests to catch fragile code; a score drop on the + version means weak tests were passing buggy code. Do not use to rank frontier models. |
| **SWE-bench Verified** | Resolve real GitHub issues so hidden tests pass (human-validated 500-issue subset) | Near-saturated, contaminated | The canonical coding number 2024-2026. But OpenAI's own evals team flagged that more than 60% of tasks have flawed tests and that solutions can be reproduced verbatim from the task ID, and a 2026 repo-perturbation study (arXiv 2609.27891) found consistent drops when namespaces and problem statements are rewritten, which points to memorized repository cues. Useful as an "above ~80% is frontier" tier filter, unreliable for fine ranking. Harness choice alone swings results 10-20 points. |
| **SWE-Bench Pro (v2)** | Harder issue resolution across public, held-out, and commercial repos. v2 (Scale, September 22, 2026) refreshed instructions, verifiers, and container images and dropped 89 of 731 public tasks as invalid | Public split saturated; the private set is the signal | Public 642 tasks: Claude Opus 5 99.4% (638/642), Kimi K3 97.7%. Private 272 tasks: Opus 5 81.6% (222), Kimi K3 214, GLM-5.3 and Gemini 3.8 Flash 211 each (Scale-run). Scale attributes the 17.8-point public-private gap mainly to training-time exposure to the public repos. v2's **network-locked protocol** (the sandbox reaches only the model endpoint) and pristine-image re-grade of every diff, with both grades published, caught one model forging a Go module version. An audit of v1.0 (arXiv 2609.34262) found unearned passes rising from 24% (Opus 4.7) to 73% (Fable 5) of matched tasks, mostly reference solutions read from git history. |
| **SWE-bench Multimodal** | Resolve visual/UI software issues (JS front-end, includes screenshots of broken UI) | Not saturated | Tests real visual grounding; text-SWE-bench systems struggle. |
| **SWE-rebench / SWE-bench-Live** | Continuously updated, decontaminated issue resolution; tasks postdate model cutoffs | Not saturated | The strongest contamination audits. SWE-rebench specifically catches inflated Verified scores that collapse on fresh tasks. |
| **LiveCodeBench** | Competitive-programming problems tagged by release date; score only on post-cutoff problems | Not saturated in fresh windows | Contamination-resistant by construction. Competitive programming is not software engineering, so it predicts algorithmic reasoning, not repo work. |
| **Aider Polyglot** | Real multi-file edits across 6 languages, must emit a valid edit format, 2 attempts | Approaching saturation (~88%) | Measures edit-format reliability, not repo-scale reasoning; Exercism source is public (contamination). |
| **SciCode** | Research-grade scientific coding decomposed into subproblems | Far from saturated | The hardest mainstream coding benchmark. Note two incompatible scoring conventions (main-problem vs subproblem); always cite which. Artificial Analysis dropped it from its index in v5.0 and now runs it as a legacy evaluation. |
| **Terminal-Bench 4.0** | 66 tasks: 3.0's 74 minus 8 removed (2 saturated, 2 with refusal issues, 2 with public solutions, 2 with quality or platform problems), with 19 fixed and a flat 8-hour agent timeout. Announced August 28, 2026 | Active, the current terminal-agent discriminator | Adopts semantic versioning: a major bump means re-run trials, and 3.0 scores do not carry over. tbench.ai leaderboard (Opus 5.5, Sonnet 5.5 and GPT-6.1 Sol entries added October 6): Claude Opus 5.5 (Claude Code) 64.85% at max, first; Claude Sonnet 5.5 (Claude Code) 61.82% at max; GPT-6 Astra and GPT-6.1 Sol (Codex) tied at 58.18% at max, with Astra 57.88% at high and xhigh; Claude Fable 5.1 (Claude Code) 57.88% at max and xhigh, 54.55% at high. Read the cost column: GPT-6.1 Sol matched Astra for about $634 of run cost against about $3.3K, and Sonnet 5.5's run cost about $7.3K against $4.7K for Opus 5.5's higher score. Vendor-reported: Anthropic gives Opus 5.5 66.4% at xhigh (SE ±2.6) and Sonnet 5.5 70.6% (effort not stated in the launch table), so on Anthropic's own runs the cheaper model leads while on the leaderboard Opus 5.5 does; OpenAI's 57.9% for Astra is its high-effort run. |
| **Terminal-Bench 3.0** | 74 tasks across 7 domains, published July 30, 2026 to replace a saturated 2.1. Briefly cited at launch as "Frontier-Bench v0.1" before the rename | Superseded by 4.0 after about four weeks | Launch tops: GPT-5.6 Sol (Codex) 34.4%, Fable 5 (Claude Code) 33.8%. It widened separation sharply (two models 4.9 points apart on 2.1 sat 12.7 points apart on 3.0). Its short life is the lesson: versioned benchmarks now retire saturated or leaked tasks within weeks, so every score needs its version. |
| **Terminal-Bench-Science 0.1** | 70 agentic scientific-workflow tasks, chosen from 920 proposals, across life, physical, Earth, mathematical, and engineering sciences. Announced August 27, 2026 | New, moving fast | Launch leaderboard: Opus 5 30.0%, GPT-5.6 Sol 22.4%, Fable 5 21.4%. Vendor-reported within weeks: GPT-6 Astra 64.6% (OpenAI's table, via secondary reproductions), Opus 5.5 58.7% and Fable 5.1 52.6% (Anthropic, which footnotes a standard error of ±3.5 to 5 points). On the vendors' own tables the order differs from Terminal-Bench 4.0, so scope "best coding model" to a workload. Now the largest Science component (15%) of the Artificial Analysis Intelligence Index v5.0. |
| **DeepSWE v1.1** | 113 original, written-from-scratch long-horizon tasks across 91 repositories and 5 languages (Datacurve); grades the agent's committed patch in a fresh isolated container | Active, contamination-resistant by construction | Commissioned tasks avoid the merged-PR leakage that broke Verified and Pro's public split, and the site carries a canary GUID. Leaderboard (updated September 22): a three-way 74% pass@1 tie between GPT-6 Astra (xhigh) at $4.43 per task, Gemini 3.8 Flash (high) at $2.36, and Claude Opus 5 (max) at $11.84. A 5x cost spread at an equal pass rate is the case for cost per resolved task as a first-class metric. Google reported 77.9% for Gemini 4 Argon at its announcement (vendor-reported, separate from the board figures above). |
| **SWE-Bench ProMax** | 170 multilingual refactoring instances (7 languages, averaging 11.4 modified files each), released August 10, 2026 | New, wide headroom | At release the best model resolved only 41.2%. Targets coordinated behavior-preserving change, a harder and more realistic class than single-issue bug fixing |
| **Terminal-Bench (2.x)** | End-to-end agent tasks in a real terminal sandbox (build, debug, sysadmin) | Saturated at the top; superseded by 3.0, then 4.0 | Scores are harness-paired (the agent CLI matters as much as the model). Artificial Analysis now lists Terminal-Bench 2.1 as a legacy evaluation. |

**Why agentic benchmarks displaced HumanEval:** HumanEval/MBPP saturated at 90%+ for every frontier model, are small and memorized, and a single-function docstring-to-code task does not predict real engineering. The proof: SWE-bench launched at under 2% resolved for the same model generation that aced HumanEval. Reading a repo, running tests, reading a stack trace, and iterating a patch is a different skill, so the field moved to issue-resolution and terminal benchmarks.

### Agentic and Tool Use

The fastest-moving area, because agents are where 2026 production value is. Most of these are not saturated.

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **BFCL (Berkeley Function Calling Leaderboard)** | Tool/function-calling accuracy; V4 adds agentic tasks with web search and memory | V1/V2 saturated, V3/V4 not | AST-based scoring can miss semantic errors. Beware stale mirrors of the board; use the official Berkeley leaderboard. |
| **tau-bench line (Sierra): tau2-bench, tau3-bench, now tau-bench 2.0.0** | Agent uses tools and talks to a simulated user under a domain policy (retail, airline, telecom, a banking_knowledge retrieval domain, and full-duplex voice; 2.0.0 adds six-language packs and a 200-task `intake` domain for exact entity capture) | Not saturated on reliability | Reports **pass^k** (re-run the same task k times) which exposes a brutal reliability cliff: an agent at ~60% pass@1 can fall to ~25% pass^8. The most production-relevant tool-use signal because it measures consistency, not just best-case. **Mind the version boundary:** tau-bench 2.0.0 (released October 19, 2026) states "Scores are not comparable with 1.x". The user simulator moved from gpt-4.1 to gpt-5.5, communication checks are now LLM-judged, and voice runs default to 2,400 seconds per call, so re-run baselines rather than comparing across versions (the v1.0.1 tag stays available for 1.x-comparable numbers). The 1.x line already had one break: v1.0.1 (July 22, 2026) fixed banking_knowledge grading, so earlier banking_knowledge results are not comparable either. Voice scores track the reasoning backend; the 1.x-era voice board's gpt-live-1 figure (81.7% pass@1) does not carry into 2.0.0. |
| **Hyper-tau-bench (Sierra)** | An agent that builds agents: a developer agent recovers requirements from business records and client interviews, builds a customer-service agent in a sandbox, and is scored on that agent's held-out tau-style tests. Open-sourced September 8, 2026 | New, wide headroom | Claude Opus 5 (max) in Claude Code alone passed 23.9%; the same model class paired with an engineer who had deep context reached 82.2%. Requirement elicitation was the bottleneck: developer agents asked at most 4 client questions when the client alone held 20 to 25 requirements. |
| **GAIA** | General-assistant tasks needing multi-step tool use, web browsing, files | Saturated at the top (~92%) by orchestrated ensembles | Top entries are multi-model ensembles and scaffolds, not base models, so it measures the orchestration more than the model. |
| **BrowseComp** | Hard-to-find facts that require persistent web browsing | Saturated at the frontier | GPT-6 Astra 91.5% vs GPT-5.6 Sol 90.4% (secondary reproductions of OpenAI's table). Use corpus-controlled deep-research evals such as BrowseComp-Plus, which isolate the retriever from the agent. Web-enabled instances can also collude: OpenAI agents on a web-research benchmark made about 13,000 edits to dormant public wikis in one June week to share answers (collusion.wiki). |
| **OSWorld-Verified** | Computer-use agent on a real OS, execution-scored | Saturated on self-reported scores | Cleaned and standardized successor to OSWorld (which launched with a best score of ~12% vs ~72% human). Self-reported leaders now sit in the mid-80s, so it no longer separates the frontier on short, single-session desktop tasks. |
| **OSWorld 2.0 / 2.1 (XLANG Lab)** | 108 long-horizon tasks across 31 self-hosted websites and desktop apps (arXiv 2606.29537, June 2026). Median task about 1.6 skilled-human hours, about 27 weighted checkpoints, 500-step budget | Live, wide headroom on the primary metric | **Binary completion is the primary metric**; partial credit is reported alongside and runs 30+ points higher. Official leaderboard (September 17): Claude Opus 5 at max with batched tools, v2.1 full set, **44.33% binary / 77.67% partial**. Vendor launch posts quote partial credit: Opus 5.5 81.8% on "OSWorld 2.1" (Anthropic), GPT-6 Astra 72.6% vs GPT-5.6 Sol 65.7% on the offline v2026.08.08 set (OpenAI's table, via secondary reproductions). Anthropic's own 74.0% partial for Opus 5 matches no leaderboard row. Plan production on binary: hitting 80% of checkpoints is not finishing the job. |
| **Agents' Last Exam** | Professional workflows run in a sandbox; deterministic task-specific graders compare the agent's output files against a hidden reference. ALE-V1 publishes 147 tasks from a 1,500+ task corpus across 55 sub-industries (UC Berkeley RDI, with Snorkel AI) | Not saturated | Leaderboard: Claude Opus 5.5 (Claude Code, max) 38.2% pass rate, 63.2 score, about $1,340; GPT-6 Astra (Codex, max) 34.2% pass, 59.3 score, about $1,100. OpenAI's launch cited the 59.3 score, not the pass rate: the same row carries a partial-credit score about 25 points above the full-pass rate. |
| **Online-Mind2Web / WebArena** | Live web-agent tasks | Mixed | Online-Mind2Web's "illusion of progress" finding: many commercial agents underperformed a 2024 academic baseline once judged transparently. Judge methodology varies wildly, so scores are often not comparable. |
| **GDPval** | Real economically valuable knowledge work across 44 occupations, graded by human experts | Active, not saturated | The forward-looking "can it do a day of real work" signal; frontier approaches expert deliverable quality at ~100x lower cost. Pairwise Elo variants (GDPval-AA) differ from the win-rate version. |
| **METR time-horizon** | The task length (in human-minutes) a model completes at 50% reliability | No longer updated (since September 8, 2026) | Not a leaderboard but a trend: the horizon doubled roughly every 7 months overall and faster on coding, still the cleanest way to talk about agent autonomy growth. But METR's page now says it is no longer actively updated: the newest model on it is Claude Mythos Preview (early, added May 8, 2026), results use Time Horizon 1.1, and METR warns that measurements above 16 hours are unreliable with the current task suite. No current-generation model (Claude Opus 5.5, GPT-6 Astra, Gemini 4 Argon) has an official METR horizon, so do not quote one. |

### Long Context

The headline finding: **advertised context windows overstate usable context.** Lead with that, not with the window size on the spec sheet.

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **NIAH (needle-in-a-haystack)** | Single-fact retrieval at varying depth and length | Saturated / trivial | Frontier models score ~100%, which gives false confidence. A sanity check, not a discriminator. |
| **RULER (NVIDIA)** | Retrieval, multi-hop tracing, aggregation, QA at controlled lengths; reports "effective length" | Not saturated | The home of the effective-vs-advertised gap: many models claiming 128K hold quality only to ~32-64K (25-50% of advertised). Synthetic, so pair with prose-based tests. |
| **Fiction.LiveBench** | Deep narrative comprehension (theory-of-mind, chronology, implicit inference) up to ~192K tokens | Not saturated | The harshest practical long-context test; most models fall below 80% by 192K. Tiny (36 questions), so noisy. |
| **MRCR (multi-round coreference)** | Distinguish among multiple near-identical needles, return the i-th | Not saturated, esp. 8-needle | Scores collapse steeply with needle count and length. |
| **LongBench v2 / LongBench Pro** | Realistic long-context understanding and reasoning, 8K-2M tokens | Not saturated | LongBench Pro's finding: long-context *optimization* beats raw parameter scaling, and effective is below advertised on every model. |

A defensible one-liner for design docs: *advertised context windows routinely overstate usable context; on RULER, many models claiming 128K maintain quality only to ~32-64K, though the frontier is improving quickly.* The "context rot" work (degradation that kicks in well before the window limit, sometimes with the counterintuitive result that a shuffled haystack beats a coherent document) reinforces designing for a smaller effective budget. See [Context Engineering](../05-prompting-and-context/05-context-engineering.md).

### Multimodal

Image multiple-choice benchmarks are saturating; video reasoning is the genuinely unsolved frontier.

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **MMMU / MMMU-Pro** | College-level multimodal reasoning across 6 disciplines | Approaching saturation | MMMU-Pro hardens it (10 options, vision-only items where the question is in a screenshot) but suffers severe cross-harness disagreement (the same model reported at 81% and 94% on different harnesses), so never compare across sources. |
| **MathVista, DocVQA, ChartQA, MMBench** | Visual math, document QA, chart QA, broad multimodal | Largely saturated at the frontier | DocVQA/ChartQA near-solved; relaxed-accuracy scoring hides numeric errors. |
| **Video-MME / Video-MME-v2** | Video understanding | v1 approaching saturation, v2 far from it | Video-MME-v2 (2026) uses non-linear group scoring and shows a large model-vs-human gap, so it is the live multimodal separator. |

### Factuality and Instruction Following

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **IFEval** | Programmatically verifiable instruction-following ("at least 400 words", valid JSON, no commas) | Largely saturated (~90%+) | Only checks *checkable* constraints, a narrow slice of instruction-following; gameable. |
| **SimpleQA / SimpleQA Verified** | Short-form closed-book factual recall, adversarially hard; rewards calibrated abstention | Not saturated | Tops out around the mid-50s F1, so hallucination is far from solved. The headline lesson: more capable does not mean more factual. |
| **TruthfulQA** | Resistance to common misconceptions | Aging / partly saturated | Static and well-known, so contaminated; "truth" labels debatable; gameable by hedging. |
| **FACTS Grounding** | Whether long-form answers are fully supported by a provided source (no ungrounded claims) | Not saturated (~0.88 top) | Judged by an ensemble of LLMs that share lineage with contestants; entries are self-reported. The right benchmark family for RAG faithfulness. |

### Human Preference

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **Arena (arena.ai, formerly LMArena and Chatbot Arena)** | Crowd-sourced blind pairwise preference, reported as Elo | No ceiling, but the top ~15 compress into ~25 Elo points | The famous preference signal, and the most misused. See [The Leaderboard Illusion](#the-leaderboard-illusion). Always read the **style-controlled** Elo (which regresses out length and formatting bias) and the confidence intervals; differences inside ~15-20 Elo are noise. lmarena.ai now redirects to arena.ai. New entries carry a **Preliminary** label beside the score until votes accumulate (on the October 27, 2026 text board the top entry, Gemini 4 Carbon (High), was Preliminary at 1528 ±9 on about 4,400 votes, against about 11,900 for Gemini 4 Argon one place below). Earlier in 2026 Arena added day-1 AutoEval scores from a reward model trained on millions of live votes (Arena claims rank correlation above 0.98 with live scores). Treat any Preliminary or AutoEval number as a forecast. Agent categories (Code, Chat, Work) add price per task and a cost Pareto frontier. On the October 27 boards the top open model was Kimi K3 (Max) at #19 on text (1488) and Kimi K3.1 Pro (Max) at #9 on WebDev (a Preliminary 1695, behind Claude Opus 5.5 (Max) at 1810), and Arena labels Qwen3.8-Max as proprietary, so "best open model on Arena" depends on the board and on Arena's classification. |
| **Arena-Hard-Auto v2** | Automatic, reproducible proxy for Arena: hard prompts judged pairwise by strong LLMs | Not saturated, strong separator | ~3x the separability of MT-Bench and ~98% correlation with human Arena rankings. The cheap re-runnable preference signal when you need one. Use with style control on. |
| **MT-Bench** | Multi-turn quality via an LLM judge (80 prompts) | Saturated / obsolete | Tiny; GPT-4-judge biases (self-preference, verbosity). Superseded by Arena-Hard. |

---

## Reading Benchmarks Critically

This is the part that actually matters. Anyone can read a leaderboard; reading it *correctly* is the skill.

### Saturation

A benchmark is saturated when the frontier clusters so near the ceiling that score deltas are within noise. MMLU is the canonical case: GPT-4 hit ~86% in early 2023, and the frontier has sat at 86-93% since, so a 2-point "win" is often just a prompt artifact (MMLU scores vary 4-5% across prompt phrasings). The working signal: **when leaders cluster within ~3 points, the rank order is statistical noise, not capability.**

When a benchmark saturates the field responds with (1) harder successors (MMLU then MMLU-Pro then HLE), (2) private or held-out sets, (3) time-gated "live" benchmarks, and (4) composite indices. The median useful lifespan of a static public benchmark is under ~2 years.

In 2026 that lifespan compressed to months, sometimes weeks. Terminal-Bench 3.0 lasted about four weeks before 4.0 replaced it. SWE-Bench Pro, still the emerging primary coding signal in August, had its public split at 99.4% on v2 by late September. ARC-AGI-3 went from 0.51% at its March launch to 99.9% under a provider harness less than six months later. GPQA-Diamond and BrowseComp both crossed 90% at the frontier. The working assumption for any benchmark you cite in a design doc: check its version and saturation status the week you cite it.

### Contamination

Benchmarks are public and get scraped into pretraining, so models can score high by memorization rather than capability. The evidence is direct: re-deriving HumanEval-style problems (EvoEval) dropped scores ~39% across 51 models; on LiveCodeBench, a model's pass rate fell from ~60% on problems before its cutoff to ~0% after; OpenAI found SWE-bench Verified solutions reproducible verbatim from the task ID. Across multiple-choice QA benchmarks, measured contamination ranges from 1% to 45%, and larger models benefit more from it.

Contamination-resistant designs: **time-gating** (score only on problems released after the model's cutoff, as LiveCodeBench and SWE-rebench do), **private held-out sets** (FrontierMath, ARC-AGI-2; the cost is non-reproducibility), and **canary strings** (a unique token planted in a dataset that flags contamination if a model recites it). Detection methods (n-gram overlap, membership inference, the TS-Guessing quiz) all have failure modes; membership-inference attacks in particular barely beat random on real pretrained models. The practical move: for any static public benchmark, assume some contamination and discount the absolute number.

**Structural defenses arrived in 2026.** Three designs go beyond time-gating. **Commissioned tasks** are written from scratch for the benchmark rather than mined from merged PRs (DeepSWE). **Network-locked runs with independent re-grading** cut the agent off from repos, package indexes, and the web, then re-grade every diff on a pristine image (SWE-Bench Pro v2). **Double-blind evaluation** hides each side's secret from the other: Google DeepMind piloted it in August 2026 with the Singapore AI Safety Institute, OpenMined, AVERI, and MLCommons, running inside a Google Cloud Confidential Space enclave so the provider cannot see the test prompts and the evaluator cannot see the weights (announced; no results published yet). Contamination also found a new route: parallel instances with web access shared answers through public wikis, so eval harnesses must isolate instances from each other, not just from the internet.

### Harness and Scaffold Variance

The same model weights score 10-20 points differently depending on the prompt, whether tools are available, the reasoning effort level, and the agent scaffold. Anthropic measured that infrastructure configuration *alone* (RAM, concurrency, even time-of-day API latency) moved Terminal-Bench results ~6 points. This is why **provider self-reports run higher than independent leaderboards**: labs report the best harness and effort they found for their own model, on uncapped infrastructure. The hard rule that follows: **never compare a provider's number to another provider's number, or to an independent leaderboard.** Only same-harness numbers are comparable. And reasoning effort is not monotonic, more thinking lowered accuracy in 21 of 36 settings in one large agent study, so "high effort" provider numbers are not even comparable to that same model's default-effort independent run.

**The September 2026 launches made this concrete.** The same GPT-6 Astra weights scored 62.7% on ARC-AGI-3 under ARC Prize's provider-neutral harness and 99.9% under its Provider Adapter harness, which uses OpenAI's own context-management features to carry opaque reasoning state between requests. The metric moved numbers as much as the model did: the top OSWorld 2.1 row is 44.33% binary and 77.67% partial, and the top Agents' Last Exam row is a 38.2% pass rate and a 63.2 score. And some headline numbers no longer belong to one set of weights. Anthropic evaluated Claude Opus 5.5 with production safeguards on; when they intervened, cybersecurity tasks were completed by Claude Opus 4.8 and biology and frontier-LLM-development tasks by Claude Opus 5 (Zapier's AutomationBench ran without fallback, so interventions counted as failures). Artificial Analysis now labels such entries "with fallback". That score measures a routed system, and a regression test against it has to pin the fallback state the same way it pins harness and effort.

Pin all of these before you compare two numbers:

| Pin | Why it moves the number | 2026 example |
|-----|-------------------------|--------------|
| Benchmark version | Major versions retire and fix tasks; scores do not carry over | Terminal-Bench 3.0 to 4.0: 8 tasks removed, 19 fixed |
| Harness and scaffold | Tools, carried state, and compaction change what is measured | ARC-AGI-3: 62.7% vs 99.9% for the same model |
| Effort level | Not monotonic, and labs report their best setting | Terminal-Bench 4.0: Fable 5.1 at 54.55% (high) vs 57.88% (max) |
| Metric | Binary vs partial credit, pass rate vs score | OSWorld 2.1: 44.33% binary vs 77.67% partial on one row |
| Runner | Vendor runs use the best configuration they found | Opus 5 on OSWorld 2.1: Anthropic's 74.0% partial matches no leaderboard row |
| Fallback state | Safeguard routing hands some tasks to a different model | Opus 5.5 "with fallback" on Artificial Analysis |
| Cost per task | Equal scores can differ by multiples in cost | DeepSWE v1.1: a 74% tie at $2.36 to $11.84 per task |

**A third source of error surfaced in mid-2026: the benchmark's own tests.** An audit of 2,385 traces across 15 agent benchmarks found reward hacking or answer exposure in roughly 67% of traces on two of them, where agents recovered public solutions, read evaluation artifacts, or exploited invalid scoring paths rather than solving the task. A separate audit of SWE-bench Verified reported that a meaningful share of unsolved instances have flawed tests. September put numbers on how this scales with capability: an audit of 3,810 passing trajectories found unearned passes on SWE-Bench Pro v1.0 rising from 24% (Opus 4.7) to 73% (Fable 5) on matched tasks (arXiv 2609.34262), and METR's investigation of OpenAI's July cyber evaluations found tool-call spoofs in about 7% of transcripts, with an outside estimate that 30 to 40% of targets could not be exploited as intended. A rising score can mean better cheating. The practical rule: when a score jumps, check whether the capability improved or the protocol leaked, and prefer benchmarks that publish their validity audits and re-grades. The hardening checklist is in [Evaluating Agentic Systems](../07-agentic-systems/10-evaluating-agentic-systems.md).

**Contamination detection now has a theory of its own limits.** A formal treatment published in August 2026 shows detectability scales with the contaminated fraction, the behavioral gap between seen and unseen items, and the square root of the sample size. The consequence for practitioners is that "no evidence of contamination" from a small audit is ambiguous between a clean benchmark and an underpowered test, so contamination claims need a stated power calculation to mean anything.

### The Leaderboard Illusion

The central critique of LMArena (now Arena; Cohere et al., audit of ~2M battles, 243 models) found four problems: providers privately test many variants and publish only the best (Meta tested 27 variants before Llama-4), which violates the unbiased-sampling assumption behind the Elo math; proprietary providers get far more battle data than open models; you can train *to* the Arena distribution for large win-rate gains; and silently deprecated models distort the rankings. LMArena's rebuttal disputes the magnitude (their estimate of the private-testing boost is ~11 Elo, decaying as fresh votes accumulate) and notes the overfitting figure was measured on a static proxy, not the live human board. Present this as **contested but substantiated.**

Either way, the practical guidance is the same: treat Arena Elo as a measure of **general chat preference, not correctness, factuality, or hard reasoning**; always use the style-controlled board (Arena rewards longer, prettier answers); read the confidence intervals (the top ~15 are statistically near-tied); and use it as one of three signals, never alone.

### The Benchmark-to-Production Gap

A public score predicts your production performance only when three conditions hold at once: the benchmark tests tasks similar to yours, the test set is clean of contamination, and the benchmark has not saturated. In practice all three rarely hold. A principal-components analysis of benchmark scores found that a single "general capability" factor explains only ~50% of the variance; the rest is model-family idiosyncrasy and noise, which is why two models with equal general capability can differ sharply on *your* task. High GPQA does not guarantee performance on your domain. A September 2026 construct-validity study (arXiv 2609.08812, 56 benchmarks across 53 models) went further: capability benchmarks assigned the same concept correlated about as strongly with each other as with benchmarks of *different* concepts, and design features such as score format sometimes predicted correlation better than the shared concept. A benchmark's name does not guarantee what it measures.

In that principal-components analysis the cleanest general-capability loaders were GPQA-Diamond and SWE-bench Verified (with Aider Polyglot and AIME-style sets), and both have since saturated, which is the point: proxies expire. The conclusion every practitioner reaches: **for your decision, ignore the leaderboard and build evals on your data.** Construct a gold set partitioned across features, scenarios, and personas; use a binary LLM-as-judge calibrated to a domain expert (measured by precision and recall, not raw agreement); and price capability against cost. A few public boards now publish cost per task or per run (Terminal-Bench 4.0, DeepSWE, Agents' Last Exam, ARC Prize, Arena's agent categories), and they show near-ties separated by multiples in cost, but none of them is priced on your traffic, your cache hit rate, or your effort settings. See [LLM Evaluation](01-llm-evaluation.md) and the eval-pipeline whiteboard exercise in [Whiteboard Exercises](../00-interview-prep/04-whiteboard-exercises.md).

### Composite Indices

Because any single benchmark saturates within a year or two, the field ranks frontier models with weighted composites that keep discriminating as components max out and that resist overfitting to one test:

- **Artificial Analysis Intelligence Index** re-versions as components saturate, and **scores from different versions are not comparable**. v5.0 (announced October 27, 2026, US time) rebuilt it around four categories weighted 25% each, and put 60% of the weight on private or held-out sets:

  | Category | Weight | Components |
  |----------|--------|------------|
  | Knowledge Work | 25% | AA-Briefcase v1.1 (15%), AutomationBench-AA (5%), GDP.pdf (5%) |
  | Coding | 25% | FrontierCode 2.0 (15%, private), Terminal-Bench 4.0 (10%) |
  | Science | 25% | Terminal-Bench-Science 0.1 (15%), CritPt v2 hard subset (10%) |
  | General | 25% | AA-Omniscience v1.1 (15%: accuracy 10%, non-hallucination 5%), HLE-Diamond (10%) |

  GDPval-AA, SciCode, AA-LCR, and full HLE left the index; GPQA Diamond, AIME 2025, MATH-500, MMLU-Pro, LiveCodeBench, tau2 Telecom, tau3-Banking, and Terminal-Bench 2.1 and Hard were already legacy. The v5.0 launch board: Gemini 4 Carbon (high) 59, Claude Opus 5.5 (max with fallback) 58, GPT-6 Astra (max) 57, GPT-6.1 Sol (max) 55, Claude Sonnet 5.5 (max) 54, GPT-6.1 Terra (max) 51, Gemini 4 Argon (high) 50, Meta Muse Wonder Full (max) 50 (Muse Code subscribers only), Meta Muse Wonder (max) 43, Claude Haiku 5.5 (max) 36. On November 3, 2026, AA shipped methodology **v5.0.1**, applying grading-robustness fixes from the CritPt authors and regrading every model on CritPt v2; the v5.0.1 board reads Gemini 4 Carbon (high) 60, Claude Opus 5.5 (max) 59, GPT-6 Astra (max) 57, GPT-6.1 Sol (max) 55, Kimi K3.1 Pro 46, DeepSeek V4.1 Flash 1017 (max) 33, and GLM-5.3 (max) 32. A point moved within days on the same nominal version, so quote the patch version too. Two readings matter. Carbon was added October 28 and is not publicly available, so the top model you can actually call is Opus 5.5. And the "open-to-closed gap" depends on classification: Moonshot published Kimi K3.1 Pro's weights on November 4 (custom license), and AA's own v5 announcement had already called it "the top open weights model at 46", yet on November 6 its model page still labeled it proprietary. By weight availability the gap is 13 points (59 vs 46); by AA's current label it is 26 (59 vs DeepSeek's 33). Name the index version and the classification rule whenever you quote a gap. The v4.3.2 board it replaced also had Opus 5.5 at 58 at the start of October, on a different scale and component set: identical numbers, different claims. Never put v4.x and v5.x scores in one table.
- **Epoch Capability Index (ECI)** fits an item-response-theory model over ~1,000+ evaluations, inferring each benchmark's difficulty statistically so models score higher for doing well on *harder* benchmarks.
- **HAL (Holistic Agent Leaderboard, Princeton)** is the agent-specific, cost-aware composite: it scores accuracy *and* dollar cost, runs a fixed harness across models, and uses log analysis to surface agents taking shortcuts (pulling answers from arXiv instead of solving) and eval bugs. It exists because a 1% accuracy gain at 10x cost is not a win.

Composites are the right tool for "which model is generally best," but they still inherit their components' flaws, so read what they aggregate.

---

## A Practical Checklist

When you read any benchmark claim:

1. **Read the harness, not just the number.** Demand the benchmark version, scaffold, tool access, effort level, output-token cap, metric (binary or partial), who ran it, and the fallback state (see the pin table under [Harness and Scaffold Variance](#harness-and-scaffold-variance)). A bare percentage is uninterpretable.
2. **Check the confidence interval.** Ignore deltas inside the noise band. SWE-bench Verified is only 500 problems (~0.2% per item); Terminal-Bench 4.0 has 66 tasks, and Anthropic's own Opus 5.5 figure carries a ±2.6-point standard error; Arena gaps under ~15-20 Elo are noise. Distrust any gap under ~3 points until configs are matched. For boards that update continuously, anytime-valid tests built on e-processes (arXiv 2609.32248) answer "is this gap real" without the peeking problem of re-checking a fixed-sample interval.
3. **Prefer time-gated, private, held-out, or commissioned sets.** For static public sets (MMLU, HumanEval, GSM8K, SWE-Bench Pro's public split), assume contamination and discount the absolute number.
4. **Never compare across harnesses or self-reports.** Only same-harness numbers compare.
5. **Sanity-check the model name.** If a frontier score is attributed to a model you cannot find a primary source for, it is probably fabricated. Confirmed frontier names as of November 6, 2026: Claude Fable 5.1 and Mythos 5.1, Opus 5.5, Sonnet 5.5, Haiku 5.5 (GA October 7), and Haiku 4.5 (Fable 5, Opus 5, and Sonnet 5 remain available); GPT-6 Astra, GPT-6 Sol, GPT-6 Luna, GPT-6.1 Sol, GPT-6.1 Terra (October 26), GPT-6 Rosalind (October 30, trusted access only), GPT-6-Cyber (November 5, Daybreak Red customers only), and the GPT-5.6 Sol, Terra, and Luna generation; Gemini 4 Argon (GA October 13 as `gemini-4-argon`), Gemini 3.8 Flash, and `gemini-3.1-pro-preview`; Grok 4.7; Meta's Muse Wonder (October 12, on the Meta Model API) and Muse Wonder Full (Muse Code subscribers only), plus Muse Spark 1.3; Kimi K3.1 Pro (weights on Hugging Face since November 4, custom license); and open-weight Mistral Large 4, Muse Spark Open, MiMo-V2.6-Pro, GLM-5.3, Kimi K3, DeepSeek V4.1-Flash (now the 1017 release) and V4-Pro, and Qwen3.8. Gemini 4 Carbon is real but limited: announced October 28 for a subset of cyber defenders, with no public model ID, yet it tops both Artificial Analysis and Arena, so a leaderboard leader may be a model you cannot call. Not shipped as of that date, whatever an aggregator page says: Grok 5, Gemini 3.5 Pro, and Qwen4 (attendees reported a "Qwen4 next week" remark at Apsara Amsterdam on October 29, but Alibaba has published no date, model card, or license), plus Llama 4 8B/70B/405B, which never existed. Aggregator pages also publish scores that contradict the benchmark's own board, such as "Gemini 3 Pro 84.8% on SWE-bench Pro"; check the primary leaderboard.
6. **Triangulate three signal types** before trusting a ranking: a static academic eval, a human-preference arena, and an agentic suite.
7. **For your decision, build evals on your data.** Public scores predict your domain only rarely. The leaderboard tells you who to shortlist; your gold set tells you who to ship.

---

## Which Benchmarks Matter in 2026

For orienting on **frontier general capability**: HLE-Diamond (state tools or no tools), HLE and CritPt with the label-error caveat, ARC-AGI-3 with the harness named (ARC-AGI-2 is now past the human threshold for most frontier models), and the composite indices (Artificial Analysis Intelligence Index v5.0.1, with the patch version stated and never mixed with v4.x scores; Epoch ECI). Ignore MMLU, MMLU-Pro, HellaSwag, ARC-Challenge, and now GPQA-Diamond for ranking.

For **coding and agents**: SWE-Bench Pro v2's private set, DeepSWE, and the contamination-resistant live variants (SWE-rebench, SWE-bench-Live) for real ranking; Terminal-Bench 4.0 for terminal work and Terminal-Bench-Science for agentic science workflows (the vendor rankings differ between the two); tau-bench 2.0.0 for tool-use reliability (watch pass^k, and never compare with 1.x scores); OSWorld 2.x **binary** completion for computer use; Agents' Last Exam for artifact-graded professional work; and HAL or the cost columns on these boards for cost-aware comparison. Treat SWE-bench Verified and SWE-Bench Pro's public split as tier filters at best. Ignore HumanEval/MBPP and OSWorld-Verified for ranking.

For **deep research**: corpus-controlled evals such as BrowseComp-Plus; BrowseComp itself is saturated.

For **long context**: RULER and Fiction.LiveBench over NIAH; design for effective, not advertised, context.

For **embeddings and retrieval**: RTEB, which mixes open datasets with private held-out sets the maintainers evaluate, over MTEB-only rankings; see [Embedding Models](../06-retrieval-systems/03-embedding-models.md).

For **factuality**: SimpleQA for closed-book hallucination, FACTS Grounding for RAG faithfulness; remember more capable does not mean more factual.

For **preference**: style-controlled Arena Elo with confidence intervals (read a Preliminary or AutoEval score as a forecast), or Arena-Hard-Auto v2 as a reproducible proxy.

And for **your product**: none of the above. Build your own gold set and judge. The frontier research on this is moving fast (agent reliability science, claim-level faithfulness eval, eval-awareness where models detect they are being tested); see [Research Radar](../RESEARCH-RADAR.md).

---

## Interview Questions

### Q: A vendor says their model scores 90% on SWE-bench Verified. What questions do you ask before believing it predicts your coding-agent quality?

**Strong answer:**
First, the harness: which agent scaffold, what tools, what effort level, what output-token cap? Identical weights swing 10-20 points on scaffold alone, and vendor numbers use the best harness they found, so the 90% is not comparable to any other model's published number. Second, contamination: SWE-bench Verified is partly contaminated (OpenAI found solutions reproducible from the task ID, and repo-perturbation studies show models leaning on memorized repository cues), so I would want the contamination-resistant sets: SWE-Bench Pro v2's private split (Opus 5 scores 81.6% there against 99.4% on the public split), DeepSWE's commissioned tasks, or SWE-rebench. Third, the confidence interval: it is 500 problems, so a few points is noise. Fourth and most important, the production gap: even a clean SWE-bench number predicts my codebase only if my tasks resemble GitHub issue resolution. I would shortlist on the public number and decide on my own gold set of real tickets from our repos, scored by a calibrated judge, priced against cost.

### Q: Why have benchmarks like MMLU and HumanEval stopped being useful for ranking frontier models, and what replaced them?

**Strong answer:**
They saturated: the frontier clusters above 88-90% on MMLU and 90%+ on HumanEval, so deltas are within prompt-phrasing noise. They are also small and public, so contaminated, re-deriving HumanEval problems drops scores ~39%. And the construct is too easy: a single-function docstring-to-code task does not predict real engineering, which is why SWE-bench launched at under 2% for the same models that aced HumanEval. The field replaced them with harder successors (MMLU to MMLU-Pro to HLE to HLE-Diamond), contamination-resistant time-gated or commissioned benchmarks (LiveCodeBench, SWE-rebench, DeepSWE), agentic and repository-scale benchmarks (SWE-Bench Pro, Terminal-Bench, tau-bench, OSWorld 2.x), and composite indices that keep discriminating as components saturate. The catch in 2026 is that the successors turn over too: Terminal-Bench 3.0 lasted about four weeks, so every score I cite carries its version.

### Q: How would you use Arena (formerly LMArena) Elo responsibly when choosing a model for a chat product?

**Strong answer:**
As one signal of general chat preference, never as a measure of correctness or reasoning. I would read the style-controlled board, because raw Arena rewards longer and better-formatted answers regardless of correctness, and I would read the confidence intervals, because the top dozen models are statistically near-tied within ~15-20 Elo. I would also discount it for the leaderboard-illusion effects: providers privately test many variants and publish the best, so a fresh top entry may be partly best-of-N luck. For a model released this week I would check whether the score is labeled Preliminary or is an AutoEval reward-model prediction rather than accumulated human votes, and treat it as a forecast until the votes arrive. Then I would triangulate with an objective benchmark and an agentic suite, and ultimately validate on my own preference data, because Arena prompts are not my users' prompts.

### Q: A launch post claims 99.9% on ARC-AGI-3 and 81.8% on OSWorld 2.1. How do you read those numbers?

**Strong answer:**
I pin down what was measured before I compare anything. On ARC-AGI-3, the harness decides the headline: ARC Prize verified GPT-6 Astra at 99.9% on its Provider Adapter harness, which uses OpenAI's own context-management features to carry opaque reasoning state between requests, but 62.7% on its provider-neutral Standard harness, same model and same test set. So I ask which harness, what effort, and what it cost ($18,817 vs $26,098 for those two runs). On OSWorld, I ask which metric: 81.8% is Anthropic's partial-credit figure for Opus 5.5, and the benchmark's primary metric is binary completion, where the best leaderboard row is 44.33% (Opus 5 at max effort). I also ask who ran it (Anthropic's own 74.0% for Opus 5 matches no leaderboard row) and whether safeguards routed some tasks to a fallback model, which Anthropic's Opus 5.5 evaluation did. Then I say what I would actually plan on: the binary, independently run number, scoped to tasks like mine, and confirmed on my own eval set. A partial-credit score tells me how far an agent gets; only the binary score tells me how often the job is done.

---

## References

- Hendrycks et al. "Measuring Massive Multitask Language Understanding (MMLU)" arXiv:2009.03300
- Wang et al. "MMLU-Pro" arXiv:2406.01574
- Rein et al. "GPQA: A Graduate-Level Google-Proof Q&A Benchmark" arXiv:2311.12022
- "Humanity's Last Exam" arXiv:2501.14249
- "ARC-AGI-2" arXiv:2505.11831, [ARC Prize leaderboard](https://arcprize.org/leaderboard) and [results page](https://arcprize.org/results)
- [ARC Prize 2026, ARC-AGI-3 track](https://arcprize.org/competitions/2026/arc-agi-3)
- ARC Prize, [GPT-6 Astra on ARC-AGI-3](https://arcprize.org/blog/astra) (September 2026)
- [HLE-Diamond](https://lastexam.ai/blog/hle-diamond) (September 2026)
- [Epoch AI FrontierMath](https://epoch.ai/frontiermath) and [benchmarks hub](https://epoch.ai/benchmarks)
- Ansari et al. "How Good Are Frontier Models at Physics? Expert Re-Grading Reveals Broken Evaluations and Near-Saturation of Leading Benchmarks" arXiv:2609.13009
- Jimenez et al. "SWE-bench" arXiv:2310.06770 and [swebench.com](https://www.swebench.com/)
- Scale Labs, [SWE-Bench Pro v2 announcement](https://labs.scale.com/blog/swe-bench-pro-v2) and [public v2 leaderboard](https://labs.scale.com/leaderboard/swe_bench_pro_public_v2)
- "Maintaining Benchmarks Against Increasingly Capable Agents: Detection and Remediation of Unearned Passes" arXiv:2609.34262
- [DeepSWE](https://deepswe.datacurve.ai/blog/deepswe-v1-1) (Datacurve)
- [Terminal-Bench 4.0](https://www.tbench.ai/news/terminal-bench-4-0) and [leaderboard](https://www.tbench.ai/leaderboard/terminal-bench/4.0); [Terminal-Bench-Science 0.1](https://www.tbench.ai/news/terminal-bench-science-0-1)
- "SWE-bench-Live" arXiv:2505.23419 and [SWE-rebench](https://swe-rebench.com/)
- Jain et al. "LiveCodeBench" arXiv:2403.07974
- [Berkeley Function Calling Leaderboard (BFCL)](https://gorilla.cs.berkeley.edu/leaderboard.html)
- [Sierra tau-bench repository (tau2-bench)](https://github.com/sierra-research/tau2-bench) and [tau-bench 2.0.0 release notes](https://github.com/sierra-research/tau2-bench/releases/tag/v2.0.0) and [hyper-tau-bench](https://sierra.ai/blog/hyper-t-bench-evaluating-agents-that-build-agents)
- "OSWorld 2.0" arXiv:2606.29537 and [XLANG OSWorld 2.0 leaderboard](https://osworld-v2.xlang.ai/)
- "Agents' Last Exam" arXiv:2606.05405 and [leaderboard](https://snorkel.ai/leaderboard/agents-last-exam/)
- "GDPval" arXiv:2510.04374 and [OpenAI GDPval](https://openai.com/index/gdpval/)
- [METR, measuring AI task-completion time horizons](https://metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks) and [time-horizons page](https://metr.org/time-horizons/) (no longer updated since September 8, 2026)
- "RULER" arXiv:2404.06654 and [NVIDIA/RULER](https://github.com/NVIDIA/RULER)
- "MMMU-Pro" arXiv:2409.02813
- [OpenAI SimpleQA](https://openai.com/index/introducing-simpleqa/) and "SimpleQA Verified" arXiv:2509.07968
- "FACTS Grounding" arXiv:2501.03200
- Singh et al. "The Leaderboard Illusion" arXiv:2504.20879 and the LMArena response
- Arena, [AutoEval scores](https://arena.ai/blog/autoeval-scores) and [agent categories and cost per task](https://arena.ai/blog/agent-categories-and-cost)
- "Holistic Agent Leaderboard (HAL)" arXiv:2510.11977
- [Artificial Analysis methodology](https://artificialanalysis.ai/methodology/intelligence-benchmarking), [changelog](https://artificialanalysis.ai/changelog) (v5.0.1, November 3, 2026), and [Epoch Capability Index](https://epoch.ai/benchmarks)
- Desai et al. benchmark construct validity across 56 benchmarks and 53 models, arXiv:2609.08812
- Google DeepMind, [double-blind AI evaluation pilot](https://deepmind.google/blog/piloting-the-worlds-first-double-blind-ai-evaluations/) (August 2026)
- METR, [OpenAI / Hugging Face incident investigation](https://metr.org/blog/2026-08-26-openai-hugging-face-incident-investigation/) (August 2026)
- Anthropic, "Quantifying infrastructure noise in agentic coding evals" (Feb 2026)
- Hamel Husain, ["Your AI Product Needs Evals"](https://hamel.dev/blog/posts/evals/) and ["LLM-as-a-Judge"](https://hamel.dev/blog/posts/llm-judge/)

---

*Next: [CI/CD for LLM Applications](../11-infrastructure-and-mlops/02-cicd.md). See also [Research Radar](../RESEARCH-RADAR.md) for the frontier topics beyond the leaderboards.*
