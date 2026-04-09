---
title: "Connecting to a Vector Database"
description: "Connect your RAG integration to vector databases like Pinecone, Weaviate, Chroma, pgvector, and Milvus."
---

# Connecting to a Vector Database

You need a vector database to store and search the embeddings that power your RAG integration. WSO2 Integrator supports several vector databases out of the box, so you can choose the one that fits your infrastructure and scale requirements.

## Supported Vector Databases

| Database | Type | Best for | Managed Option |
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

## Configuration Examples

### In-Memory (Development)

```ballerina
import ballerina/ai;

// No external dependencies — great for development and testing
final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
```

:::warning
The in-memory store does not persist data between restarts. Use it only for development and testing. For production deployments, choose a persistent vector database.
:::

You can also use the in-memory store through the official RAG API:

```ballerina
import ballerina/ai;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase = new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
```

### Pinecone

Using the official Pinecone connector:

```ballerina
import ballerina/ai;
import ballerina/io;
import ballerinax/ai.pinecone;

configurable string pineconeServiceUrl = ?;
configurable string pineconeApiKey = ?;

final ai:VectorStore vectorStore = check new pinecone:VectorStore(pineconeServiceUrl, pineconeApiKey);
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase = new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);

public function main() returns error? {
    ai:DataLoader loader = check new ai:TextDataLoader("./leave_policy.pdf");
    ai:Document|ai:Document[] documents = check loader.load();
    ai:Error? ingestionResult = knowledgeBase.ingest(documents);
    if ingestionResult is () {
        io:println("Ingestion successful");
    } else {
        io:println("Ingestion failed: ", ingestionResult.message());
    }
}
```

Alternative configuration style:

```ballerina
import ballerina/ai;
import ballerinax/ai.pinecone;

configurable string pineconeServiceUrl = ?;
configurable string pineconeApiKey = ?;

final ai:VectorStore vectorStore = check new pinecone:VectorStore(pineconeServiceUrl, pineconeApiKey);
```

### Weaviate

```ballerina
import ballerina/ai;

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
import ballerina/ai;
import ballerinax/postgresql;

configurable postgresql:ConnectionConfig pgConfig = ?;

final ai:VectorStore vectorStore = check new ai:PgVectorStore({
    connection: pgConfig,
    tableName: "document_embeddings",
    dimensions: 1536
});
```

## Similarity Search

Once your vector store is connected and populated, you query it by passing a vector and retrieving the closest matches.

### Basic Search with Top-K

```ballerina
// Basic similarity search — returns the top 5 most similar chunks
ai:SearchResult[] results = check vectorStore->search(queryVector, k = 5);

foreach ai:SearchResult result in results {
    io:println("Score: ", result.score);
    io:println("Content: ", result.content);
    io:println("Metadata: ", result.metadata);
}
```

The `k` parameter (top-K) controls how many chunks are returned. More chunks add context but increase token costs when sent to the LLM:

| Top-K | Use Case |
|-------|----------|
| **3-5** | Focused answers, lower token cost |
| **5-10** | Balanced coverage (recommended default) |
| **10-20** | Broad context, complex multi-part questions |

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

Every chunk you store can carry metadata -- key-value pairs that describe the source, section, timestamp, or any other attribute you want to filter on later:

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

| Embedding Model | Dimensions |
|----------------|------------|
| OpenAI `text-embedding-ada-002` | 1536 |
| OpenAI `text-embedding-3-small` | 1536 |
| OpenAI `text-embedding-3-large` | 3072 |
| Cohere `embed-english-v3.0` | 1024 |

## Visual Designer Configuration

You can configure the vector store through the visual designer when creating a Vector Knowledge Base:

1. In the flow editor, hover over the flow line and click the **+** icon.
2. Select **Vector Knowledge Bases** under the **AI** section.
3. Click **+ Add Vector Knowledge Base**.
4. In the **Vector Store** section, click **+ Create New Vector Store** and choose your preferred store type (e.g., **InMemory Vector Store** for development or an external store for production).
5. Click **Save** to complete the configuration.

## What's Next

- [Chunking Strategies](chunking-strategies.md) -- Choose how to split your documents for optimal retrieval
- [Configuring Embeddings](configuring-embeddings.md) -- Select and configure your embedding model
- [RAG Querying](../rag-query/rag-querying.md) -- Retrieve relevant context at query time
