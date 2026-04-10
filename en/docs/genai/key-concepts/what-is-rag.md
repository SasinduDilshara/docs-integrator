---
sidebar_position: 7
title: What is RAG?
description: Understand retrieval-augmented generation -- how to ground LLM responses in your own data.
---

# What is RAG?

Retrieval-Augmented Generation (RAG) combines the reasoning power of large language models with the accuracy of your own data. Instead of relying solely on the LLM's training data, a RAG pipeline retrieves relevant documents from a knowledge base and passes them as context to the model before generating a response.

## Why RAG?

LLMs have three key limitations that RAG addresses:

| Limitation | How RAG Solves It |
|------------|-------------------|
| **Knowledge cutoff** -- Training data has a fixed date | RAG retrieves up-to-date documents from your knowledge base |
| **Hallucination** -- Models can generate plausible but incorrect information | RAG grounds responses in actual documents |
| **Domain specificity** -- General models lack deep knowledge of your data | RAG provides your proprietary data as context |

## How RAG Works

A RAG pipeline has two phases: **ingestion** (offline) and **querying** (online).

### Ingestion: Preparing Your Data

Documents are split into chunks, converted to vector embeddings, and stored in a vector database.

```
Documents  -->  Chunking  -->  Embedding  -->  Vector Store
(PDF, HTML,    (Split into    (Convert to    (Store for
 JSON, etc.)    passages)      vectors)       similarity search)
```

### Querying: Answering Questions

When a user asks a question, it is embedded and compared against stored vectors to find relevant chunks, which are then passed to the LLM.

```
User Query  -->  Embed Query  -->  Vector Search  -->  LLM + Context  -->  Answer
                                  (Find similar      (Generate answer
                                   chunks)            from context)
```

## Key Concepts

### Chunking

Documents are split into smaller passages (chunks) that can be individually retrieved. Common strategies include:

- **Fixed-size** -- Split by token or character count
- **Paragraph-based** -- Split at natural paragraph boundaries
- **Semantic** -- Group semantically related sentences together

### Embedding

Each chunk is converted into a vector (an array of numbers) that captures its semantic meaning. Similar content produces similar vectors, enabling semantic search.

### Vector Stores

Specialized databases that store embeddings and perform fast similarity search. WSO2 Integrator ships with an in-memory vector store for development and supports Pinecone, Weaviate, Milvus, and pgvector for production deployments.

### Retrieval

When a query arrives, it is embedded and compared against stored vectors to find the most relevant chunks (typically the top 3-10 matches).

### Generation

The retrieved chunks are assembled into a prompt context and passed to the LLM alongside the user's question. The LLM generates an answer grounded in the provided context.

## RAG in WSO2 Integrator

WSO2 Integrator provides a `VectorKnowledgeBase` that wires together a vector store and an embedding provider. You ingest documents once, then call `retrieve` and augment the user query for each question.

```ballerina
import ballerina/ai;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
final ai:ModelProvider modelProvider = check ai:getDefaultModelProvider();

// Ingest documents into the knowledge base.
ai:DataLoader loader = check new ai:TextDataLoader("./policy.md");
ai:Document|ai:Document[] documents = check loader.load();
check knowledgeBase.ingest(documents);

// Answer a question using retrieved context.
string query = "What is the return policy?";
ai:QueryMatch[] matches = check knowledgeBase.retrieve(query, 10);
ai:Chunk[] context = from ai:QueryMatch m in matches select m.chunk;
ai:ChatUserMessage augmented = ai:augmentUserQuery(context, query);
ai:ChatAssistantMessage response = check modelProvider->chat([augmented]);
```

## RAG vs. Fine-Tuning

| Approach | Best For | Data Requirements |
|----------|----------|-------------------|
| **RAG** | Dynamic knowledge, frequently updated data | Any amount of text |
| **Fine-tuning** | Changing model behavior or style | Curated training examples |
| **RAG + Fine-tuning** | Domain-specific behavior with dynamic knowledge | Both |

## What's Next

- [Chunking Documents](/docs/genai/develop/rag/chunking-documents) -- Chunking strategies for RAG
- [Generating Embeddings](/docs/genai/develop/rag/generating-embeddings) -- Embedding model selection
- [Connecting to Vector Databases](/docs/genai/develop/rag/connecting-vector-dbs) -- Vector store setup
- [RAG Querying](/docs/genai/develop/rag/rag-querying) -- Building the query pipeline
