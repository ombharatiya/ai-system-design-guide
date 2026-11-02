# Tokenization Deep Dive

Tokenization is the process of converting text into discrete units (tokens) that models can process. It directly impacts model capabilities, costs, and performance.

## Table of Contents

- [Why Tokenization Matters](#why-tokenization-matters)
- [Tokenization Algorithms](#tokenization-algorithms)
- [Vocabulary Design Tradeoffs](#vocabulary-design-tradeoffs)
- [Special Tokens](#special-tokens)
- [Multilingual Tokenization](#multilingual-tokenization)
- [Multimodal Tokenization](#multimodal-tokenization-pixels-to-tokens)
- [Token Counting for Cost Estimation](#token-counting-for-cost-estimation)
- [Common Tokenization Issues](#common-tokenization-issues)
- [Practical Tokenization Patterns](#practical-tokenization-patterns)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Why Tokenization Matters

### For System Design

1. **Cost**: LLM APIs charge per token. Tokenization efficiency directly affects costs.
2. **Context limits**: Token count, not word count, determines what fits in context.
3. **Capability**: Some tasks (character counting, anagrams) are hard because of tokenization.
4. **Consistency**: Same text tokenizes differently across models.

### For Understanding LLM Behavior

**Classic interview question**: Why does GPT struggle to count letters in "strawberry"?

Because "strawberry" is tokenized as multiple subwords. The model never sees individual characters; it sees subword units. Counting letters requires reasoning about internal structure of tokens.

---

## Tokenization Algorithms

### Byte Pair Encoding (BPE)

The most common algorithm. Used by GPT-series, Llama, Qwen, DeepSeek.

**Training algorithm:**
1. Start with vocabulary of individual bytes (256 tokens)
2. Count all adjacent token pairs in training corpus
3. Merge most frequent pair into a new token
4. Repeat until vocabulary size reached

**Example:**
```
Corpus: "low lower lowest"
Initial: ['l', 'o', 'w', ' ', 'l', 'o', 'w', 'e', 'r', ' ', 'l', 'o', 'w', 'e', 's', 't']

Step 1: Most frequent pair is ('l', 'o'). Merge to 'lo'.
['lo', 'w', ' ', 'lo', 'w', 'e', 'r', ' ', 'lo', 'w', 'e', 's', 't']

Step 2: Most frequent pair is ('lo', 'w'). Merge to 'low'.
['low', ' ', 'low', 'e', 'r', ' ', 'low', 'e', 's', 't']

Step 3: Most frequent pair is ('low', 'e'). Merge to 'lowe'.
['low', ' ', 'lowe', 'r', ' ', 'lowe', 's', 't']

Continue until vocabulary size target...
```

**Properties:**
- Deterministic tokenization given trained vocabulary
- Common words tend to be single tokens
- Rare words split into subwords

### WordPiece

Used by BERT-family models.

**Key difference from BPE:**
- BPE: Merge based on frequency
- WordPiece: Merge based on likelihood improvement

```
Score = freq(AB) / (freq(A) * freq(B))
```

This favors merges that are more meaningful than random co-occurrence.

**Visual marker:** WordPiece uses ## prefix for continuation tokens:
```
"embedding" becomes ["em", "##bed", "##ding"]
```

### Unigram (SentencePiece)

Used by T5, ALBERT, some multilingual models.

**Training algorithm:**
1. Start with large candidate vocabulary
2. Compute loss if each token were removed
3. Remove tokens that increase loss least
4. Repeat until vocabulary size reached

**Key difference:** Works with probabilities rather than frequencies. Can recover from suboptimal early merges.

### Comparison

| Algorithm | Merge Criterion | Tokenization | Used By |
|-----------|-----------------|--------------|---------|
| BPE | Frequency | Deterministic | GPT, Llama, Qwen |
| WordPiece | Likelihood | Deterministic | BERT, DistilBERT |
| Unigram | Probability | Probabilistic | T5, mT5, XLNet |

---

## Vocabulary Design Tradeoffs

### Vocabulary Size

| Size | Example | Pros | Cons |
|------|---------|------|------|
| Small (10K) | Some early models | Smaller embeddings | Long token sequences |
| Medium (32K) | Llama 2, Mistral 7B | Good balance | Multilingual inefficiency |
| Large (100K-152K) | GPT-4 (cl100k), Llama 3 (128K), DeepSeek V3 and V4.1 (~129K), Mistral Tekken (~131K), Qwen3 (~152K) | **Common default for open models.** High compression ratio. | Larger embeddings table |
| Huge (200K+) | GPT-4o through GPT-5.x (o200k, ~200K), Llama 4 (~202K), Gemma 3 (~262K) | Multimodal and multilingual efficiency | Memory pressure at the LM Head |

Anthropic and Google do not publish vocabulary sizes for Claude and Gemini, and tiktoken covers only OpenAI's published encodings (o200k_base for GPT-4o through GPT-5.x). For any model whose tokenizer you cannot load and confirm, count with the provider's token-counting API or the response `usage` field, not with a proxy tokenizer.

**The vocab-expansion deep dive:**
- **Llama 3 (128K)**: Meta combined 100K tokens from tiktoken with 28K extra tokens aimed at non-English languages, and reported up to 15% fewer tokens than Llama 2 for the same text. Llama 4 went to ~202K.
- **GPT-4o through GPT-5.x (o200k_base)**: Doubling cl100k's vocabulary improved compression for code and non-English text, which lowers cost per unit of meaning even at the same per-token price.
- **Same price, different bill**: a tokenizer change moves your bill without any list-price change. Anthropic's docs say Claude 4.7 and later models use a newer tokenizer that produces roughly 30% more tokens than earlier Claude models for the same text, with the exact increase depending on content. Claude Haiku 5.5 brought it to the small tier: about 30% more tokens than Haiku 4.5, so its list price, a tenth of Haiku 4.5's, is a smaller cut per unit of text than the rate card says. Re-baseline token counts whenever you switch model families.

### Character vs Subword vs Word

| Granularity | Example | Tokens for "running" | Tradeoffs |
|-------------|---------|---------------------|-----------|
| Character | ByT5 | ['r','u','n','n','i','n','g'] | Handles any text but very long sequences |
| Subword | GPT | ['running'] or ['run','ning'] | Good balance |
| Word | Early NLP | ['running'] | Short sequences but cannot handle OOV |

Modern LLMs universally use subword tokenization for the balance of vocabulary size and sequence length.

### Byte-Level BPE

GPT-2 introduced byte-level BPE:
- Base vocabulary is 256 bytes, not characters
- Can represent any text without UNK tokens
- Unicode handled naturally as byte sequences

```python
# Character-level: Needs explicit handling of characters
text = "cafe"  # Unknown character might become [UNK]

# Byte-level: Works with any text (no UNK needed)
text = "cafe"  # Becomes bytes, then BPE operates on bytes
```

---

## Special Tokens

Special tokens handle structural information outside normal text:

| Token | Purpose | Example |
|-------|---------|---------|
| BOS | Beginning of sequence | Signals start of generation |
| EOS | End of sequence | Signals completion |
| PAD | Padding | Fill batches to equal length |
| UNK | Unknown token | Fallback for OOV (rare with byte BPE) |
| SEP | Separator | Divide segments (BERT-style) |

### Chat Templates

Modern chat models use special tokens for conversation structure:

**Llama 2 format:**
```
[INST] <<SYS>>
You are a helpful assistant.
<</SYS>>

User message here [/INST] Assistant response here
```

**ChatML (OpenAI style):**
```
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
Hello!<|im_end|>
<|im_start|>assistant
Hi there!<|im_end|>
```

**Why this matters:**
- Wrong formatting leads to poor results
- Special tokens are not in pre-training data
- Libraries like transformers use chat_template for automatic formatting

---

## Multilingual Tokenization

### The Challenge

Tokenizers trained primarily on English have poor efficiency for other languages:

| Language | Tokens for "Hello" | Tokens for equivalent greeting |
|----------|-------------------|-------------------------------|
| English | 1 ("Hello") | - |
| Chinese | - | 2-3+ for equivalent |
| Japanese | - | 3-5+ for equivalent |
| Korean | - | 2-4+ for equivalent |

**Cost implication:** On English-centric tokenizers, non-English users pay 2-3x more per semantic unit. Larger modern vocabularies narrow the gap for major languages but do not close it.

### Solutions

1. **Multilingual training corpus:** Train tokenizer on balanced multilingual data
2. **Larger vocabulary:** More room for non-English tokens
3. **Language-specific tokenizers:** Separate tokenizers per language family

**Models with good multilingual support:**
- mT5, XLM-R: Trained on 100+ languages
- Current GPT, Claude and Qwen models: large vocabularies with broad multilingual coverage
- Gemini: Designed for multilingual from the start

Approximate token multiplier versus equivalent English text (illustrative; measure on your own corpus, since ratios vary by domain and script):

| Tokenizer | Chinese | Japanese | Korean | Hindi |
|-------|---------|----------|--------|--------|
| GPT-2 | 2.5x | 3.0x | 2.8x | 6.0x |
| GPT-4 (cl100k) | 1.4x | 1.6x | 1.5x | 3.2x |
| o200k (GPT-4o through GPT-5.x) | 1.1x | 1.2x | 1.1x | 1.4x |
| Llama 3 (128K) | 1.2x | 1.3x | 1.2x | 1.5x |

---

## Multimodal Tokenization (pixels-to-tokens)

Modern native multimodal models do not just "see" images; they tokenize them.

### Image Tokenization (Vision Transformers)
Images are split into patches (e.g., 14x14 pixels). Each patch is passed through a vision encoder (like SigLIP) to produce a single visual token.
- **Fixed Token Cost**: Most models use a fixed number of tokens per image at a specific resolution (e.g., 256 or 729 tokens per image).
- **Dynamic Resolution**: Some models (Gemini 3) use a variable number of tokens depending on image aspect ratio and detail level.
- **Encoder-free**: Gemma 4 12B "Unified" (June 2026) skips the vision encoder and projects raw image patches and audio waveforms into the LLM embedding space with lightweight linear layers. Token cost then follows patch count directly.

### Audio/Video Tokenization
- **Audio**: Compressed into discrete units using codecs like EnCodec, then represented as a sequence of audio tokens.
- **Video**: Treated as a sequence of image frames (temporal tokenization). A 1-second video @ 1FPS might cost as much as 1 high-res image.
- **Agentic video processing**: Since September 1, 2026 the Gemini API offers an opt-in `"processing": "agentic"` mode on Gemini 3.8 Flash, 3.6 Flash and 3.5 Flash-Lite (requests to `gemini-3.7-flash`, also supported at launch, have routed to 3.8 Flash since October 8). The model decides which segments to inspect, at what speed and through which modality (frames, audio or transcript) instead of sampling at a fixed rate. Google reports up to 88% fewer tokens and up to 66% lower cost on standard video benchmarks, billed at normal token prices. Static 1 FPS sampling remains the default, so a cost model built on fixed-rate sampling can overestimate agentic-mode tokens by up to roughly 8x.

---

## Token Counting for Cost Estimation

### Quick Estimation Rules

For English text:
- **Words to tokens:** ~1.3 tokens per word
- **Characters to tokens:** ~4 characters per token
- **Pages to tokens:** ~500-800 tokens per page

```python
def estimate_tokens(text: str) -> int:
    # Rough estimation for English
    word_count = len(text.split())
    return int(word_count * 1.3)
```

### Accurate Counting

Use the model-specific tokenizer, or the provider's counting endpoint when the tokenizer is not public:

```python
import tiktoken

# For OpenAI models (tiktoken ships OpenAI encodings only; o200k_base
# covers GPT-4o through GPT-5.x)
encoding = tiktoken.get_encoding("o200k_base")
tokens = encoding.encode("Your text here")
token_count = len(tokens)

# For open-weight models, use the model's own Hugging Face tokenizer
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B-Instruct")
tokens = tokenizer.encode("Your text here")
token_count = len(tokens)

# For Claude, the tokenizer is not public: call the count_tokens endpoint
import anthropic
client = anthropic.Anthropic()
count = client.messages.count_tokens(
    model="claude-sonnet-5-5",
    messages=[{"role": "user", "content": "Your text here"}],
)
token_count = count.input_tokens
```

Gemini has an equivalent `countTokens` call, and OpenAI's Responses API has an input-token counting endpoint for models tiktoken does not map. Never estimate Claude or Gemini tokens with tiktoken: different vocabularies give different counts for the same text.

### Reasoning Tokens

With reasoning models, the visible answer is only part of the output bill. Thinking or reasoning tokens are billed as output on OpenAI, Anthropic and Google models even when the raw reasoning is not returned, and their volume depends on the effort or thinking level you set. Budget from the API's `usage` field on real traffic, not from the length of the visible response.

### Cost Calculation

```python
# USD per 1M tokens, standard tier list prices as of November 2026.
# Excludes cache reads/writes, batch discounts and long-context surcharges
# (OpenAI bills the whole request at higher rates above 272K input tokens,
# Claude Haiku 5.5 at 5x once the prompt passes 100K tokens).
PRICING = {
    "gpt-6-sol": {"input": 2.00, "output": 10.00},
    "gpt-6-luna": {"input": 0.10, "output": 0.50},
    "claude-sonnet-5-5": {"input": 2.00, "output": 10.00},
    "claude-haiku-5-5": {"input": 0.10, "output": 0.50},
}

def calculate_cost(input_tokens: int, output_tokens: int, model: str) -> float:
    """Token counts come from the response's usage field (output includes
    reasoning tokens), not from re-tokenizing text with a different tokenizer."""
    rates = PRICING[model]
    return (
        (input_tokens / 1_000_000) * rates["input"] +
        (output_tokens / 1_000_000) * rates["output"]
    )
```

---

## Common Tokenization Issues

### Issue 1: Token Boundary Misalignment

**Problem:** Text operations may not align with token boundaries.

```python
text = "Hello world"
# Tokens: ["Hello", " world"]  # Note: space is part of second token

# Truncating at character 6 ("Hello ") splits a token
```

**Solution:** Always truncate at token boundaries when managing context.

### Issue 2: Inconsistent Tokenization

**Problem:** Same text tokenizes differently based on context.

```python
# GPT tokenizer example
"New York"     # Might be ["New", " York"]
"NewYork"      # Might be ["New", "York"]
" New York"    # Might be [" New", " York"]
```

**Implication:** Token counts can vary based on surrounding text. Always tokenize the full context.

### Issue 3: Code and Structured Data

**Problem:** Code and JSON often tokenize inefficiently.

```python
# Python code often tokenizes poorly
"def calculate_average(numbers):"
# Becomes many tokens: ["def", " calculate", "_", "average", "(", "numbers", "):", ...]

# JSON keys tokenize individually
'{"firstName": "John"}'
# Many tokens for structure
```

**Mitigation:** 
- Some models have code-optimized tokenizers
- Consider compressing JSON before sending
- Use structured output modes when available

### Issue 4: Whitespace Handling

**Problem:** Tokenizers handle whitespace differently.

```python
# Leading spaces often become separate tokens
" Hello"  # [" ", "Hello"] or [" Hello"]

# Multiple spaces may merge or stay separate
"Hello  world"  # Behavior varies by tokenizer
```

**Best practice:** Normalize whitespace before tokenizing.

---

## Practical Tokenization Patterns

### Pattern 1: Context Window Management

```python
def fit_to_context(
    system_prompt: str,
    user_message: str,
    history: list[str],
    max_tokens: int = 8000,
    reserve_for_output: int = 2000
) -> str:
    # Use the target model's tokenizer; o200k_base fits OpenAI models only
    encoding = tiktoken.get_encoding("o200k_base")
    
    available = max_tokens - reserve_for_output
    
    # System prompt always included
    tokens_used = len(encoding.encode(system_prompt))
    available -= tokens_used
    
    # User message always included
    tokens_used = len(encoding.encode(user_message))
    available -= tokens_used
    
    # Add history from most recent, drop oldest if needed
    included_history = []
    for msg in reversed(history):
        msg_tokens = len(encoding.encode(msg))
        if msg_tokens <= available:
            included_history.insert(0, msg)
            available -= msg_tokens
        else:
            break
    
    return format_prompt(system_prompt, included_history, user_message)
```

### Pattern 2: Chunking at Token Boundaries

```python
def chunk_at_token_boundaries(
    text: str,
    chunk_size: int = 500,
    overlap: int = 50
) -> list[str]:
    encoding = tiktoken.get_encoding("o200k_base")
    tokens = encoding.encode(text)
    
    chunks = []
    start = 0
    while start < len(tokens):
        end = min(start + chunk_size, len(tokens))
        chunk_tokens = tokens[start:end]
        chunk_text = encoding.decode(chunk_tokens)
        chunks.append(chunk_text)
        start = end - overlap
    
    return chunks
```

### Pattern 3: Token Budget Allocation

```python
class TokenBudget:
    def __init__(self, total: int):
        self.total = total
        self.allocated = {}
    
    def allocate(self, component: str, tokens: int) -> bool:
        used = sum(self.allocated.values())
        if used + tokens > self.total:
            return False
        self.allocated[component] = tokens
        return True
    
    def remaining(self) -> int:
        return self.total - sum(self.allocated.values())

# Usage
budget = TokenBudget(total=8000)
budget.allocate("system_prompt", 500)
budget.allocate("retrieved_context", 2000)
budget.allocate("user_message", 200)
budget.allocate("output_reserve", 2000)
# Remaining: 3300 tokens for conversation history
```

---

## Interview Questions

### Q: Why do LLMs struggle with simple character counting?

**Strong answer:**
Tokenization converts text to subword units, not characters. When asked "How many 'r's in strawberry?", the model sees tokens like ["str", "aw", "berry"] rather than individual letters.

The model has to reason about the internal structure of tokens it does not directly observe. This requires memorizing or computing character compositions of tokens, which is an emergent capability that is not always reliable.

The solution is to prompt the model to spell out the word character by character first, then count. This forces the creation of character-level tokens.

### Q: How would you estimate token count for cost planning?

**Strong answer:**
For rough estimation: multiply word count by 1.3 for English text.

For accurate counting: Use the model-specific tokenizer.
- OpenAI: tiktoken for the encodings it ships (o200k_base for GPT-4o through GPT-5.x); for a model tiktoken does not map, the Responses API input-token counting endpoint or the `usage` field
- Open-weight models: the model's own Hugging Face tokenizer
- Claude and Gemini: the provider's token-counting endpoint (tokenizers are not public)

Important considerations:
- Non-English text uses more tokens: roughly 1.1-1.5x for major languages on modern 128K-200K vocabularies, 3x or more on older tokenizers and less-covered scripts
- Code and structured data tokenize inefficiently
- Always budget extra for output tokens (priced 3-6x input; 5x on most current frontier models), including reasoning tokens, which are billed as output and scale with the effort level
- Include system prompts, tool definitions and formatting tokens
- The same text costs a different number of tokens on each vendor, so compare cost per completed task, not price per token

For production cost estimation, I sample real requests and measure actual token usage from the API's `usage` field, then apply safety margins. I re-run the measurement on every model migration, because a tokenizer change (Claude 4.7 and later count roughly 30% more tokens for the same text than earlier Claude models, per Anthropic) or a new default effort level moves the bill without any price change.

### Q: What happens when switching tokenizers between models?

**Strong answer:**
Every model family has its own tokenizer. You cannot reuse tokens across models because:

1. **Vocabulary differs:** Token IDs mean different strings
2. **Merge rules differ:** Same text splits differently
3. **Special tokens differ:** Chat formatting varies

Practical implications:
- Always use the correct tokenizer for token counting
- Cached embeddings are model-specific
- Prompt templates need per-model adjustment
- Fine-tuned models inherit their base tokenizer

### Q: How do you handle tokenization for RAG chunking?

**Strong answer:**
Key considerations:

1. **Chunk at token boundaries:** Splitting mid-token corrupts text when decoded
2. **Account for template tokens:** System prompt, formatting consume tokens
3. **Leave headroom:** Retrieved chunks plus question must fit context

Implementation approach:
```python
# Determine available tokens for chunks
available = max_context - system_prompt_tokens - question_tokens - output_reserve

# Chunk with overlap at token boundaries
chunks = chunk_at_token_boundaries(document, chunk_size=500, overlap=50)

# Select chunks until budget exhausted
selected = []
tokens_used = 0
for chunk in ranked_chunks:
    chunk_tokens = count_tokens(chunk)
    if tokens_used + chunk_tokens <= available:
        selected.append(chunk)
        tokens_used += chunk_tokens
```

---

## References

- Sennrich et al. "Neural Machine Translation of Rare Words with Subword Units" (BPE, 2016)
- Wu et al. "Google's Neural Machine Translation System" (WordPiece, 2016)
- Kudo and Richardson "SentencePiece: A simple and language independent subword tokenizer" (2018)
- OpenAI tiktoken library: https://github.com/openai/tiktoken
- Anthropic token counting: https://platform.claude.com/docs/en/build-with-claude/token-counting
- HuggingFace tokenizers: https://github.com/huggingface/tokenizers

---

*Previous: [LLM Internals](01-llm-internals.md) | Next: [Attention Mechanisms](03-attention-mechanisms.md)*
