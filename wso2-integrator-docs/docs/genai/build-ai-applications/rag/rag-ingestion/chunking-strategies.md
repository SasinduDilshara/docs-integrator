---
title: "Chunking Strategies"
description: "Choose the right chunking method and overlap strategy to maximize retrieval quality in your RAG integration."
---

# Chunking Strategies

The way you split documents into chunks directly affects how well your RAG integration retrieves relevant context. Chunks that are too large dilute the semantic signal; chunks that are too small lose surrounding context. This guide covers the chunking methods available in WSO2 Integrator and how to choose between them.

## Chunking Methods

WSO2 Integrator supports several chunking strategies. Each suits different content types and retrieval goals.

### Fixed-Size Chunking

Split text into chunks of a fixed token or character count. Simple and predictable.

```ballerina
import ballerina/ai;

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

**When to use:** Most document types -- technical docs, articles, reports, manuals.

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
Start with recursive character splitting at 500 characters with 50-character overlap. Only move to semantic chunking if retrieval quality is not meeting your needs -- it adds latency and embedding costs to the ingestion step.
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

## Comparing Chunking Approaches

| Method | Quality | Speed | Cost | Best for |
|--------|---------|-------|------|----------|
| **Fixed-Size** | Adequate | Fastest | Lowest | Uniform, structured content |
| **Recursive Character** | Good | Fast | Low | General-purpose (recommended default) |
| **Sentence-Based** | Good | Fast | Low | Conversational and transcript content |
| **Semantic** | Best | Slow | Higher | Long-form content with topic shifts |

## Practical Tuning Workflow

Follow this process to find the right chunking settings for your data:

1. **Start with defaults** -- Recursive character splitting, 500 chars, 50 overlap.
2. **Build a test set** -- Create 20-30 representative questions with known answers from your documents.
3. **Measure retrieval** -- For each question, check if the correct chunk appears in the top-5 results.
4. **Adjust one variable at a time** -- Change chunk size, overlap, or chunking method. Re-run the test set.
5. **Stop when retrieval hits 80%+** -- If 80% of test questions retrieve the right chunk in top-5, your settings are good enough.

:::info
The ingestion pipeline in WSO2 Integrator handles chunking automatically when you call `knowledgeBase.ingest(documents)`. You can configure the chunker when creating the `VectorKnowledgeBase`, or leave it set to `AUTO` to use the default recursive character splitting strategy.
:::

## What's Next

- [Configuring Embeddings](configuring-embeddings.md) -- Choose an embedding model and configure the embedding provider
- [Connecting to a Vector Database](connecting-to-vector-database.md) -- Set up your vector store to persist embeddings
