# On-Device and Edge Deployment

Not every model has to run in someone else's cloud. Running LLMs **locally**, on a laptop, a workstation GPU, a phone, or an edge box, is a real deployment target in 2026, driven by privacy, offline operation, latency, and cost at steady volume. The catch is that the tool that makes local models easy to *try* (Ollama) is not the tool that serves them in *production* (vLLM), and the two get conflated constantly. This chapter sorts out the runtime stack, the prototype-to-production path, the hardware limits, and when local actually beats an API.

## Table of Contents

- [The Runtime Stack](#the-runtime-stack)
- [Platform Models: Apple Foundation Models](#platform-models-apple-foundation-models)
- [Why Ollama Is Not a Production Server](#why-ollama-is-not-a-production-server)
- [When Local Beats Cloud (and When It Does Not)](#when-local-beats-cloud-and-when-it-does-not)
- [Quantization for Local Serving](#quantization-for-local-serving)
- [Hardware](#hardware)
- [Prototype to Production](#prototype-to-production)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Runtime Stack

The key mental model: these tools are **not substitutes**. They occupy different layers.

| Tool | Layer | What it is for |
|------|-------|----------------|
| **Ollama** | Experience layer / local daemon | One-command model pull and run, OpenAI-ish API, single-user dev. Builds on llama.cpp; on Apple Silicon it is moving to MLX by default for supported architectures (starting with the v0.40.0 release candidate, September 2026). Keep it on the current release and off untrusted networks (see below). |
| **LM Studio** | Experience layer / GUI | A desktop GUI for browsing and running local models. Single-user focused. |
| **llama.cpp** | Inference engine | Portable C/C++ CPU/GPU inference (the GGUF format); runs almost anywhere; powers the experience-layer tools. |
| **MLX** | Inference engine | Apple's array framework; the fastest Apple Silicon path; research and fine-tuning. |
| **vLLM** | Serving system | High-throughput concurrent serving with PagedAttention and continuous batching; OpenAI-compatible. The production answer. |
| **vllm-metal** | Serving system (Apple Silicon) | vLLM's Apple Silicon plugin (v0.30.0, September 23, 2026): the vLLM scheduler, paged KV cache, continuous batching, and OpenAI server on MLX models, with batched MTP speculative decoding and a Homebrew install. |
| **SGLang / TensorRT-LLM** | Serving system | High-throughput servers (SGLang with RadixAttention prefix caching; TensorRT-LLM for maximum NVIDIA performance); production. Hugging Face TGI is in maintenance mode (repository archived March 2026), and Hugging Face recommends vLLM or SGLang for new deployments. |
| **ExecuTorch** | Embedded/mobile runtime | PyTorch-native on-device inference (phone to microcontroller); reached 1.0 in late 2025 and ships in apps for billions of users. Publishers now release official exports (Meta's Muse Glimmer 30B `.pte`, August 2026). |
| **Apple Foundation Models framework** | OS-provided model | An on-device model that ships with the OS, so the app downloads nothing; see the next section. |
| **Core ML / ONNX Runtime / MLC LLM** | Embedded/mobile runtime | Apple on-device, cross-platform, and compile-to-many-targets (including browser/WebGPU) respectively. |

---

## Platform Models: Apple Foundation Models

On iOS and macOS the first question is not which runtime to bundle but whether the OS model is enough. Apple's third-generation foundation model family (announced June 8, 2026, shipping in the iOS 27 cycle; App Store submissions for iOS 27 opened September 9) has five members:

| Model | Where it runs | What it is |
|-------|---------------|------------|
| AFM 3 Core | On device | 3B dense |
| AFM 3 Core Advanced | On device | 20B sparse, natively multimodal, activating 1B to 4B parameters per request; the full model lives in NAND flash and only the selected experts load into DRAM |
| AFM 3 Cloud | Private Cloud Compute | Server model |
| ADM 3 Cloud (Image) | Private Cloud Compute | Image generation and editing |
| AFM 3 Cloud Pro | Private Cloud Compute, on NVIDIA GPUs in Google Cloud | Agentic tool use and complex reasoning |

Apple says it trained with quantization-aware training and built the models in collaboration with Google. Two design lessons travel beyond Apple:
- **Flash-resident sparse models** break the "phones run 1-3B models" rule for *total* parameters: keep the experts in storage and page in only what a request routes to, so DRAM holds the active slice. Latency then depends on flash bandwidth and how predictable routing is per request.
- **The device-to-cloud escalation path is part of the platform.** An app can start on device and escalate to Private Cloud Compute under Apple's privacy guarantees, which is the hybrid architecture this chapter recommends, provided by the OS.

Check which of these models the Foundation Models framework exposes to third-party apps on your target OS version, and plan a fallback for devices without Apple Intelligence.

---

## Why Ollama Is Not a Production Server

Ollama and LM Studio are excellent for prototyping and wrong for a shared production endpoint, for an architectural reason worth teaching.

Ollama defaults to serving requests with very limited parallelism and queues excess requests first-in-first-out; a full queue returns an error. Each parallel slot also statically multiplies the context memory allocation. LM Studio is built for single-user scenarios without rate limiting or auth. Neither is designed to turn concurrent demand into throughput.

Nor is either hardened for exposure. Ollama's CVE-2026-103663 (CVSS v4 9.4, October 2026) is a path traversal in the `/api/pull` model download path: a malicious layer digest can write a binary into Ollama's library directory, which is writable in most Docker images, and that binary runs as root on the next restart. An Ollama port reachable from a shared network is a remote-code-execution surface. Stay on current releases, bind to localhost or a private interface, never expose `/api/pull`, and pull models only from a registry you mirror.

**vLLM** is, via two mechanisms: **PagedAttention** (the KV cache stored in non-contiguous blocks like OS paging, cutting the 60-80% KV memory waste of naive serving to under ~4%) and **continuous batching** (swap a finished request out and a queued one in mid-batch). The clearest first-party benchmark, from Red Hat: on a single datacenter GPU running an 8B model, vLLM reached roughly **793 tokens/sec versus Ollama's ~41**, about 19x, with far lower tail latency, and even a tuned Ollama trailed across all concurrency levels.

The takeaway is not "vLLM is tuned better." It is structural: Ollama and LM Studio serialize, vLLM batches continuously and pages the KV cache. For one user the difference is small; under concurrency it becomes roughly 16-20x. One caveat for honesty: most public head-to-head numbers run on a datacenter GPU to isolate the *software* difference, so do not read "vLLM beats Ollama" as "GPU beats Mac."

The Mac side of that gap is closing. vllm-metal brings vLLM's continuous batching and paged KV cache to Apple Silicon, so a shared endpoint on a large-memory Mac (a team server, an on-prem appliance) no longer forces you onto Ollama's serialized queue.

---

## When Local Beats Cloud (and When It Does Not)

**Lean local or edge when:**
- **Privacy or regulated data** that cannot leave the box (HIPAA, GDPR, contractual residency). A caveat to teach: the major API providers offer zero-data-retention enterprise tiers, so "privacy" alone no longer automatically decides for local. The counter-caveat: the newest frontier tiers come with strings. Claude Fable 5.1 requires 30-day retention unless Anthropic expressly authorizes ZDR, and both OpenAI (Private Safety Processing, in preview since August 19, 2026) and Anthropic (Enterprise Frontier Safeguards, announced September 1, with a phased rollout due to start later this fall) are pairing ZDR with automated safety monitoring. Read the retention terms per model, not per vendor.
- **Offline or air-gapped** operation (field devices, critical infrastructure).
- **A latency floor**: on-device removes the network round trip (often 50-200ms), which matters for tight interactive loops; total response latency still depends on the model and hardware.
- **Cost at steady, high volume**: with October 2026 rental prices, two H100s (about $4,000 a month) running a 70B-class open model break even against a $2/$10 API tier at roughly 1B tokens a month (about 33M a day), while the tie point against the cheapest tiers such as GPT-6 Luna (about 20B tokens a month) is at or beyond what two GPUs can sustain (worked example in the [Cost Optimization Playbook](07-cost-optimization-playbook.md#interview-questions)). Owned edge hardware you already paid for changes the math, so the break-even is workload- and quality-bar-dependent.

**Stay on cloud APIs when** you need frontier quality, have spiky or unpredictable demand (you would pay for idle GPUs; batch endpoints at ~50% off often beat local at medium volume), run low-to-moderate volume (below break-even, total cost favors APIs), or lack the ops capacity to run vLLM with autoscaling and monitoring. The 2026 consensus is usually a **hybrid**: small, private, offline, or cost-sensitive paths local, heavy or frontier or spiky paths to the cloud, within one product.

---

## Quantization for Local Serving

Quantization is what makes local serving viable; the [Quantization Deep Dive](../03-training-and-adaptation/07-quantization-deep-dive.md) covers the math, so here is just the deployment layer.

**GGUF** is the local-model format used by llama.cpp, Ollama, and LM Studio. Common quant levels trade quality for size: Q4_K_M is the practical sweet spot (roughly 1-3% quality loss versus FP16 at about a quarter of the size), Q5_K_M is noticeably better for code and reasoning at under ~1% loss, Q8_0 is effectively lossless at about half FP16, and Q2/Q3 save the most memory but degrade math and reasoning by 5-10% or more.

That last rule holds for naive post-training quantization of a model trained at 16-bit. Models built for low bits behave differently. PrismML's **Ternary Bonsai 2 27B** (September 2026, Apache 2.0, derived from Qwen3.8-27B) averages 1.72 bits per weight in a 5.95 GB file and reports 98.2% of FP16 quality across 14 thinking-mode benchmarks, with about 130 tok/s decode on an RTX 5090 and about 28 tok/s on an M5 Pro (all vendor-reported). The catch: it runs only on PrismML's forks of llama.cpp and MLX. At the other extreme, Cactus Compute's **Needle 3** is a 121M-parameter tool-calling and extraction model compressed to about 2 bits per weight in an 8-29 MB file. Prefer vendor QAT builds (Google ships QAT int4 Gemma builds, including Gemma 4 12B) over quantizing yourself.

The **VRAM rule of thumb**:

```
VRAM (GB) ≈ (params in billions × bits per weight) / 8   # model weights only
```

then add the KV cache (it grows with context length times concurrent requests) plus roughly 10-20% runtime overhead. So a 7B model's weights are roughly 14GB at FP16, ~7.7GB at Q8_0, and ~4.5GB at Q4_K_M, before that overhead. The operating rule everyone repeats: **use the highest-quality quant that fits with 10-20% headroom** for KV cache, activations, and context.

---

## Hardware

A model-size-to-hardware guide (Q4 quant assumed; planning guidance, not guarantees):

| Model size (Q4) | Min VRAM/RAM | Realistic hardware |
|-----------------|--------------|--------------------|
| 1-3B | 4-6 GB | Any modern GPU; high-end phones (NPU); AI PCs |
| 7-8B | 8 GB | Mainstream GPU; 16 GB Mac |
| 13-14B | 12 GB | Upper-mainstream GPU; 16-24 GB Mac |
| 32-35B | 24 GB | A 24 GB consumer GPU; 36-48 GB Mac |
| 27B ternary (~1.7 bits) | ~6-8 GB | 8 GB+ GPU or 16 GB Mac, with the vendor's runtime fork |
| 70B | ~40 GB+ | High-end or dual GPU; 64 GB+ Mac; or a datacenter card |
| 200B+ | 48 GB+, often multi-GPU / 128 GB+ unified | Multi-GPU rigs; large-unified-memory workstations |

Notes:
- **Consumer GPUs** top out at 24-32 GB of VRAM, which is the binding constraint on local model size.
- **Apple Silicon** shares one memory pool between CPU and GPU, so system RAM doubles as VRAM, letting a large-RAM Mac hold models a same-priced discrete GPU cannot. Apple's MLX path keeps improving: Ollama's MLX backend reports sizable prefill and decode gains on Apple Silicon from exploiting unified memory and is becoming the default for supported architectures, and a separate update adds NVFP4, NVIDIA's 4-bit floating-point format (not Apple's), reported around 20% faster than Q4_K_M.
- **NPUs** in phones and AI PCs advertise high TOPS, but a teaching nuance: TOPS alone does not predict LLM speed, because limited operator support and memory bandwidth gate real performance. NPUs suit lightweight, battery-efficient tasks; discrete GPUs still win for heavy local inference.
- **Mobile** is bandwidth-bound and memory-constrained: realistic *dense* on-phone models are sub-1B to about 3B (MBZUAI positions its open K2 Horizon 3.7B and 7B for phones and the 0.9B for watches), available app RAM is often under 4 GB even on flagships, and mobile memory bandwidth is 30-50x below a datacenter GPU. The on-device standard is 4-bit quantization. Sparse models with flash-resident experts raise the total-parameter ceiling (Apple's 20B AFM 3 Core Advanced activates 1-4B per request) without raising the DRAM budget.

### Small Models Specialize

The useful on-device answer is often several small specialists rather than one general model:

| Job | Example open model (2026) | Notes |
|-----|---------------------------|-------|
| Tool calling, routing, extraction | Cactus Needle 3 (121M, Apache 2.0) | Single 8-29 MB file; vendor claims it beats models 10x its size on mobile tool calls |
| General assistant on phone or watch | K2 Horizon 0.9B / 3.7B / 7B (Apache 2.0, open training data) | 0.9B at 128K context; 3.7B and 7B at 512K |
| Enterprise-friendly small model, no Chinese-origin weights | IBM Granite 4.2 3B / 8B (Apache 2.0) | Official FP8, MXFP4, NVFP4, GGUF, and MLX builds |
| Laptop-class multimodal (text, image, audio, video) | Gemma 4 12B (Apache 2.0) | Encoder-free design; official QAT int4 builds |
| Local coding agent | MiMo-V2.6-Distill-Qwen-9B (MIT) | Vendor-reported 61.1% SWE-bench Verified |
| Reasoning on a laptop | Ternary Bonsai 2 27B (Apache 2.0) | Needs PrismML runtime forks |

---

## Prototype to Production

1. **Prototype** with Ollama (CLI) or LM Studio (GUI) on a GGUF Q4_K_M model; validate quality and prompts on the smallest model that passes.
2. **Pick the largest model and best quant** that fits the target hardware with KV-cache headroom.
3. **Switch the serving engine** for any concurrent endpoint: vLLM (NVIDIA or AMD), vllm-metal (Apple Silicon), SGLang, or TensorRT-LLM (max NVIDIA). Keep the OpenAI-compatible API so application code barely changes.
4. **For mobile or edge**, first check the platform model (Apple's Foundation Models framework on iOS 27). Otherwise export to ExecuTorch, Core ML, or ONNX Runtime / MLC LLM, quantize to 4-bit (prefer vendor QAT builds), and budget for under 4 GB of RAM and the bandwidth limit.

Common pitfalls: treating Ollama or LM Studio as a server (it serializes under load); forgetting the KV cache when sizing memory (long context times parallel slots can dominate); over-quantizing (Q2/Q3 hurts reasoning); conflating "vLLM beats Ollama" with "GPU beats Mac"; assuming NPU TOPS equals LLM speed; mismatching engine to hardware (vLLM is GPU-first but now runs on Apple Silicon through vllm-metal, MLX is Apple-only, llama.cpp is the portability fallback); and adopting an exotic low-bit format without checking that your runtime can load it.

**Maturity:** server-side local serving is production-mature (vLLM is widely deployed with an OpenAI-compatible API). On-device and mobile is production-ready for *small* dense models (sub-1B to 3B) and for OS-provided models, and not for frontier ones. NPU-as-LLM-engine is still early; a discrete GPU and a large-unified-memory Mac remain the serious local paths in 2026. Benchmarks are catching up: MLPerf Inference v6.1 added an Edge Agentic test with growing multi-turn context.

---

## Interview Questions

### Q: A team prototyped on Ollama and wants to ship it as a shared API. What changes and why?

**Strong answer:**
Ollama is the wrong tool for a shared endpoint. It serves with limited parallelism and queues excess requests first-in-first-out, so under concurrency latency spikes and requests start failing. The fix is to switch the serving engine to vLLM (or SGLang or TensorRT-LLM), keeping the same OpenAI-compatible API so the app barely changes. vLLM wins structurally, not by tuning: PagedAttention stores the KV cache in non-contiguous blocks to eliminate most of the memory waste, and continuous batching swaps finished requests out and queued ones in mid-batch, so concurrent demand becomes throughput. First-party benchmarks show roughly an order-of-magnitude higher throughput and far lower tail latency under load. I would also right-size the model and quant to the target GPU with KV-cache headroom, and add autoscaling and monitoring, which Ollama does not provide.

### Q: When would you choose local or on-device inference over a cloud API?

**Strong answer:**
When data cannot leave the box for privacy or residency reasons, when the system must work offline or air-gapped, when I need the lowest possible latency by cutting the network round trip, or when I have steady high volume where reserved GPUs beat per-token pricing: two rented H100s break even against a $2/$10 API tier at roughly 1B tokens a month, but not against the cheapest API tiers. I would stay on an API for frontier quality, spiky demand where idle GPUs waste money, low volume below break-even, or when the team lacks the ops capacity to run a serving stack. In practice it is usually a hybrid: small, private, or offline paths run local on quantized models, and heavy or frontier or bursty paths go to the cloud. On phones specifically, I would plan for sub-1B-to-3B dense models, since mobile is memory and bandwidth constrained, or use the OS-provided model where one exists. I would also check retention terms model by model before assuming the cloud path is ZDR, since some frontier tiers now require retention or safety monitoring.

### Q: You are shipping an AI feature in an iOS app. Do you bundle your own model or use Apple's Foundation Models?

**Strong answer:**
Start with the OS model and earn the right to bundle. Apple's on-device models ship with the OS, so there is no download, no app-size hit, and no memory competition from a second model; the third-generation family adds a 20B sparse on-device model whose experts live in flash and page into DRAM per request, plus an escalation path to Private Cloud Compute for heavier reasoning. I would prototype the feature against it with my own eval set. I would bundle my own model only if evals show the OS model cannot do the task (a domain it was not tuned for, a strict output format it misses), if I need identical behavior across iOS and Android, or if I must support devices without Apple Intelligence. If I bundle, I pick the smallest specialist that passes (a sub-200M tool caller or a 1-3B dense model), use a vendor QAT int4 build, export through Core ML or ExecuTorch, and keep the OS model as the default path where it is available. Either way the eval set is the asset, because the OS model will change under me with each OS release.

---

## References

- Red Hat Developer, ["Ollama vs vLLM: a deep dive into performance benchmarking"](https://developers.redhat.com/articles/2025/08/08/ollama-vs-vllm-deep-dive-performance-benchmarking)
- vLLM, [docs](https://docs.vllm.ai/) and the [PagedAttention blog](https://blog.vllm.ai/2023/06/20/vllm.html)
- Ollama, [now powered by MLX on Apple Silicon](https://ollama.com/blog/mlx) and the [concurrency FAQ](https://docs.ollama.com/faq)
- PyTorch, [Introducing ExecuTorch 1.0](https://pytorch.org/blog/introducing-executorch-1-0/)
- Chandra and Krishnamoorthi (Meta), ["On-Device LLMs: State of the Union, 2026"](https://v-chandra.github.io/on-device-llms/)
- Apple Machine Learning Research, ["Introducing the third generation of Apple Foundation Models"](https://machinelearning.apple.com/research/introducing-third-generation-of-apple-foundation-models)
- vLLM, [vllm-metal releases](https://github.com/vllm-project/vllm-metal/releases)
- PrismML, [Ternary Bonsai 2 27B](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf)

---

*Next: [Prompt Engineering Fundamentals](../05-prompting-and-context/01-prompt-engineering-fundamentals.md)*
