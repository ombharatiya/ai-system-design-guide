# Tokenization Deep Dive

Tokenization is the process of converting text into discrete units (tokens) that models can process. It directly impacts model capabilities, costs, and performance.

## Table of Contents

- [Why Tokenization Matters](#why-tokenization-matters)
- [Tokenization Algorithms](#tokenization-algorithms)
- [Vocabulary Design Tradeoffs](#vocabulary-design-tradeoffs)
- [Special Tokens](#special-tokens)
- [Multilingual Tokenization](#multilingual-tokenization)
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

The most common algorithm. Used by GPT-series, Llama, Claude.

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
| BPE | Frequency | Deterministic | GPT, Llama, Claude |
| WordPiece | Likelihood | Deterministic | BERT, DistilBERT |
| Unigram | Probability | Probabilistic | T5, mT5, XLNet |

---

## Vocabulary Design Tradeoffs

### Vocabulary Size

| Size | Example | Pros | Cons |
|------|---------|------|------|
| Small (10K) | Some early models | Smaller embeddings | Long token sequences |
| Medium (32K) | Llama 2 | Good balance | Multilingual inefficiency |
| Large (128K) | Llama 3/4, Claude Sonnet 4.6, Mistral Medium 3.5 | **Current standard.** High compression ratio. | Larger embeddings table |
| Huge (200K+) | GPT-5.5 (o200k), Claude Opus 4.7 | Native multimodal and multilingual efficiency | Memory pressure at the LM Head |

**The vocab-expansion deep dive:**
- **Llama 3/4 (128k)**: By moving from 32k to 128k, Meta improved English compression by ~15% and non-English languages like Hindi by 3-4x. 
- **GPT-4o/5.2 (o200k_base)**: Tiktoken's latest encoding provides superior compression for code and multilingual text, reducing API costs indirectly by using fewer tokens for the same meaning.

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

**Cost implication:** Non-English users pay 2-3x more per semantic unit.

### Solutions

1. **Multilingual training corpus:** Train tokenizer on balanced multilingual data
2. **Larger vocabulary:** More room for non-English tokens
3. **Language-specific tokenizers:** Separate tokenizers per language family

**Models with good multilingual support:**
- mT5, XLM-R: Trained on 100+ languages
- GPT-4, Claude 3.5: Large vocabulary with multilingual coverage
- Gemini: Designed for multilingual from the start

| Model | Chinese | Japanese | Korean | Hindi |
|-------|---------|----------|--------|--------|
| GPT-2 | 2.5x | 3.0x | 2.8x | 6.0x |
| GPT-4 (cl100k) | 1.4x | 1.6x | 1.5x | 3.2x |
| GPT-5.2 (o200k) | 1.1x | 1.2x | 1.1x | 1.4x |
| Llama 3/4 (128k)| 1.2x | 1.3x | 1.2x | 1.5x |

---

## Multimodal Tokenization (pixels-to-tokens)

Modern native multimodal models do not just "see" images; they tokenize them.

### Image Tokenization (Vision Transformers)
Images are split into patches (e.g., 14x14 pixels). Each patch is passed through a vision encoder (like SigLIP) to produce a single visual token.
- **Fixed Token Cost**: Most models use a fixed number of tokens per image at a specific resolution (e.g., 256 or 729 tokens per image).
- **Dynamic Resolution**: Some models (Gemini 3) use a variable number of tokens depending on image aspect ratio and detail level.

### Audio/Video Tokenization
- **Audio**: Compressed into discrete units using codecs like EnCodec, then represented as a sequence of audio tokens.
- **Video**: Treated as a sequence of image frames (temporal tokenization). A 1-second video @ 1FPS might cost as much as 1 high-res image.

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

Use the model-specific tokenizer:

```python
import tiktoken

# For OpenAI models
encoding = tiktoken.encoding_for_model("gpt-4")
tokens = encoding.encode("Your text here")
token_count = len(tokens)

# For Llama/Anthropic, use transformers
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-2-7b")
tokens = tokenizer.encode("Your text here")
token_count = len(tokens)
```

### Cost Calculation

```python
def calculate_cost(input_text: str, output_text: str, model: str) -> float:
    pricing = {
        "gpt-4o": {"input": 2.50, "output": 10.00},  # per 1M tokens
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},
        "claude-3.5-sonnet": {"input": 3.00, "output": 15.00},
    }
    
    encoding = tiktoken.encoding_for_model(model)
    input_tokens = len(encoding.encode(input_text))
    output_tokens = len(encoding.encode(output_text))
    
    cost = (
        (input_tokens / 1_000_000) * pricing[model]["input"] +
        (output_tokens / 1_000_000) * pricing[model]["output"]
    )
    return cost
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
    encoding = tiktoken.encoding_for_model("gpt-4")
    
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
    encoding = tiktoken.encoding_for_model("gpt-4")
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

### Q: Why does GPT-4 struggle with simple character counting?

**Strong answer:**
Tokenization converts text to subword units, not characters. When asked "How many 'r's in strawberry?", the model sees tokens like ["str", "aw", "berry"] rather than individual letters.

The model has to reason about the internal structure of tokens it does not directly observe. This requires memorizing or computing character compositions of tokens, which is an emergent capability that is not always reliable.

The solution is to prompt the model to spell out the word character by character first, then count. This forces the creation of character-level tokens.

### Q: How would you estimate token count for cost planning?

**Strong answer:**
For rough estimation: multiply word count by 1.3 for English text.

For accurate counting: Use the model-specific tokenizer.
- OpenAI: tiktoken library
- Others: transformers AutoTokenizer

Important considerations:
- Non-English text uses 1.5-3x more tokens
- Code and structured data tokenize inefficiently
- Always budget extra for output tokens (typically priced higher)
- Include system prompts and formatting tokens

For production cost estimation, I sample real requests and measure actual token usage, then apply safety margins.

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
- HuggingFace tokenizers: https://github.com/huggingface/tokenizers

---

*Previous: [LLM Internals](01-llm-internals.md) | Next: [Attention Mechanisms](03-attention-mechanisms.md)*
