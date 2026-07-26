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
- **The headline number hides the harness.** "88% on SWE-bench" is uninterpretable without the agent scaffold, tool access, effort level, and output-token cap.

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

The benchmarks that still separate the top of the field, in rough order of remaining headroom.

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **HLE (Humanity's Last Exam)** | Closed-ended expert questions across 100+ subjects, retrieval-resistant (~2,500 questions, ~10% multimodal) | Live, wide separation | The premier frontier knowledge benchmark. No-tools scores span roughly 40-53%, so spreads are readable. At its Jan-2025 launch the SOTA was under 10%, a sign of how fast the frontier moved. Watch for tools-vs-no-tools configs that inflate and confuse comparisons. |
| **GPQA-Diamond** | 198 PhD-level biology/physics/chemistry questions, Google-proof | Compressing toward saturation | Designed so skilled non-experts with web access score ~34% and PhD experts ~65-70%. Frontier cluster now ~92-94%, so it is thinning as a top discriminator but still separates mid-tier models and is a good sanity check. |
| **ARC-AGI-2** | Abstract grid-puzzle reasoning, efficiency-aware (~400 tasks); resists memorization | Live, strongly separating | **Read this carefully:** the human panel passes 85%, and 85% is the grand-prize *threshold*. Aggregators report frontier models "at 85%," but that is a self-reported number under a non-ARC-Prize harness with heavy test-time compute. The ARC-Prize-Verified ceiling is much lower (around the mid-50s at high cost). Never quote the threshold as an achieved score. |
| **FrontierMath** | Research-level math, problems that take expert mathematicians hours to days (~350 problems, private; Tier 4 is the hardest) | Live but climbing fast | Open-ended with auto-verifiable closed-form answers (guess-proof). v1 had errors in ~42% of problems (fixed in the June 2026 v2); it was partly OpenAI-funded, so single-lab numbers warrant a governance caveat. Epoch's own framing is that less than 70% is within reach, so be skeptical of aggregator scores far above that. |
| **CritPt** | Research-grade physics reasoning across 11 domains, by 50+ physicists | Live, very low ceiling | Most frontier models score in single digits to ~45% on the full challenges, so it separates cleanly. Newer (late 2025) and becoming a standard scientific-reasoning component. |

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
| **SWE-bench Verified** | Resolve real GitHub issues so hidden tests pass (human-validated 500-issue subset) | Near-saturated, contaminated | The canonical coding number 2024-2026. But OpenAI's own evals team flagged that more than 60% of tasks have flawed tests and that solutions can be reproduced verbatim from the task ID. Useful as an "above ~80% is frontier" tier filter, unreliable for fine ranking. Harness choice alone swings results 10-20 points. |
| **SWE-bench Pro** | Harder, contamination-resistant issue resolution (private + copyleft + held-out repos) | Not saturated, the emerging primary signal | Scores run 25-35 points below Verified and separate far better. OpenAI now points to Pro over Verified for coding capability. Still read the harness: vendor-reported numbers can sit ~17 points above the same model on a standardized harness. |
| **SWE-bench Multimodal** | Resolve visual/UI software issues (JS front-end, includes screenshots of broken UI) | Not saturated | Tests real visual grounding; text-SWE-bench systems struggle. |
| **SWE-rebench / SWE-bench-Live** | Continuously updated, decontaminated issue resolution; tasks postdate model cutoffs | Not saturated | The strongest contamination audits. SWE-rebench specifically catches inflated Verified scores that collapse on fresh tasks. |
| **LiveCodeBench** | Competitive-programming problems tagged by release date; score only on post-cutoff problems | Not saturated in fresh windows | Contamination-resistant by construction. Competitive programming is not software engineering, so it predicts algorithmic reasoning, not repo work. |
| **Aider Polyglot** | Real multi-file edits across 6 languages, must emit a valid edit format, 2 attempts | Approaching saturation (~88%) | Measures edit-format reliability, not repo-scale reasoning; Exercism source is public (contamination). |
| **SciCode** | Research-grade scientific coding decomposed into subproblems | Far from saturated | The hardest mainstream coding benchmark. Note two incompatible scoring conventions (main-problem vs subproblem); always cite which. |
| **Terminal-Bench (2.x)** | End-to-end agent tasks in a real terminal sandbox (build, debug, sysadmin) | Active, not saturated | Scores are harness-paired (the agent CLI matters as much as the model). |

**Why agentic benchmarks displaced HumanEval:** HumanEval/MBPP saturated at 90%+ for every frontier model, are small and memorized, and a single-function docstring-to-code task does not predict real engineering. The proof: SWE-bench launched at under 2% resolved for the same model generation that aced HumanEval. Reading a repo, running tests, reading a stack trace, and iterating a patch is a different skill, so the field moved to issue-resolution and terminal benchmarks.

### Agentic and Tool Use

The fastest-moving area, because agents are where 2026 production value is. Most of these are not saturated.

| Benchmark | Measures | Status | Notes |
|-----------|----------|--------|-------|
| **BFCL (Berkeley Function Calling Leaderboard)** | Tool/function-calling accuracy; V4 adds agentic tasks with web search and memory | V1/V2 saturated, V3/V4 not | AST-based scoring can miss semantic errors. Beware stale mirrors of the board; use the official Berkeley leaderboard. |
| **tau-bench / tau2-bench (Sierra)** | Agent uses tools and talks to a simulated user under a domain policy (retail, airline, telecom) | Not saturated on reliability | Reports **pass^k** (re-run the same task k times) which exposes a brutal reliability cliff: an agent at ~60% pass@1 can fall to ~25% pass^8. The most production-relevant tool-use signal because it measures consistency, not just best-case. Original tau2 had buggy tasks; use the corrected forks. |
| **GAIA** | General-assistant tasks needing multi-step tool use, web browsing, files | Saturated at the top (~92%) by orchestrated ensembles | Top entries are multi-model ensembles and scaffolds, not base models, so it measures the orchestration more than the model. |
| **OSWorld-Verified** | Computer-use agent on a real OS, execution-scored | Not saturated, approaching the human band | Cleaned and standardized successor to OSWorld (which launched with a best score of ~12% vs ~72% human). Vendor numbers exceed public-harness numbers. |
| **Online-Mind2Web / WebArena** | Live web-agent tasks | Mixed | Online-Mind2Web's "illusion of progress" finding: many commercial agents underperformed a 2024 academic baseline once judged transparently. Judge methodology varies wildly, so scores are often not comparable. |
| **ClawBench** | Browser agents completing 153 everyday tasks across 144 live production websites | Active, not saturated | Request interception and five-layer traces make failures auditable, but live-site drift and anti-bot systems limit run-to-run comparability. |
| **GDPval** | Real economically valuable knowledge work across 44 occupations, graded by human experts | Active, not saturated | The forward-looking "can it do a day of real work" signal; frontier approaches expert deliverable quality at ~100x lower cost. Pairwise Elo variants (GDPval-AA) differ from the win-rate version. |
| **METR time-horizon** | The task length (in human-minutes) a model completes at 50% reliability | Active research standard | Not a leaderboard but a trend: the horizon doubles roughly every 7 months overall and faster on coding. The cleanest way to talk about agent autonomy growth. |

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
| **LMArena (Chatbot Arena Elo)** | Crowd-sourced blind pairwise preference, reported as Elo | No ceiling, but the top ~15 compress into ~25 Elo points | The famous preference signal, and the most misused. See [The Leaderboard Illusion](#the-leaderboard-illusion). Always read the **style-controlled** Elo (which regresses out length and formatting bias) and the confidence intervals; differences inside ~15-20 Elo are noise. |
| **Arena-Hard-Auto v2** | Automatic, reproducible proxy for Arena: hard prompts judged pairwise by strong LLMs | Not saturated, strong separator | ~3x the separability of MT-Bench and ~98% correlation with human Arena rankings. The cheap re-runnable preference signal when you need one. Use with style control on. |
| **MT-Bench** | Multi-turn quality via an LLM judge (80 prompts) | Saturated / obsolete | Tiny; GPT-4-judge biases (self-preference, verbosity). Superseded by Arena-Hard. |

---

## Reading Benchmarks Critically

This is the part that actually matters. Anyone can read a leaderboard; reading it *correctly* is the skill.

### Saturation

A benchmark is saturated when the frontier clusters so near the ceiling that score deltas are within noise. MMLU is the canonical case: GPT-4 hit ~86% in early 2023, and the frontier has sat at 86-93% since, so a 2-point "win" is often just a prompt artifact (MMLU scores vary 4-5% across prompt phrasings). The working signal: **when leaders cluster within ~3 points, the rank order is statistical noise, not capability.**

When a benchmark saturates the field responds with (1) harder successors (MMLU then MMLU-Pro then HLE), (2) private or held-out sets, (3) time-gated "live" benchmarks, and (4) composite indices. The median useful lifespan of a static public benchmark is under ~2 years.

### Contamination

Benchmarks are public and get scraped into pretraining, so models can score high by memorization rather than capability. The evidence is direct: re-deriving HumanEval-style problems (EvoEval) dropped scores ~39% across 51 models; on LiveCodeBench, a model's pass rate fell from ~60% on problems before its cutoff to ~0% after; OpenAI found SWE-bench Verified solutions reproducible verbatim from the task ID. Across multiple-choice QA benchmarks, measured contamination ranges from 1% to 45%, and larger models benefit more from it.

Contamination-resistant designs: **time-gating** (score only on problems released after the model's cutoff, as LiveCodeBench and SWE-rebench do), **private held-out sets** (FrontierMath, ARC-AGI-2; the cost is non-reproducibility), and **canary strings** (a unique token planted in a dataset that flags contamination if a model recites it). Detection methods (n-gram overlap, membership inference, the TS-Guessing quiz) all have failure modes; membership-inference attacks in particular barely beat random on real pretrained models. The practical move: for any static public benchmark, assume some contamination and discount the absolute number.

### Harness and Scaffold Variance

The same model weights score 10-20 points differently depending on the prompt, whether tools are available, the reasoning effort level, and the agent scaffold. Anthropic measured that infrastructure configuration *alone* (RAM, concurrency, even time-of-day API latency) moved Terminal-Bench results ~6 points. This is why **provider self-reports run higher than independent leaderboards**: labs report the best harness and effort they found for their own model, on uncapped infrastructure. The hard rule that follows: **never compare a provider's number to another provider's number, or to an independent leaderboard.** Only same-harness numbers are comparable. And reasoning effort is not monotonic, more thinking lowered accuracy in 21 of 36 settings in one large agent study, so "high effort" provider numbers are not even comparable to that same model's default-effort independent run.

### The Leaderboard Illusion

The central critique of LMArena (Cohere et al., audit of ~2M battles, 243 models) found four problems: providers privately test many variants and publish only the best (Meta tested 27 variants before Llama-4), which violates the unbiased-sampling assumption behind the Elo math; proprietary providers get far more battle data than open models; you can train *to* the Arena distribution for large win-rate gains; and silently deprecated models distort the rankings. LMArena's rebuttal disputes the magnitude (their estimate of the private-testing boost is ~11 Elo, decaying as fresh votes accumulate) and notes the overfitting figure was measured on a static proxy, not the live human board. Present this as **contested but substantiated.**

Either way, the practical guidance is the same: treat Arena Elo as a measure of **general chat preference, not correctness, factuality, or hard reasoning**; always use the style-controlled board (Arena rewards longer, prettier answers); read the confidence intervals (the top ~15 are statistically near-tied); and use it as one of three signals, never alone.

### The Benchmark-to-Production Gap

A public score predicts your production performance only when three conditions hold at once: the benchmark tests tasks similar to yours, the test set is clean of contamination, and the benchmark has not saturated. In practice all three rarely hold. A principal-components analysis of benchmark scores found that a single "general capability" factor explains only ~50% of the variance; the rest is model-family idiosyncrasy and noise, which is why two models with equal general capability can differ sharply on *your* task. High GPQA does not guarantee performance on your domain.

For coding and agents, the best public proxies are GPQA-Diamond and SWE-bench Verified (Aider Polyglot and AIME-style sets also load cleanly on general capability), but only when harness-matched. The conclusion every practitioner reaches: **for your decision, ignore the leaderboard and build evals on your data.** Construct a gold set partitioned across features, scenarios, and personas; use a binary LLM-as-judge calibrated to a domain expert (measured by precision and recall, not raw agreement); and price capability against cost, because no public benchmark contains a cost signal. See [LLM Evaluation](01-llm-evaluation.md) and the eval-pipeline whiteboard exercise in [Whiteboard Exercises](../00-interview-prep/04-whiteboard-exercises.md).

### Composite Indices

Because any single benchmark saturates within a year or two, the field ranks frontier models with weighted composites that keep discriminating as components max out and that resist overfitting to one test:

- **Artificial Analysis Intelligence Index** weights ~9 evaluations across agents, coding, scientific reasoning, and general knowledge, and re-versions as components saturate (it has dropped MMLU-Pro, LiveCodeBench, and AIME 2025 over time).
- **Epoch Capability Index (ECI)** fits an item-response-theory model over ~1,000+ evaluations, inferring each benchmark's difficulty statistically so models score higher for doing well on *harder* benchmarks.
- **HAL (Holistic Agent Leaderboard, Princeton)** is the agent-specific, cost-aware composite: it scores accuracy *and* dollar cost, runs a fixed harness across models, and uses log analysis to surface agents taking shortcuts (pulling answers from arXiv instead of solving) and eval bugs. It exists because a 1% accuracy gain at 10x cost is not a win.

Composites are the right tool for "which model is generally best," but they still inherit their components' flaws, so read what they aggregate.

---

## A Practical Checklist

When you read any benchmark claim:

1. **Read the harness, not just the number.** Demand the scaffold, tool access, effort level, and output-token cap. A bare percentage is uninterpretable.
2. **Check the confidence interval.** Ignore deltas inside the noise band. SWE-bench Verified is only 500 problems (~0.2% per item); Arena gaps under ~15-20 Elo are noise. Distrust any gap under ~3 points until configs are matched.
3. **Prefer time-gated, private, or held-out sets.** For static public sets (MMLU, HumanEval, GSM8K), assume contamination and discount the absolute number.
4. **Never compare across harnesses or self-reports.** Only same-harness numbers compare.
5. **Sanity-check the model name.** If a frontier score is attributed to a model you cannot find a primary source for, it is probably fabricated. Confirmed June 2026 frontier families: Claude (Fable 5, Opus 4.8/4.7, Sonnet 4.6, Haiku 4.5), GPT-5.5, Gemini 3.1 Pro, DeepSeek V4, Llama 4, Kimi K2.6, Qwen 3.6, Mistral Medium 3.5, Grok 4.3.
6. **Triangulate three signal types** before trusting a ranking: a static academic eval, a human-preference arena, and an agentic suite.
7. **For your decision, build evals on your data.** Public scores predict your domain only rarely. The leaderboard tells you who to shortlist; your gold set tells you who to ship.

---

## Which Benchmarks Matter in 2026

For orienting on **frontier general capability**: HLE, GPQA-Diamond (with the caveat it is thinning), CritPt, and the composite indices (Artificial Analysis Intelligence Index, Epoch ECI). Ignore MMLU, MMLU-Pro, HellaSwag, ARC-Challenge.

For **coding and agents**: SWE-bench Verified as a tier filter, SWE-bench Pro and the contamination-resistant live variants (SWE-rebench, SWE-bench-Live) for real ranking, Terminal-Bench and tau2-bench for tool-use and reliability (watch pass^k), and HAL for cost-aware agent comparison. Ignore HumanEval/MBPP.

For **long context**: RULER and Fiction.LiveBench over NIAH; design for effective, not advertised, context.

For **factuality**: SimpleQA for closed-book hallucination, FACTS Grounding for RAG faithfulness; remember more capable does not mean more factual.

For **preference**: style-controlled Arena Elo with confidence intervals, or Arena-Hard-Auto v2 as a reproducible proxy.

And for **your product**: none of the above. Build your own gold set and judge. The frontier research on this is moving fast (agent reliability science, claim-level faithfulness eval, eval-awareness where models detect they are being tested); see [Research Radar](../RESEARCH-RADAR.md).

---

## Interview Questions

### Q: A vendor says their model scores 90% on SWE-bench Verified. What questions do you ask before believing it predicts your coding-agent quality?

**Strong answer:**
First, the harness: which agent scaffold, what tools, what effort level, what output-token cap? Identical weights swing 10-20 points on scaffold alone, and vendor numbers use the best harness they found, so the 90% is not comparable to any other model's published number. Second, contamination: SWE-bench Verified is partly contaminated (OpenAI found solutions reproducible from the task ID), so I would want the contamination-resistant variants (SWE-bench Pro, SWE-rebench) where scores run 25-35 points lower and separate better. Third, the confidence interval: it is 500 problems, so a few points is noise. Fourth and most important, the production gap: even a clean SWE-bench number predicts my codebase only if my tasks resemble GitHub issue resolution. I would shortlist on the public number and decide on my own gold set of real tickets from our repos, scored by a calibrated judge, priced against cost.

### Q: Why have benchmarks like MMLU and HumanEval stopped being useful for ranking frontier models, and what replaced them?

**Strong answer:**
They saturated: the frontier clusters above 88-90% on MMLU and 90%+ on HumanEval, so deltas are within prompt-phrasing noise. They are also small and public, so contaminated, re-deriving HumanEval problems drops scores ~39%. And the construct is too easy: a single-function docstring-to-code task does not predict real engineering, which is why SWE-bench launched at under 2% for the same models that aced HumanEval. The field replaced them with harder successors (MMLU to MMLU-Pro to HLE), contamination-resistant time-gated benchmarks (LiveCodeBench, SWE-rebench), agentic and repository-scale benchmarks (SWE-bench, Terminal-Bench, tau2-bench), and composite indices that keep discriminating as components saturate.

### Q: How would you use LMArena Elo responsibly when choosing a model for a chat product?

**Strong answer:**
As one signal of general chat preference, never as a measure of correctness or reasoning. I would read the style-controlled board, because raw Arena rewards longer and better-formatted answers regardless of correctness, and I would read the confidence intervals, because the top dozen models are statistically near-tied within ~15-20 Elo. I would also discount it for the leaderboard-illusion effects: providers privately test many variants and publish the best, so a fresh top entry may be partly best-of-N luck. Then I would triangulate with an objective benchmark and an agentic suite, and ultimately validate on my own preference data, because Arena prompts are not my users' prompts.

---

## References

- Hendrycks et al. "Measuring Massive Multitask Language Understanding (MMLU)" arXiv:2009.03300
- Wang et al. "MMLU-Pro" arXiv:2406.01574
- Rein et al. "GPQA: A Graduate-Level Google-Proof Q&A Benchmark" arXiv:2311.12022
- "Humanity's Last Exam" arXiv:2501.14249
- "ARC-AGI-2" arXiv:2505.11831 and [ARC Prize leaderboard](https://arcprize.org/leaderboard)
- [Epoch AI FrontierMath](https://epoch.ai/frontiermath) and [benchmarks hub](https://epoch.ai/benchmarks)
- Jimenez et al. "SWE-bench" arXiv:2310.06770 and [swebench.com](https://www.swebench.com/)
- [Scale AI SWE-bench Pro leaderboard](https://labs.scale.com/leaderboard/swe_bench_pro_public)
- "SWE-bench-Live" arXiv:2505.23419 and [SWE-rebench](https://swe-rebench.com/)
- Jain et al. "LiveCodeBench" arXiv:2403.07974
- [Berkeley Function Calling Leaderboard (BFCL)](https://gorilla.cs.berkeley.edu/leaderboard.html)
- [Sierra tau2-bench](https://github.com/sierra-research/tau2-bench)
- Zhang et al., ["ClawBench: Can AI Agents Complete Everyday Online Tasks?"](https://arxiv.org/abs/2604.08523), [official repository](https://github.com/TIGER-AI-Lab/ClawBench), and [project site](https://claw-bench.com/)
- "GDPval" arXiv:2510.04374 and [OpenAI GDPval](https://openai.com/index/gdpval/)
- [METR, measuring AI task-completion time horizons](https://metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks)
- "RULER" arXiv:2404.06654 and [NVIDIA/RULER](https://github.com/NVIDIA/RULER)
- "MMMU-Pro" arXiv:2409.02813
- [OpenAI SimpleQA](https://openai.com/index/introducing-simpleqa/) and "SimpleQA Verified" arXiv:2509.07968
- "FACTS Grounding" arXiv:2501.03200
- Singh et al. "The Leaderboard Illusion" arXiv:2504.20879 and the LMArena response
- "Holistic Agent Leaderboard (HAL)" arXiv:2510.11977
- [Artificial Analysis methodology](https://artificialanalysis.ai/methodology/intelligence-benchmarking) and [Epoch Capability Index](https://epoch.ai/benchmarks)
- Anthropic, "Quantifying infrastructure noise in agentic coding evals" (Feb 2026)
- Hamel Husain, ["Your AI Product Needs Evals"](https://hamel.dev/blog/posts/evals/) and ["LLM-as-a-Judge"](https://hamel.dev/blog/posts/llm-judge/)

---

*Next: [CI/CD for LLM Applications](../11-infrastructure-and-mlops/02-cicd.md). See also [Research Radar](../RESEARCH-RADAR.md) for the frontier topics beyond the leaderboards.*
