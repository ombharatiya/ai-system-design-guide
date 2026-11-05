# Structured Generation

Structured Generation is the process of forcing an LLM to produce output in a machine-readable format (JSON, YAML, CSV) with 100% syntactic reliability. The discipline has moved from "prompt-based requests" to "engine-level constraints," and in 2026 the last prompt-era trick (forcing a tool call to get JSON) stopped working on the newest Claude models.

## Table of Contents

- [The JSON Mode Revolution](#the-json-mode-revolution)
- [Function Calling & Tool Use](#function-calling--tool-use)
- [Constrained Decoding (CFG & Regex)](#constrained-decoding-cfg--regex)
- [Multi-Stage Extraction Pattern](#multi-stage-extraction-pattern)
- [Validation & Formatting Errors](#validation--formatting-errors)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The JSON Mode Revolution

Historically, getting JSON was a struggle of "only return JSON, no other text."
**Standard approach**: use the provider's native JSON-schema output.

| Provider | Native structured output |
|----------|--------------------------|
| **OpenAI** | `response_format` with `json_schema` (Chat Completions) or `text.format` (Responses API); `strict: true` on function tools |
| **Anthropic** | `output_config.format` with a JSON schema (the older `output_format` parameter is deprecated); `strict: true` on tool definitions |
| **Google** | A response schema with a JSON response MIME type |

- **Benefit**: 100% syntactical validity. The model literally cannot output a string that is not a valid JSON.
- **Behind the scenes**: The serving engine masks the vocabulary at each step, ensuring only valid JSON characters (e.g., `{`, `"`, `:`, `[`) can be picked next.
- **Two exceptions to check**: output cut off by `max_tokens`, and a safety refusal (Claude returns `stop_reason: "refusal"`). Check the stop reason before parsing.

---

## Function Calling & Tool Use

Function calling is structured generation where the LLM "picks" a function and populates its arguments.

```json
// Example Tool Call
{
  "name": "get_stock_price",
  "arguments": { "symbol": "AAPL", "interval": "1d" }
}
```

**Nuance**: **Parallel Function Calling** is now standard. A model can decide to call 5 different tools simultaneously (e.g., check account balance, check credit score, check loan rates) and aggregate the results.

**The forced-tool trick is retired on the newest Claude models.** For two years the standard way to get JSON from Claude was to define a tool with the target schema and force it with `tool_choice`. Claude Fable 5.1 (September 1, 2026), Opus 5.5 (September 22), and Sonnet 5.5 (September 28) return a 400 for `tool_choice` types `any` and `tool`; `auto` and `none` still work. Haiku 5.5 (October 7) still accepts a forced tool choice, but the response then starts with the tool call and has no thinking block, so structured outputs are the better path there too. Assistant prefill (starting the reply with `{`) already returns a 400 on Claude 4.6 and later models. The replacements:

| Old pattern | Replacement |
|-------------|-------------|
| Force a tool call just to get JSON back | `output_config.format` with a JSON schema |
| Force a specific tool in an agent step | `tool_choice: auto`, a prompt instruction naming the tool, and `strict: true` for schema-valid arguments |
| Prefill `{` to start the JSON | Structured outputs |

Frameworks are rerouting underneath you: langchain-anthropic 1.7.3 (September 22, 2026) sends `with_structured_output` through the JSON-schema path for Fable and Opus 5.5, and 1.7.5 added Sonnet 5.5. If you hard-coded `tool_choice` in your own wrapper, it breaks on upgrade. On the OpenAI side, GPT-6 Astra requires the Responses API for tool calling.

---

## Constrained Decoding (CFG & Regex)

For self-hosted models, we use **Context-Free Grammars (CFG)** or **Regex**. vLLM and SGLang ship grammar backends (XGrammar, llguidance) behind their structured-output options, llama.cpp uses GBNF grammars, and Outlines works directly with Hugging Face models.

```python
# Outlines pattern (1.x API)
import outlines
from outlines.types import Regex
from transformers import AutoModelForCausalLM, AutoTokenizer

MODEL = "Qwen/Qwen3-8B"  # any Hugging Face causal LM
model = outlines.from_transformers(
    AutoModelForCausalLM.from_pretrained(MODEL, device_map="auto"),
    AutoTokenizer.from_pretrained(MODEL),
)
phone = model("Extract the phone number: call me at 415-555-0199", Regex(r"\d{3}-\d{3}-\d{4}"))
# Result: The model can ONLY output telephone numbers.
```

Grammar compilation is not free: a large schema or grammar can add noticeable first-request latency, so cache compiled grammars and keep schemas flat where you can.

---

## Multi-Stage Extraction Pattern

For complex data extraction (e.g., 50 fields from a medical record), don't do it in one pass.
- **Stage 1 (Text-to-Text)**: Extract a "messy" but complete set of facts in natural language.
- **Stage 2 (Text-to-JSON)**: Use a smaller, cheaper model to convert those natural language facts into a strict JSON schema.
- **Benefit**: Reduces "hallucination under pressure": large models struggle when forced to reason AND follow strict syntax simultaneously. On reasoning models this matters less, because the thinking phase is unconstrained and only the final answer is schema-bound.
- **Drop the `reasoning` field on the newest Claude models.** A common trick was to put a `reasoning` string first in the schema so the model "thinks" before filling the answer fields. On Claude Fable 5.1, Opus 5.5, Sonnet 5.5, and Haiku 5.5, a `reasoning`, `thinking`, or `trace` field in JSON output or a tool input is a listed trigger for `reasoning_extraction` refusals. Let native thinking do that work, and if you need a rationale for audit, use a short `explanation` or `evidence` field.

---

## Validation & Formatting Errors

Even with "JSON mode," the **Logic** inside the JSON might be wrong (e.g., a field is missing or a date is in the wrong format).

**Recovery pattern**:
1. Check the stop reason (`max_tokens`, `refusal`) before parsing.
2. Validate output against **Pydantic/Zod**.
3. If it fails, send the **Traceback** back to the model:
   "Error: Field 'age' must be an integer, got 'twenty'. Fix and re-generate."
4. Most models fix the error on the first retry.

Parse tool arguments with a JSON parser, never by string matching: newer Claude models may escape Unicode or forward slashes differently in tool inputs.

---

## Interview Questions

### Q: Why is "JSON Mode" more reliable than prompt-based JSON requests?

**Strong answer:**
Prompt-based requests rely on the model's *willingness* to follow instructions; "JSON Mode" (or Constrained Decoding) relies on the serving engine's *inability* to do anything else. By applying a "Logit Bias" or a "Grammar Mask" at the inference level, the engine restricts the choice of the next token to only those that would be valid according to the schema. This eliminates the "preamble" (e.g., "Sure, here is your JSON...") and ensures that you never get a malformed string from sampling randomness. What it does not guarantee is correct *content*, a complete response if `max_tokens` cuts it off, or any response at all if the model refuses, so validation and stop-reason checks stay in the pipeline.

### Q: What is the risk of asking an LLM for too many structured fields at once?

**Strong answer:**
There is a trade-off between **Schema Complexity** and **Information Integrity**. As the schema grows (e.g., 20+ hierarchical fields), the model's attention is consumed by maintaining the JSON structure (brackets, keys, quotes) rather than verifying the accuracy of the data. This often leads to "Omission Hallucinations" where the model skips fields or fills them with placeholder data. The mitigation is to split the extraction into smaller parallel sub-schemas, or to extract in natural language first and structure in a second pass, and to make "unknown" an explicit allowed value so the model is not pushed to invent one.

### Q: Your extraction pipeline forced a tool call to get JSON from Claude, and after moving to Opus 5.5 every request returns a 400. What do you change?

**Strong answer:**
Opus 5.5, Sonnet 5.5, and Fable 5.1 reject `tool_choice` `any` and `tool`, so the forced-tool trick is gone. If the tool existed only to carry a schema, I replace it with native structured outputs (`output_config.format` with the same JSON schema), which is the cleaner contract anyway. If the step genuinely needs the model to call a specific tool, I switch to `tool_choice: auto`, name the tool in the instruction, and set `strict: true` so the arguments are schema-valid when it does call; I then measure how often it skips the call and add a retry for that case. I would also grep the codebase and framework versions for other hard-coded assumptions from the same era (sampling parameters, assistant prefill, `budget_tokens`, a `reasoning` field in the schema), because they break or get refused on the same models.

---

## References
- OpenAI. "Structured Outputs Documentation" (August 2024 update)
- [Anthropic. Strict tool use and structured outputs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use)
- [Claude API release notes (forced tool use removed)](https://platform.claude.com/docs/en/release-notes/overview)
- [Anthropic. "Refusals and fallback": keep reasoning in thinking blocks](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#keep-reasoning-in-thinking-blocks)
- Outlines Project. "Output types: Regex, CFG, JSON" (Outlines 1.x documentation)
- Willard and Louf. "Efficient Guided Generation for Large Language Models" (2023)
- Dong et al. "XGrammar: Flexible and Efficient Structured Generation Engine for Large Language Models" (2024)

---

*Next: [Prompt Optimization (DSPy)](07-prompt-optimization-dspy.md)*
