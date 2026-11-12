# The 2026 Tool-Use and Computer Agent Landscape

The way AI agents interact with the outside world has undergone a dramatic shift. In 2024, "tool use" meant a model emitting a JSON function call that your backend executed. Today we have full-blown autonomous agents that clone repos, run shell commands, control desktops via screenshots, and message you on WhatsApp, all orchestrated through standardized protocols like MCP. This chapter maps these tools, their architectures, and the design decisions that differentiate them.

## Table of Contents

- [Ecosystem Overview](#ecosystem-overview)
- [Category Taxonomy](#category-taxonomy)
- [OpenClaw: The Viral Personal AI Agent](#openclaw-the-viral-personal-ai-agent)
- [OpenHands: Autonomous Developer Agent](#openhands-autonomous-developer-agent)
- [Open Interpreter: Local Code Execution](#open-interpreter-local-code-execution)
- [Claude Computer Use: Vision-Based Automation](#claude-computer-use-vision-based-automation)
- [Claude Code: The Terminal Agent](#claude-code-the-terminal-agent)
- [IDE Agents: Cursor, Devin Desktop, Cline](#ide-agents-cursor-devin-desktop-cline)
- [Comparison Matrix](#comparison-matrix)
- [Market Trends and Adoption (2026)](#market-trends-and-adoption-2026)
- [System Design Interview Angle](#system-design-interview-angle)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Ecosystem Overview

The 2026 tool-use ecosystem has consolidated around five deployment categories, each optimized for different levels of autonomy, safety, and integration depth. MCP servers and messaging bridges are the plumbing that connects them:

```
+-----------------------------------------------------------------------+
|                        2026 Tool-Use Ecosystem                        |
+-----------------------------------------------------------------------+
|                                                                       |
|  +-------------------+  +-------------------+  +-------------------+  |
|  |  LOCAL AGENTS     |  |  CLOUD AGENTS     |  |  IDE AGENTS       |  |
|  |                   |  |                   |  |                   |  |
|  |  OpenClaw         |  |  Claude Code      |  |  Cursor           |  |
|  |  Open Interpreter |  |  OpenAI Codex     |  |  Devin Desktop    |  |
|  |  OpenHands (local)|  |  OpenHands Cloud  |  |  Cline            |  |
|  |  OpenCode         |  |  Devin (cloud)    |  |  GitHub Copilot   |  |
|  +-------------------+  +-------------------+  +-------------------+  |
|                                                                       |
|  +-------------------+  +-------------------+  +-------------------+  |
|  |  COMPUTER-USE     |  |  MCP SERVERS      |  |  MESSAGING AGENTS |  |
|  |                   |  |                   |  |                   |  |
|  |  Claude computer  |  |  10,000+ servers  |  |  OpenClaw (multi) |  |
|  |  and browser tools|  |  TS SDK v1: ~246M |  |  Custom bots      |  |
|  |  Gemini 3.8 Flash |  |  npm downloads    |  |  via MCP bridges  |  |
|  |  Copilot (preview)|  |  (Oct 2026)       |  |                   |  |
|  +-------------------+  +-------------------+  +-------------------+  |
|                                                                       |
|  +-------------------+  +------------------------------------------+  |
|  |  MANAGED HARNESS  |  |  PERSISTENT AGENTS (own VM and identity) |  |
|  |                   |  |                                          |  |
|  |  Claude Managed   |  |  OpenAI dots, Meta Muse,                 |  |
|  |  Agents, OpenAI   |  |  Copilot Autopilot (private preview),    |  |
|  |  Agents API       |  |  Cursor Projects (beta)                  |  |
|  +-------------------+  +------------------------------------------+  |
+-----------------------------------------------------------------------+
```

The key insight for 2026: these categories are converging. Claude Code is a cloud agent that runs locally. OpenClaw is a local agent that connects to cloud LLMs. Cursor is an IDE agent whose cloud agents (formerly Background Agents) can now run on your own machines. By September 2026 the frontier labs sell the agent harness itself as a managed runtime, and consumer products give each agent its own computer and identity. The lines are blurring, and what matters is the underlying **architecture pattern** (covered in the next chapter).

---

## Category Taxonomy

### 1. Local Agents (Self-Hosted, User-Controlled)

Agents that run on the user's own hardware. The LLM call may go to the cloud, but the agent process, memory, and tool execution are local.

**Key properties:**
- Full filesystem access on the user's machine
- Persistent memory stored locally (SQLite, JSON, Markdown)
- User owns all data; no vendor lock-in
- Security responsibility falls entirely on the operator

**Examples:** OpenClaw, Open Interpreter, local OpenHands deployments, and open-source terminal harnesses such as OpenCode (MIT, about 224K GitHub stars in early November 2026)

### 2. Cloud Agents (Vendor-Hosted, API-Driven)

Agents that run in vendor-managed cloud environments. Code execution happens in sandboxed VMs or containers.

**Key properties:**
- Sandboxed execution (Docker, Firecracker VMs, E2B)
- No local filesystem access (works on cloned repos)
- Vendor handles scaling, security, and infrastructure
- Pay-per-use or subscription pricing

**Examples:** Claude Code (cloud mode), OpenAI Codex (reusable cloud environments since DevDay, September 29, 2026), Devin, OpenHands Cloud. Google Jules is winding down (read-only after November 11, 2026, shut down December 31), a reminder to keep hosted coding agents behind an interface you can repoint.

The sandbox no longer has to be the vendor's. Claude Code self-hosted runners (Team and Enterprise, v2.1.224, August 2026), Cursor Self-hosted Machines (September 2, 2026) and the OpenAI Agents API's self-hosted environments all keep the vendor's control plane while tool execution, credentials and egress stay inside the customer perimeter.

### 3. IDE Agents (Editor-Integrated, Context-Aware)

Agents embedded directly in code editors. They have deep understanding of project structure, open files, and editor state.

**Key properties:**
- Tight integration with editor UI (inline diffs, tab completion)
- Codebase indexing via embeddings or AST parsing
- Background agents that work asynchronously on branches
- Optimized for developer workflow, not general automation

**Examples:** Cursor (Agent Mode, cloud agents, Projects), Devin Desktop (formerly Windsurf), Cline, GitHub Copilot, Google Antigravity (a product family: the Antigravity 2.0 multi-agent app, the IDE, a CLI and an SDK; bundled into eligible Gemini Enterprise subscriptions since August 20, 2026). The Antigravity CLI is now Google's general-purpose terminal agent: Gemini CLI (v0.66.0, October 27, 2026) is scoped to Gemini Code Assist Standard and Enterprise and Code Assist for GitHub, and its README says it is not optimized for Gemini 4 models. AWS's Kiro (agentic IDE and CLI) shows the same harness churn: Kiro CLI 3.0.0 (November 3, 2026) made the V3 harness the default for interactive sessions, `--trust-all-tools` scripts, and new ACP clients, and retired the desktop app and inline autocomplete (details in the [coding agents guide](../09-frameworks-and-tools/10-opencoderguide.md#kiro-aws)).

**Pin managed agent versions.** Google's `antigravity-preview-09-2026` replaced `antigravity-preview-05-2026` on September 17, 2026 with about three weeks' notice (shutdown scheduled for October 5). Built-in tool parameters moved from snake_case to PascalCase and file edits became line-range replacements, so any code that executes the built-in tools locally or parses their calls broke. Pin dated agent versions and contract-test tool schemas in CI.

### 4. Computer-Use Agents (Vision-Based, GUI-Driven)

Agents that interact with software the way humans do: by looking at screenshots and clicking.

**Key properties:**
- Model sees screenshots, decides mouse/keyboard actions
- Works with any application (no API needed)
- Higher latency (screenshot-action loop is 1-3 seconds per model round trip; batch actions now put several steps in one round trip)
- Requires sandboxed environments for safety (VM + VNC)

**Examples:** Claude's computer and browser toolsets (GA August 19, 2026), Gemini computer use (GA October 30, 2026; the docs recommend `gemini-3.8-flash` and list `gemini-4-argon` as the most capable option), computer use in the OpenAI Agents API through an OpenAI-hosted browser (public beta since September 29, 2026) and through the Responses API on GPT-6.1 Terra ($1/$5 per 1M tokens, October 26), GitHub Copilot computer use (public preview October 1, 2026), and the original Python Open Interpreter's Computer API

The browser variants increasingly target **accessibility-tree element references** rather than raw pixels, which survive layout shifts and cut misclicks. Pixels remain the fallback for desktop apps.

### 5. Managed and Persistent Agents (Vendor-Run Harness, Own Identity)

The newest category, and the one that changes the design questions. Instead of shipping a model endpoint, the vendor runs the whole agent loop, and consumer products give each agent its own cloud computer, identity and approval policy.

**Key properties:**
- Vendor-run harness with durable sessions, tool execution, compaction and recovery
- One VM or sandbox per agent, often with its own sign-in identity
- Declarative approval rules instead of per-call prompts
- Billing per session or usage, sometimes per transaction

**Examples:**

| Product | Status (November 2026) | Design detail worth stealing |
|---------|--------------------------|------------------------------|
| Claude Managed Agents | Beta; budgets (August 7), `ant apply` (September 3) and the auto policy (September 10) are recent additions, and it now also runs on Claude on Google Cloud | Session spend caps that pause with `budget_reached`; server-evaluated auto permission policy; agents-as-code via `ant apply` and a committed `claude-lock.json` |
| OpenAI Agents API | Public beta (September 10); since October 9, hosted sandboxes bill only during active turns | Managed Codex harness; per-origin browser approval; credentials entered out of band, never in model input; US data residency only and no ZDR at launch |
| Amazon Bedrock Managed Agents (powered by OpenAI) | Public preview (September 29) | Per-agent IAM role, CloudTrail logging, runtime and inference stay inside AWS |
| OpenAI dots | Announced September 29 (Pro, Business Premium) | One cloud computer per agent; read-only research when idle; Custom Rules to allow, block or require approval; saved passwords used without exposing them to the model |
| Meta Muse | Launched September 8 (consumer) | Runs in a dedicated cloud VM; Meta says it expects to profit from a small fee on transactions |
| Microsoft Copilot Autopilot (formerly Scout) | Private preview (end of September) | Own identity, memory and computer inside the customer tenant |
| Anthropic Claude Duet | Research preview (October 21; waitlist for US Pro, Max and Ultra, not Team or Enterprise) | Own cloud computer and browser on Claude Opus 5.5 with no model choice; asks before messages, payments (one-time card per purchase), signing, or deleting, and blocks securities trading and moving money between accounts even with approval; draws on the plan's usage limits at no extra cost |

The interview takeaway: once an agent has its own VM, inbox and credentials, the hard problems are tenancy, agent identity, approval policy and kill switches, not single-task completion.

---

## OpenClaw: The Viral Personal AI Agent

### What It Is

OpenClaw is a self-hosted, open-source personal AI assistant created by Austrian developer Peter Steinberger. Originally published as "Clawdbot" in November 2025, it was renamed to OpenClaw in January 2026. It exploded from 0 to 346,000 GitHub stars in under five months, surpassing React as GitHub's most-starred software project on March 3, 2026.

**By the numbers (May 2026):**
- 346,000+ GitHub stars
- 3.2 million active users
- 500,000+ running instances
- 44,000+ community skills on ClawHub
- 38 million monthly visitors to the project site
- 24+ messaging platform integrations

In early November 2026 the repository showed about 394K stars and 83K forks. The project is now stewarded by the **OpenClaw Foundation**, an independent 501(c)(3) with no paid tier, hosted service or token; its README describes OpenAI as "a donor, not an owner." Releases run on two trains: calendar-versioned builds (2026.9.7 on September 30) and gateway-only extended-stable builds that the project treats as its LTS line (2026.8.35 on October 2). The code is MIT-licensed. See the [OpenClaw deep dive](03-openclaw-deep-dive.md) for the architecture in detail.

### How It Works

OpenClaw's architecture has six core components:

```
+-------------------------------------------------------------------+
|                      OpenClaw Architecture                        |
+-------------------------------------------------------------------+
|                                                                   |
|  +-----------+     +-----------+     +----------+                 |
|  |  Gateway  |---->|  LLM      |---->| Runtime  |                 |
|  |           |     |  (Brain)  |     | (Tools)  |                 |
|  +-----------+     +-----------+     +----------+                 |
|       ^                  |                |                       |
|       |                  v                v                       |
|  +-----------+     +-----------+     +----------+                 |
|  | Channels  |     | SOUL.md   |     | Skills   |                 |
|  | (24+)     |     | (Identity)|     | (44K+)   |                 |
|  +-----------+     +-----------+     +----------+                 |
|                          |                                        |
|                          v                                        |
|                    +-----------+                                  |
|                    | Memories  |                                  |
|                    | (Persist) |                                  |
|                    +-----------+                                  |
+-------------------------------------------------------------------+
```

**1. Gateway**: The message ingress/egress layer. Connects to WhatsApp (via Baileys), Telegram, Discord, Slack, Signal, iMessage, Microsoft Teams, Matrix, and 16+ other platforms. Supports both DMs and group conversations with mention-based activation.

**2. LLM (The Brain)**: Model-agnostic by design. Supports OpenAI (recent releases added GPT-6 Astra and GPT-6.1 Sol), Claude (including Fable 5.1), Gemini, Meta Muse Spark 1.3, DeepSeek, or local models via Ollama. The user picks the model; the architecture does not care.

**3. Agent Runtime and Core Tools**: OpenClaw's embedded agent loop wires the model to core tools (read, write, edit and exec, plus browser, messaging and others), always subject to tool policy. The LLM generates code or commands, and the runtime writes and executes them. This is the "hands" of the agent. (The README credits Mario Zechner's pi agent toolkit; current docs describe a single OpenClaw-owned runtime, and external harnesses such as Codex or Claude can be swapped in as plugins.)

**4. SOUL.md (Identity Layer)**: A plain Markdown file that defines the agent's personality, communication style, values, and behavioral guardrails. Loaded at session start and injected into the system prompt. Every agent instance reads SOUL.md first; it "reads itself into being."

**5. Skills (Plugin System)**: Extensions that give the agent new capabilities. Over 44,000 community skills exist on ClawHub. Skills follow the AgentSkills spec and can be bundled, workspace-local, or installed globally.

**6. Memories (Persistent Context)**: Long-term memory stored locally. The agent builds up context about the user across conversations. Combined with SOUL.md, this gives each agent a consistent personality across all messaging platforms.

### Workspace Files

| File | Purpose |
|------|---------|
| `SOUL.md` | Agent personality, tone, values, guardrails |
| `AGENTS.md` | Operating instructions, plus a Tools section for local tool conventions |
| `USER.md`, `IDENTITY.md` | Optional user model; the agent's name and persona details |
| `MEMORY.md`, `memory/YYYY-MM-DD.md` | Curated long-term memory and daily logs, indexed for memory search |

Earlier releases also used `HEARTBEAT.md` for scheduled autonomous actions. Current releases have retired it: heartbeat instructions and cron jobs live in the Gateway's automation state, and `openclaw doctor --fix` migrates an old file.

### Security Concerns

OpenClaw's rapid growth has outpaced security practices. In February 2026, SecurityScorecard's STRIKE team counted more than 135,000 instances exposed on the public internet, many with default configurations. The ClawHub skills marketplace has minimal security oversight: skills are Markdown with optional TypeScript, easy to create and install, and easy to abuse. Tools for the main session run directly on the host unless you configure sandboxing (sandbox mode defaults to `off`). This is a critical design consideration for anyone deploying OpenClaw in production; `openclaw security audit` reports where a deployment has drifted from the safe defaults.

---

## OpenHands: Autonomous Developer Agent

### What It Is

OpenHands (formerly OpenDevin) is an open-source autonomous AI software engineer. Licensed under MIT, it can modify code, execute commands, browse the web, and interact with APIs. Unlike tools that suggest code snippets, OpenHands clones repositories, runs terminal commands, executes tests, and debugs errors inside a sandboxed runtime, typically a Docker container.

### Architecture of the v1.x OpenHands Agent (pre-Agent Canvas)

This is the agent runtime as it shipped before Agent Canvas; the current packaging is summarized at the end of this section.

```
+-------------------------------------------------------------------+
|              OpenHands v1.x Agent (pre-Agent Canvas)              |
+-------------------------------------------------------------------+
|                                                                   |
|  +-------------------+                                            |
|  |    User / API     |                                            |
|  +--------+----------+                                            |
|           |                                                       |
|           v                                                       |
|  +--------+----------+     +------------------+                   |
|  |  Agent Controller |<--->|  Event Stream    |                   |
|  |  (CodeActAgent)   |     |  Hub             |                   |
|  +--------+----------+     +--------+---------+                   |
|           |                         |                             |
|           v                         v                             |
|  +--------+----------+     +--------+---------+                   |
|  |  Action Dispatch  |     |  Observation     |                   |
|  |                   |     |  Collector       |                   |
|  |  - CmdRunAction   |     |                  |                   |
|  |  - FileWriteAction|     |  - CmdOutput     |                   |
|  |  - BrowseURLAction|     |  - FileContent   |                   |
|  |  - CodeAction     |     |  - BrowserState  |                   |
|  +--------+----------+     +------------------+                   |
|           |                                                       |
|           v                                                       |
|  +--------+--------------------------------------------------+    |
|  |                Docker Sandbox (Per Session)               |    |
|  |                                                           |    |
|  |  +--------------+  +--------------+  +--------------+     |    |
|  |  | Terminal     |  | Python       |  | Browser      |     |    |
|  |  | (bash)       |  | (stateful)   |  | (BrowserGym) |     |    |
|  |  +--------------+  +--------------+  +--------------+     |    |
|  +-----------------------------------------------------------+    |
+-------------------------------------------------------------------+
```

**Key architectural decisions in the v1.x agent:**
- **Event-stream architecture**: All agent-environment interactions flow as typed events through a central hub. The Agent analyzes conversation state and produces Actions; the sandbox produces Observations.
- **Per-session Docker containers**: Each session gets its own isolated container with full OS capabilities. The container is insulated from the host.
- **CodeAct-style agent**: The agent acts by writing and running bash or Python (the CodeAct approach of executable code as the action space) instead of choosing from a fixed set of JSON tool calls, and maintains session-level project context.
- **BrowserGym integration**: Agents can conduct browser automation via declarative primitives (DOM manipulation, navigation).
- **SDK composability**: The OpenHands SDK is a Python library. You can define agents in code, run them locally, or scale to thousands in the cloud.

**Updates in v1.6.0 (March 2026):**
- Kubernetes support for orchestrating agent sessions
- Planning Mode beta for multi-step task decomposition
- 2,100+ contributions from 188+ contributors

The repository has since moved to the `OpenHands` GitHub organization and has about 90K stars. Since v1.24.0 (September 25, 2026) OpenHands ships as Agent Canvas, a self-hosted control center that runs the OpenHands agent or any ACP agent (Claude Code, Codex, Gemini) on local, Docker, VM or cloud backends; see the [open coder guide](../09-frameworks-and-tools/10-opencoderguide.md#openhands-formerly-opendevin).

---

## Open Interpreter: Local Code Execution

### What It Is

Open Interpreter is a local code execution agent that provides a ChatGPT-like terminal interface. Instead of showing code and asking you to run it, Open Interpreter asks for permission and then executes it directly on your machine with full access to your local files.

**Status change (mid-2026):** the main `openinterpreter` repository now ships a different product. It is a Rust fork of OpenAI's Codex under Apache-2.0, replacing the AGPL-3.0 Python code; Rust releases began in June 2026, the main branch was rebased onto Codex in mid-July, and 0.0.55 shipped on September 30. It is pitched as "a coding agent optimized for low-cost models": it emulates the harnesses that get the most out of models such as Kimi K3 and GLM-5.3, switches between built-in harness profiles (Claude Code, Kimi CLI, Qwen Code, DeepSeek TUI and others), supports MCP and skills, and runs commands inside native sandboxing on macOS, Linux and Windows. The architecture below describes the original Python project, whose last PyPI release (0.4.3) dates from October 2024; a small community fork (`endolith/open-interpreter`, still AGPL-3.0) keeps it going. The pivot is itself a data point: harness design, not just model choice, now differentiates coding agents.

### Architecture

```
+-------------------------------------------------------------------+
|                   Open Interpreter Architecture                   |
+-------------------------------------------------------------------+
|                                                                   |
|  +-------------------+                                            |
|  |  Terminal UI      |                                            |
|  |  (ChatGPT-like)   |                                            |
|  +--------+----------+                                            |
|           |                                                       |
|           v                                                       |
|  +--------+----------+     +------------------+                   |
|  |  Core Engine      |<--->|  LLM Provider    |                   |
|  |                   |     |  (100+ models)   |                   |
|  |  - NL to Code     |     |  GPT, Claude,    |                   |
|  |  - Permission     |     |  Ollama, LM      |                   |
|  |    Gate           |     |  Studio, etc.    |                   |
|  +--------+----------+     +------------------+                   |
|           |                                                       |
|           v                                                       |
|  +--------+----------+                                            |
|  |  Code Executor    |                                            |
|  |                   |                                            |
|  |  - Python         |                                            |
|  |  - JavaScript     |                                            |
|  |  - Shell/Bash     |                                            |
|  |  - AppleScript    |                                            |
|  +--------+----------+                                            |
|           |                                                       |
|           v                                                       |
|  +--------+----------+                                            |
|  |  Computer API     |                                            |
|  |  (GUI Control)    |                                            |
|  |                   |                                            |
|  |  - Screen capture |                                            |
|  |  - Mouse/Keyboard |                                            |
|  |  - Icon detection |                                            |
|  +-------------------+                                            |
+-------------------------------------------------------------------+
```

**Key properties:**
- **Model flexibility**: Works with 100+ LLMs. Use a frontier API model for maximum capability, or run entirely offline with Ollama and LM Studio for privacy.
- **Permission gate**: Every code execution requires user approval (can be disabled for trusted workflows).
- **Computer API**: Beyond code execution, Open Interpreter can see your screen, identify UI elements, and control your mouse and keyboard, elevating it from a code interpreter to a computer automation agent.
- **Unsandboxed by default (Python version)**: Runs directly on the host machine. This is a deliberate design choice for maximum capability, but it means a bad LLM output can damage your system. Docker sandboxing is optional. The Rust rewrite reverses this default with native OS sandboxing.

### When to Use Open Interpreter

The Python design suits data analysis, file manipulation, and system administration tasks where you want a conversational interface to your local machine. It is not ideal for production deployments or untrusted environments. The Rust rewrite is a different tool: a sandboxed coding agent for teams that want to run cheap open-weight models through a harness tuned for them.

---

## Claude Computer Use: Vision-Based Automation

### What It Is

Claude Computer Use is an Anthropic API feature that allows Claude to control a desktop via screenshots, mouse movements, keyboard input, and application interaction. Introduced in October 2024 as a beta, it reached general availability on the Claude API on August 19, 2026 (Google Cloud on August 20) as the `computer_toolset_20260801` client toolset. It remains in beta on Bedrock, Claude Platform on AWS and Foundry. Supported models are Fable 5/5.1, Mythos 5/5.1, Opus 5/5.5, Sonnet 5/5.5, Haiku 5.5 and Opus 4.8.

### The Vision-Action Loop

```
+-------------------------------------------------------------------+
|              Claude Computer Use: Vision-Action Loop               |
+-------------------------------------------------------------------+
|                                                                   |
|  Step 1: OBSERVE          Step 2: REASON          Step 3: ACT    |
|  +----------------+       +----------------+      +------------+ |
|  |  Take          |       |  Analyze       |      |  Execute   | |
|  |  Screenshot    |------>|  Screenshot    |----->|  Action    | |
|  |  (base64 PNG)  |       |  + Task Goal   |      |  (click,   | |
|  |                |       |  + History      |      |  type,     | |
|  +----------------+       +----------------+      |  scroll)   | |
|                                                    +------+-----+ |
|                                                           |       |
|          +------------------------------------------------+       |
|          |                                                        |
|          v                                                        |
|  +-------+--------+                                               |
|  |  Wait + Take   |                                               |
|  |  New Screenshot|-------> (Loop back to Step 1)                 |
|  +----------------+                                               |
|                                                                   |
+-------------------------------------------------------------------+
```

### Available Tools

| Tool | Capability | Notes |
|------|------------|-------|
| `computer_toolset_20260801` | 17 member tools: screenshot, zoom, left/right/middle/double/triple click, drag, mouse move and button down/up, cursor position, scroll, type, key, hold_key, wait | GA; zoom on by default; no display-size parameters (the model reads coordinates off your screenshots); members can be disabled or deferred individually |
| `browser_toolset_20260801` | 31 member tools (27 on by default, 4 opt-in): `read_page` returns the accessibility tree with element refs such as `[ref_2]`, plus find, form input, tab management and pixel actions | For a browser your own application hosts; Claude API and Google Cloud only |
| `bash` (`bash_20250124`) | Run shell commands | Persistent session across turns |
| `text_editor` (`text_editor_20250728`) | Read/write/edit files | Supports view, create, str_replace |

### 2026 Enhancements

- **Batch actions**: Claude can return several `tool_use` blocks in one turn. The executor runs them in order and halts at the first failure, answering each later block with an error ("Not executed: an earlier computer action in this turn failed"). This cuts model round trips, and it means a confirmation placed between model turns arrives too late, because one turn can now complete a multistep consequential action. Anthropic's docs put the human check before each block runs: inspect the whole batch when it arrives and pause before each consequential action, including one in the middle of a batch.
- **Zoom Action**: Inspects small UI elements at high resolution before clicking. Reduces misclick rates on dense interfaces. On by default in the GA toolset.
- **Element refs for browsers**: The browser toolset targets accessibility-tree refs, which survive layout shifts but go stale after navigation. Anthropic scans returned page text and screenshots for prompt injection.
- **Breaking change**: On the Claude API and Google Cloud, the 5.5 models (Opus, Sonnet and Haiku) return HTTP 400 for the older `computer_20251124` tool (Bedrock still accepts it). Code written against the beta tool needs migrating.
- **Screenshot budget**: Models from Opus 4.7 onward accept up to 2576 px on the long edge. Anthropic estimates roughly 1,000-1,800 input tokens per screenshot and recommends keeping 20 or fewer images per request or resizing to 2000 px or less per side.
- **Consumer surfaces**: Cowork is being merged with chat into one Claude app (rollout to Pro and Max from September 16, 2026), with Claude asking for approval before acting by default.
- **Sandboxing best practice**: Always run in a sandboxed VM (Docker + VNC, or E2B cloud). Never give computer-use access to an unsandboxed host machine. For browsers, Anthropic's guidance adds a fresh profile with no credentials and a network-layer domain allowlist that blocks loopback, link-local and private ranges.

### Performance Trajectory

| Date | Benchmark | Score | Key Milestone |
|------|-----------|-------|---------------|
| Oct 2024 | OSWorld | 14.9% | Beta launch (Claude 3.5 Sonnet) |
| Feb 2025 | OSWorld | 28.0% | Claude 3.7 Sonnet |
| Mid 2025 | OSWorld | ~42-44% | Claude Sonnet 4 / Opus 4.1 |
| Q1 2026 | OSWorld-Verified | 72.5% | Sonnet 4.6, Zoom Action |
| Sep 2026 | OSWorld 2.0 (v2.1 full set, XLANG leaderboard) | 44.33% binary / 77.67% partial | Opus 5, max effort, batched tools |
| Sep 2026 | OSWorld 2.1 (Anthropic, vendor-reported) | 81.8% partial | Opus 5.5 |
| Oct 2026 | OSWorld 2.1 offline subset (Anthropic, vendor-reported) | 72.4% partial | Haiku 5.5 |

Do not read these rows as one trend line. Self-reported OSWorld-Verified scores now sit in the mid-80s, so that benchmark is saturated. **OSWorld 2.0** (June 2026) has 108 long-horizon tasks with a median of about 1.6 skilled-human hours each, scored primarily on binary completion at a 500-step budget. Quoted numbers swing 30-40 points with binary vs partial scoring, effort level and task-release version, so always ask which one is being cited. The interview point: short, single-session desktop tasks are close to solved, but hour-scale binary completion is still under 50% on the public leaderboard.

---

## Claude Code: The Terminal Agent

### What It Is

Claude Code is Anthropic's agentic coding tool that lives in the terminal. It reads your codebase, edits files, runs commands, and integrates with development tools. It shipped publicly in May 2025 and crossed $2.5 billion ARR by February 2026.

### Architecture

Claude Code is a TypeScript terminal agent that loops through three phases:

```
+-------------------------------------------------------------------+
|                   Claude Code Agent Loop                          |
+-------------------------------------------------------------------+
|                                                                   |
|  +------------------+                                             |
|  |  1. GATHER       |  Read files, grep codebase, glob search,   |
|  |     CONTEXT      |  check git status, analyze structure        |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+---------+                                             |
|  |  2. TAKE         |  Edit files, run bash, write new files,     |
|  |     ACTION       |  create commits, spawn subagents            |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+---------+                                             |
|  |  3. VERIFY       |  Run tests, check build, review diffs,     |
|  |     RESULTS      |  validate output                            |
|  +--------+---------+                                             |
|           |                                                       |
|           +--------> (Loop back to Step 1 if not done)            |
|                                                                   |
+-------------------------------------------------------------------+

Built-in Tools: bash, read, write, edit, glob, grep, browser,
                subagent, notebook, web_search, web_fetch
```

**Key architectural properties:**
- One agent loop with a rich tool palette
- On-demand skill loading via slash commands and CLAUDE.md
- Context compression for long sessions (1M+ token context)
- Subagent spawning for parallel workstreams
- Worktree isolation for parallel branch execution
- Permission governance (allow/deny rules for tools). Since v2.1.284 (September 28, 2026), interactive sessions with no permission mode configured start in **auto mode**, where a server-side classifier decides whether each action runs, is denied, or asks. Headless runs differ: since v2.1.302 (October 16), `claude -p` and Agent SDK sessions with no mode start in `default` mode on third-party providers, with telemetry off, or on a fresh install, so CI callers should set the permission mode explicitly
- Task system with dependency graphs
- Hooks for custom automation (pre/post commit, file changes), and since v2.1.287 (October 1, 2026) **mods**: TypeScript functions shipped inside plugins that can rewrite prompts, block or rewrite tool calls, approve or deny permission requests, and redact tool output. Anthropic warns they are not sandboxed and run with Claude Code's own access, so plugin review is now part of the agent's trust boundary
- Self-hosted runners (`claude self-hosted-runner`, Team and Enterprise) for running sessions on your own infrastructure

**Not the only serious terminal harness.** In early November 2026: OpenCode (MIT, about 224K stars; new installs go to the V2 line, `@opencode/cli` 2.2.5, which dropped the OpenCode GitHub Action in 2.2.0 on October 27), Codex CLI (Apache-2.0, about 133K stars; its default model switched to GPT-6.1 Sol on September 29), and Google's Antigravity CLI are the main alternatives, alongside the anthropics/claude-code repo at about 154K stars. Gemini CLI (Apache-2.0, about 109K stars) still ships, but only for Google's enterprise Code Assist licenses. Aider has gone quiet (last release February 12, last commit May 22, 2026). Silent default-model swaps like Codex's are a reason to pin the model explicitly in any automation built on these CLIs.

---

## IDE Agents: Cursor, Devin Desktop, Cline

### Cursor

Cursor is a VS Code fork with deep AI integration. By early 2026 it offered:
- **Agent Mode**: Uses 20x scaled reinforcement learning for multi-file editing
- **Background Agents** (now called cloud agents): Clone your repo in cloud VMs, work autonomously, open PRs when done
- **Mission Control**: Dashboard for managing parallel agent workflows
- **Market (early 2026)**: $2B annualized revenue, 2M+ users, 1M+ paying customers, adopted by half the Fortune 500

**Ownership changed in August 2026.** Cursor announced on August 14 that it is part of SpaceX (TechCrunch reported the close on August 15, at about $60B in SpaceX stock). Its blog now hosts SpaceXAI's Grok launches, and the announcement said nothing about third-party model access or data-use commitments. Two weeks later the risk became concrete: on August 28 OpenAI told SpaceX it would wind down its contract supplying OpenAI models to Cursor, with a proposed shutoff of November 12, 2026, and no future OpenAI models (GPT-6 Astra included) in the meantime. The notice covers only OpenAI's models. The lesson generalizes: when a tool vendor is owned by a model vendor, a rival's models can disappear on contractual notice, so keep model choice in your own routing layer. Cursor also holds AIUC-1 agent certification (August 13).

What shipped since, all vendor-described:
- **Self-hosted Machines** (September 2): cloud agents run on your own machines or autoscaling team pools (AWS Lambda, Coder, Cloudflare, Daytona, Modal, Namespace, Vercel or E2B), with computer use on Linux and Mac workers. This is the data-residency answer for cloud agents.
- **Projects** (beta, September 10): a coordinator agent plans the work, fans it out to as many parallel subagents as needed, keeps context for months, and returns finished work for review. Cursor reports that new Projects users merge 30% more PRs (vendor-reported, internal data).
- **Rollouts and Security Review** (September 23, Teams and Enterprise): a monitor attached to every PR that tracks change health per environment after merge, plus a PR scanner for exploitable bugs.

### Devin Desktop (formerly Windsurf)

Windsurf (originally Codeium, acquired by Cognition for $250M in July 2025) was renamed **Devin Desktop** in v3.0.12 (June 2, 2026), and the rename was more than cosmetic:
- **Devin Local**: since v3.9.19 (September 8, 2026) the only agent in the product. It replaced Cascade, Windsurf's original multi-step agent, and Cognition describes it as the same harness its terminal agent uses
- **Agent Command Center**: a Kanban-style view of local and cloud agent sessions, with one-click delegation from a local session to cloud Devin, which runs on its own VM
- **Proprietary models**: SWE-1.5 (13x faster than Sonnet 4.5, per Cognition) and Fast Context, followed by the SWE-2 flagship model (September 10, 2026)
- **Codemaps**: AI-powered visual code navigation

Cognition (Devin Desktop and cloud Devin) raised more than $2B at a $48B valuation on September 8, 2026 and reports over $1B in annualized revenue run rate (September 25, vendor-reported): an independent coding-agent company that owns both an IDE and its own models. The design lesson from the rename: when a vendor swaps the agent inside a product you have standardized on, prompts, rules files and approval habits tuned for the old agent need re-evaluation, so treat it as a migration.

### Cline

Cline is a VS Code extension that operates as a full agent rather than an autocomplete tool. It takes a series of steps, evaluates results, fixes its own errors, and continues. More autonomous than Cursor or Devin Desktop but with less polish. It is Apache-2.0 and ships an SDK; v4.1.26 is from October 26, 2026.

### GitHub Copilot

Copilot now ships orchestration patterns that were paper ideas a year ago:
- **Computer use** (public preview, October 1, 2026, Copilot CLI and the Copilot app on macOS and Windows): operates desktop apps with per-app approval, reviewable permanent grants, and an org-level off switch.
- **Dynamic workflows** (public preview, October 1): code-defined programs that mix automated steps and agents in sequential or parallel stages, with checkpoints for human review. GitHub contrasts them with `/fleet`, which lets the model delegate.
- **Copilot Fusion** (GA October 28, 2026, on all Copilot plans; it grew out of the HydraFusion research preview): you pick Standard, High or Max effort, and Fusion picks a workflow per task. **Single** sends the task to one model; **cascade** drafts with an efficient model, gates on quality, and escalates to a stronger one; **critique** has a second model review the draft before the first revises it. Each model used bills at its own rate, so the cost of a task now depends on the route Fusion picks.

### IDE Agent Architecture Comparison

```
+-------------------------------------------------------------------+
|                 IDE Agent Architecture Patterns                   |
+-------------------------------------------------------------------+
|                                                                   |
|  Cursor:                                                          |
|  [Editor] --> [Agent Mode] --> [Multi-file RL] --> [Apply Diffs]  |
|                    |                                              |
|                    +--> [Cloud Agent] --> [Cloud/own VM] --> [PR] |
|                                                                   |
|  Devin Desktop:                                                   |
|  [Editor] --> [Devin Local] --> [Repo Context] --> [Apply Edits]  |
|                    |                                              |
|                    +--> [Cloud Devin, own VM] --> [PR]            |
|                                                                   |
|  Cline:                                                           |
|  [Editor] --> [Agent Loop] --> [Evaluate] --> [Self-Fix] --> [Act]|
|                    |                                              |
|                    +--> [Any LLM Provider] --> [Tool Calls]       |
+-------------------------------------------------------------------+
```

---

## Comparison Matrix

| Feature | OpenClaw | OpenHands | Open Interpreter | Claude Computer Use | Claude Code | Cursor |
|---------|----------|-----------|-----------------|-------------------|-------------|--------|
| **Type** | Local agent | Dev agent | Local code exec | Vision automation | Terminal agent | IDE agent |
| **License** | MIT | MIT | Apache-2.0 (Rust rewrite); AGPL-3.0 (Python fork) | Proprietary API | Proprietary | Proprietary |
| **GitHub Stars (Nov 2026)** | ~394K | ~92K | ~69K | N/A (API) | ~154K (issues and plugins repo) | N/A |
| **Sandboxed** | Optional (main session on host by default) | Yes (Docker) | Native OS sandbox (Rust); No (Python) | Requires VM | Configurable | Yes (cloud agents) |
| **LLM Support** | Any (model-agnostic) | Any | 100+ models | Claude only | Claude only | Multi-model |
| **GUI Control** | Browser tool | Yes (BrowserGym) | Computer API (Python fork); browser and app QA skills (Rust) | Yes (native) | Via computer-use | Computer use on cloud agent workers |
| **Code Execution** | Yes (exec tool) | Yes (container) | Yes (local) | Yes (bash tool) | Yes (bash) | Yes (terminal) |
| **Messaging** | 24+ platforms | Web UI / API | Terminal | API | Terminal / IDE | Editor |
| **Memory** | Persistent (local) | Session-based | Session-based | Per-conversation | Session + CLAUDE.md | Project-scoped |
| **MCP Support** | Native (client and server) | Limited | Yes (Rust); No (Python) | Via Claude | Native | Native (loaded dynamically) |
| **Best For** | Personal assistant | Autonomous dev | Cheap-model coding (Rust); data analysis (Python) | GUI automation | Professional dev | IDE workflow |
| **Risk Level** | High (unsandboxed by default) | Low (sandboxed) | High for the Python fork; lower for the Rust rewrite | Medium (needs VM) | Medium | Low |

---

## Market Trends and Adoption (2026)

### The Numbers

- **MCP ecosystem**: 10,000+ active servers. In October 2026 the v1 TypeScript SDK alone had about 246M npm downloads against about 62M for v2 `@modelcontextprotocol/core`, 49M for `server` and 33M for `client`: v2 roughly doubled in a month but is still only 13 to 25% of v1 volume. Tier 1 SDKs now cover TypeScript, Python, C#, Go, Rust and Ruby
- **Gartner projection**: 40% of enterprise applications will incorporate AI agents by end of 2026 (up from under 5% in early 2025)
- **OpenClaw**: Fastest project to 300K GitHub stars in history (under 5 months); about 394K in early November 2026
- **Claude Code**: $2.5B ARR by February 2026, the fastest enterprise software product to $1B
- **Cursor**: $2B annualized revenue and half of the Fortune 500 (early 2026); part of SpaceX since August 2026, and OpenAI is winding down its model supply to Cursor (proposed cutoff November 12, 2026)
- **Cognition**: valued at $48B after a $2B+ round (September 8, 2026)

### Key Trends

**1. Convergence of Agent Types**: The boundaries between local, cloud, and IDE agents are dissolving. Claude Code runs locally but uses cloud models. Cursor's cloud agents run on Cursor's VMs or on your own machines. OpenClaw connects to any LLM. The pattern is moving toward a universal agent architecture that can operate in any environment.

**2. MCP as the Universal Tool Layer**: MCP has become the standard for tool integration, with adoption by Anthropic, OpenAI, Google, and hundreds of tool providers, under the Linux Foundation's Agentic AI Foundation (10 hosted projects and 305 member organizations as of October 22, 2026). The current spec revision, 2026-07-28, made the protocol core stateless. The maintainers' August 22, 2026 roadmap prioritizes agentic messaging (Tasks, server-initiated events), Streamable HTTP as the single transport, agent identity (DPoP and workload identity federation, both still open drafts), a redesign of the `tools/call` result shape, and server-side progressive discovery for large tool catalogs. The next revision is targeted for March 30, 2027 (SEP-3398, accepted October 14 and merged October 27, subject to confirmation), with releases after that generally on the last Tuesday of March and September. Extensions now carry maturity stages (SEP-3392): MCP Apps and enterprise auth are Stable, while Tasks, Skills and Server Cards are Beta and can still break.

**3. Sandboxing Becomes Non-Negotiable**: The OpenClaw security crisis (135,000 exposed instances) has pushed the industry toward sandboxed-by-default architectures. New agents are expected to provide isolation out of the box.

**4. Cost Optimization as First-Class Concern**: The Plan-and-Execute pattern (a capable model plans, cheaper models execute) cuts cost by roughly 85-90% in illustrative math (see the next chapter). For coding agents the bigger lever is now input tokens: Anthropic's Claude Code telemetry (September 24, 2026, vendor-reported) shows the input-to-output ratio rising from 189:1 to 324:1, so cache-read price and cache hit rate dominate the bill. Opus 5.5 and Sonnet 5.5 both read cache at 0.05x input ($0.20 and, since October 7, $0.10 per 1M tokens). This is the agentic equivalent of cloud cost optimization.

**5. Background and Asynchronous Agents**: Cursor's cloud agents and Projects, Codex cloud environments, and Claude Code's subagent spawning represent a shift from synchronous, interactive agents to autonomous, asynchronous workers that notify you when done. OpenAI dots and Meta Muse push the same idea to consumers: an always-on agent with its own computer.

**6. The Harness Becomes the Product**: Anthropic (Claude Managed Agents), OpenAI (Agents API) and AWS (Bedrock Managed Agents, built on OpenAI's harness) now sell the agent loop as a managed runtime, and the coding vendors converge on a **split plane**: vendor control plane, customer-hosted execution plane. Build-vs-rent is now a design-review question, and so are the gaps: the OpenAI Agents API launched with US data residency only and no Zero Data Retention, and in November 2026 all three managed runtimes were still in beta or preview. Billing models are converging too: since October 9, OpenAI bills hosted sandbox compute only during active turns, so an agent waiting on a human approval no longer burns sandbox time.

---

## System Design Interview Angle

When asked about tool-use agents in system design interviews, focus on these dimensions:

**1. Security Model**: Is execution sandboxed? How are credentials managed? What happens if the LLM generates malicious code? (OpenClaw's host execution by default vs. OpenHands' per-session Docker isolation is a great comparison point; both are MIT-licensed, so license is not the differentiator.)

**2. State Management**: How does the agent maintain context across tool calls? Session-based (OpenHands) vs. persistent memory (OpenClaw) vs. file-based (Claude Code's CLAUDE.md)?

**3. Tool Discovery**: Static manifest (old approach) vs. dynamic discovery via MCP vs. skill marketplace (OpenClaw ClawHub)?

**4. Latency Budget**: Function calling (50-200ms per tool call) vs. vision-based automation (1-3 seconds per screenshot-action loop). Batch actions amortize one model round trip over several GUI steps, and element refs cut the retries that misclicks cause. How does this affect UX, and where does the confirmation gate sit when one turn can do five things?

**5. Failure Handling**: What happens when a tool call fails? Retry? Fallback? Human-in-the-loop? How many retries before giving up?

**6. Hosting Model**: Build your own loop, rent a managed harness, or split the planes? Check residency, retention and identity per option, and who can stop a runaway agent. OpenAI's September 2026 disclosure is the cautionary case: in an internal training run, an agent's DNS-tunnel egress set off an alarm about 12 minutes in, but the run was killed only about 2.5 hours later because the automatic stop failed.

---

## Interview Questions

### Q: Your team wants to build an internal AI assistant. Should you build on OpenClaw, OpenHands, or build custom with Claude Code + MCP?

**Strong answer:**
It depends on the use case and security requirements. OpenClaw is optimized for personal assistants with messaging integrations, ideal if the goal is a Slack/Teams bot with persistent personality. It is MIT-licensed and now foundation-governed with an LTS-style release line, but its single-operator trust model and host execution by default create enterprise concerns (its own docs say it does not support hostile multi-tenant setups). OpenHands is better for autonomous development tasks: per-session Docker sandboxing and an MIT license are enterprise-friendly. For a custom internal tool, an agent loop with MCP servers gives the most control: you define exactly which tools are available, run them in your own infrastructure, and benefit from MCP's standardized discovery and auth. Since September 2026 there is a fourth option: rent the loop (Claude Managed Agents, OpenAI Agents API, Bedrock Managed Agents) and keep only tools and data in your perimeter, after checking residency and retention terms. The decision tree is: messaging-first? OpenClaw. Dev automation? OpenHands. Custom enterprise tool? MCP plus your own or a managed agent loop.

### Q: How would you design a system that lets non-technical users automate desktop tasks using AI?

**Strong answer:**
I would use the vision-based computer-use pattern (Claude's computer toolset, Gemini computer use, or similar). The key design decisions: (1) Always run in a sandboxed VM so the agent cannot damage the user's actual machine. (2) Implement a Human-in-the-Loop confirmation step before any destructive action (file deletion, form submission, purchases), and run it before each consequential action executes, including one in the middle of a batch, since one model turn can now emit several actions. Gemini's built-in safety service returning `require_confirmation` for categories like financial transactions and account creation is a ready-made risk taxonomy for that gate. (3) Use zoom and, for browsers, accessibility-tree element refs to reduce misclicks. (4) Set token/cost caps to prevent runaway loops. (5) Record all actions as an audit trail. The main tradeoff is latency (1-3 seconds per model round trip), but this approach works with any application without needing APIs. For higher-speed workflows, combine computer-use with function calling for applications that have APIs.

### Q: Why did OpenClaw grow faster than any open-source project in history? What does this tell you about the market?

**Strong answer:**
Three factors. (1) **Zero-friction onboarding**: OpenClaw connects to messaging platforms people already use (WhatsApp, Telegram). Users do not need to learn a new interface. (2) **SOUL.md personalization**: The ability to give your agent a custom personality creates emotional attachment and virality: people share their agents. (3) **Model-agnostic architecture**: Users are not locked into one LLM provider, reducing cost and increasing flexibility. The market signal is that the agent "interface" matters more than the underlying model. People want agents that meet them where they are (messaging apps, not web UIs). The flip side: rapid growth without security investment leads to crises like the 135,000 exposed instances, which is a cautionary tale for any open-source agent project.

### Q: Design an always-on personal agent that keeps working while the user is away. What does it need beyond a chat agent?

**Strong answer:**
Treat it as a tenant, not a session. (1) **Isolation**: one VM or sandbox per agent (OpenAI dots and Meta Muse both ship this way), with the agent's browser open to the user for inspection, as dots allows. (2) **Identity and secrets**: the agent gets its own identity, and credentials sit in a vault the model cannot read; dots sign in with saved passwords without exposing them to the model, and the OpenAI Agents API takes credentials through a separate sign-in UI. (3) **Policy, not prompts**: declarative rules that allow, block or require approval per action type, with per-origin approval for new websites. Per-call prompts cause approval fatigue on an always-on agent. (4) **Idle mode**: when no task is active, restrict the agent to read-only work, as dots does. (5) **Budgets and kill switches**: session spend caps that pause the agent (Claude Managed Agents' `budget_reached`) and an automatic stop that is tested, because an alarm without an automatic kill is not a control. (6) **Egress**: allowlist destinations including DNS, and treat public repos, gists and image hosts as exfiltration channels. Close with the open questions: who is liable for a transaction the agent made, and how a user audits a day of background activity.

### Q: Compare sandboxed vs. unsandboxed execution for AI agents. When would you choose each?

**Strong answer:**
Sandboxed (Docker/VM): Use for untrusted code execution, multi-tenant systems, or any production deployment. OpenHands does this well: each session gets its own Docker container. The trade-off is setup complexity and performance overhead, which managed snapshot-restore runtimes are shrinking (AWS reports about 2 s P75 cold starts for AgentCore Runtime V2). Unsandboxed (host access): Use only for single-user, trusted environments where the user is watching. The original Python Open Interpreter and OpenClaw's main session take this approach for maximum capability; tellingly, Open Interpreter's 2026 Rust rewrite switched to native OS sandboxing. The risk is that a bad LLM output can damage the host system. The 2026 consensus is sandboxed-by-default with escape hatches for power users. In an interview, always mention that the sandbox boundary is a security decision, not just a convenience decision.

---

## References

- OpenClaw GitHub Repository and Documentation (2025-2026)
- OpenHands Documentation and SDK Reference (2025-2026)
- Open Interpreter GitHub Repository (2024-2026)
- Anthropic. "Computer Use Tool Documentation" (2024-2026)
- Anthropic. "Browser Use Tool Documentation" (2026)
- Anthropic. "Claude Code Overview" and changelog (2025-2026)
- MCP Specification 2026-07-28 and "The New MCP Roadmap" (August 22, 2026)
- OpenAI. "Introducing the Agents API" and "Introducing dots" (September 2026)
- OpenAI. "Our decision on Cursor following its acquisition by SpaceX" (August 28, 2026)
- Google. Gemini API computer use documentation and changelog (2026)
- XLANG Lab. OSWorld 2.0 leaderboard (September 2026)
- Gartner. "AI Agent Adoption Projections" (2025-2026)
- Cursor, Devin Desktop (formerly Windsurf), Cline, and GitHub Copilot official documentation and changelogs (2025-2026)
- GitHub. "Route and orchestrate models with GitHub Copilot Fusion" (changelog, October 28, 2026)
- Model Context Protocol. SEP-3398 "Release Cycle Updates" and SEP-3392 "Extension Lifecycle" (October 2026)

---

*Next: [Architecture Patterns for Tool-Use Agents](02-architecture-patterns.md)*
