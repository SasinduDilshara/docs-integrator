---
sidebar_position: 4
title: "Choose Chunking & Embedding Strategies"
description: "Select the right chunking method and embedding model for your RAG integration."
---

# Choose Chunking & Embedding Strategies

The quality of your RAG integration depends heavily on two decisions: how you split documents into chunks and which embedding model you use. Get these right, and your retrieval returns precise, relevant context. Get them wrong, and even the best LLM produces poor answers.

This guide gives you practical guidance to make both choices confidently.

## Chunking Methods

WSO2 Integrator supports several chunking strategies. Each suits different content types and retrieval goals.

### Fixed-Size Chunking

Split text into chunks of a fixed token or character count. Simple and predictable.

```ballerina
import ballerinax/ai;

ai:TextSplitter splitter = new ai:FixedSizeSplitter({
    chunkSize: 500,       // characters per chunk
    chunkOverlap: 50      // overlap between consecutive chunks
});

ai:Chunk[] chunks = splitter.split(document);
```

**When to use:** Uniform content like log entries, FAQs, or structured data where each section is roughly the same length.

**Trade-off:** May cut sentences or paragraphs mid-thought, which reduces retrieval quality for narrative content.

### Recursive Character Splitting

Split on natural boundaries (paragraphs, sentences, words) before falling back to character limits. This is the most commonly used strategy and the recommended default.

```ballerina
ai:TextSplitter splitter = new ai:RecursiveCharacterSplitter({
    chunkSize: 500,
    chunkOverlap: 50,
    separators: ["\n\n", "\n", ". ", " "]
});
```

The splitter tries each separator in order. It first attempts to split on double newlines (paragraph breaks), then single newlines, then sentence boundaries, and finally spaces.

**When to use:** Most document types — technical docs, articles, reports, manuals.

### Sentence-Based Chunking

Group a fixed number of sentences into each chunk. Preserves complete thoughts better than character-based splitting.

```ballerina
ai:TextSplitter splitter = new ai:SentenceSplitter({
    sentencesPerChunk: 5,
    chunkOverlap: 1       // overlap by 1 sentence
});
```

**When to use:** Conversational content, customer support transcripts, or any text where sentence boundaries carry meaning.

### Semantic Chunking

Use an embedding model to detect topic shifts and split at semantic boundaries. Produces the highest-quality chunks but costs more due to embedding calls during chunking.

```ballerina
ai:TextSplitter splitter = new ai:SemanticSplitter({
    embeddingProvider: embedder,
    breakpointThreshold: 0.3,    // similarity drop that triggers a split
    minChunkSize: 100,
    maxChunkSize: 1000
});
```

**When to use:** Long-form content with distinct sections that are not separated by consistent formatting (e.g., transcripts, legal documents).

:::tip
Start with recursive character splitting at 500 characters with 50-character overlap. Only move to semantic chunking if retrieval quality is not meeting your needs — it adds latency and embedding costs to the ingestion step.
:::

## Overlap Strategies

Overlap ensures that context at chunk boundaries is not lost. When a user's question relates to content that spans two chunks, overlap increases the chance that at least one chunk contains the full relevant passage.

| Overlap | Effect |
|---------|--------|
| **0%** | No redundancy, smallest storage footprint, risk of missing boundary context |
| **10%** (recommended) | Good balance between coverage and storage efficiency |
| **20-30%** | High coverage, useful for dense technical content, increases storage |

```ballerina
// 10% overlap for a 500-character chunk
ai:TextSplitter splitter = new ai:RecursiveCharacterSplitter({
    chunkSize: 500,
    chunkOverlap: 50   // 10% of chunkSize
});
```

## Choosing an Embedding Model

The embedding model converts text into vectors. Your choice affects retrieval accuracy, speed, and cost.

| Model | Dimensions | Relative quality | Speed | Cost |
|-------|-----------|-----------------|-------|------|
| OpenAI `text-embedding-ada-002` | 1536 | Good | Fast | Low |
| OpenAI `text-embedding-3-small` | 1536 | Better | Fast | Low |
| OpenAI `text-embedding-3-large` | 3072 | Best | Moderate | Higher |
| Cohere `embed-english-v3.0` | 1024 | Good | Fast | Low |
| Azure OpenAI Embeddings | 1536 | Good | Fast | Varies |

### Configure the Embedding Provider

```ballerina
import ballerinax/ai;

configurable string openaiApiKey = ?;

// OpenAI embeddings (default, recommended)
final ai:EmbeddingProvider embedder = check new ai:OpenAIEmbedder({
    apiKey: openaiApiKey,
    model: "text-embedding-3-small"
});
```

:::info
For LLM connection configuration details, including API key management and endpoint setup, see the [Connectors](/docs/connectors/ai-llms) section.
:::

## Dimensionality Trade-offs

Higher-dimensional embeddings capture more nuance but cost more to store and search:

- **1536 dimensions** — The sweet spot for most integrations. Balances quality with performance and storage costs.
- **3072 dimensions** — Use when retrieval precision is critical and you can tolerate higher storage and slightly slower search.
- **1024 dimensions** — Lighter weight, faster search, slightly lower quality. Good for large-scale use cases where speed matters more than precision.

:::warning
Your embedding dimensions must match your vector store configuration. If you switch embedding models, you need to re-index all documents with the new model — you cannot mix embeddings from different models in the same vector store.
:::

## Practical Tuning Workflow

Follow this process to find the right chunking and embedding settings for your data:

1. **Start with defaults** — Recursive character splitting, 500 chars, 50 overlap, `text-embedding-3-small`.
2. **Build a test set** — Create 20-30 representative questions with known answers from your documents.
3. **Measure retrieval** — For each question, check if the correct chunk appears in the top-5 results.
4. **Adjust one variable at a time** — Change chunk size, overlap, or embedding model. Re-run the test set.
5. **Stop when retrieval hits 80%+** — If 80% of test questions retrieve the right chunk in top-5, your settings are good enough.

```ballerina
// Simple retrieval evaluation
function evaluateRetrieval(string question, string expectedContent) returns boolean|error {
    float[] queryVector = check embedder->embed(question);
    ai:SearchResult[] results = check vectorStore->search(queryVector, k = 5);

    foreach ai:SearchResult result in results {
        if result.content.includes(expectedContent) {
            return true;
        }
    }
    return false;
}
```

## What's Next

- [Build Document Ingestion Pipelines](ingestion-pipelines.md) — Apply your chunking strategy in a complete ingestion integration
- [Build a RAG Query Service](building-rag-service.md) — Use your embeddings to answer questions from your knowledge base
- [RAG Architecture Overview](architecture-overview.md) — Revisit the full RAG pattern and design decisions
