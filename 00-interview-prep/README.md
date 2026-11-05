# AI System Design Interview Preparation

Interview prep for senior and staff AI engineering roles: 173 system design questions (including a dedicated Voice AI set and a set built from recent candidate interview reports), answer frameworks with a worked mock-interview transcript, common pitfalls, ten whiteboard exercises, behavioral prep, a quick-answer FAQ, and November 2026 hiring trends.

> **What's new (November 2026):** the question bank now runs continuously Q1-Q173, after eighteen questions built from candidate interview reports posted mostly between June and November 2026, [Q156-Q173](01-question-bank.md#questions-from-recent-interview-reports): design-doc review, model distribution and GPU scheduling, provider-side chat serving, KV caches and tokenizers from scratch, agents without a framework, AI-assisted coding and PR-review rounds, launch readiness, safety and mission screens, and forward deployed decomposition. The job market chapter now summarizes the loop formats candidates report by company, and the behavioral chapter covers the new "how do you use AI" and safety-screen themes. Earlier in November the bank gained eight October-2026 questions, [Q148-Q155](01-question-bank.md#advanced-questions---october-2026), covering agents that silence or outwait their own monitors, incident pipelines that separate prompt-injected from self-directed actions, an AI gateway as a supply-chain target, fine-tunes that lose their base model, prices set by prompt length and calendar, decision models, rolling an API-version change through a shared gateway, and patch SLAs once AI shrinks the patch gap. That update also gave the job market chapter November 1 compensation data, Indeed's record 7.3% AI share of US postings, the Anthropic S-1, and a caveat on reports that Anthropic dropped its coding interview (one hiring manager's practice, not company policy), and brought the FAQ up to October's model releases (Claude Haiku 5.5, GPT-6.1 Terra, and Gemini 4 Argon GA).
>
> **October 2026:** the question bank gained seven September-2026 questions (operating an agent on a model rated Critical for cyber, full-duplex voice front ends that delegate to a backend model, providers removing sampling parameters and forced tool calls, managed agent platforms, reading benchmark claims against public leaderboards, open-weight license gates, and patching a self-hosted inference stack) plus a new twelve-question [Voice AI](01-question-bank.md#voice-ai-questions) section covering architecture choice, latency budgets, turn-taking on phone audio, per-minute cost, evaluation, and calling compliance. The bank now runs continuously Q1-Q147, and the whiteboard set gained a tenth exercise on inference batching.

## Before You Start

This folder assumes you already write production code and know LLM basics (tokens, context windows, embeddings, what RAG is). If those are shaky, spend a week in [01-foundations](../01-foundations/) and the [Courses guide](../COURSES.md) first; interview prep on top of missing fundamentals produces fluent-sounding wrong answers, which is the worst possible outcome in a senior loop.

The files in this folder are designed to be read in order. Each builds on the last: questions teach the surface area, frameworks teach how to structure answers, pitfalls teach what kills offers, exercises rehearse the motion, behavioral covers the staff-level signal, and job-market trends cover who is hiring, for what, and at what pay.

## Read in Order

```mermaid
flowchart TD
    A[Start prep] --> B{Time available}
    B -->|2 weeks| C[01 Question Bank]
    B -->|1 week| D[02 Answer Frameworks]
    B -->|3 days| E[03 Common Pitfalls]
    C --> F[02 Answer Frameworks]
    D --> G[04 Whiteboard Exercises]
    F --> G
    E --> G
    G --> H[05 Behavioral]
    H --> I[06 Job Market Trends 2026]
    I --> J[Interview day]
```

## Role-Specific Prep Paths

```mermaid
mindmap
  root((Role Prep))
    AI Engineer IC
      Question Bank Q1 to Q49
      Answer Frameworks
      Whiteboard exercises
    Staff or Principal
      Advanced sets Q50 to Q135 and Q148 to Q155
      Reported rounds Q156 to Q173
      Behavioral STAR-L
      Eval and memory exercises
    Voice AI roles
      Voice AI set Q130 and Q136 to Q147
      Latency and telephony
      Calling compliance
    Applied Scientist
      Model Selection questions
      Evaluation questions
      Optimization questions
    PM or TPM for AI
      Common Pitfalls
      Job Market Trends 2026
      Answer Frameworks
```

## Files in This Folder

| File | Purpose |
|------|---------|
| [01-question-bank.md](01-question-bank.md) | 173 interview questions (Q1-Q173, continuously numbered), drawn from reported loops, job postings, and current releases and incidents, grouped by topic, with model answers and follow-ups (through November 2026), including a twelve-question Voice AI section. |
| [02-answer-frameworks.md](02-answer-frameworks.md) | Five structured answer frameworks (SPIDER, ETA, tradeoff, debugging, STAR-L) plus a worked 45-minute SPIDER mock-interview transcript. |
| [03-common-pitfalls.md](03-common-pitfalls.md) | Patterns that kill staff-level offers: hand-waving on tradeoffs, missing observability, ignoring failure modes. |
| [04-whiteboard-exercises.md](04-whiteboard-exercises.md) | Ten system design exercises with worked solutions, including evaluation pipeline design, agent memory, and inference batching. The closest simulation of a real loop. |
| [05-behavioral-for-ai-roles.md](05-behavioral-for-ai-roles.md) | Behavioral prep for AI-specific scenarios, including customer-facing deployment rounds in FDE loops and the past-project technical deep dive, with six worked STAR-L examples, compensation and leveling questions, and an out-loud practice guide. |
| [06-job-market-trends-2026.md](06-job-market-trends-2026.md) | Role taxonomy, comp ranges (with the date each levels.fyi row was checked), entry-level and layoff data, per-company AI-in-interview policies, and emerging titles (FDE, AI Eval Engineer, AI Reliability Engineer, MCP Engineer). |
| [07-faq.md](07-faq.md) | Short, direct answers to the most-asked questions about AI engineering, RAG, agents, models, eval, inference, memory, and security. Useful for quick reference and for newcomers to the field. |

## Practice With a Real Interviewer

Everything in this folder can be self-studied, but one part of the loop cannot: defending a design out loud while an interviewer interrupts, changes a constraint, and asks why you did not pick the other option. That is a skill, and it improves fastest with reps and direct feedback.

If you want those reps, Om offers 1:1 mock AI system design interviews, reviews of your written answers and stories, and ongoing mentorship for engineers moving into senior or staff AI roles. Book on [EngineBogie](https://enginebogie.com/u/ombharatiya) or [Topmate](https://topmate.io/ombharatiya). A good time to book is after you have worked through the answer frameworks and at least three whiteboard exercises, so the session tests your delivery rather than your reading.

## Companion Resources

- [Role Transition Guide](../TRANSITION_GUIDE.md) for prepping from backend, frontend, QA, PM, or EM into AI.
- [Recommended Courses](../COURSES.md) for foundational learning before interview prep.
- [Glossary](../GLOSSARY.md) for quick term definitions during prep.
- [Case Studies](../16-case-studies/) for production architectures that map directly to whiteboard prompts.

## Key Takeaways

- The files are designed to be read in order; jumping straight to questions without absorbing answer frameworks leaves answers unstructured.
- Whiteboard exercises (file 04) are the closest simulation to real interviews; do at least three before any loop.
- Behavioral prep (file 05) is what separates staff candidates from senior candidates; do not skip it.
- The November 2026 job market chapter (file 06) is a moat: candidates who know the current hiring market can ask better questions and tailor stories.
- Recheck this folder monthly; new question batches are added as hiring trends shift.
