---
sidebar_position: 3
title: Connecting to Vector Databases
description: Connect WSO2 Integrator RAG pipelines to Pinecone, Weaviate, Milvus, and pgvector for production-grade embedding storage and retrieval.
---

# Connecting to Vector Databases

Vector databases store and index high-dimensional embeddings for fast similarity search. They are the backbone of any RAG pipeline, enabling your application to find relevant document chunks in milliseconds.

In WSO2 Integrator, every vector store implements the `ai:VectorStore` interface. You can start with the built-in in-memory store for development and switch to a production database by changing a single line of code.

## Supported Vector Stores

| Store | Module | Best For |
|---|---|---|
| **In-memory** | `ballerina/ai` | Development, prototyping, tests |
| **Pinecone** | `ballerinax/ai.pinecone` | Managed SaaS, zero-ops production |
| **Weaviate** | `ballerinax/ai.weaviate` | Hybrid search, flexible schemas |
| **Milvus** | `ballerinax/ai.milvus` | High-performance, large-scale workloads |
| **pgvector** | `ballerinax/ai.pgvector` | Existing PostgreSQL infrastructure |

## In-Memory Vector Store

The in-memory store ships with the `ballerina/ai` module. It requires no external infrastructure and is ideal for local development and automated tests.

```ballerina
import ballerina/ai;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
```

Because the store lives entirely in process memory, all data is lost when the program exits.

## Pinecone

Pinecone is a fully managed vector database with built-in scaling. Use the `ballerinax/ai.pinecone` connector to plug it into your knowledge base.

```ballerina
import ballerina/ai;
import ballerinax/ai.pinecone;

configurable string pineconeServiceUrl = ?;
configurable string pineconeApiKey = ?;

final ai:VectorStore vectorStore =
        check new pinecone:VectorStore(pineconeServiceUrl, pineconeApiKey);

final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();

final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
```

```toml
# Config.toml
pineconeServiceUrl = "https://your-index.svc.region.pinecone.io"
pineconeApiKey = "pcsk-..."
```

Notice that only the `VectorStore` line changes -- the `KnowledgeBase` construction is identical to the in-memory example. This is the benefit of programming against the `ai:VectorStore` interface.

## Weaviate

Weaviate is an open-source vector database with built-in support for hybrid (vector + keyword) search.

```ballerina
import ballerina/ai;
import ballerinax/ai.weaviate;

configurable string weaviateUrl = ?;
configurable string weaviateApiKey = ?;

final ai:VectorStore vectorStore =
        check new weaviate:VectorStore(weaviateUrl, weaviateApiKey);

final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
```

```toml
# Config.toml
weaviateUrl = "https://your-cluster.weaviate.network"
weaviateApiKey = "..."
```

## Milvus

Milvus is built for high-performance, large-scale vector search workloads.

```ballerina
import ballerina/ai;
import ballerinax/ai.milvus;

configurable string milvusUrl = ?;
configurable string milvusToken = ?;

final ai:VectorStore vectorStore =
        check new milvus:VectorStore(milvusUrl, milvusToken);

final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
```

```toml
# Config.toml
milvusUrl = "http://localhost:19530"
milvusToken = "..."
```

## pgvector (PostgreSQL)

If you already run PostgreSQL, the `pgvector` extension lets you store and query embeddings alongside the rest of your relational data.

```ballerina
import ballerina/ai;
import ballerinax/ai.pgvector;

configurable string pgHost = ?;
configurable int pgPort = 5432;
configurable string pgDatabase = ?;
configurable string pgUser = ?;
configurable string pgPassword = ?;

final ai:VectorStore vectorStore = check new pgvector:VectorStore(
    host = pgHost,
    port = pgPort,
    database = pgDatabase,
    user = pgUser,
    password = pgPassword
);

final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
```

```toml
# Config.toml
pgHost = "localhost"
pgPort = 5432
pgDatabase = "rag"
pgUser = "postgres"
pgPassword = "..."
```

Make sure the `vector` extension is enabled on your database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

## Choosing a Vector Store

| Factor | In-memory | Pinecone | Weaviate | Milvus | pgvector |
|---|---|---|---|---|---|
| **Setup** | None | Managed | Docker / Cloud | Docker / Cloud | SQL extension |
| **Scaling** | N/A | Automatic | Manual / Managed | Manual / Managed | Manual |
| **Persistence** | No | Yes | Yes | Yes | Yes |
| **Hybrid search** | No | Yes | Yes | Yes | With tsvector |
| **Best for** | Dev / tests | Production SaaS | Flexible schemas | Large scale | Existing Postgres |

## What's Next

- [Chunking Documents](/docs/genai/develop/rag/chunking-documents) -- Chunking strategies for RAG
- [Generating Embeddings](/docs/genai/develop/rag/generating-embeddings) -- Embedding model selection
- [RAG Querying](/docs/genai/develop/rag/rag-querying) -- Build the complete query pipeline
