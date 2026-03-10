---
sidebar_position: 2
title: "Connect to Vector Databases"
description: "Connect your RAG integration to vector databases like Pinecone, Weaviate, Chroma, and pgvector."
---

# Connect to Vector Databases

You need a vector database to store and search the embeddings that power your RAG integration. WSO2 Integrator supports several vector databases out of the box, so you can choose the one that fits your infrastructure and scale requirements.

## Supported Vector Databases

| Database | Type | Best for | Managed option |
|----------|------|----------|----------------|
| **In-memory** | Built-in | Development, testing, small datasets | N/A (runs locally) |
| **Pinecone** | Cloud-native | Production workloads, fully managed, zero ops | Yes |
| **Weaviate** | Self-hosted / Cloud | Hybrid search (vector + keyword), GraphQL API | Yes |
| **Chroma** | Self-hosted | Lightweight, open-source, quick setup | No |
| **pgvector** | PostgreSQL extension | Teams already running PostgreSQL | Via cloud providers |
| **Milvus** | Self-hosted / Cloud | High-scale, distributed vector search | Yes (Zilliz) |

:::info
For connection setup details (API keys, endpoints, authentication), see the [Connectors](/docs/connectors/configuration) section. This page focuses on vector-specific operations and configuration.
:::

## Configure a Vector Store Connection

Each vector database requires a connection configuration. Here is how you set up the most common options:

### Pinecone

```ballerina
import ballerinax/ai;

configurable string pineconeApiKey = ?;
configurable string pineconeEnvironment = ?;
configurable string pineconeIndex = ?;

final ai:VectorStore vectorStore = check new ai:PineconeStore({
    apiKey: pineconeApiKey,
    environment: pineconeEnvironment,
    index: pineconeIndex
});
```

### Weaviate

```ballerina
import ballerinax/ai;

configurable string weaviateUrl = ?;
configurable string weaviateApiKey = ?;

final ai:VectorStore vectorStore = check new ai:WeaviateStore({
    url: weaviateUrl,
    apiKey: weaviateApiKey,
    className: "Documents"
});
```

### pgvector

```ballerina
import ballerinax/ai;
import ballerinax/postgresql;

configurable postgresql:ConnectionConfig pgConfig = ?;

final ai:VectorStore vectorStore = check new ai:PgVectorStore({
    connection: pgConfig,
    tableName: "document_embeddings",
    dimensions: 1536
});
```

### In-Memory (Development)

```ballerina
import ballerinax/ai;

// No external dependencies — great for development and testing
final ai:VectorStore vectorStore = check new ai:InMemoryStore({
    dimensions: 1536
});
```

:::warning
The in-memory store does not persist data between restarts. Use it only for development and testing. For production deployments, choose a persistent vector database.
:::

## Similarity Search

Once your vector store is connected and populated, you query it by passing a vector and retrieving the closest matches:

```ballerina
// Basic similarity search — returns the top 5 most similar chunks
ai:SearchResult[] results = check vectorStore->search(queryVector, k = 5);

foreach ai:SearchResult result in results {
    io:println("Score: ", result.score);
    io:println("Content: ", result.content);
    io:println("Metadata: ", result.metadata);
}
```

### Filtered Search

You can narrow results using metadata filters. This is useful when your knowledge base spans multiple document types, departments, or versions:

```ballerina
// Search only within a specific category
ai:SearchResult[] results = check vectorStore->search(queryVector, k = 5, filter = {
    "category": "product-docs",
    "version": "2.0"
});
```

### Similarity Thresholds

Set a minimum similarity score to avoid returning irrelevant results:

```ballerina
// Only return chunks with a similarity score above 0.75
ai:SearchResult[] results = check vectorStore->search(queryVector, k = 10, minScore = 0.75);
```

:::tip
Start with a `minScore` of 0.7 and adjust based on your data. If you get too few results, lower it. If results feel off-topic, raise it.
:::

## Metadata Handling

Every chunk you store can carry metadata — key-value pairs that describe the source, section, timestamp, or any other attribute you want to filter on later:

```ballerina
// Store a chunk with metadata
check vectorStore->insert({
    content: chunkText,
    vector: embedding,
    metadata: {
        "source": "product-guide.pdf",
        "page": 42,
        "section": "installation",
        "lastUpdated": "2026-01-15"
    }
});
```

Metadata is not included in the vector similarity calculation. It is purely for filtering and attribution.

## Embedding Dimensions

WSO2 Integrator defaults to **1536-dimension** embeddings, which is the output size of commonly used models like OpenAI's `text-embedding-ada-002`. If you use a different embedding model, make sure the dimensions in your vector store configuration match the model's output size.

| Embedding model | Dimensions |
|----------------|------------|
| OpenAI `text-embedding-ada-002` | 1536 |
| OpenAI `text-embedding-3-small` | 1536 |
| OpenAI `text-embedding-3-large` | 3072 |
| Cohere `embed-english-v3.0` | 1024 |

## What's Next

- [Build Document Ingestion Pipelines](ingestion-pipelines.md) — Load, chunk, embed, and store documents in your vector database
- [Build a RAG Query Service](building-rag-service.md) — Query your vector store and generate LLM-augmented responses
- [Connectors Configuration](/docs/connectors/configuration) — Set up authentication and connection details
