---
sidebar_position: 2
title: Generating Embeddings
description: Convert text chunks into vector embeddings using the default WSO2 Integrator embedding provider or a custom EmbeddingProvider.
---

# Generating Embeddings

Embeddings convert text chunks into numerical vectors that capture semantic meaning. Similar content produces similar vectors, enabling semantic search in your RAG pipeline.

In WSO2 Integrator, embeddings are produced by an `ai:EmbeddingProvider`. The framework ships with a default provider so you can get started without touching any API keys, and it is easy to swap in a provider of your own.

## Using the Default Embedding Provider

The default embedding provider is backed by WSO2 and is ready to use out of the box. Obtain it with `ai:getDefaultEmbeddingProvider` and pass it directly to your knowledge base.

```ballerina
import ballerina/ai;
import ballerina/io;

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:EmbeddingProvider embeddingProvider = check ai:getDefaultEmbeddingProvider();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);

public function main() returns error? {
    ai:DataLoader loader = check new ai:TextDataLoader("./leave_policy.md");
    ai:Document|ai:Document[] documents = check loader.load();

    // Embeddings are generated automatically during ingestion
    check knowledgeBase.ingest(documents);
    io:println("Documents embedded and stored");
}
```

You never call the embedding provider directly in this pattern. The knowledge base takes care of embedding each chunk before writing it to the vector store, and it embeds the query transparently when you call `retrieve`.

## Using a Different Embedding Provider

To use a provider other than the default, pull in the matching connector and pass it to `VectorKnowledgeBase`. WSO2 Integrator provides embedding providers for OpenAI, Azure OpenAI, Anthropic, and others.

```ballerina
import ballerina/ai;
import ballerinax/ai.openai;

configurable string openAiApiKey = ?;

final ai:EmbeddingProvider embeddingProvider = check new openai:EmbeddingProvider(
    apiKey = openAiApiKey,
    modelType = openai:TEXT_EMBEDDING_3_SMALL
);

final ai:VectorStore vectorStore = check new ai:InMemoryVectorStore();
final ai:KnowledgeBase knowledgeBase =
        new ai:VectorKnowledgeBase(vectorStore, embeddingProvider);
```

```toml
# Config.toml
openAiApiKey = "sk-..."
```

Because every provider satisfies the same `ai:EmbeddingProvider` interface, you can swap providers without changing any of your RAG code downstream.

## Retrieving with Embeddings

When you call `retrieve`, the knowledge base embeds your query using the same provider that was used during ingestion, so query and document vectors live in the same semantic space.

```ballerina
string query = "How many annual leave days can a full-time employee carry forward?";
ai:QueryMatch[] matches = check knowledgeBase.retrieve(query, 10);

ai:Chunk[] context = from ai:QueryMatch m in matches select m.chunk;
```

The `topK` argument (here, `10`) controls how many nearest-neighbour chunks are returned.

## Writing a Custom Embedding Provider

If you need to integrate with a provider not covered by an existing connector, implement the `ai:EmbeddingProvider` interface yourself. This gives you full control over how text is turned into vectors while keeping the rest of the RAG pipeline unchanged.

```ballerina
import ballerina/ai;

isolated client class MyEmbeddingProvider {
    *ai:EmbeddingProvider;

    isolated remote function embed(ai:Chunk chunk) returns ai:Embedding|ai:Error {
        // Call your embedding service and return an ai:Embedding
        return [0.1, 0.2, 0.3];
    }

    isolated remote function batchEmbed(ai:Chunk[] chunks)
            returns ai:Embedding[]|ai:Error {
        // Batch-embed for efficiency
        return from ai:Chunk c in chunks select <ai:Embedding>[0.1, 0.2, 0.3];
    }
}
```

Once defined, pass an instance of your provider to `VectorKnowledgeBase` the same way you would pass the default provider.

## Tips

- **Use the same provider for ingestion and retrieval.** Vectors from different models are not comparable.
- **Prefer batch embedding** when ingesting many chunks. Most providers charge per request and batch calls are significantly faster.
- **Store `Config.toml` credentials outside source control.** Use environment variables or a secrets manager in production.

## What's Next

- [Chunking Documents](/docs/genai/develop/rag/chunking-documents) -- Chunking strategies for RAG
- [Connecting to Vector Databases](/docs/genai/develop/rag/connecting-vector-dbs) -- Store embeddings for search
- [RAG Querying](/docs/genai/develop/rag/rag-querying) -- Build the query pipeline
