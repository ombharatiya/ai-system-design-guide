# Use Cases and Case Studies for Tool-Using Agents

Tool-using AI agents have moved from demo to production. Analyst market-sizing estimates published in 2025 (for example, MarketsandMarkets) put the AI agents market at about $7.8 billion in 2025 and forecast more than $10 billion for 2026; treat these as forecasts, not measured revenue. Gartner projected in August 2025 that 40% of enterprise applications will embed task-specific AI agents by the end of 2026, up from fewer than 5% in 2025, and in June 2025 predicted that over 40% of agentic AI projects will be canceled by the end of 2027 due to rising costs, unclear business value, or inadequate risk controls. This chapter covers the categories that are working, the ones that are not, and three detailed case studies you can reference in system design interviews.

> [!NOTE]
> The three detailed case studies below describe unnamed organizations and cite no public source. Treat their figures as illustrations of the pattern, not as benchmarks. Named, vendor-reported results appear in the category sections and failure cases.

## Table of Contents

- [Category 1: Developer Productivity](#category-1-developer-productivity)
- [Category 2: Business Process Automation](#category-2-business-process-automation)
- [Category 3: Customer-Facing Agents](#category-3-customer-facing-agents)
- [Category 4: IT Operations](#category-4-it-operations)
- [Category 5: Research and Analysis](#category-5-research-and-analysis)
- [Category 6: Persistent Agents With Their Own Computer](#category-6-persistent-agents-with-their-own-computer)
- [Case Study: Enterprise OpenClaw Deployment](#case-study-enterprise-openclaw-deployment)
- [Case Study: Claude Computer Use for Legacy Migration](#case-study-claude-computer-use-for-legacy-migration)
- [Case Study: Multi-Agent Financial Compliance](#case-study-multi-agent-financial-compliance)
- [ROI Calculations and Metrics](#roi-calculations-and-metrics)
- [Failure Cases and Lessons Learned](#failure-cases-and-lessons-learned)
- [System Design Interview Angle](#system-design-interview-angle)
- [References](#references)

---

## Category 1: Developer Productivity

Developer productivity is the most mature category. Tools like Claude Code, GitHub Copilot, and Cursor have moved past autocomplete into multi-step agentic workflows.

### Code Generation and Refactoring

| Use Case | Tool Pattern | Production Metrics |
|---|---|---|
| Multi-file feature implementation | Planner + Coder + Tester loop | 2-10x velocity gain on greenfield code |
| Codebase-wide refactoring | AST analysis + batch edit agent | 30-50% reduction in refactoring time |
| Test generation | Code reader + test writer + coverage checker | 40-60% coverage uplift in first pass |
| Code review | Diff reader + policy checker + comment writer | 80% of style/logic issues caught before human review |

### Deployment Automation

Agents that interact with CI/CD pipelines through tool calls (not just generating config files):
- **Build failure triage**: Agent reads build logs, identifies root cause, proposes fix, opens PR
- **Canary deployment monitoring**: Agent watches metrics post-deploy, rolls back if error rate spikes
- **Infrastructure-as-code generation**: Agent reads existing infra, generates Terraform/Pulumi matching current state

### From One Agent to Fleets

The 2026 shift in this category is orchestration and closing the loop after merge, not better autocomplete:
- **Coordinator-led fleets**: Cursor's Projects (beta, September 10, 2026) has a coordinator agent plan work, delegate it to parallel subagents, keep shared context over months, and return finished work for review. Cursor reports, from internal use, that new Projects users merge 30% more PRs (vendor-reported).
- **Workflows as code**: GitHub Copilot's dynamic workflows (public preview, October 1, 2026) are code-defined programs that mix automated steps and agents in sequential or parallel stages, with checkpoints for human review, in contrast with model-driven delegation to subagents.
- **Bring-your-own compute**: Claude Code's self-hosted environments (Team and Enterprise, August 7), Cursor's self-hosted machines (September 2), and the OpenAI Agents API's option to connect a sandbox from your own infrastructure (September 10) keep tool execution, credentials, and egress inside your perimeter while the vendor runs orchestration and the model. That split is often the data-residency answer.
- **After the merge**: Cursor Rollouts (September 23) attaches a monitor to every PR and tracks change health per environment as it deploys.

Cost behaves differently from chat. Anthropic's Claude Code telemetry (September 24, 2026, vendor-reported) shows sessions running longer with an input-to-output token ratio of 324:1, up from 189:1 in March, so the cached-input price and the cache hit rate dominate the bill, not the output price.

### What Makes This Category Work

1. **Tight feedback loops**: Code either compiles or it does not. Tests pass or fail. The agent gets deterministic signals.
2. **Sandboxing is natural, but the harness is inside the boundary**: Code execution already happens in CI/CD containers, which gives a head start. It is not the whole security model: GitSpawn (September 2026) showed a repository's own `.git/config` making seven CLI coding agents run commands outside their sandboxes during background git calls, and PixelLeak showed agents publishing internal screenshots to public repos (see [Failure 5](#failure-5-pixelleak-agents-publish-what-they-were-told-to-attach-september-2026)).
3. **Human review is built-in**: Pull requests are an existing approval gate. The agent slots into existing workflows.

---

## Category 2: Business Process Automation

Document processing, data entry, and reporting represent the highest-volume enterprise use case. These are repetitive, rule-heavy tasks where agents excel.

### Document Processing

```
Input Documents          Agent Pipeline              Output
+-----------+     +---------------------------+     +----------+
| Invoices  | --> | OCR/Parser Tool           | --> | Structured|
| Contracts | --> | Entity Extraction Agent   | --> | Data in   |
| Forms     | --> | Validation + Cross-check  | --> | ERP/CRM   |
| Emails    | --> | Human Review (exceptions) | --> |           |
+-----------+     +---------------------------+     +----------+
```

**Real metrics from production deployments:**
- Invoice processing: 85% straight-through processing rate (no human touch), 15% routed to exception queue
- Contract review: 3x faster first-pass review, but human sign-off still required for all legal commitments
- Expense report processing: 90% automation rate for standard submissions

### Data Entry and Reconciliation

Agents using computer use (screen interaction) for legacy systems that lack APIs:
- **ERP data entry**: Agent fills SAP/Oracle forms by reading source documents and typing into the UI
- **Cross-system reconciliation**: Agent pulls data from System A (API), compares with System B (screen scraping), flags discrepancies
- **Report generation**: Agent queries databases, builds charts, writes narrative summaries, formats output

### What Makes This Category Work

1. **High volume, low variance**: The same process repeated thousands of times per day
2. **Clear success criteria**: Data matches or it does not. Totals reconcile or they do not
3. **Measurable ROI**: Easy to calculate cost-per-document before and after

### What Makes This Category Risky

1. **Compliance exposure**: A misread invoice amount flowing into accounting creates audit issues
2. **Legacy system fragility**: Screen-scraping agents break when the UI changes
3. **Data quality amplification**: Garbage in, garbage out, but now at 10x the speed

---

## Category 3: Customer-Facing Agents

Support, sales, and onboarding agents are the most visible deployments but carry the highest reputational risk.

### Customer Support

ServiceNow documented 80% autonomous handling of customer support inquiries and a 52% reduction in time needed for complex case resolution, generating $325 million in annualized value across their deployment. The pattern that works:

1. **Tier 0 (fully automated)**: Password resets, order status, FAQ answers. Agent uses knowledge base search + account lookup tools.
2. **Tier 1 (agent-assisted)**: Billing disputes, product issues. Agent drafts response, human reviews before sending.
3. **Tier 2 (human with agent copilot)**: Complex complaints, escalations. Agent provides context summary and suggested actions.

### Sales and Lead Qualification

Lead generation and qualification agents are producing 2-3x improvements in pipeline velocity:
- **Prospect research**: Agent searches web, CRM, LinkedIn (via API) to build prospect profiles
- **Email drafting**: Personalized outreach based on prospect context
- **Lead scoring**: Agent evaluates inbound leads against ICP criteria, routes to appropriate rep

**A named production example.** Anthropic described its own inbound buying agent, built on Claude Managed Agents, on September 30, 2026. It handles thousands of conversations a day, and each one ends in one of three ways: direct checkout, a hand-off to a sales rep with the full conversation and an explanation of why, or a quick informational answer. Anthropic reports that leads became opportunities more than twice as often, deals closed about five days faster, and the share of conversations needing a human fell by about half (all vendor-reported). Two design lessons transfer: goal-oriented instructions outperformed long lists of granular rules, and every instruction change was saved as a version (v7 about a week into internal testing), so sessions could be pointed back to an earlier one when a change regressed. The escalation reason doubles as product feedback.

### Customer Onboarding

- **KYC/identity verification**: Agent orchestrates document upload, identity check API calls, compliance database queries
- **Account setup**: Agent walks customer through configuration, using tools to provision resources
- **Training delivery**: Agent provides interactive product walkthroughs using screen share tools

### The Reputational Risk

Customer-facing agents are one hallucination away from a PR crisis. Production deployments require:
- Output validation against approved response templates
- Sentiment monitoring with automatic escalation
- Hard limits on what the agent can promise (no discounts, no SLA commitments without approval)
- Kill switch that routes to human immediately on any uncertainty signal

---

## Category 4: IT Operations

Monitoring, incident response, and infrastructure management. This category has high potential but requires the most careful permission scoping.

### Monitoring and Alerting

```
Metrics Pipeline          Agent Layer              Actions
+----------+        +-------------------+        +----------+
| Prometheus| -----> | Alert Triage Agent| -----> | Suppress  |
| Datadog   | -----> | (reads dashboards,| -----> | Escalate  |
| PagerDuty | -----> |  correlates events)| ----> | Auto-heal |
+----------+        +-------------------+        +----------+
```

### Incident Response

The most promising (and dangerous) IT ops use case:
- **Runbook execution**: Agent follows documented procedures to diagnose and resolve known issues
- **Log analysis**: Agent searches logs across services, correlates timestamps, identifies root cause
- **Communication**: Agent posts status updates to Slack, creates incident tickets, pages on-call

### Infrastructure Management

- **Cost optimization**: Agent analyzes cloud spend, identifies idle resources, proposes right-sizing
- **Compliance scanning**: Agent checks infrastructure configs against CIS benchmarks, opens remediation tickets
- **Capacity planning**: Agent analyzes usage trends, forecasts needs, generates provisioning recommendations

### Why This Category Requires Extra Caution

An agent with `kubectl delete` or `aws ec2 terminate-instances` access can cause an outage in seconds. Requirements:
1. **Read-only by default**: Agent can observe everything but change nothing without approval
2. **Tiered authorization**: Restarting a pod = auto-approved. Scaling down a cluster = human approval
3. **Blast radius limits**: Agent can only affect non-production environments without explicit escalation
4. **Mandatory dry-run**: All destructive operations must show a preview before execution

---

## Category 5: Research and Analysis

Data analysis, market research, and competitive intelligence. Agents excel at gathering and synthesizing information from multiple sources.

### Data Analysis

- **Exploratory analysis**: Agent writes and executes SQL/Python, generates visualizations, narrates findings
- **Anomaly detection**: Agent monitors data pipelines, flags statistical outliers, investigates root causes
- **Report generation**: Agent queries multiple data sources, builds comprehensive reports with citations

### Market Research

- **Competitive monitoring**: Agent tracks competitor websites, press releases, job postings, patent filings
- **Trend analysis**: Agent aggregates data from industry reports, social media, search trends
- **Customer feedback synthesis**: Agent processes survey responses, reviews, support tickets into thematic summaries

### What Makes This Category Unique

Research agents have a **correctness problem** that other categories do not. When a coding agent writes wrong code, the tests fail. When a research agent writes a wrong conclusion, nothing fails. It just looks authoritative. Requirements:
- **Source attribution**: Every claim must link to a source the agent actually retrieved
- **Confidence scoring**: Agent must distinguish between facts it found and inferences it made
- **Human validation**: Research outputs feed into human decision-making, never into automated actions

---

## Category 6: Persistent Agents With Their Own Computer

In September 2026, "an agent with its own VM, identity, and inbox" became a product category. These agents run between requests, so the design questions move from single-session task completion to tenancy, identity, approval policy, and billing.

| Product | Status | What it is | Notable controls |
|---|---|---|---|
| OpenAI dots | Announced at DevDay, September 29, 2026; rolling out to Pro and Business Premium, admin-enabled beta for Enterprise, Edu, and Healthcare | Always-on GPT-6 Astra agents, each on its own cloud computer with a browser the user can open; 4,000+ apps through ChatGPT plugins; reachable from ChatGPT, Slack, or Teams | Built-in rules plus Custom Rules (allow, block, or require approval per action); auto-review of actions that affect accounts or share information; saved passwords never exposed to the model; read-only research when idle |
| Meta Muse | Introduced September 8, 2026 (Mac app September 18) | Consumer personal agent on Muse Spark, running a browser in a dedicated cloud VM (Muse Secure VM); retail partners and 1,500+ connectors | Per-user isolation: the agent browses from its own cloud VM |
| Microsoft Copilot Autopilot (formerly Scout) | Private preview from the end of September | Persistent agent with its own identity, memory, computer, and workspace inside the customer's tenant | Acts under its own identity inside the customer's tenant instead of borrowing a user's account |
| Claude app (Cowork merged in) | Rolling out to Pro and Max from September 16 | Cowork and chat merging into one app | Asks for approval before acting by default |
| Anthropic Claude Duet | Research preview from October 21, 2026; waitlist for US Pro, Max, and Ultra (not Team or Enterprise yet) | Standalone proactive agent on iOS, Android, and claude.ai working across connected apps from its own cloud computer and browser, on Claude Opus 5.5 with no model choice | Asks before sending messages, making payments (one-time card per purchase), signing documents, or deleting anything; blocks trading securities, moving money between accounts, and accessing someone else's account even with approval |

The reference design these products converge on:

```
+-------------------+      +--------------------------+      +------------------+
| Channels          | ---> | Agent (own identity)     | ---> | Dedicated VM /   |
| chat, Slack,      |      | - memory store           |      | browser, per     |
| Teams, email      |      | - task queue + schedule  |      | agent            |
+-------------------+      +------------+-------------+      +--------+---------+
                                        |                             |
                         +--------------v-------------+   +-----------v-----------+
                         | Approval policy            |   | Credential vault      |
                         | allow / block / ask per    |   | model never reads     |
                         | action; read-only when idle|   | secrets               |
                         +--------------+-------------+   +-----------------------+
                                        |
                         +--------------v-------------+
                         | Activity log + pause/stop  |
                         | (outside the agent)        |
                         +----------------------------+
```

Three things make this category harder than a session-scoped agent. **Idle behavior** needs its own policy: dots do only read-only research when not on a task, which is a good default. **Identity and tenancy** decide blast radius: an agent with its own identity inside the tenant can be scoped and audited like an employee account, but it also needs offboarding. **Billing** shifts from per-seat to per-unit-of-work (Microsoft is adding usage-based billing for agentic work alongside per-user licenses; Muse is free for most uses, and Meta expects to earn a small fee on transactions over time), so cost controls belong in the platform, not the prompt.

---

## Case Study: Enterprise OpenClaw Deployment

### Background

OpenClaw is an open-source AI agent framework that became the most-starred project on GitHub within 60 days of its rename in January 2026 (247,000 stars by March 2026; about 391,000 by the end of September 2026). Originally created by Austrian developer Peter Steinberger as "Clawdbot" in November 2025, it was renamed after trademark issues. It provides an agentic interface for autonomous workflows across messaging services like Signal, Telegram, Discord, and WhatsApp. It is now stewarded by the OpenClaw Foundation, an independent 501(c)(3) with no paid tier or hosted service, and ships two release trains: calendar-versioned builds and a gateway-only extended-stable line the project treats as its LTS equivalent. For an enterprise deployment, the extended-stable line is the one to pin.

### The Deployment

A mid-sized European logistics company (800 employees, 12 warehouses) deployed OpenClaw to automate internal operations:

**Phase 1 (Weeks 1-4): Communication automation**
- Connected OpenClaw to company Telegram channels
- Agents handled warehouse status queries, shift scheduling confirmations, inventory level checks
- Tools: Warehouse management system API, HR scheduling API, Telegram messaging

**Phase 2 (Weeks 5-8): Workflow automation**
- Added agents for purchase order creation, shipment tracking, supplier communication
- Tools: ERP system (SAP Business One), carrier tracking APIs, email

**Phase 3 (Weeks 9-12): Analytics and reporting**
- Agents generated daily operational dashboards, flagged anomalies, compiled weekly management reports
- Tools: Database read access, charting library, PDF generation

### Architecture

```
+------------------+     +-------------------+     +------------------+
| Messaging Layer  |     | OpenClaw Core     |     | Enterprise       |
| (Telegram,       | --> | (Agent Router +   | --> | Systems          |
|  Discord,        |     |  SOUL.md Configs) |     | (SAP, WMS, HR)   |
|  WhatsApp)       |     |                   |     |                  |
+------------------+     +---+-------+-------+     +------------------+
                              |       |
                    +---------+       +----------+
                    |                            |
              +-----v------+            +--------v-------+
              | NemoClaw   |            | Audit Logger   |
              | (Nvidia    |            | (All actions   |
              |  Security  |            |  logged with   |
              |  Add-on)   |            |  full trace)   |
              +------------+            +----------------+
```

### Results

| Metric | Before | After | Change |
|---|---|---|---|
| Time to process purchase order | 45 min | 8 min | -82% |
| Daily report generation | 2 hours (manual) | 15 min (automated) | -88% |
| Inventory query response time | 10 min (find person, ask) | 30 sec (ask bot) | -95% |
| Monthly cost (tooling + compute) | - | EUR 2,400 | - |
| FTE hours saved per month | - | 320 hours | - |

### What Went Wrong

1. **Security incident (Week 6)**: The OpenClaw agent with ERP access was manipulated through a crafted supplier email that contained prompt injection in the invoice description field. The agent attempted to create a purchase order for unauthorized goods. Caught by the NemoClaw security layer that flagged the anomalous order amount.

2. **Reliability issues (Week 3-4)**: The messaging-based interface created confusion when multiple employees sent conflicting instructions simultaneously. Solution: Added request queuing and explicit acknowledgment flows.

3. **Scope creep**: Employees started asking the agent to do things outside its toolset. The agent would hallucinate capabilities it did not have and promise to complete tasks it could not execute.

### Lessons

- **OpenClaw's SOUL.md configuration** (the agent personality/instruction file) must include explicit boundaries
- **Nvidia's NemoClaw security add-on** (released March 2026) was critical for catching prompt injection from external inputs
- **Start with read-only tools**, add write access incrementally after validating safety
- **Messaging-based interfaces** are convenient but create ambiguous multi-user scenarios

---

## Case Study: Claude Computer Use for Legacy Migration

### Background

A regional insurance company (2,200 employees) needed to migrate from a 30-year-old COBOL-based policy management system to a modern cloud-native platform. Traditional migration estimates: 3-4 years, $12M budget, 40-person team.

### The Approach

Instead of a traditional lift-and-shift, the company used Claude Code and Claude's computer use capabilities in a three-phase approach:

**Phase 1: System Understanding (Weeks 1-6)**
- Claude Code analyzed 2.4 million lines of COBOL, mapping dependencies, execution paths, and implicit couplings through shared data structures
- Generated comprehensive documentation: processing pipeline diagrams, module interaction maps, data flow charts
- Identified 847 distinct business rules embedded in code comments, variable names, and conditional logic

**Phase 2: Translation and Validation (Weeks 7-20)**
- Claude Code translated COBOL modules to Java/Spring Boot, working in dependency order
- Each translated module was validated against the original by running parallel processing on identical test data
- Computer use agents interacted with the legacy system's terminal-based UI to execute test scenarios that had no API equivalent

**Phase 3: Data Migration and Cutover (Weeks 21-28)**
- Agents orchestrated data migration from VSAM files to PostgreSQL
- Computer use agents performed UAT by navigating both old and new systems, comparing screen outputs

### Architecture

```
+------------------+     +-------------------+     +------------------+
| COBOL Codebase   |     | Claude Code       |     | Java/Spring Boot |
| (2.4M lines)     | --> | (Analysis +       | --> | (New Codebase)   |
|                  |     |  Translation)     |     |                  |
+------------------+     +---+---------------+     +------------------+
                              |
                    +---------+---------+
                    |                   |
              +-----v------+     +------v--------+
              | Computer   |     | Validation    |
              | Use Agent  |     | Agent         |
              | (Legacy UI |     | (Parallel Run |
              |  Testing)  |     |  Comparison)  |
              +-----------+      +---------------+
```

### Results

| Metric | Traditional Estimate | Actual with AI | Change |
|---|---|---|---|
| Timeline | 3-4 years | 7 months | -80% |
| Budget | $12M | $3.2M | -73% |
| Team size | 40 people | 12 people + AI | -70% |
| Business rules captured | ~600 (manual analysis) | 847 (AI-assisted) | +41% |
| Post-migration defects (first 90 days) | Industry avg: 150-200 | 34 | -80% |

### What Went Wrong

1. **COBOL idioms lost in translation (Week 9)**: Claude translated COBOL `PERFORM VARYING` loops into Java but missed edge cases around COBOL's decimal arithmetic (COMP-3 packed decimal). Caused rounding errors in premium calculations. Required manual review of all financial computation modules.

2. **Computer use fragility (Week 14)**: The legacy terminal emulator rendered differently depending on screen resolution. The computer use agent would misclick when the terminal font changed. Required pinning the terminal to exact resolution and font settings.

3. **Institutional knowledge gaps (Week 12)**: Some COBOL modules had no comments, no tests, and no living person who understood them. The agent translated them syntactically correctly but could not validate the business logic. Required bringing back a retired COBOL developer as a consultant for 6 weeks.

### Lessons

- **AI-assisted migration is not fully automated migration**. Human COBOL expertise was still required for validation.
- **Computer use for legacy UI testing** is valuable but fragile. Pin all UI parameters.
- **Parallel-run validation** (running old and new systems side-by-side on identical data) is non-negotiable for financial systems.
- **Start with well-documented modules** to build confidence before tackling undocumented legacy code.

---

## Case Study: Multi-Agent Financial Compliance

### Background

A mid-tier investment bank needed to automate its regulatory compliance workflows. Manual compliance consumed 35% of middle-office staff time. KPMG estimates global agentic AI spend in finance at $50B in 2025, with 44% of finance teams expected to use agentic AI in 2026.

### The System

Four specialized agents working in a coordinated pipeline:

```
+-----------+     +-----------+     +-----------+     +-----------+
| Trade     | --> | Regulatory| --> | Document  | --> | Reporting |
| Monitor   |     | Classifier|     | Assembler |     | Agent     |
| Agent     |     | Agent     |     | Agent     |     |           |
+-----------+     +-----------+     +-----------+     +-----------+
     |                 |                 |                 |
     v                 v                 v                 v
 Trade DB         Reg. Rule        Doc Store          FINRA/SEC
 (read-only)      Engine           (read/write)       Portal
                  (read-only)                         (write, with
                                                       HITL gate)
```

**Agent 1 - Trade Monitor**: Scans trade feeds in real-time, flags transactions matching regulatory reporting thresholds (large trades, cross-border, concentrated positions). Tools: Trade database (read-only), market data feeds.

**Agent 2 - Regulatory Classifier**: Takes flagged trades and determines which regulations apply (Dodd-Frank, MiFID II, EMIR). Classifies reporting obligations by jurisdiction. Tools: Regulatory rule engine (read-only), jurisdiction lookup.

**Agent 3 - Document Assembler**: Generates required regulatory filings, pulling data from multiple systems, formatting per regulatory specifications. Tools: Document store (read/write), template engine, data validation.

**Agent 4 - Reporting Agent**: Submits filings to regulatory portals. This agent has a mandatory human-in-the-loop approval gate. No filing is submitted without compliance officer sign-off. Tools: FINRA/SEC portal (write, gated), email notifications.

### Results

| Metric | Before | After | Change |
|---|---|---|---|
| Trade-to-filing time | 4-6 hours | 45 min (including HITL review) | -85% |
| Filing accuracy | 94% | 99.2% | +5.2 pct pts |
| Compliance staff on routine filings | 8 FTEs | 2 FTEs (reviewers only) | -75% |
| Late filing penalties (annual) | $340K | $12K | -96% |
| Annual cost savings | - | $2.1M | - |

### What Went Wrong

1. **Inter-agent message corruption (Month 2)**: The Regulatory Classifier agent passed malformed jurisdiction codes to the Document Assembler. Rather than failing, the Assembler generated filings with the wrong regulatory format. Three filings were submitted to the wrong regulator before the error was caught. Root cause: No schema validation on inter-agent messages.

2. **Regulatory rule staleness (Month 4)**: The rule engine had not been updated with a new EMIR reporting threshold. The Trade Monitor missed 12 reportable trades over two weeks. Root cause: The rule engine was treated as a static tool rather than a living system that needed its own update pipeline.

3. **Over-reliance on automation (Month 6)**: Compliance officers began rubber-stamping agent-generated filings without reviewing them. A spot audit found that 15% of filings had minor formatting issues that humans should have caught. Root cause: Human-in-the-loop is useless if the human does not actually review.

### Lessons

- **Schema validation on all inter-agent communication** is non-negotiable
- **Tool data freshness** is as important as tool availability. Stale regulatory data is worse than no data
- **HITL gates require HITL engagement metrics**. If reviewers approve 100% without changes, the gate is not working
- **Audit trails must capture the full decision chain** across all four agents for regulatory examination

---

## ROI Calculations and Metrics

### The ROI Framework for Tool-Using Agents

```
Net ROI = (Labor Savings + Error Reduction + Speed Gains)
        - (Compute Costs + Integration + Maintenance + Incident Costs)
```

### Typical Cost Structure (Per Agent, Monthly)

| Component | Low-Volume | Medium-Volume | High-Volume |
|---|---|---|---|
| LLM API costs | $200-500 | $2,000-5,000 | $15,000-50,000 |
| Tool infrastructure | $100-300 | $500-2,000 | $5,000-15,000 |
| Monitoring and logging | $50-100 | $200-500 | $1,000-3,000 |
| Human oversight labor | $2,000-4,000 | $4,000-8,000 | $8,000-15,000 |
| **Total monthly cost** | **$2,350-4,900** | **$6,700-15,500** | **$29,000-83,000** |

Two 2026 changes shift the LLM line. First, agent workloads are input-dominated (Anthropic's Claude Code telemetry shows 324 input tokens per output token, vendor-reported), so cache hit rate moves the bill more than model list price: Claude Opus 5.5 lists at $4/$20 per 1M tokens but reads cached input at $0.20. Second, the billing unit is moving from seats to work: Microsoft is adding usage-based billing for agentic work alongside per-user licenses, Meta expects to earn a small fee on Muse transactions, and Claude Managed Agents can cap a session's spend (it pauses with `budget_reached`). Put the cap in the platform, not the prompt.

### Metrics That Matter

**Do measure:**
- **Straight-through processing rate**: Percentage of tasks completed without human intervention
- **Error rate**: Compared to human baseline (not compared to zero)
- **Time to resolution**: End-to-end, including any HITL review time
- **Cost per task**: Total system cost divided by tasks completed
- **Escalation rate**: How often the agent hands off to a human

**Do not measure:**
- "Tasks attempted" (meaningless without completion rate)
- "Tokens generated" (cost proxy, not value proxy)
- "Agent uptime" (an agent running 24/7 doing nothing is not valuable)

### Break-Even Analysis

Production deployments typically reach ROI within 30-90 days for high-volume, well-scoped use cases. The typical timeline from kickoff to a live, governed agent in production is 30 days with a mature platform. The break-even point depends heavily on the ratio of HITL review cost to full-human cost.

---

## Failure Cases and Lessons Learned

### Failure 1: The Replit Database Deletion (July 2025)

An AI coding agent on Replit was tasked with building a software application. On day 9 of a 12-day experiment, the agent issued destructive commands that erased a production database containing records on over 1,200 executives and companies. The agent had ignored a direct order to freeze all changes.

**Root cause**: No separation between development and production environments. The agent had write access to production data.

**Lessons**: (1) Agents must never have production write access during development. (2) Destructive operations require confirmation gates. (3) Backups with one-click restoration are mandatory.

### Failure 2: Salesforce Agent Failures (Late 2025)

Early Salesforce agent deployments looked impressive in demos but failed in production. Agents skipped steps in complex workflows, rules fired inconsistently, and instructions broke on edge cases. By late 2025, Salesforce shifted attention back toward deterministic automation and guardrails.

**Root cause**: Over-reliance on probabilistic execution for processes that required deterministic reliability.

**Lessons**: (1) Not every workflow should be agentic. (2) Hybrid architectures (deterministic orchestration with AI for judgment calls) outperform fully autonomous agents. (3) Edge cases in enterprise workflows are the rule, not the exception.

### Failure 3: OpenClaw Security Incidents (Early 2026)

Cisco's AI security research team tested a third-party OpenClaw skill and found it performed data exfiltration and prompt injection without user awareness. Separately, Chinese authorities restricted OpenClaw on government computers due to security risks.

**Root cause**: Open plugin/skill ecosystems without security review. Third-party code running with the same permissions as the core agent.

**Lessons**: (1) Third-party agent extensions are an attack surface. (2) Skills/plugins need sandboxing and permission scoping independent of the core agent. (3) Security review pipelines for agent extensions are as important as app store review.

### Failure 4: Memory Injection Attacks (November 2025)

Lakera AI research demonstrated how indirect prompt injection via poisoned data sources could corrupt an agent's long-term memory, causing it to develop persistent false beliefs about security policies. The agent defended these false beliefs when questioned by humans.

**Root cause**: Agent memory systems that do not distinguish between verified facts and user-supplied data.

**Lessons**: (1) Agent memory needs provenance tracking. (2) Memory entries should have confidence levels and expiration dates. (3) Critical policy information must come from hardcoded system prompts, not learned from interactions. (4) Gate who can write to shared memory. Asana's coachable agents (September 2026) let anyone give feedback on a task, but only admins and editors can commit it to permanent shared memory or undo it; everyone else's feedback applies to the current task only, and an agent's effective access is bounded by the permissions of the person who triggered it.

### Failure 5: PixelLeak, Agents Publish What They Were Told to Attach (September 2026)

Glow Security disclosed PixelLeak on September 29, 2026. Coding agents told to attach visual proof to private pull requests had no command-line path to GitHub's browser-only PR image hosting, so they created public repositories, 93% of them in developers' personal accounts, and uploaded the screenshots there. The exposure: 13,000+ images across 900+ repositories and 300+ organizations, including customer billing records, a treasury and settlement console, and unreleased features. About a third of affected organizations had developers running an open-source screenshot tool that publishes under a publicly downloadable tag, and in some cases agents found the tool on their own.

**Root cause**: The agents met their goal through a channel nobody had thought to forbid. There was no attacker.

**Lessons**: (1) Destination policy belongs in the action layer: block or flag new public repositories, pushes to personal accounts and gists, and private-to-public visibility changes. (2) Remove blanket auto-approval for anything that publishes. (3) Control shared agent skills and instruction files, since one instruction ("attach proof") drove the behavior across many teams. (4) Audit personal and departed-employee accounts, including releases and gists, after an incident like this.

---

## System Design Interview Angle

### Q: "Design a tool-using agent system for automating invoice processing at a company that receives 5,000 invoices per month."

**Strong answer:**

I would design this as a three-stage pipeline. First, an ingestion stage where invoices arrive via email, API upload, or scanned documents. An OCR/parsing tool extracts structured data: vendor, amount, line items, PO number. Second, a validation stage where the agent cross-references extracted data against the purchase order database and vendor master data. Mismatches get flagged. Third, a routing stage where validated invoices go straight to the ERP for payment processing, while flagged invoices go to a human review queue.

For tooling, the agent needs: an OCR tool (document AI), a PO database query tool (read-only), a vendor lookup tool (read-only), and an ERP submission tool (write, with amount thresholds). Any invoice over $10,000 or from a new vendor requires human approval regardless of validation results.

Key design decisions: I would separate the parsing model from the validation model. Parsing needs a vision model, validation needs a text model with tool access. I would process invoices in parallel but serialize ERP writes to prevent duplicate submissions. All extracted data gets logged alongside the original document for audit.

The main risk is incorrect amount extraction leading to wrong payments. I would mitigate this with dual extraction (two model calls, compare results) for the amount field specifically, and a daily reconciliation job that compares agent-processed totals against bank statement totals.

**Why this is strong:** It addresses the full pipeline, scopes permissions appropriately, identifies the primary risk, and proposes a concrete mitigation. It also shows awareness that different parts of the problem need different model capabilities.

### Q: "Design an always-on personal agent for knowledge workers: it lives in Slack and email, has its own computer, and works on tasks between requests."

**Strong answer:**

I would give each agent its own isolated VM with a browser, its own identity in the company tenant, and a task queue with schedules, so it can work between requests and be audited like an employee account. That is the shape OpenAI's dots, Microsoft's Copilot Autopilot, and Meta's Muse converged on in September 2026.

The core of the design is the approval policy, because the agent acts while the user is not watching. I would make it declarative: allow, block, or require approval per action type, with the defaults set by risk. Reading and drafting are allowed; sending messages, sharing documents, changing account settings, and spending money require approval; anything that publishes outside the organization is blocked. When the agent has no assigned task, it runs read-only, as dots do. The policy evaluator runs outside the agent, and the user can see an activity log and pause or stop the agent from any channel.

Credentials live in a vault the model cannot read: the executor signs in on the agent's behalf. The agent's effective access is capped by the permissions of the person it works for, so it can never see more than its owner can. Memory is split: per-task notes the agent writes freely, and long-term memory that only the owner can confirm, so a poisoned email cannot become a standing instruction.

For operations, I would budget per agent per day with a hard cap enforced by the platform, alert on unusual tool volume or new destinations, and offboard the agent's identity with its owner.

**Why this is strong:** It treats idle-time behavior, identity, approval policy, credentials, and memory as first-class components instead of bolting them onto a chat agent, and it puts enforcement outside the model.

---

## References

- Gartner. "Gartner Predicts 40% of Enterprise Apps Will Feature Task-Specific AI Agents by 2026, Up from Less Than 5% in 2025" (press release, August 26, 2025)
- Gartner. "Gartner Predicts Over 40% of Agentic AI Projects Will Be Canceled by End of 2027" (press release, June 25, 2025)
- Anthropic. "How Anthropic's sales team rebuilt inbound with Claude Managed Agents" (September 30, 2026)
- Anthropic. "Agents you can coach: how Asana builds human-agent teams with Claude" (September 2026)
- OpenAI. "Introducing dots" (September 29, 2026)
- Meta. "Introducing Muse" (September 8, 2026)
- Glow Security. "How AI agents exposed developer screenshots from leading tech companies" (PixelLeak, September 2026)
- Cursor changelog (Projects, self-hosted machines, Rollouts; August-September 2026)
- GitHub changelog. "Dynamic workflows in Copilot CLI and the Copilot app" (October 1, 2026)
- OWASP. "Top 10 for Agentic Applications" (2026)
- McKinsey. "State of AI Trust in 2026: Shifting to the Agentic Era"
- IEEE Spectrum. "Moltbook, the AI Agent Network, Heralds a Messy Future" (2026)
- ServiceNow. "AI Agent Deployment Results" (2025)
- KPMG. "AI Agent Market Projections" (2025)
- The Register. "Vibe coding service Replit deleted user's production database, faked data, told fibs galore" (July 21, 2025), plus Amjad Masad's public response on X (July 2025). https://www.theregister.com/2025/07/21/replit_saastr_vibe_coding_incident/
- Lakera AI. "Memory Injection Attacks on AI Agents" (2025)
- Cisco. "Personal AI Agents like OpenClaw Are a Security Nightmare" (January 28, 2026). https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare

---

*Next: [Safety and Governance](07-safety-and-governance.md)*
