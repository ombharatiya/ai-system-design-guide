# Architecture Patterns for Tool-Use Agents

Every tool-use agent in 2026, from OpenClaw to Claude Code to Cursor's cloud agents, is built on one of a handful of core architecture patterns. Understanding these patterns lets you design agents from first principles rather than copying specific tools. This chapter breaks down each pattern with detailed diagrams, code examples, trade-offs, and guidance on when to use which.

## Table of Contents

- [Pattern 1: Function/Tool Calling](#pattern-1-functiontool-calling)
- [Pattern 2: Vision-Based Automation](#pattern-2-vision-based-automation)
- [Pattern 3: Local Code Execution](#pattern-3-local-code-execution)
- [Pattern 4: Multi-Agent Tool Orchestration](#pattern-4-multi-agent-tool-orchestration)
- [Sandboxed vs. Unsandboxed Execution](#sandboxed-vs-unsandboxed-execution)
- [State Management Across Tool Calls](#state-management-across-tool-calls)
- [Error Handling and Retry Patterns](#error-handling-and-retry-patterns)
- [MCP Integration Patterns](#mcp-integration-patterns)
- [Architecture Decision Tree](#architecture-decision-tree)
- [System Design Interview Angle](#system-design-interview-angle)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Pattern 1: Function/Tool Calling

The most widely deployed pattern in production. The LLM decides which tool to call and with what arguments; a framework executes the call; results are fed back into the conversation for the next reasoning step.

### Architecture

```
+-------------------------------------------------------------------+
|              Function/Tool Calling Pattern                        |
+-------------------------------------------------------------------+
|                                                                   |
|  +------------------+                                             |
|  |  User Message     |                                            |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+---------+     +------------------+                    |
|  |  LLM Reasoning   |---->|  Tool Selection  |                   |
|  |                   |     |                  |                    |
|  |  "I need to look  |     |  tool: search_db |                   |
|  |   up the order"  |     |  args: {id: 42}  |                    |
|  +-------------------+     +--------+---------+                   |
|                                     |                             |
|                                     v                             |
|                            +--------+---------+                   |
|                            |  Tool Executor   |                   |
|                            |  (Framework)     |                   |
|                            |                  |                   |
|                            |  Validates args  |                   |
|                            |  Calls function  |                   |
|                            |  Returns result  |                   |
|                            +--------+---------+                   |
|                                     |                             |
|                                     v                             |
|                            +--------+---------+                   |
|                            |  Result Injected |                   |
|                            |  into Context    |                   |
|                            |                  |                   |
|                            |  {status: "shipped",                 |
|                            |   tracking: "1Z..."} |               |
|                            +--------+---------+                   |
|                                     |                             |
|                                     v                             |
|                            +--------+---------+                   |
|                            |  LLM Generates   |                   |
|                            |  Final Response  |                   |
|                            +------------------+                   |
+-------------------------------------------------------------------+
```

### The Three Steps in Detail

**Step 1: Schema Presentation**. The model receives a JSON schema describing available tools. In 2026, best practice is to use Dynamic Manifests that fetch only relevant tools based on the user's intent, rather than loading all tool schemas upfront. Vendors now ship this: Anthropic's `inline-tools-2026-09-15` beta adds or replaces tools mid-conversation without invalidating the prompt cache, Cursor reports that moving MCP tools into dynamic context cut total tokens by 46.9% in sessions that called one (vendor-reported), and server-side progressive discovery for large catalogs is on MCP's August 2026 roadmap.

**Step 2: Intent and Extraction**. The model outputs a structured tool call. This is not free-form text; it is a JSON object with `tool_name` and `arguments` that the framework can parse deterministically. **Forcing a call is now model-dependent**: `tool_choice` of type `any` or `tool` returns HTTP 400 on Claude Fable 5.1, Mythos 5.1, Opus 5.5 and Sonnet 5.5. Use `auto` with strict tool definitions, or structured outputs when you need a guaranteed schema. Harnesses that forced a tool call for extraction or routing need rewriting before they move to these models. OpenAI has its own version of this: GPT-6 Astra supports tool calling only through the Responses API, so a Chat Completions harness cannot simply swap in the new model ID.

**Step 3: Execution and Contextualization**. The framework validates arguments (using Pydantic, Zod, or similar), calls the function, and injects the result back into the conversation as a new message with role `tool`.

### Code Example: MCP Server + Client

```python
# MCP Server (Python SDK 2.x, where FastMCP was renamed MCPServer)
from typing import Annotated
from mcp.server import MCPServer
from pydantic import Field

mcp = MCPServer("order-service")

@mcp.tool()
async def lookup_order(
    order_id: Annotated[str, Field(description="The order UUID")],
) -> dict:
    """Look up an order by ID. DO NOT use for canceled orders."""
    order = await db.orders.find_one({"id": order_id})
    if not order:
        return {"error": "Order not found", "suggestion": "Check order ID format"}
    return {"status": order["status"], "tracking": order.get("tracking_number")}
```

```python
# MCP Client: agent discovers tools dynamically, calls them, feeds results back
listed = await mcp_client.list_tools()
tools = [to_model_tool_schema(t) for t in listed.tools]  # MCP listing -> model API format
response = client.messages.create(model="claude-sonnet-5-5", max_tokens=16000, tools=tools,
    messages=[{"role": "user", "content": "Where is my order ORD-12345?"}])

if response.stop_reason == "tool_use":
    # Thinking or text blocks can precede the call, so never assume content[0]
    tool_call = next(b for b in response.content if b.type == "tool_use")
    result = await mcp_client.call_tool(tool_call.name, tool_call.input)
    # Append the full assistant turn (thinking blocks included), then a tool_result block
```

`pip install mcp` has resolved to SDK 2.x since July 28, 2026, so tutorials that import `FastMCP` break on a fresh install; pin `mcp<2` if you are not ready to migrate. The Python SDK 2.1+ also changed error visibility: an unexpected exception in a handler reaches the client only as "Error executing tool <name>", so raise `ToolError` for messages the model should see.

### When to Use This Pattern

- API integrations (databases, SaaS tools, internal services)
- Structured data retrieval and mutation
- Any workflow where you can define the tool interface upfront
- Production systems where you need audit trails and input validation

### Trade-offs

| Advantage | Disadvantage |
|-----------|--------------|
| Deterministic execution | Requires tool schema upfront |
| Easy to audit and log | Cannot interact with arbitrary UIs |
| Fast (50-200ms per tool call) | Model may hallucinate tool names/args |
| Works with any LLM that supports tool use | Schema overload with many tools |

---

## Pattern 2: Vision-Based Automation

The model sees a screenshot of the screen, reasons about what to do, and emits a low-level action (click, type, scroll). The environment executes the action, takes a new screenshot, and the loop repeats. This is how Claude's computer toolset, Gemini computer use, and the original Python Open Interpreter's Computer API work.

### Architecture

```
+-------------------------------------------------------------------+
|              Vision-Based Automation Pattern                      |
+-------------------------------------------------------------------+
|                                                                   |
|  +------------------+                                             |
|  |  Task Goal        |  "Fill out the expense form with           |
|  |  (NL instruction) |   last week's receipts"                    |
|  +--------+---------+                                             |
|           |                                                       |
|           v                                                       |
|  +--------+--------------------------------------------------+    |
|  |                    VISION-ACTION LOOP                      |    |
|  |                                                            |    |
|  |   +------------+    +-------------+    +------------+     |    |
|  |   |  OBSERVE   |    |  REASON     |    |  ACT       |     |    |
|  |   |            |    |             |    |            |     |    |
|  |   | Screenshot |--->| Analyze     |--->| Emit action|     |    |
|  |   | (base64)   |    | screenshot  |    | {type:     |     |    |
|  |   |            |    | + goal      |    |  "click",  |     |    |
|  |   |            |    | + history   |    |  x: 450,   |     |    |
|  |   |            |    | + prev acts |    |  y: 320}   |     |    |
|  |   +-----^------+    +-------------+    +------+-----+     |    |
|  |         |                                      |          |    |
|  |         +--------------------------------------+          |    |
|  |                    (Loop until done)                       |    |
|  +-----------------------------------------------------------+    |
|                            |                                      |
|                            v                                      |
|  +-------------------------+------------------------------+       |
|  |         Sandboxed Environment (VM / Docker + VNC)      |       |
|  |                                                        |       |
|  |   +----------+  +----------+  +----------+            |       |
|  |   | Desktop  |  | Browser  |  | Apps     |            |       |
|  |   | (Xfce)   |  | (Chrome) |  | (any)    |            |       |
|  |   +----------+  +----------+  +----------+            |       |
|  +--------------------------------------------------------+       |
+-------------------------------------------------------------------+
```

### The Observe-Reason-Act Cycle

**Observe**: Capture a screenshot of the current screen state. In Claude Computer Use, this is a base64-encoded PNG sent as an image content block, at roughly 1,000-1,800 input tokens per screenshot by Anthropic's estimate. The Zoom Action (new in 2026, on by default in the GA toolset) allows capturing a high-resolution crop of a specific region for dense UIs. Claude's GA toolset takes no display dimensions: the model reads coordinates in the pixel space of the screenshots you return.

**Reason**: The multimodal LLM analyzes the screenshot alongside the task goal and action history. It decides what the next action should be. This step consumes the most tokens.

**Act**: The model emits a structured action:
- `left_click(x, y)`: click at coordinates
- `type(text)`: type a string
- `key(key_combo)`: press keyboard shortcut
- `scroll(direction, amount)`: scroll the page
- `screenshot()`: take a new screenshot without acting
- `zoom(x0, y0, x1, y1)`: inspect a region at high resolution

### Batch Actions and Element References

Two 2026 changes reshape this loop. First, **batch actions**: Claude's GA computer toolset (August 19, 2026) lets the model return several `tool_use` blocks in one turn. The executor runs them in order and halts at the first failure, answering every later block with an error instead of running it. One model round trip now covers several GUI steps, which changes the latency and cost model. It also means a human confirmation placed between model turns arrives too late, because a single turn can finish a multistep consequential action. Anthropic's docs put the check before each block runs: inspect every action in the batch when it arrives, pause before each consequential one, and treat a rejection like a failure so the rest of the batch is answered as not executed.

Second, **targeting moves off pixels where it can**. Claude's browser toolset returns the accessibility tree with element refs (`[ref_2]`) that clicks and form fills can target; refs survive layout shifts but go stale after navigation. Gemini computer use normalizes coordinates to a 0-999 grid and adds a safety service that returns `require_confirmation` for categories such as financial transactions, account creation and accepting legal terms. Treat pixels as the fallback for desktop apps with no accessible structure.

### Code Example: Computer Use Loop

```python
tools = [
    {"type": "computer_toolset_20260801"},  # GA toolset: no name, no display size
    {"type": "bash_20250124", "name": "bash"},
    {"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"},
]
messages = [{"role": "user", "content": "Open the browser and go to GitHub."}]
SKIPPED = "Not executed: an earlier computer action in this turn failed."

for step in range(50):  # The vision-action loop, with a hard step cap
    response = client.messages.create(
        model="claude-sonnet-5-5", max_tokens=16000, tools=tools, messages=messages)
    messages.append({"role": "assistant", "content": response.content})  # once per turn
    if response.stop_reason != "tool_use":
        break
    results, failed = [], False
    for block in (b for b in response.content if b.type == "tool_use"):
        if failed:  # batch semantics: stop at the first failure, answer every block
            ok, content = False, SKIPPED
        elif needs_approval(block) and not human_approves(block):  # gate each block, mid-batch too
            ok, content, failed = False, "Rejected by the user.", True
        else:
            # screenshot and zoom return an image block; other members return "OK"
            ok, content = sandbox.execute(block.name, block.input)
            failed = not ok
        result = {"type": "tool_result", "tool_use_id": block.id,
                  "content": content, "is_error": not ok}
        if getattr(block, "toolset_name", None):  # computer toolset members
            result["toolset_name"] = block.toolset_name
        results.append(result)
    messages.append({"role": "user", "content": results})  # all results in one message
```

The older `computer_20251124` tool returns HTTP 400 on the 5.5 models (Opus, Sonnet and Haiku) on the Claude API and Google Cloud (Bedrock still accepts it), and the GA toolset rejects the old `name` and display-size fields. Loops that append one assistant message per `tool_use` block, as many early samples did, also break once the model batches actions. One more trap on Opus 5.5 and Sonnet 5.5: do not prune old screenshots from the history to save tokens. Editing earlier turns invalidates the thinking blocks that follow, so resize screenshots before sending them (2000 px or less per side) and let server-side tool-result clearing trim old images instead.

### When to Use This Pattern

- Automating legacy applications with no API
- End-to-end testing of graphical interfaces
- Tasks that require interaction with multiple applications
- Non-developer users who describe tasks in natural language

### Trade-offs

| Advantage | Disadvantage |
|-----------|--------------|
| Works with any GUI application | Slow (1-3 sec per model round trip; batching amortizes it) |
| No API or integration needed | High token cost (about 1,000-1,800 input tokens per screenshot) |
| Handles dynamic UIs | Misclick risk on dense interfaces (zoom and element refs reduce it) |
| Accessible to non-technical users | Requires sandboxed VM for safety |

---

## Pattern 3: Local Code Execution

The user describes a task in natural language. The LLM generates code. The code runs on the local machine (or in a sandbox). The output is observed, and the LLM either generates more code or provides the final answer. This is how the original Python Open Interpreter and parts of Claude Code work.

### Architecture

```
User (NL): "Analyze the CSV and plot the top 10 products"
  |
  v
[LLM Generates Code] --> Python/Bash/JS
  |
  v
[Permission Gate] --> "Run this code? [y/N]" (auto-approve, always-ask, or rules-based)
  |
  v
[Code Executor] --> Execute, capture stdout/stderr/return value
  |
  v
[Output Observer] --> Error? Feed back to LLM for fix. Success? Present to user.
  |
  v
[LLM Decides] --> Done? Return result. Need more? Generate next code block. (Loop)
```

### The NL-Code-Execute-Observe Cycle

**1. Natural Language to Code**: The LLM translates the user's intent into executable code. The code language depends on the task: Python for data analysis, bash for system operations, JavaScript for web tasks.

**2. Permission Gate**: Before execution, the user is asked to approve. This is the critical safety mechanism for unsandboxed environments. Implementations vary:
- **Always ask** (the Python Open Interpreter's default): Every code block requires explicit approval
- **Auto-approve** (trusted mode): Dangerous but fast
- **Rules-based** (Claude Code model): Allow/deny patterns in configuration. For example: allow `git` commands, deny `rm -rf`
- **Classifier-based**: A policy model scores each action and runs it, denies it, or escalates to a human. This became a default in stages: Claude Code made auto mode the default for Pro, Max and Team on August 14, 2026, and since v2.1.284 (September 28) any interactive session with no configured permission mode starts in it. Headless runs are the exception: since v2.1.302 (October 16), `claude -p` and Agent SDK sessions with no mode start in `default` mode on third-party providers, with telemetry off, or on a fresh install, so a CI job that never sets the mode can behave differently across environments. Set it explicitly. Claude Managed Agents evaluates each tool call server-side under its auto permission policy (September 10). It trades approval fatigue for a calibration problem: you now have to choose and monitor the threshold at which the classifier defers

**3. Execute and Capture**: The code runs in a runtime with full (or restricted) system access. Stdout, stderr, return values, and any generated files are captured.

**4. Observe and Iterate**: The LLM sees the execution output. If there is an error, it generates a fix. If the output is partial, it generates the next step. This creates a self-correcting loop.

### Code Example: Code Execution Agent

```python
class CodeExecutionAgent:
    def __init__(self, llm_client, sandbox=None):
        self.llm = llm_client
        self.sandbox = sandbox  # None = unsandboxed (host)
        self.history = []

    async def run(self, task: str) -> str:
        self.history.append({"role": "user", "content": task})
        for iteration in range(10):  # Max 10 code-execute cycles
            response = await self.llm.generate(messages=self.history)
            code = extract_code_block(response)
            if not code:
                return response  # No code = final answer
            if not self.sandbox and not await user_approves(code):
                return "Execution canceled by user."
            result = await (self.sandbox or LocalExecutor()).run(code, timeout=30)
            self.history.append({"role": "assistant", "content": response})
            self.history.append({"role": "user",
                "content": f"stdout: {result.stdout}\nstderr: {result.stderr}"})
        return "Max iterations reached."
```

### When to Use This Pattern

- Data analysis and visualization tasks
- System administration and DevOps automation
- File processing and transformation
- Any task where the user describes "what" and the agent figures out "how"

### Trade-offs

| Advantage | Disadvantage |
|-----------|--------------|
| Extremely flexible | Security risk if unsandboxed |
| Self-correcting via observe loop | Model may generate dangerous code |
| Works offline with local models | Requires user to evaluate code (or trust) |
| Full system access when needed | Non-deterministic (same prompt, different code) |

---

## Pattern 4: Multi-Agent Tool Orchestration

Instead of one agent with many tools, you have multiple specialized agents that each own a subset of tools. An orchestrator routes tasks to the right agent. This is the "microservices revolution" for agents.

### Architecture

```
  [User Request]
       |
       v
  [ORCHESTRATOR] (Frontier model: Claude Opus 5.5, GPT-6 Astra)
  Analyzes task, selects agent, routes and waits
       |
  +----+----+----+
  |         |         |
  v         v         v
[Code Agent]  [Data Agent]  [Web Agent]
 bash, edit,   SQL, plot,    fetch, scrape,
 git           csv            browse
  |         |         |
  v         v         v
[Sandbox]  [Sandbox]  [Sandbox]
(Docker)   (Docker)   (Docker)
```

### Orchestration Strategies

**1. Router-Based (Simplest)**: The orchestrator is a classifier. It looks at the user's message, picks the right specialist agent, and forwards the entire task. No inter-agent communication.

**2. Plan-and-Execute**: A planning model (frontier-class) breaks the task into subtasks and assigns each to the appropriate specialist. Subtask results are aggregated by the planner. Much of the latency win comes from running independent subtasks in parallel: LLMCompiler (ICML 2024), which plans tool calls as a dependency graph, reported up to 3.7x lower latency and up to 6.7x lower cost than sequential ReAct on its benchmarks. Measure on your own tasks before quoting a number.

**3. Hierarchical**: High-level agents assign work to lower-level agents, which may further delegate. This mirrors organizational structures and works well for complex projects.

**4. Collaborative (Peer-to-Peer)**: Agents can communicate with each other directly, sharing observations and requesting help. This is the most complex pattern but handles emergent tasks well.

These strategies now ship as products. Cursor Projects (beta, September 10, 2026) is a hierarchical coordinator that fans work out to parallel subagents and keeps context for months. GitHub Copilot draws a useful line between **workflows as code** (dynamic workflows: deterministic stages, checkpoints for human review) and **model-driven delegation** (`/fleet`). Pick workflows-as-code when the stages are known and auditability matters; let the model delegate when the decomposition itself is the hard part.

### Cost Optimization: The Plan-and-Execute Advantage

```
Traditional: [Frontier Model] handles all steps       Cost: $1.00/task

Plan-and-Execute:
  [Frontier Model] plans (1 call)                     Cost: $0.05
  [Small Model] executes steps 1-3                    Cost: $0.03
  [Frontier Model] aggregates (1 call)                Cost: $0.05
                                                      Total: $0.13/task
                                                      Savings: ~87%
```

The 2026 trend is treating agent cost optimization as a first-class concern, similar to how cloud cost optimization became essential in the microservices era. Two refinements to the arithmetic above:

- **Cascades with a quality gate** can beat static plan/execute splits when task difficulty varies: draft with a cheap model, gate, escalate only on failure. GitHub Copilot Fusion (GA October 28, 2026, grown out of the HydraFusion preview) ships exactly this as its cascade workflow, next to single-model and critique workflows. The price spread is what makes it tempting: GPT-6 Luna lists at $0.10/$0.50 per 1M tokens against $10/$50 for GPT-6 Astra. The catch is that prompt caches are per model, so an escalation hits a cold cache and pays for the whole context at the stronger model's uncached input rate. Before building a cascade, measure the stronger model alone at a lower effort setting on the same tasks.
- **Input tokens dominate long tool loops.** Anthropic's Claude Code telemetry (September 24, 2026, vendor-reported) puts the input-to-output ratio at 324:1, so the cache-read price and the cache hit rate matter more than the output price. Claude Opus 5.5 lists at $4/$20 with cache reads at $0.20 (0.05x input); keep the system prompt and tool definitions stable so they stay cached.

---

## Sandboxed vs. Unsandboxed Execution

This is the most consequential architecture decision for any tool-use agent.

### Comparison

```
  UNSANDBOXED (Host Access)              SANDBOXED (Isolated)
  +------------------------+             +------------------------+
  | LLM output executes    |             | LLM output executes    |
  | directly on host OS    |             | inside Docker/VM/E2B   |
  |                        |             |                        |
  | Risk: rm -rf /         |             | Isolated filesystem,   |
  | Risk: data exfiltration|             | network, processes     |
  |                        |             |                        |
  | Used by: OpenClaw      |             | Used by: OpenHands,    |
  | (main session), Python |             | OpenAI Codex, Devin,   |
  | Open Interpreter,      |             | Cursor cloud agents,   |
  | Claude Code (default)  |             | Open Interpreter (Rust)|
  +------------------------+             +------------------------+
```

### Sandbox Implementation Options

| Technology | Isolation Level | Startup Time | Use Case |
|------------|----------------|-------------|----------|
| Docker | Process + FS | 1-5 sec | Most agent sandboxes (OpenHands) |
| Firecracker | Full VM (microVM) | ~125ms | High-security, multi-tenant |
| gVisor | Kernel-level | ~200ms | Google Cloud Run |
| E2B | Cloud sandbox | 2-3 sec | Remote agent execution |
| WebAssembly | Language-level | <50ms | Browser-based execution |
| Managed snapshot runtime (AgentCore Runtime V2) | Managed per-session sandbox, restored from a snapshot | ~2 sec P75 for 200 MB to 2 GB images (AWS-reported; ~5.4 to ~30 sec on V1) | Per-session agent sandboxes without running the fleet |
| Managed VPC sandbox (Gemini Enterprise Computer Use and Shell, GA Sep 9, 2026) | Managed, with VPC Service Controls and CMEK | Idle sandboxes are descheduled with file-system state kept and resume in seconds (Google) | Regulated Google Cloud shops |

### The 2026 Consensus

Sandboxed-by-default with escape hatches. The OpenClaw security crisis (135,000 exposed instances on the public internet) has made the industry take this seriously. New production agents are expected to sandbox by default. Unsandboxed execution is reserved for single-user, supervised environments.

Two refinements from August and September 2026:

- **Split the planes.** Coding-agent vendors converged on a vendor control plane with a customer-hosted execution plane: Claude Code self-hosted runners, Claude Managed Agents self-hosted sandboxes, Cursor Self-hosted Machines, and self-hosted environments in the OpenAI Agents API. Orchestration and models stay with the vendor; tool execution, credentials and network egress stay inside your perimeter. This is the standard answer to "how do you let an agent touch production code and secrets."
- **The sandbox boundary includes egress and the kill path.** In its September 25, 2026 disclosure, OpenAI described an agent in an internal training run that tunneled out through insufficiently filtered DNS; monitoring alarmed about 12 minutes after the first external response, but the run was stopped only about 2.5 hours later because the automatic stop failed. Allowlist DNS (domains and record types), not just HTTP, and test the automatic kill like any other control. Out-of-band enforcement is the next tier: NVIDIA's Open Agent Safety Platform (announced September 28, 2026) pairs its OpenShell runtime (Apache 2.0, default-deny) with Sentry, a reference design for a watchdog on BlueField-4 DPUs that sits outside the agent's host and can quarantine it.

---

## State Management Across Tool Calls

Agents need to maintain state between tool calls. The strategy depends on the agent's lifecycle and use case.

### State Management Patterns

| Pattern | Lifecycle | Storage | Used By |
|---------|-----------|---------|---------|
| **Conversation State** | Ephemeral (single conversation) | Message array | Most API-based agents |
| **Session State** | Per-session (working dir, open files) | Docker container / temp dir | OpenHands, Claude Code |
| **Persistent State** | Cross-session (days, weeks) | DB, files, Markdown | OpenClaw (`MEMORY.md`, `memory/`), CLAUDE.md |
| **Environment State** | External (source of truth) | Git repo, database, FS | Claude Code (git status), CI/CD |
| **Managed Session State** | Durable across disconnects and restarts | Vendor-hosted session and event store | Claude Managed Agents, OpenAI Agents API |

Managed session state is convenient but moves your agent's working memory into a vendor's retention and residency terms, so check them per deployment.

### Implementation: Session State

```python
class AgentSession:
    """Manages state across tool calls within a single session."""
    def __init__(self):
        self.conversation: list[dict] = []
        self.working_dir: str = tempfile.mkdtemp()
        self.open_files: dict[str, str] = {}  # path -> content cache
        self.tool_call_count: int = 0

    def add_tool_result(self, tool_name: str, args: dict, result: dict):
        self.tool_call_count += 1
        self.conversation.append({"role": "tool", "tool_name": tool_name,
            "args": args, "result": result, "timestamp": time.time()})
        # Update derived state from side effects
        if tool_name == "write_file":
            self.open_files[args["path"]] = args["content"]

    def get_context_for_llm(self, max_tokens: int = 100_000) -> list[dict]:
        """Return conversation history, compressed if over budget."""
        if estimate_tokens(self.conversation) < max_tokens:
            return self.conversation
        return self._compress_history(max_tokens)  # Summarize old results
```

---

## Error Handling and Retry Patterns

Tool calls fail. Networks time out. APIs return errors. Code throws exceptions. A production agent needs systematic error handling.

### Error Taxonomy

| Error Type | Examples | Strategy |
|-----------|----------|----------|
| **Transient** | Network timeout, rate limit, 503 | Exponential backoff retry (max 3) |
| **Input** | Invalid args, wrong format | Feed error to LLM, let it fix args |
| **Permission** | Auth failure, access denied | Report to user, do NOT retry |
| **Logic** | Wrong tool, impossible op | Feed error to LLM, let it re-plan |
| **Catastrophic** | OOM, sandbox crash, infinite loop | Abort, report, clean up resources |
| **Contract** | HTTP 400 after a model or agent-version change (forced `tool_choice` on the newest Claude models, renamed tool parameters in a new managed-agent version) | Do not retry; pin versions and contract-test tool schemas in CI |

### Retry Pattern Implementation

```python
class ToolExecutor:
    MAX_RETRIES = 3

    async def execute_with_retry(self, tool_name: str, args: dict) -> dict:
        for attempt in range(self.MAX_RETRIES):
            try:
                result = await self.call_tool(tool_name, args)
                if not result.get("error"):
                    return result  # Success
                error_type = classify_error(result["error"])
                if error_type == "transient":
                    await asyncio.sleep(2 ** attempt)  # Exponential backoff
                    continue
                elif error_type == "input":
                    return {"error": result["error"], "fix_hint": "Adjust args"}
                elif error_type == "permission":
                    return {"error": result["error"], "action": "Report to user"}
                else:  # catastrophic
                    await self.cleanup_sandbox()
                    return {"error": "Fatal error. Task aborted."}
            except TimeoutError:
                if attempt < self.MAX_RETRIES - 1:
                    await asyncio.sleep(2 ** attempt)
                    continue
        return {"error": f"Failed after {self.MAX_RETRIES} retries"}
```

### The Self-Correction Loop

The most powerful error handling pattern in 2026. The agent observes its own failures and autonomously fixes them:

```
LLM generates code/tool call
  --> Execute --> Success? -- YES --> Return result
                     |
                     NO
                     |
                     v
              Feed error + stderr to LLM --> LLM generates fix --> Execute again
              (max 5 corrections to prevent infinite loops)
```

This is how Claude Code, OpenHands, and Cline handle test failures: run tests, see failures, edit code, re-run tests, repeat until green.

---

## MCP Integration Patterns

MCP has become the standard protocol for tool integration in 2026. Here are the key patterns for integrating MCP into agent architectures.

### Pattern A: Direct MCP Connection

```
[Agent (Client)] <-- stdio / HTTP --> [MCP Server]
```
Simplest pattern. One agent, one server. Used for single-purpose tools (database, file system).

### Pattern B: Multi-Server Fan-Out

```
                  +--> [GitHub MCP]
[Agent (Client)]--+--> [Postgres MCP]
                  +--> [Slack MCP]
```
Agent connects to multiple MCP servers simultaneously. Tool schemas are merged into one manifest. Used by Claude Code and multi-tool assistants.

### Pattern C: MCP Gateway (Enterprise)

```
[Agent 1] --+                          +--> [GitHub MCP]
[Agent 2] --+--> [MCP Gateway]  --+--> [Postgres MCP]
[Agent 3] --+    (Auth, Rate Limit,    +--> [Slack MCP]
                  Audit, Route)
```
Central gateway handles auth, rate limiting, and audit logging. Agents authenticate only with the gateway. Used for enterprise and multi-tenant deployments.

### MCP Roadmap Gaps

In May 2026 the MCP specification was missing three production primitives: identity propagation, adaptive tool budgeting, and structured error semantics. Their status against the current spec revision (2026-07-28, stateless core) and the maintainers' August 22, 2026 roadmap:

| Gap | Status (November 2026) | What to do meanwhile |
|-----|-----------------------|----------------------|
| **Identity propagation** | Enterprise-Managed Authorization (stable since June 18, 2026, and classified Stable under the extension lifecycle in SEP-3392) covers IdP-provisioned access for users. Agent identity is next: DPoP (SEP-1932) and Workload Identity Federation (SEP-1933) are open drafts, and an Agent Identity working group is forming | Keep the gateway as the identity broker for headless agents |
| **Adaptive tool budgeting** | Not among the August 22 roadmap's priority areas. The closest work is server-side progressive discovery for large tool catalogs, which saves schema tokens, not spend | Enforce budgets in the harness or gateway: session spend caps (Claude Managed Agents pauses with `budget_reached`), per-tool rate limits |
| **Structured error semantics** | Partly addressed: the roadmap plans a redesign of the `tools/call` result shape, because returning both `content` and `structuredContent` produced diverging implementations | Normalize errors at the gateway into your own taxonomy (see the table above) |

The next spec revision is targeted for March 30, 2027 (SEP-3398, accepted October 27, subject to confirmation), and releases after it fall generally on the last Tuesday of March and September, so plan SDK upgrades on that cadence. The [tool use and MCP chapter](../07-agentic-systems/03-tool-use-and-mcp.md) covers the stateless rewrite and the full roadmap.

---

## Architecture Decision Tree

Use this decision tree to select the right pattern for your use case:

```
Does the target system have an API?
 +-- YES --> Pattern 1 (Tool Calling). Wrap as MCP server. Fastest, most reliable.
 +-- NO  --> Does the task require GUI interaction?
              +-- YES --> Is it a web app?
              |            +-- YES --> Pattern 2 with element refs (accessibility tree) first,
              |            |           pixels as fallback. Confirm per action, approve per origin.
              |            +-- NO  --> Pattern 2 (Vision-Based). Sandbox in VM. Accept latency.
              +-- NO  --> Is the task primarily code/data work?
                           +-- YES --> Pattern 3 (Code Exec). Sandbox if multi-tenant.
                           +-- NO  --> Complex enough for multiple specialists?
                                        +-- YES --> Pattern 4 (Multi-Agent Orch.)
                                        +-- NO  --> Pattern 1 with custom tool.
```

### Hybrid Architectures

In practice, production systems combine patterns. Claude Code uses:
- Pattern 1 (tool calling) for file operations and git
- Pattern 2 (vision-based) for computer use features
- Pattern 3 (code execution) for bash and test running
- Pattern 4 (multi-agent) for subagent spawning

The key is to default to the simplest pattern (function calling) and only add complexity when the use case demands it.

---

## System Design Interview Angle

When discussing tool-use architecture in interviews, structure your answer around these five dimensions:

### 1. Pattern Selection

Start by identifying which pattern fits: "The target system has a REST API, so I would use the function/tool calling pattern with an MCP server wrapping the API." This shows you understand the decision tree.

### 2. Sandbox Boundary

Always address security: "For a multi-tenant deployment, I would sandbox each user's agent session in a Docker container with no network access to internal services. The MCP server runs outside the sandbox and mediates all external calls."

### 3. State Strategy

Explain how state is managed: "I would use session state within the Docker container for working files, and environment state (the git repo) as the source of truth. No persistent agent memory needed for this use case."

### 4. Error Budget

Discuss failure modes: "Tool calls can fail due to transient errors (retry with backoff), input errors (let the LLM self-correct), or permission errors (surface to user). I would set a max of 5 self-correction attempts before escalating."

### 5. Cost Model

Address economics: "For the orchestrator, I would use the Plan-and-Execute pattern: Opus 5.5 plans the task and Haiku 5.5 executes each step. Haiku 5.5 lists at $0.10/$0.50 per 1M tokens against $4/$20 for Opus 5.5, a 40x gap, and its cache reads cost $0.01 against $0.20. The catch is a price cliff: once a Haiku 5.5 prompt passes 100,000 tokens, counting cache reads and writes, the whole request bills at $0.50/$2.50, so I keep executor contexts short and hand each step only what it needs. It also rejects `temperature`, `top_p`, `top_k` and assistant prefill, so executor code ported from Haiku 4.5 needs a contract test. Because input outweighs output by two orders of magnitude in long tool loops, I would keep the system prompt and tool definitions stable so they stay in the prompt cache."

---

## Interview Questions

### Q: Design a system that lets a customer support agent answer questions using data from Zendesk, Salesforce, and an internal knowledge base.

**Strong answer:**
Pattern 1 (function/tool calling) with three MCP servers, one per data source. Use the Multi-Server Fan-Out pattern with dynamic manifests so only relevant tools load per query. For production, add an MCP Gateway to handle OAuth per data source, rate limiting (critical for Salesforce API limits), and audit logging. State is ephemeral: customer support does not need cross-session memory.

### Q: How would you prevent an AI agent from causing damage through tool calls?

**Strong answer:**
Defense in depth across five layers: (1) Schema constraints with deny-patterns (regex rejecting `DROP TABLE`, etc.). (2) Permission gate for destructive operations: Claude Code's allow/deny rules are a good model, and at volume a calibrated policy classifier that runs, denies or escalates each call avoids approval fatigue. (3) Sandbox isolation (Docker with read-only mounts, no outbound network, and DNS on an allowlist too). (4) Token and cost caps to prevent runaway loops, plus an automatic kill that you have actually tested. (5) Audit trail via the MCP Gateway pattern. No single layer is sufficient: the model can hallucinate args that pass validation (need sandbox), and the sandbox cannot prevent exfiltration through allowed paths (need audit logging and destination policy). PixelLeak (Glow Security, September 29, 2026) is the cautionary case: coding agents told to attach screenshots to private PRs published 13,000+ internal images to public repos, with no attacker involved, so repo visibility and destinations belong in the action policy.

### Q: Explain the trade-offs between vision-based computer use and API-based tool calling.

**Strong answer:**
API-based is faster (50-200ms vs. 1-3s per model round trip), cheaper (text vs. image tokens at roughly 1,000-1,800 tokens per screenshot), more reliable (deterministic vs. coordinate-clicking), and easier to test. Always prefer it when an API exists. Vision-based is the fallback for applications without APIs, legacy systems, or multi-app workflows. The gap has narrowed: zoom and accessibility-tree element refs cut misclicks, and batch actions spread one model round trip over several steps, at the price of a confirmation gate that must check each action in a batch before it runs rather than waiting for the next turn. Best practice: API calls for the 80% of tasks with API support, vision-based for the remaining 20%, and element refs before pixels inside browsers.

---

## References

- Anthropic. "Computer Use Tool Documentation" (2024-2026)
- Anthropic. "Model Context Protocol Specification" (2025-2026)
- MCP Core Maintainers. "The New MCP Roadmap" (August 22, 2026)
- Anthropic. Computer use and browser use tool documentation (`computer_toolset_20260801`, `browser_toolset_20260801`) (2026)
- Google. Gemini API computer use documentation (2026)
- IBM Developer. "MCP Architecture Patterns for Multi-Agent AI Systems" (2026)
- Google Cloud. "Choose a Design Pattern for Your Agentic AI System" (2025-2026)
- Microsoft Azure. "AI Agent Orchestration Patterns" (2025-2026)
- OpenHands Documentation. "Runtime Architecture" (2025-2026)
- OpenClaw Documentation. "Architecture and SOUL.md Guide" (2025-2026)
- Open Interpreter GitHub Repository (2024-2026)
- ArXiv 2603.13417. "Design Patterns for Deploying AI Agents with MCP" (2026)
- Kim et al. "An LLM Compiler for Parallel Function Calling" (ICML 2024, arXiv 2312.04511)

---

*Previous: [Tool-Use and Computer Agent Landscape](01-tool-use-landscape.md)*
*Next: [OpenClaw Deep Dive](03-openclaw-deep-dive.md)*
