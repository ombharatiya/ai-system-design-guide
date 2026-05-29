# Chunking Strategies

Chunking is the process of splitting a document into discrete segments for retrieval. Production pipelines have moved beyond blind fixed-size splits to **structure-aware and semantic segments**, with newer techniques like late chunking and contextual prepending now in the mainstream toolkit.

## Table of Contents

- [The Retrieval-Context Tension](#tension)
- [Recursive Structure Splitting](#recursive)
- [Semantic Chunking](#semantic)
- [Hierarchical (Parent-Child) Chunking](#hierarchical)
- [Content-Specific Strategies (Code, PDF, Tables)](#content-specific)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## The Retrieval-Context Tension

| Aspect | Small Chunks (100t) | Large Chunks (1000t) |
|--------|---------------------|----------------------|
| **Precision** | High (Exact match) | Low (Diluted) |
| **Context** | Poor (Broken sentences) | Rich (Surrounding info) |
| **Storage** | High (More vectors) | Low (Fewer vectors) |
| **Latency** | Low (Fast search) | High (Heavy retrieval) |

**Rule**: Smaller is better for *finding*, but larger is better for *thinking*. Use **Hierarchical Chunking** to get both.

---

## Recursive Structure Splitting

Instead of splitting at every 500 characters, we split at logical boundaries:
`[Double Newline] > [Single Newline] > [Period] > [Space]`.

**Best practice**: Use **Markdown-Aware Splitting**. If a document has `#` headers, ensure the header is prepended to *every* child chunk to preserve context (Contextual Chunking).

---

## Semantic Chunking

Semantic chunking uses an embedding model to detect "topic shifts."

1. Split text into individual sentences.
2. Group sentences as long as their embedding similarity stays above a threshold (e.g., 0.82).
3. If similarity drops, start a new chunk.

**Nuance**: Production pipelines increasingly use **Cross-Encoder Segmenters**. A tiny model scans the text and predicts a "Separator token" at every semantic break. This is 10x more accurate than cosine-similarity thresholding.

---

## Hierarchical (Parent-Child) Chunking

This is the industry standard for production RAG.

- **Process**: 
  1. Create "Parent" chunks of 1,500 tokens.
  2. Sub-divide each parent into 5 "Child" chunks of 300 tokens.
  3. **Index only the children**.
  4. At retrieval, if a child matches, **return the full parent context** to the LLM.
- **Why?**: The child is small and easy for the vector DB to match. The parent provides enough context for the LLM to actually reason correctly without "Broken Context" hallucinations.

---

## Content-Specific Strategies

### 1. Code Chunking
- **Strategy**: Use AST (Abstract Syntax Tree) parsing.
- **Rule**: Never split a function mid-body. Keep imports and class declarations with their methods.

### 2. Table Chunking
- **Strategy**: Use Markdown formatting for tables.
- **Modern pattern**: "Summarized Tables." Store a natural language summary of the table in the vector DB, but return the full Markdown table to the LLM.

### 3. PDF/Layout Chunking
- **Strategy**: Use **Vision-Language Model (VLM)** pre-processing (e.g., ColPali).
- **Nuance**: Instead of just text, store embeddings that represent the *positional layout* of the page, ensuring charts and sidebars don't get mixed into body text.

---

## Interview Questions

### Q: Why is fixed-size chunking with overlap problematic for production systems?

**Strong answer:**
Fixed-size chunking is "content-blind." It frequently splits sentences mid-thought, breaks mathematical equations, and separates headers from their descriptive text. While "Overlap" (e.g., 10%) mitigates this by duplicating 10% of text across chunks, it doesn't solve the core issue: the model's attention is forced to reconstruct meaning from fragmented strings. Modern pipelines prefer **Semantic or Logical Chunking** because it ensures each vector represents a "Complete Semantic Unit," leading to significantly higher retrieval precision.

### Q: What is "Contextual Retrieval" (the Anthropic pattern)?

**Strong answer:**
Contextual Retrieval involves prepending a 1-sentence global context to every chunk before embedding it. For example, if a chunk is about "battery life," but it's from a manual for a "2025 Model X Drone," the text `[Drone_Model_X_Manual]:` is added to the chunk. This ensures that the vector for "battery life" is influenced by the "Drone" context, preventing it from being accidentally retrieved for "phone battery" queries.

---

## References
- Anthropic. "Contextual Retrieval: Improving RAG Accuracy" (2024)
- LlamaIndex. "Advanced Chunking Strategies for RAG" (2025)
- LangChain. "RecursiveCharacterTextSplitter Benchmarks" (2024)

---

*Next: [Embedding Models](03-embedding-models.md)*
