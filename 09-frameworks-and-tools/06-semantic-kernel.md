# Semantic Kernel

**Semantic Kernel (SK)** is Microsoft's engine for enterprise-grade AI orchestration and the framework most existing **Azure/Microsoft** and **C#/.NET** AI code is written against. Its forward momentum has moved to the **Microsoft Agent Framework (MAF)**, the successor to AutoGen and SK built by the same teams. MAF 1.0 went GA for .NET and Python on April 2, 2026, with long-term support, and ships roughly weekly (Python 1.24.0 and .NET 1.27.0 on Oct 30, 2026). SK still ships (Python `semantic-kernel` 1.46.0 on Oct 28, .NET 1.81.0 on Oct 6, 2026) and Microsoft publishes a migration guide from SK to MAF.

**The practical rule**: maintain existing SK code, start new agent work on MAF. The concepts below (kernel functions, plugins, connectors, function calling) carry straight over.

## Table of Contents

- [Enterprise DNA](#enterprise-dna)
- [Plugins and Function Calling (Planners Are Gone)](#plugins-and-function-calling)
- [Memory and Connectors](#memory-and-connectors)
- [Multi-Language Support (C# vs. Python)](#multi-language-support)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Enterprise DNA

While LangChain is favored by startups, Semantic Kernel is favored by **Banks and Fortune 500s**.
- **Dependency Injection**: SK follows standard enterprise design patterns.
- **Strong Typing**: First-class support for C# types makes it highly reliable in large-scale mission-critical systems.
- **Security**: Deep integration with Azure Active Directory (Microsoft Entra ID) and Managed Identities.

---

## Plugins and Function Calling

1. **Kernel Functions**: The basic unit of logic (Native code or LLM prompts).
2. **Plugins**: A collection of functions (e.g., a "GitHub Plugin" or an "SQL Plugin").
3. **Planning is native function calling**: The Stepwise and Handlebars planners have been **deprecated and removed** from SK in Python, .NET and Java (Microsoft Learn, updated May 26, 2026). Automatic function calling (`FunctionChoiceBehavior.Auto()`) is the planning mechanism: the model picks the next function, the kernel invokes it, and the loop repeats. Microsoft publishes a Stepwise Planner migration guide.

This is the framework-churn story in miniature. Prompt-based planners existed because early models could not call tools reliably; once native tool calling arrived, the abstraction became overhead and was deleted. Long-running, multi-day processes belong in MAF workflows with checkpointing, not in a planner.

---

## Memory and Connectors

Semantic Kernel uses **Connectors** to abstract away the underlying infrastructure.
- **Universal Connectors**: One interface for Azure OpenAI, OpenAI, Mistral, and local ONNX models.
- **Vector Store Abstraction**: Switch between Azure AI Search, Pinecone, and Qdrant without changing the core business logic. On .NET, MAF builds on the same `Microsoft.Extensions.AI` chat-client abstractions, which shortens the migration.

---

## Multi-Language Support

SK is one of the few major frameworks that treats C# and Python as equals (Java is supported too, with a smaller surface). MAF keeps that promise for .NET and Python and adds Go in public preview.
- **The Pattern**: Develop and prototype in Python; deploy the core orchestration in C# for performance and type-safety.
- **Logic Sharing**: Shared prompt templates (.yaml) that work across both languages.

---

## Interview Questions

### Q: Why would a Staff Engineer choose Semantic Kernel over LangChain?

**Strong answer:**
**Architectural Alignment**. If an organization is already built on the .NET/Azure stack, the Microsoft stack fits into their existing CI/CD, monitoring (App Insights), and security (Entra ID) pipelines. LangChain often feels like an "external" piece of tech. Furthermore, the **Strong Typing** and **Dependency Injection** patterns prevent the "spaghetti code" that often plagues large LangChain projects. For an enterprise handling sensitive financial data, the **Native Azure integration** for security and auditing is the deciding factor. In late 2026 I would phrase the choice as **Microsoft Agent Framework versus LangGraph** for new work: MAF is GA with long-term support and has absorbed SK's enterprise features, so I keep SK for existing code and plan its migration rather than starting new projects on it.

### Q: What is the "Function Calling" abstraction in Semantic Kernel?

**Strong answer:**
SK uses a **Plugin-based model**. Every function (native C# or LLM-based) is registered with the Kernel, and its schema is advertised to the model as a tool. When the model emits a tool call, the Kernel looks up the function in the Plugin registry, validates the parameters, runs any **filters** (for approval, logging or redaction), and executes it. With `FunctionChoiceBehavior.Auto()` the kernel loops until the model stops calling tools, which is why SK's old planners were removed: native function calling does the planning. The governance hooks are the filters, so that is where I put human approval for sensitive functions and audit logging.

---

## References
- Microsoft Learn. "Introduction to Semantic Kernel": https://learn.microsoft.com/en-us/semantic-kernel/overview/
- Microsoft Learn. "Planning" (planners deprecated and removed; function calling): https://learn.microsoft.com/en-us/semantic-kernel/concepts/planning
- Microsoft. "Microsoft Agent Framework Version 1.0" (Apr 2026): https://devblogs.microsoft.com/agent-framework/microsoft-agent-framework-version-1-0/

---

*Next: [Microsoft Agent Framework, CrewAI, and Agent SDKs](07-autogen-crewai.md)*
