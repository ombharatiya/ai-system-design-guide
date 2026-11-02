# Computer-Use Agents

Computer-use agents let an LLM see a screen, reason about it, and act through mouse clicks and keystrokes, the same way a human operates a computer. Instead of calling structured APIs, the model works with raw pixels (and, for browsers, increasingly with the page's accessibility tree). This chapter covers how they work, when they beat traditional automation, and how to design production systems around them.

## Table of Contents

- [What Are Computer-Use Agents?](#what-are-computer-use-agents)
- [The Screenshot-Reason-Act Loop](#the-screenshot-reason-act-loop)
- [Claude Computer Use: Tools and API](#claude-computer-use-tools-and-api)
- [Architecture: Sandboxed Environments](#architecture-sandboxed-environments)
- [Browser vs Desktop Automation](#browser-vs-desktop-automation)
- [Comparison with Traditional Automation](#comparison-with-traditional-automation)
- [When Computer-Use Beats API Calls](#when-computer-use-beats-api-calls)
- [Error Handling and Recovery](#error-handling-and-recovery)
- [Performance: Latency, Cost, Throughput](#performance-latency-cost-throughput)
- [Benchmarks: Reading the Numbers](#benchmarks-reading-the-numbers)
- [Real-World Applications](#real-world-applications)
- [Security Considerations](#security-considerations)
- [Code Examples](#code-examples)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## What Are Computer-Use Agents?

A computer-use agent is an LLM that controls a graphical interface by interpreting screenshots and issuing low-level input commands (mouse moves, clicks, keystrokes). It replaces the human in the human-computer interaction loop.

```
Traditional Tool Use:           Computer Use:

User Request                    User Request
     |                               |
     v                               v
 LLM reasons                    LLM reasons
     |                               |
     v                               v
 Structured API call             Screenshot captured
 {"tool": "search",                  |
  "query": "..."}                    v
     |                          LLM sees pixels, finds button
     v                               |
 API returns JSON                    v
     |                          Mouse click at (x=340, y=220)
     v                               |
 LLM formats answer                  v
                                New screenshot captured
                                     |
                                     v
                                LLM verifies result, continues...
```

The key difference: traditional tool use requires pre-defined APIs with known schemas. Computer use works with any application that has a visual interface, no API required.

### The Landscape (2026)

Most major labs and platforms now ship computer use, and the product has split into three shapes: an API tool you host, a managed runtime the vendor hosts, and a persistent agent with its own cloud computer.

| Provider | Offering | Approach | Key Strength |
|----------|----------|----------|--------------|
| Anthropic | `computer_toolset_20260801` (GA on the Claude API since Aug 19, 2026) and `browser_toolset_20260801` | Client toolsets you host: pixels for desktops, accessibility-tree element refs for browsers | Desktop and browser in one API; batch actions |
| OpenAI | Agents API computer use (hosted browser added Sep 29, 2026, public beta), `computer_use` in the Responses API (GPT-6.1 Terra, Oct 26, 2026, $1/$5 per 1M), ChatGPT agent, dots | OpenAI-hosted browser on the managed Codex harness; a client-hosted tool on the Responses API; persistent agents on their own cloud computer | Per-origin approvals; credentials kept out of model input |
| Google | Gemini API computer use (GA Oct 30, 2026; `gemini-3.8-flash` recommended, `gemini-4-argon` the most capable option); Gemini Enterprise Computer Use sandboxes (GA Sep 9, 2026) | Browser, Android, and desktop actions on a normalized 0-999 grid | Built-in `require_confirmation` safety service; VPC-isolated managed sandboxes |
| Microsoft | GitHub Copilot computer use (public preview Oct 1, 2026), Copilot Autopilot (private preview), the UFO research framework | Accessible app content plus vision across desktop apps | Per-app approval grants; agent identity inside the customer tenant |
| Amazon | Nova Act | Purpose-built browser model | E-commerce workflows |
| Meta | Muse (Sep 8, 2026) | Consumer agent that runs a browser in a dedicated cloud VM | Retail and connector partnerships |

### What Differs Between Vendor APIs

The loop is the same everywhere. The contract around it is not, and these differences decide how much of the safety layer you build yourself.

| Concern | Anthropic computer toolset | Gemini computer use | OpenAI Agents API (hosted browser) |
|---------|----------------------------|---------------------|------------------------------------|
| Who hosts the environment | You | You (or Gemini Enterprise sandboxes) | OpenAI |
| Coordinate space | Pixels of the screenshot you return | Normalized 0-999 grid you map to the display | Not your concern; OpenAI executes |
| Built-in confirmation | Yours to build; Anthropic advises checking each action before it runs, because one batch can finish a multistep action | Safety service returns `require_confirmation` for categories such as financial transactions, communication tools, account creation, and legal terms | `browser_origin_access` approval for each new website origin; `browser_authentication` for sign-in |
| Credentials | Yours to keep out of the screen | Yours to keep out of the screen | Separate sign-in UI and session events endpoint; kept out of model input (not passkeys or QR sign-in) |
| Billing and data terms | Model tokens; ZDR-eligible, since screenshots and actions stay in your environment, except on Fable and Mythos, which require 30-day retention unless Anthropic expressly authorizes ZDR | Model tokens at the model's price | No separate API fee: model, tool, and container usage; US data residency only, no ZDR during the beta |

---

## The Screenshot-Reason-Act Loop

Every computer-use agent follows the same core loop, often called the "agent loop" or "action loop":

```
+------------------+
|  Capture Screen  |<-----------+
+--------+---------+            |
         |                      |
         v                      |
+------------------+            |
|  Send to LLM     |            |
|  (screenshot +   |            |
|   task context)  |            |
+--------+---------+            |
         |                      |
         v                      |
+------------------+            |
|  LLM Reasons     |            |
|  about next      |            |
|  action(s)       |            |
+--------+---------+            |
         |                      |
    +----+----+                 |
    |         |                 |
    v         v                 |
 [Action]  [Done]               |
    |                           |
    v                           |
+------------------+            |
| Execute Actions  |            |
| in order (click, |            |
|  type, key, ...) |            |
+--------+---------+            |
         |                      |
         +----------------------+
```

Each iteration:
1. **Capture**: Take a screenshot of the current display state.
2. **Send**: Pass the screenshot (base64 image) plus the conversation history to the LLM.
3. **Reason**: The model analyzes what is on screen and determines the next step toward the goal.
4. **Act**: The model outputs one or more tool calls (for example `left_click` at `[450, 320]`), which the runtime executes in order.
5. **Repeat**: A new screenshot is captured and the loop continues until the model signals completion.

The model maintains context across iterations through the conversation history, which accumulates screenshots and actions like a visual "memory" of what has happened. That history is also the cost driver: long sessions are input-dominated (see [Performance](#performance-latency-cost-throughput)).

**Batch actions change the loop's economics and its safety gate.** Claude's computer toolset can return several actions in one turn (click a field, type, press Return, take a screenshot). The executor runs them in order and stops at the first failure. One model round trip now covers several steps, which cuts latency and cost per step, but a single turn can also complete a consequential multistep action. Anthropic's guidance is to run the confirmation check before each action executes, not once per turn. In practice: inspect the whole batch when it arrives and pause before any consequential action, so a submit click in the middle of a batch cannot slip past a check that only looked at the first one.

---

## Claude Computer Use: Tools and API

Anthropic ships computer use as client-side tools: you host the environment and execute the actions, and Claude decides what to do. Computer use left beta on the Claude API on August 19, 2026 (Google Cloud on August 20) as a **toolset**, one `tools` entry that expands into many member tools. It is still beta on Amazon Bedrock, Claude Platform on AWS, and Microsoft Foundry. Supported models are Fable 5 and 5.1, Mythos 5 and 5.1, Opus 5 and 5.5, Sonnet 5 and 5.5, Haiku 5.5, and Opus 4.8.

### The Tools

| Tool | Declared as | What it does |
|------|-------------|--------------|
| Computer toolset | `{"type": "computer_toolset_20260801"}` (no `name`, no display size) | 17 member tools for screen, mouse, and keyboard |
| Browser toolset | `browser_toolset_20260801` | 31 member tools that drive a browser your application hosts |
| Bash | `{"type": "bash_20250124", "name": "bash"}` | Persistent shell session; commands share environment and working directory |
| Text editor | `{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}` | `view`, `create`, `str_replace` (unique match), `insert` |

**The 17 computer toolset members:**
- **Screen:** `screenshot`, `zoom` (captures a region at full resolution; on by default)
- **Mouse:** `left_click`, `right_click`, `middle_click`, `double_click`, `triple_click`, `left_click_drag`, `mouse_move`, `left_mouse_down`, `left_mouse_up`, `cursor_position`, `scroll`
- **Keyboard:** `type`, `key` (a key or combination such as `ctrl+s`, with `repeat` from 1 to 100), `hold_key` (up to 300 s)
- **Timing:** `wait` (up to 300 s)

Every member is on by default. A `configs` map sets `enabled` and `defer_loading` per member, which is a least-privilege lever: turn off what your executor does not implement or the task does not need.

### API Request and Response Shape

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(  # GA: no beta header
    model="claude-sonnet-5-5",
    max_tokens=16000,
    tools=[
        # No name or display size: coordinates come from the screenshots you return.
        {"type": "computer_toolset_20260801",
         "configs": {"zoom": {"enabled": False}}},  # disable members you do not implement
        {"type": "bash_20250124", "name": "bash"},
        {"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"},
    ],
    messages=[
        {"role": "user",
         "content": "Open Firefox, navigate to github.com, and find repos trending today."}
    ],
)
```

Claude's calls come back as `tool_use` blocks whose `name` is the member and which carry `"toolset_name": "computer"`:

```json
{"type": "tool_use", "id": "toolu_01WkoTUvSHDzTBu2xnGk8Ep8", "name": "left_click",
 "toolset_name": "computer", "input": {"coordinate": [512, 742]}}
```

Return one `tool_result` per call, all in the next user message, each echoing `"toolset_name": "computer"` (a result without it is rejected). Only `screenshot` and `zoom` need an image; a short `OK` is enough for the rest. In a batch, run the calls in order; if one fails, answer every later call with `is_error: true` and the text `Not executed: an earlier computer action in this turn failed.`

**Migration traps from the beta tool:**
- On the Claude API and Google Cloud, the 5.5 models (Opus, Sonnet, and Haiku) return HTTP 400 for the older `computer_20251124` tool. Amazon Bedrock still accepts it for all three. The two forms cannot share a request.
- The action moved from `input.action` to the block's `name`, so an executor written for the old tool needs a rewrite, not a type swap.
- `display_width_px`, `display_height_px`, and `display_number` are rejected. Coordinates are in the pixel space of the screenshots you return, and the API does not downscale for you, so screenshots must already fit the model's image limits: 2576 px on the long edge for Opus 4.7 and later (every toolset model), 1568 px for earlier models.
- Samples that target `claude-sonnet-4-20250514` or `claude-3-7-sonnet-20250219` point at retired models, and the `computer_20250124` version they use supports only Claude Sonnet 4.5, Haiku 4.5, and older models.

### The Browser Toolset

`browser_toolset_20260801` (August 19, 2026; Claude API and Google Cloud only, not Bedrock, Claude Platform on AWS, or Foundry) drives a browser your application hosts. It has 31 members: 27 on by default plus 4 opt-in (`javascript_exec`, `file_upload`, `read_console`, `read_network`). The important one is `read_page`, which returns the accessibility tree with element references such as `[ref_2]` (default depth 15, capped at 50,000 characters). Clicks, hovers, `scroll_to`, `form_input`, and `file_upload` can target a ref instead of a coordinate.

- **Why refs matter:** a ref survives layout shifts, animations, and resolution changes that break pixel coordinates. Refs are scoped to a tab and go stale after navigation or a material DOM change, so re-read the page after either.
- **Pixels remain available:** screenshot-and-click still works for canvas content and widgets with no accessible name.
- **Other members:** `find`, `get_page_text`, `form_input`, tab management (new, list, switch, close), and a `browser_state` block that reports the tab inventory and state changes such as downloads.
- **Built-in injection scanning:** Anthropic's classifiers scan returned page text and screenshots for prompt injection and steer the model to confirm that an instruction really came from the user (customers can opt out through support).

Anthropic's security guidance doubles as a least-privilege checklist: a fresh browser profile with no credentials, a network-layer domain allowlist that blocks loopback, link-local, and private ranges, http/https URLs only, and the four opt-in members left off unless the executor implements them and the task needs them.

---

## Architecture: Sandboxed Environments

Computer-use agents must run in isolated environments. The model has full control of mouse and keyboard, and you do not want that on your production workstation.

### Standard Architecture: Docker + VNC

```
+-----------------------------------------------------+
|  Docker Container                                   |
|                                                     |
|  Xvfb (Virtual X11) + Mutter (WM) + Tint2 (Panel)  |
|         |                                           |
|         v                                           |
|  +------------------+     +-------------------+     |
|  | Virtual Desktop  |---->| Screenshot Capture|     |
|  | 1280x800         |     | (scrot/maim)      |     |
|  | Firefox, apps    |     +--------+----------+     |
|  +------------------+              |                |
|                                    v                |
|                           +--------+----------+     |
|                           | Agent Runtime     |     |
|                           | - Calls Claude API|     |
|                           | - Executes actions|     |
|                           | - Manages loop    |     |
|                           +-------------------+     |
+-----------------------------------------------------+
```

### Managed Alternatives

You no longer have to run the display stack yourself:

| Option | What you get | Tradeoff |
|--------|--------------|----------|
| E2B and similar sandbox services | Ephemeral VMs with browsers preinstalled, an API for screenshots and input, automatic cleanup | You still own the agent loop and the executor |
| Gemini Enterprise Computer Use and Shell sandboxes (GA Sep 9, 2026) | VPC Service Controls, Private Service Connect ingress and egress, CMEK through Cloud KMS; idle sandboxes are descheduled with file-system state kept and resume in seconds | Tied to Google Cloud |
| OpenAI Agents API hosted browser (beta, Sep 29, 2026) | OpenAI runs the browser and the loop; per-origin approvals; sign-in outside model input | US residency only and no ZDR in the beta; OpenAI advises restricting the browser, or using one you control, when you need guaranteed confirmation before consequential actions |
| Cursor self-hosted machines (Sep 2, 2026) | Cloud coding agents, including computer use on Linux and Mac workers, run on machines or pools you own | Coding-agent context only |

### Key Environment Components

| Component | Purpose | Example |
|-----------|---------|---------|
| Xvfb | Virtual X11 display server | Creates a framebuffer without physical display |
| Mutter/Xfwm | Window manager | Handles window positioning, resizing |
| Tint2 | Task panel | Shows running applications |
| xdotool | Input injection | Executes mouse/keyboard commands |
| scrot/maim | Screenshot capture | Takes display snapshots as PNG |

---

## Browser vs Desktop Automation

| Dimension | Browser-Only | Full Desktop |
|-----------|-------------|--------------|
| Scope | Web apps only | Any GUI application |
| Setup complexity | Lower (headless browser) | Higher (full desktop env) |
| Targeting | Accessibility-tree refs or DOM, pixels as fallback | Pixel coordinates, plus OS accessibility APIs where available |
| Performance | Faster (smaller screenshots, text page reads) | Slower (full screen captures) |
| Reliability | Higher (predictable layouts, stable refs) | Lower (OS variations) |
| Use case | Web scraping, form filling | Legacy software, cross-app workflows |

Browser automation controls a web browser (navigate, fill forms, click buttons, handle SPAs). Desktop automation controls the full OS environment (launch applications, use native dialogs, interact with thick-client software, chain operations across multiple apps).

**Ref-based targeting is the reliability lever for web work.** Anthropic's browser toolset targets accessibility-tree refs first, and GitHub Copilot's computer use reads accessible app content alongside visual context. If an application exposes an accessibility tree, use it. Pure-pixel control is for canvases, remote desktops, and legacy thick clients.

---

## Comparison with Traditional Automation

Selenium, Playwright, and Puppeteer automate browsers via direct DOM access. Computer-use agents work with pixels. Both have a place in production.

| Feature | Selenium/Playwright | Computer Use Agent |
|---------|--------------------|--------------------|
| Speed | Fast (direct DOM) | Slow (screenshot + LLM) |
| Reliability | Brittle (selector changes) | Resilient (visual recognition) |
| Maintenance | Constant selector updates | Minimal (adapts to UI changes) |
| Anti-bot detection | Frequently blocked | Harder to detect |
| Cost per action | ~$0.001 | ~$0.01-0.06 (Sonnet-class model; see [Cost Per Action](#cost-per-action)) |
| Non-web support | No | Yes (any GUI) |

**Hybrid approaches** work best in production: Playwright handles high-volume, well-defined flows (login, navigation) while computer-use agents handle dynamic, unpredictable steps (visual verification, novel layouts, anti-bot sites). The browser toolset narrows the gap from the other direction: the agent gets DOM-level targets without anyone maintaining selectors.

---

## When Computer-Use Beats API Calls

**Use computer use when:** no API exists (legacy systems), anti-bot protections block Selenium, visual judgment is required (chart verification, PDF layout), UIs change faster than selectors can be maintained, or the workflow spans multiple desktop applications.

**Stick with APIs when:** a structured API is available (always prefer it), latency matters (sub-second), volume is high (thousands of actions/hour), or determinism is required (same input, same output).

---

## Error Handling and Recovery

Computer-use agents fail differently from API-based tools. The main failure modes:

### 1. Misclicks (Wrong Coordinates)

The model calculates coordinates from the screenshot but may miss by a few pixels:
- **Mitigation**: Capture a screenshot after consequential clicks to verify the expected state change. Use `zoom` to inspect dense UI before clicking, and target refs instead of pixels in browsers.
- **Recovery**: If the wrong element was clicked, the model can reason about the new state and correct course.

### 2. Stale Screenshots

The screen may have changed between capture and action execution (animations, popups, loading):
- **Mitigation**: Add a short wait before screenshots. Use the `wait` action when pages are loading. In browsers, re-read the page after navigation, because element refs go stale.
- **Recovery**: Re-capture and re-assess before continuing.

### 3. Infinite Loops

The model repeats the same action without making progress:
- **Mitigation**: Set a maximum iteration count sized to the task (around 50 actions for a form fill; OSWorld 2.0's hour-scale tasks use a 500-step budget) plus a spend cap enforced outside the agent.
- **Recovery**: After N repeated identical actions, force a different approach or escalate to a human.

### 4. Unexpected Dialogs

Cookie banners, popups, permission dialogs appear unexpectedly:
- **Mitigation**: Include instructions in the system prompt about handling common dialogs.
- **Recovery**: The model's visual reasoning usually handles these naturally: it sees the dialog and dismisses it.

### 5. Resolution and Scaling Mismatches

Coordinates come back in the pixel space of the screenshot you sent. If the executor injects them into a display of a different size, every click lands in the wrong place:
- **Mitigation**: Keep display scaling at 100% and send screenshots at a modest resolution (Anthropic's docs suggest 1024x768 or 1280x720 for desktop tasks, 1280x800 or 1366x768 for web apps, and nothing above 1920x1080). On macOS Retina displays, account for the 2x device pixel ratio.
- **Recovery**: If you downscale screenshots to save tokens or fit the image limit, scale every returned coordinate back up before injecting input.

### 6. Partial Batches

A batch can fail midway, for example when a click lands on a modal that just opened:
- **Mitigation**: Execute in order, stop at the first failure, return the halt text for the remaining calls, and capture a fresh screenshot.
- **Recovery**: Let the model re-plan from the new screenshot. Never "finish" the remaining actions on your own; they were planned against a screen that no longer exists.

### Error Handling Pattern

The agent loop should track action history and detect repeats. If the same action is emitted 3+ times consecutively, inject a message telling the model to try a different approach. Always set a hard maximum iteration count and capture a verification screenshot after consequential actions to detect state changes. See the full agent loop in the Code Examples section below.

---

## Performance: Latency, Cost, Throughput

### Latency Breakdown

Each iteration of the agent loop involves:

```
Screenshot capture:     ~100ms
Image encoding (base64): ~50ms
API call (with image):   ~2-5s  (model inference)
Action execution:        ~100ms
                        --------
Total per model turn:    ~2.5-5.5s  (a batch of several actions shares one turn)
```

A typical 10-step task takes 25-55 seconds without batching. Compare this to Playwright, which completes the same 10 steps in under 2 seconds. Long-horizon tasks are a different regime: OpenAI reports GPT-6 Astra averaging about 40 minutes per task on its OSWorld 2.0 offline set, against about 75 minutes for GPT-5.6 Sol (vendor-reported).

### Cost Per Action

Each action sends a new screenshot (Anthropic's docs estimate roughly 1,000 to 1,800 input tokens each) plus the accumulated history, so computer use is input-dominated and the cached-input price matters more than the output price. Declaring the computer toolset itself adds about 4,500 input tokens per request (disabling `zoom` saves about 410), a fixed prefix that caching absorbs.

The estimates below assume about 25K tokens of prior context per step, 1.5K new screenshot tokens, and 400 output tokens including thinking, at list prices:

| Model | List price per 1M (input / output / cache read) | Per action, warm cache | Per action, no cache |
|-------|-------------------------------------------------|------------------------|----------------------|
| Claude Haiku 5.5 (prompts up to 100K tokens) | $0.10 / $0.50 / $0.01 | ~$0.001 | ~$0.003 |
| Claude Sonnet 5.5 | $2 / $10 / $0.10 | ~$0.01 | ~$0.06 |
| Claude Opus 5.5 | $4 / $20 / $0.20 | ~$0.02 | ~$0.11 |

A 20-step workflow costs roughly $0.20 to $1.15 on Sonnet 5.5, or $0.40 to $2.30 on Opus 5.5, depending mostly on cache hit rate. Haiku 5.5 is an order of magnitude cheaper again, but its price applies only while the whole prompt, cache reads and writes included, stays at or under 100,000 tokens. A long session that crosses that line bills every request at $0.50/$2.50, so trim history on Haiku before it gets there, and weigh the capability gap: Anthropic reports 72.4% for Haiku 5.5 on its OSWorld 2.1 offline subset (partial credit, vendor-reported). For comparison, GPT-6 Astra lists at $10/$50 per 1M ($1 cached input), 2.5x Opus 5.5's list price, GPT-6.1 Terra at $1/$5 ($0.05 cached; above 272K input the whole request bills $2/$7.50), and Gemini 3.8 Flash at an introductory $0.75/$3.75 through December 31, 2026 ($1.50/$7.50 from January 1, 2027). Image token counts differ by vendor, so measure per-action cost on your own screenshots rather than porting these estimates.

Effort is the other cost knob: Opus 5.5 defaults to `medium` effort and Sonnet 5.5 to `high`, and thinking tokens bill as output.

### Throughput Optimization

- **Parallel sessions**: Run multiple sandboxes for concurrent tasks.
- **Batch actions**: Let the model chain predictable steps (click, type, Return) in one round trip, with the confirmation check applied to every action before it runs.
- **Prompt caching**: Keep the system prompt and tool list stable so history reads hit the cache.
- **Selective screenshots**: Only capture after uncertain actions; skip after typing text.
- **Resolution reduction**: Use 1024x768 instead of 1920x1080 to reduce token cost, and keep each side at 2000 px or less, because a request carrying more than 20 images is held to a stricter per-side limit.
- **Server-side pruning**: Drop old screenshots with the API's server-side tool-result clearing (context editing) rather than rewriting history in the client. On Fable 5.1, Opus 5.5, Sonnet 5.5, and Haiku 5.5, removing an earlier screenshot client-side changes the prefix that every later thinking block is bound to. For accounts created on or after August 31, 2026 (and older accounts that opt in), the request returns 400. With the `thinking-binding-controls-2026-08-01` beta header and `thinking.block_binding.prefix_mismatch_behavior: "drop_block"`, those blocks are dropped instead.
- **Early termination**: Teach the model to signal completion as soon as the goal is verified.

---

## Benchmarks: Reading the Numbers

Quoted computer-use scores swing by 30 points or more depending on the benchmark version, the metric, the effort level, and who ran the test. Ask for all four.

| Benchmark | Status | What it measures |
|-----------|--------|------------------|
| OSWorld-Verified | Saturated: self-reported leaders sit in the mid-80s | Short, single-session desktop tasks |
| OSWorld 2.0 / 2.1 (XLANG Lab, June 2026, arXiv 2606.29537) | The current long-horizon yardstick | 108 tasks across 31 self-hosted websites and desktop apps; a median of about 1.6 skilled-human hours per task; about 27 scoring checkpoints per task; a 500-step budget. **Binary completion is the primary metric**; partial credit is reported alongside |

Where the numbers stand:
- **Official leaderboard (XLANG, September 17, 2026 update):** Claude Opus 5 at max effort with batched tools scored 44.33% binary and 77.67% partial on the v2.1 full set. Hour-scale binary completion was still under 50%.
- **Vendor-reported, partial credit only:** Anthropic reports Opus 5.5 at 81.8% and Sonnet 5.5 at 80.1% on "OSWorld 2.1" and gives no binary figure; its own run puts Opus 5 at 74.0%, which matches no leaderboard row. Its Haiku 5.5 launch (October 7) reports 72.4% on an "OSWorld 2.1 (offline subset)", a smaller task set, so do not line that number up against the launch figures for the larger models. OpenAI reports GPT-6 Astra at 72.6% against 65.7% for GPT-5.6 Sol on its offline v2026.08.08 set.
- **Paper baseline (June 2026):** Claude Opus 4.8 at max thinking scored 20.6% binary and 54.8% partial.

Snorkel AI's analysis of OSWorld 2.0 runs names four failure modes, and each maps to a control you can build: agents rarely ask to clarify missing information (give them a clarification tool), they misperceive unfamiliar apps (add app-specific instructions), they submit prematurely or refuse to backtrack (add a verification step before submit), and they lose track over roughly 300-step trajectories (keep task state outside the context window).

Plan capacity on binary completion: a workflow that hits 80% of its checkpoints has still not finished the job. And re-run your own evals when a vendor changes serving, not only when the model ID changes. On September 25, 2026, OpenAI fixed an image-encoding bug that had degraded image understanding, including computer use, in GPT-6 Sol and Luna under unchanged model IDs, and told customers to rerun image evaluations.

---

## Real-World Applications

| Application | How It Works | Why Computer Use |
|------------|--------------|------------------|
| Legacy system integration | Agent navigates mainframe/thick-client UI, extracts data to structured format | No API exists for legacy software |
| Form filling / data entry | Reads source documents, fills web forms field by field, handles multi-page wizards | Government portals, insurance claims with complex conditional logic |
| QA and visual testing | Navigates app as a user, verifies visual rendering, reports issues in natural language | Goes beyond pixel-diff: understands layout and UX |
| Visual proof for code changes | Coding agents run the app, screenshot the change, and attach it to the pull request | Reviewers see the UI without checking out the branch (see PixelLeak below for how this goes wrong) |
| Competitive intelligence | Navigates product pages, captures pricing data from JS-rendered widgets | Works on sites that block traditional scrapers |
| Persistent personal and workplace agents | An always-on agent with its own cloud computer works across apps between requests (OpenAI dots, Meta Muse, Copilot Autopilot) | Most of the apps it touches have no agent API; see [Use Cases and Case Studies](06-use-cases-and-case-studies.md) |

---

## Security Considerations

| Risk | What Happens | Mitigation |
|------|-------------|------------|
| **Visible secrets** | Model sees passwords, sessions, notifications in screenshots | Ephemeral containers, a fresh browser profile with no saved credentials, sign-in through a channel the model cannot read (OpenAI's Agents API takes credentials through a separate sign-in UI) |
| **Unrestricted actions** | Agent can run shell commands, navigate anywhere, download files | Firewall rules, read-only FS, session time limits, toolset members the task does not need turned off, HITL before any consequential action (checked per action, including mid-batch) |
| **Data exfiltration** | Screenshots sent to the LLM provider contain sensitive data | On-premise deployment for regulated industries, mask sensitive UI fields |
| **Screenshot leakage** | Agents publish screenshots through channels nobody forbade. PixelLeak (Glow Security, September 29, 2026) found 13,000+ internal screenshots in 900+ public repositories after coding agents told to attach visual proof to private pull requests created public repos to host them | Destination policy at the action level: block or flag new public repos, pushes to personal accounts and gists, and visibility changes |
| **Prompt injection via UI** | Malicious site displays text to manipulate the agent | Vendor scanning where offered (automatic on Anthropic's browser toolset; opt-in, off by default, on Gemini computer use), confirmation gates the injected text cannot satisfy, and a network allowlist. System-prompt warnings alone are not a control |
| **Internal network reach** | Agent browses to loopback, cloud metadata, or private services | Network-layer domain allowlist that blocks loopback, link-local, and private ranges; http/https only |

Gemini's safety service doubles as a ready-made risk taxonomy for your own confirmation gate: it flags financial transactions, sensitive data modification, communication tools, account creation, data modification, user consent management, and legal terms and agreements.

Routing a computer-use agent to a cheaper model changes its injection exposure, not just its cost. Anthropic's Haiku 5.5 system card reports a Gray Swan prompt-injection attack success rate within 15 attempts of 7.1% overall but 24.4% on GUI computer use (vendor-reported). The GUI surface is the weakest one, so a model-routing change on a computer-use workload should trigger a security review, not only a cost review.

The cardinal rule: **never run computer-use agents on your production workstation or with access to real credentials unless in a fully sandboxed container**. For the full defense-in-depth design, see [Safety and Governance](07-safety-and-governance.md).

---

## Code Examples

### Minimal Agent Loop

```python
import anthropic, base64, subprocess

client = anthropic.Anthropic()
HALT = "Not executed: an earlier computer action in this turn failed."

def capture_screenshot() -> str:
    # Capture at the size you map clicks into: the toolset takes no display
    # dimensions, and the API does not downscale for you.
    subprocess.run(["scrot", "-o", "/tmp/screen.png"], check=True)
    with open("/tmp/screen.png", "rb") as f:
        return base64.standard_b64encode(f.read()).decode()

def xdo(*args: str) -> None:
    subprocess.run(["xdotool", *args], check=True)

def run_member(name: str, args: dict) -> None:
    """Execute one computer-toolset member. Raises on failure."""
    if name in ("left_click", "right_click", "middle_click", "double_click", "triple_click"):
        if "coordinate" in args:
            x, y = args["coordinate"]
            xdo("mousemove", str(x), str(y))
        button = {"right_click": "3", "middle_click": "2"}.get(name, "1")
        repeat = {"double_click": "2", "triple_click": "3"}.get(name, "1")
        xdo("click", "--repeat", repeat, button)
    elif name == "type":
        xdo("type", "--", args["text"])
    elif name == "key":
        xdo("key", "--repeat", str(args.get("repeat", 1)), args["text"])
    elif name != "screenshot":  # scroll, drag, wait, ... omitted for brevity
        raise NotImplementedError(name)

def run_agent(task: str, max_turns: int = 30):
    messages = [{"role": "user", "content": task}]
    tools = [
        {"type": "computer_toolset_20260801", "configs": {"zoom": {"enabled": False}}},
        {"type": "bash_20250124", "name": "bash"},
    ]
    for _ in range(max_turns):
        response = client.messages.create(
            model="claude-sonnet-5-5", max_tokens=16000, tools=tools, messages=messages,
        )
        if response.stop_reason != "tool_use":  # end_turn, refusal, max_tokens
            return response
        # Check every block in the batch against your confirmation policy here,
        # before any of them runs: a consequential action can sit mid-batch.
        results, failed = [], False
        for block in response.content:
            if block.type != "tool_use":
                continue
            if getattr(block, "toolset_name", None) == "computer":
                result = {"type": "tool_result", "tool_use_id": block.id,
                          "toolset_name": "computer"}
                if failed:
                    result.update(is_error=True, content=HALT)
                else:
                    try:
                        run_member(block.name, block.input)
                        if block.name == "screenshot":
                            result["content"] = [{"type": "image", "source": {
                                "type": "base64", "media_type": "image/png",
                                "data": capture_screenshot()}}]
                        else:
                            result["content"] = [{"type": "text", "text": "OK"}]
                    except Exception as e:
                        failed = True  # halt the rest of the batch
                        result.update(is_error=True, content=f"Error: {e}")
                results.append(result)
            elif block.name == "bash":
                if failed:  # do not run later blocks after a failure
                    results.append({"type": "tool_result", "tool_use_id": block.id,
                                    "is_error": True, "content": "Not executed: an earlier action failed."})
                    continue
                r = subprocess.run(block.input.get("command", ""), shell=True,
                                   capture_output=True, text=True, timeout=60)
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": (r.stdout + r.stderr) or "(no output)"})
        # Append the assistant turn unchanged (thinking blocks included).
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": results})
    raise RuntimeError("Max turns reached")
```

### Dockerfile for Sandboxed Environment

```dockerfile
FROM debian:trixie-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    xvfb mutter tint2 xdotool scrot firefox-esr python3 python3-venv \
    && rm -rf /var/lib/apt/lists/*
RUN python3 -m venv /opt/agent && /opt/agent/bin/pip install "anthropic>=1,<2"
RUN useradd --create-home agent
USER agent
ENV DISPLAY=:1
COPY agent.py /home/agent/agent.py
CMD Xvfb :1 -screen 0 1280x800x24 & sleep 1 && mutter & tint2 & \
    sleep 1 && /opt/agent/bin/python /home/agent/agent.py
```

---

## Interview Questions

### Q: A client has 500 insurance claim PDFs per day that must be entered into a legacy web portal with no API. Design a system using computer-use agents.

**Strong answer:**
I would build a pipeline with three stages. First, a document processing stage using an LLM to extract structured data from the PDFs (claim number, claimant name, amounts, dates). Second, a computer-use agent stage where each claim is processed by a Claude computer-use agent running in an isolated container with a virtual display. Because the portal is a web app, I would use the browser toolset's accessibility-tree refs for form fields rather than pixel coordinates, and my confirmation check would inspect every action in a batch before any of it runs, so a submit click in the middle of a batch still stops for approval. The agent captures a confirmation screenshot after submission. Third, a verification stage that uses a separate LLM call to compare the confirmation screenshot against the expected data to catch any entry errors.

For scale, I would run sandboxes in parallel, each processing claims sequentially. At roughly 2 minutes per claim, one sandbox handles about 240 claims in an 8-hour day, so three cover the steady-state volume; I would run 5 to 10 for headroom against retries, slow portal pages, and end-of-month peaks. I would add a dead-letter queue for claims that fail after 3 retries, with human review.

On Sonnet 5.5 list prices, 20 actions per claim at roughly $0.01 to $0.06 each (warm versus cold prompt cache) comes to about $0.20 to $1.15 per claim, or $100 to $575 a day for 500 claims. Keeping the prompt prefix stable so history reads hit the cache is the biggest cost lever, and even the top of that range is likely cheaper than the manual data entry team it replaces.

### Q: Compare computer-use agents with Selenium for web automation. When would you choose each?

**Strong answer:**
Selenium interacts with the DOM directly: it is fast, deterministic, and cheap. But it breaks when selectors change, gets blocked by anti-bot systems, and cannot handle tasks requiring visual judgment.

Computer-use agents are about 100x slower and 10x to 60x more expensive per action, but they adapt to UI changes because they work with what is rendered rather than with selectors. They handle anti-bot detection better because they generate human-like interaction patterns. And they can reason about visual layouts: verifying a chart rendered correctly or reading content from a canvas element that Selenium cannot inspect. Browser toolsets that target accessibility-tree refs now give the agent much of Selenium's precision without anyone maintaining selectors.

I would choose Selenium for high-volume, stable workflows where the target site is under my control. I would choose computer-use agents for one-off tasks, third-party sites that change frequently, cross-application desktop workflows, and any task where the human cost of maintaining selectors exceeds the LLM inference cost.

The best production systems use both: Playwright handles the predictable steps (authentication, navigation), and the computer-use agent handles the dynamic steps (interpreting results, making judgment calls).

### Q: A vendor says its computer-use model scores 81.8% on OSWorld. How do you turn that into a production capacity plan?

**Strong answer:**
I would first find out what the number is. Which benchmark: OSWorld-Verified is saturated in the mid-80s and tells me little, while OSWorld 2.x measures hour-scale tasks. Which metric: 81.8% is Anthropic's partial-credit figure for Opus 5.5 on OSWorld 2.1, and partial credit can sit 30 or more points above binary completion; the top binary score on the official leaderboard's v2.1 full set was 44.33% at its September 17, 2026 update (Opus 5 at max effort with batched tools). Which effort level and harness, and who ran it: vendor runs and leaderboard runs disagree, and Anthropic's own 74.0% for Opus 5 matches no leaderboard row.

Then I would ignore the headline and build our own eval: 50 to 100 real tasks on our applications, scored on binary completion with an independent verification check, plus minutes and dollars per completed task. If binary completion lands around 45%, the plan is a human-in-the-loop system, not an autonomous one: an escalation queue sized for more than half the volume at launch, with the agent doing the preparation and a human finishing or approving.

Finally, I would treat the eval as continuous. Serving-side changes move results without a model ID change, as OpenAI's September 2026 image-encoding fix for GPT-6 Sol and Luna showed, so the eval runs on a schedule and after every vendor notice, not just at model selection.

---

## References

- Anthropic. "Computer use tool" API documentation (`computer_toolset_20260801`, 2026)
- Anthropic. "Browser use tool" API documentation (`browser_toolset_20260801`, 2026)
- Anthropic. "Bash Tool" and "Text Editor Tool" API documentation
- Google. "Computer use" Gemini API documentation and changelog (GA October 30, 2026)
- Anthropic. Claude Haiku 5.5 launch post and system card (October 2026)
- OpenAI. "Introducing the Agents API" (September 2026)
- E2B. "Sandboxed Cloud Environments for AI Agents" (2025)
- XLANG Lab et al. "OSWorld 2.0" (arXiv 2606.29537, June 2026) and the official leaderboard at osworld-v2.xlang.ai
- WebArena Benchmark: Web Agent Evaluation Suite (2024)
- Glow Security. "How AI agents exposed developer screenshots from leading tech companies" (PixelLeak, September 2026)

---

*Next: [Building Tool-Use Agents](05-building-tool-agents.md)*
