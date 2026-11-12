# Navigating Framework Churn

AI orchestration frameworks change faster than the content teaching them can keep up. LlamaIndex and LangChain each re-architected their entire package layout in 2024 and removed their original headline abstractions within a year, and Haystack did the same in its 3.0 release in July 2026. The result: a course recorded twelve months ago often fails on the first `import`. This page is about that problem, why it happens, and how to learn and build so your knowledge and your code survive the churn.

The one-line version: **frameworks are how you ship this quarter; primitives are what you keep. Pin the former, learn the latter.**

## Table of Contents

- [The Trigger: Why a Course Breaks on a Fresh Install](#the-trigger-why-a-course-breaks-on-a-fresh-install)
- [What Actually Changed](#what-actually-changed)
  - [The November 2026 Snapshot](#the-november-2026-snapshot)
- [Why Courses and Tutorials Go Stale](#why-courses-and-tutorials-go-stale)
- [Is This Tutorial Current? A 30-Second Check](#is-this-tutorial-current-a-30-second-check)
- [Surviving Churn: Pin, Lock, Isolate](#surviving-churn-pin-lock-isolate)
- [Framework vs Raw SDK vs Thin Layer](#framework-vs-raw-sdk-vs-thin-layer)
- [What Transfers Across Versions](#what-transfers-across-versions)
- [Migrating When You Must Upgrade](#migrating-when-you-must-upgrade)
- [A Durable-Learning Playbook](#a-durable-learning-playbook)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Trigger: Why a Course Breaks on a Fresh Install

A common, real example: a learner starts a well-regarded LlamaIndex video course, copies the first cell, and hits

```
ImportError: cannot import name 'SimpleDirectoryReader' from 'llama_index'
```

or, a little later,

```
TypeError: Can't instantiate abstract class OpenAI with abstract method _prepare_chat_with_tools
```

Nothing is wrong with the learner or the course as recorded. The course's notebook environment pins old versions (one popular LlamaIndex course ships a `requirements.txt` with `llama-index==0.10.30` and `llama-index-llms-openai==0.1.26`), and the video was recorded against them. On a fresh `pip install llama-index` today you get a version several minor releases newer, where the import paths and class hierarchy have changed. The first error is a moved import; the second is a *partial* version mismatch, where the core package and an integration package were upgraded out of lockstep and a new abstract method exists on the base class that the older integration package never implemented.

This is not a LlamaIndex problem or a course-quality problem. It is the default outcome of fast-moving frameworks plus pinned, recorded teaching content. Understanding the mechanism is what lets you fix it in seconds instead of giving up.

---

## What Actually Changed

### The November 2026 Snapshot

Version churn does not only break tutorials; it also retires whole product surfaces and changes behavior without touching your code. Two lists from August to October 2026 are worth putting on a calendar rather than discovering at runtime.

**Retired surfaces and deadlines**

| Change | Date | What it means |
|---|---|---|
| **OpenAI Assistants API shut down** | August 26, 2026 (executed) | `/v1/assistants`, threads and runs now error, on OpenAI and Azure OpenAI. The replacement is the Responses API plus the Conversations API, with **no automated migration** for Threads. Vector stores survived: Responses `file_search` takes `vector_store_ids`, so teams re-wrote orchestration, not indexes. Any tutorial still using Assistants, Threads and Runs describes a dead API |
| **OpenAI Evals, Agent Builder and Reusable Prompts** | All three shut down November 30, 2026 (the Evals guide also announced a read-only step from October 31) | OpenAI's migration guide points eval users to Promptfoo, which is now part of OpenAI and remains MIT-licensed (0.125.1, Oct 25). This is consolidation onto a tool OpenAI owns, not an exit. Agent Builder users go to the Agents SDK or ChatGPT Workspace Agents. Note the irony: Reusable Prompts, the object Assistants map to in OpenAI's own migration guide, shuts down November 30, 2026, about three months after Assistants did |
| **OpenAI self-serve fine-tuning wind-down** | No new jobs for active customers from January 6, 2027 | "Fine-tune the closed model later" is no longer a plan on OpenAI. Customization moves to open-weight fine-tuning (for example LangSmith Fine-Tuning through Fireworks or Baseten) or to prompt and context methods |
| **Model snapshots** | OpenAI's legacy batch shut down on schedule October 23, 2026 (announced April 22): `gpt-4o-2024-05-13`, `gpt-4-turbo`, `gpt-4-0613`, `gpt-4-1106-preview`, `gpt-3.5-turbo-0125`, `gpt-4.1-nano`, `o1`, `o1-pro`, `o3-mini`, `o4-mini` and `gpt-image-1`. Still ahead: Claude Sonnet 4.5 retires November 30, 2026 (Claude API and Foundry); `gpt-5-2025-08-07`, `o3-2025-04-16` and sibling snapshots December 11, 2026; and, announced October 14, `gpt-4o-2024-08-06`, `gpt-4o-2024-11-20`, `gpt-4o-mini-2024-07-18`, `gpt-4.1-2025-04-14` and `gpt-4.1-mini-2025-04-14`, **plus fine-tunes of them**, on January 14, 2027 | Systems pinned to 2023- to 2025-era snapshots have hard deadlines. The October 14 notice gave about three months, and its replacement, `gpt-6-luna`, cannot be fine-tuned, so a GPT-4o or GPT-4.1 fine-tune has no like-for-like successor on OpenAI. The October 23 batch also took the bare `gpt-4` and `gpt-3.5-turbo` aliases and fine-tunes of GPT-3.5 Turbo, GPT-4, GPT-4.1-nano, o4-mini, `babbage-002` and `davinci-002`; OpenAI's listed substitutes are `gpt-5.6-sol`, `gpt-5.6-terra` or `gpt-5.6-luna`, and `gpt-image-2.5-sunburst` or `gpt-image-2.5-flare` for images. `gpt-5.4-cyber` was removed October 1 with about three weeks' notice, well under OpenAI's stated three months for specialized models |
| **Same model, different clouds** | Ongoing | Claude Opus 4.1 left the Claude API on August 5, 2026 but runs on Bedrock until January 8, 2027 (at higher extended-access prices from October 8). Claude Sonnet 4.5 runs the other way: it leaves the Claude API on November 30, 2026, while Bedrock moved it to Legacy on October 8 with extended access from January 8, 2027 and end of life April 8, 2027. Bedrock models launched from September 7, 2026 get a "no sooner than" EOL and a Legacy period of 6 months or only 45 days, and the Anthropic-managed Claude on Google Cloud (October 22) follows Anthropic's dates. Track lifecycle per (model, platform) pair, not per model |
| **Ownership changes** | Dynatrace completed its acquisition of Arize on Oct 1, 2026 (agreed Aug 13, $915M); Promptfoo is now part of OpenAI (still MIT-licensed); NVIDIA confirmed it is acquiring Hugging Face (Sep 3, $12.93B, no closing date disclosed) | Pick observability tools whose data you can export, and mirror the models and datasets you depend on rather than assuming a hub stays neutral |

**Silent behavior changes**

| Change | Date | Why nothing in your code warned you |
|---|---|---|
| **Default model swaps** | Aug 11 to Sep 29, 2026 | OpenAI Agents SDK 0.20.0 made `gpt-5.6-luna` the implicit default; Claude Code made Opus 5.5 (2.1.280) and Sonnet 5.5 (2.1.284) its defaults; Codex CLI rust-v0.159.1 switched to GPT-6.1 Sol; the Strands harness packages moved to Claude Opus 5. Every caller that never set a model got a new one. Azure Foundry will auto-upgrade `gpt-4o` 2024-05-13 Standard deployments to `gpt-5.6-sol` on December 9, 2026 |
| **October default and alias moves** | Oct 2026 | Claude Code 2.1.293 made Haiku 5.5 the default Haiku on the Anthropic API; the Gemini API began routing `gemini-3.7-flash` to `gemini-3.8-flash` and `gemini-3.5-flash` to `gemini-3.6-flash` (Oct 8), so a model string kept working while the model behind it changed; xAI pointed `grok-voice-latest` at `grok-voice-3.0`; and the OpenAI Agents SDK 0.23 line (first on PyPI as 0.23.1, Oct 2) made `gpt-transcribe` the default voice speech-to-text model |
| **Headless permission default** | Claude Code 2.1.302 (Oct 16) | `claude -p` and Agent SDK sessions with no permission mode now start in `default` mode on third-party providers, with telemetry off, or on a fresh install, while interactive sessions still start in auto. The same CI job can stall on a prompt in one environment and run unattended in another; set `--permission-mode` explicitly |
| **Telemetry and stream shapes** | Oct 2026 | ADK Python 2.13.0 (Oct 23) defaults to the OpenTelemetry GenAI semconv 1.44 schema and drops the `call_llm` span; OpenAI Agents SDK 0.23 omits reasoning from default trace exports and redacts default tool failure details; `langchain-core` 1.7.0 (Oct 22) and `@langchain/core` 1.3.0 stream tool-call arguments as `tool-call-delta` slices instead of snapshots; Pydantic AI 2.62.0 (Oct 27) prefixes hosted MCP tool names with the service. Dashboards go blank and name-keyed allowlists stop matching, with no exception |
| **Telemetry and call caps in a minor release** | ADK Go v2.7.0 (Nov 2) | Traces switch to OpenTelemetry GenAI semantic conventions v2 (one `gen_ai.client.inference.operation.details` event per model call, a new `invoke_workflow` parent span), and each invocation is capped at 500 model calls by default, ending with `agent.ErrLLMCallsLimitExceeded`. A long-running agent that used to finish now stops mid-task; `ADK_MAX_LLM_CALLS=-1` removes the cap, and unlike adk-python, 0 means the default |
| **Fail-open hooks** | Claude Agent SDK, through Nov 2026 | A hook callback that raises or returns `None` counts as no decision, so the guarded action runs. Claude Code 2.1.317 (Nov 2) added an opt-in `onFailure: "block"` (Python `on_failure`, `claude-agent-sdk` 0.2.177); the default is still `"continue"` |
| **Provider SDK transport swap** | OpenAI Python SDK 3.0.0 (Aug 12) and Anthropic Python SDK 1.0.0 (Aug 20) | Both moved to `httpx2`. OpenTelemetry's HTTPX instrumentation, Sentry, `respx`, `pytest-httpx` and `vcrpy` can silently miss SDK calls unless `httpx2.alias_httpx()` runs before anything imports `httpx` (Anthropic's migration guide). Missing spans and unmocked live calls raise no error |
| **Sampling parameters removed** | Gemini API (Jul 21), Anthropic SDK 1.0 (Aug 20), GPT-6 Astra (Sep 3), Anthropic API version `2026-10-21` (Oct 21) | `temperature`, `top_p` and `top_k` are deprecated in the Gemini API, unsupported on Astra, and were dropped from the Anthropic Python SDK 1.x method signatures (a `TypeError`). Under API version `2026-10-21`, which all eight new Anthropic SDK and CLI majors (Python 2.0.0, TypeScript 1.0.0, C# 13.0.0, Go 2.0.0, Java 3.0.0, PHP 1.0.0, Ruby 2.0.0, and the `ant` CLI 2.0.0) send by default, the Claude API returns 400 for all three **even at default values, on every model**, and also rejects `document` blocks in assistant messages and `citations` on user text blocks; on Bedrock the version works only for Claude 4.7 and later. Effort or thinking level is the new control surface |
| **SDK license change** | Anthropic SDK and CLI majors (Oct 21) | All eight new majors, including Java, C#, PHP, Ruby and the `ant` CLI, are licensed Apache 2.0 instead of MIT (earlier versions stay MIT). Nothing breaks at runtime, but a routine lockfile bump changes the license your legal review approved |
| **Forced tool choice rejected** | Claude Fable 5.1 (Sep 1), Opus 5.5 (Sep 22), Sonnet 5.5 (Sep 28) | `tool_choice` of type `any` or `tool` returns 400, breaking framework structured-output paths that forced a tool call until partner packages caught up (`langchain-anthropic` 1.7.3 and 1.7.5) |
| **MCP Python SDK 2.0** | July 28, 2026 | `pip install mcp` now resolves to 2.x, `FastMCP` became `MCPServer`, and frameworks followed within weeks (OpenAI Agents SDK 0.20.0, DSPy 3.3.1, LangChain 1.4.0). Pin `mcp<2` until you have ported |

The durable lesson is the same one this chapter makes about imports: **pin what you depend on, including model IDs, and subscribe to the deprecation feed of every vendor and cloud in your critical path**. The Assistants sunset was announced a full year ahead, which means the teams it broke were the ones that never read the notice. The silent changes are worse: only an eval harness and a telemetry check catch them.

Three re-architectures define the modern churn. Version numbers below carry their release dates; treat them as a snapshot, since they will keep moving.

### LlamaIndex

- **v0.10 (Feb 2024): the great split.** The monolithic `llama-index` package was broken into `llama-index-core` (abstractions only) plus hundreds of independently versioned integration packages (`llama-index-llms-openai`, `llama-index-embeddings-*`, `llama-index-vector-stores-*`, `llama-index-readers-*`). The separate `llama-hub` was folded in. Imports changed in two ways at once, which is why old notebooks fail immediately:
  - top-level to core: `from llama_index import VectorStoreIndex` becomes `from llama_index.core import VectorStoreIndex`
  - integrations into their own packages: `from llama_index.llms import OpenAI` becomes `from llama_index.llms.openai import OpenAI`
- **v0.11 (Aug 2024): the flagship abstraction removed.** `ServiceContext` (the object every pre-0.10 tutorial used to wire up the LLM, embeddings, and parser) was deprecated in 0.10 and *removed* in 0.11. Its replacement is the global `Settings` object. The same release moved the codebase to Pydantic v2.
  ```python
  # OLD (pre-0.10, removed in 0.11)
  from llama_index import ServiceContext, set_global_service_context
  service_context = ServiceContext.from_defaults(llm=llm, embed_model=embed)
  set_global_service_context(service_context)

  # CURRENT
  from llama_index.core import Settings
  Settings.llm = llm
  Settings.embed_model = embed
  ```
- **Workflows (1.0 in June 2025, now 2.x): the new application surface.** Event-driven, typed-state agentic orchestration, extracted into its own `llama-index-workflows` package. Note the correction many summaries get wrong: it is *Workflows* that hit 1.0 and then 2.x (`llama-index-workflows` 2.26.1, Oct 21, 2026); the core framework itself is still on the 0.x line (`llama-index` 0.14.25, Sep 21, 2026), not a "1.x" line. The company said in September 2026 that its primary focus is now its parsing and extraction products, and core releases have slowed accordingly: there was no core release in October.
- **Codemod:** `llamaindex-cli upgrade <dir>` rewrites old imports automatically.

### LangChain

- **The package split.** `langchain-core` (Runnables, messages, base interfaces, the only package with a backwards-compatibility guarantee), `langchain-community` (third-party integrations), `langchain` (chains and agents), and per-vendor partner packages (`langchain-openai`, `langchain-anthropic`, ...). LCEL, the `|`-pipe composition model, replaced the old `Chain` subclasses.
- **v0.3 (Sep 2024): Pydantic v1 to v2.** User code passing Pydantic v1 models broke.
- **v1.0 (Oct 2025): agents on LangGraph.** The blessed way to build an agent became `create_agent`, running on the LangGraph runtime with a middleware system. Legacy chains (`LLMChain`, `RetrievalQA`, `AgentExecutor`, `initialize_agent`) were moved to `langchain-classic`, deprecated but not deleted. `langchain` is at 1.4.9 (Nov 4, 2026) and requires Python 3.10+ for now: 1.4.8 and `langchain-core` 1.7.4 (Nov 2) warn that Python 3.10 support ends in `langchain-core` 1.8.0 and `langchain` 1.5.0, after Python 3.10 reached end of life on October 1, 2026, and Semantic Kernel Python 1.47.0 (Nov 5) already requires 3.11, so move agent services to 3.11+ before the next minor; 1.4.0 (Sep 3) moved the MCP adapter into the core package. `langchain-core` 1.7.0 (Oct 22) changed the stream format, not the API: tool-call arguments now arrive as `tool-call-delta` slices instead of snapshots of the arguments so far, so code that reads raw stream events mid-call must change (or pin `langchain-core<1.7`), while `.tool_calls` and the assembled output are unchanged. LangChain 0.3 and LangGraph 0.4 receive security and critical fixes only until December 2026.
- **Deprecation map:** `LLMChain` to an LCEL pipe (`prompt | llm | parser`); `RetrievalQA` to `create_retrieval_chain`; `AgentExecutor` / `initialize_agent` to `create_agent`; legacy `Memory` classes to LangGraph checkpointers.

### Haystack

- **3.0 (Jul 20, 2026): the agent takes over.** The standalone `ToolInvoker` is removed; the `Agent` owns tool execution end to end, with lifecycle hooks (`before_run`, `before_llm`, `before_tool`, `after_tool`, `on_exit`, `after_run`), a `SkillToolset` with progressive disclosure, human-in-the-loop as a `ConfirmationHook`, and `ToolResultOffloadHook` for large tool outputs.
- **Pipelines merged, integrations split out.** `Pipeline` and `AsyncPipeline` became one class, legacy generators were removed, and 30 components (Sentence Transformers, Hugging Face, Whisper, the OpenTelemetry and Datadog tracers, among others) moved to separately released integration packages. The line is at 3.4.1 (Oct 21, 2026).

The deeper detail is in the [LangChain deep dive](01-langchain-deep-dive.md) and [LlamaIndex chapter](04-llamaindex.md). The point here is the *pattern*: a monolith splits into core plus plugins, the original convenience abstraction is removed, and the agent layer moves onto a richer runtime. All three frameworks followed it, roughly a year apart, and lab SDKs are now on the same treadmill: OpenAI's 0.x Agents SDK shipped a default-model change and a breaking client change within eight days in August 2026, and DSPy 3.4 removed an experimental API that 3.3 had introduced less than eight weeks earlier. The next lab SDK rewrite is already in preview: OpenAI's `openai-agents-python` v0.24.0-alpha.6 and `openai-agents-js` v0.22.0-alpha.1 (Nov 3, 2026, GitHub Releases only) replace `Runner` with `Agent.run()`, make agent declarations immutable, and embed the native runtime and Codex harness (details in [the Agent SDK landscape](07-autogen-crewai.md#openai-agents-sdk)).

---

## Why Courses and Tutorials Go Stale

Recorded courses and blog posts capture a *snapshot*: the video, and usually a pinned `requirements.txt` or hosted notebook environment, are fixed at recording time. The live package index is not. When a learner installs fresh, the resolver pulls current versions that have moved past the pin, and the recorded code no longer matches the installed API.

The failure modes are predictable:

- **Moved imports** (`cannot import name ... from 'llama_index'`): the symbol relocated to `.core` or a partner package.
- **Removed symbols** (`ImportError: ServiceContext`, references to `LLMChain` / `RetrievalQA`): the abstraction was deleted, not just moved.
- **Partial-upgrade mismatches** (`Can't instantiate abstract class ...`): core and an integration package drifted out of lockstep; the usual fix is to upgrade the *set* together (`pip install -U llama-index llama-index-llms-openai`).
- **Model-name deprecations** (`gpt-3.5-turbo-0301` no longer available; the `gpt-3.5-turbo` and `gpt-4` aliases common in 2023-era tutorials shut down October 23, 2026; `claude-sonnet-4-5-20250929` retires November 30, 2026): the tutorial pinned a model ID the provider has since retired. This is the same churn, one layer down.
- **Removed parameters** (`TypeError: ... unexpected keyword argument 'temperature'` on the Anthropic Python SDK 1.x, a 400 for any `temperature`, `top_p` or `top_k` under Anthropic API version `2026-10-21`, or a 400 for forced `tool_choice` on Opus 5.5, Sonnet 5.5 and Fable 5.1): the API surface under the framework moved.
- **Silent changes** (no error at all): a new default model, or traces that stopped arriving after an SDK moved to `httpx2`. These are found by evals and dashboards, not by stack traces.

Most teaching platforms encode their version contract only as a bundled lockfile or a frozen hosted environment, not as a visible "this course was recorded against version X" banner. So the staleness is invisible until the code breaks.

---

## Is This Tutorial Current? A 30-Second Check

Before investing hours in any course, post, or notebook:

1. **Check the date against the framework's release cadence.** A 2024 LlamaIndex or LangChain tutorial predates at least one full re-architecture by construction.
2. **Open the bundled `requirements.txt` or lockfile and compare the pin to the current release.** A `llama-index==0.10.x` pin against a current `0.14.x`, or any `langchain<1.0`, means expect breakage.
3. **Grep the code for known-removed symbols.** Their presence dates the material instantly:
   - LlamaIndex: `ServiceContext`, `LLMPredictor`, `set_global_service_context`, or `from llama_index import` without `.core`.
   - LangChain: `LLMChain`, `RetrievalQA`, `initialize_agent`, `AgentExecutor`.
   - OpenAI: `client.beta.assistants`, `threads`, `runs` (the Assistants API shut down August 26, 2026).
   - MCP: `from mcp.server.fastmcp import FastMCP` (renamed `MCPServer` in MCP Python SDK 2.0).
   - DSPy: `dspy.Assert`, `dspy.Suggest` (gone in 3.x). Semantic Kernel: Stepwise or Handlebars planners (removed).
   - Anthropic SDK: `temperature=` or `top_p=` passed to `messages.create` (removed from the signatures in 1.0, and rejected with a 400 on every model under API version `2026-10-21`, the default from SDK 2.0).
4. **Prefer the project's own current quickstart as the source of truth**, and use the third-party course for *concepts* rather than copy-paste code.

---

## Surviving Churn: Pin, Lock, Isolate

The discipline that prevents "worked yesterday, broken today":

- **Pin exact versions.** A loose, unversioned `llama-index` is the single biggest cause of surprise breakage. At minimum, `==`-pin your direct dependencies.
- **Use a real lockfile** that captures transitive dependencies too. [`uv`](https://docs.astral.sh/uv/) (`uv.lock`, `uv sync`) is the fast-moving 2026 favorite; Poetry (`poetry.lock`) and pip-tools (`pip-compile`) are established. The emerging standard is the tool-agnostic `pylock.toml` (PEP 751). Treat `pyproject.toml` as intent and the lockfile as reality, and commit the lockfile.
- **Pin split packages as a set.** For LlamaIndex and LangChain, `core` and every integration package must move together. The "abstract class" error is precisely a partial upgrade. Upgrade the set, not one package.
- **Isolate every project** in its own virtualenv or container. Never install into system Python. A container that pins the Python base image plus the lockfile is what hosted course notebooks effectively do, and what a local learner usually skips.
- **Treat deprecation warnings as a clock, not noise.** Run with warnings visible; each one names the replacement and often the removal version. Silenced warnings are how a working app becomes a broken one on the next routine upgrade.
- **Pin model IDs, not just packages.** Set the model explicitly in every SDK and agent harness call, because a package upgrade can carry a model swap. Track lifecycle per (model, platform) pair, since the same model retires on different dates on the vendor API, Bedrock, Google Cloud and Foundry.
- **Verify telemetry after every upgrade.** Check that spans, token counts and mocks still see the provider calls; a transport change can blind them without an error.

---

## Framework vs Raw SDK vs Thin Layer

A live 2026 question, because the original reason frameworks existed has partly evaporated. When LangChain and LlamaIndex appeared, provider APIs were inconsistent and a unifying layer paid for itself. Since then, tool/function calling and structured outputs have converged into native, similar features across the major provider SDKs, so the framework's abstraction value has shrunk while its churn cost has not.

| Altitude | Use when | Cost |
|----------|----------|------|
| **Raw provider SDK** (`anthropic`, `openai`) | You make a handful of model calls, want the most stable surface and the clearest stack traces, or are writing library code | You build retrieval, the agent loop, and retries yourself |
| **Framework** (LangChain, LlamaIndex) | You need breadth of integrations (dozens of vector stores, loaders) or batteries-included RAG/agent scaffolding to move fast | Dependency sprawl, deep stack traces, version churn |
| **Thin layer** (your own interface over the SDK) | Production systems that want to swap models or frameworks without touching call sites | A little upfront design |

For production, the thin layer is often the sweet spot: depend on the provider SDK (or only `langchain-core`), wrap it behind a small interface of your own, and keep framework specifics in one replaceable module. The raw SDK is not churn-free either (both major Python SDKs changed their HTTP transport in August 2026, Anthropic's removed sampling parameters, and its October majors changed the default API version and the license), but its changes are fewer, documented in one place, and land in one module of yours. The rule of thumb on abstraction leakage: the more a layer hides things you must understand to debug (retrieval ranking, token budgeting, the tool-call loop), the riskier it is. Leaky agent abstractions are exactly what pushed LangChain to build LangGraph. See the [Framework Selection Guide](08-framework-selection-guide.md) for the choice in depth.

---

## What Transfers Across Versions

This is the core of learning durably. The half-life of a framework *API* is roughly a year. The half-life of the *concepts under it* is the field itself. Invest accordingly.

**Transfers (learn deeply):**
- **RAG mechanics:** chunking and splitting strategy, embedding plus similarity search, retrieval, re-ranking, and the context-relevance / groundedness / answer-relevance evaluation triad. These survive every rename of `VectorStoreIndex`.
- **The agent loop:** model call, tool selection, tool execution, observation, repeat, plus state, memory, and human-in-the-loop. Whether it is `AgentExecutor`, `create_agent`, or a hand-rolled `while` loop, the loop is the same.
- **Provider-native primitives:** tool/function calling, structured outputs, streaming, token and context budgeting. Now standardized across vendors, so this is the most durable layer of all.
- **Engineering discipline:** lockfiles, reproducible environments, changelog reading, eval harnesses. Pure transfer value.

**Does not transfer (do not over-invest):** exact import paths, class names, constructor signatures, the global-config object of the month (`ServiceContext` versus `Settings`), and which chain helper is blessed this quarter (`LLMChain` versus LCEL versus `create_agent`). Memorizing these is memorizing a depreciating asset.

---

## Migrating When You Must Upgrade

When you do have to move a real codebase forward:

1. **Upgrade in a branch, lockfile first**, one major step at a time (0.10 to 0.11 to 0.12), not many at once.
2. **Run the official codemod** where one exists (`llamaindex-cli upgrade`), then let deprecation warnings and import errors drive the worklist.
3. **Lean on bridge packages** (`langchain-classic`, `llama-index-legacy`) to keep the app running while you migrate incrementally instead of big-bang. Upgrade a framework's partner package together with the provider SDK and the model ID it must support, not one at a time.
4. **Confirm behavior with an eval harness**, not just that imports resolve. A migration that compiles but quietly changes retrieval quality or agent success rate is a regression you want caught before production. See [LLM Evaluation](../14-evaluation-and-observability/01-llm-evaluation.md).

---

## A Durable-Learning Playbook

1. **Build the loop once from the raw SDK**, no framework, so you understand what the framework automates. You will debug framework failures far faster afterward.
2. **Then adopt a framework** for breadth and speed, but treat its API as replaceable, behind a thin interface.
3. **Pin everything, commit the lockfile, keep deprecation warnings visible.**
4. **Re-derive, do not re-memorize.** When a framework renames things, map the new API back to the primitive it implements ("`create_agent` is just the agent loop on LangGraph") instead of relearning from scratch.
5. **Vet course currency before investing** with the 30-second check above. Use stale courses for concepts, the project's current docs for code.

For curated, currency-checked courses, see [COURSES.md](../COURSES.md). The reason that file is dated and re-verified is exactly the churn this page describes.

---

## Interview Questions

### Q: A teammate followed a six-month-old LlamaIndex tutorial and it fails on import. Walk me through what happened and how you would fix it.

**Strong answer:**
The tutorial was recorded against an older pinned version, and a fresh install pulled a newer one where the package layout changed. Since v0.10, LlamaIndex is `llama-index-core` plus separate integration packages, so a top-level import like `from llama_index import SimpleDirectoryReader` now has to be `from llama_index.core import SimpleDirectoryReader`, and `ServiceContext` was removed in v0.11 in favor of the global `Settings` object. If the error is instead "can't instantiate abstract class OpenAI," that is a partial upgrade where core and the OpenAI integration package drifted apart; the fix is to upgrade them together. The durable fix is a pinned lockfile so the environment is reproducible, and reading the migration guide rather than guessing. Longer term I would point the teammate at the project's current quickstart for code and use the tutorial only for the concepts.

### Q: Given how fast these frameworks churn, how do you decide whether to use one at all?

**Strong answer:**
I look at what the framework is actually buying me. Its original job was smoothing over inconsistent provider APIs, but tool calling and structured outputs have converged across the major SDKs, so that value has shrunk. If I need breadth of integrations or batteries-included scaffolding to move fast, the framework earns its keep. If I am making a handful of model calls or writing library code, the raw provider SDK is more stable and easier to debug. For production I usually wrap the SDK behind a thin interface of my own, so a framework or model swap touches one module. Whatever I choose, I pin and lock it and keep the framework-specific code isolated, because I am assuming this quarter's blessed API will be deprecated.

### Q: An agent's cost per task rose 40% and its tone changed overnight, with no code or dependency change on your side. What happened, and how do you make this impossible to miss next time?

**Strong answer:**
"No change on our side" usually means no change we *pinned*. My first checks: did anything resolve a model implicitly (an SDK or harness default, a `-latest` alias, a cloud auto-upgrade such as Foundry moving `gpt-4o` 2024-05-13 deployments to `gpt-5.6-sol`), did a floating tool version update in CI or on developer machines (Claude Code and Codex CLI both changed default models in September 2026, and Claude Code changed the headless default permission mode in October), and did the provider change something server-side such as a snapshot retirement redirect. Traces answer this fast if they record the resolved model ID per call, which is why I log the model the provider reports, not the one I requested. Prevention is three habits: explicit model IDs everywhere, pinned versions for every SDK and agent CLI with upgrades going through an eval gate, and alerts on cost per task, tokens per task and cache hit rate, so a silent swap shows up as an anomaly the same day.

### Q: Design a model registry that survives divergent deprecation calendars across vendors and clouds.

**Strong answer:**
The key is the unit: a lifecycle record per (model, platform, region) pair, not per model, because the same model now retires on different dates on the vendor API, Bedrock, Google Cloud and Foundry (Claude Opus 4.1 left the Claude API in August 2026 but runs on Bedrock until January 2027). Each record holds the pinned ID, the "not sooner than" floor, any announced shutdown date, the vendor's notice policy (OpenAI states 6 months for GA models, 3 months for specialized variants and as little as 2 weeks for previews; Anthropic 60 days; Bedrock 6 months or 45 days for models launched from September 7, 2026) and the eval-approved replacement. A daily job scrapes the deprecation pages and changelogs and diffs them against the registry. Alerts fire at 90, 60 and 30 days, and a failover route is only valid if its target is not retiring sooner than the primary. Bedrock's Legacy rules add one more check: once a model enters Legacy, an existing customer can lose access after 15 days of inactivity, so disaster-recovery routes to Legacy models need a synthetic heartbeat or, better, a replacement.

---

## References

- LlamaIndex v0.10 migration guide: https://developers.llamaindex.ai/python/framework/getting_started/v0_10_0_migration/
- LlamaIndex ServiceContext to Settings guide: https://developers.llamaindex.ai/python/framework/module_guides/supporting_modules/service_context_migration/
- LlamaIndex Workflows 1.0 announcement (June 2025): https://www.llamaindex.ai/blog/announcing-workflows-1-0-a-lightweight-framework-for-agentic-systems
- LangChain and LangGraph 1.0 (Oct 2025): https://www.langchain.com/blog/langchain-langgraph-1dot0
- LangChain v0.3 migration (Pydantic v2): https://docs.langchain.com
- `uv` (lockfiles and reproducible environments): https://docs.astral.sh/uv/
- PEP 751 (`pylock.toml`, standard lockfile format): https://peps.python.org/pep-0751/
- LangChain release policy: https://docs.langchain.com/oss/python/release-policy
- Haystack 3.0.0 release notes (Jul 2026): https://github.com/deepset-ai/haystack/releases/tag/v3.0.0
- OpenAI deprecations: https://developers.openai.com/api/docs/deprecations
- Anthropic model deprecations: https://platform.claude.com/docs/en/about-claude/model-deprecations
- Amazon Bedrock model lifecycle: https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html
- OpenAI Python SDK `httpx2` notes: https://github.com/openai/openai-python/blob/main/httpx2.md
- Promptfoo. "Promptfoo is joining OpenAI" (Mar 2026): https://www.promptfoo.dev/blog/promptfoo-joining-openai/
- Anthropic. API version `2026-10-21` migration guide: https://platform.claude.com/docs/en/api/migration-2026-10-21
- `langchain-core` 1.7.0 release notes (Oct 22, 2026): https://github.com/langchain-ai/langchain/releases/tag/langchain-core%3D%3D1.7.0

---

*Next: [Document Processing](../10-document-processing/01-ocr-and-layout.md)*
