# Claude Code: The Autonomous Coding Agent

Claude Code is Anthropic's **terminal-native autonomous coding agent**. Unlike IDE plugins that suggest completions, Claude Code acts as a full-stack software engineer: it reads your codebase, edits files, runs commands, executes tests, and iterates until the task is done.

## Table of Contents

- [What Claude Code Is](#what-it-is)
- [Core Architecture](#architecture)
- [Core Tools](#tools)
- [The CLAUDE.md Manifest Pattern](#claude-md)
- [Running Claude Code](#running)
- [Sub-Agents and Parallelism](#subagents)
- [Custom MCP Integration](#mcp-integration)
- [Safety and Permission Model](#safety)
- [Production Use: CI Pipelines](#production)
- [Comparison: Claude Code vs Alternatives](#comparison)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## What Claude Code Is

Released by Anthropic in early 2025, Claude Code is:

- **A CLI tool**: `claude` command in your terminal
- **An MCP-native agent**: Uses bash, text_editor, and computer tools
- **An SDK**: Can be embedded in Python/TypeScript applications
- **Not just a chatbot**: It autonomously plans, implements, and verifies

```
# Install
pip install claude-code  # or: npm install -g @anthropic-ai/claude-code

# Run interactively
claude

# Run headlessly (for CI)
claude -p "Add unit tests for all functions in src/utils.py" --output-format json
```

**The key difference from Copilot/Cursor:**
- Copilot/Cursor: Suggests code you accept or reject
- Claude Code: **Autonomously implements the entire task**, running tests to verify

---

## Core Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   CLAUDE CODE ARCHITECTURE               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  User Request                                           │
│       ↓                                                 │
│  ┌─────────────┐    ┌──────────────┐                   │
│  │  Claude 3.7 │    │  CLAUDE.md   │                   │
│  │   Sonnet    │ ←  │  (manifest)  │                   │
│  │ (Extended   │    └──────────────┘                   │
│  │  Thinking)  │                                       │
│  └──────┬──────┘                                       │
│         │ Tool calls                                    │
│         ↓                                               │
│  ┌──────────────────────────────────────┐              │
│  │           TOOL LAYER                 │              │
│  │  ┌─────────┐ ┌───────────┐ ┌──────┐ │              │
│  │  │  bash   │ │text_editor│ │  MCP │ │              │
│  │  └────┬────┘ └─────┬─────┘ └──┬───┘ │              │
│  └───────┼────────────┼──────────┼─────┘              │
│          │            │          │                      │
│   Shell cmds     File edits    Custom tools             │
│   (test, lint,   (read/write)  (DB, APIs,               │
│    git, build)                  internal)               │
└─────────────────────────────────────────────────────────┘
```

Claude Code uses **Claude 3.7 Sonnet** as its backbone model, with Extended Thinking enabled by default for complex planning tasks.

---

## Core Tools

Claude Code has three native tools and supports custom MCP tools:

### 1. `bash` — Shell Execution

```python
# Claude calls this internally:
bash(command="pytest tests/ -v --tb=short", timeout=60)
# Returns: stdout, stderr, exit_code
```

**What Claude uses it for:**
- Running test suites (`pytest`, `jest`, `cargo test`)
- Git operations (`git diff`, `git commit`, `git log`)
- Build commands (`npm build`, `make`, `docker build`)
- Package installation (`pip install`, `npm install`)

The bash session is **persistent across turns** — environment variables and working directory carry over within a session.

### 2. `text_editor` — File Operations

```python
# Read a file
text_editor(command="view", path="/project/src/auth.py")

# Find in file
text_editor(command="view", path="/project/src/auth.py", view_range=[1, 50])

# Edit (surgical replacement)
text_editor(
    command="str_replace",
    path="/project/src/auth.py",
    old_str="def authenticate(user, password):",
    new_str="def authenticate(user: str, password: str) -> AuthResult:"
)

# Create new file
text_editor(command="create", path="/project/tests/test_auth.py", file_text="...")
```

**Why surgical replacement beats rewriting:**
- Preserves file context
- Reduces hallucination (only changes what needs changing)
- Enables atomic, reviewable diffs

### 3. `computer` — GUI Automation (optional)

Full desktop control (screenshots, mouse, keyboard) — used for browser testing and UI verification. Requires sandboxed environment.

---

## The CLAUDE.md Manifest Pattern

The `CLAUDE.md` file is the **single most important pattern** for using Claude Code productively. It injects persistent project context into every Claude Code session.

```markdown
# CLAUDE.md — Project: E-Commerce API

## Architecture
- Python 3.11 FastAPI backend
- PostgreSQL 15 with Alembic migrations
- Redis for session caching
- All API responses must be Pydantic models

## Test Commands
- Run all tests: `pytest tests/ -v`
- Run single test: `pytest tests/test_auth.py::test_login -v`
- Lint: `ruff check . --fix`
- Type check: `mypy src/`

## Coding Standards
- Always add type hints
- Never use `global` variables
- All database queries through SQLAlchemy ORM, never raw SQL
- New features require tests with >80% coverage

## Forbidden Patterns
- Do NOT use `os.system()` — use `subprocess.run()` instead
- Do NOT commit secrets — use environment variables
- Do NOT modify `alembic/versions/` — create new migrations

## Architecture Decisions
- Auth: JWT tokens, 1hr expiry, refresh token pattern
- Errors: Always return RFC 7807 Problem Details format
- Logging: structlog with JSON output, always include request_id
```

**Nesting CLAUDE.md files:**
```
project/
  CLAUDE.md          # global project rules
  src/
    auth/
      CLAUDE.md      # auth-specific rules (stricter security)
    payments/
      CLAUDE.md      # payment-specific rules (PCI compliance notes)
```

Claude automatically reads the closest CLAUDE.md when working in a directory.

---

## Running Claude Code

### Interactive Mode

```bash
# Start session (reads CLAUDE.md automatically)
claude

# With specific model
claude --model claude-3-7-sonnet-20250219

# With MCP config
claude --mcp-config .claude/mcp.json
```

### Headless Mode (for scripting)

```bash
# Single task, JSON output
claude -p "Fix all type errors in src/" \
  --output-format json \
  --max-turns 20

# Pipe from file
echo "Refactor src/utils.py to use async/await" | claude -p -

# Stream output
claude -p "Add logging to all API endpoints" --output-format stream-json
```

### Python SDK

```python
import asyncio
from claude_code_sdk import query, ClaudeCodeOptions

async def run_coding_task(task: str) -> str:
    options = ClaudeCodeOptions(
        max_turns=30,
        allowed_tools=["bash", "str_replace_based_edit_tool"],
        system_prompt_suffix="Always run tests after making changes.",
    )
    
    messages = []
    async for message in query(prompt=task, options=options):
        messages.append(message)
    
    return messages[-1].content[0].text

result = asyncio.run(run_coding_task(
    "Add input validation to all POST endpoints in src/api/"
))
```

---

## Sub-Agents and Parallelism

Claude Code supports **sub-agent dispatch** for large codebases:

```
Main Claude Code session
    ↓
"This codebase has 5 modules. I'll spawn sub-agents for each."
    ├── Sub-agent 1: Fix auth module tests
    ├── Sub-agent 2: Add type hints to utils/
    ├── Sub-agent 3: Migrate payments to async
    └── Sub-agent 4: Update API documentation
```

Each sub-agent runs in parallel, then the main agent reviews and merges the results.

**When to use sub-agents:**
- Codebase >50K lines of code
- Parallel independent changes (no shared state)
- Module-level refactoring tasks

---

## Custom MCP Integration

Claude Code reads MCP servers from `~/.claude/config.json` or `.claude/mcp.json`:

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"],
      "description": "Live library documentation"
    },
    "postgres": {
      "command": "uvx",
      "args": ["mcp-server-postgres"],
      "env": {"DATABASE_URL": "postgresql://localhost/myapp"},
      "description": "Direct DB access for schema inspection"
    },
    "jira": {
      "command": "uvx",
      "args": ["mcp-server-jira"],
      "env": {"JIRA_URL": "https://company.atlassian.net"},
      "description": "Task tracking integration"
    }
  }
}
```

With this config, Claude Code can:
1. Look up current library docs before writing code (Context7)
2. Read the actual DB schema before writing SQL (postgres MCP)
3. Mark Jira tickets as done after completing implementations (jira MCP)

---

## Safety and Permission Model

Claude Code has a **layered permission model**:

```
Permission Level    Who approves       What it covers
────────────────────────────────────────────────────────
Auto               Claude (no prompt)  Read files, run tests
Ask per-turn       User confirms       Shell command execution
Explicit allow     User pre-approves   Specific commands/dirs
Blocked            Never runs          Network calls outside allowlist
```

### Configuration

```json
{
  "permissions": {
    "allow": [
      "bash(pytest*)",           // Always allow test runs
      "bash(ruff*)",             // Always allow linting
      "bash(git diff*)",         // Always allow git reads
      "str_replace_based_edit_tool"  // Always allow file edits
    ],
    "deny": [
      "bash(rm -rf*)",           // Block destructive deletions
      "bash(curl https://external*)", // Block external network
      "bash(pip install*)"       // Block package installs without approval
    ]
  }
}
```

### Production Safety Rules

1. **Always sandbox**: Run in Docker container or E2B cloud VM
2. **Git isolation**: Create a feature branch before starting; review diff before merge
3. **Human checkpoint**: For prod deployments, require human review of the final diff
4. **Secret scanning**: Run `truffleHog` or `git-secrets` on every Claude Code output
5. **Rate limits**: Set `max_turns` to prevent runaway loops (recommended: 20-30)

---

## Production Use: CI Pipelines

### GitHub Actions Integration

```yaml
# .github/workflows/ai-fix.yml
name: AI Bug Fix
on:
  issues:
    types: [labeled]

jobs:
  ai-fix:
    if: github.event.label.name == 'ai-fix'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Claude Code
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          pip install claude-code
          
          ISSUE_BODY="${{ github.event.issue.body }}"
          
          claude -p "Fix the following bug: $ISSUE_BODY
          
          Rules:
          - Read the relevant files first
          - Make minimal changes
          - Run tests and verify they pass
          - Do not change unrelated code
          " --output-format json --max-turns 15 > result.json
      
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v5
        with:
          title: "AI Fix: ${{ github.event.issue.title }}"
          body: "Automated fix by Claude Code"
          branch: "ai-fix/${{ github.event.issue.number }}"
```

### Cost Model for CI

| Task Type | Avg Turns | Avg Tokens | Estimated Cost |
|-----------|-----------|------------|----------------|
| Bug fix (small) | 8 | 15K | $0.23 |
| Test generation | 12 | 25K | $0.38 |
| Feature implementation | 20 | 50K | $0.75 |
| Large refactor | 30 | 100K | $1.50 |

*At 100 CI runs/day: ~$75-150/day depending on task mix.*

---

## Comparison: Claude Code vs Alternatives

| Feature | Claude Code | Cursor/Windsurf | Cline | OpenHands |
|---------|-------------|-----------------|-------|-----------|
| **Interface** | CLI + SDK | IDE (VS Code fork) | VS Code extension | Web UI + CLI |
| **Model** | Claude only | Any (GPT, Claude, Gemini) | Any | Any |
| **Autonomy** | Full | Medium (requires clicks) | Full | Full |
| **CI/Headless** | ✅ Native | ❌ | ✅ | ✅ |
| **MCP support** | ✅ Native | ✅ | ✅ | ✅ |
| **CLAUDE.md** | ✅ | ❌ (similar: .cursorrules) | ❌ | ❌ |
| **Open source** | ❌ | ❌ | ✅ | ✅ |
| **Best for** | Backend devs, CI/CD | UI/frontend devs, visual | Any developer | Self-hosted teams |

### SWE-bench Verified Scores (May 2026)

| Agent | Score | Notes |
|-------|-------|-------|
| GPT-5.5 (raw model leader) | 88.7% | #1 on SWE-Bench Verified leaderboard |
| Claude Opus 4.7 (raw model) | 87.6% | Leads SWE-Bench Pro at 64.3% |
| Claude Code (Opus 4.7 / Sonnet 4.6) | ~87% | Anthropic's official agent |
| OpenHands + Claude Sonnet 4.6 | ~75% | Open-source framework |
| Aider + Claude Sonnet 4.6 / GPT-5.5 | ~74% | Open-source CLI |
| Devin (commercial) | ~65% | Cognition AI product |
| SWE-agent + GPT-5.5 | ~55% | Princeton research baseline |

---

## Interview Questions

### Q: How does Claude Code differ from GitHub Copilot?

**Strong answer:**
Copilot is a **completion tool** — it predicts the next few lines of code as you type. Claude Code is an **autonomous agent** — you give it a task (e.g., "add authentication to this API"), and it reads the codebase, plans the implementation, edits multiple files, runs tests, fixes failures, and only finishes when tests pass. The experience is fundamentally different: Copilot helps you code faster; Claude Code codes *for* you while you review the output.

### Q: What is CLAUDE.md and why is it critical?

**Strong answer:**
CLAUDE.md is like a `README` specifically written for an AI colleague. Without it, Claude Code treats your project as a generic Python/JS project. With it, Claude knows: your exact test command, your forbidden patterns (no raw SQL, use ORM), your architecture decisions (JWT auth, specific error format), and your coding standards. It converts a general-purpose agent into a **project-specialist**. I've seen 2-3x faster task completion and 60% fewer mistakes with a well-written CLAUDE.md.

### Q: How do you safely run Claude Code in production CI?

**Strong answer:**
Three layers:
1. **Sandbox**: Run Claude Code inside a Docker container with no external network access. Only the git repo and test runner are accessible.
2. **Permission allow-list**: Use the permissions config to whitelist exactly which bash commands are allowed (test runners, linters) and block destructive operations (rm -rf, pip install without review).
3. **Human gate**: Claude Code outputs a branch with a diff. A human reviews the diff in a PR and merges. Claude never merges directly to main. This keeps human judgment in the loop for the final decision.

### Q: How do you handle the cost of Claude Code for high-volume CI?

**Strong answer:**
I optimize in three ways:
1. **Task scoping**: Claude Code is cost-effective for independent, bounded tasks (bug fixes, test generation). I don't use it for open-ended exploration — that's still cheaper with a human.
2. **Max turns**: Setting `max_turns=15` prevents runaway jobs that burn $10+ on circular reasoning.
3. **Model routing**: For simple bug fixes (syntax errors, obvious typos), I use Claude 3.5 Haiku via the SDK — 5x cheaper. For architectural refactoring, I use Claude 3.7 Sonnet with Extended Thinking.

---

## References

- Anthropic. "Claude Code: Building Agentic Coding Experiences" (2025) — https://docs.anthropic.com/claude-code
- Anthropic. "Claude Code SDK Documentation" — https://github.com/anthropics/claude-code
- Anthropic. "CLAUDE.md Best Practices" — https://docs.anthropic.com/claude-code/settings#claudemd
- SWE-bench Verified Leaderboard — https://www.swebench.com/

---

*Next: [OpenCoder / AI Coding Agents Landscape](10-opencoderguide.md)*
