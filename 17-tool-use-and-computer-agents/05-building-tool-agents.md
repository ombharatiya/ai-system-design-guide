# Building Tool-Use Agents

This chapter covers the practical engineering of tool-use agents: designing tool schemas that LLMs can call reliably, building MCP servers to host those tools, composing tools into workflows, and testing the entire system. These are the patterns that separate a demo from a production deployment.

## Table of Contents

- [Designing Tool Schemas for LLMs](#designing-tool-schemas-for-llms)
- [MCP Server Creation](#mcp-server-creation)
- [Tool Registration and Discovery](#tool-registration-and-discovery)
- [Input Validation and Output Formatting](#input-validation-and-output-formatting)
- [Tool Composition: Chaining Tools](#tool-composition-chaining-tools)
- [Building Custom Agent Skills](#building-custom-agent-skills)
- [Creating Function-Calling Endpoints](#creating-function-calling-endpoints)
- [Testing Tool-Use Agents](#testing-tool-use-agents)
- [Observability for Tool Use](#observability-for-tool-use)
- [Common Mistakes and Anti-Patterns](#common-mistakes-and-anti-patterns)
- [Tool Versioning and Backwards Compatibility](#tool-versioning-and-backwards-compatibility)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Designing Tool Schemas for LLMs

The tool schema is the contract between the LLM and your system. A well-designed schema reduces hallucinated arguments, prevents misuse, and makes the model's tool selection more reliable.

### Anatomy of a Good Tool Definition

```json
{
  "name": "search_customers",
  "description": "Search for customers by name, email, or account ID. Returns up to 10 matching customer records. Use this when the user asks about a specific customer. Do NOT use this for aggregate queries like 'how many customers do we have'.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search term: customer name, email address, or account ID (e.g., 'john@acme.com' or 'ACC-12345')"
      },
      "limit": {
        "type": "integer",
        "description": "Max results to return (1-10). Default: 5",
        "default": 5,
        "minimum": 1,
        "maximum": 10
      }
    },
    "required": ["query"]
  }
}
```

### Schema Design Rules

**1. Name precisely**: Use `verb_noun` format. `search_customers` not `search` or `customer_tool`.

**2. Describe when NOT to use**: The model needs negative examples. "Do NOT use for aggregate queries" prevents misuse better than only listing valid uses.

**3. Give argument examples**: Include example values in the description string. The model uses these to calibrate its outputs.

**4. Constrain ranges**: Use `minimum`, `maximum`, `enum`, and `pattern` to document and constrain valid arguments. Keep the same checks in your handler: strict mode does not enforce every keyword (see rule 6).

**5. Keep tools atomic**: One tool does one thing. Avoid a `manage_customer` tool that creates, reads, updates, and deletes; split it into four tools.

**6. Use `strict: true`**: Anthropic's strict mode guarantees the tool input validates against the schema. Enable it in production, with `additionalProperties: false` and a `required` list on every object. It does not cover numeric constraints (`minimum`, `maximum`, `multipleOf`), string lengths, or recursive schemas; the Python and TypeScript SDK schema helpers strip those constraints before sending and check them client-side, which is one more reason for handler-side validation.

**7. Do not force tool calls to get JSON**: On Claude Fable 5.1, Mythos 5.1, Opus 5.5, and Sonnet 5.5, `tool_choice` of type `any` or `tool` returns HTTP 400 (`auto` and `none` still work). Use `auto` plus a prompt instruction that names the tool and `strict: true` for schema-valid arguments, and check that a call was actually made. If the forced call only existed to extract structured data, use structured outputs (`output_config.format`) instead. Forcing a tool call is now a legacy technique, and frameworks have followed: langchain-anthropic now routes `with_structured_output` to JSON-schema output on these models (1.7.3 for Fable and Opus 5.5, 1.7.5 for Sonnet 5.5).

```
Good Tool Design:                    Bad Tool Design:

+-------------------+                +-------------------+
| search_customers  |                | customer_tool     |
| - query (string)  |                | - action (string) |
| - limit (int 1-10)|                | - data (object)   |
+-------------------+                | - options (any)   |
| create_customer   |                +-------------------+
| - name (string)   |                "action" can be
| - email (string)  |                "search", "create",
+-------------------+                "update", "delete"
| update_customer   |                => model confused,
| - id (string)     |                   schema too loose,
| - fields (object) |                   hard to validate
+-------------------+
```

---

## MCP Server Creation

An MCP server is a standalone process that exposes tools, resources, and prompts to any MCP-compatible client (Claude, GPT, Llama-based agents). You write the server once and any LLM can use it.

The current spec revision is **2026-07-28**, which made the core protocol stateless: no initialize handshake and no protocol-level session IDs, so a remote server can sit behind an ordinary load balancer. The next revision is targeted for March 30, 2027 (SEP-3398, subject to confirmation), after which releases follow a March and September cadence. Clients on the earlier 2025-11-25 stateful revision are still common, so plan for a long dual-version period (see [Tool Use and MCP](../07-agentic-systems/03-tool-use-and-mcp.md) for the migration details).

### MCP Architecture

```
+------------------+          JSON-RPC           +------------------+
|                  |  ========================>  |                  |
|   MCP Client     |                             |   MCP Server     |
|   (AI App)       |  <========================  |   (Your Code)    |
|                  |                             |                  |
|  - Claude Code   |  Transport:                 |  Exposes:        |
|  - Custom Agent  |  - stdio (local)            |  - Tools         |
|  - IDE Plugin    |  - Streamable HTTP (remote) |  - Resources     |
|                  |                             |  - Prompts       |
+------------------+                             +------------------+
```

### TypeScript MCP Server (SDK v2)

```typescript
import { McpServer } from "@modelcontextprotocol/server";
import { StdioServerTransport } from "@modelcontextprotocol/server/stdio";
import * as z from "zod/v4";

const server = new McpServer({ name: "customer-service", version: "1.0.0" });

server.registerTool(
  "search_customers",
  {
    description: "Search customers by name, email, or ID. Returns up to 10 matches.",
    inputSchema: z.object({
      query: z.string().describe("Search term: name, email, or account ID"),
      limit: z.number().min(1).max(10).default(5).describe("Max results"),
    }),
  },
  async ({ query, limit }) => ({
    content: [{ type: "text",
      text: JSON.stringify(await db.customers.search(query, limit), null, 2) }],
  })
);

await server.connect(new StdioServerTransport());
```

The v2 TypeScript SDK splits into `@modelcontextprotocol/server`, `@modelcontextprotocol/client`, and `@modelcontextprotocol/core`. The v1 package (`@modelcontextprotocol/sdk`) is in maintenance (security and critical fixes only) but still dominates installs (about 246M npm downloads in October 2026 against about 62M for v2 `core`, 49M for `server`, and 33M for `client`), so expect to read both styles.

### Python MCP Server (SDK v2, MCPServer)

```python
import json
from mcp.server import MCPServer  # v2 renamed FastMCP to MCPServer; pin mcp<2 to stay on v1

mcp = MCPServer("customer-service")

@mcp.tool()
async def search_customers(query: str, limit: int = 5) -> str:
    """Search customers by name, email, or ID. Returns up to 10 matches.
    Args:
        query: Search term - customer name, email, or account ID
        limit: Max results to return (1-10, default 5)
    """
    return json.dumps(await db.customers.search(query, limit), indent=2)

if __name__ == "__main__":
    mcp.run()  # stdio by default; mcp.run(transport="streamable-http") for remote
```

Since MCP Python SDK 2.0.0 (July 28, 2026), `pip install mcp` installs the 2.x line, and tutorials that import `mcp.server.fastmcp` break on a fresh install. Both SDKs follow the same pattern: create a server, register tools with typed schemas, connect a transport. The TypeScript SDK uses Zod for validation; Python uses type hints and docstrings.

### Deployment Modes

| Mode | Transport | Use Case |
|------|-----------|----------|
| Local (stdio) | stdin/stdout pipe | Desktop tools, IDE plugins |
| Remote (Streamable HTTP) | HTTP POST to one endpoint; responses as JSON or a stream | Cloud services, shared servers |
| Hybrid | Both | Develop local, deploy remote |

A remote server is a network service and gets attacked like one. Between August 15 and October 1, 2026, the GitHub Advisory Database published 73 advisories with MCP in the title (9 critical), led by 25 for the popular `mcp-atlassian` server, whose HTTP token verifier accepted any non-empty string (CVE-2026-77244, fixed in 0.22.0). Verify tokens against the issuer, not for presence; bind local HTTP servers to loopback with DNS-rebinding protection on; and treat upload or attachment tools as exfiltration channels.

The official SDKs are part of that attack surface. In late September 2026 both published advisories for an OAuth mix-up (CVE-2026-104850 in TypeScript, GHSA-qx49-fqc8-xw99 in Python, CVSS 7.5) in which a malicious MCP server could name its own authorization server and receive the client's stored refresh tokens and client secrets. The floors for that fix are TypeScript 1.31.0 or 2.2.0 and Python 1.30.0 or 2.2.0, and upgrading is not the whole fix: machine-to-machine credential providers also need an expected issuer configured, and credentials saved before the upgrade stay exposed until they are tagged or cleared. The design rule for any multi-server client: bind every credential to its issuer, and never let a server decide where secrets go.

October added two TypeScript advisories where upgrading is again not enough. In CVE-2026-111770 (CVSS 7.7), a server or transport shared across clients leaked one client's responses to another. It affects `@modelcontextprotocol/server` 2.0.0 through 2.2.x and is fixed in 2.3.0, but the fix requires code changes: create one server and transport per request or session. A module-level singleton is the convenient pattern and the vulnerable one (a stdio server like the example above serves a single client, so it is not exposed). CVE-2026-111771, accepting tokens issued for other services, is fixed in `sdk` 1.32.0 and `server` 2.3.0, but the audience check is opt-in (`expectedResource`), so set it. The lesson for any SDK: read the advisory, not just the version bump, because secure behavior is often a configuration you have to turn on.

Recent releases also tightened server defaults that a production deployment should keep rather than override: TypeScript 2.1.0 and later and current Python releases cap request bodies at 4 MiB, and Python 1.30.0 and 2.2.0 close legacy stateful sessions after 30 idle minutes, with at most 10,000 per server.

---

## Tool Registration and Discovery

In production, agents need to discover available tools dynamically rather than hardcoding them.

### Static Registration

Declare MCP servers in a config file (e.g., `claude_desktop_config.json`). Each entry maps a server name to a command, args, and optional env vars. Simple but inflexible: every server loads on startup regardless of relevance.

### Dynamic Discovery (Tool Search)

Anthropic's Tool Search (2025) solves schema overload. Instead of loading 200 tool schemas into context (which degrades reasoning), you mark most tools `defer_loading: true` and give the model a search tool (`tool_search_tool_regex_20251119` or `tool_search_tool_bm25_20251119`); it searches and receives only the 3-5 relevant schemas. Discovered schemas are appended to the request rather than swapped in, so the prompt cache survives. At least one tool, the search tool itself, must stay non-deferred.

### Changing Tools Mid-Session Without Losing the Cache

Editing the top-level `tools` array changes the front of the prompt and invalidates the whole cache, which is expensive for long agent sessions. Anthropic's `inline-tools-2026-09-15` beta (September 22, 2026) lets an application add a tool, change its schema, or swap in an MCP toolset inside a mid-conversation system message without invalidating the cache. With the MCP connector, the response records each server's fetched tool list in an `mcp_tool_listing` block, and sending that block back pins the list, which also blocks a server from silently changing a tool definition mid-session.

### MCP Discovery Protocol

MCP clients discover capabilities via standard JSON-RPC methods: `tools/list` returns available tools, `resources/list` returns data resources, `prompts/list` returns prompt templates, and the 2026-07-28 revision added `server/discover`, which every server must implement to advertise its supported protocol versions, capabilities, and identity (clients may call it first to pick a version). Before connecting at all, clients can find servers through Server Cards (SEP-2127, Final since October 6, 2026; the extension is at the Beta maturity stage), which are listed in AI Catalog entries at `/.well-known/ai-catalog.json`, or through the MCP Registry (still a preview). Server-side progressive discovery for very large catalogs is on the August 2026 MCP roadmap but not yet in the spec.

---

## Input Validation and Output Formatting

### Input Validation Layers

```
+---------------------+
|  Schema Validation   |  <-- JSON Schema / Zod / Pydantic
|  (type, range, enum) |      Catches: wrong types, out-of-range
+----------+----------+
           |
           v
+---------------------+
|  Business Validation |  <-- Your handler code
|  (exists, permitted) |      Catches: invalid IDs, unauthorized
+----------+----------+
           |
           v
+---------------------+
|  Execution           |  <-- Actual operation
+---------------------+
```

Always validate at both layers. Schema validation catches malformed input. Business validation catches semantically invalid input.

Decide which errors the model may see. Since MCP Python SDK 2.1.0, an unexpected exception in a handler is logged on the server and the client sees only `Error executing tool <name>`. That is the right default for stack traces and connection strings, but it hides the actionable messages the model needs to recover. Return them explicitly, as the handler below does, or raise `ToolError` (from `mcp.server.mcpserver.exceptions`) for a message the model should read.

```python
@mcp.tool()
async def transfer_funds(
    from_account: str,
    to_account: str,
    amount: float
) -> str:
    """Transfer funds between accounts."""
    # Schema already enforced types via type hints

    # Business validation
    if amount <= 0:
        return "Error: Amount must be positive."
    if amount > 10000:
        return "Error: Transfers over $10,000 require manual approval."
    if from_account == to_account:
        return "Error: Cannot transfer to the same account."

    from_acct = await db.accounts.get(from_account)
    if not from_acct:
        return f"Error: Account {from_account} not found."

    # Execute
    result = await db.transfers.execute(from_account, to_account, amount)
    return f"Transferred ${amount:.2f}. Confirmation: {result.id}"
```

### Output Formatting

Return structured data when the model needs to reason about it. Return human-readable text when the result is final.

```python
# Good: structured for further reasoning
return json.dumps({
    "customers": [
        {"id": "ACC-123", "name": "Jane Smith", "email": "jane@acme.com"},
        {"id": "ACC-456", "name": "John Doe", "email": "john@acme.com"}
    ],
    "total_matches": 2,
    "has_more": False
})

# Bad: unstructured blob
return "Found Jane Smith (ACC-123, jane@acme.com) and John Doe (ACC-456, john@acme.com)"
```

---

## Tool Composition: Chaining Tools

Real tasks require multiple tools called in sequence. There are two composition patterns:

### Pattern 1: LLM-Orchestrated Chaining

The LLM decides which tool to call next based on previous results:

```
User: "Find customer Jane Smith and create a high-priority ticket for her billing issue"

Turn 1:  LLM -> search_customers("Jane Smith")
         Result: {"id": "ACC-123", "name": "Jane Smith", ...}

Turn 2:  LLM -> create_ticket("ACC-123", "Billing issue", "...", "high")
         Result: "Ticket TK-789 created."

Turn 3:  LLM -> "I found Jane Smith (ACC-123) and created ticket TK-789."
```

Each tool call is a separate API round-trip. The model reasons about results between calls.

### Pattern 2: Programmatic Tool Calling

Anthropic's programmatic tool calling (2025) lets the model write code that chains tools without round-trips:

```
LLM generates code:
  customer = search_customers("Jane Smith")
  if customer.results:
    ticket = create_ticket(customer.results[0].id, ...)
    return f"Created {ticket.id} for {customer.results[0].name}"
  else:
    return "Customer not found"
```

This executes as a single API call, reducing latency from 3 round-trips to 1. On the Claude API you enable it by adding the code execution tool and listing it in each custom tool's `allowed_callers`; it does not combine with `strict: true`, forced `tool_choice`, or MCP tools, so pick it per tool. The same idea, often called code mode or CodeAct, spread across frameworks in 2026 (Agno CodeMode, Microsoft Agent Framework's Hyperlight CodeAct provider, DSPy 3.4's interpreter), with Microsoft reporting more than 60% lower token use in the workload it evaluated (vendor-reported).

### Pattern 3: Server-Side Composition

Compose tools inside the MCP server itself: a single `resolve_customer_issue` tool internally calls search and create_ticket, hiding the multi-step logic from the LLM. Use this for fixed, well-defined workflows where the LLM does not need to reason between steps.

### When to Use Each

| Pattern | Latency | Flexibility | Best For |
|---------|---------|-------------|----------|
| LLM-orchestrated | High (N round-trips) | Very high | Complex, branching logic |
| Programmatic | Low (1 round-trip) | High | Linear chains, batches |
| Server-side | Lowest | Low | Fixed, common workflows |

---

## Building Custom Agent Skills

Agent Skills (Anthropic, 2025; out of beta on the Claude API since August 19, 2026) are folders of instructions, scripts, and resources that an agent loads only when a task needs them. A skill is a folder:

```
my-skill/
  SKILL.md          # YAML frontmatter (name, description) + instructions
  scripts/          # Code the agent can run
  references/       # Docs it reads on demand
  assets/           # Templates, schemas, data files
  tests/            # Your evaluation cases (not part of the spec, but keep them)
```

Loading is progressive: only each skill's name and description sit in context at the start, the body of `SKILL.md` loads when the model decides the skill is relevant, and supporting files are read as needed. This keeps the base agent lightweight and the prompt prefix stable, which matters for caching; injecting every skill into the system prompt defeats both.

**Skills are now a supply chain.** Two developments in September 2026 changed how to ship them:
- **Skills over MCP:** the official MCP extension `io.modelcontextprotocol/skills` (SEP-2640, Final on September 13) lets a server publish skills next to the tools they describe. Each skill carries a file manifest with a SHA-256 digest per file; hosts must verify digests before use, bind any approval to the whole manifest (a changed, added, or removed file revokes it), treat skill content as untrusted, and get explicit approval before a skill grants tools or runs host-side code. TypeScript SDK 2.4.0 (October 23) ships Skills over MCP. A Final SEP is not a frozen interface, though: under the extension lifecycle adopted in October (SEP-3392), Skills, Tasks, and Server Cards are Beta and can still break, while MCP Apps and enterprise auth are Stable, so pin the SDK version you build on.
- **Scanners are not enough:** Pretext (arXiv 2609.39607) crafted malicious skills that got past NVIDIA's SkillSpector, a static-plus-LLM skill scanner, and still delivered their payload in up to 97% of attempts (77% against a version of the scanner that learns from its misses). The trick was moving payloads from code into natural language and splitting instructions across files. Pin skill versions, review updates like dependency upgrades, and contain what a skill can do at runtime instead of trusting a pre-install scan.

---

## Creating Function-Calling Endpoints

To make your API callable by any LLM, expose it via FastAPI with Pydantic models. The auto-generated OpenAPI spec (`/openapi.json`) doubles as a tool schema for function calling. Alternatively, wrap the same logic in an MCP server for direct integration with Claude, GPT, or other MCP-compatible clients.

---

## Testing Tool-Use Agents

### Three Testing Layers

```
+---------------------------+
|   Eval Suites             |  End-to-end: does the agent
|   (Agent + LLM + Tools)  |  complete the task?
+-------------+-------------+
              |
+-------------v-------------+
|   Integration Tests       |  Does tool X work correctly
|   (Tool + Dependencies)   |  with real DB / API?
+-------------+-------------+
              |
+-------------v-------------+
|   Unit Tests              |  Does validation logic
|   (Tool Logic Only)       |  handle edge cases?
+---------------------------+
```

### Unit Tests for Tools

Test each tool handler in isolation with mocked dependencies. Cover: input validation edge cases (out-of-range values, missing fields), error message quality (does it guide the model to recover?), and output format (valid JSON, correct schema).

### Eval Suites for Agent Behavior

Build a dataset of 100+ realistic queries with expected outcomes:

```python
eval_cases = [
    {
        "input": "Find Jane Smith's account and check her last payment",
        "expected_tools": ["search_customers", "get_payment_history"],
        "max_tool_calls": 5,
    },
    {
        "input": "What is the meaning of life?",
        "expected_tools": [],  # Should NOT call any tools
        "max_tool_calls": 0,
    },
]
```

For each case, measure: tool selection accuracy (right tool?), argument quality (correct args?), task completion rate, and efficiency (number of tool calls). Run evals on every model version change and every tool schema change.

---

## Observability for Tool Use

Every tool call should log: trace/span IDs, timestamp, tool name, input args, output size, latency, status, model used, token usage, and session ID.

**Check that your tracing still sees the calls after an SDK upgrade.** The Anthropic Python SDK 1.0 (August 2026) and OpenAI Python SDK 3.0 (August 2026) moved to `httpx2`. Instrumentation and test doubles built on `httpx` (OpenTelemetry HTTP instrumentation, respx, vcrpy) can silently miss every model call unless you apply the shim Anthropic's upgrade guide describes (`httpx2.alias_httpx()`). An empty trace is not an error, so add a canary test that asserts spans exist.

### Key Metrics

| Metric | What It Measures | Alert Threshold |
|--------|-----------------|-----------------|
| Tool call success rate | % of calls returning valid results | < 95% |
| Tool selection accuracy | Was the right tool chosen? | < 90% |
| Avg tool calls per task | Efficiency of tool use | > 2x baseline |
| Latency per tool call | Response time of tool handlers | > 5s (p99) |
| Hallucinated arguments | Invalid args despite schema | > 2% |
| Cost per task | Total LLM + tool execution cost | > budget |

### Tracing Architecture

```
+-------------+     +----------------+     +--------------+
|  Agent      |---->|  Tool Handler  |---->|  Backend     |
|  (LLM call) |     |  (MCP Server)  |     |  (DB/API)    |
+------+------+     +--------+-------+     +------+-------+
       |                     |                     |
       v                     v                     v
+------+---------------------+---------------------+------+
|                    Trace Collector                       |
|              (OpenTelemetry / Langfuse)                  |
+---------------------------+------------------------------+
                            |
                            v
                   +--------+--------+
                   |   Dashboard     |
                   |   - Success %   |
                   |   - Latency     |
                   |   - Cost        |
                   +-----------------+
```

---

## Common Mistakes and Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Tool overload | 50+ tools degrades selection accuracy | Dynamic discovery, load 5-10 per turn |
| Vague descriptions | "Handles customer operations" is too vague | Include when to use, when NOT to use, examples |
| Forced tool calls for JSON | `tool_choice` `any`/`tool` returns 400 on the newest Claude models | `auto` + `strict: true`, or structured outputs |
| Leaking raw exceptions | Stack traces and connection strings reach the model and the logs it can echo | Return curated error messages; let the SDK hide unexpected exceptions |
| God tools | One tool with `action` param does everything | Split into atomic tools, one operation each |
| Missing error context | Tool returns "Error" with no details | Actionable messages: "ACC-999 not found. Use search_customers..." |
| Unstructured output | Tool returns prose the model must parse | Return JSON for structured reasoning |
| No idempotency | `create_ticket` called twice creates duplicates | Accept idempotency key, check before creating |
| Exposing internal IDs | Tool requires database UUIDs model cannot know | Accept human-readable identifiers, resolve internally |
| Ignoring rate limits | Agent loops 100 API calls, gets throttled | Backoff in handlers, return "retry in X seconds" |

---

## Tool Versioning and Backwards Compatibility

As tools evolve, you must maintain compatibility with agents that depend on them.

**Rules:**
1. **Additive changes** (new optional params): No version bump needed. Old calls still work.
2. **Breaking changes** (rename, remove param, change semantics): Create a new tool name with the new schema. Keep the old tool running and add "DEPRECATED: Use new_tool instead" to its description. Log every deprecated call for monitoring.
3. **Never remove a tool** until you verify no active agents depend on it.

**The same discipline applies to tools you consume.** Vendors change built-in tool schemas on short notice. Google's `antigravity-preview-09-2026` agent (September 17, 2026) renamed its built-in tool parameters from snake_case to PascalCase, replaced full-file rewrites with line-range edits, and added `find_by_name` and `grep_search`; the 05-2026 version it replaced was scheduled to shut down on October 5, 2026, 18 days later. Code that executes those tools locally or parses `function_call` steps had to change. Pin dated agent and tool versions, run contract tests against the tool definitions you receive, and alert when a fetched MCP tool list differs from the pinned one (a changed definition after approval is the "rug pull" attack). SDK majors belong in the same contract tests: Anthropic's eight new SDK and CLI majors (Python 2.0.0, TypeScript 1.0.0, C# 13.0.0, Go 2.0.0, Java 3.0.0, PHP 1.0.0, Ruby 2.0.0, and the `ant` CLI 2.0.0) send API version `2026-10-21` by default, under which `temperature`, `top_p`, and `top_k` return HTTP 400 on every model, even at default values, and the new majors are licensed Apache 2.0 rather than MIT.

---

## Interview Questions

### Q: You need to give an LLM agent access to 200 internal tools. How do you handle schema overload?

**Strong answer:**
I would not load all 200 tool schemas into the context. Instead, I would implement a two-phase approach. First, a tool discovery phase where the agent describes what it needs to do, and a lightweight search (embedding similarity or keyword match) returns the 5-10 most relevant tool schemas. Second, a tool execution phase where only the selected tools are included in the context for the actual LLM call.

This mirrors Anthropic's Tool Search pattern. The discovery step can be a separate, cheaper LLM call or even a non-LLM search. The key insight is that context window space used by irrelevant tool schemas directly reduces the model's reasoning quality. I would measure tool selection accuracy as a key metric: if the agent calls `search_customers` when it should call `get_customer_by_id`, the discovery phase needs tuning.

The second constraint is the prompt cache. Long agent sessions are input-dominated, so swapping tool schemas in and out of the top-level tools array, which invalidates the cache on every change, can cost more than the context it saves. I would use a mechanism that appends discovered schemas (deferred loading with a search tool, or mid-conversation tool definitions where the provider supports them) so the cached prefix survives.

For the MCP implementation, I would group tools into domain-specific servers (customer-service, billing, analytics) and only connect to the servers relevant to the current conversation, and I would pin each server's fetched tool list so a definition cannot change mid-session.

### Q: Design a testing strategy for a tool-use agent that handles customer support.

**Strong answer:**
I would test at three layers. First, unit tests for each tool handler: validate input edge cases, error messages, and output format. These run in CI on every commit with mocked dependencies.

Second, integration tests that verify tools work against real (staging) databases. For example, `create_ticket` actually creates a record and `search_customers` returns it. These catch schema drift between the tool and the backend.

Third, eval suites that test the full agent: LLM plus tools. I would build a dataset of 100+ realistic customer queries with expected tool call sequences and output criteria. The eval measures tool selection accuracy (did it pick the right tool?), argument quality (were the arguments correct?), task completion rate (did it solve the problem?), and efficiency (how many tool calls did it take?).

I would run evals on every model version change and every tool schema change. A 2% drop in tool selection accuracy after a schema change means the description needs revision, not the model.

### Q: Your extraction service forces a tool call to get JSON back. After moving to the newest Claude model, every request returns HTTP 400. What happened, and how do you fix it without losing the schema guarantee?

**Strong answer:**
Claude Fable 5.1, Mythos 5.1, Opus 5.5, and Sonnet 5.5 reject `tool_choice` of type `any` or `tool`; only `auto` and `none` remain. Forcing a tool call was always a workaround for getting structured output, and the provider has now removed it on its largest models. Haiku 5.5 still accepts a forced `tool_choice`, but the response then starts with the tool call and has no thinking block.

The fix depends on why the call was forced. If the tool only existed to shape the output, I would delete it and use native structured outputs (`output_config.format` with a JSON schema), which constrains the response itself. If the tool does real work, I would switch to `auto`, name the tool in the prompt, set `strict: true` so any call that happens has schema-valid arguments, and add a check in the loop for the case where the model answered without calling it, with one retry and then an error.

Two follow-ups matter more than the fix. First, this broke at a model upgrade, so I would add a pre-upgrade contract test that runs every request shape we send against the new model ID before the switch, including `count_tokens` calls, which apply the same check. Second, I would keep range and length checks in the handler, because strict mode does not enforce `minimum`, `maximum`, or string lengths.

---

## References

- Anthropic. "Tool Use with Claude" API documentation, including strict tool use and structured outputs
- Anthropic. Claude Platform release notes (forced `tool_choice` removal; `inline-tools-2026-09-15` beta; August-September 2026)
- Model Context Protocol. Specification revision 2026-07-28 and "Build an MCP Server"
- Model Context Protocol. "The New MCP Roadmap" (August 22, 2026)
- Model Context Protocol. Skills extension, SEP-2640 (September 2026)
- MCP TypeScript SDK: github.com/modelcontextprotocol/typescript-sdk
- MCP Python SDK: github.com/modelcontextprotocol/python-sdk (v2 migration guide at py.sdk.modelcontextprotocol.io)
- Anthropic. "Introducing Advanced Tool Use" (2025)
- Anthropic. "Agent Skills" documentation (GA on the Claude API since August 2026)
- Kaisar and Dhar. "Pretext" (arXiv 2609.39607, September 2026)
- GitHub Advisory Database: reviewed MCP advisories (github.com/advisories?query=type%3Areviewed+mcp)
- MCP SDK security advisories GHSA-6qxp-vccf-f47h / CVE-2026-104850 (TypeScript) and GHSA-qx49-fqc8-xw99 (Python), OAuth authorization-server mix-up (September 2026)
- MCP TypeScript SDK advisories CVE-2026-111770 (shared server or transport) and CVE-2026-111771 (token audience) (October 2026)
- Model Context Protocol. SEP-3392 "Extension Lifecycle" and SEP-3398 "Release Cycle Updates" (October 2026)

---

*Previous: [Computer-Use Agents](04-computer-use-agents.md)*
