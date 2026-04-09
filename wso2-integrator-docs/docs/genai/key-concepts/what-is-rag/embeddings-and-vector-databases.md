---
title: "Embeddings and Vector Databases"
description: "Understand how text is converted to vector embeddings and how vector databases enable similarity search."
---

# Embeddings and Vector Databases

Embeddings and vector databases are the foundation of RAG. Embeddings convert text into numerical representations that capture meaning, and vector databases store and search those representations efficiently. Together, they enable the "retrieval" part of Retrieval-Augmented Generation.

## What Are Embeddings?

An **embedding** is a list of numbers (a vector) that represents the meaning of a piece of text. When you pass text through an embedding model, it produces a dense numerical vector -- typically with hundreds or thousands of dimensions -- where the position in this high-dimensional space encodes semantic meaning.

The key property of embeddings is that **similar text produces similar vectors**. Two sentences about the same topic will have vectors that are close together in space, even if they use completely different words.

For example:
- "How do I reset my password?" and "I forgot my login credentials" produce vectors that are close together.
- "How do I reset my password?" and "What is the weather today?" produce vectors that are far apart.

### How Embedding Models Work

Embedding models are neural networks trained on large text corpora to produce vectors where semantic similarity corresponds to geometric proximity. The process is straightforward:

1. **Input** -- You send a text string to the embedding model.
2. **Processing** -- The model processes the text through its neural network layers.
3. **Output** -- The model returns a fixed-length vector (e.g., 1536 numbers for OpenAI's `text-embedding-ada-002`).

The dimensionality of the vector is fixed per model:

| Embedding model | Dimensions |
|---|---|
| OpenAI `text-embedding-ada-002` | 1536 |
| OpenAI `text-embedding-3-small` | 1536 |
| OpenAI `text-embedding-3-large` | 3072 |
| Cohere `embed-english-v3.0` | 1024 |

Higher dimensions capture more nuance but require more storage and slower search. For most integration scenarios, 1536 dimensions provide a good balance of quality and performance.

## How Vector Similarity Works

Once text is converted to vectors, you can measure how similar two pieces of text are by calculating the distance between their vectors. The most common similarity measure is **cosine similarity**, which measures the angle between two vectors:

- A cosine similarity of **1.0** means the vectors point in the same direction (identical meaning).
- A cosine similarity of **0.0** means the vectors are orthogonal (unrelated).
- Values in between indicate degrees of semantic similarity.

When a user asks a question in a RAG system, the question is embedded into a vector, and the system finds the stored vectors that are most similar -- the ones closest in meaning to the question.

## What Vector Databases Do

A **vector database** is a specialized database designed to store vectors and perform fast similarity searches across them. Traditional databases excel at exact matching (find the row where `id = 42`), but vector databases excel at **approximate nearest neighbor** search (find the 5 vectors most similar to this query vector).

Vector databases handle:

- **Storage** -- Persist millions or billions of vectors along with their associated text content and metadata.
- **Indexing** -- Build efficient index structures (like HNSW or IVF) that enable fast approximate search without comparing against every stored vector.
- **Search** -- Given a query vector, return the top-K most similar vectors ranked by similarity score.
- **Filtering** -- Narrow search results by metadata (e.g., only search within documents from a specific department or date range).

## Supported Vector Databases in Ballerina

WSO2 Integrator supports several vector databases, so you can choose one that fits your infrastructure:

| Database | Type | Best for |
|---|---|---|
| **In-memory** | Built-in | Development, testing, small datasets |
| **Pinecone** | Cloud-native | Production workloads, fully managed, zero ops |
| **Weaviate** | Self-hosted / Cloud | Hybrid search (vector + keyword), GraphQL API |
| **Chroma** | Self-hosted | Lightweight, open-source, quick setup |
| **pgvector** | PostgreSQL extension | Teams already running PostgreSQL |
| **Milvus** | Self-hosted / Cloud | High-scale, distributed vector search |

For development and testing, the in-memory store requires no external infrastructure:

```ballerina
import ballerina/ai;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
```

For production, choose a persistent vector database based on your scale, latency requirements, and operational preferences.

## The Embedding-Storage-Search Pipeline

Putting it all together, the flow from text to searchable knowledge looks like this:

1. **Chunk** your documents into smaller segments (paragraphs, sections, or fixed-size blocks).
2. **Embed** each chunk by passing it through an embedding model to get a vector.
3. **Store** each vector alongside its original text and metadata in the vector database.
4. **Search** by embedding the user's question and finding the most similar stored vectors.
5. **Retrieve** the original text associated with the top matching vectors.

This pipeline runs during both the ingestion phase (steps 1-3) and the query phase (steps 4-5) of a RAG system.

## What's Next

- [RAG Ingestion](rag-ingestion.md) -- How the ingestion pipeline transforms documents into stored embeddings
- [RAG Querying](rag-querying.md) -- How the query pipeline retrieves context and generates answers
