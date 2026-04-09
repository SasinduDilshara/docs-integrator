---
title: "Configuring Embeddings"
description: "Choose an embedding model, understand dimensionality trade-offs, and configure the embedding provider in your RAG integration."
---

# Configuring Embeddings

The embedding model converts text into numerical vectors that capture semantic meaning. Your choice of embedding model affects retrieval accuracy, speed, and cost. This guide helps you select the right model and configure it in WSO2 Integrator.

## Embedding Models Comparison

| Model | Dimensions | Relative Quality | Speed | Cost |
|-------|-----------|-----------------|-------|------|
| OpenAI `text-embedding-ada-002` | 1536 | Good | Fast | Low |
| OpenAI `text-embedding-3-small` | 1536 | Better | Fast | Low |
| OpenAI `text-embedding-3-large` | 3072 | Best | Moderate | Higher |
| Cohere `embed-english-v3.0` | 1024 | Good | Fast | Low |
| Azure OpenAI Embeddings | 1536 | Good | Fast | Varies |

## Configuring the Embedding Provider

### Using the Default Embedding Provider

WSO2 Integrator provides a default embedding provider that you can use without managing API keys. This is the simplest way to get started.

```ballerina
import ballerina/ai;

final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
```

The default embedding provider generates dense vectors with **1536 dimensions**.

:::info
To configure the default WSO2 provider, press `Ctrl/Cmd + Shift + P` in VS Code and run the command: **Ballerina: Configure default WSO2 model provider**.
:::

### Using OpenAI Embeddings

```ballerina
import ballerina/ai;

configurable string openaiApiKey = ?;

// OpenAI embeddings (recommended for production)
final ai:EmbeddingProvider embedder = check new ai:OpenAIEmbedder({
    apiKey: openaiApiKey,
    model: "text-embedding-3-small"
});
```

:::info
For LLM connection configuration details, including API key management and endpoint setup, see the [Connectors](/docs/connectors/ai-llms) section.
:::

### Using Embeddings in a RAG Pipeline

The following example shows a complete RAG ingestion flow using the default embedding provider with an in-memory vector store:

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase = new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);

public function main() returns error? {
    // Ingestion
    ai:DataLoader loader = check new ai:TextDataLoader("./leave_policy.md");
    ai:Document|ai:Document[] documents = check loader.load();
    check knowledgeBase.ingest(documents);
    io:println("Ingestion successful");
}
```

When you call `knowledgeBase.ingest(documents)`, the knowledge base automatically chunks the documents, generates embeddings using the configured embedding provider, and stores the resulting vectors in the vector store.

## Dimensionality Trade-offs

Higher-dimensional embeddings capture more nuance but cost more to store and search:

- **1536 dimensions** -- The sweet spot for most integrations. Balances quality with performance and storage costs. This is the default in WSO2 Integrator.
- **3072 dimensions** -- Use when retrieval precision is critical and you can tolerate higher storage and slightly slower search.
- **1024 dimensions** -- Lighter weight, faster search, slightly lower quality. Good for large-scale use cases where speed matters more than precision.

:::warning
Your embedding dimensions must match your vector store configuration. If you switch embedding models, you need to re-index all documents with the new model -- you cannot mix embeddings from different models in the same vector store.
:::

## Visual Designer Configuration

You can also configure the embedding provider through the visual designer:

1. When creating a **Vector Knowledge Base**, click **+ Create New Embedding Model** in the **Embedding Model** section.
2. Select **Default Embedding Provider (WSO2)** for quick setup, or configure a custom provider.
3. Click **Save** to apply.

:::warning Embedding Dimensions
The **Default Embedding Provider (WSO2)** generates dense vectors with **1536 dimensions**. If you are using an external vector store (Pinecone, Milvus, Weaviate, etc.), ensure your vector store index is configured to support 1536-dimensional vectors.
:::

## What's Next

- [Chunking Strategies](chunking-strategies.md) -- Choose the right chunking method for your content
- [Connecting to a Vector Database](connecting-to-vector-database.md) -- Store your embeddings in a persistent vector database
