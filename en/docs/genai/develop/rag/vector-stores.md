---
sidebar_position: 4
title: Vector Stores
description: Reference for vector stores in WSO2 Integrator — In-Memory, Pinecone, Weaviate, Milvus, pgvector, Azure AI Search.
---

# Vector Stores

A **vector store** is the database for embeddings. It stores the vectors produced by the embedding provider and answers the question *"give me the vectors closest to this one"* in milliseconds. Every Knowledge Base in WSO2 Integrator references exactly one vector store.

## Supported Stores

| Store | Module | Persistent? | Hybrid search? | Best for |
|---|---|---|---|---|
| **In-memory** | `ballerina/ai` | No | No | Development, prototyping, automated tests. |
| **Pinecone** | `ballerinax/ai.pinecone` | Yes | Yes | Managed SaaS, zero ops, easy to scale. |
| **Weaviate** | `ballerinax/ai.weaviate` | Yes | Yes | Open-source, flexible schemas, hybrid search out of the box. |
| **Milvus** | `ballerinax/ai.milvus` | Yes | Yes | Very large datasets, low-latency search at scale. |
| **pgvector** | `ballerinax/ai.pgvector` | Yes | With `tsvector` | You already run PostgreSQL and want vectors next to relational data. |
| **Azure AI Search** | `ballerinax/ai.azure` | Yes | Yes | Existing Azure investment, managed service with full-text + vector. |

The picker shows whichever stores are available based on the modules in your project. The **+ Create New Vector Store** link from the Knowledge Base form opens this picker.

## Adding a Vector Store

From the Knowledge Base configuration form (or the AI section of any **Add Connection** dialog):

1. Click **Vector Store** → **+ Create New Vector Store**.
2. Pick the store type from the picker.
3. Fill in the configuration form (varies by store — see below).
4. Click **Save**.

## In-Memory

No external infrastructure. The fastest way to build and test a RAG pipeline.

```ballerina
final ai:VectorStore aiInmemoryVectorStore = check new ai:InMemoryVectorStore();
```

| Pros | Cons |
|---|---|
| Zero setup. | Vectors disappear on restart. |
| Great for tests. | Not suitable for production. |

When the integration restarts you'd need to re-ingest. For demo apps, build pipelines that re-ingest at startup.

## Pinecone

Managed SaaS vector database. The path of least resistance for production.

```ballerina
import ballerinax/ai.pinecone;

configurable string pineconeServiceUrl = ?;
configurable string pineconeApiKey = ?;

final ai:VectorStore pineconeStore =
        check new pinecone:VectorStore(pineconeServiceUrl, pineconeApiKey);
```

```toml
# Config.toml
pineconeServiceUrl = "https://your-index.svc.region.pinecone.io"
pineconeApiKey = "pcsk-..."
```

| Form field | What it does |
|---|---|
| Service URL | The Pinecone index URL — visible in your Pinecone dashboard. |
| API key | A Pinecone API key. Always store as a `configurable`. |

## Weaviate

Open-source, hybrid search (vector + keyword) built in.

```ballerina
import ballerinax/ai.weaviate;

configurable string weaviateUrl = ?;
configurable string weaviateApiKey = ?;

final ai:VectorStore weaviateStore =
        check new weaviate:VectorStore(weaviateUrl, weaviateApiKey);
```

Useful when you need rich metadata filters (filter by tag, by document, by date) alongside vector similarity.

## Milvus

High-performance, optimised for very large datasets and low-latency queries.

```ballerina
import ballerinax/ai.milvus;

configurable string milvusUrl = ?;
configurable string milvusToken = ?;

final ai:VectorStore milvusStore =
        check new milvus:VectorStore(milvusUrl, milvusToken);
```

Pick this when you have tens of millions of chunks or strict tail-latency requirements.

## pgvector (PostgreSQL)

The `vector` extension in PostgreSQL. Useful if you already operate Postgres and want to keep vector data close to relational data.

```ballerina
import ballerinax/ai.pgvector;

configurable string pgHost = ?;
configurable int pgPort = 5432;
configurable string pgDatabase = ?;
configurable string pgUser = ?;
configurable string pgPassword = ?;

final ai:VectorStore pgStore = check new pgvector:VectorStore(
    host = pgHost, port = pgPort,
    database = pgDatabase, user = pgUser, password = pgPassword
);
```

Make sure the extension is enabled on your database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

## Azure AI Search

If you're already on Azure, Azure AI Search provides vector + full-text search in one service. Use it via the **Azure AI Search Knowledge Base** rather than wiring it as a Vector Store under a Vector Knowledge Base — the dedicated implementation supports Azure-specific features (built-in chunking, vectorisation skills) more naturally.

## Choosing a Vector Store

Quick decision rules of thumb:

| Situation | Pick |
|---|---|
| First time setting up RAG, want to see it working today. | **In-memory**, then move when you actually need persistence. |
| Need persistence with the least operations work. | **Pinecone**. |
| You're already on Postgres and want to avoid a new service. | **pgvector**. |
| Need hybrid search and want to self-host. | **Weaviate**. |
| Hundreds of millions of vectors. | **Milvus**. |
| You're on Azure. | **Azure AI Search Knowledge Base**. |

## Filters and Metadata

Every chunk stored has metadata — at minimum, the source document and (with `ai:DataLoader`) the file name and line range. Most production stores support filtering: *"retrieve from documents tagged `policy=2024`"*. The `ai:Chunk` record's `metadata` field is where you put your own tags during ingestion; the **Delete By Filter** action uses the same metadata for selective deletion.

The exact filter syntax varies by store — check the store's connector documentation.

## Operational Notes

- **Index hygiene.** Re-ingestion appends; over time, stale chunks build up. Tag chunks with a document version and clean old versions out periodically with **Delete By Filter**.
- **Embedding/store coupling.** A vector store doesn't care which embedding model you used, but the chunks inside it are only meaningful to the matching model. Don't share a store across two pipelines using different embedding providers.
- **Backups.** Persistent stores are state. Treat backups the way you would for any other database.
- **Network proximity.** RAG is on the hot path of every query. Keep the vector store in the same region as your integration.

## What's Next

- **[Embedding Providers](embedding-providers.md)** — the other pluggable piece.
- **[Knowledge Bases](knowledge-bases.md)** — wire the vector store into the pipeline.
- **[Query Nodes](query-node.md)** — use the Knowledge Base from a flow.
