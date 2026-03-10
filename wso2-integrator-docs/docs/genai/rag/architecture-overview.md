---
sidebar_position: 1
title: "RAG Architecture Overview"
description: "Understand the Retrieval-Augmented Generation pattern and when to use it in your integrations."
---

# RAG Architecture Overview

Retrieval-Augmented Generation (RAG) lets you ground LLM responses in your own data instead of relying solely on the model's training data. You use RAG when you need accurate, up-to-date answers drawn from documents, knowledge bases, or any structured content your organization owns.

## How RAG Works

RAG splits into two distinct phases: **indexing** (preparing your data) and **querying** (answering questions at runtime). Understanding both phases helps you design integrations that are accurate, fast, and cost-effective.

### Indexing Phase

During indexing, you transform raw documents into searchable vector embeddings and store them for later retrieval.

```
Documents (PDF, HTML, Markdown)
        │
        ▼
   ┌─────────┐
   │  Parse   │  ── Extract raw text from source formats
   └────┬─────┘
        ▼
   ┌─────────┐
   │  Chunk   │  ── Split text into meaningful segments
   └────┬─────┘
        ▼
   ┌─────────┐
   │  Embed   │  ── Convert chunks into vector embeddings (1536 dimensions)
   └────┬─────┘
        ▼
   ┌──────────────┐
   │ Vector Store  │  ── Persist embeddings for similarity search
   └──────────────┘
```

Each chunk becomes a vector — a numerical representation that captures semantic meaning. When you store these vectors in a database like Pinecone or Weaviate, you create a searchable knowledge base.

### Query Phase

At query time, the user's question follows a retrieve-then-generate flow:

```
User Question
        │
        ▼
   ┌─────────┐
   │  Embed   │  ── Convert question into a vector
   └────┬─────┘
        ▼
   ┌───────────┐
   │ Retrieve  │  ── Find the most similar chunks from the vector store
   └────┬──────┘
        ▼
   ┌───────────┐
   │ Augment   │  ── Combine retrieved chunks with the original question
   └────┬──────┘
        ▼
   ┌───────────┐
   │ Generate  │  ── Send augmented prompt to the LLM
   └────┬──────┘
        ▼
   Grounded Response
```

:::info
The LLM never sees your entire knowledge base. It only receives the most relevant chunks for each query, which keeps costs down and responses focused.
:::

## RAG in WSO2 Integrator

WSO2 Integrator gives you two ways to build RAG:

- **Visual designer** — Drag components onto the canvas to wire up ingestion and query integrations without writing code.
- **Pro-code** — Write Ballerina code directly for full control over chunking strategies, embedding models, and retrieval logic.

Both approaches produce the same deployable artifact. You can start in the visual designer and switch to pro-code when you need fine-grained control.

```ballerina
import ballerinax/ai;

// Embedding a query and retrieving relevant context
public function retrieveContext(string question) returns string|error {
    // Embed the user's question
    float[] queryVector = check ai:embed(question);

    // Search the vector store for similar chunks
    ai:SearchResult[] results = check vectorStore->search(queryVector, k = 5);

    // Combine retrieved chunks into context
    string context = "";
    foreach ai:SearchResult result in results {
        context += result.content + "\n\n";
    }
    return context;
}
```

## When to Use RAG

RAG is not always the right approach. Here is a comparison to help you decide:

| Approach | Best for | Trade-offs |
|----------|----------|------------|
| **RAG** | Domain-specific Q&A, frequently updated data, internal knowledge bases | Requires vector database infrastructure, retrieval adds latency |
| **Fine-tuning** | Specialized tone/style, domain terminology, consistent behavior patterns | Expensive to train, data becomes stale, no real-time updates |
| **Prompt engineering** | Simple instructions, formatting rules, few-shot examples | Limited by context window, no persistent knowledge |

:::tip
Start with RAG if your data changes regularly or you need source attribution. Use fine-tuning only when RAG retrieval quality plateaus and you need the model to deeply internalize domain knowledge.
:::

## Key Design Decisions

When you plan a RAG integration, consider these decisions upfront:

1. **Chunk size** — Smaller chunks (200-500 tokens) give precise retrieval; larger chunks (500-1000 tokens) preserve more context. See [Chunking & Embedding Strategies](chunking-embedding.md).
2. **Embedding model** — The model determines vector dimensions and semantic quality. WSO2 Integrator defaults to 1536-dimension embeddings.
3. **Vector database** — Choose based on scale, latency requirements, and managed vs. self-hosted preferences. See [Connect to Vector Databases](vector-database.md).
4. **Top-k retrieval** — How many chunks to retrieve per query. More chunks add context but increase token costs.
5. **Reranking** — Optionally rerank retrieved chunks before sending them to the LLM for higher answer quality.

## What's Next

- [Connect to Vector Databases](vector-database.md) — Set up your vector store for embeddings
- [Build Document Ingestion Pipelines](ingestion-pipelines.md) — Create the indexing phase of your RAG integration
- [Build a RAG Query Service](building-rag-service.md) — Create the query phase as an HTTP service
